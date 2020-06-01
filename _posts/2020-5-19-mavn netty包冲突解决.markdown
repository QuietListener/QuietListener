---
layout: post
title: mavn netty包冲突解决
date: 2020-5-8 14:32:00
categories:  java maven
---
报错:
```java
java.lang.NoSuchMethodError: io.netty.util.internal.PlatformDependent.isOsx()Z

	at io.netty.resolver.dns.DnsServerAddressStreamProviders.<clinit>(DnsServerAddressStreamProviders.java:41)
	at org.redisson.connection.MasterSlaveConnectionManager.<init>(MasterSlaveConnectionManager.java:211)
	at org.redisson.connection.MasterSlaveConnectionManager.<init>(MasterSlaveConnectionManager.java:163)
	at org.redisson.connection.SingleConnectionManager.<init>(SingleConnectionManager.java:34)
	at org.redisson.config.ConfigSupport.createConnectionManager(ConfigSupport.java:197)
	at org.redisson.Redisson.<init>(Redisson.java:64)

```

