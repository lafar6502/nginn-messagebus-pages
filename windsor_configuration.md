---
title: Configuring nginn-messagebus with Castle Windsor container
layout: default
---
#Configuring nginn-messagebus with Castle Windsor container


### Required libraries

NGinn.MessageBus binaries consist of two assemblies:

  * NGinn.MessageBus.dll - this is the main assembly
  * NGinn.MessageBus.Windsor.dll - this library is responsible for configuring NGinn.Messagebus with Castle Windsor IOC container. It contains all Windsor-specific code (the core library is not dependent on any particular IoC)
There are also few utility libraries needed to run NGinn.Messagebus:
  * NLog.dll - logging
  * Newtonsoft.Json.dll - json serialization
  * Castle.Core.dll
  * Castle.Windsor.dll - IOC container
  
However, with the introduction of NuGet package, all these dependencies are resolved automatically at installation time, so I'm keeping this list just to document the libraries necessary to run nginn-messagebus.

### Configuring NGinn.MessageBus 

NGinn.MessageBus provides a configuration API that allows you to set up the message bus and host it in your application. This API is implemented on top of Castle Windsor container (it configures the services inside a Windsor container), but you can also host nginn-messagebus in any other DI container
by providing your own configuration builder. The framework itself (NGinn.MessageBus.dll) is not dependent on any container - all Windsor dependencies are isolated in NGinn.MessageBus.Windsor.dll and this is basically just the configuration code plus some application utilities that make
interacting with the container easier.

So, in order to use NGinn-Messagebus in your application you need to provide some bootstrap code that will configure the services necessary and link them to your application so you can send and receive messages. You do it using the `'NGinnBPM.MessageBus.Windsor.MessageBusConfigurator` class
that provides a fluent config API. 

###  Using fluent interface

Let's start with the simplest possible but a complete example of how to configure and start the message bus:

{% highlight C# %}
using System;
using System.Collections.Generic;
using System.Text;
using NLog;
using NGinnBPM.MessageBus;
using System.Collections;
using NGinnBPM.MessageBus.Windsor;
using Castle.Windsor;

namespace Tests
{
    public class ConfigExample
    {

        public static void ConfigureTheBus()
        {
            var configurator = MessageBusConfigurator.Begin()
                .AddConnectionString("mydb", "Data Source=(local);Initial Catalog=NGinn;User Id=nginn;Password=PASS")
                .SetEndpoint("sql://mydb/MQueue1")
                .AddMessageHandlersFromAssembly(typeof(ConfigExample).Assembly)
                .AutoStartMessageBus(true)
                .FinishConfiguration();

            Console.WriteLine("Now the message bus is running");
            Console.ReadLine();
        }
    }
}
{% endhighlight %}

All configuration is done using the `MessageBusConfigurator` class from `NGinnBPM.MessageBus.Windsor` namespace. This class provides a fluent API that allows you to chain method calls. This is what exactly  happens in this code:

  * `MessageBusConfigurator.Begin()` - starts building the host config
  * `.AddConnectionString` configures the connection string for local message queue database
  * `.SetEndpoint` sets the queue name (MQueue1 table)
  * `.AddMessageHandlersFromAssembly` registers all message handlers defined in current assembly
  * `.AutoStartMessageBus(true)` will automatically start receiving messages after configuration is done
  * `.FinishConfiguration` just configures everything according to earlier settings and starts the message bus if AutoStart is set

To retrieve the message bus main interface from the container you woulduse the following code:

{% highlight C# %}
IMessageBus bus = configurator.ServiceResolver.GetInstance<IMessageBus>();
{% endhighlight %}

but this is necessary when you want to use the message bus from outside the container. All components inside the container (e.g. your message handlers) will get the IMessageBus dependency injected by the container. So it is quite natural to use the same container for hosting all other components in the application - and the NGinn.MessageBus config API allows you to share the container with the application or customize the container used by NGinn.

Of course this is the simplest configuration possible, but it's enough to get started with NGinn.MessageBus.

### Using app.config

Instead of setting all these options in code, you can tell the configurator to take them from the application config file. You do that with

{% highlight C# %}
MessageBusConfigurator.Begin()
	.UseAppConfig()
	.AddMessageHandlersFromAssembly(typeof(ConfigExample).Assembly)
	.AutoStartMessageBus(true)
	.FinishConfiguration();

{% endhighlight %}
	
You can of course mix these two approaches and set some options in app.config and other in the code. And you still must call
`AddMessageHandlers...` to register your message handlers - this is not possible from the app.config.

Database connection strings are configured in the `<connectionStrings>` section of app config, like so:

{% highlight xml %}
<connectionStrings>
    <add name="testdb1" connectionString="Data Source=(local);Initial Catalog=NGinn;User Id=nginn;Password=PASS" providerName="System.Data.SqlClient" />
    <add name="testdb2" connectionString="Data Source=(local);Initial Catalog=NGinn;User Id=nginn;Password=PASS" providerName="System.Data.SqlClient" />
	<add name="oradb" connectionString="Data source=(DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.9.106)(PORT = 1521))(CONNECT_DATA =(SID = xe)));User ID=testuser;Password=PASS;" providerName="Oracle.DataAccess.Client" />
</connectionStrings>
{% endhighlight %}

Please note that each connection string name is then used in endpoint names as a database alias.

All other config options are set in the `<appSettings>` section:

{% highlight xml %}

<appSettings>
    <add key="NGinnMessageBus.Endpoint" value="sql://testdb1/MQ_Test2" />
    <add key="NGinnMessageBus.MaxConcurrentMessages" value="4"/>
    <add key="NGinnMessageBus.RoutingFile" value="routing.json"/>
    <add key="NGinnMessageBus.HttpListener" value="http://+:1234" />
    <add key="NGinnMessageBus.MessageRetentionPeriod" value="1.00:00:00"/>
    <add key="NGinnMessageBus.EnableSagas" value="true"/>
    <add key="NGinnMessageBus.AutoCreateDatabase" value="true"/>
    <add key="NGinnMessageBus.AlwaysPublishLocal" value="false"/>
    <add key="NGinnMessageBus.SendOnly" value="false"/>
    
</appSettings>
{% endhighlight %}

### Customizing and sharing the container

You can customize the container used by NGinn.MessageBus with this API:

{% highlight C# %}
var configurator = MessageBusConfigurator.Begin()
                .AddConnectionString("mydb", "Data Source=(local);Initial Catalog=NGinn;User Id=nginn;Password=PASS")
                .SetEndpoint("sql://mydb/MQueue1")
                .CustomizeContainer(delegate(IWindsorContainer wc)
                {
                    //here you can do everything with the container
                    wc.AddComponent("myService", typeof(MyService));
                })
                ...
{% endhighlight %}

this is useful if you want to introduce changes before NGinn-MessageBus  configuration is completed.

You can also provide your own container and tell NGinn-MessageBus to configure itself in that container by using an overloaded Begin method:

{% highlight C# %}
IWindsorContainer wc = new WindsorContainer(); // application's container
var configurator = MessageBusConfigurator.Begin(wc)
...
{% endhighlight %}

And finally, the container is always accessible by the `configurator.Container` property.

Please read `MessageBusConfigurator` source code to see how the components interact and what configuration options are available.
