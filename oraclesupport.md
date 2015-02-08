---
title: Oracle support
layout: default
---
## Running nginn-messagebus with Oracle database

### Supported features

NGinn-messagebus was originally designed to run on SQL Server and makes use of several SQL server-specific features, but it was possible to port it to Oracle database. 
The approach and database structure is almost identical and the set of supported functionalities remains the same. There are, however, quite a lot tiny differences in SQL syntax, the way Oracle handles queries or in how the driver works,
that affect the overall system performance. For example, there are no batch inserts in Oracle and therefore we can't push all outgoing messages to database in one query. 


### Differences

There are no differences in the API or in the way messages are delivered to handlers and processed. 
The code that uses nginn-messagebus does not need to be recompiled or modified in any other way - everything should just work as with SQL Server. The only thing that differs is the configuration.

### Configuration

We're using the official Oracle .Net driver: "Oracle Data Provider for .NET, Managed Driver" that can be downloaded from <a href="http://www.oracle.com/technetwork/topics/dotnet/index-085163.html">their website</a>. 
Using ODP.Net package from NuGet is not recommended because it does not include a native "Oracle.ManagedDataAccessDTC.dll" library that is required for supporting distributed transaction and cooperation with System.Transactions API. <br/>
So, the first step to using nginn-messagebus with Oracle driver is to add required binaries to your .Net project. NGinn-messagebus will not do that (it has no dependency on Oracle driver), 
so please install `ODP.Net` and add `Oracle.ManagedDataAccess.dll` to your depends. Make sure proper version (x86 or x64) of `Oracle.ManagedDataAccessDTC.dll` is present in the binaries directory.

Next thing to do is configuration of connection strings. Below is an example that worked in our tests. There are other possible forms of Oracle connection strings but we didn't check them. 
Please note that the easiest way to configure NGinn-messagebus connection strings is to specify them in app.config file in connectionStrings section.

```
<add name="oradb" connectionString="Data source=(DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.9.106)(PORT = 1521))(CONNECT_DATA =(SID = xe)));User ID=testuser;Password=PASS;" providerName="Oracle.DataAccess.Client" />
```

