# EventLoopGroup和EventLoop

一个 Netty 程序启动时, 至少要指定一个 EventLoopGroup(如果使用到的是 NIO, 那么通常是 NioEventLoopGroup), 那么这个 NioEventLoopGroup 在 Netty 中到底扮演着什么角色


## Reactor线程模型

- 单线程模型
- 多线程模型
- 主从多线程模型

### 单线程模型

所谓单线程, 即 acceptor 处理和 handler 处理都在一个线程中处理.
理论上由于所有操作都是异步的，所以这个模型已经能够适应一些相对较小的应用．

 这个模型的坏处显而易见: 当其中某个 handler 阻塞时, 会导致其他所有的 client 的 handler 都得不到执行, 并且更严重的是, handler 的阻塞也会导致整个服务不能接收新的 client 请求(因为 acceptor 也被阻塞了). 因为有这么多的缺陷, 因此单线程Reactor 模型用的比较少.

![单线程模型][1]

### 多线程模型
 Reactor 的多线程模型与单线程模型的区别就是 acceptor 是一个单独的线程处理, 并且有一组特定的 NIO 线程来负责各个客户端连接的 IO 操作.

 Reactor 多线程模型 有如下特点:

 - 有专门一个线程, 即 Acceptor 线程用于监听客户端的TCP连接请求.
 - 客户端连接的 IO 操作都是由一个特定的 NIO 线程池负责. 每个客户端连接都与一个特定的 NIO 线程绑定, 因此在这个客户端连接中的所有 IO 操作都是在同一个线程中完成的.
 - 客户端连接有很多, 但是 NIO 线程数是比较少的, 因此一个 NIO 线程可以同时绑定到多个客户端连接中.

![多线程模型][2]

一般情况下, Reactor 的多线程模式已经可以很好的工作了, 但是我们考虑一下如下情况: 如果我们的服务器需要同时处理大量的客户端连接请求或我们需要在客户端连接时, 进行一些权限的检查, 那么单线程的 Acceptor 很有可能就处理不过来, 造成了大量的客户端不能连接到服务器. Reactor 的主从多线程模型就是在这样的情况下提出来的, 它的特点是: 服务器端接收客户端的连接请求不再是一个线程, 而是由一个独立的线程池组成.

![主从多线程模型][3]

可以看到, Reactor 的主从多线程模型和 Reactor 多线程模型很类似, 只不过 Reactor 的主从多线程模型的 acceptor 使用了线程池来处理大量的客户端请求.

## EventLoopGroup

首先看一下类结构：
![][4]

NioEventLoop 有几个重载的构造器, 不过内容都没有什么大的区别, 最终都是调用的父类MultithreadEventLoopGroup构造器。

在类MultithreadEventLoopGroup的初始化过程中， 我们可以发现如果没有指定thread数量， 就会默认指定一个数量．
```java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}

```

这个类是比较核心的初始化过程，
```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }
```
初始化过程主要涉及以下几个步骤:
- 初始化一个Executors， 这里初始化的是一个很简单的执行器， 这个执行器会为每一个task创建一个线程去执行．
- 创建一个大小为 nThreads 的 SingleThreadEventExecutor 数组
- 调用 newChhild 方法初始化 children 数组.

根据上面的代码, 我们知道, MultithreadEventExecutorGroup 内部维护了一个 EventExecutor 数组, Netty 的 EventLoopGroup 的实现机制其实就建立在 MultithreadEventExecutorGroup 之上. 每当 Netty 需要一个 EventLoop 时, 会调用 next() 方法获取一个可用的 EventLoop.
上面代码的最后一部分是 newChild 方法, 这个是一个抽象方法, 它的任务是实例化 EventLoop 对象. 我们跟踪一下它的代码, 可以发现, 这个方法在 NioEventLoopGroup 类中实现了, 其内容很简单

```java
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0]);
}
```
其实就是实例化一个 NioEventLoop 对象, 然后返回它.

最后总结一下整个 EventLoopGroup 的初始化过程

1. EventLoopGroup(其实是MultithreadEventExecutorGroup) 内部维护一个类型为 EventExecutor children 数组, 其大小是 nThreads, 这样就构成了一个线程池

2. 如果我们在实例化 NioEventLoopGroup 时, 如果指定线程池大小, 则 nThreads 就是指定的值, 反之是处理器核心数 * 2

3. MultithreadEventExecutorGroup 中会调用 newChild 抽象方法来初始化 children 数组, 抽象方法 newChild 是在 NioEventLoopGroup 中实现的, 它返回一个 NioEventLoop 实例.**NioEventLoop 属性:**
  - SelectorProvider provider 属性: NioEventLoopGroup 构造器中通过 SelectorProvider.provider() 获取一个 SelectorProvider
  - Selector selector 属性: NioEventLoop 构造器中通过调用通过 selector = provider.openSelector() 获取一个 selector 对象.

## EventLoop

NioEventLoop 继承于 SingleThreadEventLoop, 而 SingleThreadEventLoop 又继承于 SingleThreadEventExecutor. SingleThreadEventExecutor 是 Netty 中对本地线程的抽象, 它内部有一个 Thread thread 属性, 存储了一个本地 Java 线程. 因此我们可以认为, 一个 NioEventLoop 其实和一个特定的线程绑定, 并且在其生命周期内, 绑定的线程都不会再改变.

**NioEventLoop 类层次结构**
![][5]

在 AbstractScheduledEventExecutor 中, Netty 实现了 NioEventLoop 的 schedule 功能, 即我们可以通过调用一个 NioEventLoop 实例的 schedule 方法来运行一些定时任务. 而在 SingleThreadEventLoop 中, 又实现了任务队列的功能, 通过它, 我们可以调用一个 NioEventLoop 实例的 execute 方法来向任务队列中添加一个 task, 并由 NioEventLoop 进行调度执行.

通常来说, NioEventLoop 肩负着两种任务, 第一个是作为 IO 线程, 执行与 Channel 相关的 IO 操作, 包括 调用 select 等待就绪的 IO 事件、读写数据与数据的处理等; 而第二个任务是作为任务队列, 执行 taskQueue 中的任务, 例如用户调用 eventLoop.schedule 提交的定时任务也是这个线程执行的.

### NioEventLoop 的实例化过程

![][6]

NioEventLoop的构造器同样是存在继承关系的， NioEventLoop只有一个构造器
```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider) {
    super(parent, executor, false);
    if (selectorProvider == null) {
        throw new NullPointerException("selectorProvider");
    }
    provider = selectorProvider;
    selector = openSelector();
}
```
在向上初始化的过程中， SingleThreadEventLoop类会初始化一个DefaultChannelHandlerInvoker类型的字段invoker， 并且把NioEventLoop对象传进去，
这个invoker内部只有一个对象，就是NioEventLoop， 其内部所有的方法都是在操作NioEventLoop。

```java
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {
    private final ChannelHandlerInvoker invoker = new DefaultChannelHandlerInvoker(this);
}
```

```java
public DefaultChannelHandlerInvoker(EventExecutor executor) {
    if (executor == null) {
        throw new NullPointerException("executor");
    }

    this.executor = executor;
}
```
我们再回到NioEventLoop初始化的过程中去

NioEventLoop 会在SingleThreadEventExecutor类中真正的赋值。
```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp) {
    super(parent);

    if (executor == null) {
        throw new NullPointerException("executor");
    }

    this.addTaskWakesUp = addTaskWakesUp;
    this.executor = executor;

    taskQueue = newTaskQueue();
}
```
这里面主要就是添加了一个taskQueue，在方法  newTaskQueue() 就是初始化了一个LinkedBlockingQueue队列．

我们仔细观察一下SingleThreadEventExecutor类，这个类里面维护了一个thread， 或者说每一个SingleThreadEventExecutor都绑定了一个thread。
在其整个生命周期内，都不会更换thread。



[1]:https://github.com/yongshun/learn_netty_source_code/blob/master/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop/Reactor%20%E5%8D%95%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B.png

[2]: https://github.com/yongshun/learn_netty_source_code/blob/master/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop/Reactor%20%E5%A4%9A%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B.png


[3]: https://github.com/yongshun/learn_netty_source_code/blob/master/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop/Reactor%20%E4%B8%BB%E4%BB%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B.png

[4]: https://github.com/yongshun/learn_netty_source_code/raw/master/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop/NioEventLoopGroup.png

[5]: https://github.com/yongshun/learn_netty_source_code/raw/master/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop/NioEventLoop.png

[6]: https://github.com/yongshun/learn_netty_source_code/blob/master/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop/NioEventLoop%20%E5%AE%9E%E4%BE%8B%E5%8C%96%E9%A1%BA%E5%BA%8F%E5%9B%BE.png
