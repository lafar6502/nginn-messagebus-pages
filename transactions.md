---
title: Transactional processing with nginn-messagebus.
layout: default
---
# Transactional processing with nginn-messagebus

### Receiving messages
Nginn-messagebus receives each message in a separate transaction. The transaction is started when retrieving the message from database and ends after all handlers of that message have executed. Any exception that is thrown by a handler will cause the transaction to rollback and the message will be returned to the queue (and then it will be re-scheduled for a  retry later). The transaction will be committed only if all handlers execute successfully.
By default you don't have to do anything to participate in the receiving transaction - nginn-messagebus uses .Net `System.Transactions` API to manage an ambient transaction. If you don't want to participate in the transaction just enclose your operation inside a new TransactionScope.

You are encouraged here to rely on .Net's automatic transaction management instead of doing your own. This will make your code cooperate better with the rest of the application and probably will save you some unpleasant surprises. There is, however, one thing to be careful about: distributed transactions. 

NGinn-messagebus starts a new transaction and opens a database connection when receiving a message. The connection participates in transaction and stays open until message handling is complete. If you open a new database connection in the same transaction (and most certainly you will) you will trigger a transaction promotion - your new connection will be also enlisted in the transaction and the transaction will be promoted to a distributed one, managed by MSDTC. In case of MS SQL 2005 this will happen even if your connection is for the same database as the nginn-messagebus receiving connection (with SQL server 2008 transaction will stay local). But this is normal, only this way can two database connections be a part of the same transactions and usually you will want it to happen exactly this way. BTW: sending a message to message bus inside a receiving transaction will not trigger a transaction promotion even in MSSQL 2005 because the receiving connection is re-used for sending internally.

Sometimes, however, you'll want to avoid distributed transactions - mostly because of performance penalty related to it. Of course you can always 'opt out' and use TransactionScopeOption.Suppress to ignore the ambient transaction, but then your code will not be transactional. But if you happen to use the same database for application data and nginn-messagebus queues you can avoid a distributed transaction. With SQL Server 2008 the client code is smart enough to avoid promotion to a distributed transaction when connecting to the same database and you don't have to do anything. With SQL 2005 you can use some tricks (describe ...).



### Sending messages

This paragraph is about sending messages with nginn-messagebus as a part of your application's transaction. We have discussed the message receiving case earlier so this time we will concentrate on what happens when we are not inside a message handler, that is we are not receiving a message and not participating in  a receiving transaction.
Well, nothing unusual happens. When you send a message with `IMessageBus.Notify` or `IMessageBus.Send` the message will be inserted to the database. This means a database connection will be opened. If you are not participating in a managed transaction each send will open (and close) a new connection.
If you want the messages to be sent as a part of your application's transaction you have to participate in a managed transaction - for example like that:

{% highlight C# %}
public void TransactionalOperation()
{
    using (TransactionScope ts = new TransactionScope())
    {
        using (IDbConnection con = new SqlConnection(ConnectionString))
        {
            con.Open();
            //use the connection to perform operations in the application database
            using (IDbCommand cmd = con.CreateCommand())
            {
                cmd.CommandText = "update something where possible is true";
                cmd.ExecuteNonQuery();
            }
            //and send some messages
            MessageBus.Notify(new SomethingUpdatedMessage());
            MessageBus.Send("sql://db2/MQueue", new AnotherMessage());
        }
        ts.Complete();
    }
}
{% endhighlight %}

Here, it is the application who initiates and manages a transaction. Nginn-messagebus will participate in this transaction when sending messages but this also means it will trigger a promotion to a distributed transaction (because Send/Notify opens a db connection).
As I have said earlier, you don't usually have to worry about this but it's better to know what's happening.

Using a TransactionScope when sending messages has one additional benefit to not using it: it will batch the send operations into one database operation. See:

{% highlight C# %}
//no transaction - this will open the db connection 10 times
//and execute 10 sql queries
for (int i = 0; i < 10; i++)
    MessageBus.Notify(new TestMessage1());
    
//in transaction
//this will batch all messages and send them in one
//database operation
using (TransactionScope ts = new TransactionScope())
{
    for (int i = 0; i < 10; i++)
        MessageBus.Notify(new TestMessage1());
    ts.Complete();
}
{% endhighlight %}
The code inside TransactionScope will execute faster because it will perform only one batch database insert. It's one more reason for using managed transactions.

Again, if you have a problem with distributed transaction and you happen to use the same database for application's data and nginn-messagebus, you can avoid the transaction promotion by giving your db connection to nginn-messagebus (this applies only to MSSQL 2005, for 2008 it's unnecessary):

{% highlight C# %}
public void TransactionalOperation()
{
    using (TransactionScope ts = new TransactionScope())
    {
        using (IDbConnection con = new SqlConnection(ConnectionString))
        {
            con.Open();
            //give the connection to nginn-messagebus
            MessageBusCurrentThreadContext.AppManagedConnection = con;
            
            using (IDbCommand cmd = con.CreateCommand())
            {
                cmd.CommandText = "update something where possible is true";
                cmd.ExecuteNonQuery();
            }
            MessageBus.Notify(new SomethingUpdatedMessage());
            MessageBus.Send("sql://db2/MQueue", new AnotherMessage());
            MessageBusCurrentThreadContext.AppManagedConnection = null;
        }
        ts.Complete();
    }
}
{% endhighlight %}
Nginn-messagebus will use that connection if it is possible aind if allowed by configuration. If the connection is not open or if it's for a different database, nginn-messagebus will not use it.

### Handling failures
Message processing failure occurs whenever an exception is thrown in the receiving transaction. In such case the receiving transaction is rolled back and the message is re-scheduled for processing later.  By default nginn-messagebus will retry the message 10 times before giving up. The time gap between consecutive retries is increasing about twice with each retry - so the first retry will be about 30 seconds after the failure, the second retry about minute after the first one, third one about two minutes after the second and so on. The last retry is almost a week after the initial failure. Failing message does not stop the processing of other messages - they will be handled as usual (but of course the order of messages will not be kept because the failing messages has been rescheduled). Such retrying strategy gives the application quite good fault tolerance and makes it self-healing after the cause of failure is removed. And the admins have almost a week for detecting and fixing the failure.