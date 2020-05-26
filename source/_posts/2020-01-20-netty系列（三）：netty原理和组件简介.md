---

title: netty系列（三）：netty原理和组件简介
tags:
  - Netty
categories:
  - Netty
date: 2020-01-20 16:50:22
---


> 在上一篇文章我们了解到网络编程的相关知识以及 Netty 给网络编程开发带来的便利性，本文我们介绍 Netty 的基础线程模型 Reactor 模型、Netty 组件和基于这两者的 Netty 线程模型，有助于理解 Netty 整体原理结构。

<!--more-->

本文首先介绍 Reactor 模型的发展历史，该线程模型是 Netty 组件设计和线程模型的基础。然后介绍 Netty 基于 Reactor 模型设计的各种组件，最后介绍 Netty 组件组成的 Netty 线程模型。

# Reactor 模型

Reactor 模型是一个 IO 设计模式，Java 中的 NIO 就对  Reactor 模式提供了很好的支持，比较著名的就是 [Doung Lea](https://baike.baidu.com/item/Doug Lea/6319404) 大神在 [《Scalable IO in Java》](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)演示如何使用 NIO 实现Reactor模式。

在维基百科上对  Reactor 模式定义如下：

> The reactor design pattern is an **event handling pattern** for handling service requests **delivered concurrently to a service handler** by one or more inputs. The service handler then **demultiplexes** the incoming requests and dispatches them synchronously to the associated request handlers

从这段定义我们可以得到三个关键点：

* 事件驱动（event handling）
* 可以处理一个或多个输入源（one or more inputs）
* 通过 Service Handler 同步的将输入事件（Event）采用多路复用分发给相应的 Request Handler（多个）处理

基本流程图如下：

<div style="width: 80%; margin: auto">![base](base.png)</div>

对流程图进行进一步抽象，得到 Reactor 模型的 OMT 类图如下：

<div style="width: 80%; margin: auto">![omt](omt.png)</div>

五种类解释如下：

* Handle(描述符)：表示触发事件，是触发所有操作的发源地。在 Linux 中 Handle 就是文件描述符 fd。

* Synchronous Event Demultiplexer(同步事件分离器)：事件发生后通知 Initiation Dispatcher，对于 Linux 来说，同步事件分离器指的就是常用的 I/O 多路复用机制，比如说 select、poll、epoll 等。在 Java NIO 领域中，同步事件分离器对应的组件就是 Selector，对应的阻塞方法就是 select 方法。

* Initiation Dispatcher(初始分发器)：相当于 Reactor 的角色，Initiation Dispatcher 会通过 Synchronous Event Demultiplexer 来等待事件的发生。一旦事件发生，Initiation Dispatcher 会调用 Event Handler 来处理事件。
* Event Handler(事件处理器)：事件产生时实现相应的回调方法进行业务逻辑，类似于接口。
* Concrete Event Handler(具体事件处理器)： Event Handler的实现。

对 OMT 类图进行简化后，Reactor 模型定义了三种角色：

- **Reactor**: 负责监听和响应事件，将事件分发绑定了该事件的 Handler 处理
- **Handler**: 事件处理器，绑定了某类事件，负责对事件进行处理
- **Acceptor**：Handler 的一种，绑定了 connect 事件，当客户端发起 connect 请求时，Reactor 会将 accept 事件分发给 Acceptor 处理

这三种角色也是我们下面介绍的三种模型的基础。

## Reactor 单线程模型

<div style="width: 80%; margin: auto">![reactor1](reactor1.png)</div>

**处理流程**

1. Reactor 对象通过 select 监控连接事件，收到事件后通过 dispatch 进行分发
2. 如果是连接建立的事件，则交由 Acceptor 通过 accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理
3. 如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应
4. Handler 会完成 read -> 业务处理 -> send 的完整业务流程

**特点分析**

* 优点：模型简单，没有多线程，进程通信，竞争的问题，全部都在一个线程中完成。Redis 就是使用 Reactor 单进程的模型。
* 缺点：所有的 IO 操作（read、send）和非 IO 操作（decode、compute、encode）都在一个线程里面完成，当非 IO 操作处理速度较慢时，会导致 IO 响应速度严重下降。

## Reactor 多线程模型

<div style="width: 80%; margin: auto">![reactor2](reactor2.png)</div>

**处理流程**

1. 主线程中，Reactor 对象通过 select  监听连接事件，收到事件后通过 dispatch 进行分发。
2. 如果是连接建立的事件，则由 Acceptor 处理，Acceptor 通过 accept 接受连接，并创建一个 Handler 来处理连接后续的各种事件。
3. 如果不是连接建立事件，则 Reactor 会调用连接对应的 Handler 来进行相应。
4. Handler 只负责请求响应事件，不进行业务处理。Handler 通过 read 读取到数据后，会发给业务线程池 Thread Pool 进行业务处理。
5. Thread Pool 会在独立的子线程中完成真正的业务处理，然后将响应结果返回给 Handler。
6. Handler 收到响应后通过 send 将响应结果返回给 client。

**特点分析**

* 优点：和单线程模型相比，多线程模型最大的特点就是把非 IO 操作（decode、compute、encode）抽取出来放到专门的业务线程池去进行处理，Handler 只需要负责 IO 操作（read、send），这样会大大提升系统的 IO 响应速度。
* 缺点：这个模型仍然把管理连接的 acceptor 和负责 IO 的 Handler 放在同一个 Reactor 中。在瞬间高并发的场景中，系统可能会因为忙于处理新的连接，导致 IO 响应速度瞬间下降。

## 主从 Reactor 多线程模型

<div style="width: 80%; margin: auto">![reactor3](reactor3.png)</div>

**处理流程**

1. 主进程中 mainReactor 对象通过 select 监控连接建立事件，收到事件后通过 Acceptor 接收，将新的连接分配给子进程 subReactor（可以有多个）。
2. 子进程中的 subReactor 将 mainReactor 分配的连接加入队列进行监听，并把该连接从 mainReactor 的管理队列中移除，然后创建一个 Handler 用于处理连接的各种事件
3. 当有新的事件发生时，subReactor 会调用里连接对应的 Handler 来进行处理。
4. Handler 通过 Read 读取数据后，会分发给后面的 Thread Pool 线程池进行业务处理。
5.  Thread Pool 线程池会分配独立的线程完成真正的业务处理，然后将响应结果返回给 Handler。
6. Handler 收到响应结果后通过 send 将响应结果返回给 client。

**特点分析**

和多线程模型相比，该模型将 Reactor 分成两部分，mainReactor 只负责管理连接（accept），subReactor 负责管理除了连接外的其他操作（read、decode、compute、encode、send）。mainReactor 和 subReactor 相互独立，之间的交互非常简单，mainReactor 只需要把连接传给 subReactor 就完成任务了。Reactor 具有以下特点：

* 响应快，不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的
* 编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销
* 可扩展性，可以方便地通过增加 Reactor 实例个数来充分利用 CPU 资源；
* 可复用性，Reactor模型本身与具体事件处理逻辑无关，具有很高的复用性。

主从 Reactor 多线程模型是 Netty 线程模型的基础，Netty 的主要组件都是围绕该模型进行设计的。

# Netty 组件

## Bootstrap

Bootstrap 意思是引导，一个 Netty 应用通常由一个 Bootstrap 开始，主要作用是配置整个 Netty 程序，串联各个组件。Bootstrap 分为 Bootstrap 和，ServerBootstrapNetty， Bootstrap 类是客户端程序的启动引导类，ServerBootstrap 是服务端启动引导类。

## EventLoop、EventLoopGroup

一个 EventLoop 对应一个线程，可以绑定多个 Channel，内部会维护一个 selector 和 taskQueue。

* selector：处理 IO 任务，即 selectionKey 中 ready 的事件，如 accept、connect、read、write 等。
* taskQueue：处理非 IO 任务，如 register0、bind0 等任务。

两种任务的执行时间比由变量 ioRatio 控制，默认为 50，则表示允许非 IO 任务执行的时间与 IO 任务的执行时间相等。

一个 EventLoopGroup 可以理解为一个线程池，包含一到多个 EventLoop。

## Channel

Channel 是 Netty 网络操作的基础组件，可以把他理解为 BIO 的 Socket 类和 NIO 的 Channel 的升级版，把相关 api 进行进一步封装，提供了基本的 I/O 操作如 bind、connect、read、write 等，并且将 Netty 的相关功能类（例如 EventLoop）聚合在 Channel 中，由 Channel 统一负责和调度（例如可以通过 channel.eventLoop() 获取 channel 所在的 EventLoop 实例），可以功能实现更加灵活。

一个 Channel 会注册到一个 EventLoop 上，然后在它的整个生命周期过程中，都会用这个 EventLoop。

用 Netty 编写网络程序的时候，我们一般不会直接操纵 Channel，而是直接操作 Channel 的组件例如最常用的 ChannelHandler。

Channel 涉及的组件有 ChannelHandler、ChannelHandlerContext、ChannelPipline 和 ChannelFuture。这些组件的关系如下：

<div style="width: 80%; margin: auto">![channel](channel.png)</div>

### ChannelHandler

ChannelHandler 是 Netty 的核心组件，用来处理各种事件，例如连接、接收、数据转换、异常处理以及我们的业务逻辑逻辑。ChannelHandler 包括两个核心子类：

* ChannelInboundHandler：用于接收、处理入站数据和事件
* ChannelOutboundHandler：用于接收、处理出站数据和事件

一个 Channel 可以包含多个 ChannelHandler，这些 ChannelHandler 呈流水线式处理事件。一个 ChannelHandler 也可以被多个 Channel 复用。

### ChannelHandlerContext

Channel 的上下文，一个 ChannelHandlerContext 对应一个 ChannelHandler，并且与前后的 ChannelHandler 的 ChannelHandlerContext 产生关联。可以把一些数据放到 ChannelHandlerContext 中进行上下文传递。

### ChannelPipline

ChannelPipline 是 ChannelHandler 的链表，负责把多个 ChannelHandler 串行起来，提供了一种截取过滤模式（类似 serverlet 中的 filter 功能），拦截处理 Channel 的输入输出 event。

当一个数据流进入 ChannlePipeline 时，它会从 ChannelPipeline 头部开始传给第一个 ChannelInboundHandler ，当第一个处理完后再传给下一个，一直传递到管道的尾部。与之相对应的是，当数据被写出时，它会从管道的尾部开始，先经过管道尾部的 “最后” 一个ChannelOutboundHandler，当它处理完成后会传递给前一个 ChannelOutboundHandler 。

一个 Channel 包含一个 ChannelPipline。当 ChannelHandler 被添加到 ChannelPipeline 时，它将会被分配一个 ChannelHandlerContext，它代表了 ChannelHandler 和 ChannelPipeline 之间的绑定。ChannelPipeline 通过 ChannelHandlerContext 来间接管理 ChannelHandler。

### ChannelFuture、ChannelPromise

在 Netty 中所有的 IO 操作都是异步的，不能立刻得知消息的处理处理。这时候我们通过 ChannelFuture 或 ChannelPromise，对结果注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件。ChannelPromise 是 ChannelFuture 的扩展，可以设置结果。

## ByteBuf

ByteBuf 是 Netty 的数据容器，本质是一个由不同索引分别控制读访问和写访问的字节数组。Netty 的 ByteBuf 对 NIO 的 ByteBuffer 进行封装，提供更友好的 api 和更强大的功能。

## 组件关系

以上我们介绍的组件有以下关系：

* 一个 EventLoopGroup 包含一个或多个 EventLoop。
* 一个 EventLoop 在它的生命周期内只能与一个 Thread 绑定。
* 所有有 EnventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理。
* 一个 Channel 在它的生命周期内只能注册与一个 EventLoop。
* 一个 EventLoop 可被分配至一个或多个 Channel 。
* 一个 Channel 对应一个 ChannelPipline，一个 ChannelPipline 包含多个 ChannelHandler，每个 ChannelHandler 对应一个 ChannelHandlerContext，与前后 ChannelHandler 的 ChannelHandlerContext 产生关联。

# Netty 线程模型

根据 Reactor 模型和 Netty 的基本组件，我们可以得到 Netty Server 端的线程模型如下：

<div style="width: 80%; margin: auto">![netty](netty.png)</div>

**线程模型分析**

* Server 端包含 1 个 Boss NioEventLoopGroup 和 1 个 Worker NioEventLoopGroup。
* 每个 Boss NioEventLoopGroup 通常包含 1 个 NioEventLoop，1 个 NioEventLoop 包含 1 个 Selector 和 1 个事件循环线程。Boss NioEventLoopGroup 的工作过程如下：
  * 轮询 Accept 事件。
  * 处理 Accept I/O 事件，与 Client 建立连接，生成 NioSocketChannel，并将 NioSocketChannel 注册到某个 Worker NioEventLoop 的 Selector 上。
  * 处理任务队列中的任务，runAllTasks。任务队列中的任务包括用户调用 eventloop.execute 或 schedule 执行的任务，或者其他线程提交到该 eventloop 的任务。
* 每个 Worker NioEventLoopGroup 通常包含多个 NioEventLoop。Worker NioEventLoopGroup 的工作过程如下：
  * 轮询 Read、Write 事件。
  * 处理 I/O 事件，即 Read、Write 事件，在 NioSocketChannel 可读、可写事件发生时进行处理。
  * 处理任务队列中的任务，runAllTasks。

该线程模型是 netty 框架的基础，能清晰地认识该模型，对理解 netty 的整体结构设计非常重要。

# 总结

Reactor 模型是一个 IO 设计模式，发展衍生出三种模型：

* Reactor 单线程模型
* Reactor 多线程模型
* 主从 Reactor 多线程模型

Netty 根据主从 Reactor 多线程模型设计出多种基础组件：

* Bootstrap
* EventLoop、EventLoopGroup
* Channel
  * ChannelHandler
  * ChannelHandlerContext
  * ChannelPipline
  * ChannelFuture、ChannelPromise
* ByteBuf

Netty 的线程模型就是基于主从 Reactor 多线程模型和 Netty 基础组件进行设计的，该线程模型是 Netty 整体结构的基础。

# 参考

[《Netty In Action》]()

[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)

[Java NIO 系列文章之 浅析Reactor模式](https://juejin.im/post/5ba3845e6fb9a05cdd2d03c0)

[彻底搞懂Reactor模型和Proactor模型](https://cloud.tencent.com/developer/article/1488120)