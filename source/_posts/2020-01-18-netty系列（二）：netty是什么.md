---
title: netty系列（二）：netty是什么
tags:
  - Netty
categories:
  - Netty
date: 2020-01-18 16:50:22
---


> 上一篇文章介绍了 netty 的同步/异步/阻塞/非阻塞的概念和四种常用 IO 模型。本文介绍网络编程的发展，从 bio 发展到 nio 最后到 netty，我们可以感受到 netty 给网络编程带来的极大便利性。

<!--more-->

网络编程，简单来说就是实现一个客户端与服务端进行网络通信的程序。我们从最传统的 BIO 开始介绍，本章只做简单介绍，不涉及深入原理。

## BIO

BIO 全称是 Blocking IO，是 JDK1.4 之前的传统 IO 模型，本身是同步阻塞模式。 

BIO分为 ServerSocket 和 Socket 两种类，分别对应服务端和客户端。

### ServerSocket

ServerSocket 对象在服务端运行，负责接收客户端连接。

1. 服务端调用 `ServerSocket serverSocket = new ServerSocket(600);` 绑定一个本地端口用于接收客户端请求。

2. 然后调用 `serverSocket.accept()` ，这个方法的执行将使 Server 端一直阻塞，直到捕捉到一个来自 Client 端的请求，并返回一个用于与该 Client 通信的 Socket 对象 Link-Socket。此后 Server 程序只要向这个 Socket 对象读写数据，就可以实现向远端的 Client 读写数据。
3. `serverSocket` 可以接收多次客户端请求，当需要结束服务端程序时调用 `serverSocket.close()`。

ServerSocket 一般仅用于设置端口号和监听，真正进行通信的是服务器端的 Socket 与客户端的 Socket，在ServerSocket 进行 accept 之后，就将主动权转让了。

### Socket

客户端通过创建 Socket 对象，向服务端发起连接。

1. 客户端调用 `Socket socket = new Socket(“127.0.0.1”，600);`，创建到服务端的连接。第一个参数是 Server 的主机地址，第二个参数是 Server 绑定的端口号。
2. 在服务端和客户端之间创建连接后，调用 `OutputStream outputStream = socket.getOutputStream()` 和 `InputStream inputStream = socket.getInputStream()` 获取输出/输出流，然后可以调用 `outputStream.write()` 和 `inputStream.read()` 向服务器端写/读数据。
3. 客户端活动完成调用 `socket.close()` 结束客户端程序。

### 示例

**服务端代码**

```
public class BIOServer {
    public static void main(String[] args) throws IOException {
        // 服务端绑定本地端口并启动
        ServerSocket serverSocket = new ServerSocket(7000);
        while (true) {
            // accept() 会阻塞服务端，直到获取到一个客户端的请求
            Socket socket = serverSocket.accept();
            // 接收到客户端请求后，开始读取数据
            byte[] data = new byte[1024];
            InputStream inputStream = socket.getInputStream();
            while (true) {
                int len;
                // 按字节流方式读取数据
                while ((len = inputStream.read(data)) != -1) {
                    System.out.println(new String(data, 0, len) + " receive");
                }
            }
        }
    }
}
```

这是最简单的服务端代码，服务端代码创建一个 `ServerSocket` 用于监听本地 7000 端口，然后在死循环里面调用 serverSocket.accept() 阻塞服务端直到获取到一个客户端连接，最后以字节流方式读取客户端数据并打印。

**客户端代码**

```
public class BIOClient {
    public static void main(String[] args) throws IOException, InterruptedException {
        Socket socket = new Socket("127.0.0.1", 7000);
        while (true) {
            String str = new Date() + ": hello world";
            socket.getOutputStream().write(str.getBytes());
            socket.getOutputStream().flush();
            System.out.println(str + " send");
            Thread.sleep(2000);
        }
    }
}
```

服务端代码连接本地 7000 端口，每隔两秒向服务端发送 "hello world" 的数据。

### 分析

BIO 编程模型在客户端较少的情况下运行良好，但是对于客户端比较多的业务来说，单机服务端可能需要支撑成千上万的连接，BIO 模型可能就不太合适了。

从服务端代码中我们可以看到，在传统的 BIO 模型中，每个连接创建成功之后都需要一个线程来维护，每个线程包含一个 while 死循环，那么 1w 个连接对应 1w 个线程，继而 1w 个 while 死循环，这就带来如下几个问题：

1. 线程资源受限：线程是操作系统中非常宝贵的资源，同一时刻有大量的线程处于阻塞状态是非常严重的资源浪费，操作系统耗不起
2. 线程切换效率低下：单机 cpu 核数固定，线程爆炸之后操作系统频繁进行线程切换，应用性能急剧下降。
3. 数据读写是以字节流为单位，效率不高。

## NIO

为了解决 BIO 存在的问题，在 Java 1.4 中引入了 NIO 框架。 NIO 可以理解为 Non-blocking。

在传统的 BIO 模型中一个连接对应一个 while 死循环，死循环的目的就是不断监测这条连接上是否有数据可以读，大多数情况下，1w个连接里面同一时刻只有少量的连接有数据可读，因此，很多个while死循环都白白浪费掉了，因为读不出啥数据。

<div style="width: 80%; margin: auto">![bio](bio.png)</div>

而在NIO模型中，他把这么多 while 死循环变成一个死循环，这个死循环由一个线程控制。这个线程就是 NIO 模型中的 selector。一条连接来了直接把这条连接注册到 selector 上，通过检查这个selector，就可以批量监测出有数据可读的连接，进而读取数据。这样导致线程数量大大减少。

<div style="width: 80%; margin: auto">![nio](nio.png)</div>

NIO 提供了 Channel、Buffer 和 Selector 三个核心组件。BIO 基于字节流和字符流进行操作，而 NIO 基于Channel 和 Buffer 进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector 用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个线程可以监听多个数据通道。整体流程大致如下：

<div style="width: 80%; margin: auto">![component](component.png)</div>

### Channel

Channel 类似于 BIO 中的 Stream。只不过 Stream 是单向的，譬如：InputStream, OutputStream.而 Channel 是双向的，既可以用来进行读操作，又可以用来进行写操作。

NIO 中的 Channel 的主要实现有：

- FileChannel
- DatagramChannel
- SocketChannel
- ServerSocketChannel

分别可以对应文件 IO、UDP 和 TCP（Client 和 Server）

### Buffer

BIO 模型中，每次都是从操作系统底层一个字节一个字节地读取数据，直至读取所有字节，它们没有被缓存在任何地方。此外，它不能前后移动流中的数据。如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区。

而 NIO 维护一个缓冲区 Buffer，在读取数据时，它是直接读到缓冲区中的；在写入数据时，写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作。

与Java基本类型相对应，NIO 提供了多种 Buffer 类型，如 ByteBuffer、CharBuffer、IntBuffer等，区别就是读写缓冲区时的单位长度不一样（以对应类型的变量为单位进行读写）。

Buffer 中有三个很重要的变量：

- capacity：缓冲区数组的总长度
- position：下一个要操作的数据元素的位置
- limit：读/写边界位置，limit<=capacity

我们读写 buffer 的流程一般如下：

1. 通过 `ByteBuffer.allocate(11)` 方法创建了一个 11 个 byte 的数组的缓冲区，初始状态如上图，position 的位置为 0，capacity 和 limit 默认都是数组长度。这个时候 ByteBuffer 处于写模式，等待 Channel 数据写入。

<div style="width: 50%; margin: auto">![buffer1](buffer1.png)</div>

2. 写入 5 个字节，postion 移动了 5 个位置，limit 和 capacity 保持不变。

<div style="width: 50%; margin: auto">![buffer2](buffer2.png)</div>

3. 调用 `ByteBuffer.flip()` 方法，将写模式改成读模式，position 设回 0，并将 limit 设成之前的 position 的值。这时 position 到 limit 这个区间内的就是可读数据。

<div style="width: 50%; margin: auto">![buffer3](buffer3.png)</div>

4. 在下一次写数据之前我们调用 `ByteBuffer.clear()` 方法，将读模式改成写模式。缓冲区的索引位置又回到了初始位置。注意这个时候 Buffer 中的数据并未被清除，只是这些标记告诉我们可以从哪里开始往 Buffer 里写数据。
5. 如果 Buffer 中仍有未读的数据，且后续还需要这些数据，但是此时想要先写些数据，那么使用 `ByteBuffer.compact()` 方法。将所有未读的数据拷贝到 Buffer 起始处，然后将 position 设到最后一个未读元素正后面。limit 属性依然像 clear() 方法一样，设置成 capacity。现在 Buffer 准备好写数据了，但是不会覆盖未读的数据。
6. 如果你有一部分数据需要重复读写，调用 `ByteBuffer.mark()` 方法，可以标记 Buffer 中的一个特定的 position，之后可以通过调用 `ByteBuffer.reset()` 方法恢复到这个position。
7. 如果你需要将同一个 Buffer 的数据写入多个通道时，在第一次读取完 Buffer 后，调用 `ByteBuffer.rewind()` 方法，将 position 设回 0，limit 保持不变，这样就可以重读 Buffer 中的所有数据。

### Selector

Selector 可以理解为一个单线程管理器，管理多个 Channel。我们将 Channel 注册到 Selector 上，然后设置关注的事件，然后调用 Selector.select() 方法，等待事件发生。

Selector 可以监听以下四种事件，并对应 SelectionKey 的四个常量：

* Connect：连接就绪事件，客户端 channel 成功连接到服务器上，对应 SelectionKey.OP_CONNECT

* Accept：接收就绪事件，服务端 socket channel 准备好接收客户端的连接，对应 SelectionKey.OP_ACCEPT

* Read：读就绪事件，channel 有数据可读，对应 SelectionKey.OP_READ

* Write：写就绪事件，准备写数据到 channel，对应 SelectionKey.OP_WRITE

### 示例

**服务端代码**

```
public class NIOServer {

    private void initServer() throws IOException {
        // 创建 serverSelector
        Selector serverSelector = Selector.open();

        // 创建 serverSocketChannel，设置为非阻塞，并绑定本地端口
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.socket().bind(new InetSocketAddress(7000));

        // 把 serverSocketChannel 注册到 selector 上并设置关心事件为 ON_ACCEPT 事件，ACCEPT 事件表明这是一个服务器 channel
        SelectionKey selectionKey = serverSocketChannel.register(serverSelector, SelectionKey.OP_ACCEPT);

        //轮询
        while (true) {
            // select() 会一直阻塞，直到有事件发生
            serverSelector.select();
            // 运行到这一步说明已经有事件发生，获取并遍历的所有事件key集合
            Set keys = serverSelector.selectedKeys();
            Iterator iterator = keys.iterator();
            while (iterator.hasNext()) {
                // 获取 key 示例
                SelectionKey key = (SelectionKey) iterator.next();
                // 删除 key 实例，因为 selector 不会删除 key 实例，需要手动删除
                iterator.remove();
                if (key.isAcceptable()) {
                    // 服务端待连接就绪事件
                    doAccept(key);
                } else if (key.isReadable()) {
                    // 读就绪事件
                    doRead(key);
                }
            }
        }
    }

    private void doAccept(SelectionKey key) throws IOException {
        // 当 accept 事件发生的时候，创建 clientChannel 注册到 serverSelector 上并注册关心事件为 OP_READ 和 OP_WRITE，等待读/写事件发生
        System.out.println("服务端等待客户端连接就绪");
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
        SocketChannel clientChannel = serverChannel.accept();
        clientChannel.configureBlocking(false);
        clientChannel.register(key.selector(), SelectionKey.OP_READ);
    }

    private void doRead(SelectionKey key) throws IOException {
        System.out.println("服务端开始读取数据");
        SocketChannel clientChannel = (SocketChannel) key.channel();
        // 初始化，设置postition=0，limit=capacity
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        // 写模式，设置position=可读字节数，limit=capacity
        long bytesRead = clientChannel.read(byteBuffer);
        while (bytesRead > 0) {
            // 写模式转成读模式，设置postition=0，limit=可读字节数
            byteBuffer.flip();
            byte[] data = byteBuffer.array();
            String info = new String(data).trim();
            System.out.println("服务端接收的消息是： " + info);
            // 读模式转成写模式，设置position=0，limit=capacity
            byteBuffer.clear();
            // 继续开始下一次读入
            bytesRead = clientChannel.read(byteBuffer);
        }
        clientChannel.close(); //关闭channel
    }

    public static void main(String[] args) throws IOException {
        NIOServer nioServer = new NIOServer();
        nioServer.initServer();
    }
}
```

服务端代码简单解析

1. 服务端创建 serverSelector。
2. 服务端创建 serverSocketChannel，绑定一个本地端口，然后注册到 serverSelector，并设置关心事件为  OP_ACCEPT。调用 `serverSelector.select()` 方法等待事件发生。
3. 事件发生，获取并遍历 selectedKeys，对事件类型为 Accept 和 Read 的事件进行相应操作。

**客户端代码**

```
public class NIOClient {

    private void initClient() throws IOException {
        Selector clientSelector = Selector.open();

        SocketChannel clientChannel = SocketChannel.open();
        clientChannel.configureBlocking(false);
        clientChannel.connect(new InetSocketAddress(7000));
        clientChannel.register(clientSelector, SelectionKey.OP_CONNECT);

        while (true) {
            clientSelector.select();
            Iterator<SelectionKey> iterator = clientSelector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                if (key.isConnectable()) {
                    doConnect(key);
                } else if (key.isWritable() && key.isValid()) {
                    doWrite(key);
                }
            }
        }
    }

    public void doConnect(SelectionKey key) throws IOException {
        System.out.println("客户端连接成功");
        SocketChannel clientChannel = (SocketChannel) key.channel();
        if (clientChannel.isConnectionPending()) {
            clientChannel.finishConnect();
        }
        clientChannel.configureBlocking(false);
        clientChannel.register(key.selector(), SelectionKey.OP_WRITE);
    }

    private void doWrite(SelectionKey key) throws IOException {
        System.out.println("客户端开始向服务端写数据");
        SocketChannel clientChannel = (SocketChannel) key.channel();
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        String info = "hello world";
        byteBuffer.clear();
        byteBuffer.put(info.getBytes());
        byteBuffer.flip();
        while (byteBuffer.hasRemaining()) {
            clientChannel.write(byteBuffer);
        }
        System.out.println("客户端发送的消息是： " + info);
        clientChannel.close();
    }

    public static void main(String[] args) throws IOException {
        NIOClient nioClient = new NIOClient();
        nioClient.initClient();
    }
}
```

客户端代码类似于服务端代码，区别是对事件类型为 Connect 和 Write 的事件做了相应处理。在这里不再做详细介绍。

### 分析

由此我们可以看出来虽然 NIO 虽然解决了 BIO 的一些痛点，但是 NIO 的原生编程也带来了新的问题：

1. JDK的NIO编程需要了解很多的概念，编程复杂，对 NIO 入门非常不友好，编程模型不友好，ByteBuffer 的 api 简直反人类
2. 对 NIO 编程来说，一个比较合适的线程模型能充分发挥它的优势，而 JDK 没有给你实现，你需要自己实现，就连简单的自定义协议拆包都要你自己实现
3. JDK 的 NIO 底层由 epoll 实现，该实现饱受诟病的空轮训 bug 会导致 cpu 飙升100%
4. 项目庞大之后，自行实现的 NIO 很容易出现各类bug，维护成本较高

下面我们就看下 Netty 是怎么解决 NIO 带来的各种问题的。

## Netty

Netty 是一个异步事件驱动的网络应用框架，用于快速开发可维护的高性能服务器和客户端。简单来说就是 Netty 封装了 JDK 的 NIO，可以简化开发流程。

相比起 NIO，Netty 的优势有：

1. 使用JDK自带的NIO需要了解太多的概念，编程复杂，一不小心bug横飞
2. Netty底层IO模型随意切换，而这一切只需要做微小的改动，改改参数，Netty可以直接从NIO模型变身为IO模型
3. Netty自带的拆包解包，异常检测等机制让你从NIO的繁重细节中脱离出来，让你只需要关心业务逻辑
4. Netty解决了JDK的很多包括空轮询在内的bug
5. Netty底层对线程，selector做了很多细小的优化，精心设计的reactor线程模型做到非常高效的并发处理
6. 自带各种协议栈让你处理任何一种通用协议都几乎不用亲自动手
7. Netty社区活跃，遇到问题随时邮件列表或者issue
8. Netty已经历各大rpc框架，消息中间件，分布式通信中间件线上的广泛验证，健壮性无比强大

Netty 的具体内容我们后面再详细介绍，这里简单介绍一下 Netty 的使用，感受一下 Netty 给开发带来的便利性。

**引入 Maven 依赖**

```java
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-all</artifactId>
        <version>4.1.6.Final</version>
    </dependency>
```

**服务端代码**

```
public class NettyServer {
    public static void main(String[] args) {
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        NioEventLoopGroup boos = new NioEventLoopGroup();
        NioEventLoopGroup worker = new NioEventLoopGroup();
        serverBootstrap
                .group(boos, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    protected void initChannel(NioSocketChannel ch) {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new SimpleChannelInboundHandler<String>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext ctx, String msg) {
                                System.out.println(msg);
                            }
                        });
                    }
                })
                .bind(8000);
    }
}
```

**客户端代码**

```
public class NettyClient {
    public static void main(String[] args) throws InterruptedException {
        Bootstrap bootstrap = new Bootstrap();
        NioEventLoopGroup group = new NioEventLoopGroup();
        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel ch) {
                        ch.pipeline().addLast(new StringEncoder());
                    }
                });
        Channel channel = bootstrap.connect("127.0.0.1", 8000).channel();
        while (true) {
            channel.writeAndFlush(new Date() + ": hello world!");
            Thread.sleep(2000);
        }
    }
}
```

代码的具体内容在这里先不做分析，但是我们可以看到，这两段简单的代码，就可以完成 NIO 原生编程那一大段代码的功能，代码非常清晰易于维护。

## 总结

本文简单介绍了 BIO、NIO 的一些基本原理，认识到 NIO 相对于 BIO 在性能和资源管理方面带来的巨大提升，但 NIO 的原生编程过于复杂，难以维护。Netty 通过对 NIO 进行封装，大大降低了开发的复杂性。如果涉及到网络编程，Netty 将是你是不二选择。

## 参考

[Netty实战]()

[《跟闪电侠学Netty》开篇：Netty是什么？](https://www.jianshu.com/p/a4e03835921a)

[Java NIO？看这一篇就够了！](https://blog.csdn.net/forezp/article/details/88414741)