---
title: Netty系列（五）：EventLoop和EventLoopGroup
tags:
  - Netty
categories:
  - Netty
date: 2020-05-28 17:33:07
---

> 上一篇文章介绍了 Netty 的启动器 Bootstrap，这篇开始介绍 Netty 的线程机制 EventLoop 和 EventLoopGroup，包括 EventLoop 和 EventLoopGroup 的简介、类继承结构和实例化过程。

<!--more-->

# EventLoop 和 EventLoopGroup 是什么

Netty 的非阻塞线程模型的实现基于 NIO Reactor 模型，其中 EventLoop 和 EventLoopGroup 是该线程模型的核心组件。当然 Netty 也提供了阻塞线程模型的实现，在这里对阻塞模型不做介绍。

一个 EventLoop 只包含一个死循环的线程，用于处理绑定到该 EventLoop 的所有 Channel 的 IO 操作和业务操作。一个 EventLoop 包含一到多个 Channel。

一个 EventLoopGroup 包含一到多个 EventLoop，EventLoopGroup 负责将 EventLoop 分配给每个新创建的 Channel。

EventLoop、EventLoopGroup 和 Channel 的整体关系图如下：

<div style="width: 80%; margin: auto">![relation](relation.png)</div>
# EventLoop

EventLoop 是用来执行任务的单线程，任务包括周期性的 IO 任务（accept、read、write等）和用户自定义任务（decode、compute、encode）等。为了清晰了解 EventLoop 的内部结构，下面我们从 EventLoop 接口的继承结构开始介绍。

## EventLoop 继承结构

<div style="width: 50%; margin: auto">![Eventloop](Eventloop.png)</div>
在这个类图里面，EventExecutorGroup 上层是  JDK 提供的并发包接口，下层是 Netty 提供的抽象类和实现。EventLoop 的继承结构这么复杂，有一部分原因是为了使用父接口或父类里面的部分特性或者方法而已，和父接口或父类的用途定位并没有太多的关联。下面我们按接口用途分组介绍：

* Executor、ExecutorService、ScheduledExecutorService
* Iterable
* EventExecutorGroup
* EventExecutor、OrderedEventExecutor
* EventLoopGroup、EventLoop

### Executor、ExecutorService、ScheduledExecutorService

这三个都是 java.util.concurrent 里面提供的接口，是 Java5 之后引进的，基本定义了 Java 的线程池机制，用来控制线程的启动、执行和关闭。

Executor 是最基础的父接口，只定义了一个方法 `execute(Runnable command)`，用来执行一个 Runable 实例。

ExecutorService 接口继承自 Executor 接口，它提供了更丰富的实现多线程的方法。例如 `submit(Runnable task)` 用于提交任务并返回异步执行结果 Future，`shutdown()` 用于平滑地关闭线程池（先拒绝接收新的任务，然后等待已接收的任务全部执行完毕，再关闭线程池）。

ScheduledExecutorService 接口继承自 ExecutorService 接口，是一种用于延时和定期执行任务的特殊线程池。EventLoop 继承这个接口是为了继承这里面的延时或周期执行的方法，在 Netty 中，有可能用户线程和 I/O 线程同时操作网络资源，而为了减少并发锁竞争，Netty 将用户线程的任务包装成 task，然后像 Netty 的 I/O 任务一样延期执行，另外有些时候需要周期性执行任务例如心跳检测。

### Iterable

上面我们谈到 EventLoopGroup 包含一到多个 EventLoop，实现 Iterable 接口可以对所有 EventLoop 进行遍历。

### EventExecutorGroup

EventExecutorGroup 是 Netty 自定义线程模型的基础接口，下面是接口方法定义

<div style="width: 50%; margin: auto">![EventExecutorGroup](EventExecutorGroup.png)</div>
我们可以看到 EventExecutorGroup 是对继承的接口的大部分方法进行 Override，并自定义了部分方法。EventExecutorGroup 提供的功能有：

* 提供线程池的基础功能，例如 `submit` 提交任务功能，并废弃了原始的关闭线程池接口 `shutdown()`，自定义了关闭线程池接口 `shutdownGracefully()`。
* 提供延时和周期执行的功能。
* 提供迭代功能。

### EventExecutor、OrderedEventExecutor

EventExecutor 相当于 EventExecutorGroup 的扩展，提供了 `next()`、`parent()` 等判断集合关系的方法。

<div style="width: 50%; margin: auto">![EventExecutor](EventExecutor.png)</div>
OrderedEventExecutor 是一个空接口，作用是为了说明继承这个接口就可以使用顺序执行任务等功能。

### EventLoopGroup、EventLoop

EventLoopGroup 提供了把 Channel 注册到具体 EventLoop 的方法。

<div style="width: 50%; margin: auto">![EventLoopGroup](EventLoopGroup.png)</div>
EventLoop 只提供一个 `parent()` 方法，用于判断 EventLoop 所属的 EventLoopGroup。

NioEventLoop 是 EventLoop 接口最常用的实现类。下面我们简单分析下 NioEventLoop 类。

## NioEventLoop 继承结构

<div style="width: 80%; margin: auto">![NioEventLoop](NioEventLoop.png)</div>
我们可以看到 NioEventLoop 的继承结构更复杂，右侧是上面介绍 EventLoop 的接口的继承结构，左侧是抽象类的继承关系。下面我们简单介绍几个抽象类：

* AbstractExecutorService
* AbstractEventExecutor
* SingleThreadEventExecutor
* SingleThreadEventLoop

### AbstractExecutorService

AbstractExecutorService 也是 java.util.concurrent 包里面的抽象类，实现了 ExecutorService 接口的部分方法。

<div style="width: 50%; margin: auto">![AbstractExecutorService](AbstractExecutorService.png)</div>
AbstractExecutorService 主要实现了执行任务的相关方法，例如 `submit`、`invokeAny` 等，把返回结果封装为 FutureTask 实例。

### AbstractEventExecutor

AbstractEventExecutor 以下是 Netty 定义的类，继承了 AbstractExecutorService 类，增加了 group 的概念。

<div style="width: 50%; margin: auto">![AbstractEventExecutor](AbstractEventExecutor.png)</div>
### AbstractScheduledEventExecutor

AbstractScheduledEventExecutor 继承了 AbstractEventExecutor，增加了可以延时或周期执行的任务队列 scheduledTaskQueue，并实现了 `schedule` 延时执行以及 `scheduleAtFixedRate` 周期执行等方法。

<div style="width: 50%; margin: auto">![AbstractScheduledEventExecutor](AbstractScheduledEventExecutor.png)</div>
### SingleThreadEventExecutor

SingleThreadEventExecutor 继承了 AbstractScheduledEventExecutor，实现了 EventLoop 的大部分功能，基本上算是最终 EventLoop 的一个雏型了。这个类的功能非常丰富，我们看下最关键的几个属性和方法。

<div style="width: 50%; margin: auto">![SingleThreadEventExecutor](SingleThreadEventExecutor.png)</div>
**关键属性**

* thread：每个 EventLoop 和一个死循环的线程进行绑定，这个线程就是在这里定义的。
* taskQueue：除了 IO 操作之外的其他任务，都放在这个队列里面依次执行。
* executor：在这里可以把这个理解为只有一个线程的线程池，可通过 executor 执行任务或获取执行线程的状态等。

这个类里面实现 EventLoop 在生命周期中的大部分工作：启动、添加任务、执行任务和关闭等。

**启动**

    private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                // 新建一个线程，并把线程和当前实例的thread对象绑定，这个thread就是NioEventLoop绑定的线程
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }
    
                boolean success = false;
                updateLastExecutionTime();
                try {
                    // 启动NioEventLoop的run方法
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    // 循环判断线程状态，如果是关闭中状态就进行下一步操作
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                            break;
                        }
                    }
    
                    ......
                }
            }
        });
    }
启动代码里面新建一个线程，并把线程和当前实例的 thread 对象绑定，这个 thread 就是 NioEventLoop 实例绑定的线程，然后再启动 NioEventLoop 实例。

**添加任务**

```
protected void addTask(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (!offerTask(task)) {
        reject(task);
    }
}

final boolean offerTask(Runnable task) {
    if (isShutdown()) {
        reject();
    }
    return taskQueue.offer(task);
}
```

添加任务过程很简单，就是把新的 task 添加到 taskQueue。

**执行任务**

```
protected final boolean runAllTasksFrom(Queue<Runnable> taskQueue) {
    Runnable task = pollTaskFrom(taskQueue);
    if (task == null) {
        return false;
    }
    for (;;) {
        //安全执行任务
        safeExecute(task);
        //继续执行剩余任务
        task = pollTaskFrom(taskQueue);
        if (task == null) {
            return true;
        }
    }
}

protected final Runnable pollTaskFrom(Queue<Runnable> taskQueue) {
    for (;;) {
        Runnable task = taskQueue.poll();
        //忽略WAKEUP_TASK类型任务
        if (task == WAKEUP_TASK) {
            continue;
        }
        return task;
    }
}

protected boolean runAllTasks(long timeoutNanos) {
    //先执行周期任务
    fetchFromScheduledTaskQueue();
    //从taskQueue提一个任务，如果为空执行所有tailTasks
    Runnable task = pollTask();
    //如果taskQueue没有任务，立即执行子类的tailTasks
    if (task == null) {
         afterRunningAllTasks();
        return false;
    }
    //计算出超时时间 = 当前 nanoTime + timeoutNanos
    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);

        runTasks ++;
        //当执行任务次数大于64判断是否超时，防止长时间独占CPU
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

这里包含安全执行执行单个任务、取出任务和执行所有任务。所有任务包括 taskQueue 和 下面提到到的 tailTasks。

### SingleThreadEventLoop

SingleThreadEventLoop 继承了 SingleThreadEventExecutor 类，主要增加了 tailTasks 属性。

<div style="width: 50%; margin: auto">![SingleThreadEventLoop](SingleThreadEventLoop.png)</div>
tailTasks 中是用户自定义的一些列在本次事件循环遍历结束后会执行的任务，我们可以通过类似如下的方式来添加 tailTask。

```java
((NioEventLoop)ctx.channel().eventLoop()).executeAfterEventLoopIteration(() -> {
    // add some task to execute after eventLoop iteration
});
```

### NioEventLoop

NioEventLoop 是最终实现类，这里定义了较多的方法和属性，其中最重要的有三个属性：

* SELECTOR_AUTO_REBUILD_THRESHOLD：用于解决 NIO 空轮询导致 cpu 飙升到 100% 的 bug。
* selector：选择器。
* selectorProvider：选择器的生产类。
* ioRatio：在事件循环中期待用于处理 I/O 操作时间的百分比。

<div style="width: 50%; margin: auto">![NioEventLoop1](NioEventLoop1.png)</div>
**selector**

netty 的 selector 对 NIO 的 selector 进行进一步封装，解决了空轮询导致 cpu 100% 的问题。

```
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        // select操作计数
        int selectCnt = 0;
        // 记录当前系统时间
        long currentTimeNanos = System.nanoTime();
        // delayNanos方法用于计算定时任务队列，最近一个任务的截止时间
        // selectDeadLineNanos 表示当前select操作所不能超过的最大截止时间
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        for (;;) {
            // 计算超时时间，判断是否超时
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            // 如果 timeoutMillis <= 0， 表示超时，进行一个非阻塞的 select 操作。设置 selectCnt 为 1. 并终止本次循环。
            if (timeoutMillis <= 0) {
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }
    
            // 如果当前任务队列为空，并且超时时间未到，则进行一个阻塞式的selector操作。timeoutMillis 为最大的select时间
            int selectedKeys = selector.select(timeoutMillis);
            // 操作计数 +1
            selectCnt ++;
            
            // 记录当前时间
            long time = System.nanoTime();
            // 如果time > currentTimeNanos + timeoutMillis(超时时间)，则表明已经执行过一次select操作
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                selectCnt = 1;
            } 
            // 如果 time <= currentTimeNanos + timeoutMillis，表示触发了空轮训
            // 如果空轮训的次数超过 SELECTOR_AUTO_REBUILD_THRESHOLD (512)，则重建一个新的selctor，避免空轮训
            else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                logger.warn(
                        "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                        selectCnt, selector);
    
                // 重建创建一个新的selector
                rebuildSelector();
                selector = this.selector;
    
                // Select again to populate selectedKeys.
                // 对重建后的selector进行一次非阻塞调用，用于获取最新的selectedKeys
                selector.selectNow();
                // 设置select计数
                selectCnt = 1;
                break;
            }
            currentTimeNanos = time;
        }
    
    } catch (CancelledKeyException e) {
    
    }
}
```

**selectorProvider**

我们调用 `Selector selector = Selector.open()` 创建 selector 的时候，内部会调用到 SelectorProvider。而在创建 SelectorProvider 对象本身的时候，不同系统会调用各自版本的 JDK 里自带的 sun.nio.ch.DefaultSelectorProvider。

```
// 实际上调用SelectorProvider来创建Selector
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}

// windows版本下SelectorProvider的创建
public class DefaultSelectorProvider {
    public static SelectorProvider create() {
        return new WindowsSelectorProvider();
    }
}

// mac版本下SelectorProvider的创建
public class DefaultSelectorProvider {
    public static SelectorProvider create() {
        return new KQueueSelectorProvider();
    }
}

// linux版本下SelectorProvider的创建
public class DefaultSelectorProvider {
    public static SelectorProvider create() {
        String str1 = (String) AccessController.doPrivileged(new GetPropertyAction(
                    "os.name"));

        if ("SunOS".equals(str1)) {
            return new DevPollSelectorProvider();
        }

        if ("Linux".equals(str1)) {
            String str2 = (String) AccessController.doPrivileged(new GetPropertyAction(
                        "os.version"));
            String[] arrayOfString = str2.split("\\.", 0);

            if (arrayOfString.length >= 2) {
                try {
                    int i = Integer.parseInt(arrayOfString[0]);
                    int j = Integer.parseInt(arrayOfString[1]);

                    if ((i > 2) || ((i == 2) && (j >= 6))) {
                        return new EPollSelectorProvider();
                    }
                } catch (NumberFormatException localNumberFormatException) {
                }
            }
        }

        return new PollSelectorProvider();
    }
}
```

**ioRatio**

当 ioRatio 变量为100的时候（默认50），处理 select 事件，处理完之后执行任务队列中的所有任务。 反之当不是 100 的时候，处理 select 事件，之后给定一个时间内执行任务队列中的任务。

```
final int ioRatio = this.ioRatio;
if (ioRatio == 100) {
    try {
        processSelectedKeys();
    } finally {
        // Ensure we always run tasks.
        runAllTasks();
    }
} else {
    final long ioStartTime = System.nanoTime();
    try {
        processSelectedKeys();
    } finally {
        // Ensure we always run tasks.
        final long ioTime = System.nanoTime() - ioStartTime;
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
}
```

## EventLoop 总结

* EventLoop 可以理解为只有一个死循环线程的线程池。
* EventLoop 归属某个 EventLoopGroup，可以获取在同一个 EventLoopGroup 下的其他 EventLoop 实例。
* EventLoop 可以执行线程池的各种功能，例如启动、提交任务、执行任务和关闭等
* EventLoop 可以执行普通任务、延时任务或者周期性任务等，任务全部放到任务队列里面，按照 FIFO 规则执行，任务队列分类有：
  1. scheduledTaskQueue，周期任务队列，例如 IO 操作等。scheduledTaskQueue 中的任务都会取出来先放入 taskQueue，再从 taskQueue 取出来执行。
  2. taskQueue，用户任务队列
  3. tailTasks，也是用户任务队列，但优先级比 taskQueue 低，用于存储当前或下一次事件循环(eventloop)迭代结束后需要执行的任务。

# EventLoopGroup

EventLoopGroup 是用来管理 EventLoop 的，负责把 Channel 绑定到 EventLoop 上，并返回其管理的 EventLoop 的各种状态。

EventLoopGroup 接口的继承结构已经包含在 EventLoop 的继承结构里面，下面我们介绍 EventLoopGroup 接口最常用的实现类 NioEventLoopGroup 的继承结构。

## NioEventLoopGroup 继承结构

<div style="width: 50%; margin: auto">![NioEventLoopGroup](NioEventLoopGroup.png)</div>
EventLoopGroup 接口及以上的继承结构在上面已经介绍过，下面我们简单介绍下几个抽象类：

* AbstractEventExecutorGroup
* MultithreadEventExecutorGroup
* MultithreadEventLoopGroup

### AbstractEventExecutorGroup

AbstractEventExecutorGroup 对应 AbstractEventExecutor 的集合，我们看下具体的类方法

<div style="width: 50%; margin: auto">![AbstractEventExecutorGroup](AbstractEventExecutorGroup.png)</div>
可以看到和 AbstractEventExecutor 类里面实现的方法基本一致，实际上最终也是调用到 AbstractEventExecutor 类里面的方法。

```
public Future<?> submit(Runnable task) {
		// next()返回下一个EventExecutor实例，最终会调到AbstractEventExecutor类的submit方法
    return next().submit(task);
}
```

### MultithreadEventExecutorGroup

MultithreadEventExecutorGroup 继承了 AbstractEventExecutorGroup，增加了 children 队列存储 EventExecutor 实例，以及选择器 chooser 负责从 children 里面选择 EventExecutor 实例来执行任务。

<div style="width: 50%; margin: auto">![MultithreadEventExecutorGroup](MultithreadEventExecutorGroup.png)</div>
MultithreadEventExecutorGroup 的构造方法会初始化包含 n 个 Executor 元素的 children 数组。

    protected MultithreadEventExecutorGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, Object... args) {
    
        children = new EventExecutor[nThreads];
    
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            }
        }
    }
MultithreadEventExecutorGroup 也包含关闭线程池的方法，看源码我们可以看到实际上是遍历 children 数组，逐个调用 Executor 元素的 `shutdownGracefully` 方法。

```
public Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit) {
    for (EventExecutor l: children) {
        l.shutdownGracefully(quietPeriod, timeout, unit);
    }
    return terminationFuture();
}
```

### MultithreadEventLoopGroup

MultithreadEventLoopGroup 继承了 MultithreadEventExecutorGroup 类，增加了 DEFAULT_EVENT_LOOP_THREADS 属性，这个属性定义了 children 数组的初始化大小。

<div style="width: 50%; margin: auto">![MultithreadEventLoopGroup](MultithreadEventLoopGroup.png)</div>
MultithreadEventLoopGroup 包含一个 static 块，定义 DEFAULT_EVENT_LOOP_THREADS 的值为 CPU 逻辑核数的两倍。

```
static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}
```

然后在构造方法里面传入 DEFAULT_EVENT_LOOP_THREADS 参数，然后会调用父类 MultithreadEventExecutorGroup 的构造方法，设置 children 数组的长度为 DEFAULT_EVENT_LOOP_THREADS 的值。

```
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

另外 MultithreadEventLoopGroup 也会负责完成把 Channel 注册到某一个 EventLoop 上。所以假如一共有 n 个 EventLoop，请求过来的第一个 Channel 被注册到第一个 EventLoop 上，第二个 Channel 被注册到下一个也就是第二个 EventLoop 上…..第 n + 1 个 Channel 又注册到第一个 EventLoop 上，如此循环。

```
public ChannelFuture register(Channel channel) {
		// 这里的next()最终会调用到chooser.next()，由chooser来选择下一个EventLoop
    return next().register(channel);
}
```

## EventLoopGroup 总结

* 一个 EventLoopGroup 包含多个 EventLoop，使用 children 数组存放 EventLoop 对象，children 的长度初始化为 cpu 逻辑核数 * 2。
* 调用 EventLoopGroup 的某个方法时，EventLoopGroup 会通过 chooser 选择策略获取到某个具体的 EventLoop，然后调用该 EventLoop 里面的具体方法，例如注册 Channel。

# EventLoopGroup 和 EventLoop 实例化过程

下面我们结合 EventLoop 和 EventLoopGroup 的继承结构涉及的接口和类，通过时序图的方式介绍 EventLoop 和 EventLoopGroup 的实例化和启动过程。

## NioEventLoopGroup 实例化过程

EventLoopGroup(其实是MultithreadEventExecutorGroup) 内部维护一个类型为 EventExecutor children 数组, 其大小是 nThreads, 这样就构成了一个线程池。如果我们在实例化 NioEventLoopGroup 时, 如果指定线程池大小, 则 nThreads 就是指定的值, 反之是处理器核心数 * 2。MultithreadEventExecutorGroup 中会调用 newChild 抽象方法来初始化 children 数组，抽象方法 newChild 是在 NioEventLoopGroup 中实现的, 它返回一个 NioEventLoop 实例。

![NioEventLoopGroupInstance](NioEventLoopGroupInstance.png)

## NioEventLoop 实例化过程

NioEventLoop 的实例化过程比较简单，主要是向上调用构造函数初始化 parent 等属性。

![NioEventLoopInstance](NioEventLoopInstance.png)

## NioEventLoop 和 Channel 绑定过程

channel 关联 Eventloop 有三种情况

* 客户端SocketChannel关联EventLoop。

* 服务端ServerSocketChannel关联EventLoop。

* 由服务端ServerSocketChannel创建的SocketChannel关联EventLoop。

Netty 厉害的就是把这三种情况都都能复用 Multithread EventLoopGroup 中的 register 方法

![NioEventLoopChannel](NioEventLoopChannel.png)

# 总结

本文介绍了 EventLoop 和 EventLoopGroup 的继承结构，最后结合继承结构涉及的类介绍了 EventLoop 和 EventLoopGroup 的实例化过程。

# 参考

[netty源码]()

[netty系列之（三）——EventLoop和EventLoopGroup](https://www.jianshu.com/p/f94f7005c2cd)

[Netty 源码分析之 三 我就是大名鼎鼎的 EventLoop(一)](https://segmentfault.com/a/1190000007403873)

[Netty进阶：Netty核心NioEventLoop原理解析](https://juejin.im/entry/5c0908ee6fb9a049d131efa7)

[[Netty NioEventLoop 启动过程源码分析](https://segmentfault.com/a/1190000016875286)](https://segmentfault.com/a/1190000016875286)

[netty补充NIO的SelectableChannel和SelectorProvider](https://www.jianshu.com/p/84412c2c34f1)