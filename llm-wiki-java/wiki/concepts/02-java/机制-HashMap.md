---
type: concept
status: active
name: "HashMap"
layer: L4
aliases: ["HashMap", "哈希表", "散列表", "hash冲突", "扩容", "ConcurrentHashMap", "CHM", "分段锁", "节点锁", "fail-safe", "fail-fast", "COW", "CopyOnWriteArrayList"]
related:
  - "[[机制-红黑树]]"
  - "[[概念-Java集合]]"
  - "[[机制-CAS]]"
  - "[[机制-Synchronized]]"
sources:
  - "../../../raw/note/Hollis/集合类/"
---

# HashMap

> 基于"数组+链表+红黑树"的散列表，通过 hash 函数将 key 映射到数组下标实现均摊 O(1) 增删查。整个设计围绕一个目标：**尽量少冲突、冲突了尽量快**。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、HashMap 本质](#sec-2) | hash 映射、冲突处理、O(1) 的代价 |
| [二、数据结构演进](#sec-3) | JDK7 数组+链表 → JDK8 +红黑树、树化/退化阈值 |
| [三、Hash 计算与寻址](#sec-4) | 扰动函数、2^n 容量、& 寻址、tableSizeFor |
| [四、put / get 完整流程](#sec-5) | put 七步、get 三步、hashCode/equals 契约 |
| [五、扩容机制](#sec-6) | 0.75 负载因子、触发条件、JDK8 高低位拆分 |
| [六、线程安全陷阱](#sec-7) | JDK7 死循环机制、JDK8 数据覆盖与 size 不准 |
| [七、ConcurrentHashMap 演进](#sec-8) | JDK7 分段锁 → JDK8 CAS+节点锁、为何选 synchronized |
| [八、fail-fast vs fail-safe](#sec-9) | modCount 检测、弱一致性迭代、COW 写时复制 |
| [九、对比与选型](#sec-10) | HashMap/CHM/Hashtable/COW 全量对比、选型决策树 |
| [十、生产风险](#sec-11) | 可变 key、容量预估、null 二义性、大 Map 扩容卡顿 |

<a id="sec-2"></a>
## 一、HashMap 本质

HashMap 要解决的根本问题：**在无限 key 空间中实现接近 O(1) 的查找**。

核心手段是 hash 函数 — 将任意 key 压缩成有限整数（数组下标），冲突时用链表/红黑树兜底。整个设计的所有细节（2^n 容量、0.75 负载因子、红黑树阈值 8/6）都是围绕"减少冲突 + 冲突后加速查找"这一目标展开。

| 设计决策 | 目的 | 权衡 |
|---------|------|------|
| 2^n 容量 | `& (n-1)` 位运算寻址，快于 `%` | 空间只能按 2 倍增长 |
| 0.75 负载因子 | 泊松分布下冲突率与空间的最优平衡 | 非整数倍，threshold 需取整 |
| 链表 → 红黑树 | 极端冲突时从 O(n) 降到 O(log n) | 树节点内存开销更大（左/右/父指针）|
| 扰动函数 | 高低位混合，减少低位相同导致的桶碰撞 | 额外一次异或运算 |

<a id="sec-3"></a>
## 二、数据结构演进

```
JDK 7：  数组 + 链表（链地址法）
JDK 8：  数组 + 链表 + 红黑树
         链表长度 ≥ 8 且数组容量 ≥ 64 → 树化
         红黑树节点 ≤ 6 → 退化回链表
```

**为什么树化阈值是 8？** 泊松分布计算：负载因子 0.75 下，同一桶超过 8 个元素的概率 < 0.00000006，即正常使用几乎不会触发树化。阈值 8 是"极端防御"而非"常态优化"。

**为什么退化阈值是 6 不是 8？** 如果树化和退化都用 8，刚好 8 个元素时反复增删会导致链表⇄红黑树频繁转换（抖动）。留 2 个间隔（8 树化 / 6 退化）作为缓冲带。

**为什么还要 `MIN_TREEIFY_CAPACITY = 64`？** 数组容量太小时冲突是必然的（桶太少），此时应该优先扩容（让元素分散到更多桶），而不是在小桶上建树。

<a id="sec-4"></a>
## 三、Hash 计算与寻址

### 3.1 扰动函数

```java
// JDK 8
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  // ← 高 16 位异或低 16 位
}
```

**为什么需要扰动？** 数组下标 `(n-1) & hash`，当 n 较小（如 16）时只有低 4 位参与寻址。如果两个 key 的 hashCode 高位不同但低 4 位相同，它们会落在同一桶。异或高低位后，高位的差异也会影响最终下标，减少冲突。

### 3.2 为什么容量必须是 2^n

- **位运算快**：`hash % n` → `hash & (n-1)`，位运算快约 10 倍
- **负数安全**：`%` 对负数结果可能为负，`&` 永远非负
- **扩容友好**：2 倍扩容后，每个元素只需判断 1 个 bit（`hash & oldCap`）即可确定新位置

`tableSizeFor(cap)` 方法通过连续右移 + 或运算，将任意输入向上取到最近的 2^n：

```java
// 输入 10 → 输出 16；输入 17 → 输出 32
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1; n |= n >>> 2; n |= n >>> 4; n |= n >>> 8; n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

<a id="sec-5"></a>
## 四、put / get 完整流程

### 4.1 put 流程（JDK 8）

```
① hash(key)                             ← 扰动计算 hash 值
② 桶数组为空？                           → 首次 resize() 初始化（懒加载）
③ (n-1) & hash 定位桶，桶为空？           → 直接创建 Node 放入
④ 桶不为空，key 相同？                    → 覆盖 value（equals 判定）
⑤ 桶头是 TreeNode？                      → 红黑树插入
⑥ 桶头是链表？                           → 尾插法遍历链表
   ├── 遍历中发现 key 相同               → 覆盖 value
   └── 遍历到末尾未找到                   → 尾部插入新节点
       └── 插入后链表长度 ≥ 8？           → treeifyBin()（容量 < 64 则扩容，≥ 64 则树化）
⑦ ++size > threshold？                   → resize() 扩容
```

### 4.2 get 流程（JDK 8）

```
① hash(key) → (n-1) & hash 定位桶
② 桶头节点 key 匹配？                    → 直接返回
③ 桶头是 TreeNode？                      → 红黑树查找 O(log n)
④ 桶头是链表？                           → 遍历链表逐个 equals 比较
⑤ 未找到                                → 返回 null
```

### 4.3 hashCode / equals 契约

HashMap 正确工作依赖 key 的 hashCode 和 equals 必须满足契约：

| 契约 | 含义 | 违反后果 |
|------|------|---------|
| equals 相等 → hashCode 必须相等 | 同一个 key 必须落在同一个桶 | put 能存，get 找不到（hash 不同，去了别的桶）|
| hashCode 相等 → equals 不一定相等 | 不同 key 可以有相同 hash（冲突）| 正常，链表/红黑树处理 |
| equals 要和 hashCode 一起重写 | 默认 Object.hashCode 是内存地址 | 两个逻辑相等的对象被当成不同 key |

> **面试口径**：HashMap 用 hashCode 定位桶，用 equals 在桶内精确匹配。如果只重写 equals 不重写 hashCode，两个逻辑相等的 key 可能落在不同桶，put 进去的 value 用另一个 equals 相等的 key get 不出来。

<a id="sec-6"></a>
## 五、扩容机制

### 5.1 触发条件

`++size > threshold`，其中 `threshold = capacity × loadFactor（默认 0.75）`。

**0.75 负载因子的权衡**：

| 负载因子 | 冲突率 | 空间利用率 | 扩容频率 |
|---------|--------|-----------|---------|
| 0.5 | 低 | 50%，浪费 | 高 |
| **0.75** | **适中** | **75%，平衡** | **适中** |
| 1.0 | 高，链表变长 | 100% | 低 |

0.75 是泊松分布下冲突概率与空间利用率的最优平衡点（注释中有数学推导）。

### 5.2 JDK 8 扩容优化：高低位拆分

```
旧容量 oldCap = 16（二进制 10000），新容量 newCap = 32（二进制 100000）
对每个节点判断 hash & oldCap：
  == 0 → 留在原位置 index
  != 0 → 迁移到 index + oldCap
```

**为什么这样有效？** 扩容后 `(newCap-1) & hash` 比 `(oldCap-1) & hash` 多看 1 个 bit（就是 oldCap 那个位）。这个 bit 为 0 则位置不变，为 1 则偏移 oldCap。不需要重算 hash，且保留链表相对顺序。

**对比 JDK 7**：全量 rehash，每个节点都要重算 `indexFor(hash, newCapacity)`，且头插法会反转链表顺序。

<a id="sec-7"></a>
## 六、线程安全陷阱

### 6.1 JDK 7：并发扩容死循环

```
线程 A、B 同时触发 resize()，对同一链表执行 transfer()（头插法迁移）：

原链表：a → b → c（桶 3）

线程 A 执行到 e=a, next=b 时挂起
线程 B 完成迁移，头插法反转：c → b → a（桶 7）

线程 A 恢复：
  头插 a → 桶 7 变成 a → c → b → a  ← 环形链表！
  后续 get() 遍历该桶永远不会结束 → CPU 100%
```

**根因**：头插法在并发下反转链表顺序，两个线程交叉执行导致节点互相指向形成环。

### 6.2 JDK 8：无死循环，但仍不安全

JDK 8 改用尾插法，不会反转链表，消除了环形链表问题。但仍有：

| 问题 | 原因 |
|------|------|
| 数据覆盖 | 两线程同时 put 到同一空桶，CAS 失败的那个直接覆盖（无重试）|
| 数据丢失 | 并发扩容时节点可能被遗漏 |
| size 不准 | `++size` 非原子操作，并发下计数偏小 |

> **结论**：HashMap 在任何 JDK 版本下都不是线程安全的。并发场景必须用 ConcurrentHashMap。

<a id="sec-8"></a>
## 七、ConcurrentHashMap 演进

### 7.1 JDK 7：Segment 分段锁

```
ConcurrentHashMap
├── Segment[0]（继承 ReentrantLock）
│   └── HashEntry[] table
├── Segment[1]
│   └── HashEntry[] table
├── ...
└── Segment[15]                   ← 默认 16 个 Segment，初始化后不可扩展
    └── HashEntry[] table

写操作：定位 Segment → 加锁（只锁该 Segment）→ 操作内部 HashEntry 数组
不同 Segment 可并行 → 并发度 = Segment 数（默认 16）
```

**缺点**：Segment 固定 16 个，高并发下热点 Segment 仍是瓶颈；每个 Segment 是独立哈希表，内存额外开销大。

### 7.2 JDK 8：CAS + synchronized 节点锁

| 操作 | 并发控制 |
|------|---------|
| 初始化桶数组 | `sizeCtl` + CAS 自旋，保证只有一个线程初始化 |
| put 到空桶 | `casTabAt()` CAS 无锁插入 |
| put 到非空桶 | `synchronized(桶头节点)` 加锁，操作链表或红黑树 |
| 扩容 | 多线程协作迁移，`transferIndex` CAS 分配迁移区段 |
| 计数 | `baseCount` + `CounterCell[]`（类似 LongAdder 分段计数）|

```java
if (桶为空) {
    casTabAt(tab, i, null, newNode);       // ← 空桶 CAS 无锁
} else {
    synchronized (桶头节点) {               // ← 非空桶锁住头节点
        // 链表尾插 或 红黑树插入
    }
}
```

### 7.3 为什么 JDK 8 选 synchronized 而非 ReentrantLock

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 竞争概率 | 节点锁竞争极低，大部分时间停留在偏向锁/轻量级锁 | 无论竞争高低，始终有 AQS 开销 |
| JVM 优化 | 锁消除、锁粗化、自适应自旋 | 无法享受 JVM 运行时优化 |
| 内存 | 对象头 mark word 复用 | 每个节点需要独立 AQS 队列对象 |

### 7.4 CHM 不允许 null 的原因

并发场景下 `get(key)` 返回 null 存在二义性：

```
返回 null → key 不存在？还是 value 就是 null？
单线程 HashMap：可以先 containsKey() 再 get() 消歧
多线程 CHM：containsKey() 和 get() 之间其他线程可能修改了 Map → 消歧无效
```

Doug Lea 的设计决策：**直接禁止 null，从根源消除二义性**。

<a id="sec-9"></a>
## 八、fail-fast vs fail-safe

| 机制 | 原理 | 代表集合 | 行为 |
|------|------|---------|------|
| **fail-fast** | 迭代器记录 `expectedModCount`，每次 `next()` 检查 `modCount != expectedModCount` | `HashMap`、`ArrayList` | 立即抛 `ConcurrentModificationException` |
| **fail-safe** | 迭代基于快照或弱一致性视图，不依赖 modCount | `ConcurrentHashMap`、`CopyOnWriteArrayList` | 不抛异常，但可能读到旧数据 |

**ConcurrentHashMap 的 fail-safe**：迭代器创建时拿到桶数组引用，遍历过程中其他线程的修改可能可见也可能不可见（弱一致性），但不会抛异常也不会死循环。

**CopyOnWriteArrayList（COW）**：

```
写操作
  └─ 加 ReentrantLock                    ← 防止并发写
  └─ 复制整个底层数组
  └─ 在副本上修改
  └─ volatile 替换数组引用               ← 读线程立即可见新引用
  └─ 解锁

读操作
  └─ 直接读当前数组（无锁）              ← 读到的可能是写之前的快照
```

适用：读极多写极少（监听器列表、配置白名单）。代价：每次写复制整个数组，大数组写频繁时 GC 压力大。

<a id="sec-10"></a>
## 九、对比与选型

### 9.1 全量对比

| 维度 | HashMap | ConcurrentHashMap | Hashtable | CopyOnWriteArrayList |
|------|---------|-------------------|-----------|---------------------|
| 线程安全 | 否 | 是 | 是 | 是 |
| 锁粒度 | 无 | 桶级（CAS+节点锁）| 方法级（整表）| 写时全量复制 |
| null key/value | 允许 | 不允许 | 不允许 | 允许 |
| 并发度 | 单线程 | 高 | 低（=1）| 读高 / 写低 |
| 迭代 | fail-fast | fail-safe | fail-fast | fail-safe（快照）|
| 状态 | 主力 | 主力 | **已废弃** | 读多写少专用 |

### 9.2 选型决策树

```
单线程场景
  ├── 通用 KV                    → HashMap
  ├── 需要 key 排序              → TreeMap
  └── 需要保持插入顺序 / LRU     → LinkedHashMap

并发场景
  ├── 高并发读写 Map             → ConcurrentHashMap
  ├── 读极多写极少的 List        → CopyOnWriteArrayList
  └── 需要并发有序 Map           → ConcurrentSkipListMap

不要用
  └── Hashtable / Collections.synchronizedMap   ← 全局锁，无任何优势
```

<a id="sec-11"></a>
## 十、生产风险

| 风险 | 触发条件 | 后果 | 应对 |
|------|---------|------|------|
| 可变 key | key 对象 put 后被修改，hashCode 变化 | get 时定位到错误的桶，永远找不到 | key 用不可变类型（String/Integer/枚举）|
| 未重写 hashCode | 只重写 equals 不重写 hashCode | equals 相等的两个 key 落在不同桶，get 取不出 put 的值 | equals 和 hashCode 必须一起重写 |
| 未预估容量 | 默认容量 16，大量 put 触发多次扩容 | 频繁 resize 导致性能抖动；大 Map（百万级）扩容时 STW 级卡顿 | `new HashMap<>(expectedSize / 0.75 + 1)` |
| 并发使用 HashMap | 多线程同时 put | JDK7 CPU 100%（死循环）；JDK8 数据丢失 | 用 ConcurrentHashMap |
| CHM 的 size() 不精确 | 多线程并发 put/remove | 统计值可能偏差 | 用 `mappingCount()` 或自行用 LongAdder 计数 |
| COW 写频繁 | 大数组 + 高频写入 | 每次写复制整个数组，GC 压力和内存翻倍 | 仅用于读多写极少场景，数据量别太大 |
