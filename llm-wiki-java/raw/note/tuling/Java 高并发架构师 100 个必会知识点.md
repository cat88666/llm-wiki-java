# Java 高并发架构师 100 个必会知识点

所有者: junk01

1️⃣ CPU / 硬件并发

2️⃣ JVM 内存与并发模型

3️⃣ Java 并发工具

4️⃣ 并发数据结构

5️⃣ 高性能网络与消息系统

6️⃣ 高并发系统架构设计

---

### 一、CPU / 硬件并发（1-15）

1. CPU Cache 层级（L1 / L2 / L3）
2. Cache Line（64 byte）
3. False Sharing（伪共享）
4. MESI Cache 一致性协议
5. NUMA 架构
6. 内存访问延迟模型
7. CPU 指令流水线
8. Instruction Reordering（指令重排）
9. Memory Barrier（内存屏障）
10. CAS CPU 指令（cmpxchg）
11. CPU Context Switch 成本
12. CPU Pipeline Stall
13. 数据局部性（Locality Principle）
14. CPU Branch Prediction
15. CPU Prefetch 机制

核心理解：

```
性能 = CPU cache命中率
```

---

### 二、JVM 内存与并发模型（16-30）

1. Java Memory Model（JMM）
2. Happens-Before 原则
3. volatile 内存语义
4. 指令重排与禁止重排
5. synchronized 实现机制
6. 偏向锁 / 轻量级锁 / 重量级锁
7. ObjectMonitor
8. 锁消除（Lock Elimination）
9. 锁粗化（Lock Coarsening）
10. CAS 原理
11. ABA 问题
12. Unsafe 类
13. VarHandle（JDK9）
14. ThreadLocal 实现
15. GC 对并发系统影响

---

### 三、Java 并发工具（31-50）

1. Thread 生命周期
2. ThreadPoolExecutor 原理
3. 线程池参数设计
4. Future 与 Callable
5. CompletableFuture
6. ForkJoinPool
7. Work Stealing 算法
8. CountDownLatch
9. CyclicBarrier
10. Semaphore
11. ReentrantLock
12. Condition
13. StampedLock
14. LockSupport
15. BlockingQueue
16. ArrayBlockingQueue
17. LinkedBlockingQueue
18. DelayQueue
19. PriorityBlockingQueue
20. SynchronousQueue

---

### 四、并发数据结构（51-65）

1. ConcurrentHashMap（JDK7 Segment）
2. ConcurrentHashMap（JDK8 CAS + synchronized）
3. HashMap 并发问题
4. LongAdder
5. LongAccumulator
6. ConcurrentLinkedQueue
7. CopyOnWriteArrayList
8. CopyOnWriteArraySet
9. SkipList
10. ConcurrentSkipListMap
11. RingBuffer
12. 无锁队列（Lock-Free Queue）
13. MPSC Queue
14. SPSC Queue
15. Disruptor 数据结构

代表系统：

- LMAX Disruptor

---

### 五、高性能网络与消息系统（66-80）

1. Reactor 模式
2. Proactor 模式
3. IO Multiplexing

操作系统机制：

```
select
poll
epoll
kqueue
```

1. 零拷贝（Zero Copy）
2. mmap 内存映射
3. Page Cache
4. Sendfile

网络框架：

1. Netty EventLoop
2. Netty ChannelPipeline
3. Netty ByteBuf 内存池

消息系统：

1. Apache Kafka Log结构
2. Kafka 分区机制
3. Kafka 批处理发送
4. Kafka PageCache优化
5. Kafka 顺序写

---

### 六、高并发系统架构（81-100）

1. Actor Model

框架：

- Akka

---

1. Event Driven Architecture
2. Pipeline 架构
3. Backpressure（背压）
4. Reactive Streams
5. CQRS（读写分离）
6. Event Sourcing

---

高并发系统设计：

1. 限流（Rate Limiting）
2. 滑动窗口算法
3. Token Bucket
4. Leaky Bucket

---

高可用设计：

1. 负载均衡
2. 服务降级
3. 熔断器（Circuit Breaker）
4. 重试机制

---

分布式并发：

1. 分布式锁
2. 一致性协议（Raft）
3. CAP 理论
4. BASE 理论
5. 幂等设计

---

# Java 高并发知识金字塔

```
            系统架构
        Event Driven / CQRS
      ─────────────────────
        网络与消息系统
     Netty / Kafka / ZeroCopy
      ─────────────────────
        并发数据结构
   ConcurrentHashMap / Disruptor
      ─────────────────────
         Java并发工具
      AQS / ThreadPool
      ─────────────────────
          JVM并发模型
      JMM / CAS / volatile
      ─────────────────────
            CPU硬件
       Cache / NUMA
```

---

# 架构师真正的高并发思维

```
CPU Cache
顺序IO
无锁
少线程
事件驱动
```

---