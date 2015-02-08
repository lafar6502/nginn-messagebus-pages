---
layout: page
title: Getting started
---
Short introduction to how NGinn.MessageBus works and how to use it in your applications.


## Basic information 

### What is it 
NGinn.MessageBus is an asynchronous message bus - a communication mechanism that transports messages between communicating components using message queues. This one was designed for the Microsoft.Net platform and is targeted mainly at applications using MS SQL Server as the database, but of course can be also used with other types of software.

### Messages 
Let's start with the message. In NGinn.MessageBus a message is simply a .Net object - an instance of a message class. Usually messages are plain DTOs - they contain only data fields and no logic. Here's a sample message class:

{% highlight C# %}
    public class StartProcess
    {
        public string DefinitionId { get; set; }
        public Dictionary<string, object> Data { get; set; }
        public string StartedBy { get; set; }
        public string ExternalId { get; set; }
        public bool NotifyOnCompletion { get; set; }
    }
{% endhighlight %}

You don't have to mark messages as [Serializable] because NGinn.Messsagebus uses Json serialization. If you want to control what properties are serialized you can use DataContract attribute on the message class and DataMember / IgnoreDataMember attributes on its properties.

###= Endpoints ###=
An endpoint in NGinn.MessageBus is a message queue. Since message queues are just database tables an endpoint is simply a table in some SQL server database. This fact is reflected in the endpoint naming convention.
Take this endpoint: `sql://appdb1/MessageQueue1`. The first part, 'sql://', indicates that this is a SQL queue. The second part - 'appdb1'  - is the database alias. It means our queue resides in appdb1 database and 'appdb1' is an alias of an actual connection string for that database. The third part - MessageQueue1 - is the name of queue table in that database. Each queue is stored in its own table so the endpoint uniquely identifies the queue.

###= Local and remote endpoints ###=
When you configure NGinn.MessageBus in your application you specify its endpoint - the queue it will receive messages from. This queue is the 'local' endpoint. Usually you will have one local endpoint per application.

When you send a message to another application you specify the destination endpoint (the endpoint where that application is listening at) - in this case it's a remote endpoint. 


###= Sending messages ###=

You can send messages using the IMessageBus interface. This is the main interface for accessing the message bus functionality.
Generally there are two 'modes' of sending a message - you can publish it or send it. 

Publishing a message will distribute that message to all recipients that are interested in receiving this message type. When publishing you don't specify the destination address because the message bus decides itself where to deliver that message. By default, when no other endpoints have subscribed to the message, the message will be published to local endpoint.

When sending a message you have to specify a destination endpoint and the message will be sent to that endpoint only.

This is how you send and publish a message:
{% highlight C# %}
IMessageBus messageBus;
                
//publishing
messageBus.Notify(new TestMessage { Id = "32343", Text = "test message" });

//sending to a specific endpoint
messageBus.Send(new TestMessage { Id = "32223", Text = "test message 2" }, 
	"sql://otherdb/MessageQueueX");
{% endhighlight %}

###= Receiving messages ###=
NGinn.MessageBus delivers the incoming messages to your application by routing them to their handlers (classes responsible for handling incoming messages). This means that your application doesn't have to do anything to 'receive' a message - the message will get delivered to a handler automatically.

Messages are routed by their type - each message handler declares what type of messages it wants to receive and NGinn.MessageBus takes care of delivering incoming messages to their handlers. You define a message handler by implementing the *`IMessageConsumer`* interface. This is a generic interface with only one method: *`Handle`*.

Here's a message handler class:

{% highlight C# %}
public class TestHandler : IMessageConsumer<TestMessage1>
{
    void Handle(TestMessage1 message)
    {
        log.Info("TestMessage1 arrived with Id={0}", message.Id);
    }
}
{% endhighlight %}
This handler receives only one message type: *`TestMessage1`*. You can also handle more types of messages in one handler class:
{% highlight C# %}
public class TestHandler : 
    IMessageConsumer<TestMessage1>, 
    IMessageConsumer<TestMessage2>
{
    void Handle(TestMessage1 message)
    {
        log.Info("TestMessage1 arrived with Id={0}", message.Id);
    }

    public void Handle(TestMessage2 message)
    {
        throw new NotImplementedException();
    }
}
{% endhighlight %}
It is important to know that a message will be delivered to all handlers of this message's type.
Another important thing is *local message routing* - messages published to local endpoint will be received by the same application and routed to their handlers. This way the message bus is also a mechanism for local communication between components of the same application.

As you can see, there are two levels of message routing in NGinn.MessageBus. First level is the inter-application communication by delivering messages to particular queue (endpoint). The second level is the in-application routing of messages from local endpoint to appropriate handler class.

See the [Transactions] page for more details about message handling.

### Publish/Subscribe 
Publish/subscribe message distribution allows you to configure the rules of automatic distribution of messages between endpoints. These rules are used to determine the destinations of a message when someone publishes it.  

Basically, you subscribe to a specified message type at a specified remote endpoint: after you subscribe to message type *T* at endpoint *X* NGinn.MessageBus will send the *T* message to your local endpoint every time it is published at endpoint *X*.

You have seen how to publish a message earlier, now let's see how to subscribe to some messages:

{% highlight C# %}
//my local endpoint is sql://test/MessageQueue and I'm subscribing at sql://appX/MQueue1
messageBus.SubscribeAt("sql://appX/MQueue1", typeof(TestMessage));
{% endhighlight %}
After calling this all TestMessages published at *sql://appX/MQueue1* will be delivered to our local queue. The remote application at sql://appX/MQueue1 doesn't even have to know that we are now receiving messages from it.

In local communication you don't have to subscribe to messages - all messages published at local endpoint are received at this endpoint. 

Subscriptions are permanent (for some time) - they will survive your application's shutdown and restart so it is important to unsubscribe if you are no longer interested in receiving messages or if you are removing the receiving queue. This is the code to unsubscribe:
{% highlight C# %}
messageBus.UnsubscribeAt("sql://appX/MQueue1", typeof(TestMessage));
{% endhighlight %}
NGinn.MessageBus has a mechanism of automatic cleanup of forgotten subscriptions so undelivered messages won't pile up indefinitely when the subscriber is unavailable. By default the expiration time is 48 hours - if the receiver is unavailable for 48 hours its subscriptions will be cancelled.

### Scheduled messages 
When sending or publishing a message you can specify the delivery date - there is special version of `IMessageBus.Notify` and `IMessageBus.Send` for that purpose. Messages with a delivery date will be delivered at that date (or few seconds later). Example:
{% highlight C# %}
//both messages will be delivered after 5 minutes
messageBus.NotifyAt(DateTime.Now.AddMinutes(5), new TestMessage());
messageBus.SendAt(DateTime.Now.AddMinutes(5), "sql://appX/MQueue1", new TestMessage());

{% endhighlight %}
 
## Setup
[ConfigurationWindsor] describes steps necessary for configuring NGinn.MessageBus in your application.

*(to be continued)*