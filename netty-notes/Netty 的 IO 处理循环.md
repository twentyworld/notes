
# Netty 的 IO 处理循环

> 文章引用自[github](https://github.com/yongshun/learn_netty_source_code/blob/master/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop/Netty%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20%E4%B8%89%20%E6%88%91%E5%B0%B1%E6%98%AF%E5%A4%A7%E5%90%8D%E9%BC%8E%E9%BC%8E%E7%9A%84%20EventLoop.md)

**在`Netty`中, 一个 `EventLoop` 需要负责两个工作, 第一个是作为 `IO` 线程, 负责相应的 `IO` 操作; 第二个是作为任务线程, 执行 `taskQueue` 中的任务.**  接下来我们先从 `IO` 操纵方面入手, 看一下 `TCP` 数据是如何从 `Java NIO Socket` 传递到我们的 `handler` 中的.

`Netty` 是 `Reactor` 模型的一个实现, 并且是基于 `Java NIO` 的, `Netty` 中必然有一个 `Selector` 线程, 用于不断调用 `Java NIO` 的 `Selector.select` 方法, 查询当前是否有就绪的 `IO` 事件. 回顾一下在 `Java NIO` 中所讲述的 `Selector` 的使用流程:
 1. 通过 `Selector.open()` 打开一个 `Selector`.
 2. 将 `Channel` 注册到 `Selector` 中, 并设置需要监听的事件(`interest set`)
 3. 不断重复:
  - 调用 `select()` 方法
  - 调用 `selector.selectedKeys()` 获取 `selected keys`
  - 迭代每个 `selected key`:
   - 1) 从 `selected key` 中获取 对应的 `Channel` 和附加信息(如果有的话)
   - 2) 判断是哪些 `IO` 事件已经就绪了, 然后处理它们. **如果是 `OP_ACCEPT` 事件, 则调用 "`SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept()`" 获取 `SocketChannel`, 并将它设置为 非阻塞的, 然后将这个 `Channel` 注册到 `Selector` 中.**
   - 3) 根据需要更改 `selected key` 的监听事件.
   - 4) 将已经处理过的 `key` 从 `selected keys` 集合中删除.


上面的使用流程用代码来体现就是:
```java
public class MultiplexerTimeServer implements Runnable {

    Logger logger = LoggerFactory.getLogger(MultiplexerTimeServer.class);
    private Selector selector;
    private ServerSocketChannel serverSocketChannel;
    private volatile boolean stop;

    public MultiplexerTimeServer(int port) {
        try {
            selector = Selector.open();

            //打开，用于监听客户端的连接，是所有客户端连接的父Channel
            serverSocketChannel = ServerSocketChannel.open();
            //设置为非阻塞
            serverSocketChannel.configureBlocking(false);
            //绑定监听端口
            serverSocketChannel.socket().bind(new InetSocketAddress(port), 1024);
            //将ServerSocketChannel注册到Selector上，监听Accept事件
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

            logger.info("port: " + port);

        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }

    public void stop() {
        this.stop = true;
    }

    @Override
    public void run() {
        //多路复用器在线程run方法中无线轮询准备就绪的key
        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                SelectionKey key;

                while (iterator.hasNext()) {
                    key = iterator.next();
                    iterator.remove();
                    try {
                        handleInput(key);

                    } catch (Exception e) {
                        if (key != null) {
                            key.cancel();
                            if (key.channel() != null) key.channel().close();
                        }
                        logger.info("wrong info.");
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey selectionKey) throws IOException {
        if (selectionKey.isValid()) {
            if (selectionKey.isAcceptable()) {
                logger.info("connect");
                //多路复用器监听到新的客户端的接入，处理请求
                ServerSocketChannel serverSocketChannel = (ServerSocketChannel) selectionKey.channel();
                SocketChannel socketChannel = serverSocketChannel.accept();
                socketChannel.configureBlocking(false);
                socketChannel.register(selector, SelectionKey.OP_READ);


                logger.info("isReadable: " + selectionKey.isReadable());
            }
        }

        if (selectionKey.isReadable()) {
            SocketChannel socketChannel1 = (SocketChannel) selectionKey.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            int readBuffer = socketChannel1.read(buffer);

            logger.info("read context");
            if (readBuffer > 0) {
                buffer.flip();
                byte[] bytes = new byte[buffer.remaining()];
                buffer.get(bytes);
                String context = new String(bytes, "UTF-8");
                logger.info(context);
                logger.info("current time: " + System.currentTimeMillis());
                write(socketChannel1, System.currentTimeMillis() + "");
            } else if (readBuffer < 0) {
                selectionKey.cancel();
                socketChannel1.close();
            }
        }

    }


    private void write(SocketChannel socketChannel, String response) throws IOException {
        ByteBuffer buffer = ByteBuffer.allocate(response.length());
        buffer.put(response.getBytes());
        buffer.flip();
        socketChannel.write(buffer);

    }
}
```

还记得不, 上面操作的第一步 **通过 Selector.open() 打开一个 Selector** 我们已经在第一章的 **Channel 实例化** 这一小节中已经提到了, Netty 中是通过调用 SelectorProvider.openSocketChannel() 来打开一个新的 Java NIO SocketChannel:
```java
private static SocketChannel newSocket(SelectorProvider provider) {
    ...
    return provider.openSocketChannel();
}
```
第二步 **将 Channel 注册到 Selector 中, 并设置需要监听的事件(interest set)** 的操作我们在 **channel 的注册过程** 中也分析过了, 我们在来回顾一下, 在客户端的 Channel 注册过程中, 会有如下调用链:

``` java
Bootstrap.initAndRegister ->
	AbstractBootstrap.initAndRegister ->
		MultithreadEventLoopGroup.register ->
			SingleThreadEventLoop.register ->
				AbstractUnsafe.register ->
					AbstractUnsafe.register0 ->
						AbstractNioChannel.doRegister
```

在 AbstractUnsafe.register 方法中调用了 register0 方法:

```java
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
	// 省略条件判断和错误处理
    AbstractChannel.this.eventLoop = eventLoop;
    register0(promise);
}
```

register0 方法代码如下:

```java
private void register0(ChannelPromise promise) {
    boolean firstRegistration = neverRegistered;
    doRegister();
    neverRegistered = false;
    registered = true;
    safeSetSuccess(promise);
    pipeline.fireChannelRegistered();
    // Only fire a channelActive if the channel has never been registered. This prevents firing
    // multiple channel actives if the channel is deregistered and re-registered.
    if (firstRegistration && isActive()) {
        pipeline.fireChannelActive();
    }
}
```
register0 又调用了 AbstractNioChannel.doRegister:

```java
@Override
protected void doRegister() throws Exception {
	// 省略错误处理
    selectionKey = javaChannel().register(eventLoop().selector, 0, this);
}
```

在这里 javaChannel() 返回的是一个  Java NIO SocketChannel 对象, 我们将此 SocketChannel 注册到前面第一步获取的 Selector 中.

那么接下来的第三步的循环是在哪里实现的呢? 第三步的操作就是分析的关键, 下面一步一步展示出来.

### thread 的 run 循环

在 **EventLoop 的启动** 一小节中, 我们已经了解到了, 当 EventLoop.execute **第一次被调用**时, 就会触发 **startThread()** 的调用, 进而导致了 EventLoop 所对应的 Java 线程的启动. 接着更深入一些, 来看一下此线程启动后都会做什么

```java
private void startThread() {
    synchronized (stateLock) {
        if (state == ST_NOT_STARTED) {
            state = ST_STARTED;
            delayedTaskQueue.add(new ScheduledFutureTask<Void>(
                    this, delayedTaskQueue, Executors.<Void>callable(new PurgeTask(), null),
                    ScheduledFutureTask.deadlineNanos(SCHEDULE_PURGE_INTERVAL), -SCHEDULE_PURGE_INTERVAL));
            doStartThread();
        }
    }
}
```

下面是此线程的 doStartThread() 方法, 已经把一些异常处理和收尾工作的代码都去掉了. 这个 doStartThread 方法可以说是十分简单, 主要就是调用了 **SingleThreadEventExecutor.this.run()** 方法.

```java
private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
                success = true;
              }
        }
    });
}

```

而 SingleThreadEventExecutor.run() 是一个抽象方法, 它的实现在 NioEventLoop 中.
继续跟踪到 NioEventLoop.run() 方法, 其源码如下:

```java
@Override
protected void run() {
    for (;;) {
        boolean oldWakenUp = wakenUp.getAndSet(false);
        try {
            if (hasTasks()) {
                selectNow();
            } else {
                select(oldWakenUp);
                if (wakenUp.get()) {
                    selector.wakeup();
                }
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                processSelectedKeys();
                runAllTasks();
            } else {
                final long ioStartTime = System.nanoTime();

                processSelectedKeys();

                final long ioTime = System.nanoTime() - ioStartTime;
                runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
            }

            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    break;
                }
            }
        } catch (Throwable t) {
            ...
        }
    }
}
```

看到了上面代码的 **for( ; ; )** 所构成的死循环了? NioEventLoop 事件循环的核心就是这里!
现在把上面所提到的 Selector 使用步骤的第三步的部分也找到了.
这个 run 方法可以说是 Netty NIO 的核心, 属于重中之重, 把它分析明白了, 那么对 Netty 的事件循环机制也就了解了大部分了.

### IO 事件的轮询
首先, 在 `run` 方法中, 第一步是调用 **hasTasks()** 方法来判断当前任务队列中是否有任务:

```java
protected boolean hasTasks() {
    assert inEventLoop();
    return !taskQueue.isEmpty();
}
```

这个方法很简单, 仅仅是检查了一下 **taskQueue** 是否为空. 至于 taskQueue 是什么呢, 其实它就是存放一系列的需要由此 EventLoop 所执行的任务列表. 关于 taskQueue, 我们这里暂时不表, 等到后面再来详细分析它.
当 taskQueue 不为空时, 就执行到了 if 分支中的 selectNow() 方法. 然而当 taskQueue 为空时, 执行的是 select(oldWakenUp) 方法. 那么 **selectNow()** 和 **select(oldWakenUp)** 之间有什么区别呢? 来看一下, selectNow() 的源码如下:

```java
void selectNow() throws IOException {
    try {
        selector.selectNow();
    } finally {
        // restore wakup state if needed
        if (wakenUp.get()) {
            selector.wakeup();
        }
    }
}
```
首先调用了 **selector.selectNow()** 方法, 这里 selector 是什么大家还有印象不? 我们在第一章 **Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头 (客户端)** 时对它有过介绍, 这个 **selector** 字段正是 Java NIO 中的多路复用器 **Selector**. 那么这里 **selector.selectNow()** 就很好理解了, selectNow() 方法会检查当前是否有就绪的 IO 事件, 如果有, 则返回就绪 IO 事件的个数; 如果没有, 则返回0. `注意, selectNow() 是立即返回的, 不会阻塞当前线程.` 当 selectNow() 调用后, finally 语句块中会检查 wakenUp 变量是否为 true, 当为 true 时, 调用 selector.wakeup() 唤醒 select() 的阻塞调用.


看了 if 分支的 selectNow 方法后, 我们再来看一下 else 分支的 **select(oldWakenUp)** 方法.
其实 else 分支的 **select(oldWakenUp)** 方法的处理逻辑比较复杂, 而我们这里的目的暂时不是分析这个方法调用的具体工作, 因此我这里长话短说, 只列出我们我们关注的内如:

```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        ...
        int selectedKeys = selector.select(timeoutMillis);
        ...
    } catch (CancelledKeyException e) {
        ...
    }
}
```

在这个 select 方法中, 调用了 **selector.select(timeoutMillis)**, 而这个调用是会阻塞住当前线程的, timeoutMillis 是阻塞的超时时间.
到来这里, 我们可以看到, 当 **hasTasks()** 为真时, 调用的的 **selectNow()** 方法是不会阻塞当前线程的, 而当  **hasTasks()** 为假时, 调用的 **select(oldWakenUp)** 是会阻塞当前线程的.
这其实也很好理解: 当 taskQueue 中没有任务时, 那么 Netty 可以阻塞地等待 IO 就绪事件; 而当 taskQueue 中有任务时, 我们自然地希望所提交的任务可以尽快地执行, 因此 Netty 会调用非阻塞的 selectNow() 方法, 以保证 taskQueue 中的任务尽快可以执行.

### IO 事件的处理
在 NioEventLoop.run() 方法中, 第一步是通过 select/selectNow 调用查询当前是否有就绪的 IO 事件. 那么当有 IO 事件就绪时, 第二步自然就是处理这些 IO 事件啦.
首先让我们来看一下 NioEventLoop.run 中循环的剩余部分:

```java
final int ioRatio = this.ioRatio;
if (ioRatio == 100) {
    processSelectedKeys();
    runAllTasks();
} else {
    final long ioStartTime = System.nanoTime();

    processSelectedKeys();

    final long ioTime = System.nanoTime() - ioStartTime;
    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
}
```
上面列出的代码中, 有两个关键的调用, 第一个是 **processSelectedKeys()** 调用, 根据字面意思, 我们可以猜出这个方法肯定是查询就绪的 IO 事件, 然后处理它; 第二个调用是 **runAllTasks()**, 这个方法我们也可以一眼就看出来它的功能就是运行 taskQueue 中的任务.
这里的代码还有一个十分有意思的地方, 即 **ioRatio**. 那什么是 **ioRatio** 呢? 它表示的是此线程分配给 IO 操作所占的时间比(即运行 processSelectedKeys 耗时在整个循环中所占用的时间). 例如 ioRatio 默认是 50, 则表示 IO 操作和执行 task 的所占用的线程执行时间比是 1 : 1. 当知道了 IO 操作耗时和它所占用的时间比, 那么执行 task 的时间就可以很方便的计算出来了:

```
//设 IO 操作耗时为 ioTime, ioTime 占的时间比例为 ioRatio, 则:
	ioTime / ioRatio = taskTime / taskRatio
	taskRatio = 100 - ioRatio
	=> taskTime = ioTime * (100 - ioRatio) / ioRatio
```

根据上面的公式, 当我们设置 ioRate = 70 时, 则表示 IO 运行耗时占比为70%, 即假设某次循环一共耗时为 100ms, 那么根据公式, 我们知道 **processSelectedKeys()** 方法调用所耗时大概为70ms(即 IO 耗时), 而 **runAllTasks()** 耗时大概为 30ms(即执行 task 耗时).
当 ioRatio 为 100 时, Netty 就不考虑 IO 耗时的占比, 而是分别调用 **processSelectedKeys()**、**runAllTasks()**; 而当 ioRatio 不为 100时, 则执行到 else 分支, 在这个分支中, 首先记录下 **processSelectedKeys()** 所执行的时间(即 IO 操作的耗时), 然后根据公式, 计算出执行 task 所占用的时间, 然后以此为参数, 调用 **runAllTasks()**.

我们这里先分析一下 **processSelectedKeys()** 方法调用, **runAllTasks()** 我们留到下一节再分析.
**processSelectedKeys()** 方法的源码如下:

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(selectedKeys.flip());
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```
这个方法中, 会根据 **selectedKeys** 字段是否为空, 而分别调用 **processSelectedKeysOptimized** 或 **processSelectedKeysPlain**. **selectedKeys** 字段是在调用 openSelector() 方法时, 根据 JVM 平台的不同, 而有设置不同的值, 在我所调试这个值是不为 null 的. 其实 **processSelectedKeysOptimized** 方法 **processSelectedKeysPlain** 没有太大的区别, 为了简单起见, 我们以 **processSelectedKeysOptimized** 为例分析一下源码的工作流程吧.
```
private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
    for (int i = 0;; i ++) {
        final SelectionKey k = selectedKeys[i];
        if (k == null) {
            break;
        }
        selectedKeys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
        ...
    }
}
```
其实你别看它代码挺多的, 但是关键的点就两个: 迭代 **selectedKeys** 获取就绪的 IO 事件, 然后为每个事件都调用 **processSelectedKey** 来处理它.
这里正好完美对应上了我们提到的 Selector 的使用流程中的第三步里操作.
还有一点需要注意的是, 我们可以调用 **selectionKey.attach(object)** 给一个 selectionKey 设置一个附加的字段, 然后可以通过 **Object attachedObj = selectionKey.attachment()** 获取它. 上面代代码正是通过了 **k.attachment()** 来获取一个附加在 selectionKey 中的对象, 那么这个对象是什么呢? 它又是在哪里设置的呢? 我们再来回忆一下 SocketChannel 是如何注册到 Selector 中的:
在客户端的 Channel 注册过程中, 会有如下调用链:
```
Bootstrap.initAndRegister ->
	AbstractBootstrap.initAndRegister ->
		MultithreadEventLoopGroup.register ->
			SingleThreadEventLoop.register ->
				AbstractUnsafe.register ->
					AbstractUnsafe.register0 ->
						AbstractNioChannel.doRegister
```
最后的 AbstractNioChannel.doRegister 方法会调用 **SocketChannel.register** 方法注册一个 SocketChannel 到指定的 Selector:
```
@Override
protected void doRegister() throws Exception {
	// 省略错误处理
    selectionKey = javaChannel().register(eventLoop().selector, 0, this);
}
```
特别注意一下 **register** 的第三个参数, 这个参数是设置 selectionKey 的附加对象的, 和调用 **selectionKey.attach(object)** 的效果一样. 而调用 **register** 所传递的第三个参数是 **this**, 它其实就是一个 `NioSocketChannel` 的实例. 那么这里就很清楚了, 我们在将 SocketChannel 注册到 Selector 中时, 将 SocketChannel 所对应的 NioSocketChannel 以附加字段的方式添加到了selectionKey 中.
再回到 **processSelectedKeysOptimized** 方法中, 当我们获取到附加的对象后, 我们就调用 **processSelectedKey** 来处理这个 IO 事件:
```
final Object a = k.attachment();

if (a instanceof AbstractNioChannel) {
    processSelectedKey(k, (AbstractNioChannel) a);
} else {
    @SuppressWarnings("unchecked")
    NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
    processSelectedKey(k, task);
}
```
**processSelectedKey** 方法源码如下:
```
private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final NioUnsafe unsafe = ch.unsafe();
    ...
    try {
        int readyOps = k.readyOps();

        // 可读事件
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
            if (!ch.isOpen()) {
                // Connection already closed - no need to handle write.
                return;
            }
        }

		// 可写事件
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // 连接建立事件
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```
这个代码是不是很熟悉啊? 完全是 Java NIO 的 Selector 的那一套处理流程嘛!
**processSelectedKey** 中处理了三个事件, 分别是:
 - OP_READ, 可读事件, 即 Channel 中收到了新数据可供上层读取.
 - OP_WRITE, 可写事件, 即上层可以向 Channel 写入数据.
 - OP_CONNECT, 连接建立事件, 即 TCP 连接已经建立, Channel 处于 active 状态.

下面我们分别根据这三个事件来看一下 Netty 是怎么处理的吧.
#### OP_READ 处理
当就绪的 IO 事件是 **OP_READ**, 代码会调用 **unsafe.read()** 方法, 即:
```
// 可读事件
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
    if (!ch.isOpen()) {
        // Connection already closed - no need to handle write.
        return;
    }
}
```
unsafe 这个字段, 我们已经和它打了太多的交道了, 在第一章 **Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头 (客户端)** 中我们已经对它进行过浓墨重彩地分析了, 最后我们确定了它是一个 **NioSocketChannelUnsafe** 实例, 负责的是 Channel 的底层 IO 操作.
我们可以利用 Intellij IDEA 提供的 **Go To Implementations** 功能, 寻找到这个方法的实现. 最后我们发现这个方法没有在 **NioSocketChannelUnsafe** 中实现, 而是在它的父类 **AbstractNioByteChannel** 实现的, 它的实现源码如下:
```
@Override
public final void read() {
    ...
    ByteBuf byteBuf = null;
    int messages = 0;
    boolean close = false;
    try {
        int totalReadAmount = 0;
        boolean readPendingReset = false;
        do {
            byteBuf = allocHandle.allocate(allocator);
            int writable = byteBuf.writableBytes();
            int localReadAmount = doReadBytes(byteBuf);

            // 检查读取结果.
            ...

            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;

            ...

            totalReadAmount += localReadAmount;

            // 检查是否是配置了自动读取, 如果不是, 则立即退出循环.
            ...
        } while (++ messages < maxMessagesPerRead);

        pipeline.fireChannelReadComplete();
        allocHandle.record(totalReadAmount);

        if (close) {
            closeOnRead(pipeline);
            close = false;
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close);
    } finally {
    }
}
```
**read()** 源码比较长, 我为了篇幅起见, 删除了部分代码, 只留下了主干. 不过我建议读者朋友们自己一定要看一下 **read()** 源码, 这对理解  Netty 的 EventLoop 十分有帮助.
上面 **read** 方法其实归纳起来, 可以认为做了如下工作:
 1. 分配 ByteBuf
 2. 从 SocketChannel 中读取数据
 3. 调用 **pipeline.fireChannelRead** 发送一个 inbound 事件.

前面两点没什么好说的, 第三点 **pipeline.fireChannelRead** 读者朋友们看到了有没有会心一笑地感觉呢? 反正我看到这里时是有的. **pipeline.fireChannelRead** 正好就是我们在第二章 **Netty 源码分析之 二 贯穿Netty 的大动脉 ── ChannelPipeline (二)** 中分析的 **inbound** 事件起点. 当调用了 **pipeline.fireIN_EVT()** 后, 那么就产生了一个 **inbound** 事件, 此事件会以 **head -> customContext -> tail** 的方向依次流经 ChannelPipeline 中的各个 handler.
调用了 **pipeline.fireChannelRead** 后, 就是 ChannelPipeline 中所需要做的工作了, 这些我们已经在第二章中有过详细讨论, 这里就展开了.

#### OP_WRITE 处理
**OP_WRITE** 可写事件代码如下. 这里代码比较简单, 没有详细分析的必要了.
```
if ((readyOps & SelectionKey.OP_WRITE) != 0) {
    // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
    ch.unsafe().forceFlush();
}
```
#### OP_CONNECT 处理
最后一个事件是 **OP_CONNECT**, 即 TCP 连接已建立事件.
```
if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
    // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
    // See https://github.com/netty/netty/issues/924
    int ops = k.interestOps();
    ops &= ~SelectionKey.OP_CONNECT;
    k.interestOps(ops);

    unsafe.finishConnect();
}
```
**OP_CONNECT** 事件的处理中, 只做了两件事情:
 1. 正如代码中的注释所言, 我们需要将 **OP_CONNECT** 从就绪事件集中清除, 不然会一直有 **OP_CONNECT** 事件.
 2. 调用 unsafe.finishConnect() 通知上层连接已建立


unsafe.finishConnect() 调用最后会调用到 **pipeline().fireChannelActive()**, 产生一个 **inbound** 事件, 通知 pipeline 中的各个 handler TCP 通道已建立(即 **ChannelInboundHandler.channelActive** 方法会被调用)

到了这里, 我们整个 NioEventLoop 的 IO 操作部分已经了解完了, 接下来的一节我们要重点分析一下 **Netty 的任务队列机制**.
## Netty 的任务队列机制
我们已经提到过, 在Netty 中, 一个 NioEventLoop 通常需要肩负起两种任务, 第一个是作为 IO 线程, 处理 IO 操作; 第二个就是作为任务线程, 处理 taskQueue 中的任务. 这一节的重点就是分析一下 NioEventLoop 的任务队列机制的.
### Task 的添加
#### 普通 Runnable 任务
NioEventLoop 继承于 SingleThreadEventExecutor, 而 `SingleThreadEventExecutor` 中有一个 **Queue<Runnable> taskQueue** 字段, 用于存放添加的 Task. 在 Netty 中, 每个 Task 都使用一个实现了 Runnable 接口的实例来表示.
例如当我们需要将一个 Runnable 添加到 taskQueue 中时, 我们可以进行如下操作:
```
EventLoop eventLoop = channel.eventLoop();
eventLoop.execute(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello, Netty!");
    }
});
```
当调用 execute 后, 实际上是调用到了 SingleThreadEventExecutor.execute() 方法, 它的实现如下:
```
@Override
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();
        addTask(task);
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```
而添加任务的 **addTask** 方法的源码如下:
```
protected void addTask(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (isShutdown()) {
        reject();
    }
    taskQueue.add(task);
}
```
因此实际上, `taskQueue` 是存放着待执行的任务的队列.

#### schedule 任务
除了通过 execute 添加普通的 Runnable 任务外, 我们还可以通过调用 eventLoop.scheduleXXX 之类的方法来添加一个定时任务.
 EventLoop 中实现任务队列的功能在超类 `SingleThreadEventExecutor` 实现的, 而 schedule 功能的实现是在 `SingleThreadEventExecutor` 的父类, 即 `AbstractScheduledEventExecutor` 中实现的.
在 `AbstractScheduledEventExecutor` 中, 有以 scheduledTaskQueue 字段:
```
Queue<ScheduledFutureTask<?>> scheduledTaskQueue;
```

scheduledTaskQueue 是一个队列(Queue), 其中存放的元素是 **ScheduledFutureTask**. 而 **ScheduledFutureTask** 我们很容易猜到, 它是对 Schedule 任务的一个抽象.
我们来看一下 `AbstractScheduledEventExecutor`  所实现的 **schedule** 方法吧:
```
@Override
public  ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
    ObjectUtil.checkNotNull(command, "command");
    ObjectUtil.checkNotNull(unit, "unit");
    if (delay < 0) {
        throw new IllegalArgumentException(
                String.format("delay: %d (expected: >= 0)", delay));
    }
    return schedule(new ScheduledFutureTask<Void>(
            this, command, null, ScheduledFutureTask.deadlineNanos(unit.toNanos(delay))));
}
```
这是其中一个重载的 schedule, 当一个 Runnable 传递进来后, 会被封装为一个 **ScheduledFutureTask** 对象, 这个对象会记录下这个 Runnable 在何时运行、已何种频率运行等信息.
当构建了 **ScheduledFutureTask** 后, 会继续调用 另一个重载的 schedule 方法:
```
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    if (inEventLoop()) {
        scheduledTaskQueue().add(task);
    } else {
        execute(new OneTimeTask() {
            @Override
            public void run() {
                scheduledTaskQueue().add(task);
            }
        });
    }

    return task;
}
```
在这个方法中, ScheduledFutureTask 对象就会被添加到 **scheduledTaskQueue** 中了.
### 任务的执行
当一个任务被添加到 taskQueue 后, 它是怎么被 EventLoop 执行的呢?
让我们回到 NioEventLoop.run() 方法中, 在这个方法里, 会分别调用 **processSelectedKeys()** 和 **runAllTasks()** 方法, 来进行 IO 事件的处理和 task 的处理. **processSelectedKeys()** 方法我们已经分析过了, 下面我们来看一下 **runAllTasks()** 中到底有什么名堂吧.
runAllTasks 方法有两个重载的方法, 一个是无参数的, 另一个有一个参数的. 首先来看一下无参数的 runAllTasks:
```
protected boolean runAllTasks() {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        return false;
    }

    for (;;) {
        try {
            task.run();
        } catch (Throwable t) {
            logger.warn("A task raised an exception.", t);
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            return true;
        }
    }
}
```
我们前面已经提到过, EventLoop 可以通过调用 **EventLoop.execute** 来将一个 Runnable 提交到 taskQueue 中, 也可以通过调用 **EventLoop.schedule** 来提交一个 schedule 任务到 **scheduledTaskQueue** 中. 在此方法的一开始调用的 **fetchFromScheduledTaskQueue()** 其实就是将 **scheduledTaskQueue** 中已经可以执行的(即定时时间已到的 schedule 任务) 拿出来并添加到 taskQueue 中, 作为可执行的 task 等待被调度执行.
它的源码如下:
```
private void fetchFromScheduledTaskQueue() {
    if (hasScheduledTasks()) {
        long nanoTime = AbstractScheduledEventExecutor.nanoTime();
        for (;;) {
            Runnable scheduledTask = pollScheduledTask(nanoTime);
            if (scheduledTask == null) {
                break;
            }
            taskQueue.add(scheduledTask);
        }
    }
}
```
接下来 **runAllTasks()** 方法就会不断调用 **task = pollTask()** 从 **taskQueue** 中获取一个可执行的 task, 然后调用它的 **run()** 方法来运行此 task.

>`注意`, 因为 EventLoop 既需要执行 IO 操作, 又需要执行 task, 因此我们在调用 EventLoop.execute 方法提交任务时, 不要提交耗时任务, 更不能提交一些会造成阻塞的任务, 不然会导致我们的 IO 线程得不到调度, 影响整个程序的并发量.
