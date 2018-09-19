
## IO 演进之路
---

之前的高性能开发会有很多的缺陷。
* 没有缓冲区
* 没有`Channel`的概念，只有输入流和输出流。
* 同步阻塞`IO`，通常会导致线程长时间阻塞。
* 支持的字符串有限，硬件的可移植性有限。

---
#### `Linux`的网络`IO`的模型。
`Linux`将内核的所有外部设备都看作一个文件来看做。

`UNIX`提供了五中不同的`IO`模型。
* 最常用的阻塞`IO`模型：进程空间调用，此后一直等待，进程从到用到结束都是被阻塞的。
* 非阻塞`IO`模型：会检查缓冲区，轮询检查，看内核是否有数据到来。
* `IO`复用模型：`Linux`提供`select/poll`, `epoll`.
* 信号驱动`IO`模型：进程开启信号驱动模型，通过系统调用执行一个信号函数，进程调用后立即返回，非阻塞。当数据准备就绪时，为改进程生成一个信号，通知进程读取数据。
* 异步`IO`：进程告知内核启动操作，操作完成之后通知进程。

这里面有信号驱动是告诉我们什么时候数据准备好，异步`IO`是告诉我们什么时候`IO`操作已经完成（准备好内核数据到把数据复制到用户空间）。

---

本文介绍`Java BIO`（同步阻塞`IO`）,伪异步`IO`，`NIO`（非阻塞`IO`），`AIO`（异步`IO`）这四种IO的情况，并对不同`IO`模型作比较。

### 1. `BIO`
采用`BIO`通信模型的服务器,通常由一个独立的`Acceptor`线程负责监听客户端的连接，它接收到客户端连接请求后为每个客户端创建一个新的线程进程链路连接处理，处理完后，通过输出流返回应答给客户端,线程销毁。

![img][1]

该模型最大的问题性能问题，当客户端并发访问增加后，服务端线程增加，当线程数膨胀后，系统的性能下降，随着并发量增大，系统会发生线程堆栈溢出、创建新线程失败等问题，最终导致线程宕机或者僵死,不能对外提供服务。而且开线程有很大的开销，影响服务器性能。

##### 代码示例
你可以在`[github][6]`上看到这些代码。

**Client 代码：**
```Java
public class TimeClient {
    public static void main(String[] args) {
        int port = 8080;
        if (args!=null && args.length > 0) {

            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                // 采用默认值
            }
        }
        Socket socket = null;
        BufferedReader in = null;
        PrintWriter out = null;
        try {
            socket = new Socket("127.0.0.1", port);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new PrintWriter(socket.getOutputStream(), true);
            out.println("QUERY TIME ORDER");
            System.out.println("Send order 2 server succeed.");
            String resp = in.readLine();
            System.out.println("Now is : " + resp);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (out!=null) {
                out.close();
                out = null;
            }

            if (in!=null) {
                try {
                    in.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                in = null;
            }
            if (socket!=null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                socket = null;
            }
        }
    }
}

```

**Server 代码：**

```Java
public class TimeServer {
    public static void main(String[] args) throws IOException {
        int port = 8080;
        if (args!=null && args.length > 0) {

            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                // 采用默认值
            }
        }
        ServerSocket server = null;
        try {
            server = new ServerSocket(port);
            System.out.println("The time server is start in port : " + port);
            Socket socket = null;
            while (true) {
                socket = server.accept();
                new Thread(new TimeServerHandler(socket)).start();
            }
        } finally {
            if (server!=null) {
                System.out.println("The time server close");
                server.close();
                server = null;
            }
        }
    }
}

class TimeServerHandler implements Runnable {

    private Socket socket;

    public TimeServerHandler(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        BufferedReader in = null;
        PrintWriter out = null;
        try {
            in = new BufferedReader(new InputStreamReader(this.socket.getInputStream()));
            out = new PrintWriter(this.socket.getOutputStream(), true);
            String currentTime = null;
            String body = null;
            while (true) {
                body = in.readLine();
                if (body==null) break;
                System.out.println("The time server receive order : " + body);
                currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new java.util.Date(System.currentTimeMillis()).toString() : "BAD ORDER";
                out.println(currentTime);
            }

        } catch (Exception e) {
            if (in!=null) {
                try {
                    in.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
            }
            if (out!=null) {
                out.close();
                out = null;
            }
            if (this.socket!=null) {
                try {
                    this.socket.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
                this.socket = null;
            }
        }
    }
}

```

2. 伪异步IO

为了解决同步阻塞IO面临的一个链接需要一个线程处理情况，现在引入了“池”的概念，加入了线程池。

当有新的客户端连接的时候，将客户端的`Socket`封装为`Task`（`java`的`Runnable`接口实现了）投递到后端线程池中进行处理。由于线程池可以设置消息队列的大小和最大线程数，因此它的资源是可控的,无论多少个客户端并发访问，都不会导致资源的耗尽和宕机。

![img][2]

伪异步`IO`通讯框架采用了线程池的实现，因此避免了为每个请求都创建一个独立的线程造成的线程资源耗尽问题。但是由于它的底层的通信依然采用的同步阻塞模型,因此无法从根本上解决问题。

`java`输入流`InputStream`：当对`socket`的输入流进行读操作时，它会一直阻塞下去，直到发生以下三种事件。
- 有数据可读；
- 可用数据已经读取完毕；
- 发生空指针或者`IO`异常。

这意味着当对方发数据请求或者应答消息缓慢（网络传输慢）时，读取写入流一方的通讯线程将长时间阻塞，如果对方要`100s`才有消息发生完成，读取的一方的`IO`线程也会将同步阻塞`100s`，在此时间里，其他接入消息只能在消息队列中排队。

`java`输入流`OutputStream`：当调用`OutputStream`的`write`方法写输出流的时候，它将会被阻塞，直到所有要发送的字节全部写入完毕，或者发生异常。搞过`TCP/IP`的都晓得，当消息的接收方处理缓慢的时候，将不能及时从`TCP`缓冲区读取数据，这将导致发送方的`TCP` `window size`不断减小,直到为0，双方处于`keep-alive`状态，消息发送方就不能再将`TCP`缓冲区写入数据，这时采用同步阻塞的`IO`，`write`操作将会无限期阻塞,直到`tcp window size`大于0或者发生`IO`异常。


##### 代码示例
你可以在`[github][7]`上看到这些代码。

主要发生的变化在Server上:
```Java
public class TimeServer {
    public static void main(String[] args) throws IOException {
        int port = 8080;
        if (args!=null && args.length > 0) {

            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                // 采用默认值
            }

        }
        ServerSocket server = null;
        try {
            server = new ServerSocket(port);
            System.out.println("The time server is start in port : " + port);
            Socket socket = null;
            TimeServerHandlerExecutePool singleExecutor = new TimeServerHandlerExecutePool(50, 10000);// 创建IO任务线程池
            while (true) {
                socket = server.accept();
                singleExecutor.execute(new TimeServerHandler(socket));
            }
        } finally {
            if (server!=null) {
                System.out.println("The time server close");
                server.close();
                server = null;
            }
        }
    }
}

public class TimeServerHandlerExecutePool {

    private ExecutorService executor;

    public TimeServerHandlerExecutePool(int maxPoolSize, int queueSize) {
        executor = new ThreadPoolExecutor(
                Runtime.getRuntime().availableProcessors(),
                maxPoolSize,
                120L,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(queueSize));
    }

    public void execute(Runnable task) {
        executor.execute(task);
    }
}
```


### 3. NIO

`NIO`库，是在`JDK1.4`中引入的，`NIO`弥补了同步阻塞`IO`的不足。在所有的数据，`NIO`都是用缓冲区处理掉的`Buffer`，任何时候访问`NIO`中的数据，都是通过缓冲区进行操作。缓冲区实际就是一个数组。`Java NIO`的基础是多路复用器`Selector`，简单来说，`selector`会不断的轮询注册在其上的`Channel`（通道，全双工的），如果某个`Channel`上有新的`TCP`连接接入、读写事件，这个`Channel`会处于就绪状态,会被`Selector`轮询出来，然后通过`SelectionKey`可以获取就绪的`select`集合，进行后续的`IO`操作。

一个多路复用器可以同时轮询多个`Channel`，而且由于`jdk`使用了`epoll`替代了`select`实现，所以没有最大连接句柄的限制。（题外话，这里说的`eopll`、`select`是说的`linux`下的`IO`复用，和`select`、`epoll`一样，清楚流程概念请直接看源码）。

##### 代码示例
你可以在`[github][8]`上看到这些代码。

**Client:**
```Java
public class TimeClient {
    public static void main(String[] args) {

        int port = 8080;
        if (args!=null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                // 采用默认值
            }
        }
        new Thread(new TimeClientHandle("127.0.0.1", port), "TimeClient-001").start();
    }
}


public class TimeClientHandle implements Runnable {
    private String host;
    private int port;

    private Selector selector;
    private SocketChannel socketChannel;

    private volatile boolean stop;

    public TimeClientHandle(String host, int port) {
        this.host = host==null ? "127.0.0.1" : host;
        this.port = port;
        try {
            selector = Selector.open();
            socketChannel = SocketChannel.open();
            socketChannel.configureBlocking(false);
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }

    @Override
    public void run() {
        try {
            doConnect();
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectedKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectedKeys.iterator();
                SelectionKey key = null;
                while (it.hasNext()) {
                    key = it.next();
                    it.remove();
                    try {
                        handleInput(key);
                    } catch (Exception e) {
                        if (key!=null) {
                            key.cancel();
                            if (key.channel()!=null) key.channel().close();
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
                System.exit(1);
            }
        }

        // 多路复用器关闭后，所有注册在上面的Channel和Pipe等资源都会被自动去注册并关闭，所以不需要重复释放资源
        if (selector!=null) try {
            selector.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    private void handleInput(SelectionKey key) throws IOException {

        if (key.isValid()) {
            // 判断是否连接成功
            SocketChannel sc = (SocketChannel) key.channel();
            if (key.isConnectable()) {
                if (sc.finishConnect()) {
                    sc.register(selector, SelectionKey.OP_READ);
                    doWrite(sc);
                } else System.exit(1);// 连接失败，进程退出
            }
            if (key.isReadable()) {
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(readBuffer);
                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes, "UTF-8");
                    System.out.println("Now is : " + body);
                    this.stop = true;
                } else if (readBytes < 0) {
                    // 对端链路关闭
                    key.cancel();
                    sc.close();
                } else ; // 读到0字节，忽略
            }
        }

    }

    private void doConnect() throws IOException {
        // 如果直接连接成功，则注册到多路复用器上，发送请求消息，读应答
        if (socketChannel.connect(new InetSocketAddress(host, port))) {
            socketChannel.register(selector, SelectionKey.OP_READ);
            doWrite(socketChannel);
        } else socketChannel.register(selector, SelectionKey.OP_CONNECT);
    }

    private void doWrite(SocketChannel sc) throws IOException {
        byte[] req = "QUERY TIME ORDER".getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
        writeBuffer.put(req);
        writeBuffer.flip();
        sc.write(writeBuffer);
        if (!writeBuffer.hasRemaining()) System.out.println("Send order 2 server succeed.");
    }

}
```

**Server 代码**
```Java
public class TimeServer {

    public static void main(String[] args) throws IOException {
        int port = 8080;
        if (args!=null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                // 采用默认值
            }
        }
        MultiplexerTimeServer timeServer = new MultiplexerTimeServer(port);
        new Thread(timeServer, "NIO-MultiplexerTimeServer-001").start();
    }
}

class MultiplexerTimeServer implements Runnable {

    private Selector selector;

    private ServerSocketChannel servChannel;

    private volatile boolean stop;

    public MultiplexerTimeServer(int port) {
        try {
            selector = Selector.open();
            servChannel = ServerSocketChannel.open();
            servChannel.configureBlocking(false);
            servChannel.socket().bind(new InetSocketAddress(port), 1024);
            servChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("The time server is start in port : " + port);
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
        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectedKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectedKeys.iterator();
                SelectionKey key = null;
                while (it.hasNext()) {
                    key = it.next();
                    it.remove();
                    try {
                        handleInput(key);
                    } catch (Exception e) {
                        if (key!=null) {
                            key.cancel();
                            if (key.channel()!=null) key.channel().close();
                        }
                    }
                }
            } catch (Throwable t) {
                t.printStackTrace();
            }
        }

        // 多路复用器关闭后，所有注册在上面的Channel和Pipe等资源都会被自动去注册并关闭，所以不需要重复释放资源
        if (selector!=null) try {
            selector.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void handleInput(SelectionKey key) throws IOException {

        if (key.isValid()) {
            // 处理新接入的请求消息
            if (key.isAcceptable()) {
                // Accept the new connection
                ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                SocketChannel sc = ssc.accept();
                sc.configureBlocking(false);
                // Add the new connection to the selector
                sc.register(selector, SelectionKey.OP_READ);
            }
            if (key.isReadable()) {
                // Read the data
                SocketChannel sc = (SocketChannel) key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(readBuffer);
                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes, "UTF-8");
                    System.out.println("The time server receive order : " + body);
                    String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new java.util.Date(System.currentTimeMillis()).toString() : "BAD ORDER";
                    doWrite(sc, currentTime);
                } else if (readBytes < 0) {
                    // 对端链路关闭
                    key.cancel();
                    sc.close();
                } else ; // 读到0字节，忽略
            }
        }
    }

    private void doWrite(SocketChannel channel, String response) throws IOException {
        if (response!=null && response.trim().length() > 0) {
            byte[] bytes = response.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            channel.write(writeBuffer);
        }
    }
}
```


`NIO`服务端序列图
![img][3]

1.打开`ServerSocketChannel`，用于监听客户端的连接，它是所有客户端连接的父管道。
```Java
 ServerSocketChannel accptorSvr = ServerSocketChannel.open();

 ```
 2.绑定监听端口，设置连接为非阻塞模式。
```Java
acceptorSvr.socket().bind(
　　new InetSocketAddress(InetAddress.getByName("IP"),port));
acceptorSvr.configureBlocking(false);
```
3.创建`Reactor`线程，创建多路复用器并启动线程。
```Java
Selector selectot = Selector.open();
new Thread(new RectorTask()).start();
```
4.将`SelectSocketChannel`注册到`Reactor`线程的多路复用器`selector`上，监听`accept`事件。
```Java
SelectionKey key = acceptorSvr.register(selector,SelectionKey.OP_ACCEPT,ioHandler);
```

5.多路复用器在线程`run`方法中无线循环里轮询准备就绪的`key`。
```Java
int num = selector.select();
Set selectkeys = selector.selectedKeys();
Iterator it = selectkeys.iterator();
while(it.hasNext)
{
      SelectionKey key = (SelectionKey)it.next;
      /*     deal with  IO event    \*/    
}
```

 6.多路复用监听到有新的用户接入，处理新的接入请求,完成`TCP`三次握手，建立物理连接。
```Java
SocketChannel sc = ssc.accept();
```
7.设置客户端链路为非阻塞模式
```Java
sc.configureBlocking(false);
sc.socket().setReuseAddress(true);

```
8.将新接入的客户端连接注册到`Reactor`线程的多路复用器上，监听读操作,用来读取客户端发送的网络消息。
```Java
SelectionKey key = sc.register(selector,SelectionKey.OP_READ,ioHangler);
```
9.异步读取客户端请求消息到缓冲区
```Java
int readNumber = channel.read(receivedBuffer);
```
10.对`bytebuffer`进行编解码,如果有半包消息指针`reset`，继续读取后续的报文，将解码成功的消息封装成`task`，投递到业务线程池中，进行业务逻辑处理。
```Java
Object message = null;
while (buffer.hasRemain()){
    byteBuffer.mark();
    Object message = decode(byteBuffer);
    if(message==null){
        byteBuffer.reset();
        break;
    }
    messageList.add(message);
}
if(!byteBuffer.hasRemain()){
    byteBuffer.clear();
}
else byteBuffer.compact();
if(messageList!=null & !messageList.isEmpty()) {
    for(Object messageF:messageList)
        handleTask(messageE);
}
```

11.将`pojo`对象`encode`成`bytebuffer`，调用`SocketChannel`的异步`write`接口,将消息异步发送到客户端。
```Java
socketChannel.wite(buffer);
```

>注意：如果发送区TCP缓冲区满了，会导致写半包，此时，需要注册写操作位，循环写，直到整个包消息写入TCP缓冲区。

**`NIO`客户端序列图**
![img][4]

1.打开`SocketChannel`，绑定客户端本地地址（可选，默认系统会随机分配一个可用的本地地址）
```Java
 SocketChannel clientChannel = SocketChannel.open();
```
2.设置`SocketChannel`为非阻塞模式,同时设置连接的`TCP`参数。
```Java
SocketChannel.configureBlocking(false);
socket.setReuseAddress(true);
socket.setReceiveBufferSize(BUFFER_SIZE);
socket.setSendBufferSize(BUFFER_SIZE);
```
3.异步连接服务器。
```Java
boolean connected = clientChannel.connect(new InetSocketAdress("ip",port));
```
4.判断是否连接成功，如果成功，则直接注册读状态位到多路复用器中，如果没成功（异步连接,返回`false`，说明客户端已经已经发送`sync`包，服务端没有返回`ack`包,物理连接还没建立——关于`ack`、`sync`包，请读者自行查阅`TCP/IP`中的`TCP`的三次握手，四次分手的过程）
```Java
if(connect)
　　clientChannel.register(selector,SelectionKey.OP_READ,ioHandler);
else
　　clientChannel.register(selector,SelectionKey.OP_CONNECT,ioHandler);
```
5.向`Reactor`线程的多路复用器注册`OP_CONNECT`状态位，监听服务器的`TCP ACK`应答。
```Java
clientChannel.register(selector,SelectionKey.OP_CONNECT,ioHandler);
```
 6.创建`Reactor`线程，创建多路复用器并启动线程。
```Java
Selector selectot = Selector.open();
new Thread(new RectorTask()).start();
```
7.多路复用器在线程`run`方法中无线循环里轮询准备就绪的`key`。
```Java
int num = selector.select();
Set selectkeys = selector.selectedKeys();
Iterator it = selectkeys.iterator();
while(it.hasNext)
{
      SelectionKey key = (SelectionKey)it.next;
      /*     deal with  IO event    \*/    
}
```

8.接收`connect`事件进行处理
```Java
if(key.isConnectable())
　　//handlerConnect();
```
9.判断连接结果，如果连接成功，注册读事件到多路复用器
```Java
if(channel.finishConnect())
　　registerRead();
```
10.注册读事件到多路复用器
```Java
clientChannel.register(selector,SelectionKey.OP_READ,ioHandler);
```
11.异步读取客户端请求消息到缓冲区
```Java
int readNumber = channel.read(receivedBuffer);
```
12.对`bytebuffer`进行编解码,如果有半包消息指针`reset`，继续读取后续的报文，将解码成功的消息封装成`task`，投递到业务线程池中，进行业务逻辑处理。
```Java
Object message = null;
while (buffer.hasRemain()){
    byteBuffer.mark();
    Object message = decode(byteBuffer);
    if(message==null){
        byteBuffer.reset();
        break;
    }
    messageList.add(message);
}
if(!byteBuffer.hasRemain()){
    byteBuffer.clear();
}
else byteBuffer.compact();
if(messageList!=null & !messageList.isEmpty()) {
    for(Object messageF:messageList)
        handleTask(messageE);
}

```
13.将`pojo`对象`encode`成`bytebuffer`，调用`SocketChannel`的异步`write`接口,将消息异步发送到客户端。
```Java
socketChannel.wite(buffer);
```


4. AIO

`NIO 2.0` 中引入了新的异步通道的概念，并提供了异步文件通道和异步套接字通道的实现。

异步通道提供2种方式获取操作结果：

- 通过`java.util.concurrent.Futurn`类来表示异步操作的结果；
- 在执行异步操作的时候传入一个`java.nio.channels`.

`CompletionHandler`接口的实现类作为操作完成的回溯。

`NIO2.0`的异步套接字通道，对应`UNIX`网络编程中的`事件驱动IO`（AIO），它不需要通过多路复用器`Selector`对注册的通道进行轮询操作。
##### 代码示例
你可以在`[github][9]`上看到这些代码。

5. 四种IO比较

![img][5]



[1]:https://images2015.cnblogs.com/blog/478846/201511/478846-20151125123138265-1932927247.png
[2]:https://images2015.cnblogs.com/blog/478846/201511/478846-20151125124945734-1006835068.png
[3]:https://images2015.cnblogs.com/blog/478846/201511/478846-20151125150330265-1118717241.png
[4]:https://images2015.cnblogs.com/blog/478846/201511/478846-20151127115604187-1347290251.png
[5]:https://images2015.cnblogs.com/blog/478846/201511/478846-20151127125602796-10241097.png

[6]:https://github.com/twentyworld/learn/tree/master/NettyLearn/src/main/java/com/learn/TimeHandler/bio
[7]:https://github.com/twentyworld/learn/tree/master/NettyLearn/src/main/java/com/learn/TimeHandler/pio
[8]:https://github.com/twentyworld/learn/tree/master/NettyLearn/src/main/java/com/learn/TimeHandler/nio
[9]:https://github.com/twentyworld/learn/tree/master/NettyLearn/src/main/java/com/learn/TimeHandler/aio
