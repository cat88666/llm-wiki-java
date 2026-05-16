---
type: concept
status: active
name: "Netty"
layer: L7
aliases: ["Netty", "NIO框架", "Reactor模型", "ChannelPipeline", "ByteBuf", "主从Reactor", "粘包拆包", "EventLoop", "IdleStateHandler", "WriteBufferWaterMark"]
tags: ["#distributed"]
related:
  - "[[概念-IO模型]]"
  - "[[机制-Dubbo]]"
  - "[[主题-设计模式]]"
---

# Netty

> 基于 NIO 的异步事件驱动网络框架，用**主从 Reactor 多线程模型 + ChannelPipeline 责任链**解决 Java 原生 NIO 使用复杂、select/poll O(n) 轮询低效的问题。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | epoll O(1) vs select/poll O(n)，原生 NIO API 问题 |
| [二、IO 模型对比](#二io-模型对比) | BIO/NIO/AIO + select/poll/epoll |
| [三、主从 Reactor 架构](#三主从-reactor-架构) | BossGroup / WorkerGroup 职责分离 |
| [四、核心组件](#四核心组件) | EventLoopGroup、ChannelPipeline、ChannelHandler、ByteBuf |
| [五、粘包拆包解决方案](#五粘包拆包解决方案) | 入站解码、出站编码、3种拆包解码器 |
| [六、心跳机制](#六心跳机制) | IdleStateHandler 检测 + 应用层心跳 |
| [七、断线重连与关闭链路](#七断线重连与关闭链路) | ChannelFuture 监听、channelInactive 重连、关闭清理 |
| [八、TCP 参数调优](#八tcp-参数调优) | SO_BACKLOG、TCP_NODELAY、WriteBufferWaterMark |
| [九、高性能设计要点](#九高性能设计要点) | 无锁串行化、内存池、零拷贝、epoll ET |
| [十、关键权衡](#十关键权衡) | EventLoop 单线程 vs 耗时操作，ByteBuf 手动释放 |
| [十一、与其他概念的关系](#十一与其他概念的关系) | IO模型、Dubbo、设计模式 |
| [十二、应用边界](#十二应用边界) | 适合 vs 不适合场景 |

## 一、第一性原理

Java 原生 NIO 有两个根本问题：
1. **API 繁琐**：flip/position/limit 易出错，读写需要手动切换模式
2. **select/poll O(n) 轮询**：每次需要遍历所有 fd，连接数多时 CPU 浪费在无效轮询

Netty 在 Linux 上使用 **epoll 事件通知**（O(1) 就绪事件回调），并通过 **Channel 绑定固定 EventLoop** 消除锁竞争，从根本上解决这两个问题。

## 二、IO 模型对比

| 模型 | 性质 | 实现 | 特点 |
|------|------|------|------|
| **BIO** | 同步阻塞 | 一连接一线程 | 简单但线程开销大，1000 连接需 1000 线程 |
| **NIO** | 同步非阻塞 | Selector 多路复用 | 一线程监控多连接，靠 select/poll/epoll |
| **AIO** | 异步非阻塞 | 内核回调通知 | Linux 实现不成熟，Java 实际使用极少 |
| **select** | 数组，fd 限制 1024 | 轮询 O(n) | 简单但有上限 |
| **poll** | 链表，无上限 | 轮询 O(n) | 去掉上限但仍低效 |
| **epoll** | 红黑树 + 就绪链表 | 事件回调 O(1) | 仅通知就绪 fd，高并发首选 |

## 三、主从 Reactor 架构

**Netty 默认采用主从 Reactor 多线程模型**：

```
客户端连接
    ↓
BossGroup（1个EventLoop，负责 accept）
    → 创建 NioSocketChannel
    → 选一个 WorkerEventLoop 注册（负载均衡）
WorkerGroup（N个EventLoop，每个固定一个线程）
    ├── 处理 read/write 事件
    ├── 调用 ChannelPipeline → Handler 链
    └── 执行 taskQueue（普通任务 + 定时任务）
```

- 主 Reactor（BossGroup）只处理连接事件，不阻塞在数据拷贝
- 从 Reactor（WorkerGroup）处理 IO 读写，天然多核并行

**三种 Reactor 模型**：

| 模型 | 线程分配 | 适用场景 |
|------|----------|----------|
| Reactor 单线程 | accept、read/write、业务 Handler 都在一个线程 | 低并发、教学示例 |
| Reactor 多线程 | 一个 accept 线程 + 多个 IO Worker 线程 | 中等并发，连接建立不是瓶颈 |
| 主从 Reactor 多线程 | BossGroup 负责 accept，WorkerGroup 负责读写和 Pipeline | Netty 默认模型，高并发长连接 |

**完整链路**：
```
服务端启动 → BossGroup select ACCEPT
    → 创建 NioSocketChannel → 注册到 WorkerEventLoop.selector

数据读入 → WorkerGroup select READ
    → 从 socket 读数据到 DirectByteBuf
    → pipeline.fireChannelRead(buf)
    → HeadContext → Decoder → BusinessHandler

业务响应 → ctx.write(msg)
    → OutboundHandler 链（tail→head）→ Encoder
    → HeadContext → ChannelOutboundBuffer → flush() → socket.write()

连接关闭 → channelInactive() → deregister()
    → ChannelOutboundBuffer 清理 → ByteBuf 引用归零 → 归还内存池
```

**关键链路拆解**：

- **服务端启动**：`bind()` → `initAndRegister()` → 创建 `NioServerSocketChannel` → 注册到 BossGroup 的 Selector
- **连接接入**：BossEventLoop `select(ACCEPT)` → `processSelectedKeys()` → `NioMessageUnsafe.read()` → `doReadMessages()` 创建客户端 Channel
- **注册 Worker**：Boss 将 `NioSocketChannel` 注册到某个 WorkerEventLoop，后续该 Channel 的所有 IO 都固定在这个线程执行
- **数据读取**：WorkerEventLoop `select(READ)` → `Unsafe.read()` → 分配 `DirectByteBuf` → `pipeline.fireChannelRead(buf)`
- **数据写出**：`ctx.write(msg)` → Outbound 链 tail 到 head → Encoder → `ChannelOutboundBuffer` → `flush()` → `socket.write()`
- **任务执行**：EventLoop 每轮循环通常是 `select` → 处理 IO 事件 → `runAllTasks()`，普通任务和定时任务都在同一线程串行执行

## 四、核心组件

### EventLoopGroup / EventLoop

NioEventLoopGroup 内含多个 NioEventLoop，每个 EventLoop 持有一个固定线程 + 一个 Selector。**同一个 Channel 永远由同一个 EventLoop 处理**，所有 IO 在该线程串行执行 → **天然无锁**。

### ChannelPipeline

责任链模式 + 双向链表，分 Inbound（入站，head→tail）和 Outbound（出站，tail→head）：

```
读：网络数据 → HeadContext → Decoder → BusinessHandler → TailContext
写：BusinessHandler → Encoder → HeadContext → Socket
```

### ChannelHandler / ChannelHandlerContext

`ChannelHandler` 承载具体业务逻辑，分为入站 Handler（处理读事件）和出站 Handler（处理写事件）。`ChannelHandlerContext` 是 Handler 在 Pipeline 中的上下文，既能访问当前 Channel，也能控制事件继续向前传播：

- `ctx.fireChannelRead(msg)`：把入站事件交给下一个 InboundHandler
- `ctx.write(msg)`：从当前节点开始向前查找 OutboundHandler
- `ctx.executor()`：拿到绑定的 EventLoop，避免跨线程直接操作 Channel

### Unsafe 与 ChannelOutboundBuffer

`Unsafe` 是 Netty 内部封装底层 Socket 操作的组件，负责 `register/read/write/flush/close` 等低层动作，业务代码不直接使用。`ChannelOutboundBuffer` 是出站缓冲区，`write()` 先把消息放入缓冲，`flush()` 再批量写到 Socket；配合 `WriteBufferWaterMark` 可以对慢客户端做背压保护。

### ByteBuf

对比 Java 原生 NIO ByteBuffer（只有 position/limit，读写需 flip）：

| 特性 | ByteBuf | Java ByteBuffer |
|------|---------|-----------------|
| 读写指针 | `readerIndex` / `writerIndex` 分离 | 一个 position，读写需 flip |
| 容量 | 动态扩容 | 固定容量 |
| 零拷贝 | slice/duplicate/CompositeByteBuf/FileRegion(sendfile) | 无 |
| 池化 | PooledByteBufAllocator（类 jemalloc）| 无 |
| 引用计数 | retain() / release()，需手动释放 | GC 管理 |
| 泄漏检测 | ResourceLeakDetector 辅助定位未 release 的 ByteBuf | 依赖 GC |

**四种 ByteBuf 实现**（按分配方式 × 是否池化）：

|  | Pooled（高并发）| UnPooled（正常流量）|
|--|---|---|
| **HeapByteBuf** | 业务处理（JVM堆）| 业务处理 |
| **DirectByteBuf** | Socket IO（堆外，避免内核拷贝）| Socket IO |

## 五、粘包拆包解决方案

TCP 是流协议，消息可能被拆分或合并发送。Netty 提供三种解码器：

| 解码器 | 方式 | 场景 |
|--------|------|------|
| `FixedLengthFrameDecoder` | 固定长度 | 消息等长 |
| `DelimiterBasedFrameDecoder` | 特定分隔符 | 文本协议 |
| **`LengthFieldBasedFrameDecoder`** | **消息头带长度字段** | **生产首选** |

常见编解码链路：

- **入站解码**：`ByteToMessageDecoder` 把字节流拆成业务对象，`LengthFieldBasedFrameDecoder` 先解决半包/粘包，再交给业务协议 Decoder
- **出站编码**：`MessageToByteEncoder` 把业务对象编码为字节，`LengthFieldPrepender` 可自动补长度字段，和长度字段解码器配套使用

## 六、心跳机制

`IdleStateHandler(readerIdle, writerIdle, allIdle)`：只负责**检测**空闲（读/写/双向），不发心跳包。心跳逻辑写在下一个 ChannelHandler 中，收到 `IdleStateEvent` 后主动发心跳包或关闭连接。

## 七、断线重连与关闭链路

### 7.1 断线重连

- **启动失败重连**：客户端 `connect()` 返回 `ChannelFuture`，通过 `addListener()` 判断失败并调度下一次连接
- **运行中断开重连**：在 `channelInactive()` 中触发重连，通常使用 `EventLoop.schedule()` 做延迟重试
- **退避策略**：生产环境不要固定间隔无限重试，建议指数退避 + 最大间隔，避免服务端故障时形成重连风暴

```java
bootstrap.connect(host, port).addListener((ChannelFutureListener) future -> {
    if (!future.isSuccess()) {
        future.channel().eventLoop().schedule(this::connect, 3, TimeUnit.SECONDS);
    }
});
```

### 7.2 关闭清理链路

```
channel.close()
    → pipeline.fireChannelInactive()
    → deregister() 从 Selector 移除 SelectionKey
    → ChannelOutboundBuffer 失败并释放未写出的消息
    → ByteBuf release() 引用计数归零
    → 堆外内存归还给 PooledByteBufAllocator
```

关闭链路的关键不是只断开 Socket，而是确保出站缓冲、Selector 注册关系和 ByteBuf 引用都被正确释放，否则容易出现堆外内存泄漏或无效 fd 残留。

## 八、TCP 参数调优

```java
ServerBootstrap bootstrap = new ServerBootstrap()
    .group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    // 服务端 Socket 选项
    .option(ChannelOption.SO_BACKLOG, 1024)          // accept 队列大小（防 SYN 洪水）
    .option(ChannelOption.SO_REUSEADDR, true)         // 端口复用（快速重启）
    // 子 Channel（每条连接）选项
    .childOption(ChannelOption.TCP_NODELAY, true)     // 禁用 Nagle 算法（游戏/IM 实时场景必须）
    .childOption(ChannelOption.SO_KEEPALIVE, true)    // TCP 层心跳，检测死连接
    .childOption(ChannelOption.SO_RCVBUF, 4 * 1024 * 1024)  // 接收缓冲 4MB（跨地域高延迟需大窗口）
    .childOption(ChannelOption.SO_SNDBUF, 4 * 1024 * 1024)  // 发送缓冲 4MB
    .childOption(ChannelOption.WRITE_BUFFER_WATER_MARK,
        new WriteBufferWaterMark(512 * 1024, 1024 * 1024))   // 写缓冲水位：背压保护
    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT); // 池化内存分配
```

| 参数 | 作用 | 典型值 |
|------|------|--------|
| `TCP_NODELAY` | 禁用 Nagle：小包立即发送，不等 MSS 攒批 | `true`（游戏/IM 必须） |
| `SO_RCVBUF/SNDBUF` | 调大 TCP 窗口：高 RTT 链路需大窗口才能打满带宽 | 4MB（跨地域），256KB（同机房） |
| `SO_KEEPALIVE` | 2h 无数据后 TCP 层发探测包，清除死连接 | 应配合 `IdleStateHandler` 应用层心跳 |
| `WriteBufferWaterMark` | 超 high 水位关闭 `channel.isWritable()`；降到 low 水位重开 | 防慢客户端导致内存溢出 |
| `SO_BACKLOG` | 内核半连接队列大小，高并发下调大防连接排队超时 | 1024-4096 |

**`TCP_NODELAY` 原理**：Nagle 算法将小包积累到一个 MSS（1460字节）才发送，适合 FTP 大文件；对游戏、IM（每包几十字节）会引入 200ms+ 额外延迟。关闭后包立即发出，延迟降低 20-30%。

**写缓冲水位背压实践**：
```java
if (ctx.channel().isWritable()) {
    ctx.writeAndFlush(msg);
} else {
    // 暂停接收上游消息，等待缓冲区回落到 low 水位
    upstreamChannel.config().setAutoRead(false);
}
```

## 九、高性能设计要点

1. **无锁串行化**：同 Channel 固定 EventLoop，单线程执行，无锁竞争
2. **内存池**（PooledByteBufAllocator）：减少堆外内存频繁分配，类 jemalloc 分级管理
3. **零拷贝**：DirectByteBuf 避免 JVM 堆到内核的拷贝，CompositeByteBuf/slice/duplicate 避免业务层拼接复制，FileRegion 支持 sendfile 文件传输
4. **高性能队列**（MPSC）：多生产者单消费者队列，减少 CAS 竞争
5. **epoll 边缘触发**：Linux 下自动选用 epoll，仅通知就绪事件，O(1)

## 十、关键权衡

1. **EventLoop 单线程无锁 vs 耗时操作阻塞**：ChannelHandler 内不能做数据库查询、RPC 调用等耗时操作，必须提交到业务线程池（`ctx.executor().submit()`），否则阻塞整个 EventLoop 的 IO 处理
2. **ByteBuf 需手动 release**：DirectByteBuf 在堆外，GC 无法管理；若忘记 release，会有堆外内存泄漏（ResourceLeakDetector 可检测）
3. **Boss/Worker 线程数**：BossGroup 默认 1 个线程（accept 很快，单线程足够）；WorkerGroup 默认 `2 × CPU 核数`
4. **Netty vs Tomcat**：Netty 关注网络数据传输（任意协议），Tomcat 是 Servlet 容器（HTTP 协议）；Netty 可定制性更高但需自定义协议处理

## 十一、与其他概念的关系

- **依赖 [[概念-IO模型]]**：Netty 的 BIO/NIO/AIO 对应同步阻塞/同步非阻塞/异步，底层 epoll 是 NIO 多路复用的 Linux 实现
- **支撑 [[机制-Dubbo]]**：Dubbo 底层网络传输基于 Netty；Dubbo 的 Provider/Consumer 通信通道就是 Netty Channel
- **应用 [[主题-设计模式]]**：ChannelPipeline = 责任链模式；ByteBufAllocator = 工厂模式；SelectStrategy = 策略模式；EventExecutorChooser = 策略模式

## 十二、应用边界

**适合 Netty**：需要高性能网络通信的底层框架（RPC、WebSocket、IM 系统、游戏服务器）；需要自定义协议的场景（二进制协议、混合协议）；服务端需要承载 10万+ 长连接。

**不适合 Netty**：标准 HTTP REST 服务（用 Tomcat/Spring MVC 更简单）；简单的内部调用（直接用 HTTP Client 即可）。
