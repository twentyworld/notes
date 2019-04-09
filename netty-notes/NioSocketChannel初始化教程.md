# NioSocketChannel 的初始化过程

首先我们可以以客户端的代码为例，一步一步进入源码:

> 源码可以在[github][1]上看到.

```java
public class TimeClient {

    public void connect(int port, String host) throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try{
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group).channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY,true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new TimeClientHandler());
                        }
                    });

            ChannelFuture future = bootstrap.connect(host, port).sync();
            future.channel().closeFuture().sync();
        }finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) {
        try {
            new TimeClient().connect(18080,"127.0.0.1");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class TimeClientHandler extends ChannelHandlerAdapter {
    private static Logger logger = LoggerFactory.getLogger(TimeClientHandler.class);

    private final ByteBuf message;

    public TimeClientHandler() {
        byte[] req = "query time order".getBytes();
        message = Unpooled.buffer(req.length);
        message.writeBytes(req);
    }


    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(message);
    }


    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, "UTF-8");
        System.out.println("now is: " + body);
    }


    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

首先我们可以着重分析一下写main这段代码：
```java
//赋值
Bootstrap bootstrap = new Bootstrap();
 bootstrap.group(group).channel(NioSocketChannel.class)
         .option(ChannelOption.TCP_NODELAY,true)
         .handler(new ChannelInitializer<SocketChannel>() {
             @Override
             protected void initChannel(SocketChannel ch) throws Exception {
                 ch.pipeline().addLast(new TimeClientHandler());
             }
         });

//实例化
 ChannelFuture future = bootstrap.connect(host, port).sync();
 future.channel().closeFuture().sync();
```

通过阅读Bootstrap的源码可以看到， 第一个步骤主要是对bootstrap进行复制， 真正去实例化的过程是在`bootstrap.connect(host, port).sync()`中进行的.

### 初始化过程
```java
public ChannelFuture connect(String inetHost, int inetPort) {
    return connect(new InetSocketAddress(inetHost, inetPort));
}

public ChannelFuture connect(SocketAddress remoteAddress) {
    if (remoteAddress == null) {
        throw new NullPointerException("remoteAddress");
    }

    validate();
    return doConnect(remoteAddress, localAddress());
}

```
BootStrap会通过执行connect方法去真正的连接server, 方法`connect(String inetHost, int inetPort)`又会通过检查，校验之后执行`doConnect(final SocketAddress remoteAddress, final SocketAddress localAddress)`:

```java
private ChannelFuture doConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    final ChannelPromise promise = channel.newPromise();
    if (regFuture.isDone()) {
        doConnect0(regFuture, channel, remoteAddress, localAddress, promise);
    } else {
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                doConnect0(regFuture, channel, remoteAddress, localAddress, promise);
            }
        });
    }

    return promise;
}
```

#### NioSocketChannel实例化
```java
final ChannelFuture initAndRegister() {
    Channel channel;
    try {
        channel = createChannel();
    } catch (Throwable t) {
        return VoidChannel.INSTANCE.newFailedFuture(t);
    }

    try {
        init(channel);
    } catch (Throwable t) {
        channel.unsafe().closeForcibly();
        return channel.newFailedFuture(t);
    }

    ChannelPromise regFuture = channel.newPromise();
    channel.unsafe().register(regFuture);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    return regFuture;
}
```
##### createChannel()
首先，会先创建一个channel, 这个方法很简单，就是通过ChannelFactory来创建一个channel
```java
Channel createChannel() {
    EventLoop eventLoop = group().next();
    return channelFactory().newChannel(eventLoop);
}

private static final class BootstrapChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Class<? extends T> clazz;

    BootstrapChannelFactory(Class<? extends T> clazz) {
        this.clazz = clazz;
    }

    @Override
    public T newChannel(EventLoop eventLoop) {
        try {
            Constructor<? extends T> constructor = clazz.getConstructor(EventLoop.class);
            return constructor.newInstance(eventLoop);
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(clazz) + ".class";
    }
}
```


> BootstrapChannelFactory类里的field就是我们在初始化BootStrap的时候填入的channel：
> ```java
> public Bootstrap channel(Class<? extends Channel> channelClass) {
>    if (channelClass == null) {
>        throw new NullPointerException("channelClass");
>    }
>    return channelFactory(new BootstrapChannelFactory<Channel>(channelClass));
> }
> ```

在BootstrapChannelFactory的newChannel方法中，NioSocketChannel正式被调用并且被初始化，调用了`NioSocketChannel(EventLoop eventLoop)`构造方法.

接下来是NioSocketChannel的初始化过程:
```java

private static SocketChannel newSocket() {
    try {
        return SocketChannel.open();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a socket.", e);
    }
}

private final SocketChannelConfig config;

/**
 * Create a new instance
 */
public NioSocketChannel(EventLoop eventLoop) {
    this(eventLoop, newSocket());
}

/**
 * Create a new instance using the given {@link SocketChannel}.
 */
public NioSocketChannel(EventLoop eventLoop, SocketChannel socket) {
    this(null, eventLoop, socket);
}

/**
 * Create a new instance
 *
 * @param parent    the {@link Channel} which created this instance or {@code null} if it was created by the user
 * @param socket    the {@link SocketChannel} which will be used
 */
public NioSocketChannel(Channel parent, EventLoop eventLoop, SocketChannel socket) {
    super(parent, eventLoop, socket);
    config = new DefaultSocketChannelConfig(this, socket.socket());
}

protected AbstractNioByteChannel(Channel parent, EventLoop eventLoop, SelectableChannel ch) {
    super(parent, eventLoop, ch, SelectionKey.OP_READ);
}
protected AbstractNioChannel(Channel parent, EventLoop eventLoop, SelectableChannel ch, int readInterestOp) {
    super(parent, eventLoop);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            if (logger.isWarnEnabled()) {
                logger.warn(
                        "Failed to close a partially initialized socket.", e2);
            }
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
protected AbstractChannel(Channel parent, EventLoop eventLoop) {
    this.parent = parent;
    this.eventLoop = validate(eventLoop);
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}
```
这些构造函数存在很大的依赖关系，或者说NioSocketChannel的类继承关系比较深，我们一步一步看， 这里牵涉了以下步骤:
- 创建一个SocketChannel
- 给NioSocketChannel一个默认的SelectionKey， 默认值为SelectionKey.OP_READ。
- 创建一个默认的pipeline, 关于pipiline，会单独细说.
- 把创建的socketChannel 赋值给SelectableChannel, 或者说，在这里，NioSocketChannel其实就是通过操作最开始创建的SocketChannel
- 把SocketChannel设置为非阻塞.

到这里，我们一个完整地创建NioSocketChannel的过程就很清晰了， 其实也可以说NioSocketChannel主要做了三件事儿， 创建了一个SocketChannel，创建了一个pipeline.

接下来我们看看SocketChannel的init过程
##### init(channel)
```java
void init(Channel channel) throws Exception {
    ChannelPipeline p = channel.pipeline();
    p.addLast(handler());

    final Map<ChannelOption<?>, Object> options = options();
    synchronized (options) {
        for (Entry<ChannelOption<?>, Object> e: options.entrySet()) {
            try {
                if (!channel.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                    logger.warn("Unknown channel option: " + e);
                }
            } catch (Throwable t) {
                logger.warn("Failed to set a channel option: " + channel, t);
            }
        }
    }

    final Map<AttributeKey<?>, Object> attrs = attrs();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }
    }
}
```
这个类主要做了三件事儿
- 把handler添加到pipeline
- 把options赋值到该去的地方
- 把attrs赋值到该去的地方

##### 创建ChannelPromise

// todo




































/






/
