---
title: Static routing
layout: default
---
## Static message routing

1. Configuration

{% highlight c# %}
MessageBusConfigurator.Begin()
	.UseStaticMessageRouting("routing.json")
	...
{% endhighlight %}

This tells message bus to use static message routing file named `routing.json`.
The file contains a json document that maps message types to their destination queues.
Each message type can point to more than one queue, in this case messages will be send to all queues listed.
And there's one 'catch-all' case "*" that is used for all other message types, and here it tells message bus to send them to the local queue
(identified by message bus local endpoint).

{% highlight javascript %}

{
	"*" : ["local"],
	"Tests.TestMessage2" : ["sql://testdb1/MQueue2"],
	"Tests.PingMessage" : ["sql://testdb1/MQueue2", "sql://otherdb/QueueX"]
}

{% endhighlight %}

Please note we're using just message class names with a namespace here, without specifying the assembly reference.
Also, message inheritance will be taken into account by the static router - for example, all messages inheriting from Tests.TestMessage2 class
will also be directed to sql://testdb1/MQueue2.
