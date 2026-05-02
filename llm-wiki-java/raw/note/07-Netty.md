# 07-Netty

所有者: junk01

### **一、 IO模型**

| 模型 | 性质 | 特点 |
| --- | --- | --- |
| **BIO** | 同步阻塞 | 一连接一线程，线程开销大 |
| **NIO** | 同步非阻塞 | 多路复用，一个线程处理多个连接,靠Selector轮询事件 |
| **AIO** | 异步非阻塞 | 内核回调通知完成（Linux不成熟，Java实际使用少） |
| **select** | 数组 | 连接数限制（1024），每次拷贝所有 fd,实现简单 |
| **poll** | 链表 | 仍需轮询,无连接数上限 |
| **epoll** | 回调(事件通知） | 事件回调模型,需要 epoll_ctl 管理,**高性能、无轮询、事件驱动** |

### **二、NIO 核心组件**

**(1) Channel:** 双向通信，可读可写。

**(2) Buffer:** 存储数据；内部结构是数组；通过 readerIndex / writerIndex 控制读写位置。

**(3) Selector(多路复用器）:** 监视多个Channel的IO事件（read、write、accept）。

---

### **三、Netty 高性能架构**

**(1) Reactor线程模型：**默认主从Reactor多线程模型，1、BossGroup处理accept连接事件 2、WorkerGroup：处理读写事件

**(2) NIO多路复用：**使用select/poll/epoll（Linux自动使用epoll）。

**(3) 单线程串行化无锁化：**同一个Channel 永远由同一个 EventLoop 处理 → 避免锁竞争。

**(4) 内存池PooledByteBufAllocator：**减少频繁创建销毁，内部是分级 / chunk / page / subpage 管理体系（类似 jemalloc）。

**(5) 并发优化：**CAS、Atomic、volatile、高性能队列 MPSC、任务队列执行runAllTasks()

**(6) 零拷贝：**

- DirectByteBuf（堆外内存，无用户空间和系统空间二次拷贝，把I/O成本压到最低）
- CompositeByteBuf（无需复制拼接多个 buffer）
- FileRegion（文件传输用sendfile）
- ByteBuf slice/duplicate 共享内存

---

### **四、Netty 线程模型**

**NioEventLoopGroup：**内部维护多个 EventLoop，每个 EventLoop 一个固定线程。

**NioEventLoop:** 单线程循环执行，Channel绑定EventLoop生命周期唯一不会换线程 → 所有IO都在同线程执行 → 无锁。

**BossEventLoop：**1、处理accept 2、创建 NioSocketChannel 3、注册到WorkerEventLoop的selector

**WorkerEventLoop：**1、处理 read/write 2、调用 Pipeline → Handler 3、执行任务队列（普通任务 / 定时任务）

---

### **五、Netty 核心组件**

**（1）EventLoopGroup / EventLoop：**线程池 + 单线程事件循环。所有IO操作串行执行，天然无锁。

**（2）Channel：**网络连接对象，绑定一个 EventLoop。

**（3）ChannelPipeline：**责任链模式 + 双向链表，分发事件：（Inbound：head → tail ）（Outbound：tail → head）

**（4）ChannelHandler / ChannelHandlerContext：**处理IO事件，context 保存所有上下文、可发起事件传播。

**（5）Unsafe：**内部 API真正执行底层 I/O的类(用户不直接用)。

**（6）ChannelOutboundBuffer（写队列）：**write → 入队 ；flush → 真正写 socket

---

### **六、ByteBuf 内存**

| 特性 | 说明 |
| --- | --- |
| **读写分离** | **readerIndex / writerIndex**同时支持读写 |
| **容量可自动扩展** | 超出时自动扩充 |
| **零拷贝** | slice、duplicate、CompositeByteBuf |
| **池化** | 复用内存、提高性能、PooledByteBufAllocator |
| **引用计数机制** | `retain()` / `release()` |
| **Leak 检测** | ResourceLeakDetector |
| 释放 | 必须手动释放 ByteBuf，否则会内存泄漏 |

### **七、编解码器模型**

**Inbound（解码Decoder）**`ByteToMessageDecoder` (ecode 不得阻塞/消耗不完整数据会出半包问题）`LengthFieldBasedFrameDecoder(`解决粘拆包)

**Outbound（编码Encoder）**`MessageToByteEncoder、LengthFieldPrepender`

**粘包拆包解决:**`FixedLengthFrameDecoder`固定长度,`DelimiterBasedFrameDecoder`分隔符,**`LengthFieldBasedFrameDecoder`**长度字段(生产常用)

---

### **八、Netty 心跳机制**

**心跳机制 IdleStateHandler**

- 只负责检测，不负责发送心跳包
- 心跳逻辑写在下一个 handler
- IdleStateHandler(readerIdle, writerIdle, allIdle, TimeUnit.SECONDS)

---

### **九、Netty 断线重连**

- 启动失败：channelFuture.addListener
- 运行中断开：`channelInactive()` 发起重连

---

### **十、Netty 链路**

```jsx
1. ServerBootstrap.bind()
2. BossGroup accept连接
3. 创建NioSocketChannel
4. 注册到Worker线程
5. pipeline.fireChannelActive()

---- 数据读取 ----
6. Worker select 到 READ
7. 读取 ByteBuf（unsafe.read）
8. pipeline.fireChannelRead(byteBuf)
9. decoder → handler

---- 写数据 ----
10. handler.write()
11. pipeline outbound（编码等）
12. headContext → unsafe.write → OutboundBuffer

---- 刷新 ----
13. flush → socket.write()

---- 回压 ----
14. channelWritabilityChanged()
backpressure回压机制：
if (!channel.isWritable()) {
    channel.config().setAutoRead(false); // 暂停读
}

---- 任务队列 ----
15. runAllTasks()

---- 关闭 ----
16. close → inactive → unregistered
```

**1. Boss线程启动和连接**

```jsx
**1、启动阶段**
	bind()->initAndRegister()->NioServerSocketChannel
	register()->channel->bossGroup->NioEventLoop
	bossEventLoop执行->doBind()->完成端口绑定
	Selector->开始select循环->监听accept事件

bossGroup = new NioEventLoopGroup(1);
workerGroup = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelInitializer<SocketChannel>() {...})
 .bind(8080);
 
2、**accept新连接->BossGroup**
NioEventLoop.run()
    → select()
    → processSelectedKeys()
        → NioMessageUnsafe.read()
            → doReadMessages()  //accept新连接，返回SocketChannel
            
3、**连接建立**
	Boss EPOLL 返回ACCEPT
	NioServerSocketChannel$Unsafe.read()处理accept
	创建->**NioSocketChannel**
	选择一个WorkerEventLoop：workerGroup.next() → workerEventLoop
	NioSocketChannel 注册到 WorkerEventLoop
	触发 pipeline 事件->channelRegistered()->channelActive()
```

**2. Worker线程读入站事件**

```jsx
NioEventLoop.run()
  → select()
  → processSelectedKeys()
      → NioSocketChannel$Unsafe.read()
      
1、Worker EventLoop select 到 READ 事件：
2、**socket → ByteBuf(**池化)->调用JNI:socket.read()->DirectByteBuf->ByteBuf->入站事件fire
****3、pipeline.fireChannelRead(byteBuf)
4、pipeline入站：HeadContext → Decoder → Handler → TailContext(入站事件传播顺序：head → next → next → ... → tail)
	**Decoder->** ByteToMessageDecoder->cumulation(累积区)->decode(ByteBuf in)->List<Object>->fireChannelRead(obj)(Decoder不允许阻塞、不能卡住EventLoop)
	**Inboundhandler入站事件:**
	channelRead()            → 处理字节数据
	channelReadComplete()    → 通常触发 ctx.flush()
	channelWritabilityChanged() → 回压
	exceptionCaught()        → 异常通道
```

**3. Worker线程出站事件**

```java
业务handler发送响应：
ctx.write(msg);
write**出站链路**：tail → prev → prev → ... → head
1. 经过所有OutboundHandler（编码器、加密器…）
2. HeadContext.write->Unsafe.write
3. 进入ChannelOutboundBuffer（写队列）
4.flush 或者手动 writeAndFlush()
	channelReadComplete() -> ctx.flush()
	ctx.flush() →
	  OutboundHandler.flush()... →
	    HeadContext.flush() →
	      Unsafe.flush() →
	        ChannelOutboundBuffer.removeBytes() →
	          socket.write()（JNI调用）

```

**4. Worker线程任务队列**

```
EventLoop除了 IO 事件，还有两类任务：
****1、普通任务队列taskQueue
	ctx.executor().submit(...)
2、定时任务队列 cheduledTaskQueue
	ctx.executor().schedule(() -> {...})
3、EventLoop.run() 的执行顺序：
	select → process IO → runAllTasks（普通 + 定时）
```

**5. 连接关闭链路**

```
当连接关闭：
channel.close()
  → pipeline.fireChannelInactive()
  → deregister()
  → close ChannelOutboundBuffer
  → Selector removeKey
最终：
channelInactive()
channelUnregistered()
ByteBuf引用计数为 0 → 内存归还池化系统。
```

---

### Netty的线程模型

Netty 通过 Reactor 模型基于多路复用器接收并处理用户请求的。多路复用IO模型参考：

多路复用就是首先去阻塞的调用系统，询问内核数据是否准备好，如果准备好，再重新进行系统调用，进行数据拷贝。常见的实现有select，epoll和poll三种。

Netty 的线程模型并不是一成不变的，它实际取决于用户的启动参数配置。通过设置不同的启动参数，Netty支持三种模型，分别是**Reactor单线程模型、Reactor多线程模型、Reactor主从多线程模型。**

**单Reactor单线程模型**

这是最简单的Reactor模型，当有多个客户端连接到服务器的时候，服务器会先通过线程A和客户端建立连接，

有连接请求后，线程A会将不同的事件（如连接事件，读事件，写事件）进行分发，譬如有IO读写事件之后，会把该事件交给具体的Handler进行处理。

而线程A，就是我们说的Reactor模型中的Reactor，Reactor内部有一个dispatch（分发器）。【注意，这里的Reactor单线程，主要负责事件的监听和分发】

此时一个Reactor既负责处理连接请求，又要负责处理读写请求，一般来说处理连接请求是很快的，但是处理具体的读写请求就要涉及字节的复制，相对慢太多了。Reactor正在处理读写请求的时候，其他请求只能等着，只有等处理完了，才可以处理下一个请求。

通过一个Reactor线程，只能对应一个CPU，发挥不出来多核CPU的优势。所以，一个Reactor线程处理简单的小容量场景，还是OK的，但是对于高负载来说，还需要进一步升级。

**单Reactor多线程模型**

为了利用多核CPU的优势，也为了防止在Reactor线程等待读写事件时候浪费CPU，所以可以增加一个Worker的线程池，由此升级为单Reactor多线程模式。

整体流程如下：

当多个客户端进入服务器后，Reactor线程会监听多种事件（如连接事件，读事件，写事件），如果监听到连接事件，则把该事件分配给acceptor处理，如果监听到读事件，那么则会发起系统调用，将数据写入内存，之后再把数据交给工作线程池进行业务处理。

这个时候我们会发现，业务处理的逻辑已经变成多线程处理了。不过一个Reactor既要负责连接事件，又要负责读写事件，同时还要负责数据准备的过程。因为拷贝数据是阻塞的，假如说Reactor阻塞到拷贝数据的时候，服务器进来了很多连接，这个时候，这些连接是很有可能会被服务器拒绝掉的。

所以，单个Reactor看来是不够的，我们需要使用多个Reactor来处理。

**主从Reactor模型**

在主从Reactor模型中，主Reactor线程只负责连接事件的处理，它把读写事件全部交给了子Reactor线程。这样即使在数据准备阶段子线程被阻塞，主Reactor还是可以处理连接事件。巧妙的解决了高负载下的连接问题。

---

### Netty的Buffer

在网络编程中，基本都是基于TCP报文的字节流的操作，所以Java的NIO又新增了ByteBuffer，只不过Java原生的ByteBuffer，非常难操作，也不能扩缩容，所以Netty又重新封装了自己的Bytebuf，除了性能上的优势之外，Netty的Buffer在使用上相对于NIO也非常简洁，有如下特点：

**动态扩缩容**

顾名思义，Netty中的ByteBuffer可以像Java中的ArrayList一样，根据写入数据的字节数量，自动扩容。代码如下所示：

这个在编写代码的时候，满足ByteBuf最大缓冲区的情况下，我们可以毫无顾忌地调用#write方法增加字节，而不用手动去check容量满足，然后去重新申请读写指针代替#flip

**原生ByteBuffer的弊端**

Java原生的ByteBuffer的数据结构，分为limit，capacity两个指针，如果我们写入“Hollis”之后，ByteBuffer的内容如下：

此时，如果我们要从该ByteBuffer中read数据，ByteBuffer会默认从position开始读，这样就什么也读不到，所以我们必须调用#flip方法，将position指针移动，如下：

这样我们才可以读到“Hollis”这个数据，万一我们调用的时候忘记使用flip，就会很坑爹。

**Netty的ByteBuf**

Netty自带的ByteBuf通过读写双指针避免了上面的问题，假如我们写入“Hollis”后，ByteBuf的内容如下：

在写入的同时，我们可以直接通过readPointer读取数据，如下所示：

在这个过程中，我们完全不用像JavaNIO的ByteBufer一样，感知其结构内部的操作，也不用调用flip，随意的读取和写入即可。

同时，假如我们读Hollis这个数据，读到了一半，还剩下“is”没有读完，我们可以调用discardReadBytes方法将指针移位，为可写区域增加空间，如下所示：

**多种ByteBuf实现**

Netty根据不同的场景，有不同的ByteBuf实现，主要的几种分别是：Pooled，UnPooled，Direct，Heap，列表格如下：

|  | Pooled | UnPooled |
| --- | --- | --- |
| HeapByteBuf | 业务处理使用+高并发 | 业务处理使用+正常流量 |
| DireactByteBuf | Socket相关操作使用+高并发 | Socket相关操作使用+正常流量 |

当然Netty中的Buffer性能相比于Java NIO的Buffer也更强，譬如我们熟知的Zero-Copy等，这个我们放到另一篇中剖析

---

### Netty 的对象池

Netty 内置了对象池，用于重复利用一些已经创建过的对象，避免频繁地创建和销毁对象，从而提升系统的性能和可靠性。

对象池是一种非常常见的设计模式，它在多线程的环境中特别有用，能够有效地减少线程的上下文切换和资源的浪费，同时也有利于避免内存泄漏等问题。Java中的字符串池，其实也就是一种对象池技术

当我们使用Netty编写一个网络应用程序时，可能需要频繁地创建和释放ByteBuf对象来处理输入和输出数据。如果我们每次需要时都创建新的ByteBuf对象，会导致频繁的垃圾回收和内存分配，降低性能。为了避免这种情况，Netty提供了对象池技术，通过对象池来重用ByteBuf对象，从而减少垃圾回收和内存分配。

我们使用DefaultObjectPool类创建了一个对象池，并通过实现ByteBufAllocator接口的方式来指定如何创建ByteBuf对象。

通过调用borrowObject方法，我们可以从对象池中获取一个可用的ByteBuf对象，并在处理完数据后，调用returnObject方法将它归还到对象池中，供下次使用。这样就能够重复利用ByteBuf对象，从而减少了垃圾回收和内存分配。

Netty 对象池技术主要有以下几个优势：

1. 提高性能：重复利用对象可以避免频繁地创建和销毁对象，从而减少了系统开销，提高了系统的性能。
2. 提高可靠性：通过避免对象的重复创建和销毁，可以避免一些潜在的内存泄漏问题，从而提高系统的可靠性和稳定性。
3. 简化编程：通过使用对象池，可以让开发人员更加专注于业务逻辑的实现，而不必过于关心对象的创建和销毁。

---

### Netty 中设计模式

单例模式

NioEventLoop 通过核心方法 select() 不断轮询注册的 I/O 事件，Netty 提供了选择策略 SelectStrategy 对象，这个对象就是个单例：

工厂模式

工厂模式是一个比较常见的设计模式，在很多框架中都会用到，Netty也不例外，只要我们的Netty中搜索Factory，得到的类几乎都是和工厂模式有关的。SelectStrategy也是用工厂创建的：

装饰者模式

Netty中的WrappedByteBuf就是你对ByteBuf的装饰。来实现对他的增强：

责任链模式

责任链在Netty中用的比较多，Netty中有大量的[ChannelPipeline](https://github.com/netty/netty/blob/d34212439068091bcec29a8fad4df82f0a82c638/transport/src/main/java/io/netty/channel/ChannelPipeline.java)，而这些Pipeline就是通过责任链来驱动的。

策略模式

Netty 在多处地方使用了策略模式，例如 [EventExecutorChooser](https://github.com/netty/netty/blob/d34212439068091bcec29a8fad4df82f0a82c638/common/src/main/java/io/netty/util/concurrent/DefaultEventExecutorChooserFactory.java) 提供了不同的策略选择 NioEventLoop，newChooser() 方法会根据线程池的大小情况来动态选择取模运算的方式：

而这里的PowerOfTwoEventExecutorChooser和GenericEventExecutorChooser是是EventExecutorChooser的两种具体策略实现：