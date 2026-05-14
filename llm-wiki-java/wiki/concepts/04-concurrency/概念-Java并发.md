---
type: concept
status: active
name: "Java并发"
layer: L3
aliases: ["Java并发体系", "并发编程", "JUC", "Java Concurrency"]
tags: ["#concurrency"]
concepts:
  - "[[概念-JMM]]"
  - "[[机制-synchronized]]"
  - "[[机制-volatile]]"
  - "[[机制-AQS]]"
  - "[[机制-CAS]]"
  - "[[机制-线程池]]"
  - "[[概念-ThreadLocal]]"
sources:
  - "../../raw/note/Hollis/Java并发/"
created: 2026-05-02
updated: 2026-05-02
---

# Java并发

> 本页是 L3 并发编程的知识地图，聚合 7 个核心概念页的依赖关系与高频考点。

---

## 知识地图

```
         硬件层
    ┌─────────────────────────────┐
    │  多核 CPU + 多级缓存（L1/L2/L3）  │
    │  CPU 指令重排 / 时间片切割        │
    └──────────────┬──────────────┘
                   │ 引出三大并发问题
                   ▼
         规范层：[[概念-JMM]]
    ┌─────────────────────────────┐
    │  主内存 / 工作内存模型            │
    │  三大问题：可见性/原子性/有序性     │
    │  happens-before 推理规则        │
    └────────┬──────────┬──────────┘
             │          │
    ┌──────────────────┐  ┌──────────────────┐
    │ [[机制-synchronized]] │  │ [[机制-volatile]]  │
    │ 互斥+三性保证       │  │ 可见性+有序性      │
    │ 偏向→轻量→重量级    │  │ 内存屏障           │
    │ 锁升级             │  │ 不保证原子性        │
    └──────────────────┘  └──────────────────┘

         无锁算法层：[[机制-CAS]]
    ┌─────────────────────────────┐
    │  cmpxchg 硬件原子指令           │
    │  ABA 问题 → AtomicStampedRef  │
    │  忙等待 → LongAdder 分散竞争    │
    └──────────────┬──────────────┘
                   │ 支撑
                   ▼
         框架层：[[机制-AQS]]
    ┌─────────────────────────────┐
    │  volatile state + CLH 双向队列 │
    │  同步队列（锁竞争）             │
    │  条件队列（Condition.await）    │
    │  独占/共享两种模式              │
    └──────────────┬──────────────┘
                   │ 实现
          ┌────────┴────────┐
    ReentrantLock    CountDownLatch/Semaphore

         资源管理层：[[机制-线程池]]
    ┌─────────────────────────────┐
    │  7参数：核心/最大/队列/策略等    │
    │  执行流：核→队→最大→拒绝        │
    │  Worker 机制实现线程复用        │
    └──────────────┬──────────────┘
                   │ 风险交互
                   ▼
         线程隔离：[[概念-ThreadLocal]]
    ┌─────────────────────────────┐
    │  Thread → ThreadLocalMap     │
    │  Entry.key 弱引用（JDK半自动） │
    │  value 强引用（线程池需手动    │
    │  remove，否则内存泄漏）        │
    └─────────────────────────────┘
```

---

## 并发三大问题

| 问题 | 根因 | 解决手段 |
|------|------|---------|
| **可见性** | CPU 多级缓存，各线程工作内存副本不同步 | `volatile`（写回主内存）；`synchronized`（释放锁前刷新）|
| **原子性** | 时间片切割，"读-改-写"可被中断 | `synchronized`；`CAS + AtomicXxx`；`Lock` |
| **有序性** | 编译器/CPU 指令重排 | `volatile`（内存屏障）；`synchronized`（单线程串行）|

---

## 高频考点汇总

### JMM / happens-before
- JMM vs JVM 内存结构的区别（并发规范 vs 物理内存布局）
- 8 条 happens-before 规则，重点：程序次序、监视器锁、volatile 变量规则、传递性
- 内存屏障四种：StoreStore / StoreLoad / LoadLoad / LoadStore

### synchronized
- 锁升级链：无锁 → 偏向锁 → 轻量级锁（CAS）→ 重量级锁（OS 级阻塞）
- JDK 15 废弃偏向锁原因：STW 撤销开销高
- `ObjectMonitor`：`_owner`、`_count`（重入）、`_EntryList`（BLOCKED）、`_WaitSet`（WAITING）
- synchronized vs ReentrantLock 6 维度对比

### volatile
- LOCK 前缀指令 → 写回主内存 → MESI 缓存失效
- 4 种内存屏障插入位置
- 双重检验锁（DCL）必须加 volatile 的原因（new 的三步可被重排）
- **不保证原子性**：`i++` 是 load-add-store 三步

### CAS
- cmpxchg 硬件指令，CPU 锁总线/缓存行保证原子
- ABA 问题 + AtomicStampedReference（版本号解决）
- 忙等待：高竞争时 CPU 浪费 → 用 LongAdder
- CAS ≠ 自旋：自旋是失败后的重试策略，CAS 只是比较替换操作本身

### AQS
- volatile state + CLH 双向 FIFO 队列
- 同步队列（锁竞争）vs 条件队列（await/signal）
- 独占模式（ReentrantLock）vs 共享模式（CountDownLatch）
- park/unpark 实现线程阻塞和唤醒（比 wait/notify 更灵活）

### 线程池
- 7 参数：`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`unit`、`workQueue`、`threadFactory`、`handler`
- 执行流：核心线程 → 队列 → 最大线程 → 拒绝策略
- 4 种拒绝策略：Abort（默认）、Discard、DiscardOldest、CallerRuns
- 不建议用 Executors：newFixed/newCached 均有无界队列或无界线程数的 OOM 风险

### ThreadLocal
- `Thread → ThreadLocalMap → Entry(弱引用key, 强引用value)`
- 弱引用 key 解决了"ThreadLocal 对象无法 GC"的问题，但 value 是强引用
- 线程池复用场景下必须手动 `tl.remove()`，防止 value 泄漏
- JDK 25 引入 `ScopedValue` 替代（不可变、生命周期明确）

---

## 层间依赖

- L3 并发依赖 **L2 JVM**：锁升级利用 JVM 对象头 Mark Word；GC 的弱引用机制支撑 ThreadLocalMap
- L3 并发被 **L5 存储层**使用：MySQL 的 MVCC 是乐观锁思想；Redis 分布式锁是 synchronized 在分布式环境的延伸
- L3 并发被 **L6 框架**使用：Spring AOP 用动态代理，但事务隔离依赖 ThreadLocal 存储连接；Spring 容器本身是线程安全的（无状态 Bean）
