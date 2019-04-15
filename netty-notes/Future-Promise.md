# Future 和 Promise

## Future
uture顾名思义，是一个未来完成的异步操作，可以获得未来返回的值。

### Java Future
Java 中的Future 是一个接口， 内部有这些方法:
```java
boolean	cancel(boolean mayInterruptIfRunning) 试图取消对此任务的执行。
V	get() 如有必要，等待计算完成，然后获取其结果。
V	get(long timeout, TimeUnit unit) 如有必要，最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）。
boolean	isCancelled() 如果在任务正常完成前将其取消，则返回 true。
boolean	isDone() 如果任务已完成，则返回 true。
```

### Netty Future
Netty 的Future在concurrent Future的基础上，增加了更多的功能，在Java的Future中，主要是任务的运行/取消，而Netty的Future增加了更多的功能。

```java
public interface Future<V> extends java.util.concurrent.Future<V>{
  boolean isSuccess(); 只有IO操作完成时才返回true
  boolean isCancellable(); 只有当cancel(boolean)成功取消时才返回true
  Throwable cause();  IO操作发生异常时，返回导致IO操作以此的原因，如果没有异常，返回null
  // 向Future添加事件，future完成时，会执行这些事件，如果add时future已经完成，会立即执行监听事件
  Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
  // 移除监听事件，future完成时，不会触发
  Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
  Future<V> sync() throws InterruptedException;  //等待future done
  Future<V> syncUninterruptibly();   // 等待future done，不可打断
  Future<V> await() throws InterruptedException; // 等待future完成
  Future<V> awaitUninterruptibly();  // 等待future 完成，不可打断
  V getNow(); // 立刻获得结果，如果没有完成，返回null
  boolean cancel(boolean mayInterruptIfRunning); // 如果成功取消，future会失败，导致CancellationException}
}

```

Netty为Future加入的功能主要是添加/删除监听事件，其他的方法是为get()方法服务的，get()方法可以通过调用await/getNow等方法实现。

## Pormise
Netty的Future与Java自带到Future略有不同，其引入了Promise机制。在Java的Future中，业务逻辑为一个Callable或Runnable实现类，该类的call()或run()执行完毕意味着业务逻辑的完结；而在Promise机制中，可以在业务逻辑中人工设置业务逻辑的成功与失败。

```java
public interface Promise<V> extends Future<V> {
	// 设置future执行结果为成功
    Promise<V> setSuccess(V result);
   	// 尝试设置future执行结果为成功,返回是否设置成功
   boolean trySuccess(V result);
   // 设置失败
    Promise<V> setFailure(Throwable cause);
    boolean tryFailure(Throwable cause);
    // 设置为不能取消
    boolean setUncancellable();
    //一下省略了覆盖Future的一些方法
}
```

下面以一个例子来说明Promise的使用方法:

```java
// main 方法
NettyFuture4Promise test = new NettyFuture4Promise();
Promise<String> promise = test.search("Netty In Action");
String result = promise.get();
System.out.println("price is " + result);

//
private Promise<String> search(String prod) {
		NioEventLoopGroup loop = new NioEventLoopGroup();
        // 创建一个DefaultPromise并返回
		DefaultPromise<String> promise = new DefaultPromise<String>(loop.next());
		loop.schedule(new Runnable() {
			@Override
			public void run() {
				try {
					System.out.println(String.format("	>>search price of %s from internet!",prod));
					Thread.sleep(5000);
					promise.setSuccess("$99.99");// 等待5S后设置future为成功，
                   // promise.setFailure(new NullPointerException()); //当然，也可以设置失败
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		},0,TimeUnit.SECONDS);

		return promise;
	}
```

可以看到，Promise能够在业务逻辑线程中通知Future成功或失败，由于Promise继承了Netty的Future，因此可以加入监听事件。

```java
// main方法中，查询结束后获取promise，加入两个监听事件，分别给小Hong发通知和Email
Promise<String> promise = test.search("Netty In Action");
promise.addListener(new GenericFutureListener<Future<? super String>>() {
	@Override
	public void operationComplete(Future<? super String> future) throws Exception {
		System.out.println("Listener 1, make a notifice to Hong,price is " + future.get());
	}

});

promise.addListener(new GenericFutureListener<Future<? super String>>() {
	@Override
	public void operationComplete(Future<? super String> future) throws Exception {
		System.out.println("Listener 2, send a email to Hong,price is " + future.get());
	}

});
```
Future和Promise的好处在于，获取到Promise对象后可以为其设置异步调用完成后的操作，然后立即继续去做其他任务。

## Netty常用的Promise类












































/
