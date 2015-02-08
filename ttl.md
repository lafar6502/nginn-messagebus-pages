---
layout: default
title: Message TTL
---

# Feature: TTL (time to live) for messages

When sending a message you can set its TTL and the framework will silently discard it when it's not consumed in the 
{% highlight C# %}
MessageBus.NewMessage(new Ping()).SetTTL().Publish()
{% endhighlight %}
