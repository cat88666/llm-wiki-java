---
type: concept
status: active
name: "ConcurrentHashMap"
layer: L3
aliases: ["CHM", "并发HashMap", "分段锁", "CAS+synchronized"]
related:
  - "[[机制-HashMap]]"
  - "[[机制-CAS]]"
  - "[[机制-synchronized]]"
  - "[[机制-AQS]]"
sources:
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap是如何保证线程安全的？.md"
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap在哪些地方做了并发控制.md"
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap为什么在JDK 1 8中废弃分段锁？.md"
  - "../../raw/note/Hollis/集合类/✅ConcurrentHashMap为什么在JDK1 8中使用synchronized而不是Reen.md"
  - "../../raw/note/Hollis/集合类/✅为什么ConcurrentHashMap不允许null值？.md"
  - "../../raw/note/Hollis/集合类/✅HashMap、Hashtable和ConcurrentHashMap的区别？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# ConcurrentHashMap

> ConcurrentHashMap 是面向并发访问的哈希表，通过缩小锁粒度、CAS 和弱一致遍历，在线程安全与吞吐量之间折中。

## 第一性原理

普通 HashMap 的数组初始化、桶插入、链表/树修改、扩容迁移都可能被多个线程同时破坏。ConcurrentHashMap 的目标不是让所有操作串行，而是只在必要位置同步，让不同桶、不同迁移区间尽量并行。

## 核心机制

1. **JDK 7 分段锁**：用多个 `Segment` 分片，每个段独立加锁，减少整表锁竞争。
2. **JDK 8 CAS + synchronized**：空桶插入用 CAS；非空桶锁桶头节点；冲突概率低时 `synchronized` 成本可控。
3. **初始化控制**：用 `sizeCtl` 和 CAS 控制只有一个线程初始化 table，其他线程让出或重试。
4. **并发扩容**：通过 `ForwardingNode` 标记已迁移桶，用 `transferIndex` 给线程分配迁移区间，多线程协作搬迁。
5. **不允许 null**：避免 `get(key) == null` 在并发下无法区分“不存在”与“存在但值为 null”的语义二义性。

## 关键权衡

- **弱一致遍历**：遍历时不会像 fail-fast 一样抛异常，但可能看到并发修改过程中的部分状态。
- **锁粒度更细但实现更复杂**：性能来自 CAS、节点锁、迁移协作等多处机制组合。
- **复合操作要用原子 API**：`putIfAbsent`、`replace` 等比 `containsKey` + `put` 更可靠。
- **不保证全局强一致快照**：它适合在线并发访问，不适合作为无锁事务容器。

## 与其他概念的关系

- 基于 [[机制-HashMap]] 的哈希桶思想，但补齐并发控制。
- 依赖 [[机制-CAS]]：空桶插入、状态变量、扩容分配都依赖 CAS。
- 依赖 [[机制-synchronized]]：JDK 8 对非空桶头节点加锁。
- 区别于 [[机制-AQS]]：JDK 7 `Segment` 继承 `ReentrantLock`，JDK 8 避免每个节点引入 AQS 对象开销。

## 应用边界

**适合**：
- 多线程共享读写的 key-value 缓存、注册表、计数辅助结构
- 需要高并发 put/get，但不要求遍历获得强一致快照

**不适合**：
- 需要排序，选择 `ConcurrentSkipListMap`
- 需要存储 null key/value
- 需要多步骤事务语义，仍需外部同步或更高层并发模型

