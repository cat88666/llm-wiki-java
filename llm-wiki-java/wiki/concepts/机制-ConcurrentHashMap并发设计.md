---
type: concept
status: active
name: "ConcurrentHashMap并发设计"
layer: L4
aliases: ["ConcurrentHashMap", "CHM", "分段锁", "节点锁", "fail-safe", "COW"]
related:
  - "[[机制-HashMap底层实现]]"
  - "[[机制-CAS]]"
  - "[[机制-synchronized]]"
  - "[[概念-线性数据结构]]"
sources:
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap是如何保证线程安全的？.md"
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap为什么在JDK 1 8中废弃分段锁？.md"
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap为什么在JDK1 8中使用synchronized而不是Reen.md"
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap在哪些地方做了并发控制.md"
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap是如何保证fail-safe的？.md"
  - "../../raw/note/Hollis/集合类/✅为什么ConcurrentHashMap不允许null值？.md"
  - "../../raw/note/Hollis/集合类/✅什么是COW，如何保证的线程安全？.md"
  - "../../raw/note/Hollis/集合类/✅什么是fail-fast？什么是fail-safe？.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# ConcurrentHashMap并发设计

> 线程安全的哈希表，JDK 7 用分段锁（Segment），JDK 8 改用 CAS + synchronized 节点锁，将锁粒度从"一段"细化到"单个桶"。

## 第一性原理

线程安全的 Map 有三个选项：
1. `Hashtable`：方法级 synchronized，整张表一把锁，并发度 = 1，吞吐量差
2. `Collections.synchronizedMap()`：同 Hashtable，锁更粗
3. `ConcurrentHashMap`：锁拆分——不同桶的写操作可并行，锁粒度越细并发度越高

JDK 7 → JDK 8 的演变核心：**锁粒度从"段（16个桶一组）"细化到"单个桶的头节点"**，并大量用 CAS 替换加锁操作。

## 核心机制

### JDK 7：分段锁（Segment + ReentrantLock）

```
数组被分为 16 个 Segment，每个 Segment 继承 ReentrantLock
写操作只锁对应 Segment，不同 Segment 可并行
并发度 = Segment 数（默认 16，初始化后不可扩展）
```

**缺点**：Segment 固定 16 个，高并发下热点 Segment 仍是瓶颈；每个 Segment 是独立哈希表，内存额外占用大。

### JDK 8：CAS + synchronized 节点锁

**并发控制点**：

| 操作阶段 | 控制手段 |
|----------|---------|
| 初始化桶数组 | `sizeCtl` 变量 + CAS 自旋，防止多线程同时初始化 |
| put 到空桶 | `casTabAt()` CAS 无锁插入 |
| put 到非空桶 | `synchronized(桶头节点)` 加锁，操作链表或红黑树 |
| 扩容 | 多线程协作迁移，`transferIndex` 用 CAS 分配迁移区段 |

```java
// 简化的 put 逻辑
if (桶为空) {
    casTabAt(tab, i, null, newNode);  // 无锁
} else {
    synchronized (桶头节点) {         // 最小粒度锁
        // 插入链表或红黑树
    }
}
```

### 为什么 JDK 8 选 synchronized 而非 ReentrantLock

JDK 8 锁的竞争场景是单个桶的并发写，概率极低（不同 key 冲突同一桶的概率很小），所以：
- synchronized 在低竞争下停留在偏向锁/轻量级锁，性能与 ReentrantLock 相当
- synchronized 由 JVM 内置，支持锁消除、锁粗化等运行时优化
- ReentrantLock 每个 Node 都需要一个独立对象，内存开销（AQS 队列）比 synchronized 更大

### fail-fast vs fail-safe

| 机制 | 原理 | 代表集合 |
|------|------|---------|
| **fail-fast** | 迭代期间检测 `modCount != expectedModCount`，立即抛 `ConcurrentModificationException` | `ArrayList`、`HashMap` |
| **fail-safe** | 迭代基于快照或弱一致性，不抛异常，但可能看不到最新修改 | `ConcurrentHashMap`、`CopyOnWriteArrayList` |

ConcurrentHashMap 的 fail-safe 实现：迭代器遍历时只需拿到桶头节点（原子更新），不依赖 `modCount`，并发修改不影响迭代。

### COW（Copy-On-Write）

`CopyOnWriteArrayList` / `CopyOnWriteArraySet` 的并发策略：
- **读**：直接读当前数组，无锁
- **写**：复制整个底层数组，在副本上修改，写完后 `volatile` 替换引用

```java
// add 时
synchronized (lock) {
    Object[] newElements = Arrays.copyOf(array, len + 1);
    newElements[len] = e;
    setArray(newElements);  // volatile 写
}
```

**适合读多写少**（如配置白名单）。代价：写时复制整个数组，内存翻倍；写频繁时性能差。

## 关键权衡

1. **不允许 null key/value**：并发场景下 `map.get(key)` 返回 null 存在二义性（不存在 key vs value 本身为 null），无法通过 `contains()` 区分（因为检测过程可能被其他线程修改）。HashMap 单线程可用 `contains()` 消歧，CHM 直接禁止
2. **弱一致性**：迭代器不保证能看到创建后的修改，是 fail-safe 的代价
3. **size() 不精确**：CHM 的 size() 是近似值，多线程并发 put 时统计可能滞后（使用 `mappingCount()` 稍好）
4. **COW 写开销**：`CopyOnWriteArrayList` 每次写都复制整个数组，写多时不适用

## 与其他概念的关系

- 演化自 [[机制-HashMap底层实现]]：同样的数组+链表+红黑树，加了并发层
- 底层用 [[机制-CAS]]：空桶插入、`transferIndex` 分配均用 CAS 无锁操作
- 低竞争下用 [[机制-synchronized]]：节点锁在偏向锁阶段几乎无开销
- 在 L3 并发：与 `ConcurrentLinkedQueue`、`BlockingQueue` 共同构成并发容器体系

## 应用边界

**适合 CHM**：多线程高并发读写 Map；key 分布均匀（避免桶集中）。

**适合 COW**：读极多、写极少；数据量小（写时复制全量）；可容忍读到旧数据。

**不要用 Hashtable / synchronizedMap**：全局锁，并发度 = 1，无优势场景。
