---
title: Netty系列（四）：Bootstrap
date: 2020-05-27 17:33:07
tags:
  - Netty
categories:
  - Java组件
  - Netty
---

> 上篇文章我们简单介绍了 Netty 的线程模型和基本组件，后面我们开始详细介绍 Netty 每个基本组件。本文先介绍 Netty 的启动器 Bootstrap，包括 Bootstrap 的简介、类分析和实战。

<!--more-->

# Bootstrap是什么

Bootstrap 是 Netty 的启动器，负责把 Netty 的组件例如 EventLoopGroup 等组装起来，并配置系统属性，然后启动 Netty 服务。例如下面是 Bootstrap 的常用用法：

```
// 新建Bootstrap
Bootstrap b = new Bootstrap();

try {   
    //1 设置reactor 线程
    b.group(bossLoopGroup, workerLoopGroup);
    //2 设置nio类型的channel
    b.channel(NioServerSocketChannel.class);
    //3 设置监听端口
    b.localAddress(new InetSocketAddress(port));
    //4 设置通道选项
    b.option(ChannelOption.SO_KEEPALIVE, true);
    b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

    //5 装配流水线
    b.childHandler(new ChannelInitializer<SocketChannel>()
    {
        //有连接到达时会创建一个channel
        protected void initChannel(SocketChannel ch) throws Exception
        {
            ch.pipeline().addLast(new ProtobufDecoder());
            ch.pipeline().addLast(new ProtobufEncoder());
            // pipeline管理channel中的Handler
            // 在channel队列中添加一个handler来处理业务
            ch.pipeline().addLast("serverHandler", serverHandler);
        }
    });
    // 6 开始绑定server
    // 通过调用sync同步方法阻塞直到绑定成功

    ChannelFuture channelFuture = b.bind().sync();
    LOGGER.info(ChatServer.class.getName() +
            " started and listen on " +
            channelFuture.channel().localAddress());

    // 7 监听通道关闭事件
    // 应用程序会一直等待，直到channel关闭
    ChannelFuture closeFuture=  channelFuture.channel().closeFuture();
    closeFuture.sync();
} catch (Exception e) {
    e.printStackTrace();
} finally {
    // 8 优雅关闭EventLoopGroup，
    // 释放掉所有资源包括创建的线程
    workerLoopGroup.shutdownGracefully();
    bossLoopGroup.shutdownGracefully();
}
```

为了更清楚地了解 Bootstrap 的用法，我们先介绍  Bootstrap 的类继承关系。

# Bootstrap 类分析

## 类分析

![inherit](inherit.png)

AbstractBootstrap 中定义了一系列属性供子类使用，每个属性都有 setter 和 getter 方法，另外还有用于验证、启动等其他方法。

ServerBootstrap 和 Bootstrap 都继承自抽象类 AbstractBootstrap，都能使用父类中的属性。其中 ServerBootstrap 是服务器端的启动类，Bootstrap 是客户端的系统类。两者在类方法上的区别有：

* ServerBootstrap 的 group 的 setter 方法可以设置两个 eventLoopGroup：一个 bossLoopGroup 和一个 workerLoopGroup（也可以只设置一个 eventLoopGroup 同时完成 bossLoopGroup 和 workerLoopGroup 两个线程组的工作）。Bootstrap 的 group 的 setter 方法只需要设置一个 eventLoopGroup，用于与服务端的所有交互。
  * bossLoopGroup 用于监听连接，专门 accept 新的客户端连接。
  * workerLoopGroup 用于处理与客户端除了 accept 之外的其他交互。
* ServerBootstrap 有 child 前缀的一些属性，用于配置 ServerSocketChannel 的相关属性，而 Bootstrap 没有 child 前缀的属性。
* Bootstrap 的 remoteAddress 属性可以设置远程主机的地址和端口，ServerBootstrap 没有该属性。
* ServerBootstrap 使用父类的 `bind` 方法进行启动，Bootstrap 使用自定义的 `connect` 方法进行启动。

ServerBootstrap 和 Bootstrap 都使用 Builder 链式模式设置属性，例如

```
bootstrap.group(group)
	.channel(NioSocketChannel.class)
	.option(ChannelOption.TCP_NODELAY, true);
```

## 属性介绍

### group 属性

设置 EventLoopGroup。服务端使用 ServerBootstrap 的 group 的 setter 方法可以设置两个 eventLoopGroup：一个 bossLoopGroup 和一个 workerLoopGroup。

* **bossLoopGroup**：用于监听连接，专门 accept 新的客户端连接。
* **workerLoopGroup**：用于处理与客户端除了 accept 之外的其他交互。

```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
serverBootstrap.group(bossGroup, workerGroup);
```

上面这种模型对应上篇介绍的主从 Reactor 多线程模型，也可以只设置一个 eventLoopGroup 同时完成 bossLoopGroup 和 workerLoopGroup 两个线程组的工作，对应单 Reactor 多线程模型。

```
EventLoopGroup eventGroup = new NioEventLoopGroup();
serverBootstrap.group(eventGroup);
```

客户端一般只需要设置一个 eventLoopGroup 就可以完成和服务端的所有交互。

```
EventLoopGroup eventGroup = new NioEventLoopGroup();
bootstrap.group(eventGroup);
```

### channelFactory 属性

设置 Channel 属性，ServerBootstrap 和 Bootstrap 通用，例如

```
serverBootstrap.channel(NioServerSocketChannel.class);
```

这里我们使用 NioServerSocketChannel.class，看似是直接创建了 ServerSocketChannel 实例，但实际上是调用工厂类 BootstrapChannelFactory 来创建实例，这就是属性名称 channelFactory 的由来。从源码可以看到：

    public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        // 调用 BootstrapChannelFactory 获取 Channel 实例
        return channelFactory(new BootstrapChannelFactory<C>(channelClass));
    }
    
    private static final class BootstrapChannelFactory<T extends Channel> implements ChannelFactory<T> {
        private final Class<? extends T> clazz;
    
        BootstrapChannelFactory(Class<? extends T> clazz) {
            this.clazz = clazz;
        }
        
        // 通过 newInstance 创建实例
        @Override
        public T newChannel() {
            try {
                return clazz.newInstance();
            } catch (Throwable t) {
                throw new ChannelException("Unable to create Channel from class " + clazz, t);
            }
        }
    } 
serverBootstrap.channel() 方法可配置的常用类有：

* **NioDatagramChannel**：基于 NIO，服务端和客户端通用
* **NioServerSocketChannel**：基于 NIO，服务端 channel() 方法使用
* **NioSocketChannel**：基于 NIO，服务端 childChannel() 和客户端 channel() 方法使用
* **OioDatagramChannel**：基于 BIO，同 NioDatagramChannel
* **OioServerSocketChannel**：基于 BIO，同 NioServerSocketChannel
* **OioSocketChannel**：基于 BIO，同 NioSocketChannel

我们看到除了可以使用 NIO 的相关 Channel 外，还可以使用 OIO（BIO）的相关 Channel，只需要修改一下类名称即可在 NIO 和 OIO 之间进行切换，对开发非常友好。一般情况下我们只使用 NIO 的相关 Channel。

### localAddress、remoteAddress 属性

localAddress 用于设置本地端口，一般用于服务端。

```
// 直接使用端口
serverBootstrap.localAddress(8000);

// 或者使用 InetSocketAddress 对象
serverBootstrap.localAddress(new InetSocketAddress(8000));
```

也可以不通过 localAddress 来设置端口，在服务端调用 `bind` 方法的时候再设置端口，两者效果一致。

```
serverBootstrap.bind(8000).sync();
// 或
serverBootstrap.bind(new InetSocketAddress(8000)).sync();
```

remoteAddress 用于设置远程主机地址和端口，一般用于客户端。

```
// 直接使用地址端口
bootstrap.remoteAddress("192.168.10.1", 8000);

// 或者使用 InetSocketAddress 对象
bootstrap.remoteAddress(new InetSocketAddress("192.168.10.1", 8000));
```

也可以不通过 remoteAddress 来设置远程主机地址和端口，在客户点调用 `connect` 方法的时候再设置，两者效果一致。

```
bootstrap.connect("192.168.10.1", 8000).sync();
// 或
bootstrap.connect(new InetSocketAddress("192.168.10.1", 8000)).sync();
```

### options 属性

设置 Channel 的 options 属性，ServerBootstrap 和 Bootstrap 通用，options 对服务端而言是设置 ServerSocketChannel 的属性，对客户端而言是设置 SocketChannel 的属性。

```
serverBootstrap.option(ChannelOption.TCP_NODELAY, true);
```

options 可设置的属性有：

**通用参数**

* **CONNECT_TIMEOUT_MILLIS**：Netty 参数，连接超时毫秒数，默认值30000毫秒即30。
* **MAX_MESSAGES_PER_READ**：Netty 参数，一次 Loop 读取的最大消息数，对于 ServerChannel或者NioByteChannel，默认值为16，其他 Channel 默认值为1。默认值这样设置，是因为：ServerChannel 需要接受足够多的连接，保证大吞吐量，NioByteChannel 可以减少不必要的系统调用 select。
* **WRITE_SPIN_COUNT**：Netty 参数，一个 Loop 写操作执行的最大次数，默认值为16。也就是说，对于大数据量的写操作至多进行16次，如果16次仍没有全部写完数据，此时会提交一个新的写任务给 EventLoop，任务将在下次调度继续执行。这样，其他的写请求才能被响应不会因为单个大数据量写请求而耽误。
* **ALLOCATOR**：Netty 参数，ByteBuf 的分配器，默认值为 ByteBufAllocator.DEFAULT，4.0版本为 UnpooledByteBufAllocator，4.1版本为 PooledByteBufAllocator。该值也可以使用系统参数io.netty.allocator.type 配置，使用字符串值："unpooled"，"pooled"。
* **RCVBUF_ALLOCATOR**：Netty 参数，用于 Channel 分配接受 Buffer 的分配器，默认值为 AdaptiveRecvByteBufAllocator.DEFAULT，是一个自适应的接受缓冲区分配器，能根据接受到的数据自动调节大小。可选值为 FixedRecvByteBufAllocator，固定大小的接受缓冲区分配器。
* **AUTO_READ**：Netty 参数，自动读取，默认值为 True。Netty 只在必要的时候才设置关心相应的I/O事件。对于读操作，需要调用 channel.read() 设置关心的 I/O 事件为 OP_READ，这样若有数据到达才能读取以供用户处理。该值为 True 时，每次读操作完毕后会自动调用 channel.read()，从而有数据到达便能读取；否则，需要用户手动调用 channel.read()。需要注意的是：当调用 config.setAutoRead(boolean) 方法时，如果状态由 false 变为 true，将会调用 channel.read() 方法读取数据；由 true 变为 false，将调用 config.autoReadCleared()方法终止数据读取。
* **WRITE_BUFFER_HIGH_WATER_MARK**：Netty 参数，写高水位标记，默认值 64KB。如果Netty的写缓冲区中的字节超过该值，Channel的isWritable() 返回False。
* **WRITE_BUFFER_LOW_WATER_MARK**：Netty 参数，写低水位标记，默认值 32KB。当 Netty 的写缓冲区中的字节超过高水位之后若下降到低水位，则 Channel 的isWritable()返回True。写高低水位标记使用户可以控制写入数据速度，从而实现流量控制。推荐做法是：每次调用 channl.write(msg) 方法首先调用 channel.isWritable() 判断是否可写。
* **MESSAGE_SIZE_ESTIMATOR**：Netty 参数，消息大小估算器，默认为 DefaultMessageSizeEstimator.DEFAULT。估算 ByteBuf、ByteBufHolder 和 FileRegion 的大小，其中 ByteBuf和ByteBufHolder 为实际大小，FileRegion 估算值为0。该值估算的字节数在计算水位时使用，FileRegion 为 0 可知 FileRegion 不影响高低水位。
* **SINGLE_EVENTEXECUTOR_PER_GROUP**：Netty 参数，单线程执行 ChannelPipeline 中的事件，默认值为 True。该值控制执行 ChannelPipeline 中执行 ChannelHandler 的线程。如果为 Trye，整个 pipeline 由一个线程执行，这样不需要进行线程切换以及线程同步，是 Netty4 的推荐做法；如果为 False，ChannelHandler 中的处理过程会由 Group 中的不同线程执行。

**SocketChannel 参数**

* **SO_RCVBUF**：Socket 参数，TCP 数据接收缓冲区大小。该缓冲区即 TCP 接收滑动窗口，linux 操作系统可使用命令：cat /proc/sys/net/ipv4/tcp_rmem 查询其大小。一般情况下，该值可由用户在任意时刻设置，但当设置值超过 64KB 时，需要在连接到远端之前设置。
* **SO_SNDBUF**：Socket 参数，TCP 数据发送缓冲区大小。该缓冲区即 TCP 发送滑动窗口，linux 操作系统可使用命令：cat /proc/sys/net/ipv4/tcp_smem 查询其大小。
* **TCP_NODELAY**：TCP 参数，立即发送数据，默认值为 Ture（Netty 默认为 True 而操作系统默认为 False）。该值设置 Nagle 算法的启用，改算法将小的碎片数据连接成更大的报文来最小化所发送的报文的数量，如果需要发送一些较小的报文，则需要禁用该算法。Netty 默认禁用该算法，从而最小化报文传输延时。注意这个参数与是否开启 Nagle 算法是反着来的，true 表示关闭，false 表示开启。通俗地说，如果要求高实时性，有数据发送时就马上发送，就设置为 true；如果需要减少发送次数减少网络交互，就设置为 false。
* **SO_KEEPALIVE**：Socket 参数，连接保活，默认值为 False。启用该功能时，TCP 会主动探测空闲连接的有效性。可以将此功能视为 TCP 的心跳机制，需要注意的是：默认的心跳间隔是 7200s 即 2 小时。Netty 默认关闭该功能。
* **SO_REUSEADDR**：Socket 参数，地址复用，默认值 False。有四种情况可以使用
  * 当有一个有相同本地地址和端口的 socket1 处于 TIME_WAIT 状态时，而你希望启动的程序的 socket2 要占用该地址和端口，比如重启服务且保持先前端口。
  * 有多块网卡或用 IP Alias 技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同。
  * 单个进程绑定相同的端口到多个 socket 上，但每个 socket 绑定的 ip 地址不同。
  * 完全相同的地址和端口的重复绑定。但这只用于 UDP 的多播，不用于 TCP。
* **SO_LINGER**：Socket 参数，关闭 Socket 的延迟时间，默认值为 -1，表示禁用该功能。-1 表示 socket.close() 方法立即返回，但 OS 底层会将发送缓冲区全部发送到对端。0 表示 socket.close() 方法立即返回，OS 放弃发送缓冲区的数据直接向对端发送 RST 包，对端收到复位错误。非 0 整数值表示调用 socket.close() 方法的线程被阻塞直到延迟时间到或发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。
* **IP_TOS**：IP 参数，设置 IP 头部的 Type-of-Service 字段，用于描述 IP 包的优先级和 QoS 选项。
* **ALLOW_HALF_CLOSURE**：Netty 参数，一个连接的远端关闭时本地端是否关闭，默认值为 False。值为 False 时，连接自动关闭；为 True 时，触发 ChannelInboundHandler 的 userEventTriggered() 方法，事件为 ChannelInputShutdownEvent。

**ServerSocketChannel 参数**

* **SO_RCVBUF**：已说明，需要注意的是：当设置值超过 64KB 时，需要在绑定到本地端口前设置。该值设置的是由 ServerSocketChannel 使用 accept 接受的 SocketChannel 的接收缓冲区。
* **SO_REUSEADDR**：已说明
* **SO_BACKLOG**：Socket 参数，服务端接受连接的队列长度，如果队列已满，客户端连接将被拒绝。默认值， Windows 为 200，其他为 128。

**DatagramChannel参数**

* **SO_BROADCAST**：Socket 参数，设置广播模式。
* **SO_RCVBUF**：已说明
* **SO_SNDBUF**：已说明
* **SO_REUSEADDR**：已说明
* **IP_MULTICAST_LOOP_DISABLED**：对应 IP 参数 IP_MULTICAST_LOOP，设置本地回环接口的多播功能。由于 IP_MULTICAST_LOOP 返回 True 表示关闭，所以 Netty 加上后缀 _DISABLED 防止歧义。
* **IP_MULTICAST_ADDR**：对应 IP 参数 IP_MULTICAST_IF，设置对应地址的网卡为多播模式。
* **IP_MULTICAST_IF**：对应 IP 参数 IP_MULTICAST_IF2，同上但支持 IPV6。
* **IP_MULTICAST_TTL**：IP 参数，多播数据报的 time-to-live 即存活跳数。
* **IP_TOS**：已说明
* **DATAGRAM_CHANNEL_ACTIVE_ON_REGISTRATION**：Netty 参数，DatagramChannel 注册的 EventLoop 即表示已激活。

### attrs 属性

设置 Channel 的 attrs 属性，attrs 是用户自定义的 key-value 对，ServerBootstrap 和 Bootstrap 通用，对服务端而言会把 key-value 添加到 ServerSocketChannel，对客户端而言会把 key-value 添加到 SocketChannel。

> Bootstrap 在启动过程中会调用 initAndRegister() 方法创建 Channel，然后把 options 和 attrs 属性设置到 Channel 上。

使用 Bootstrap 的 attrs 属性添加的自定义 key-value 作为全局变量，被和该 Bootstrap 关联的所有 Channel 共用。

```
serverBootstrap.attr(AttributeKey.valueOf("key"), "value");
```

### handler 属性

把 handler 添加到 Channel 上，ServerBootstrap 和 Bootstrap 通用，对服务端而言会把 handler 添加到 ServerSocketChannel，对客户端而言会把 handler 添加到 SocketChannel。

如果只有单个 handler，可以直接添加

```
serverBootstrap.handler(new PacketDecoder());
```

如果有多个 handler，可以借助 NettyClientInitializer 类

```
serverBootstrap.handler(new ChannelInitializer<ServerSocketChannel>() {
    @Override
    protected void initChannel(ServerSocketChannel ch) throws Exception {
        // 解密
        ch.pipeline().addLast("packetDecoder", new PacketDecoder());
        // 子命令解码器
        ch.pipeline().addLast("subPacketDecoder", new SubPacketDecoder());
        // 返回封包
        ch.pipeline().addLast("packetEncoder", new PacketEncoder());
        // 子命令封包
        ch.pipeline().addLast("subPacketEncoder", new SubPacketEncoder());
        // 登陆
        ch.pipeline().addLast("loginSendHandler", new LoginSenderhandler());
    }
});
```

ChannelInitializer 是一种特殊的 ChannelHandler，负责把多个 Handler 和自己添加到 Channel 对应的 pipeline 中，最后把自己移除，这部分看源码可以看出来

    private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
        if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
            try {
                // 这个是抽象方法，我们在实例化 ChannelInitializer 类时必须实现的就是这个方法
                initChannel((C) ctx.channel());
            } catch (Throwable cause) {
                exceptionCaught(ctx, cause);
            } finally {
                // 移除多余的 ChannelHandler，具体见下
                remove(ctx);
            }
            return true;
        }
        return false;
    }
    
    private void remove(ChannelHandlerContext ctx) {
        try {
            ChannelPipeline pipeline = ctx.pipeline();
            if (pipeline.context(this) != null) {
                // 把本实例也就是 ChannelInitializer 实例从 pipeline 中移除
                pipeline.remove(this);
            }
        } finally {
            initMap.remove(ctx);
        }
    }
### childOptions、childAttrs、childHandler 属性

这四个属性是用来设置服务端的 workerLoopGroup 的属性，具体设置方法和没有带 child 前缀的属性基本相同，两者的区别是：不带 child 前缀的属性在程序初始化时就会执行，带 child 前缀的属性在客户端 connect 成功后才会执行。

# 实战

下面我们开始使用上面介绍的属性创建一个 Bootstrap 对象，示例化方式分为服务端 ServerBootstrap 和客户端 Bootstrap。

## 服务端

```
NioEventLoopGroup bossGroup = new NioEventLoopGroup();
NioEventLoopGroup workerGroup = new NioEventLoopGroup();
// 服务端启动类使用ServerBootstrap
ServerBootstrap bootstrap = new ServerBootstrap();
try {
   // 设置eventLoopGroup，服务端一般情况下会使用bossGroup和workerGroup两组NioEventLoopGroup
   bootstrap.group(bossGroup, workerGroup)
         // 设置channelFactory，服务端设置NioServerSocketChannel，表示这是NIO模式的TCP连接
         .channel(NioServerSocketChannel.class)
         // 设置localAddress，也可以不在这里设置，在下面的bind方法里面设置
         .localAddress(8000)
         // 设置option，SO_BACKLOG表示服务端接受连接的队列长度为100，如果队列已满，客户端连接将被拒绝。
         .option(ChannelOption.SO_BACKLOG, 100)
         // 设置attr
         .attr(AttributeKey.valueOf("key"), "value")
         // 设置handler，使用Netty自带的LoggingHandler作为acceptor
         .handler(new LoggingHandler())
         // 设置childOption，TCP_NODELAY表示不管数据大小有数据就发送，SO_KEEPALIVE表示开启心跳探测
         .childOption(ChannelOption.TCP_NODELAY, true)
         .childOption(ChannelOption.SO_KEEPALIVE, true)
         // 设置childAttr
         .childAttr(AttributeKey.valueOf("childKey"), "childValue")
         // 设置childHandler，因为有多个ChannelHandler，所以使用ChannelInitializer
         .childHandler(new ChannelInitializer<ServerSocketChannel>() {
            @Override
            protected void initChannel(ServerSocketChannel ch) throws Exception {
               // 解密
               ch.pipeline().addLast("packetDecoder", new PacketDecoder());
               // 子命令解码器
               ch.pipeline().addLast("subPacketDecoder", new SubPacketDecoder());
               // 返回封包
               ch.pipeline().addLast("packetEncoder", new PacketEncoder());
               // 子命令封包
               ch.pipeline().addLast("subPacketEncoder", new SubPacketEncoder());
               // 登陆
               ch.pipeline().addLast("loginSendHandler", new LoginSenderhandler());
            }
         });

   // 调用bind()启动服务器，并对启动结果增加回调监听
   ChannelFuture future = bootstrap.bind().sync().addListener(f -> {
      if(f.isSuccess()) {
         log.info("服务端启动成功");
      }
   });

   // 对服务端关闭增加回调监听
   future.channel().closeFuture().sync().addListener(f -> {
      if(f.isSuccess()) {
         log.info("服务端关闭服务");
      }
   });
} finally {
   // 销毁资源
   bossGroup.shutdownGracefully();
   workerGroup.shutdownGracefully();
}
```

**服务端启动流程代码解析**

1. 创建两组 NioEventLoopGroup 实例：bossGroup 和 workerGroup。
2. 创建 ServerBootstrap 实例。
3. 把 bossGroup 和 workerGroup 注册到 ServerBootstrap 实例。下面 4 ～ 8 为设置 bossGroup 实例属性，9 ～ 11 为设置 workerGroup 实例属性。
4. 设置 channelFactory，服务端一般配置为 NioServerSocketChannel.class。
5. 设置 localAddress，绑定本地端口 8000。
6. 设置 option，SO_BACKLOG 是服务端属性，表示服务端接受连接的队列长度为100，如果队列已满，客户端连接将被拒绝。
7. 设置 attr。
8. 设置 handler，直接使用 Netty 自带的 LoggingHandler 作为 acceptor。
9. 设置 childOption，TCP_NODELAY 表示不管数据大小有数据就发送，SO_KEEPALIVE 表示开启心跳探测。
10. 设置 childAttr。
11. 设置 childHandler，因为有多个 ChannelHandler，所以使用 ChannelInitializer。
12. ServerBootstrap 实例调用 `bind()` 启动服务器，并对启动结果增加回调监听。
13. 对服务端关闭增加回调监听
14. 服务端关闭后销毁资源

## 客户端

```
NioEventLoopGroup group = new NioEventLoopGroup();
// 客户端启动类使用Bootstrap
Bootstrap bootstrap = new Bootstrap();
try {
   // 设置eventLoopGroup，客户端只需要设置一组
   bootstrap.group(group)
         // 设置channelFactory，客户端设置NioSocketChannel，表示这是NIO模式的TCP连接
         .channel(NioSocketChannel.class)
         // 设置remoteAddress，也可以不在这里设置，在下面的connect方法里面设置
         .remoteAddress("192.168.10.1", 8000)
         // 设置option，TCP_NODELAY 表示不管数据大小有数据就发送，SO_KEEPALIVE表示开启心跳探测
         .option(ChannelOption.TCP_NODELAY, true)
         .option(ChannelOption.SO_KEEPALIVE, true)
         // 设置attr
         .attr(AttributeKey.valueOf("key"), "value")
         // 设置handler，因为有多个ChannelHandler，所以使用ChannelInitializer
         .handler(new ChannelInitializer<ServerSocketChannel>() {
            @Override
            protected void initChannel(ServerSocketChannel ch) throws Exception {
               // 解密
               ch.pipeline().addLast("packetDecoder", new PacketDecoder());
               // 子命令解码器
               ch.pipeline().addLast("subPacketDecoder", new SubPacketDecoder());
               // 返回封包
               ch.pipeline().addLast("packetEncoder", new PacketEncoder());
               // 子命令封包
               ch.pipeline().addLast("subPacketEncoder", new SubPacketEncoder());
               // 登陆
               ch.pipeline().addLast("loginSendHandler", new LoginSenderhandler());
            }
         });

   // 客户端调用connect()开始连接服务器，并增加回调监听
   ChannelFuture future = bootstrap.connect().sync().addListener(f -> {
      if(f.isSuccess()) {
         log.info("客户端连接成功");
      }
   });

   // 断开服务器增加回调监听
   future.channel().closeFuture().sync().addListener(f -> {
      if(f.isSuccess()) {
         log.info("客户端连接断开");
      }
   });
} finally {
   // 销毁资源
   group.shutdownGracefully();
}
```

**客户端启动流程代码解析**

客户端的启动代码相对于服务端就简单很多

1. 创建一个 NioEventLoopGroup 实例：group
2. 创建 Bootstrap 实例。
3. 把 group 注册到 Bootstrap 实例，下面 4 ～ 8 为设置 group 实例属性。
4. 设置 channelFactory，客户端一般设置 NioSocketChannel.class。
5. 设置 remoteAddress，设置远程主机地址和端口。
6. 设置 option，TCP_NODELAY 表示不管数据大小有数据就发送，SO_KEEPALIVE 表示开启心跳探测。
7. 设置 attr。
8. 设置 handler，因为有多个 ChannelHandler，所以使用 ChannelInitializer。
9. Bootstrap 实例使用 `connect()` 连接远程服务器，并对连接结果增加回调监听。
10. 对客户端连接关闭增加回调监听。
11. 客户端关闭后销毁资源。

# 总结

本文介绍了 Netty 的启动类 Bootstrap。Bootstrap 按用途分为 ServerBootstrap 和 Bootstrap，分别对应服务端和客户端。Bootstrap 的常用属性有：

* group
* channelFactory
* localAddress、remoteAddress
* options
* attrs
* handler
* childOptions、childAttrs、childHandler

其中 localAddress、childOptions、childAttrs 和 childHandler是服务端属性，remoteAddress 是客户端属性，其他两者通用。最后通过实战例子分别分析 ServerBootstrap 和 Bootstrap 的具体启动过程。

# 参考

[《Netty In Action》]()

[ Netty服务器启动过程](http://www.kancloud.cn:8080/ssj234/netty-source/433213)

