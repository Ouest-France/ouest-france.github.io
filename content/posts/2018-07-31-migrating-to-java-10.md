---
title: "[EN] Migrating to Java 10"
date: 2018-07-31T23:20:12+02:00
draft: false
emoji: true
---

![Welcome Java 10](https://cdn.crunchify.com/wp-content/uploads/2018/04/Java-10-Released-Crunchify-Tutorial.jpg)


Today was a great day. We decided to migrate from Java 8 to Java 10 :tada:.

# Migration

Our main application is running spring-boot 1.5.x, and despite this [disclaimer](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-with-Java-9-and-above#requirements), we gave a try and succeded! (we are not ready to migrate to spring-boot 2.x because we have a lot of tooling to migrate and this needs to be planned)

Even if Java 9 and 10 did not introduced major features we need, the migration is a good opportunity to prepare the future and to stick to latest versions. Our application is massively using `Threads` and generates dozen of objects in memory.
As the CMS response time is one of our key metrics, we are always happy to try new versions that enhance those aspects (G1 to rule them all!).

The migration was quiet easy. We basically added the missing dependencies requiered for Java > 8 and voilà !

> Our continuous integration ran the test suites and everything was fine.

# Reality versus expectation

But... after deploying the application in our staging environment, randomly, the application start throwing some `NoClassDefFoundError` about missing class for Redis :cold_sweat::cold_sweat::cold_sweat:.

After diging in spring source code, we figure out that the exception was coming from `org/springframework/data/redis/connection/jedis/JedisConnection.java:132`.

{{< highlight java "linenos=table,hl_lines=7-9 14-16,linenostart=118" >}}
	static {

		CLIENT_FIELD = ReflectionUtils.findField(BinaryJedis.class, "client", Client.class);
		ReflectionUtils.makeAccessible(CLIENT_FIELD);

		try {
			Class<?> commandType = ClassUtils.isPresent("redis.clients.jedis.ProtocolCommand", null)
					? ClassUtils.forName("redis.clients.jedis.ProtocolCommand", null)
					: ClassUtils.forName("redis.clients.jedis.Protocol$Command", null);

			SEND_COMMAND = ReflectionUtils.findMethod(Connection.class, "sendCommand",
					new Class[] { commandType, byte[][].class });
		} catch (Exception e) {
			throw new NoClassDefFoundError(
					"Could not find required flavor of command required by 'redis.clients.jedis.Connection#sendCommand'.");
		}

		ReflectionUtils.makeAccessible(SEND_COMMAND);

		GET_RESPONSE = ReflectionUtils.findMethod(Queable.class, "getResponse", Builder.class);
		ReflectionUtils.makeAccessible(GET_RESPONSE);
}
{{< / highlight >}}

This piece of code is responsible of guessing the version of Jedis (the Redis driver for Java), based on the presence of some classes.

After several tests, we spot that there was two ways in our application to reach this code.

 1. directly from the thread that handles the http request
 1. from a separate thread that was spawned by one of our `ForkJoinPool`

The second was the problem.

---

When you run a Tomcat server, there is a lot of 'magic' that occurs around the `ClassLoader`. In order to be able to hot load / reload / unload and isolate the applications, Tomcat run its own `ClassLoader`. To do so, it overrides the `ClassLoader` associated with the request thread.

So when you migrate to Java > 8, in fact the `ClassLoader` you are using during the request rendering is a `WebappClassLoader`. This hides the Java 9 new modules behavior, makes your application interact with classes as before.

When you spawn a `Thread` for a `ForkJoinPool`, your thread inherit from a `AppClassLoader` which directly comes from JDK, and this one loves modules! 
It actullay implements Java modules and the visibility between them. 

When we are running a spring-boot 1.5.x application with a Java 9 `ClassLoader` it does not work (as explained in the disclaimer :grin:)

# Workaround

As explained, Tomcat is automatically overrinding the `ClassLoader` from the jvm with one of its own. This class loader is module-proof and hides the new Java 9 features.

Then we decided to do the same with our thread. As soon as we spawn a new `Thread`, we set its class loader to the one from the current thread (issued from Tomcat), and voila!

We eventually managed to migrate our application with Java 10.

Of course, this workaround is clearly showing the limit of migrating to Java 10 without taking the modules into account.

Definitively, we will need to migrate to spring-boot 2 soon !