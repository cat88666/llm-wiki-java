---
type: concept
status: active
name: "Java并发"
layer: L3
aliases: ["Java并发体系", "并发编程", "JUC", "Java Concurrency", "java.util.concurrent", "并发三大问题", "可见性", "原子性", "有序性"]
tags: ["#concurrency"]
related:
  - "[[概念-JMM]]"
  - "[[机制-Synchronized]]"
  - "[[机制-Volatile]]"
  - "[[机制-AQS]]"
  - "[[机制-CAS]]"
  - "[[机制-线程池]]"
  - "[[概念-ThreadLocal]]"
  - "[[机制-CompletableFuture]]"
---

# Java 并发

> Java 并发编程体系以 JMM 为规范基础，以 synchronized/volatile 为基本关键字，以 CAS 为无锁原子操作，以 AQS 为 JUC 工具类骨架，以线程池为资源管理中心，以 ThreadLocal 为线程隔离机制，共同解决多核 CPU 下的可见性、原子性、有序性三大问题。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、可见性、原子性、有序性](#一可见性原子性有序性) | 可见性/原子性/有序性的根因与解法 |
| [二、Java 并发知识地图](#二java-并发知识地图) | 各层依赖关系与模块定位 |
| [三、并发高频考点速查](#三并发高频考点速查) | JMM、synchronized、volatile、CAS、AQS、线程池、ThreadLocal |
| [四、并发能力的跨层依赖](#四并发能力的跨层依赖) | L2 JVM / L5 存储 / L6 框架 的并发依赖 |
| [五、Java 并发概念索引](#五java-并发概念索引) | 概念页链接 |
| [六、并发工具选型边界](#六并发工具选型边界) | 各工具的选型决策树 |

## 一、可见性、原子性、有序性

多核 CPU 引入了三个根本性并发问题，Java 并发体系的所有工具都是为了解决其中一个或多个：

| 问题 | 根因 | 核心解法 |
|------|------|---------|
| **可见性** | CPU 多级缓存（L1/L2/L3），各线程工作内存副本不同步 | `volatile`（写回主内存 + MESI 失效）；`synchronized`（退出时刷主内存）|
| **原子性** | CPU 时间片切割，"读-改-写"可在任意步骤被打断 | `synchronized`（monitorenter/exit）；`CAS`（硬件原子指令 cmpxchg）|
| **有序性** | 编译器/CPU 指令重排（as-if-serial 只保证单线程语义）| `volatile`（内存屏障）；`synchronized`（临界区单线程串行）|

**JMM（Java 内存模型）** 是这三个问题的统一规范——定义主内存/工作内存交互规则，用 **happens-before** 作为跨线程可见性推理工具。

## 二、Java 并发知识地图

```
         硬件层（根因）
    ┌──────────────────────────────────┐
    │  多核 CPU + 多级缓存（L1/L2/L3）   │
    │  Store Buffer / Invalidation Queue│
    │  指令重排 / 时间片切割              │
    └──────────────┬───────────────────┘
                   │ 引发三大并发问题
                   ▼
         规范层：[[概念-JMM]]
    ┌──────────────────────────────────┐
    │  主内存 / 工作内存抽象模型           │
    │  happens-before 8条推理规则        │
    │  内存屏障（StoreStore/StoreLoad等）│
    └────────┬─────────────┬────────────┘
             │             │
    ┌──────────────┐  ┌──────────────────┐
    │[[机制-Synchronized]]│  │ [[机制-Volatile]]│
    │ Monitor+三性保证  │  │ 可见性+有序性    │
    │ 偏向→轻量→重量锁  │  │ 不保证原子性     │
    └──────────────┘  └──────────────────┘

         无锁算法层：[[机制-CAS]]
    ┌──────────────────────────────────┐
    │  cmpxchg 硬件原子指令               │
    │  ABA 问题 → AtomicStampedReference │
    │  高竞争 → LongAdder 分段 Cell 数组  │
    └──────────────┬───────────────────┘
                   │ 支撑 state 修改
                   ▼
         框架层：[[机制-AQS]]
    ┌──────────────────────────────────┐
    │  volatile state + CLH 双向队列    │
    │  同步队列（锁竞争）                │
    │  条件队列（Condition.await/signal）│
    │  独占模式：ReentrantLock           │
    │  共享模式：CountDownLatch/Semaphore│
    └──────────────┬───────────────────┘

    ┌──────────────────┐  ┌──────────────────────┐
    │ [[机制-线程池]]    │  │ [[概念-ThreadLocal]]  │
    │ 7参数 ThreadPoolExecutor│  │ Thread→ThreadLocalMap│
    │ 核心→队列→最大→拒绝│  │ 弱引用key+强引用value │
    │ ForkJoinPool 工作窃取│  │ 线程池必须 remove()  │
    └──────────────────┘  └──────────────────────┘

         异步编排层：[[机制-CompletableFuture]]
    ┌──────────────────────────────────┐
    │  Completion 链 + 事件驱动          │
    │  allOf/anyOf 并行聚合              │
    │  thenApply/thenCompose 串行编排    │
    │  exceptionally/handle 异常处理     │
    └──────────────────────────────────┘
```

## 三、并发高频考点速查

### JMM / happens-before

- JMM vs JVM 内存结构的区别：并发规范 vs 物理内存布局，描述维度完全不同
- 8 条 happens-before 规则，高频：监视器锁规则（synchronized 可见性基础）、volatile 变量规则、传递性
- 内存屏障 4 种：StoreStore / StoreLoad（最重）/ LoadLoad / LoadStore
- Store Buffer + Invalidation Queue 是 MESI 不够用的原因，LOCK 指令强制刷 Store Buffer

### synchronized

- 锁升级链：无锁 → 偏向锁（ThreadID 记录，零开销）→ 轻量级锁（CAS Mark Word）→ 重量级锁（OS futex）
- JDK 15 废弃偏向锁：STW 撤销代价高于收益
- ObjectMonitor：`_owner`（持锁线程）、`_count`（重入计数）、`_EntryList`（BLOCKED）、`_WaitSet`（WAITING）
- JIT 优化：锁消除（逃逸分析）、锁粗化（合并循环锁）
- synchronized vs ReentrantLock：后者支持可中断、公平锁、多条件队列、tryLock

### volatile

- LOCK 前缀指令 → 写回主内存 → MESI 缓存失效（其他核缓存行置 Invalid）
- 4 种内存屏障插入位置，StoreLoad 代价最大（刷 Store Buffer）
- DCL 必须加 volatile：`new` 的三步（alloc→init→assign）JIT 可能重排 init 和 assign
- 不保证原子性：`i++` 是 load-add-store 三步，必须用 AtomicInteger

### CAS

- cmpxchg 硬件指令，CPU 缓存行锁保证原子，不依赖 OS 锁
- ABA 问题：`A→B→A` 后 CAS 成功，中间变化被忽略 → `AtomicStampedReference`（版本号）
- 忙等待：高竞争时 CPU 浪费 → `LongAdder`（base + Cell 数组分段）
- CAS ≠ 自旋：CAS 是"比较替换"原子操作；自旋是 CAS 失败后的重试策略，AQS 失败后 park 而非纯自旋

### AQS

- volatile state + CLH 双向 FIFO 队列（双向便于 O(1) 取消等待）
- 同步队列（锁竞争，park 阻塞）vs 条件队列（await 让出锁，signal 移入同步队列）
- 独占模式（ReentrantLock）vs 共享模式（CountDownLatch/Semaphore）
- park/unpark 比 wait/notify 更灵活（不需要持有监视器，支持先 unpark）
- CyclicBarrier 基于 ReentrantLock + Condition 实现（非直接 AQS 子类）

### 线程池

- 7 参数：corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、handler
- 执行流：核心线程 → 任务队列 → 临时线程（达最大）→ 拒绝策略（⚠️ 队列满了才创临时线程）
- 4 种拒绝策略：Abort（默认抛异常）、Discard（静默丢弃）、DiscardOldest（丢旧任务）、CallerRuns（调用者执行）
- 不用 Executors：newFixed/newSingle 无界队列 OOM，newCached 无界线程数 OOM
- ctl 变量：高3位=线程池状态，低29位=工作线程数，原子编码两个信息
- corePoolSize 公式：CPU密集型 = CPU+1；IO密集型 = CPU×2

### ThreadLocal

- `Thread → ThreadLocalMap → Entry(弱引用key, 强引用value)`
- 弱引用 key：ThreadLocal 实例没有外部引用时 GC 回收，key 变 null
- 线程池场景：Thread 长期存活 → ThreadLocalMap 不回收 → **value 永远存活 = 内存泄漏**
- 解法：`try { tl.set(v); ... } finally { tl.remove(); }` 强制清理
- 父子线程传递：普通场景用 InheritableThreadLocal，线程池场景用 TransmittableThreadLocal（阿里 TTL）
- JDK 21+ `ScopedValue`：不可变、生命周期明确，是 ThreadLocal 的长期替代方案

### CompletableFuture

- 底层：`volatile result` + Completion 链表（事件驱动，阶段完成触发下一阶段）
- 默认 ForkJoinPool.commonPool()：IO 任务必须指定自定义线程池，否则 commonPool 线程耗尽
- thenApply（同线程）vs thenApplyAsync（提交线程池）：重计算用 Async
- allOf（AND，全完成）/ anyOf（OR，任一完成）/ thenCombine（双完成合并）
- 异常吞噬：链末必须加 `exceptionally` 或 `handle`，否则静默失败
- join() 抛 CompletionException（unchecked）vs get() 抛 ExecutionException（checked）

## 四、并发能力的跨层依赖

| 依赖方向 | 具体联系 |
|---------|---------|
| **依赖 L2 JVM** | 锁升级利用 JVM 对象头 Mark Word；GC 弱引用机制支撑 ThreadLocalMap key 自动回收 |
| **被 L5 存储层使用** | MySQL MVCC 是乐观锁思想（类 CAS）；Redis 分布式锁是 synchronized 的分布式延伸 |
| **被 L6 框架使用** | Spring 事务依赖 ThreadLocal 绑定数据库连接；Spring 容器本身线程安全（无状态 Bean）；Spring AOP 动态代理不涉及并发但依赖 JVM 类加载 |
| **支撑 L7 分布式** | BlockingQueue 是生产者-消费者的核心；线程池是 RPC 客户端/服务端的执行基础 |

## 五、Java 并发概念索引

- [[概念-JMM]]：并发规范基础，所有关键字的语义来源
- [[机制-Synchronized]]：JVM 内置互斥锁，三性保证
- [[机制-Volatile]]：轻量级可见性+有序性，不保证原子性
- [[机制-CAS]]：硬件级无锁操作，AQS 和 AtomicXxx 的底层
- [[机制-AQS]]：JUC 工具类骨架（ReentrantLock/CountDownLatch/Semaphore）
- [[机制-线程池]]：线程资源管理中心，避免频繁创建销毁
- [[概念-ThreadLocal]]：线程变量隔离，无锁线程安全
- [[机制-CompletableFuture]]：异步任务编排，事件驱动回调

## 六、并发工具选型边界

### 工具选型决策树

```
需要并发控制？
  ├── 多线程共享变量
  │     ├── 只需可见性（一写多读）→ volatile
  │     ├── 需要原子性（计数器/状态转换）→ AtomicXxx / LongAdder
  │     ├── 需要互斥（多步复合操作）→ synchronized 或 ReentrantLock
  │     └── 需要高级特性（中断/超时/公平/多条件）→ ReentrantLock（AQS）
  │
  ├── 线程隔离（不需要共享）→ ThreadLocal
  │
  ├── 线程间协调
  │     ├── 等待 N 个任务完成（一次性）→ CountDownLatch
  │     ├── N 个线程互相等待（可重用）→ CyclicBarrier
  │     └── 控制并发数（许可证）→ Semaphore
  │
  └── 异步执行
        ├── 简单独立任务 → ExecutorService.submit()
        ├── 任务编排（串行/并行/聚合）→ CompletableFuture
        └── 响应式流控（背压）→ Project Reactor / RxJava
```

### 常见误区

| 误区 | 正确认知 |
|------|---------|
| volatile 保证原子性 | volatile 只保证可见性和有序性，`i++` 不是原子的 |
| CAS 一定自旋 | CAS 是操作本身；是否自旋由调用者决定，AQS 失败后 park 挂起 |
| synchronized 性能一定差 | JDK 6+ 引入偏向锁/轻量级锁，低竞争场景性能与 ReentrantLock 相当 |
| Executors.newFixedThreadPool 安全 | 无界 LinkedBlockingQueue，任务堆积会 OOM |
| ThreadLocal 弱引用自动清理 | 弱引用只防止 key 泄漏，value 是强引用，线程池场景必须手动 remove() |
| CompletableFuture 不阻塞 | allOf().join() 仍然阻塞调用线程，应在合适的时机调用 |
