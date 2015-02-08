---
title: Configuring nginn-messagebus with Castle Windsor container
layout: default
---
#Configuring nginn-messagebus with Castle Windsor container


### == Required libraries ==

NGinn.MessageBus binaries consist of two assemblies:
  * NGinn.MessageBus.dll - this is the main assembly
  * NGinn.MessageBus.Windsor.dll - this library is responsible for configuring NGinn.Messagebus with Castle Windsor IOC container. It contains all Windsor-specific code (the core library is not dependent on any particular IoC)
There are also few utility libraries needed to run NGinn.Messagebus:
  * NLog.dll - logging
  * Newtonsoft.Json.dll - json serialization
  * Castle.Core.dll
  * Castle.Windsor.dll - IOC container

### == Configuring NGinn.MessageBus ==

NGinn.MessageBus provides a configuration API that allows you to set up the message bus and host it in your application. This API is implemented on top of Castle Windsor container so if you need to use a different container you'll have to implement your own config builder.

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
  # `MessageBusConfigurator.Begin()` - starts building the host config
  # `.AddConnectionString` configures the connection string for local message queue database
  # `.SetEndpoint` sets the queue name (MQueue1 table)
  # `.AddMessageHandlersFromAssembly` registers all message handlers defined in current assembly
  # `.AutoStartMessageBus(true)` will automatically start receiving messages after configuration is done
  # `.FinishConfiguration` just configures everything according to earlier settings and starts the message bus if AutoStart is set

To retrieve the message bus main interface from the container you woulduse the following code:
{% highlight C# %}
IMessageBus bus = configurator.ServiceResolver.GetInstance<IMessageBus>();
{% endhighlight %}
but this is necessary when you want to use the message bus from outside the container. All components inside the container (e.g. your message handlers) will get the IMessageBus dependency injected by the container. So it is quite natural to use the same container for hosting all other components in the application - and the NGinn.MessageBus config API allows you to share the container with the application or customize the container used by NGinn.

Of course this is the simplest configuration possible, but it's enough to get started with NGinn.MessageBus.

### == Customizing and sharing the container ==

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

And finally, the container is always accessible by the *`configurator.Container`* property.