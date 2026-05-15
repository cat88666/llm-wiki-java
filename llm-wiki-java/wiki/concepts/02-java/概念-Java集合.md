---
type: concept
status: active
name: "Java集合"
layer: L4
aliases: ["Java集合框架", "Collection", "List", "Set", "Map", "Queue", "JCF"]
sources:
  - "../../../raw/note/Hollis/集合类/"
created: 2026-05-06
updated: 2026-05-15
lint_notes: ""
---

# Java集合

> Java 集合框架（JCF）统一抽象了内存中数据的存储与访问方式，核心关注三件事：元素如何组织（数组/链表/树/哈希）、是否允许重复与排序、是否线程安全。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 数组不够用、统一数据容器抽象 |
| [二、核心机制](#二核心机制) | Collection 体系、Map 体系、并发容器、类层次全景 |
| [三、Java 核心使用](#三java-核心使用) | HashMap 原理、ConcurrentHashMap 原理、fail-fast vs fail-safe |
| [四、核心使用原则](#四核心使用原则) | 集合选型决策树 |
| [五、综合对比](#五综合对比) | List/Set/Map/Queue 横向对比 |
| [六、生产风险](#六生产风险) | 常见误区与踩坑 |
| [七、与其他概念的关系](#七与其他概念的关系) | 数据结构、并发、存储 |
| [八、应用边界](#八应用边界) | 并行流、并发容器边界 |

## 一、第一性原理

Java 数组长度固定、类型单一、没有内置去重/排序/线程安全能力。当业务需要动态增长、按 key 查找、有序遍历、并发读写时，裸数组无法满足。

集合框架的本质：**用接口抽象"数据容器的行为"（List/Set/Map/Queue），用不同数据结构实现"不同的性能权衡"（数组/链表/哈希/红黑树/堆）**。选型的核心就是在访问模式、内存开销、线程安全三个维度上做取舍。

## 二、核心机制

### 2.1 类层次全景

```
java.util.Collection
├── List（有序，允许重复）
│   ├── ArrayList        数组实现，O(1)随机访问，扩容1.5x
│   ├── LinkedList       双向链表，O(1)头尾操作，实现 Deque
│   └── Vector           已废弃，方法级synchronized
├── Set（无序，不重复）
│   ├── HashSet          基于 HashMap，hashCode+equals 去重
│   ├── LinkedHashSet    基于 LinkedHashMap，保留插入顺序
│   └── TreeSet          基于 TreeMap（红黑树），自然/自定义排序
└── Queue（FIFO）
    ├── LinkedList       也实现了 Deque
    ├── ArrayDeque       循环数组，替代 Stack 和 LinkedList 队列
    └── PriorityQueue    小顶堆，O(log n) 出队

java.util.Map
├── HashMap              数组+链表+红黑树，非线程安全
├── LinkedHashMap        HashMap + 双向链表维护顺序，可实现 LRU
├── TreeMap              红黑树，key 有序
└── Hashtable            已废弃，方法级 synchronized

并发容器（java.util.concurrent）
├── ConcurrentHashMap    CAS+节点锁，替代 Hashtable
├── CopyOnWriteArrayList 写时复制，读多写少
├── ConcurrentLinkedQueue 无锁队列（CAS）
└── BlockingQueue（接口）
    ├── LinkedBlockingQueue  链表，可选边界
    ├── ArrayBlockingQueue   数组，有界
    └── PriorityBlockingQueue 堆+锁
```

### 2.2 List 体系

| 实现 | 底层结构 | 随机访问 | 插入/删除 | 扩容 | 线程安全 |
|------|---------|---------|----------|------|---------|
| ArrayList | 动态数组 | O(1) | O(n)（中间位置需移动元素）| 1.5 倍 | 否 |
| LinkedList | 双向链表 | O(n) | O(1)（已定位节点时）| 无需 | 否 |
| Vector | 动态数组 | O(1) | O(n) | 2 倍 | 是（方法级 synchronized，性能差）|

### 2.3 Set 体系

| 实现 | 底层结构 | 去重依据 | 有序性 |
|------|---------|---------|--------|
| HashSet | HashMap（value 用固定 Object） | hashCode + equals | 无序 |
| LinkedHashSet | LinkedHashMap | hashCode + equals | 插入顺序 |
| TreeSet | TreeMap（红黑树）| Comparable / Comparator | 自然/自定义排序 |

### 2.4 Map 体系

| 实现 | 底层结构 | key 有序 | 允许 null key | 线程安全 |
|------|---------|---------|--------------|---------|
| HashMap | 数组+链表+红黑树 | 否 | 是（1个）| 否 |
| LinkedHashMap | HashMap + 双向链表 | 插入/访问顺序 | 是 | 否 |
| TreeMap | 红黑树 | 自然/自定义排序 | 否 | 否 |
| Hashtable | 数组+链表 | 否 | 否 | 是（方法级 synchronized）|
| ConcurrentHashMap | 数组+链表+红黑树 | 否 | 否 | 是（CAS+节点锁）|

### 2.5 Queue 体系

| 实现 | 底层结构 | 有界 | 阻塞 | 适用 |
|------|---------|------|------|------|
| ArrayDeque | 循环数组 | 自动扩容 | 否 | 替代 Stack 和 LinkedList 队列用途 |
| PriorityQueue | 小顶堆 | 自动扩容 | 否 | 按优先级出队 |
| LinkedBlockingQueue | 链表 | 可选 | 是 | 生产-消费模型 |
| ArrayBlockingQueue | 数组 | 有界 | 是 | 生产-消费模型（固定容量）|

## 三、Java 核心使用

### 3.1 HashMap 核心问题（高频考点）

| 考点 | 关键答案 |
|------|---------|
| 数据结构 | JDK8：数组+链表+红黑树（链表≥8且容量≥64时树化，≤6时退化）|
| 容量为何是 2^n | `& (n-1)` 代替 `% n`（位运算快+解决负数问题）|
| 负载因子为何 0.75 | 泊松分布下冲突概率与空间利用率的最优平衡 |
| hash 扰动 | `(h = key.hashCode()) ^ (h >>> 16)`，高低位混合减少冲突 |
| JDK7 并发死循环 | 头插法 + 多线程扩容 → 环形链表；JDK8 改尾插法修复 |
| 扩容机制 | `++size > threshold(cap×0.75)` 时触发；JDK8 高低位拆分免 rehash |

### 3.2 ConcurrentHashMap 核心问题（高频考点）

| 考点 | 关键答案 |
|------|---------|
| JDK7 并发方案 | Segment 分段锁（继承 ReentrantLock），并发度 = Segment 数（16）|
| JDK8 并发方案 | 空桶 CAS 插入；非空桶 synchronized 节点锁，粒度细化到单个桶 |
| 为何不用 ReentrantLock | 节点锁竞争低，偏向锁即可；synchronized 有 JVM 运行时优化；ReentrantLock 额外内存（AQS）|
| 不允许 null | 二义性：并发场景无法用 `contains()` 区分"key不存在"与"value为null" |
| fail-safe 实现 | 弱一致性迭代器，不依赖 modCount，并发修改不抛异常 |

### 3.3 fail-fast vs fail-safe

| 机制 | 代表 | 行为 | 原理 |
|------|------|------|------|
| fail-fast | ArrayList、HashMap | 迭代时修改集合 → `ConcurrentModificationException` | 检测 `modCount` 变化 |
| fail-safe | ConcurrentHashMap、CopyOnWriteArrayList | 迭代快照或弱一致，不抛异常 | 不依赖 modCount |

**安全遍历删除**：`iterator.remove()` / `removeIf()` / `Stream.filter()`。

## 四、核心使用原则

### 集合选型决策树

```
需要线程安全？
├── 是 → 读多写少？
│        ├── 是 → CopyOnWriteArrayList（List）/ ConcurrentHashMap（Map）
│        └── 否 → ConcurrentHashMap / BlockingQueue（生产-消费）
└── 否 → 需要排序？
         ├── 是 → TreeMap / TreeSet
         └── 否 → 需要保留插入顺序？
                  ├── 是 → LinkedHashMap / LinkedHashSet
                  └── 否 → HashMap / ArrayList / HashSet
```

**选型核心原则**：
- **默认选 ArrayList + HashMap**，覆盖 90% 场景
- **需要线程安全时不要用 Vector/Hashtable**，用 `java.util.concurrent` 包下的实现
- **需要队列语义时用 ArrayDeque**，不要用 Stack（继承自 Vector，已过时）
- **LRU 缓存用 LinkedHashMap**，重写 `removeEldestEntry()`

## 五、综合对比

### 5.1 ArrayList vs LinkedList

| 维度 | ArrayList | LinkedList |
|------|-----------|------------|
| 随机访问 | O(1)，数组下标直接定位 | O(n)，需从头/尾遍历 |
| 尾部追加 | 均摊 O(1) | O(1) |
| 中间插入 | O(n)，需移动元素 | O(n)（遍历定位）+ O(1)（改指针）|
| 内存连续性 | 连续，CPU Cache 友好 | 非连续，每个节点额外 prev/next 指针开销 |
| 结论 | **绝大多数场景首选** | 仅在频繁头部插入/删除且不需随机访问时考虑 |

### 5.2 HashMap vs TreeMap vs LinkedHashMap

| 维度 | HashMap | TreeMap | LinkedHashMap |
|------|---------|---------|---------------|
| 时间复杂度 | O(1) 均摊 | O(log n) | O(1) 均摊 |
| key 有序 | 否 | 自然/自定义排序 | 插入/访问顺序 |
| 额外开销 | 最低 | 红黑树节点 | 双向链表 |
| 适用 | 通用 KV 查找 | 范围查询、排序需求 | LRU 缓存、保序遍历 |

### 5.3 线程安全容器对比

| 容器 | 机制 | 读性能 | 写性能 | 适用 |
|------|------|--------|--------|------|
| Collections.synchronizedXxx | 全局互斥锁 | 低 | 低 | 临时方案 |
| ConcurrentHashMap | CAS + 节点锁 | 高 | 高 | 通用并发 Map |
| CopyOnWriteArrayList | 写时复制整个数组 | 极高（无锁）| 低（复制开销）| 读多写极少 |
| BlockingQueue 系列 | ReentrantLock + Condition | 中 | 中 | 生产-消费模型 |

## 六、生产风险

| 风险 | 说明 | 应对 |
|------|------|------|
| Vector/Hashtable 仍在用 | 方法级 synchronized 性能差，且复合操作（check-then-act）仍不安全 | 用 ConcurrentHashMap / Collections.synchronizedList + 手动同步 |
| LinkedList 比 ArrayList 插入快 | 不一定。LinkedList 先要 O(n) 遍历到位置；小数据量 ArrayList 的 Cache 连续性优势更大 | 默认用 ArrayList，实测为准 |
| HashMap 多线程 put | JDK7 死循环（环形链表），JDK8 数据丢失 | 并发场景用 ConcurrentHashMap |
| HashMap 的 size() 精确 | 多线程并发 put 时不精确，ConcurrentHashMap 也只是近似 | 精确计数用 AtomicLong 或 LongAdder |
| 并行流一定更快 | 小数据量（< 1000）线程开销 > 计算收益；有状态操作（sorted/limit）需同步反而更慢 | 大数据量 + 无状态操作时才用并行流 |
| CopyOnWriteArrayList 写多场景 | 每次写都复制整个数组，GC 压力大 | 仅用于读多写极少场景（如监听器列表）|

## 七、与其他概念的关系

- L4 数据结构基础：[[机制-HashMap]] → [[机制-红黑树]]（树化）/ [[概念-数据结构]]（数组+链表）
- L3 并发：[[机制-ConcurrentHashMap]] → [[机制-CAS]] + [[机制-synchronized]]
- L5 存储：[[机制-InnoDB索引]] 的 B+ 树是 MySQL 的 TreeMap；[[概念-Redis]] 的 ZSet 类似 TreeMap

## 八、应用边界

**集合框架能解决的**：单 JVM 内的数据组织、查找、排序、去重、线程安全访问。

**集合框架不能解决的**：
- **持久化**：JVM 重启数据丢失，需要数据库或外部缓存
- **跨 JVM 共享**：需要 Redis、分布式缓存等方案
- **超大数据量**：内存放不下时需要外部排序、分页查询或流式处理
- **分布式并发**：ConcurrentHashMap 只保证单 JVM 内线程安全，跨进程需分布式锁
