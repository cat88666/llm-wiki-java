---
type: concept
status: active
name: "Netty"
layer: L7
aliases: ["Netty", "NIO框架", "Reactor模型", "ChannelPipeline", "ByteBuf", "主从Reactor", "粘包拆包", "EventLoop", "IdleStateHandler"]
tags: ["#distributed"]
related:
  - "[[概念-IO模型]]"
  - "[[机制-RPC与Dubbo]]"
  - "[[机制-设计模式]]"
sources:
  - "../../../raw/note/tuling/07-Netty.md"
  - "../../../raw/note/tuling/架构设计.md"
created: 2026-05-07
updated: 2026-05-07
lint_notes: ""
---

# Netty

> 基于 NIO 的异步事件驱动网络框架，用**主从 Reactor 多线程模型 + ChannelPipeline 责任链**解决 Java 原生 NIO 使用复杂、select/poll O(n) 轮询低效的问题。

## 第一性原理

Java 原生 NIO 有两个根本问题：① API 繁琐（flip/position/limit 易出错）；② select/poll 每次要遍历所有 fd（O(n)），连接数多时 CPU 浪费在无效轮询。Netty 在 Linux 上使用 **epoll 事件通知**（O(1) 就绪事件回调），并通过 **Channel 绑定固定 EventLoop** 消除锁竞争，从根本上解决这两个问题。

---

## 核心机制

### IO 模型对比

| 模型 | 性质 | 实现 | 特点 |
|------|------|------|------|
| **BIO** | 同步阻塞 | 一连接一线程 | 简单但线程开销大，1000 连接需 1000 线程 |
| **NIO** | 同步非阻塞 | Selector 多路复用 | 一线程监控多连接，靠 select/poll/epoll |
| **AIO** | 异步非阻塞 | 内核回调通知 | Linux 实现不成熟，Java 实际使用极少 |
| **select** | 数组 | fd 限制 1024，轮询 | 简单，但连接数有上限 |
| **poll** | 链表 | 无连接数上限，仍需轮询 | 去掉了 select 的上限但仍是 O(n) |
| **epoll** | 红黑树 + 就绪链表 | 事件回调，O(1) | 仅通知就绪 fd，高并发首选 |

### Reactor 线程模型

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

### 核心组件

**(1) EventLoopGroup / EventLoop**

NioEventLoopGroup 内含多个 NioEventLoop，每个 EventLoop 持有一个固定线程 + 一个 Selector。**同一个 Channel 永远由同一个 EventLoop 处理**，所有 IO 在该线程串行执行 → **天然无锁**。

**(2) ChannelPipeline**

责任链模式 + 双向链表，分 Inbound（入站，head→tail）和 Outbound（出站，tail→head）：

```
读：网络数据 → HeadContext → Decoder → BusinessHandler → TailContext
写：BusinessHandler → Encoder → HeadContext → Socket
```

**(3) ByteBuf**

对比 Java 原生 NIO ByteBuffer（只有 position/limit，读写需 flip）：

| 特性 | ByteBuf | Java ByteBuffer |
|------|---------|-----------------|
| 读写指针 | `readerIndex` / `writerIndex` 分离 | 一个 position，读写需 flip |
| 容量 | 动态扩容 | 固定容量 |
| 零拷贝 | slice/duplicate/CompositeByteBuf | 无 |
| 池化 | PooledByteBufAllocator（类 jemalloc）| 无 |
| 引用计数 | retain() / release()，需手动释放 | GC 管理 |

**四种 ByteBuf 实现**（按分配方式 × 是否池化）：

|  | Pooled（高并发）| UnPooled（正常流量）|
|--|---|---|
| **HeapByteBuf** | 业务处理（JVM堆）| 业务处理 |
| **DirectByteBuf** | Socket IO（堆外，避免内核拷贝）| Socket IO |

### 粘包/拆包解决方案

TCP 是流协议，消息可能被拆分或合并发送。Netty 提供三种解码器：

| 解码器 | 方式 | 场景 |
|--------|------|------|
| `FixedLengthFrameDecoder` | 固定长度 | 消息等长 |
| `DelimiterBasedFrameDecoder` | 特定分隔符 | 文本协议 |
| **`LengthFieldBasedFrameDecoder`** | **消息头带长度字段** | **生产首选** |

### 心跳机制

`IdleStateHandler(readerIdle, writerIdle, allIdle)`：只负责**检测**空闲（读/写/双向），不发心跳包。心跳逻辑写在下一个 ChannelHandler 中，收到 `IdleStateEvent` 后主动发心跳包或关闭连接。

### 完整链路概览

```
服务端启动 → BossGroup select ACCEPT
    → 创建 NioSocketChannel
    → 注册到 WorkerEventLoop.selector

数据读入 → WorkerGroup select READ
    → 从 socket 读数据到 DirectByteBuf
    → pipeline.fireChannelRead(buf)
    → HeadContext → Decoder → BusinessHandler

业务响应 → ctx.write(msg)
    → OutboundHandler 链（出站，tail→head）
    → 经过 Encoder
    → HeadContext → ChannelOutboundBuffer 入队
    → flush() → socket.write()

连接关闭 → channelInactive() → deregister()
    → ChannelOutboundBuffer 清理
    → ByteBuf 引用计数归零 → 归还内存池
```

### 高性能设计要点（5 个）

1. **无锁串行化**：同 Channel 固定 EventLoop，单线程执行，无锁竞争
2. **内存池**（PooledByteBufAllocator）：减少堆外内存频繁分配，类 jemalloc 分级管理
3. **零拷贝**：DirectByteBuf（避免 JVM 堆到内核的拷贝）+ FileRegion（sendfile 文件传输）
4. **高性能队列**（MPSC）：多生产者单消费者队列，减少 CAS 竞争
5. **epoll 边缘触发**：Linux 下自动选用 epoll，仅通知就绪事件，O(1)

---

## 关键权衡

1. **EventLoop 单线程无锁 vs 耗时操作阻塞**：ChannelHandler 内不能做数据库查询、RPC 调用等耗时操作，必须提交到业务线程池（`ctx.executor().submit()`），否则会阻塞整个 EventLoop 的 IO 处理
2. **ByteBuf 需手动 release**：DirectByteBuf 在堆外，GC 无法管理；若忘记 release，会有堆外内存泄漏（ResourceLeakDetector 可检测）
3. **Boss/Worker 线程数**：BossGroup 默认 1 个线程（accept 很快，单线程足够）；WorkerGroup 默认 `2 × CPU 核数`
4. **Netty vs Tomcat**：Netty 关注网络数据传输（任意协议），Tomcat 是 Servlet 容器（HTTP 协议）；Netty 可定制性更高但需自定义协议处理

## 与其他概念的关系

- **依赖 [[概念-IO模型]]**：Netty 的 BIO/NIO/AIO 对应同步阻塞/同步非阻塞/异步，底层 epoll 是 NIO 多路复用的 Linux 实现
- **支撑 [[机制-RPC与Dubbo]]**：Dubbo 底层网络传输基于 Netty；Dubbo 的 Provider/Consumer 通信通道就是 Netty Channel
- **应用 [[机制-设计模式]]**：ChannelPipeline = 责任链模式；ByteBufAllocator = 工厂模式；SelectStrategy = 策略模式；EventExecutorChooser = 策略模式

## 应用边界

**适合 Netty**：需要高性能网络通信的底层框架（RPC、WebSocket、IM 系统、游戏服务器）；需要自定义协议的场景（二进制协议、混合协议）；服务端需要承载 10万+ 长连接。

**不适合 Netty**：标准 HTTP REST 服务（用 Tomcat/Spring MVC 更简单）；简单的内部调用（直接用 HTTP Client 即可）。
