---
title: Saga support
layout: default
---
# Using sagas with nginn-messagebus

## What are sagas 

Saga is a piece of data that represents current state of some long-running process, a process that reacts to incoming messages. Sagas in nginn-messagebus are initiated by a message and then can process other incoming messages. Each saga is a distinct instance of a process and therefore handles only messages that are sent to that instance - nginn-messagebus makes sure saga messages are delivered to proper saga instances.
Sagas are persistent and transactional.

### But why are they in the message bus? 

Just for your convenience. Long running processes and async messaging quite often go together. Sagas make it easy to implement such processes so you don't have to implement process state persistence, message correlation, concurrency, versioning and so on.

## Example of a saga - sending SMS messages 

Let's take a real world example of a process that can be implemented using a saga: 
Our task is to create a service for sending SMS messages to customers. After the message is sent our service should also collect a delivery report from the SMS gateway and send a status update to the application that has sent the SMS. If no delivery report is received in 3 days we should inform the application that the SMS has not been delivered.

Sending an SMS in our example is a long running process because after sending the message to the SMS gateway we need to wait for the delivery report for that message. The wait time can be quite long - up to 3 days - so the process is certainly a long-running one.

## Implementation 

Basically, our SMS service sits between an application and the SMS gateway and acts as an intermediary in communication between the application and the SMS gateway. The 'value added' by our service is handling of message delivery reports and a reliable API for sending SMS messages.

In our process of sending an SMS we'll have to handle the following events:

  1. Application sends an SMS (this initiates the process)
  2. SMS service sends an SMS to the SMS gateway
  3. SMS gateway sends us a delivery report
  4. We (SMS service) send the delivery report to the application
  5. Delivery time-out occurs (no delivery report received)

Each of these events will be represented by a message.
Disclaimer: we wont't go into details of how the communication with the SMS gateway is implemented - let's just assume that proper messages are published to the messagebus by some component responsible for communication and that's all.

So, let's start with defining our messages:

{% highlight C# %}
/// <summary>
/// Application's request to send an SMS
/// </summary>
public class SendSMSRequest
{
    public string MSISDN { get; set; }
    public string Text { get; set; }
    public string MessageId { get; set; }
}

/// <summary>
/// SMS delivery report for the application
/// </summary>
public class SMSDeliveryReport
{
    public bool Delivered { get; set; }
    public string Reason { get; set; }
    public string MessageId { get; set; }
}

/// <summary>
/// Send a SMS to the SMS gateway
/// </summary>
public class SendSMSToGateway
{
    public string MSISDN { get; set; }
    public string Text { get; set; }
    public string SMSId { get; set; }
}

/// <summary>
/// SMS gateway's delivery status message
/// </summary>
public class GatewayDeliveryReport
{
    public bool Success { get; set; }
    public string SMSId { get; set; }
}

/// <summary>
/// Message for handling delivery report timeout
/// </summary>
public class SMSDeliveryTimeout
{
    public string MessageId { get; set; }
}
{% endhighlight %}


Now we'll implement a saga that orchestrates all these messages. This is a saga class that doesn't yet handle any messages but holds some state (a SmsState object):

{% highlight C# %}
public class SmsSaga : Saga<SmsSaga.SmsState>
{
    public class SmsState
    {
        public DateTime MessageSentDate { get; set; }
    }
}
{% endhighlight %}

Now we'll add message handlers to it. Let's start with the `SendSMSRequest` message that initiates the whole process:

{% highlight C# %}
public class SmsSaga : Saga<SmsSaga.SmsState>, InitiatedBy<SendSMSRequest>
{
    public class SmsState
    {
        public DateTime MessageSentDate { get; set; }
        public bool DeliveryReportReceived { get; set; }
    }

    public IMessageBus MessageBus { get; set; }

    protected override void Configure()
    {
        //tell the message bus that saga ID will be passed in the MessageId field fo SendSMSRequest message
        SagaId<SendSMSRequest>(x => x.MessageId);
    }

    public void Handle(SendSMSRequest message)
    {
        //update saga state
        Data.MessageSentDate = DateTime.Now;
        Data.DeliveryReportReceived = false;

        //send a request to the SMS gateway
        MessageBus.NewMessage(new SendSMSToGateway { MSISDN = message.MSISDN, SMSId = message.MessageId, Text = message.Text })
            .Send("sql://gateway/Queue");
        
        //and schedule a timeout message to be delivered in 3 days from now
        MessageBus.NewMessage(new SMSDeliveryTimeout { MessageId = message.MessageId })
            .SetDeliveryDate(DateTime.Now.AddDays(3))
            .Publish();

    }
}

{% endhighlight %}

Lot of things is happening here. First of all, we implemented the `InitiatedBy<SendSmsRequest>` interface that adds the `Handle(SendSMSRequest)` method. This interface informs message bus that it should create a new instance of `SmsSaga` when a `SendSMSRequest` message arrives.
Apart from that, we have implemented a `Configure` method. This method's purpose is to define how messages handled by the saga should be used for identifying the saga instance. In our case we told message bus that the saga ID is in `SendSMSRequest.MessageId` field. Now we only have to take care that the MessageId is unique.

Now let's take a look what SmsSaga is doing after receiving the `SendSMSRequest`. First, we update the saga state (in the `Data` property). Then we send a request to the SMS gateway (the `SendSMSToGateway` message). And finally, we publish the `SMSDeliveryTimeout` message to our local message queue and schedule it for delivery 3 days from now. That's all - now we are waiting for the delivery report from the SMS gateway or the timeout - whichever occurs first. So, let's add the implementation of these two cases:

{% highlight C# %}
public class SmsSaga : Saga<SmsSaga.SmsState>, 
    InitiatedBy<SendSMSRequest>,
    IHandleSagaMessage<GatewayDeliveryReport>,
    IHandleSagaMessage<SMSDeliveryTimeout>
{
    public class SmsState
    {
        public DateTime MessageSentDate { get; set; }
        public bool DeliveryReportReceived { get; set; }
        public string SenderEndpoint { get; set; }
    }

    public IMessageBus MessageBus { get; set; }

    protected override void Configure()
    {
        //tell the message bus that saga ID will be passed in the MessageId field of SendSMSRequest message
        SagaId<SendSMSRequest>(x => x.MessageId);
        SagaId<GatewayDeliveryReport>(x => x.SMSId);
        SagaId<SMSDeliveryTimeout>(x => x.MessageId);
    }

    public void Handle(SendSMSRequest message)
    {
        //update saga state
        Data.MessageSentDate = DateTime.Now;
        Data.DeliveryReportReceived = false;
        Data.SenderEndpoint = MessageBus.CurrentMessageInfo.Sender; //store the sender's address for later use

        //send a request to the SMS gateway
        MessageBus.NewMessage(new SendSMSToGateway { MSISDN = message.MSISDN, SMSId = message.MessageId, Text = message.Text })
            .Send("sql://gateway/Queue");
        
        //and schedule a timeout message to be delivered in 3 days from now
        MessageBus.NewMessage(new SMSDeliveryTimeout { MessageId = message.MessageId })
            .SetDeliveryDate(DateTime.Now.AddDays(3))
            .Publish();

    }

    public void Handle(GatewayDeliveryReport message)
    {
        if (!Data.DeliveryReportReceived)
        {
            Data.DeliveryReportReceived = true;
            //send the report to the application
            MessageBus.NewMessage(new SMSDeliveryReport { Delivered = message.Success, Reason = message.Success ? "" : "Delivery failure", MessageId = Id })
                .Send(Data.SenderEndpoint);
        }
        //we could complete the saga here but let's wait for the timeout message
        //so it is processed without an error
    }

    public void Handle(SMSDeliveryTimeout message)
    {
        if (!Data.DeliveryReportReceived)
        {
            //inform the application about timeout
            MessageBus.NewMessage(new SMSDeliveryReport { Delivered = false, Reason = "Timeout", MessageId = this.Id })
                .Send(Data.SenderEndpoint);
        }
        SetCompleted(); //mark the saga as completed so it can be removed from the db
    }
}
{% endhighlight %}

Now the implementation is complete. We implemented two additional `IHandleSagaMessage` interfaces for `GatewayDeliveryReport` and `SMSDeliveryTimeout` messages. We also added saga 'resolvers' for these messages in the `Configure` method. 
Our saga is complete after the `SMSDeliveryTimeout` message is received. The handler of that message calls `SetCompleted` at the end to inform message bus that the saga is completed and can be deleted from the database (an no further messages are expected).
We could also complete the saga after receiving the `GatewayDeliveryReport` message but then the saga would be deleted before receiving `SMSDeliveryTimeout` message and there would be an error reported for that message.

And basically, that's all you need to know to implement sagas with nginn-messagebus.

## Saga Id and correlation 
As you have seen in the application above, the Id assigned to the saga was retrieved from the initiating `SendSMSRequest` message and then it was present in all other message used in this scenario. This was the common identifier for all messages in that particular process instance and in this particular case the saga Id was the same as the message Id assigned by the application. The benefit was that everyone was using the same identifier so no mapping had to be done. 

Sometimes, however, you will not specify saga Id in the initiating message - in this case nginn-messagebus will generate new, unique saga Id on receiving the message and this Id will be then used for correlating all other messages in that saga. This means that if the initiating application needs to 'talk to' the saga later it must know the saga's Id. The id can be passed back in the reply to the initating message.

There's one more way of correlating sagas and messages: NGinn-messagebus allows you to specify an additional identifier ('CorrelationId') for any message sent:

{% highlight C# %}
MessageBus.NewMessage(new SomeMessage())
    .SetCorrelationId("298128993932320")
    .Publish();
{% endhighlight %}

If the message correlation id is specified and no other mappings have been defined in the saga's `Configure` method, the message correlation Id will be used as the saga Id. It will also become the saga Id if specified for the initiating message.

## Sagas and transactions 

Sagas are updated as a result of receiving a saga message and therefore all modifications are a part of the message receiving transaction. There's one great thing about nginn-messagebus and sagas: if the saga state is kept in the same database as the message queues (the default setting) you will have transactional sagas and messaging without using distributed transaction - this means great performance.