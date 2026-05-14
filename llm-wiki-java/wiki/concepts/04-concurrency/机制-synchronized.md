---
type: concept
status: active
name: "synchronized"
layer: L3
aliases: ["Monitor", "ObjectMonitor", "对象锁", "类锁", "重量级锁", "轻量级锁", "偏向锁", "锁升级", "monitorenter", "monitorexit", "Mark Word", "锁消除", "锁粗化", "可重入锁"]
related:
  - "[[概念-JMM]]"
  - "[[机制-CAS]]"
  - "[[机制-AQS]]"
sources:
  - "../../../raw/note/Hollis/Java并发/✅synchronized是怎么实现的？.md"
  - "../../../raw/note/Hollis/Java并发/✅synchronized的锁升级过程是怎样的？.md"
  - "../../../raw/note/Hollis/Java并发/✅synchronized是如何保证原子性、可见性、有序性的？.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# synchronized

> synchronized 是 Java 的内置互斥锁——底层通过每个对象关联的 ObjectMonitor 实现，JDK 6 引入偏向锁→轻量级锁→重量级锁自适应升级链，同时保证原子性、可见性、有序性三性，无需手动 unlock。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 互斥保证三性、Monitor 抽象 |
| [二、核心机制](#二核心机制) | 用法与锁对象、Mark Word、ObjectMonitor、三性保证 |
| [三、锁升级链](#三锁升级链) | 无锁→偏向→轻量→重量、各阶段原理、JDK 15 废弃偏向锁 |
| [四、JIT 优化](#四jit-优化) | 锁消除、锁粗化 |
| [五、综合对比](#五综合对比) | synchronized vs ReentrantLock 6维度 |
| [六、生产风险](#六生产风险) | 死锁、锁对象误用、wait/notify 注意事项 |
| [七、与其他概念的关系](#七与其他概念的关系) | JMM、CAS、AQS |
| [八、应用边界](#八应用边界) | 何时用 synchronized / ReentrantLock |

## 一、第一性原理

多线程共享数据时，CPU 时间片切割可让"读-改-写"被打断导致数据不一致。synchronized 的存在理由：**用互斥（同一时刻只有一个线程持有 Monitor）保证代码块的原子性，并附带在进入/退出时刷新主内存保证可见性**。

synchronized 是 JVM 层面的内置实现，不需要手动释放锁（异常自动释放），是 Java 并发的最基础保障工具。

## 二、核心机制

### 用法与锁对象

| 用法 | 锁的对象 | 字节码实现 |
|------|---------|----------|
| `synchronized` 实例方法 | `this`（当前对象实例）| `ACC_SYNCHRONIZED` 标志 |
| `synchronized` 静态方法 | `类.class` 对象 | `ACC_SYNCHRONIZED` 标志 |
| `synchronized(obj) {}` 代码块 | 显式指定的 `obj` | `monitorenter` / `monitorexit` |

**类锁也是对象锁**：`类.class` 是 Class 对象，synchronized 锁的永远是 Java 对象，万物皆对象。

**锁对象选择原则**：锁对象必须是所有竞争线程共享的同一对象。常见错误是将 `new Object()` 作为局部变量用于锁，导致每个线程拿到不同对象，锁完全无效。

### Mark Word 与对象头

每个 Java 对象的对象头（Object Header）包含 Mark Word（8 字节），存储锁状态：

```
Mark Word（64-bit JVM）：
┌─────────────────────┬────────┬─────────┬──────┐
│ 锁状态               │ 后2位  │ 说明    │ 高位  │
├─────────────────────┼────────┼─────────┼──────┤
│ 无锁                │ 01     │ hashcode, GC age │
│ 偏向锁               │ 01     │ ThreadID + epoch │
│ 轻量级锁             │ 00     │ 栈帧 Lock Record 指针 │
│ 重量级锁             │ 10     │ ObjectMonitor 指针 │
│ GC 标记              │ 11     │         │
└─────────────────────┴────────┴─────────┴──────┘
```

### ObjectMonitor（重量级锁底层）

每个 Java 对象都关联一个 ObjectMonitor（C++ 实现）：

```
ObjectMonitor
├── _owner     → 持有锁的线程（Thread*）
├── _count     → 可重入计数器（同一线程重入时 +1）
├── _recursions→ 重入层数
├── _EntryList → 竞争失败后 BLOCKED 的线程队列
└── _WaitSet   → 调用 wait() 后 WAITING 的线程集合
```

**可重入原理**：同一线程再次获取锁时 `_count + 1`，退出 synchronized 时 `_count - 1`，归零才真正释放。这让同一线程的嵌套 synchronized 块不会死锁。

**`wait()` 和 `notify()` 必须在 synchronized 块内**：它们是 ObjectMonitor 的操作，离开 Monitor 语义就无效（JVM 会抛 `IllegalMonitorStateException`）。

### 三性保证

| 特性 | 机制 |
|------|------|
| **原子性** | `monitorenter` 获得 Monitor，`monitorexit` 释放；临界区内同一时刻只有一个线程 |
| **可见性** | 退出 synchronized 时（`monitorexit`），所有修改写回主内存；进入时（`monitorenter`），从主内存刷新工作内存 |
| **有序性** | synchronized 块内单线程串行，as-if-serial 保证线程自身看到有序；线程间通过 happens-before 的监视器锁规则保证有序 |

## 三、锁升级链

JDK 6 引入自适应锁升级（Adaptive Locking），根据竞争状态自动选择最优策略：

```
无锁（Unlocked）
  ↓ 首次进入 synchronized（JDK 15 前默认开启偏向锁）
偏向锁（Biased Locking）
  Mark Word 记录持有线程 ID + epoch
  同一线程再次进入：只检查 ThreadID，无 CAS，接近零开销
  适合：单线程反复进出同一 synchronized 块的场景
  ↓ 另一线程尝试获取 → 触发偏向锁撤销（SafePoint 暂停）
轻量级锁（Lightweight Lock）
  当前线程栈帧中分配 Lock Record，存储 Mark Word 快照
  用 CAS 把 Mark Word 替换为 Lock Record 指针
  成功 → 获得轻量级锁；竞争线程 CAS 失败 → 自旋等待
  ↓ 自旋一定次数后仍失败（或有第三个线程加入）
重量级锁（Heavyweight Lock）
  Mark Word 指向堆上的 ObjectMonitor（C++ 对象）
  竞争失败的线程进入 EntryList，OS 级阻塞（futex 系统调用）
  持锁线程释放后唤醒 EntryList 中的线程（OS 调度，代价高）
```

**锁只升不降（正常情况下）**：STW（如 Full GC）期间 JVM 可以批量降级偏向锁，正常运行时不降。

**JDK 15 废弃偏向锁（JEP 374）**：
- 撤销偏向锁需要 SafePoint（STW），高并发场景频繁竞争时撤销开销比偏向锁带来的收益更大
- 现代 JVM 轻量级锁的 CAS 性能已大幅提升，偏向锁收益降低
- JDK 15 标记为废弃，JDK 21 完全移除

### 各阶段性能对比

| 锁状态 | 获取方式 | 开销 | 适用场景 |
|--------|---------|------|---------|
| 偏向锁 | 只检查 ThreadID | 接近零（无 CAS）| 单线程反复进入 |
| 轻量级锁 | CAS | 纳秒级，无系统调用 | 低竞争、短临界区 |
| 重量级锁 | OS mutex（futex）| 微秒级，用户态→内核态 | 高竞争、长临界区 |

## 四、JIT 优化

### 锁消除（Lock Elimination）

JIT 通过逃逸分析发现，某个对象只在单线程中使用（不逃逸到堆），其 synchronized 块是无效的，会直接消除：

```java
// JIT 会消除这里的 synchronized（sb 不逃逸）
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer();  // 局部变量，不共享
    sb.append(a);
    sb.append(b);
    return sb.toString();
}
```

### 锁粗化（Lock Coarsening）

JIT 发现循环体内有频繁加解锁，会合并为一次大锁：

```java
// 原始代码
for (int i = 0; i < 1000; i++) {
    synchronized (lock) { list.add(i); }  // 1000次加解锁
}

// JIT 优化后等效
synchronized (lock) {
    for (int i = 0; i < 1000; i++) { list.add(i); }  // 1次加解锁
}
```

## 五、综合对比

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现层 | JVM 内置关键字（C++ ObjectMonitor）| JDK 类（基于 [[机制-AQS]]）|
| 可中断等待 | 不支持 | `lockInterruptibly()` |
| 尝试加锁（超时）| 不支持 | `tryLock(timeout)` |
| 公平锁 | 非公平 | 可选（`new ReentrantLock(true)`）|
| 条件变量 | 1 个（`wait()/notify()`）| 多个（`newCondition()`）|
| 释放方式 | 自动（退出块/方法时 JVM 保证）| 必须手动 `unlock()`，需 `finally` |
| 性能 | JDK 6+ 优化后差距很小 | 高竞争场景略好（无 STW）|
| 锁状态可查询 | 不支持 | `isLocked()`、`getQueueLength()` |

**选择原则**：默认用 synchronized（JVM 持续优化，代码更简洁）；需要高级特性（可中断、超时、公平锁、多条件队列）时用 ReentrantLock。

## 六、生产风险

| 风险 | 场景 | 解决方案 |
|------|------|---------|
| 死锁 | 两个线程持有不同锁互相等待对方锁 | 固定锁顺序；使用 `tryLock(timeout)` |
| 锁对象误用 | 对局部 `new Object()` 加锁 | 确保锁对象是共享的同一实例 |
| String 作锁对象 | String 常量池可能导致意外共享 | 避免用 String、Integer 等有缓存的类型作锁 |
| wait() 虚假唤醒 | `notify()` 后条件仍不满足 | 用 `while (condition)` 而非 `if (condition)` |
| 锁粒度过大 | 把无关代码放入同一 synchronized 块 | 尽量缩小临界区，只保护真正共享的数据 |
| 持锁做 IO | synchronized 块内执行 DB/HTTP 调用，长时间持锁 | 临界区内只做内存操作，IO 操作移到锁外 |

## 七、与其他概念的关系

- 语义建立在 [[概念-JMM]] 的监视器锁规则上：解锁 happens-before 后续加锁，保证跨线程可见性
- 轻量级锁的 CAS 实现依赖 [[机制-CAS]]：竞争时用 CAS 替换 Mark Word
- 与 [[机制-AQS]] 对比：synchronized 是 JVM 内置；AQS 是 Java 层锁框架（ReentrantLock 基于 AQS），功能更丰富但需手动 unlock
- ObjectMonitor 的 `_EntryList` 类似 AQS 的同步队列；`_WaitSet` 类似 AQS 的条件队列

## 八、应用边界

**用 synchronized**：
- 简单互斥场景，无需高级特性
- 不确定时默认选 synchronized（JVM 持续优化，代码更安全）
- 需要 `wait()/notify()` 协调线程时（注意用 while 检查条件）

**用 ReentrantLock**：
- 需要可中断等待（防止死锁）
- 需要公平锁（防止线程饥饿）
- 需要多个条件队列（如生产者-消费者有两个条件）
- 需要 `tryLock()` 尝试加锁（非阻塞锁）

**不用 synchronized 做的事**：
- 复杂的多线程协调 → `CountDownLatch`/`CyclicBarrier`/`Semaphore`（AQS 子类）
- 高频计数器 → `AtomicInteger` 或 `LongAdder`（CAS 无锁）
