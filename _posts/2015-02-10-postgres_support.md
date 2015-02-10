---
title: Postgres support (almost)
layout: post
---
The upcoming postgres version will finally support 'skip locked' feature
that makes efficient queue implementation possible. 
[This post](http://michael.otacoo.com/postgresql-2/postgres-9-5-feature-highlight-skip-locked-row-level/) gives some details on the feature - it has been implemented in current development tree and will be a part of the upcoming 9.5 version. Today I compiled Postgres from sources and did some successful testing - the database behaves just as it should, the following SQL is basically an atomic 'select next unlocked row and lock it' operation.

{% highlight sql %}
select  * from mq_test where status='I' order by id for update skip locked limit 1;
{% endhighlight %}

I hope to have the implementation tested and ready before v9.5 of Postgres is released - hopefully this will happen in next few months.
