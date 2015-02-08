---
layout: default
title: Message sequences
---

# Message sequences - guaranteed order of delivery


By default nginn-messagebus does not guarantee that messages will be handled in the same order they were sent. If you want to make sure a group of messages will be handled in particular order, you have to use message sequence.

A message sequence is a group of messages with the same sequence ID. Each message in sequence must also have a sequence number - that is an integer that defines the order of messages in that sequence. Message numbers start with 0, must be consecutive and must not be duplicated - that is, it must be a contiguous range of numbers starting from 0.

To send messages in sequence you have to use message bus fluent interface, for example:

{% highlight C# %}
IMessageBus mb1 = wc1.Resolve<IMessageBus>();
mb1.NewMessage(new Ping()).InSequence("389293894003", 0, 5).Publish();
mb1.NewMessage(new Ping()).InSequence("389293894003", 1, null).Publish();                
{% endhighlight %}

This code sends two `Ping` messages with sequence Id = 389293894003. The second parameter of InSequence method is the order of message in sequence. The third one is sequence length (5 in this case). Sequence length is optional but if known it should be specified in at least one message of that sequence. This way the message bus will know when the sequence is completed and will be able to clean up it sequence database.
SequenceId must be unique at least for the destination queue, that is the queue that will receive the messages in sequence. 
Warning: if the sequence is incomplete (that is, one of messages is missing), all messages later than the missing one will not be handled - message bus will wait for the missing message indefinitely.

In order for sequences to work a sequence repository must be configured when initializing message bus. You have to configure it only for the receiving queue.

