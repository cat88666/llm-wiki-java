---
type: concept
status: active
name: "AQS"
layer: L3
aliases: ["AbstractQueuedSynchronizer", "抽象队列同步器", "同步队列", "条件队列", "CLH队列", "LockSupport", "park/unpark", "ReentrantLock", "CountDownLatch", "Semaphore", "CyclicBarrier", "独占模式", "共享模式"]
related:
  - "[[机制-CAS]]"
  - "[[机制-Volatile]]"
  - "[[机制-Synchronized]]"
  - "[[机制-线程池]]"
---

# AQS（AbstractQueuedSynchronizer）

> AQS 是 JDK 1.5 引入的并发同步框架基础类——内部用 `volatile int state` 表示同步状态，用 CLH 变体双向 FIFO 队列管理等待线程，通过 CAS 修改 state，通过 `LockSupport.park/unpark` 阻塞/唤醒线程；ReentrantLock、CountDownLatch、Semaphore 等均构建于其上。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 同步器骨架的价值、tryAcquire 模板方法 |
| [二、核心机制](#二核心机制) | state、CLH 双向队列、同步队列流程 |
| [三、条件队列](#三条件队列) | Condition.await/signal、与同步队列的关系 |
| [四、独占 vs 共享模式](#四独占-vs-共享模式) | ReentrantLock vs CountDownLatch/Semaphore |
| [五、AQS 实现的工具类](#五aqs-实现的工具类) | ReentrantLock/CountDownLatch/Semaphore/CyclicBarrier 原理 |
| [六、综合对比](#六综合对比) | synchronized vs ReentrantLock、公平锁 vs 非公平锁 |
| [七、与其他概念的关系](#七与其他概念的关系) | CAS、volatile、synchronized |
| [八、应用边界](#八应用边界) | 何时自定义同步器、何时用现成工具 |

## 一、第一性原理

实现互斥锁或共享锁需要两个基本能力：
1. **记录状态**：谁拿着锁、还剩几个许可（state）
2. **管理等待**：让没拿到的线程等待，拿到后唤醒

手写这两件事既繁琐又容易出 bug（死锁、饥饿、内存泄漏）。AQS 的存在理由：**提供可复用的同步器骨架——子类只需实现 `tryAcquire/tryRelease`（或共享模式变体），状态管理和队列操作由 AQS 统一处理**。

模板方法设计：

```java
// AQS 骨架：子类只需实现这些方法
protected boolean tryAcquire(int arg) { throw new UnsupportedOperationException(); }
protected boolean tryRelease(int arg) { throw new UnsupportedOperationException(); }
protected int tryAcquireShared(int arg) { throw new UnsupportedOperationException(); }
protected boolean tryReleaseShared(int arg) { throw new UnsupportedOperationException(); }
```

## 二、核心机制

### 两大核心数据结构

```
AQS
├── volatile int state                      ← 同步状态（CAS 修改）
│     ReentrantLock: 0=空闲, n=重入n次
│     CountDownLatch: 计数器值
│     Semaphore: 剩余许可数
└── CLH 双向 FIFO 队列（同步队列 / Sync Queue）
      head → [dummy] ↔ [Node:线程A] ↔ [Node:线程B] → tail
                        waitStatus: SIGNAL(-1)
```

**state 操作三方法**：
- `getState()`：读取 state
- `setState(int)`：直接设置 state（不保证原子，供子类在确保互斥时调用）
- `compareAndSetState(expect, update)`：CAS 原子修改 state

### 同步队列（Sync Queue）—— 锁竞争流程

```
线程尝试获取资源
  ├── tryAcquire(arg) 成功 → 直接执行
  └── 失败 → 封装成 Node（EXCLUSIVE 或 SHARED）入队尾（CAS addWaiter）
              → 前驱节点 waitStatus 设为 SIGNAL（表示释放时需唤醒后继）
              → LockSupport.park(this) 阻塞当前线程

持有线程释放资源
  → tryRelease(arg)（state 变为 0 或增加许可）
  → 唤醒 head 的后继节点：LockSupport.unpark(successor)
  → 被唤醒线程重新 tryAcquire，成功则出队成为新 head
```

**CLH 双向链表的原因**：取消等待时需要找前驱节点并修改其 waitStatus，单向链表无法 O(1) 完成；双向链表可以。

**Node 的 waitStatus 状态**：
| 值 | 含义 |
|----|------|
| `0` | 初始状态 |
| `SIGNAL(-1)` | 后继节点需要被唤醒（持有者释放时需 unpark 后继）|
| `CANCELLED(1)` | 等待被取消（超时、中断），需从队列移除 |
| `CONDITION(-2)` | 节点在条件队列中等待 |
| `PROPAGATE(-3)` | 共享模式下需传播唤醒 |

### LockSupport.park/unpark

AQS 线程阻塞/唤醒的底层，比 `Object.wait/notify` 更灵活：

| 维度 | `LockSupport.park/unpark` | `Object.wait/notify` |
|------|--------------------------|---------------------|
| 是否需要持有锁 | 不需要 | 必须在 synchronized 块内 |
| unpark 先于 park | 支持（许可证机制，park 直接返回）| 不支持（notify 在 wait 前无效）|
| 中断处理 | park 感知中断但不抛异常，需调用方检查 | wait 抛 InterruptedException |

## 三、条件队列

```
ReentrantLock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();  // 每个 Condition 有独立条件队列
Condition notFull  = lock.newCondition();  // 多条件队列，ObjectMonitor 只有一个

线程调用 notEmpty.await()
  → 释放锁（state 归零，相当于 unlock）
  → 封装 Node（CONDITION 状态）加入 notEmpty 条件队列
  → LockSupport.park() 阻塞

其他线程调用 notEmpty.signal()
  → 条件队列头节点从条件队列移入同步队列（enq）
  → 节点状态从 CONDITION → 0（等待重新竞争锁）
  → 被唤醒线程从 park 返回，重新 tryAcquire 获取锁
```

**同步队列 vs 条件队列**：

| 维度 | 同步队列（Sync Queue）| 条件队列（Condition Queue）|
|------|--------------------|-----------------------|
| 目的 | 管理锁竞争（未获得锁的线程）| 管理条件等待（持锁后主动让出）|
| 每个 AQS 实例数量 | 1 个 | 每个 `Condition` 各 1 个（可多个）|
| 类比 | ObjectMonitor._EntryList | ObjectMonitor._WaitSet |
| 节点移动 | 竞争失败入队，获得锁出队 | await 入队，signal 移入同步队列 |

## 四、独占 vs 共享模式

| 模式 | 子类实现 | state 语义 | 典型实现 |
|------|---------|----------|---------|
| 独占（Exclusive）| `tryAcquire` / `tryRelease` | 0=空闲，>0=被占（重入层数）| `ReentrantLock` |
| 共享（Shared）| `tryAcquireShared` / `tryReleaseShared` | 剩余资源数（≥0 可获取，-1 失败）| `CountDownLatch`、`Semaphore` |

独占模式：同一时刻只有一个线程能成功 `tryAcquire`。

共享模式：多个线程可以同时持有，`tryAcquireShared` 返回值 ≥ 0 表示成功（共享节点会唤醒后继共享节点，形成传播）。

## 五、AQS 实现的工具类

### ReentrantLock（独占 + 可重入）

```java
// state = 0: 无锁；state = n: 同一线程重入 n 次
protected boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {                              // 无锁
        if (CAS(0, acquires)) {                // 非公平：直接 CAS
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {  // 可重入
        setState(c + acquires);
        return true;
    }
    return false;
}
```

**公平锁 vs 非公平锁**：非公平锁直接 CAS 抢 state，抢不到才入队；公平锁先检查队列是否有前驱节点，有则直接入队（避免插队，但吞吐量略低）。

### CountDownLatch（共享 + 一次性）

```java
// state = count（初始值），countDown() 每次 CAS state-1
// state 降为 0 时，所有 await() 线程被唤醒
protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0) return false;
        if (CAS(c, c - 1)) return c == 1;  // 减到 0 时返回 true（触发传播唤醒）
    }
}
```

**不可重置**：state 归零后不能再用，需要重置用 `CyclicBarrier`。

### Semaphore（共享 + 许可证）

```java
// state = 许可证数量，acquire() CAS state-1，release() CAS state+1
protected int tryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 || CAS(available, remaining)) return remaining;
    }
}
```

**控制并发数**：数据库连接池、限流的经典实现。

### CyclicBarrier（非 AQS 直接子类，基于 ReentrantLock + Condition）

```java
// 所有参与线程都调用 await() 时，barrier 打开，所有线程同时继续
// 可重置（reset()），可循环使用，故名 Cyclic
```

与 `CountDownLatch` 的区别：CountDownLatch 一个或多个线程等待 N 个线程完成；CyclicBarrier 是 N 个线程互相等待，全部到达后再同时推进。

## 六、综合对比

### 公平锁 vs 非公平锁

| 维度 | 非公平锁 | 公平锁 |
|------|---------|-------|
| 获取策略 | 新线程先 CAS 抢，失败再入队 | 有前驱节点时直接入队，不插队 |
| 吞吐量 | 更高（减少线程切换）| 较低（严格按序）|
| 公平性 | 可能线程饥饿 | 不会饥饿 |
| 适用 | 默认选非公平 | 任务优先级敏感、防饥饿场景 |

### AQS 工具类横向对比

| 工具类 | 模式 | state 语义 | 是否可重置 | 典型场景 |
|--------|------|----------|----------|---------|
| ReentrantLock | 独占 | 重入层数 | - | 互斥临界区 |
| ReentrantReadWriteLock | 独占+共享 | 高16位=读锁数，低16位=写锁重入 | - | 读多写少 |
| CountDownLatch | 共享 | 倒计数 | 不可 | 主线程等待 N 子线程完成 |
| CyclicBarrier | 基于 Lock+Condition | 到达计数 | 可 | N 线程互相等待 |
| Semaphore | 共享 | 许可证数 | 可增减 | 限制并发数 |

## 七、与其他概念的关系

- 依赖 [[机制-CAS]]：`compareAndSetState` 是 AQS 修改 state 的核心，CAS 是无锁并发的基础
- 依赖 [[机制-Volatile]]：`state` 字段是 volatile，保证多线程间的可见性
- 替代/增强 [[机制-Synchronized]]：AQS 是 Java 层的锁框架，支持可中断、超时、多条件等 synchronized 不具备的特性
- 被 [[机制-线程池]] 使用：`ThreadPoolExecutor` 内部用 `ReentrantLock` 保护 workers 集合，`BlockingQueue` 实现中用 Condition 协调生产者-消费者

## 八、应用边界

**不需要自己实现 AQS 子类的场景**（优先使用现成工具）：
- 互斥 → `synchronized` 或 `ReentrantLock`
- 等待 N 个任务 → `CountDownLatch`
- 控制并发数 → `Semaphore`
- 分阶段等待 → `CyclicBarrier`
- 读多写少 → `ReentrantReadWriteLock`

**需要自定义 AQS 子类的场景**：
- 需要特定的 state 语义（非标准互斥或许可证计数）
- 需要特定的公平性策略
- 框架/中间件开发（如 Redisson 基于 AQS 思想实现分布式锁的 Java 端）
