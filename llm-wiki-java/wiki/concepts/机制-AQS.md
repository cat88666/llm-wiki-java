---
type: concept
status: active
name: "AQS"
layer: L3
aliases: ["AbstractQueuedSynchronizer", "抽象队列同步器", "同步队列", "条件队列", "CLH队列"]
related:
  - "[[机制-CAS]]"
  - "[[机制-volatile]]"
  - "[[机制-synchronized]]"
sources:
  - "../../raw/note/Hollis/Java并发/✅如何理解AQS？.md"
  - "../../raw/note/Hollis/Java并发/✅AQS的同步队列和条件队列原理？.md"
  - "../../raw/note/Hollis/Java并发/✅AQS是如何实现线程的等待和唤醒的？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# AQS（AbstractQueuedSynchronizer）

> AQS 是 JDK 1.5 引入的并发同步框架基础类——内部用 volatile int state 表示同步状态，用 CLH 双向 FIFO 队列管理等待线程，通过 CAS 修改 state，通过 LockSupport.park/unpark 阻塞/唤醒线程；ReentrantLock、CountDownLatch、Semaphore 均构建于其上。

## 第一性原理

实现互斥锁或共享锁需要两个基本能力：①记录"谁拿着锁"（状态）②让"没拿到锁"的线程等待（队列）。手写这两件事既繁琐又容易出错。AQS 的存在理由：**提供一个可复用的同步器骨架——子类只需实现 tryAcquire/tryRelease（或 tryAcquireShared/tryReleaseShared），状态管理和队列操作由 AQS 统一处理**。

## 核心机制

### 两大核心数据结构

```
AQS
├── volatile int state         ← 同步状态（CAS 修改）
│     state=0: 锁空闲
│     state=1: 锁被占用（重入时 state 累加）
└── CLH 双向 FIFO 队列（同步队列）
      head → [dummy] ↔ [Node:线程A] ↔ [Node:线程B] → tail
```

**state 操作三方法**：`getState()`、`setState()`、`compareAndSetState()`（CAS）。

### 同步队列（Sync Queue）—— 锁竞争

```
线程尝试获取锁（CAS state）
  ├── 成功 → 直接执行
  └── 失败 → 封装成 Node 入队尾 → LockSupport.park() 阻塞

持锁线程释放锁
  → state 归零 → 唤醒队头后继节点 → LockSupport.unpark()
  → 被唤醒线程重新 CAS 竞争 state
```

**CLH 双向链表的原因**：取消等待时需要找到前驱节点将自己移除，单向链表无法 O(1) 完成，双向链表可以。

### 条件队列（Condition Queue）—— 条件等待

```
ReentrantLock lock = new ReentrantLock();
Condition cond = lock.newCondition();  // 每个 Condition 有独立条件队列

线程调用 cond.await()
  → 释放锁（state 归零）
  → 当前线程封装成 Node 加入条件队列
  → LockSupport.park() 阻塞

其他线程调用 cond.signal()
  → 条件队列头节点移入同步队列
  → 等待重新竞争锁
```

**同步队列 vs 条件队列**：

| 维度 | 同步队列 | 条件队列 |
|------|---------|---------|
| 目的 | 管理锁竞争 | 管理条件等待 |
| 数量 | 每个 AQS 实例一个 | 每个 Condition 一个（可多个）|
| 接口 | AQS 自动管理 | 通过 `Condition.await/signal` 显式控制 |
| 类比 | ObjectMonitor 的 EntryList | ObjectMonitor 的 WaitSet |

### 独占模式 vs 共享模式

| 模式 | 子类实现 | 典型实现 |
|------|---------|---------|
| 独占（Exclusive）| `tryAcquire` / `tryRelease` | ReentrantLock |
| 共享（Shared）| `tryAcquireShared` / `tryReleaseShared` | CountDownLatch、Semaphore |

### 公平锁 vs 非公平锁

- **非公平锁**：新线程直接 CAS 抢 state，抢不到才入队
- **公平锁**：先检查同步队列是否有前驱节点，有则直接入队，避免插队

## 关键权衡

1. **LockSupport.park 比 Object.wait 更灵活**：park 不需要持有监视器锁，且 unpark 可以"先于" park 调用（许可证机制）
2. **CLH 双向链表的取消开销**：取消等待需要 O(1) 找前驱，但维护双向指针增加了复杂度
3. **CAS 重试开销**：公平锁减少了无效竞争，非公平锁吞吐量更高但可能产生饥饿

## 与其他概念的关系

- 依赖 [[机制-CAS]]：`compareAndSetState` 是 AQS 修改 state 的核心手段
- 依赖 [[机制-volatile]]：`state` 字段是 volatile，保证可见性
- 替代 [[机制-synchronized]]：AQS 是 Java 层的锁框架；synchronized 是 JVM 内置关键字
- ReentrantLock、CountDownLatch、Semaphore 均是 AQS 的具体实现

## 应用边界

**适合用 AQS 的场景（自定义同步器）**：
- 需要可中断等待（`lockInterruptibly()`）
- 需要超时获取锁（`tryLock(timeout)`）
- 需要多个条件队列（多个 `Condition`）
- 需要公平锁选项

**不要自己实现 AQS 子类的场景**：
- `java.util.concurrent` 包中已有现成工具（ReentrantLock、Semaphore、CountDownLatch）
- 简单互斥场景直接用 `synchronized`
