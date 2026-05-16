---
type: concept
status: active
name: "Java集合"
layer: L4
aliases: ["Java集合框架", "Collection", "List", "Set", "Map", "Queue", "JCF", "数组", "链表", "栈", "队列", "Stack", "Array"]
related:
  - "[[机制-HashMap]]"
  - "[[机制-红黑树]]"
  - "[[机制-优先队列]]"
  - "[[概念-图]]"
  - "[[机制-B+树]]"
  - "[[概念-BitMap]]"
  - "[[概念-前缀树]]"
  - "[[机制-HashMap]]"
---

# Java集合

> Java 集合框架（JCF）统一抽象了内存中数据的存储与访问方式，核心关注三件事：元素如何组织（数组/链表/树/哈希）、是否允许重复与排序、是否线程安全。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 数组不够用、统一数据容器抽象 |
| [二、核心机制](#二核心机制) | Collection 体系、Map 体系、并发容器、类层次全景 |
| [三、Java 核心使用](#三java-核心使用) | HashMap 原理、ConcurrentHashMap 原理、CopyOnWriteArrayList 原理、fail-fast vs fail-safe |
| [四、核心使用原则](#四核心使用原则) | 集合选型决策树 |
| [五、综合对比](#五综合对比) | List/Set/Map/Queue 横向对比 |
| [六、生产风险](#六生产风险) | 常见误区与踩坑 |
| [七、与其他概念的关系](#七与其他概念的关系) | 数据结构、并发、存储 |
| [八、应用边界](#八应用边界) | 并行流、并发容器边界 |
| [九、L4数据结构全景](#九l4数据结构全景) | 树/堆/图/BitMap知识地图，高频考点速查 |

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
| LinkedBlockingQueue | 链表 | 可选 | 是 | 生产-消费模型（两把锁，吞吐高于 ArrayBlockingQueue）|
| ArrayBlockingQueue | 数组 | 有界 | 是 | 生产-消费模型（固定容量，单锁）|
| SynchronousQueue | 无容量（直接移交）| 无 | 是 | 生产者必须等消费者接收才能返回，用于线程池直接提交（`Executors.newCachedThreadPool`）|
| DelayQueue | 堆（PriorityQueue）| 无界 | 是 | 延迟任务、定时调度（元素需实现 `Delayed` 接口）|

### 2.6 线性结构基础：数组 · 链表 · 栈 · 队列

**数组 vs 链表**（底层原理层）：

| 维度 | 数组 | 链表 |
|------|------|------|
| 内存 | 连续 | 不连续（每个节点存指针）|
| 按下标访问 | O(1) | O(n)（必须从头遍历）|
| 按值查找 | O(n)；有序数组 O(log n) | O(n) |
| 插入/删除（中间）| O(n)（需移动元素）| O(1)（改指针，前提是已有节点引用）|
| 空间利用 | 预分配，可能浪费 | 按需分配，但有指针额外开销 |

**链表变体**：
- **单向链表**：只有 next 指针
- **双向链表**：有 prev + next（LinkedList 底层），支持 O(1) 头尾操作
- **环形链表**：尾节点指向头节点，用于循环缓冲区

**栈（Stack）— LIFO**：
```
push(x) → 压栈   pop() → 弹栈   peek() → 查看栈顶
```
- Java 用法：`Deque<Integer> stack = new ArrayDeque<>()`（不用 `Stack` 类，有同步开销）
- 应用：函数调用栈、括号匹配、表达式求值、DFS 非递归

**队列（Queue）— FIFO**：
```
offer(x) → 入队   poll() → 出队   peek() → 查看队头
```
- 变体：Deque（双端）、循环队列（循环数组防假溢出）、优先队列（堆，见 [[机制-优先队列]]）
- Java 用法：`ArrayDeque`（性能优于 LinkedList，Cache 友好）

**四者选型**：
```
需要 O(1) 随机访问 → 数组（ArrayList）
需要 O(1) 头尾增删 → 链表（LinkedList / ArrayDeque）
只访问"最新加入"的元素 → 栈（ArrayDeque）
只访问"最早加入"的元素 → 队列（ArrayDeque / LinkedBlockingQueue）
```

### 2.7 Set 去重机制与排序接口

**Set 去重依赖**：

| Set 实现 | 去重依赖 | 注意 |
|---------|---------|------|
| HashSet | `hashCode()` + `equals()` | 两者必须同时重写，否则去重失效 |
| TreeSet | `compareTo()`（Comparable）或 `compare()`（Comparator）| equals 与 compareTo 结果不一致时行为异常 |
| LinkedHashSet | 同 HashSet | 额外保留插入顺序 |

**排序接口**：
- `Comparable`（内置排序）：对象自身实现，`compareTo()`，适合有"自然顺序"的对象
- `Comparator`（外置排序）：临时规则，`compare()`，适合多种排序维度或不能修改源类时

```java
// Comparable：对象自带排序
class Student implements Comparable<Student> {
    public int compareTo(Student o) { return this.age - o.age; }
}

// Comparator：临时规则，Lambda 写法
list.sort((a, b) -> a.getName().compareTo(b.getName()));
```

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

### 3.3 CopyOnWriteArrayList 核心问题

**底层原理**：

```
写操作流程
  └─ 加锁（ReentrantLock）                  ← 防止并发写入丢失数据
  └─ 将原数组复制一份新数组
  └─ 写操作在新数组上进行
  └─ 写完后将内部引用（volatile array）指向新数组
  └─ 解锁

读操作流程
  └─ 直接读原数组（无锁）                    ← 读写操作在不同数组上，完全不互斥
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| 内部存储 | `volatile Object[] array`（volatile 保证引用切换的可见性）|
| 写操作 | 加 `ReentrantLock`，整体复制数组后写新数组，写完切换引用 |
| 读操作 | 直接读当前数组，**无锁**，读写互不阻塞 |
| 迭代器 | 构造时拿到当前数组快照，遍历过程中写操作对其不可见（fail-safe）|
| 一致性 | **最终一致性**：读可能读到写操作前的旧数据，不保证实时 |

**适用与不适用**：

| 场景 | 结论 |
|------|------|
| 读多写极少（监听器列表、白名单）| ✅ 适合：读无锁，吞吐高 |
| 数据量大 + 写频繁 | ❌ 不适合：每次写复制整个数组，GC 压力大，内存占用高 |
| 实时性要求高 | ❌ 不适合：读到的可能是写操作前的旧数据 |

### 3.4 fail-fast vs fail-safe

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
| ArrayList.subList() 陷阱 | 返回的是原 List 的**视图**（SubList 内部类），不是新 List；对父 List 做结构性修改（增/删元素）后再访问 subList 会抛 `ConcurrentModificationException` | 需要独立副本时用 `new ArrayList<>(list.subList(1, 3))`；subList 不可序列化 |
| ArrayList 序列化 | `elementData` 是 `transient`，只序列化 `[0, size)` 范围有效元素，节省空间；但自定义序列化时若直接操作 elementData 会丢失数据 | 使用标准 `ObjectOutputStream`，序列化逻辑已由 `writeObject` 正确处理 |

## 七、与其他概念的关系

- L4 数据结构：[[机制-HashMap]] 内桶用数组+链表+[[机制-红黑树]]；[[机制-优先队列]]（堆）是 PriorityQueue 底层
- L3 并发：[[机制-HashMap]] → [[机制-CAS]] + [[机制-synchronized]]
- L5 存储：[[机制-B+树]] 是 InnoDB 索引底层；[[概念-Redis]] ZSet 底层是跳表（有序链表变体）
- 延伸结构：[[概念-图]]（BFS 用队列/DFS 用栈）、[[概念-BitMap]]（紧凑 boolean 数组）、[[概念-前缀树]]（共享前缀字符串检索）

## 八、应用边界

**集合框架能解决的**：单 JVM 内的数据组织、查找、排序、去重、线程安全访问。

**集合框架不能解决的**：
- **持久化**：JVM 重启数据丢失，需要数据库或外部缓存
- **跨 JVM 共享**：需要 Redis、分布式缓存等方案
- **超大数据量**：内存放不下时需要外部排序、分页查询或流式处理
- **分布式并发**：ConcurrentHashMap 只保证单 JVM 内线程安全，跨进程需分布式锁

---

## 九、L4数据结构全景

### 结构全景图

```
数据结构
│
├── 线性结构（按访问模式分类）
│   ├── 数组：随机访问 O(1)，连续内存
│   ├── 链表：O(1) 插删（已有引用），不连续内存
│   ├── 跳表（SkipList）：链表 + 多层索引，O(log n) 查找，Redis ZSet / ConcurrentSkipListMap 底层
│   ├── 栈（LIFO）：ArrayDeque 实现，函数调用/DFS
│   └── 队列（FIFO）：ArrayDeque/LinkedList，BFS/任务调度
│
├── 树结构（利用有序性加速查找）
│   ├── [[机制-红黑树]]      自平衡 BST，O(log n) 增删查，HashMap/TreeMap 底层
│   ├── [[机制-B+树]]        多路平衡，极低树高，MySQL InnoDB 索引底层
│   └── [[概念-前缀树]]      共享公共前缀，O(m) 字符串检索，搜索补全
│
├── 堆（维护动态集合最值）
│   └── [[机制-优先队列]]    完全二叉树 + 数组实现，Top K / TP99 / 定时器
│
├── 图（多对多关系）
│   └── [[概念-图]]          有向/无向，DFS/BFS，Dijkstra 依赖小顶堆
│
└── 空间压缩结构
    └── [[概念-BitMap]]      1 bit 表示整数存在性，布隆过滤器基础，去重/签到
```

### 高频考点速查

**红黑树**：
- 5 条规则：黑根；红节点子必黑；所有路径黑高相同 → 最长路径 ≤ 最短 × 2
- Java 使用：`HashMap`（链表长 ≥ 8 → 红黑树）；`TreeMap`/`TreeSet`
- vs AVL：红黑树旋转次数少，写多读少优选；AVL 查询略快，读多写少优选

**B+ 树**：
- 数据只在叶节点：非叶节点纯路由，单页存更多键，树更矮（通常 3-4 层）
- 叶节点双向链表：天然支持范围查询、顺序扫描
- B 树 vs B+ 树：B 树单点查询可提前终止；B+ 树范围查询更优

**堆 / Top K**：
- 反直觉口诀：**找最大 K → 用小顶堆**（维护 K 个候选，新元素 > 堆顶才替换）
- 原因：小顶堆只存 K 个元素，内存 O(k)；大顶堆需存全量数据
- Java：`PriorityQueue`（默认小顶堆）；大顶堆 `new PriorityQueue<>(Comparator.reverseOrder())`

**跳表（SkipList）**：
- 链表 + 多层索引，每层索引以概率 1/2 晋升，期望层高 O(log n)
- 查找/插入/删除均 O(log n)，范围查询效率高（底层链表有序，顺序扫描不需要额外操作）
- Java：`ConcurrentSkipListMap`（并发有序 Map，替代 TreeMap 的线程安全方案）
- vs 红黑树：实现更简单；并发场景下跳表做分段/无锁优化更自然；Redis ZSet 采用跳表

**图遍历**：
- BFS：用队列，按层扩散，无权图最短路径
- DFS：用栈/递归，一路到底再回溯，适合连通性、环检测
- Dijkstra：有权最短路，BFS 基础 + 小顶堆选最短距离节点

**BitMap 与布隆过滤器**：
- BitMap：1 bit = 1 个整数的存在状态，10 亿整数仅需 125 MB
- 布隆过滤器：k 个哈希 → k 个 bit 位；有误判（假阳性），无漏判（假阴性）；不支持删除

### 结构间依赖

```
线性结构（栈/队列）
    ↓ 是 DFS/BFS 的运行时容器
图遍历（DFS/BFS）
    ↓ BFS + 小顶堆 →
Dijkstra 最短路径 ← 依赖 堆（优先队列）← 数组实现

红黑树 → 磁盘场景多路化 → B+ 树 → MySQL InnoDB 索引底层

BitMap → 扩展为布隆过滤器（多哈希 + 多 bit 位）→ 缓存穿透防护 / URL 去重
```

### L4 与上下层关系

| 上层依赖 | 具体联系 |
|---------|---------|
| L4 容器（集合框架）| `HashMap` 内桶用红黑树；`PriorityQueue` 是堆；`ArrayDeque` 是循环数组 |
| L5 MySQL | InnoDB 索引 = B+ 树；回表 = 二级索引叶节点存主键再查聚簇索引 |
| L5 Redis | ZSet 底层 = 跳表（有序链表变体）；BitMap 命令直接暴露位图 |
| L3 并发 | `PriorityBlockingQueue` 是线程安全堆；`ConcurrentSkipListMap` 是跳表 |
