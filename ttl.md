---
layout: default
title: Message TTL
---

# Feature: TTL (time to live) for messages

When sending a message you can set its TTL and the framework will silently discard it when it's not consumed within the specified time.
This is useful for example when sending email notifications - if the mail server is unavailable for several days then it might make sense
to discard queued notifications instead of sending outdated ones.

{% highlight C# %}
MessageBus.NewMessage(new Ping()).SetTTL(DateTime.Now.AddMinutes(10)).Publish()
{% endhighlight %}

