---
type: summary
status: active
name: "Java集合框架"
layer: L4
sources:
  - "../../raw/note/Hollis/集合类/"
created: 2026-05-06
updated: 2026-05-06
---

# 主题 — Java 集合框架

> L4 Java 集合类知识地图，涵盖 List/Set/Map/Queue 体系 + 并发容器 + Stream。

## 知识地图

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

## 高频考点

### HashMap 核心问题

| 考点 | 关键答案 |
|------|---------|
| 数据结构 | JDK8：数组+链表+红黑树（链表≥8且容量≥64时树化，≤6时退化） |
| 容量为何是 2^n | `& (n-1)` 代替 `% n`（位运算快+解决负数问题） |
| 负载因子为何 0.75 | 泊松分布下冲突概率与空间利用率的最优平衡 |
| hash 扰动 | `(h = key.hashCode()) ^ (h >>> 16)`，高低位混合减少冲突 |
| JDK7 并发死循环 | 头插法 + 多线程扩容 → 环形链表；JDK8 改尾插法修复 |
| 扩容机制 | `++size > threshold(cap×0.75)` 时触发；JDK8 高低位拆分免 rehash |

### ConcurrentHashMap 核心问题

| 考点 | 关键答案 |
|------|---------|
| JDK7 并发方案 | Segment 分段锁（继承 ReentrantLock），并发度 = Segment 数（16） |
| JDK8 并发方案 | 空桶 CAS 插入；非空桶 synchronized 节点锁，粒度细化到单个桶 |
| 为何不用 ReentrantLock | 节点锁竞争低，偏向锁即可；synchronized 有 JVM 运行时优化；ReentrantLock 额外内存（AQS） |
| 不允许 null | 二义性：并发场景无法用 `contains()` 区分"key不存在"与"value为null" |
| fail-safe 实现 | 弱一致性迭代器，不依赖 modCount，并发修改不抛异常 |

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

### fail-fast vs fail-safe

- **fail-fast**（ArrayList、HashMap）：迭代时修改集合 → `modCount` 不匹配 → `ConcurrentModificationException`
- **fail-safe**（ConcurrentHashMap、CopyOnWriteArrayList）：迭代快照或弱一致，不抛异常
- **安全遍历删除**：`iterator.remove()` / `removeIf()` / `Stream.filter()`

## 与其他层级的联系

- L4 数据结构基础：[[机制-HashMap]] → [[机制-红黑树]]（树化）/ [[概念-数据结构]]（数组+链表）
- L3 并发：[[机制-ConcurrentHashMap]] → [[机制-CAS]] + [[机制-synchronized]]
- L5 存储：[[机制-InnoDB索引]] 的 B+ 树是 MySQL 的 TreeMap；[[概念-Redis数据类型与底层结构]] 的 ZSet 类似 TreeMap

## 常见误区

1. **Vector 还是线程安全的首选**：错。`synchronized` 方法锁性能差，且复合操作不安全；用 CHM 或 Collections.synchronizedList + 手动锁
2. **LinkedList 比 ArrayList 插入快**：不一定。ArrayList 按下标 add(index, e) 需移动元素，但 Cache 连续；LinkedList 先要遍历到位置（O(n)），再改指针（O(1)）；小数据量 ArrayList 通常更快
3. **HashMap 的 size() 精确**：多线程并发 put 时不精确，CHM 也只是近似
4. **并行流一定更快**：小数据量（< 1000）线程开销 > 计算收益；有状态操作（sorted/limit）需同步反而更慢
