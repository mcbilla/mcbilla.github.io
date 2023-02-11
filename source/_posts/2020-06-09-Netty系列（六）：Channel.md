---
title: Netty系列（六）：Channel
date: 2020-06-09 11:50:55
tags:
  - Netty
categories:
  - Netty
---

> 上一篇介绍了 Netty 的线程模型 EventLoop 和 EventLoopGroup，本文开始介绍 Netty 的传输流组件 Channel 以及相关组件 ChannelHandler、ChannelHandlerContext、ChannelPipline 和 ChannelFuture 等。

<!--more-->

## Channel

### Channel是什么

Channel 是一个管道，用于连接字节缓冲区 Buf 和另一端的实体，这个实例可以是 Socket，也可以是 File，在 NIO 网络编程模型中，服务端和客户端进行 IO 数据交互(得到彼此推送的信息)的媒介就是 Channel。

Netty 对 NIO 的原生的 Channel 进行了封装和增强封装成 NioXXXChannel，相对于原生的 Channel，Netty 的 Channel 增加了如下的组件：

- id 标识唯一身份信息
- 可能存在的 parent Channel
- 管道 pepiline
- 用于数据读写的 unsafe 内部类
- 关联上相伴终生的 NioEventLoop

### Channel的工作流程

Channel 本身做的事情不多，主要通过其包含的成员完成大量工作，包含的主要成员如下：

- EventLoop：每个 Channel 整个生命周期都绑定到一个特定的 EventLoop。
- Unsafe：承担 Channel 网络相关的功能，例如真正的网络读写操作。
- DefaultChannelPipeLine：每个 Channel 绑定到一个 ChannelPipeLine，Channel 在生命周期的大部分动作都通过调用 ChannelPipeLine 的方法来完成。

Channel 通过调用成员，可以完成以下工作流程： 

* 一旦用户端连接成功，将新建一个 Channel 同该用户端进行绑定。
* Channel 从 EventLoopGroup 获得一个 EventLoop，并注册到该 EventLoop，Channel 生命周期内都和该 EventLoop 在一起（注册时获得 selectionKey）。
* Channel 同用户端进行网络连接、关闭和读写，生成相对应的 event（改变 selectinKey 信息），触发eventloop调度线程进行执行。

### Channel的分类

Netty 中的 Channel 有很多种类型。

从 IO 类型来说，分为同步（Oio）和异步（Nio）两种类型。一般情况下我们都会使用异步的 Channel，特殊情况下需要切换成同步通讯时，我们只需要修改 Channel 的类型即可，所以在 Netty 通讯中同步和异步的切换非常灵活。

从数据传输类型来说，分为按事件消息传递（Message）以及按字节传递（Byte）两种类型。

从使用角色类型来说，分为服务器（ServerSocket）以及客户端（Socket）两种。

从使用协议类型来说，分为 TCP、UDT 和 SCTP 协议等。

Channel 的具体实现类如下：

- NioSocketChannel：异步的客户端 TCP Socket 连接
- NioServerSocketChannel：异步的服务器端 TCP Socket 连接
- NioDatagramChannel：异步的 UDP 连接
- NioSctpChannel：异步的客户端 Sctp 连接
- NioSctpServerChannel：异步的 Sctp 服务器端连接
- OioSocketChannel：同步的客户端 TCP Socket 连接
- OioServerSocketChannel：同步的服务器端 TCP Socket 连接
- OioDatagramChannel：同步的 UDP 连接
- OioSctpChannel：同步的 Sctp 服务器端连接
- OioSctpServerChannel：同步的客户端 TCP Socket 连接

### Channel 整体结构图

![channel1](channel1.png)

一个 Channel 里面包含 ChannelPipeLine、ChannelHandler 和 ChannelHandlerContext 等组件，组件间的关系如下：

* 一个 Channel 绑定到一个 ChannelPipeLine 上，Channel 的大部分功能都通过调用 ChannelPipeLine 的方法来完成。
* 一个 ChannelPipeLine 通过 filter 的模式把一到多个 ChannelHandler 组织到一起，ChannelHandler 之间不会产生直接关联。
* 把一个 ChannelHandler 添加到 ChannelPipeLine 的时候，会生成一个 ChannelHandlerContext 和该 ChannelHandler 对应。ChannelHandlerContext 是一个双向链表，使前后的 ChannelHandler 产生关联。

因为 ChannelHandler 都是通过 ChannelHandlerContext 产生关联 ，所以事件在 Channel 中进行传播的过程，实际上是通过 ChannelHandlerContext 来完成的。

![channel2](channel2.png)

下面我们简单介绍一下 Channel 的主要组件。

## ChannelPipline

ChannelPipline 可以理解为拦截流经 Channel 的入站和出站事件的 **ChannelHandler实例链** ，ChannelPipline 的官方结构图如下：

```
                                    I/O Request  via Channel or  ChannelHandlerContext
                                                  |
+---------------------------------------------------+---------------+
|                           ChannelPipeline         |               |
|                                                  \|/              |
|    +---------------------+            +-----------+----------+    |
|    | Inbound Handler  N  |            | Outbound Handler  1  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
|               |                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  .               |
|               .                                   .               |
| ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
|        [ method call]                       [method call]         |
|               .                                   .               |
|               .                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
|               |                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler  1  |            | Outbound Handler  M  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
+---------------+-----------------------------------+---------------+
              |                                  \|/
+---------------+-----------------------------------+---------------+
|               |                                   |               |
|       [ Socket.read() ]                    [ Socket.write() ]     |
|                                                                   |
|  Netty Internal I/O Threads (Transport Implementation)            |
+-------------------------------------------------------------------+
```

可以看到，流经 ChannelPipline 的事件分为 Inbound 和 Outbound 两种类型，Inbound 事件的流向是从下至上，而 Outbound 刚好相反。Inbound 的传递方式是通过调用相应的 **ChannelHandlerContext.fireIN_EVT()** 方法，而 Outbound 方法的的传递方式是通过调用 **ChannelHandlerContext.OUT_EVT()** 方法。例如 **ChannelHandlerContext.fireChannelRegistered()** 调用会发送一个 **ChannelRegistered** 的 Inbound 事件给下一个ChannelHandlerContext，而 **ChannelHandlerContext.bind** 调用会发送一个 **bind** 的 Outbound 事件给 下一个 ChannelHandlerContext。

在 ChannelPipeline 传播事件时，它会测试 ChannelPipeline 中的下一个 ChannelHandler 的类型是否和事件的运动方向相匹配。如果不匹配，ChannelPipeline 将跳过该 ChannelHandler 并前进到下一个，直到它找到和该事件所期望的方向相匹配的为止。

DefaultChannelPipeline 是 ChannelPipeline 的默认实现类，新建 Channel 时默认，部分源码如下：

```
final class DefaultChannelPipeline implements ChannelPipeline {
    private static final WeakHashMap<Class<?>, String>[] nameCaches = new WeakHashMap[Runtime.getRuntime().availableProcessors()];

    final AbstractChannel channel;

    final DefaultChannelHandlerContext head;
    final DefaultChannelHandlerContext tail;

    private final Map<String, DefaultChannelHandlerContext> name2ctx = new HashMap<String, DefaultChannelHandlerContext>(4);

    final Map<EventExecutorGroup, ChannelHandlerInvoker> childInvokers = new IdentityHashMap<EventExecutorGroup, ChannelHandlerInvoker>();
    //...
}
```

在 DefaultChannelPipeline 构造器中，首先将与之关联的 Channel 保存到字段 channel 中，然后实例化两个 ChannelHandlerContext，一个是 HeadContext 实例 head，另一个是 TailContext 实例 tail。接着将 head 和 tail 互相指向，构成一个双向链表。**这个链表是 Netty 实现 Pipeline 机制的关键**。

## ChannelHandler

ChannelHandler 是 Channel 的核心，**数据编解码和所有的业务逻辑都需要通过自定义 ChannelHandler 来实现**。

ChannelHandler 分为 ChannelInboundHandler 和 ChannelOutboundHandler：

* ChannelInboundHandler：处理入站事件，如链路建立、链路关闭或者读完成等事件。
* ChannelOutboundHandler：处理出站事件，如由用户线程或者代码发起的 IO 操作事件。

其中 ChannelOutboundHandler 用的比较少（Netty 5 后面已经有废弃的趋势），下面介绍下 ChannelInboundHandler 的生命周期：

![image-20200701181542816](/Users/mochuangbiao/Library/Application%20Support/typora-user-images/image-20200701181542816.png)

1. handlerAdded：新建立的连接会按照初始化策略，把 handler 添加到该 channel 的 pipeline 里面，也就是 channel.pipeline.addLast(new LifeCycleInBoundHandler) 执行完成后的回调；
2. channelRegistered：当该连接分配到具体的 worker 线程后，该回调会被调用。
3. channelActive：channel 的准备工作已经完成，所有的 pipeline 添加完成，并分配到具体的线上上，说明该 channel 准备就绪，可以使用了。
4. channelRead：客户端向服务端发来数据，每次都会回调此方法，表示有数据可读；
5. channelReadComplete：服务端每次读完一次完整的数据之后，回调该方法，表示数据读取完毕；
6. channelInactive：当连接断开时，该回调会被调用，说明这时候底层的TCP连接已经被断开了。
7. channelUnREgistered：对应 channelRegistered，当连接关闭后，释放绑定的workder线程；
8. handlerRemoved：对应 handlerAdded，将 handler 从该 channel 的 pipeline 移除后的回调方法。

ChannelInboundHandler 的生命周期里面除了正常的读写流程外， 还提供了定时处理和异常处理的方法：

1. userEventTriggered：处理心跳超时事件，在 IdleStateHandler 设置超时时间，达到了超时时间就会直接调用该方法。
2. exceptionCaught：处理连接出现的异常情况。

## ChannelHandlerContext

ChannelHandlerContext 代表了 ChannelHandler 和 ChannelPipeline 之间的关联，每当有 ChannelHandler 添加到 ChannelPipeline 中时，都会创建 ChannelHandlerContext。**ChannelHandlerContext 的主要功能是管理它所关联的 ChannelHandler 和在同一个 ChannelPipeline 中的其他 ChannelHandler 之间的交互**。

我们可以通过 ChannelHandlerContext 来获取整个 Channel 的上下文属性。例如通过 ChannelHandlerContext 来访问 Channel：

```
//获取到与 ChannelHandlerContext相关联的 Channel 的引用
ChannelHandlerContext ctx = ..;
Channel channel = ctx.channel();
//通过 Channel 写入缓冲区
channel.write(Unpooled.copiedBuffer("Netty in Action", CharsetUtil.UTF_8));
```

通过 ChannelHandlerContext 访问 ChannelPipeline：

```
//获取到与 ChannelHandlerContext相关联的 ChannelPipeline 的引用
ChannelHandlerContext ctx = ..;
ChannelPipeline pipeline = ctx.pipeline();
//通过 ChannelPipeline写入缓冲区
pipeline.write(Unpooled.copiedBuffer("Netty in Action", CharsetUtil.UTF_8));
```

最开始的时候 ChannelPipeline 中含有 head 和 tail 两个 ChannelHandlerContext(同时也是 ChannelHandler)，但是这个 Pipeline并不能实现什么特殊的功能，因为我们还没有给它添加自定义的 ChannelHandler。通常来说，我们在初始化 Bootstrap，会添加我们自定义的 ChannelHandler。

如果你在 Channel 或者 ChannelPipeline 实例上调用这些方法，它们的调用会穿过整个 pipeline。而在 ChannelHandlerContext 上调用的同样的方法，仅仅从当前 ChannelHandler 开始，走到 pipeline 中下一个可以处理这个 event 的 ChannelHandler。例如调用 `ctx.writeAndFlush`，是从当前 handler 直接发出这个消息，而调用 `channel.writeAndFlush` 是从整个 pipline 最后一个 outhandler 发出消息。

## ChannelFuture、ChannelPromise

ChannelFuture 的作用是用来保存 Channel 异步操作的结果。

在 Netty 中所有的 I/O 操作都是异步的。这意味着任何的 I/O 调用都将立即返回，而不保证这些被请求的 I/O 操作在调用结束的时候已经完成。取而代之地，你会得到一个返回的 ChannelFuture 实例，这个实例将给你一些关于 I/O 操作结果或者状态的信息。下面是官方关于 ChannelFuture 的状态迁移图：

```
                                       +---------------------------+
                                       | Completed successfully    |
                                       +---------------------------+
                                  +---->      isDone() = true      |
  +--------------------------+    |    |   isSuccess() = true      |
  |        Uncompleted       |    |    +===========================+
  +--------------------------+    |    | Completed with failure    |
  |      isDone() = false    |    |    +---------------------------+
  |   isSuccess() = false    |----+---->      isDone() = true      |
  | isCancelled() = false    |    |    |       cause() = non-null  |
  |       cause() = null     |    |    +===========================+
  +--------------------------+    |    | Completed by cancellation |
                                  |    +---------------------------+
                                  +---->      isDone() = true      |
                                       | isCancelled() = true      |
                                       +---------------------------+
```

ChannelFuture 有 `Completed` 和 `Uncompleted` 两种父状态，父状态下分为 `isDone`、`isSuccess`、`isCancelled` 和 `cause` 四种子状态

* ChannelFuture 新创建的时候是 `Uncompleted` 状态，这个时候子状态 为`isDone`、`isSuccess` 和 `isCancelled`  都是 false，`cause` 是一个 null 值。
* 当 ChannelFuture 变成 `completed` 状态并且结果成功，`isDone` 和 `isSuccess`  变成 true。
* 当 ChannelFuture 变成 `completed` 状态并且结果失败，`isDone` 变成 true，`cause` 为非 null 值。
* 当 ChannelFuture 变成 `completed` 状态并且结果取消，`isDone` 和 `isCancelled`  变成 true。

我们创建 ChannelFuture 一般会使用 `addListener(GenericFutureListener)` 为 ChannelFuture 添加监听器，监听 ChannelFuture 的结果状态变化。

```
ChannelFuture future = bootstrap.connect(serverIp, serverPort).sync();
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        log.info("连接服务器成功");
    }
});
```

ChannelPromise 是 ChannelFuture 的子类，ChannelFuture 是只读的，而 ChannelPromise 是可写的，提供了 `setSuccess` 和 `setFailure` 等方法对结果进行修改。

## 总结

* Channel 是 Netty 的通讯载体，是对 NIO 原生 Channel 的二次封装，提供更完善的功能。
* Channel 有很多类型，比较常用的有 NioSocketChannel 和 NioServerSocketChannel。
* 一个 Channel 在创建之后整个生命周期都绑定到一个特定的 EventLoop。
* 一个 Channel 对应一个 ChannelPipeLine。
* 一个 ChannelPipeLine 通过 filter 的模式把一到多个 ChannelHandler 组织到一起，ChannelHandler 之间不会产生直接关联。
* 每个 ChannelHandler 对应一个 ChannelHandlerContext，通过 ChannelHandlerContext 使前后的 ChannelHandler 产生关联。
* ChannelFuture 的作用是用来保存 Channel 异步操作的结果，ChannelPromise 提供了可写的方法。

## 参考

[《Netty in Action》]()

[Netty Channel的生命周期](https://jianyuan.fun/post/netty-channel-lifecycle/)

[Netty核心概念(5)之Channel](https://www.cnblogs.com/lighten/p/8950347.html)

[netty中的Channel、ChannelPipeline](https://www.cnblogs.com/duanxz/p/3724247.html)