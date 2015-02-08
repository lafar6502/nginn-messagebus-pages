---
layout: default
title: Load balancing
---
## Distributing load between multiple message consumers
Here we're dealing with a situation when a message consumer is unable to handle incoming messages at an acceptable rate and as a result messages begin to pile up in the queue. To handle such cases you can simply run more instance of your program - all processes
will consume messages from same queue and the load will be distributed evenly between them. The framework guarantees that each message will be consumed by exactly one recipient so you can run any number of consumer processes without having to handle
concurrency in any special way. In case you're running in a single process you can just increase the number of message receiver threads (by adjusting the `MaxConcurrentMessages` configuration option in message bus setup).

