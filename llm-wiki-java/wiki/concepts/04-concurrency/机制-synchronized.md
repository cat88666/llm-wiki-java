---
type: concept
status: active
name: "synchronized"
layer: L3
aliases: ["Monitor", "对象锁", "类锁", "重量级锁", "轻量级锁", "偏向锁", "锁升级", "monitorenter"]
related:
  - "[[概念-JMM]]"
  - "[[机制-CAS]]"
  - "[[机制-AQS]]"
sources:
  - "../../../raw/note/Hollis/Java并发/✅synchronized是怎么实现的？.md"
  - "../../../raw/note/Hollis/Java并发/✅synchronized的锁升级过程是怎样的？.md"
  - "../../../raw/note/Hollis/Java并发/✅synchronized是如何保证原子性、可见性、有序性的？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# synchronized

> synchronized 是 Java 的内置互斥锁——底层通过每个对象的 Monitor（ObjectMonitor）实现，JDK 6 后引入偏向锁→轻量级锁→重量级锁的自适应升级链，在低竞争场景大幅降低同步开销。

## 第一性原理

多线程共享数据时，CPU 时间片切割可让"读-改-写"被打断导致数据不一致。synchronized 的存在理由：**用互斥（同一时刻只有一个线程持有锁）保证代码块的原子性，同时附带保证进入/退出代码块时的内存可见性**。

## 核心机制

### 用法与锁对象

| 用法 | 锁的对象 | 字节码实现 |
|------|---------|----------|
| `synchronized` 实例方法 | `this`（当前对象）| `ACC_SYNCHRONIZED` 标志 |
| `synchronized` 静态方法 | `类.class` 对象 | `ACC_SYNCHRONIZED` 标志 |
| `synchronized(obj) {}` 代码块 | 显式指定的 `obj` | `monitorenter` / `monitorexit` |

**类锁也是对象锁**：`类.class` 也是 Java 对象，万物皆对象。

### Monitor（ObjectMonitor）

每个 Java 对象都关联一个 Monitor（C++ 实现的 ObjectMonitor）：

```
ObjectMonitor
├── _owner     → 持有锁的线程
├── _count     → 锁的重入计数器
├── _EntryList → BLOCKED 状态线程队列（等待获取锁）
└── _WaitSet   → WAITING 状态线程集合（调用了 wait() 的线程）
```

**可重入原理**：同一线程再次获取锁时 `_count + 1`，退出时 `_count - 1`，归零才真正释放锁。

### 三性保证

| 特性 | 机制 |
|------|------|
| **原子性** | monitorenter/monitorexit 保证同一时刻只有一个线程在 synchronized 块内 |
| **可见性** | 退出 synchronized 时，所有修改写回主内存；进入时，从主内存刷新工作内存副本 |
| **有序性** | 单线程串行执行 synchronized 块，as-if-serial 保证看起来有序 |

### 锁升级链（JDK 6+）

```
无锁
  ↓ 首次进入 synchronized（JDK 15 前）
偏向锁（Biased Locking）
  只记录线程 ID，同一线程无需 CAS，几乎零开销
  ↓ 另一线程尝试获取
轻量级锁（Lightweight Lock）
  Mark Word 复制到栈帧 Lock Record，用 CAS 竞争
  成功：获得锁；失败（多次）→ 升级
  ↓ CAS 多次失败
重量级锁（Heavyweight Lock）
  Mark Word 指向 ObjectMonitor，竞争者进入 EntryList 阻塞（OS 级线程切换）
```

**锁只能升级不能降级**（STW 期间可降，正常运行时不降）

**JDK 15 废弃偏向锁**：偏向锁撤销开销高（需要 STW），在多线程竞争场景下弊大于利。

### synchronized vs ReentrantLock

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM 内置关键字 | JDK 类（基于 AQS）|
| 可中断等待 | 不支持 | `lockInterruptibly()` |
| 公平锁 | 非公平 | 可选（公平/非公平）|
| 条件变量 | `wait()/notify()`（1个）| `Condition`（多个）|
| 释放方式 | 自动（退出 synchronized 块）| 必须手动 `unlock()`（需 finally）|
| 性能 | JDK 6+ 差距很小 | 高竞争场景略好 |

## 关键权衡

1. **重量级锁慢的根本原因**：线程阻塞/唤醒需用户态→内核态切换，这个上下文切换代价可能比同步块本身的执行时间还长
2. **偏向锁的撤销代价**：撤销偏向锁需要 STW，多线程高竞争时频繁撤销反而更慢 → JDK 15 废弃
3. **锁粗化与锁消除**：JIT 会合并循环中频繁的 synchronized（锁粗化），也会消除无逃逸对象的 synchronized（锁消除）

## 与其他概念的关系

- 语义建立在 [[概念-JMM]] 上：synchronized 是 JMM 中"监视器锁规则"的实现
- 与 [[机制-AQS]] 对比：synchronized 是 JVM 内置；AQS 是 Java 层面的锁框架（ReentrantLock 基于 AQS）
- 与 [[机制-CAS]] 关联：轻量级锁的竞争用 CAS 实现；CAS 是无锁算法

## 应用边界

**用 synchronized**：简单互斥场景；无需高级特性（中断、多条件队列）；不确定时默认用 synchronized（JVM 持续优化）

**用 ReentrantLock**：需要可中断等待；需要公平锁；需要多个条件队列（`newCondition()`）；需要尝试获取锁（`tryLock()`）

**不要用 synchronized 做 Object 的 wait/notify 替代品**：复杂协调场景用 `CountDownLatch`/`CyclicBarrier`/`Semaphore`
