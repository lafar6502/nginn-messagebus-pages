---
layout: page
title: About
---
nginn-messagebus is a lightweight .Net message bus / ESB framework that provides reliable and performant messaging 
infrastructure for your applications and services. It uses a relational database as queue storage 
and builds on top of strong transactional/reliability guarantees provided by the database while making sure queue operations are 
as fast as possible.

### Supported databases
 * MS SQL Server
 * Oracle
 * upcoming support for Postgres (v 9.5)

### Applications
The primary use-case for nginn-messagebus is extending applications with asynchronous messaging infrastructure without introducing too
many dependencies and technologies and to make everything 'just work'. We assume your application is built on top of a SQL database
(MS SQL, Oracle, other) and so we use the database to provide message queuing functionality. almost for free, in the most desirable flavor
(transactional, integrated with app logic, performance matching application needs and no maintenance). Nginn-messagebus can be used within
single application to connect its internal components, or as an inter-process messaging tool.

### Cool features
 * fully transactional - messaging is a part of your application logic and works seamlessly with your transactions
 * super-performant if you use same database for your application data and message queues 
 * built-in message scheduling (timers), retrying, ttl, sequences, pub-sub routing, load balancing, sagas (long-running transactions)
 * powerful management tools - just use SQL for all queue operations
 * friction-free, lightweight, clean design and simple API

