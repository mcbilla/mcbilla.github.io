---
title: Netty系列（七）：ByteBuf
tags:
  - Netty
categories:
  - Java组件
  - Netty
date: 2020-06-11 13:50:55

---

> 上一节我们介绍了 Netty 的通讯通道 Channel，这一节我们开始介绍 Netty 的数据容器 ByteBuf，包括 ByteBuf 的原理、内存管理和常用方法等。

<!--more-->

## ByteBuf是什么

ByteBuf 是 Netty 的数据容器，简单来说就是申请一块内存，后续可以在这块内存中执行读写数据等操作。相比起 NIO 原生的数据容器 ByteBuffer，ByteBuf 做了许多优化和改进：

- Bytebuffer 只有一个 index，读写公用，写模式到读模式转换需要调用 flip()，设置 index 值；Bytebuf 有读写两个 index,模式转换不用调用 flip()。
- Bytebuffer 不能扩容，每次 put 操作时，都会对可用空间进行校检，如果剩余空间不足，需要重新创建一个新的 ByteBuffer，然后将旧的 ByteBuffer 复制到新的 ByteBuffer 中去；而 ByteBuf 支持自动扩容。
- ByteBuf 用其内置的复合缓冲区可实现透明的零拷贝。
- ByteBuf 支持方法链接调用。
- ByteBuf 支持引用计数。
- ByteBuf 支持池技术(比如：线程池、数据库连接池)。

## ByteBuf的工作原理

### ByteBuf的结构

**ByteBuf 本质是一个由不同的索引分别控制读访问和写访问的字节数组**。以下是官方提供的 ByteBuf 的结构图：

```
      +-------------------+------------------+------------------+
      | discardable bytes |  readable bytes  |  writable bytes  |
      |                   |     (CONTENT)    |                  |
      +-------------------+------------------+------------------+
      |                   |                  |                  |
      0      <=      readerIndex   <=   writerIndex    <=    capacity
```

ByteBuf 包含三个参数：

- readerIndex：读索引，不超过 writerIndex，调用 `readXXX()` 或 `skipBytes()` 方法会移动该索引，调用 `getXXX()` 方法则不会移动。
- writerIndex：写索引，不超过 capacity，调用 `writeXXX()` 方法会移动该索引，调用 `setXXX()` 则不会移动。
- capacity：容量，超过 capacity 后会自动扩容，最大容量不超过 Integer.MAX_VALUE。

ByteBuf 被起始索引和这三个参数所在的索引分成三部分：

- discardable bytes（可丢弃字节）：范围为 0 ~ readerIndex，这部分字节数据已经被读取完，空间等待回收。
- readable bytes（可读字节）：范围为 readerIndex ~ writerIndex，这部分字节数据已经准备好，等待用户读取。
- writable bytes（可写字节）：范围为 writerIndex ~ capacity，这部分空闲空间可写入字节。

### ByteBuf的的工作过程

#### 1、初始化

![work1](work1.png)

ByteBuf 初始化后 readerIndex 和 writeIndex 的取值一开始都是 0。

#### 2、写入数据

![work2](work2.png)

写入 N 个数据后（N < capacity），writerIndex = N。

#### 3、读取数据

![work3](work3.png)

读取 M 个数据后（M < writerIndex），readerIndex = M。

#### 4、丢弃数据

![work4](work4.png)

读取完数据后，我们调用 `discardReadBytes()` 丢弃 0 ~ readerIndex 间的数据，相当于 readerIndex 和 writerIndex 同时向前移动 M 个字节，最后 readerIndex = 0，writerIndex = N - M。

#### 5、清空数据

![work5](work5.png)

当我们已经读取完所有的数据后，可以调用 `clear()` 使 ByteBuf 恢复到初始状态，这时候 readerIndex = writerIndex = 0。

#### 6、注意

- 调用 `clear() `的开销没有 `discardReadBytes() `那么大，因为它不需要任何内存复制。
- readerIndex 和 writerIndex 超过边界值，会发生 IndexOutOfBoundException 异常。

## ByteBuf的内存管理

### ByteBuf内存类型

ByteBuf 按照数据的存储位置，可以分为 **堆缓冲区(HEAP BUFFER)**、**直接缓冲区(DIRECT BUFFER)** 和 **复合缓冲区(COMPOSITE BUFFER)**。

* 堆缓冲区(HEAP BUFFER)：将数据存储在 JVM 的堆内存中，这些内存需要被 jvm 管理。

* 直接缓冲区(DIRECT BUFFER)：将数据存在  JVM 的堆内存外面，这些内存不需要被 jvm 管理。
* 复合缓冲区(COMPOSITE BUFFER)：以上两种方式的混合。

#### 1.1、堆缓冲区

最常用的模式，这种模式直接将数据存储在 JVM 的堆空间中，这种情况下 ByteBuf 数组被称为支撑数组（backing array）。

> 优点：能在没有使用池化的情况下提供快速的分配和释放，非常适合于有遗留的数据需要处理的情况。
>
> 缺点：在 socket 发送前，需要先把数据拷贝到直接缓冲区，导致 IO 效率不高。

```
// 创建支撑数组
ByteBuf buf = new UnpooledHeapByteBuf(ByteBufAllocator.DEFAULT, initalCapacity, maxCapacity);
// 检查 ByteBuf 是否有一个支撑数组，当 hasArray()方法返回 false 时，尝试访问支撑数组将触发一个 UnsupportedOperationException。
if (buf.hasArray()) { 
  byte[] array = buf.array();
  // 处理
  handleArray(array);
}
```

#### 1.2、直接缓冲区

将数据存储在 JVM 堆以外的内存，需要调用系统级别的 API

> 优点：免去中间交换的内存拷贝，提高 IO 速度。
>
> 缺点：分配和释放代价大，下文中池化的缓冲区可以缓解这个缺点。

```
// 创建直接缓冲区
ByteBuf buf = new UnpooledDirectByteBuf(ByteBufAllocator.DEFAULT, initalCapacity, maxCapacity);
// buf.hasArray() 为 false表示为这是直接缓冲区
if (!buf.hasArray()) { 
  byte[] array = buf.array();
  // 处理
  handleArray(array);
}
```

#### 1.3、复合缓冲区

Netty通过ByteBuf的子类 CompositeByteBuf，提供了将多个 buffer 虚拟成一个合并的 Buffer 的技术。CompositeByteBuf 中的 ByteBuf 实例可能同时包含堆缓冲区的和直接缓冲区的。如果 CompositeByteBuf 只含有一个实例，调用 hasArray() 方法会返回这个实例的 hasArray() 方法的值；否则总是返回 false。

例如一条消息包含 header 和 body 两部分，每条消息需要使用不同的 header 搭配同一个 body，就可以使用复合缓冲区。

![image-20200706211316006](/Users/mochuangbiao/Library/Application Support/typora-user-images/image-20200706211316006.png)

```
CompositeByteBuf messageBuf = Unpooled.compositeBuffer();
// header使用堆内存每次分配
ByteBuf headerBuf = new UnpooledHeapByteBuf(ByteBufAllocator.DEFAULT, initalCapacity, maxCapacity);
// body需要重复使用，所以使用直接缓冲区，省去每次复制内存导致效率下降
ByteBuf bodyBuf = new UnpooledDirectByteBuf(ByteBufAllocator.DEFAULT, initalCapacity, maxCapacity);
// 将headerBuf和bodyBuf添加到CompositeByteBuf中
messageBuf.addComponents(headerBuf, bodyBuf);
.....
//遍历messageBuf中的ByteBuf
for (ByteBuf buf : messageBuf) {
    System.out.println(buf.toString());
}
```



按内存对象是否可以重复利用，可以分为 **池化内存(Pooled)** 和 **非池化内存(Unpooled)**。

- 池化内存(Pooled)：每次都从预先分配好的内存中去取出一段连续内存封装成一个 ByteBuf 给应用程序使用。
- 非池化内存(Unpooled)：每次分配内存的时候，直接调用系统 api，向操作系统申请一块内存。

#### 2.1、池化内存

Netty 会先向系统申请一大块内存，通过池化算法管理这块内存，并向上层提供申请内存接口。我们需要使用内存的时候，再通过接口申请一段连续内存封装成一个 ByteBuf 给应用程序使用。

> 优点：池对象可以回收复用，提高内存分配的效率。
>
> 缺点：需要对预先申请的内存进行管理。

#### 2.2、非池化内存

同 java 对象的创建销毁过程，直接向 JVM 申请内存创建 ByteBuf 对象，使用完后交给 JVM 进行 gc 回收。

> 优点：使用简单
>
> 缺点：对象不能复用，创建销毁代价很大。

### 创建ByteBuf实例

#### 1、ByteBufAllocator

Netty 为创建 ByteBuf 实例专门提供了一个接口 **ByteBufAllocator**，该接口有 **PooledByteBufAllocator** 和 **UnpooledByteBufAllocator** 两种实现。前者返回池化实例，后者则直接返回非池化的新实例。

Netty 默认使用了池化的 ByteBufAllocator，但我们可以在 Bootstrap 配置参数来自定义使用池化还是非池化分配器。

ByteBufAllocator 适用于可以获取到 channelHandlerContext 或 channel 实例的场景。

* 使用 PooledByteBufAllocator 创建 ByteBuf

```
// 通过 channelHandlerContext 获取 allocator
ByteBufAllocator allocator1 = ctx.alloc();
// 通过 channel 获取 allocator
ByteBufAllocator allocator2 = ctx.channel().alloc();

// 返回一个直接内存或者堆内存对象，具体实现由 netty 根据环境自动配置
ByteBuf buf = allocator1.buffer();
// 创建直接内存对象
ByteBuf directBuf = allocator1.directBuffer();
// 创建堆内存对象
ByteBuf heapBuf = allocator1.heapBuffer();
```

* 使用 UnpooledByteBufAllocator 创建 ByteBuf

```
// 在 bootstrap 中配置参数使用 UnpooledByteBufAllocator
bootstrap.option(ChannelOption.ALLOCATOR, UnpooledByteBufAllocator.DEFAULT)

// 后面使用同 PooledByteBufAllocator
ByteBufAllocator allocator1 = ctx.alloc();
......
```

#### 2、Unpooled

当没有办法获取到 channelHandlerContext 或 channel 实例的时候，推荐使用 Unpooled 类来创建 ByteBuf。

```
// 创建堆内存对象
ByteBuf heapBuf = Unpooled.buffer();
// 创建直接内存对象
ByteBuf directBuf = Unpooled.directBuffer();
```

### ByteBuf引用计数

上面谈到 Netty 主要有四种类型的 ByteBuf，其中 UnpooledHeapByteBuf 能够依赖 JVM GC 回收器自动回收；UnpooledDirectByteBuf 除了等 JVM GC，最好也能主动进行回收；而 PooledHeapByteBuf 和 PooledDirectByteBuf，则必须要主动将用完的 bytebuf 放回池里，如果不释放，内存池会越来越大，直到内存溢出。所以，Netty ByteBuf 需要在 JVM 的 GC 机制之外，有自己的引用计数器和回收过程（主要是回收到netty申请的内存池）。

所有 ByteBuf 的引用计数器初始值为1。可以增加/减少 ByteBuf 的引用计数器，当 ByteBuf 的引用计数器为 0 时该对象就会被释放内存或回收到内存池。访问一个被释放的 ByteBuf 会抛 IllegalReferenceCountException 异常。

ByteBuf 引用计数器的常用方法有：

* `release()`：计数器减1。从 InBound 里读取的 ByteBuf 和自己创建的 ByteBuf 要自己调用 `release()` 手动释放，写入到 OutBound 的 Bytebuf 例如调用 `writeAndFlush()` 由 netty 负责释放，不需要手动调用`release()`。
* `retain()`：计数器加1。在使用派生缓冲区的时候，为了防止源 ByteBuf 突然被释放导致派生缓冲区操作异常，需要调用 `retain()` 来显示派生缓冲区的存在。
* `refCnt()`：返回计数器的值。
* `copy()`：返回源 ByteBuf 的完全拷贝。该拷贝和源 ByteBuf 完全独立，在新 ByteBuf 上修改不会影响旧 ByteBuf。

以上方法都是直接在原生内存上进行操作，ByteBuf 还提供了一种视图的方式来读写内容，这种视图被称作**派生缓冲区**。派生缓冲区维护单独的 readIndex、writeIndex 和 markIndex，但与原生 ByteBuf 共享数据和引用计数器，这意味着派生缓冲区的创建开销很低，但是如果修改了其他内容，也同时修改对应的原生 ByteBuf。派生缓冲区的常用方法有：

* `duplicate()`：返回源 ByteBuf 的完全拷贝的派生缓冲区区。该方法内部不会调用 `retain()` 方法，所以计数器不会增加。
* `slice()`：返回源 ByteBuf 已经写入数据区域的拷贝的派生缓冲区。该方法内部不会调用 `retain()` 方法，所以计数器不会增加。

## 总结

* ByteBuf 本质是一个由不同的索引分别控制读访问和写访问的字节数组。
* ByteBuf 有 readerIndex、writerIndex 和 capacity 三个参数，这三个负责控制 ByteBuf 的读写操作。
* ByteBuf 的内存管理，按照数据的存储位置，可以分为堆缓冲区、直接缓冲区和复合缓冲区；按内存对象是否可以重复利用，可以分为池化内存和非池化内存。
* 创建 ByteBuf 实例，可以使用 ByteBufAllocator 接口或者 Unpooled 工具类。
* ByteBuf 具有引用计数器的概念，当引用计数器为 0 时 ByteBuf 对象会被回收。

## 参考

[《Netty In Action》]()

[ByteBuf：Netty的数据容器](https://www.jianshu.com/p/3fbf54b8e8ec)

[ByteBuf工作原理](https://www.dazhuanlan.com/2019/11/09/5dc5cfc71d669/)

[ByteBuf : Netty的数据容器类](https://www.jianshu.com/p/175cd3644728)