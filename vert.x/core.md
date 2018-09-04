# Core

---
## In the beginning there was Vert.x

### Are you fluent?
You may have noticed that in the previous examples a fluent API was used.

A fluent API is where multiple methods calls can be chained together.

Chaining calls like this allows you to write code that’s a little bit less verbose.

### Don’t call us, we’ll call you.

The Vert.x APIs are largely **event driven**.
This means that when things happen in Vert.x that you are interested in, Vert.x will call you by sending you events.

Some example events are:

- a timer has fired

- some data has arrived on a socket,

- some data has been read from disk

- an exception has occurred

- an HTTP server has received a request

```Java
server.requestHandler(request -> {
  // This handler will be called every time an HTTP request is received at the server
  request.response().end("hello world!");
});
```

Some time later when Vert.x has an event to pass to your handler Vert.x will call it asynchronously.

---

**This leads us to some important concepts in Vert.x:**

---
#### 1. Don’t block me!
none of the APIs in Vert.x block the calling thread.

If a result can be provided immediately, it will be returned immediately, otherwise you will usually provide a handler to receive events some time later.

Because none of the Vert.x APIs block threads that means you can use Vert.x to handle a lot of concurrency using just a small number of threads.

With a conventional blocking API the calling thread might block when:

- Reading data from a socket

- Writing data to disk

- Sending a message to a recipient and waiting for a reply.

- …​ Many other situations

This means that if you want a lot of concurrency using blocking APIs then you need a lot of threads to **prevent your application grinding to a halt**.


#### 2. Reactor and Multi-Reactor
Vert.x passes events to handlers when they are available.

In most cases Vert.x calls your handlers using a thread called an **event loop**.

Because nothing blocks, an event loop can potentially deliver huge amounts of events in a short amount of time. For example a single event loop can handle many thousands of HTTP requests very quickly.

In a standard reactor implementation there is a single event loop thread which runs around in a loop delivering all events to all handlers as they arrive.

The trouble with a single thread is it can only run on a single core at any one time, so if you want your single threaded reactor application (e.g. your Node.js application) to scale over your multi-core server you have to start up and manage many different processes.

Vert.x works differently here. Instead of a single event loop, each Vertx instance maintains several event loops. By default we choose the number based on the number of available cores on the machine, but this can be overridden.

#### 3. The Golden Rule - Don’t Block the Event Loop

Examples of blocking include:

- Thread.sleep()

- Waiting on a lock

- Waiting on a mutex or monitor (e.g. synchronized section)

- Doing a long lived database operation and waiting for a result

- Doing a complex calculation that takes some significant time.

- Spinning in a loop


If your application is not responsive it might be a sign that you are blocking an event loop somewhere. To help you diagnose such issues, Vert.x will automatically log warnings if it detects an event loop hasn’t returned for some time. If you see warnings like these in your logs, then you should investigate.

#### 4. Running blocking code

As discussed before, you can’t call blocking operations directly from an event loop, as that would prevent it from doing any other useful work. So how can you do this?

**calling executeBlocking**
It’s done by calling executeBlocking specifying both the blocking code to execute and a result handler to be called back asynchronous when the blocking code has been executed.

```Java
vertx.executeBlocking(future -> {
  // Call some blocking API that takes a significant amount of time to return
  String result = someAPI.blockingMethod("hello");
  future.complete(result);
}, res -> {
  System.out.println("The result is: " + res.result());
});
```

**use a worker verticle**

An alternative way to run blocking code is to use a worker verticle

A worker verticle is always executed with a thread from the worker pool.

By default blocking code is executed on the Vert.x worker pool, configured with setWorkerPoolSize.

Additional pools can be created for different purposes:

```Java
WorkerExecutor executor = vertx.createSharedWorkerExecutor("my-worker-pool");
executor.executeBlocking(future -> {
  // Call some blocking API that takes a significant amount of time to return
  String result = someAPI.blockingMethod("hello");
  future.complete(result);
}, res -> {
  System.out.println("The result is: " + res.result());
});
```

The worker executor must be closed when it’s not necessary anymore:
```Java
executor.close();
```

When several workers are created with the same name, they will share the same pool. The worker pool is destroyed when all the worker executors using it are closed.

#### 5. Async coordination

Coordination of multiple asynchronous results can be achieved with Vert.x `futures`

 It supports `concurrent composition` (run several async operations in parallel) and `sequential composition` (chain async operations).

**concurrent composition**
`CompositeFuture.all` takes several futures arguments (up to 6) and returns a future that is succeeded when all the futures are and failed when at least one of the futures is failed:

```Java
Future<HttpServer> httpServerFuture = Future.future();
httpServer.listen(httpServerFuture.completer());

Future<NetServer> netServerFuture = Future.future();
netServer.listen(netServerFuture.completer());

CompositeFuture.all(httpServerFuture, netServerFuture).setHandler(ar -> {
  if (ar.succeeded()) {
    // All servers started
  } else {
    // At least one server failed
  }
});
```

`CompositeFuture.any`:While the all composition waits until all futures are successful (or one fails), the any composition waits for the first succeeded future. CompositeFuture.any takes several futures arguments (up to 6) and returns a future that is succeeded when one of the futures is, and failed when all the futures are failed:

```Java
CompositeFuture.any(future1, future2).setHandler(ar -> {
  if (ar.succeeded()) {
    // At least one is succeeded
  } else {
    // All failed
  }
});
```

`CompositeFuture.join`:The join composition waits until all futures are completed, either with a success or a failure. CompositeFuture.join takes several futures arguments (up to 6) and returns a future that is succeeded when all the futures are succeeded, and failed when all the futures are completed and at least one of them is failed.

```Java
CompositeFuture.join(future1, future2, future3).setHandler(ar -> {
  if (ar.succeeded()) {
    // All succeeded
  } else {
    // All completed and at least one failed
  }
});
```
**Sequential composition**

```Java
FileSystem fs = vertx.fileSystem();

Future<Void> startFuture = Future.future();

Future<Void> fut1 = Future.future();
fs.createFile("/foo", fut1.completer());

fut1.compose(v -> {
  // When the file is created (fut1), execute this:
  Future<Void> fut2 = Future.future();
  fs.writeFile("/foo", Buffer.buffer(), fut2.completer());
  return fut2;
}).compose(v -> {
          // When the file is written (fut2), execute this:
          fs.move("/foo", "/bar", startFuture.completer());
        },
        // mark startFuture it as failed if any step fails.
        startFuture);
```
In this example, 3 operations are chained:

1. a file is created (fut1)

2. something is written in the file (fut2)

3. the file is moved (startFuture)

When these 3 steps are successful, the final future (startFuture) is succeeded. However, if one of the steps fails, the final future is failed.

---

## Verticles

### Asynchronous Verticle start and stop

Sometimes you want to do something in your verticle start-up which takes some time and you don’t want the verticle to be considered deployed until that happens. For example you might want to start an HTTP server in the start method and propagate the asynchronous result of the server listen method.

The way to do it is to implement the asynchronous start method. This version of the method takes a Future as a parameter. When the method returns the verticle will not be considered deployed.

```Java
public class MyVerticle extends AbstractVerticle {

  private HttpServeer server;

  public void start(Future<Void> startFuture) {
    server = vertx.createHttpServer().requestHandler(req -> {
      req.response()
        .putHeader("content-type", "text/plain")
        .end("Hello from Vert.x!");
      });

    // Now bind the server:
    server.listen(8080, res -> {
      if (res.succeeded()) {
        startFuture.complete();
      } else {
        startFuture.fail(res.cause());
      }
    });
  }
}
```

### Verticle Types
There are three different types of verticles:

- **Standard Verticles**
  These are the most common and useful type - they are always executed using an event loop thread. We’ll discuss this more in the next section.

- **Worker Verticles**
  These run using a thread from the worker pool. An instance is never executed concurrently by more than one thread.

- **Multi-threaded worker verticles**
  These run using a thread from the worker pool. An instance can be executed concurrently by more than one thread.

#### Standard Verticles
Standard verticles are assigned an event loop thread when they are created and the start method is called with that event loop. When you call any other methods that takes a handler on a core API from an event loop then Vert.x will guarantee that those handlers, when called, will be executed on the same event loop.

This means we can guarantee that all the code in your verticle instance is always executed on the same event loop (as long as you don’t create your own threads and call it!).

This means you can write all the code in your application as single threaded and let Vert.x worry about the threading and scaling. No more worrying about synchronized and volatile any more, and you also avoid many other cases of race conditions and deadlock so prevalent when doing hand-rolled 'traditional' multi-threaded application development.

#### Worker verticles
A worker verticle is just like a standard verticle but it’s executed using a thread from the Vert.x worker thread pool, rather than using an event loop.

Worker verticles are designed for calling blocking code, as they won’t block any event loops.

#### Multi-threaded worker verticles

A multi-threaded worker verticle is just like a normal worker verticle but it can be executed concurrently by different threads.


## The Event Bus
The event bus is the nervous system of Vert.x.

There is a single event bus instance for every Vert.x instance and it is obtained using the method `eventBus`.

The event bus allows different parts of your application to communicate with each other irrespective of what language they are written in, and whether they’re in the same Vert.x instance, or in a different Vert.x instance.

**Addressing**

Messages are sent on the event bus to an address.

Vert.x doesn’t bother with any fancy addressing schemes. In Vert.x an address is simply a string. Any string is valid. However it is wise to use some kind of scheme.

**Handlers**

Messages are received in handlers. You register a handler at an address.

- Many different handlers can be registered at the same address.

- A single handler can be registered at many different addresses.

**Publish/subscribe messaging**

The event bus supports publishing messages.

Messages are published to an address. Publishing means delivering the message to all handlers that are registered at that address.

This is the familiar publish/subscribe messaging pattern.

**Point to point and Request-Response messaging**

The event bus also supports point to point messaging.

Messages are sent to an address. Vert.x will then route it to just one of the handlers registered at that address.

If there is more than one handler registered at the address, one will be chosen using a non-strict round-robin algorithm.
















```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```












































































































































































































































--
