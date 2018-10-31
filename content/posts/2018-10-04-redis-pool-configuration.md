---
title: "[EN] Redis pool configuration with Spring Boot"
date: 2018-10-04T20:16:38+02:00
draft: true
emoji: true
---

![Spring boot and Redis](https://sdqali.in//images/redis-spring-boot.png)

To make our CMS fast and efficient, we obviously use caching. At Ouest-France, we choosed Redis.

But sometimes, after a brand new deployement on our kubernetes we were experiencing a lot more of cache MISS and this very strange and generic exception:

{{< highlight java>}}
org.springframework.data.redis.RedisConnectionFailureException: Cannot get Jedis connection; nested exception is redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool
	at org.springframework.data.redis.connection.jedis.JedisConnectionFactory.fetchJedisConnector(JedisConnectionFactory.java:204)
	at org.springframework.data.redis.connection.jedis.JedisConnectionFactory.getConnection(JedisConnectionFactory.java:348)
	at org.springframework.data.redis.core.RedisConnectionUtils.doGetConnection(RedisConnectionUtils.java:129)
	at org.springframework.data.redis.core.RedisConnectionUtils.getConnection(RedisConnectionUtils.java:92)
	at org.springframework.data.redis.core.RedisConnectionUtils.getConnection(RedisConnectionUtils.java:79)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:194)
    ...
Caused by: redis.clients.jedis.exceptions.JedisConnectionException: java.net.SocketTimeoutException: connect timed out
	at redis.clients.jedis.Connection.connect(Connection.java:207)
	at redis.clients.jedis.BinaryClient.connect(BinaryClient.java:93)
	at redis.clients.jedis.BinaryJedis.connect(BinaryJedis.java:1767)
	at redis.clients.jedis.JedisFactory.makeObject(JedisFactory.java:106)
	at org.apache.commons.pool2.impl.GenericObjectPool.create(GenericObjectPool.java:868)
	at org.apache.commons.pool2.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:435)
{{< / highlight >}}

As usual with this exception, you cannot find anything over the internet :worried:.

At Ouest-France, the whole stack is deployed in kubernetes, so we start searching in this direction and spot that this issue was only happening when a node was hosting twice the same pod :thinking_face:. And in this case, one of the pod was nearly always failing to establish a Redis connection :sweat_smile:.

This is very clear on the dashboard below. 3 pods are responding around 120ms when 3 others are around 200/250ms.

![Pod response time disparity](images/posts/2018-10-04-pod-response-time-disparity.png)

We then start digging in the kubernetes issues and found one that was describing strange behaviour when 2 pods are trying to establish a connection to the same target host simultanetly (https://github.com/kubernetes/kubernetes/issues/62628).

The issue is in fact related to an issue / race condition in the linux kernel. It is welle explained here: https://blog.quentin-machu.fr/2018/06/24/5-15s-dns-lookups-on-kubernetes/

**TLDR;**

All the sockets between a client and a server are etablished between the source / target ip and **source** / target **ports**. You can easily see that by runniing the `netstat` command: 

```
tcp        0      0 mpousse.ouest-fra:40748 stackoverflow.com:443   ESTABLISHED
```

In this case, a connection is established between my local computer and stackoverflow by using port 40748 on my local.

The source port is automatically determined by the kernel by trying to find an available port number.

To determine the next available port, the kernel tries to find a free port, associate it to process, but, because of a race condition, it can associate the same port to two process.

In this case, only one of the two will be able to use this socket, the other one will fail.

For information, this issue is being fixed in the kernell. A workaround is to change the strategy to determine the next free port by using randomness.

---

Even if we know this issue, we understand that everytime we create a new connection, it might fail. But as far as we are using connection pool, this should happen only until the pool are provisionned and then it should stop.

Why do we always see this exception? : thinking_face:

Well, we dig in our redis monitoring dashboard, we spot that we had about 1200 new connections from our application to redis :cold_sweat:. The goal of using a connection pool is to avoid such situation.

We then double check our redis configuration: 

{{< highlight yaml>}}
spring:
  redis:
    pool:
      min-idle: 100
      max-wait: 100
      max-active: 300
    timeout: 200
{{< / highlight >}}

With such configuration, we really thought this means we want at least 100 open connections, and up to 300 simultaneously connection.

But... the mistery hides in this class: `org.springframework.boot.autoconfigure.data.redis.RedisProperties.Pool`.  This class holds the configuration defined in the `application.yaml`.


And in this class we spot this line: 

{{< highlight java>}}
    private int maxIdle = 8;
{{< / highlight >}}

As you can now understand, the max-idle configuration is stronger than the min-idle and cap the number of open connection to 8 :cry:.

Be careful when you configure a connection pool in spring. Most of the time, if you forget to set the max-idle property, it will result in ignoring more or less the min-idle and generating dozen of new connection per second.