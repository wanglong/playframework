<!--- Copyright (C) 2009-2016 Typesafe Inc. <http://www.typesafe.com> -->
# Understanding Play thread pools

Play Framework is, from the bottom up, an asynchronous web framework.  Streams are handled asynchronously using iteratees.  Thread pools in Play are tuned to use fewer threads than in traditional web frameworks, since IO in play-core never blocks.

Because of this, if you plan to write blocking IO code, or code that could potentially do a lot of CPU intensive work, you need to know exactly which thread pool is bearing that workload, and you need to tune it accordingly.  Doing blocking IO without taking this into account is likely to result in very poor performance from Play Framework, for example, you may see only a few requests per second being handled, while CPU usage sits at 5%.  In comparison, benchmarks on typical development hardware (eg, a MacBook Pro) have shown Play to be able to handle workloads in the hundreds or even thousands of requests per second without a sweat when tuned correctly.

## Knowing when you are blocking

The most common place where a typical Play application will block is when it's talking to a database.  Unfortunately, none of the major databases provide asynchronous database drivers for the JVM, so for most databases, your only option is to using blocking IO.  A notable exception to this is [ReactiveMongo](http://reactivemongo.org/), a driver for MongoDB that uses Play's Iteratee library to talk to MongoDB.

Other cases when your code may block include:

* Using REST/WebService APIs through a 3rd party client library (ie, not using Play's asynchronous WS API)
* Some messaging technologies only provide synchronous APIs to send messages
* When you open files or sockets directly yourself
* CPU intensive operations that block by virtue of the fact that they take a long time to execute

In general, if the API you are using returns `Future`s, it is non-blocking, otherwise it is blocking.

> Note that you may be tempted to therefore wrap your blocking code in Futures.  This does not make it non-blocking, it just means the blocking will happen in a different thread.  You still need to make sure that the thread pool that you are using has enough threads to handle the blocking.

In contrast, the following types of IO do not block:

* The Play WS API
* Asynchronous database drivers such as ReactiveMongo
* Sending/receiving messages to/from Akka actors

## Play's thread pools

Play uses a number of different thread pools for different purposes:

* **Netty boss/worker thread pools** - These are used internally by Netty for handling Netty IO.  An application's code should never be executed by a thread in these thread pools.
* **Play default thread pool** - This is the thread pool in which all of your application code in Play Framework is executed.  It is an Akka dispatcher, and is used by the application `ActorSystem`. It can be configured by configuring Akka, described below.

> Note that in Play 2.4 several thread pools were combined together into the Play default thread pool.

## Using the default thread pool

All actions in Play Framework use the default thread pool.  When doing certain asynchronous operations, for example, calling `map` or `flatMap` on a future, you may need to provide an implicit execution context to execute the given functions in.  An execution context is basically another name for a `ThreadPool`.

In most situations, the appropriate execution context to use will be the **Play default thread pool**.  This can be used by importing it into your Scala source file:

@[global-thread-pool](code/ThreadPools.scala)

### Configuring the Play default thread pool

The default thread pool can be configured using standard Akka configuration in `application.conf` under the `akka` namespace. Here is default configuration for Play's thread pool:

@[default-config](code/ThreadPools.scala)

This configuration instructs Akka to create 1 thread per available processor, with a maximum of 24 threads in the pool.

You can also try the default Akka configuration:

@[akka-default-config](code/ThreadPools.scala)

The full configuration options available to you can be found [here](http://doc.akka.io/docs/akka/2.3.11/general/configuration.html#Listing_of_the_Reference_Configuration).

## Using other thread pools

In certain circumstances, you may wish to dispatch work to other thread pools.  This may include CPU heavy work, or IO work, such as database access.  To do this, you should first create a `ThreadPool`, this can be done easily in Scala:

@[my-context-usage](code/ThreadPools.scala)

In this case, we are using Akka to create the `ExecutionContext`, but you could also easily create your own `ExecutionContext`s using Java executors, or the Scala fork join thread pool, for example.  To configure this Akka execution context, you can add the following configuration to your `application.conf`:

@[my-context-config](code/ThreadPools.scala)

To use this execution context in Scala, you would simply use the scala `Future` companion object function:

@[my-context-explicit](code/ThreadPools.scala)

or you could just use it implicitly:

@[my-context-implicit](code/ThreadPools.scala)

## Class loaders and thread locals

Class loaders and thread locals need special handling in a multithreaded environment such as a Play program.

### Application class loader

In a Play application the [thread context class loader](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#getContextClassLoader--) may not always be able to load application classes. You should explicitly use the application class loader to load classes.

Java
: @[using-app-classloader](code/detailedtopics/ThreadPoolsJava.java)

Scala
: @[using-app-classloader](code/ThreadPools.scala)

Being explicit about loading classes is most important when running Play in development mode (using `run`) rather than production mode. That's because Play's development mode uses multiple class loaders so that it can support automatic application reloading. Some of Play's threads might be bound to a class loader that only knows about a subset of your application's classes.

In some cases you may not be able to explicitly use the application classloader. This is sometimes the case when using third party libraries. In this case you may need to set the [thread context class loader](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#getContextClassLoader--) explicitly before you call the third party code. If you do, remember to restore the context class loader back to its previous value once you've finished calling the third party code.

### Java thread locals

Java code in Play uses a `ThreadLocal` to find out about contextual information such as the current HTTP request. Scala code doesn't need to use `ThreadLocal`s because it can use implicit parameters to pass context instead. `ThreadLocal`s are used in Java so that Java code can access contextual information without needing to pass context parameters everywhere.

The problem with using thread locals however is that as soon as control switches to another thread, you lose thread local information. So if you were to map a `CompletionStage` using `thenApply`, and then try to access the HTTP context (eg, the session or request), it won't work.  To address this, Play provides an [`HttpExecutionContext`](api/java/play/libs/concurrent/HttpExecutionContext.html).  This allows you to capture the current context in an `Executor`, which you can then pass to the `CompletionStage` `*Async` methods such as `thenApplyAsync()`, and when the executor executes your callback, it will ensure the thread local context is setup so that you can access the request/session/flash/response objects.

To use the `HttpExecutionContext`, inject it into your component, and then pass the current context anytime a `CompletionStage` is interacted with.  For example:

@[http-execution-context](code/detailedtopics/httpec/MyController.java)

If you have a custom executor, you can wrap it in an `HttpExecutionContext` simply by passing it to the `HttpExecutionContext`s constructor.

## Best practices

How you should best divide work in your application between different thread pools greatly depends on the types of work that your application is doing, and the control you want to have over how much of which work can be done in parallel.  There is no one size fits all solution to the problem, and the best decision for you will come from understanding the blocking-IO requirements of your application and the implications they have on your thread pools. It may help to do load testing on your application to tune and verify your configuration.

> Given the fact that JDBC is blocking thread pools can be sized to the # of connections available to a db pool assuming that the thread pool is used exclusively for database access. Any lesser amount of threads will not consume the number of connections available. Any more threads than the number of connections available could be wasteful given contention for the connections.

Below we outline a few common profiles that people may want to use in Play Framework:

### Pure asynchronous

In this case, you are doing no blocking IO in your application.  Since you are never blocking, the default configuration of one thread per processor suits your use case perfectly, so no extra configuration needs to be done.  The Play default execution context can be used in all cases.

### Highly synchronous

This profile matches that of a traditional synchronous IO based web framework, such as a Java servlet container.  It uses large thread pools to handle blocking IO.  It is useful for applications where most actions are doing database synchronous IO calls, such as accessing a database, and you don't want or need control over concurrency for different types of work.  This profile is the simplest for handling blocking IO.

In this profile, you would simply use the default execution context everywhere, but configure it to have a very large number of threads in its pool, like so:

@[highly-synchronous](code/ThreadPools.scala)

This profile is recommended for Java applications that do synchronous IO, since it is harder in Java to dispatch work to other threads.

Note that we use the same value for `parallelism-min` and `parallelism-max`. The reason is that the number of threads is defined by the following formulas :

>base-nb-threads = nb-processors * parallelism-factor
 parallelism-min <= actual-nb-threads <= parallelism-max

So if you don't have enough available processors, you will never be able to reach the `parallelism-max` setting.

### Many specific thread pools

This profile is for when you want to do a lot of synchronous IO, but you also want to control exactly how much of which types of operations your application does at once.  In this profile, you would only do non blocking operations in the default execution context, and then dispatch blocking operations to different execution contexts for those specific operations.

In this case, you might create a number of different execution contexts for different types of operations, like this:

@[many-specific-contexts](code/ThreadPools.scala)

These might then be configured like so:

@[many-specific-config](code/ThreadPools.scala)

Then in your code, you would create `Future`s and pass the relevant `ExecutionContext` for the type of work that `Future` was doing.

> **Note:** The configuration namespace can be chosen freely, as long as it matches the dispatcher ID passed to `Akka.system.dispatchers.lookup`.

### Few specific thread pools

This is a combination between the many specific thread pools and the highly synchronized profile.  You would do most simple IO in the default execution context and set the number of threads there to be reasonably high (say 100), but then dispatch certain expensive operations to specific contexts, where you can limit the number of them that are done at one time.
