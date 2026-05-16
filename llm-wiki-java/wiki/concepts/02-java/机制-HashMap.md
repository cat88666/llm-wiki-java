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
  - "[[机制-synchronized]]"
---

# HashMap

> 基于"数组+链表+红黑树"的散列表，通过 hash 函数将 key 映射到数组下标，平均 O(1) 增删查，代价是哈希冲突和扩容。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 散列、冲突、近似 O(1) |
| [二、核心机制](#二核心机制) | 扰动函数、寻址、扩容、树化 |
| [三、ConcurrentHashMap](#三concurrenthashmap) | 分段锁→节点锁、CAS、fail-safe、COW |
| [四、核心使用原则](#四核心使用原则) | 线程安全、null、弱一致性、容量预估 |
| [五、综合对比](#五综合对比) | HashMap vs CHM vs Hashtable、COW vs CHM |
| [六、与其他概念的关系](#六与其他概念的关系) | 红黑树、CAS、synchronized |
| [七、应用边界](#七应用边界) | 单线程/并发/排序/顺序场景选型 |

## 一、第一性原理

HashMap 要解决的根本问题：**在无限 key 空间中实现接近 O(1) 的查找**。核心手段是 hash 函数——将任意 key 压缩成有限整数（数组下标），冲突时用链表/红黑树兜底。整个设计的所有细节（2^n 容量、0.75 负载因子、红黑树阈值）都是围绕"尽量少冲突、冲突了尽量快"这一目标展开的。

## 二、核心机制

### 数据结构演变

```
JDK 7：  数组 + 链表（链地址法）
JDK 8：  数组 + 链表 + 红黑树（链表长度 ≥ 8 且数组容量 ≥ 64 时树化；≤ 6 时退化回链表）
```

**为什么 JDK 8 引入红黑树？** 链表查找 O(n)，极端哈希冲突时退化严重；树化后降到 O(log n)。阈值 8 基于泊松分布：在负载因子 0.75 下，同一桶超过 8 个元素的概率 < 0.00000006。

### Hash 计算与寻址

```java
// JDK 8 扰动函数
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 数组下标：(n - 1) & hash  —— 等价于 hash % n（仅当 n 为 2^k 时成立）
```

**扰动的原因**：hashCode 的高 16 位在小容量数组中不参与寻址（被与掉了），不同 key 高位不同但低位相同时会堆在同一桶。异或高低位混合后，任何一位变化都影响最终下标，减少冲突。

**为什么容量必须是 2^n**：`%` 运算改为 `& (n-1)` 位运算（快约 10 倍），且负数也能得到正确结果。`tableSizeFor()` 方法通过一系列位运算保证传入任意值后向上取最近 2^n。

### 扩容机制

**触发条件**：`++size > threshold`，其中 `threshold = capacity × 0.75`。

**0.75 负载因子的权衡**：
- 太低（如 0.5）：空间浪费，扩容频繁
- 太高（如 1.0）：冲突严重，链表变长
- 0.75 是泊松分布下冲突概率与空间利用率的最优平衡点

**JDK 8 扩容优化（高低位拆分）**：
```
旧容量 oldCap = 16，新容量 newCap = 32
判断 hash & oldCap：
  == 0：低位链表，留在原 index
  != 0：高位链表，迁移到 index + oldCap
```
JDK 7 全量 rehash（O(n)×冲突）→ JDK 8 按位判断，不重算 hash，且保留链表顺序。

### JDK 7 vs JDK 8 关键差异

| 维度 | JDK 7 | JDK 8 |
|------|-------|-------|
| 数据结构 | 数组+链表 | 数组+链表+红黑树 |
| 插入方式 | 头插法 | 尾插法 |
| 扩容 | 全量 rehash | 高低位拆分，免 rehash |
| 并发死循环 | 头插法导致环形链表 | 尾插法修复，但仍非线程安全 |

## 三、ConcurrentHashMap

### 第一性原理

线程安全的 Map 有三个选项：

| 方案 | 锁粒度 | 并发度 |
|------|--------|--------|
| `Hashtable` | 方法级 synchronized，整表一把锁 | = 1，吞吐差 |
| `Collections.synchronizedMap()` | 同 Hashtable | = 1 |
| `ConcurrentHashMap` | 锁拆分，不同桶可并行 | 高 |

JDK 7 → JDK 8 核心演变：**锁粒度从"段（16 个桶一组）"细化到"单个桶的头节点"**，并大量用 CAS 替换加锁。

### JDK 7：分段锁（Segment + ReentrantLock）

```
数组被分为 16 个 Segment，每个 Segment 继承 ReentrantLock
写操作只锁对应 Segment，不同 Segment 可并行
并发度 = Segment 数（默认 16，初始化后不可扩展）
```

**缺点**：Segment 固定 16 个，高并发下热点 Segment 仍是瓶颈；每个 Segment 是独立哈希表，内存额外占用大。

### JDK 8：CAS + synchronized 节点锁

| 操作阶段 | 控制手段 |
|----------|---------|
| 初始化桶数组 | `sizeCtl` 变量 + CAS 自旋，防止多线程同时初始化 |
| put 到空桶 | `casTabAt()` CAS 无锁插入 |
| put 到非空桶 | `synchronized(桶头节点)` 加锁，操作链表或红黑树 |
| 扩容 | 多线程协作迁移，`transferIndex` 用 CAS 分配迁移区段 |

```java
if (桶为空) {
    casTabAt(tab, i, null, newNode);  // 无锁
} else {
    synchronized (桶头节点) {         // 最小粒度锁
        // 插入链表或红黑树
    }
}
```

### 为什么 JDK 8 选 synchronized 而非 ReentrantLock

- 节点锁的竞争场景（不同 key 冲突同一桶）概率极低，synchronized 在低竞争下停留在偏向锁/轻量级锁，性能与 ReentrantLock 相当
- synchronized 由 JVM 内置，支持锁消除、锁粗化等运行时优化
- ReentrantLock 需要每个 Node 一个独立 AQS 队列对象，内存开销更大

### fail-fast vs fail-safe

| 机制 | 原理 | 代表集合 |
|------|------|---------|
| **fail-fast** | 迭代期间检测 `modCount != expectedModCount`，立即抛 `ConcurrentModificationException` | `ArrayList`、`HashMap` |
| **fail-safe** | 迭代基于快照或弱一致性，不抛异常，但可能看不到最新修改 | `ConcurrentHashMap`、`CopyOnWriteArrayList` |

ConcurrentHashMap 的 fail-safe：迭代器只需拿到桶头节点（原子更新），不依赖 `modCount`，并发修改不影响迭代。

### COW（Copy-On-Write）

`CopyOnWriteArrayList` / `CopyOnWriteArraySet` 的并发策略：

- **读**：直接读当前数组，无锁
- **写**：复制整个底层数组，在副本上修改，写完后 `volatile` 替换引用

```java
synchronized (lock) {
    Object[] newElements = Arrays.copyOf(array, len + 1);
    newElements[len] = e;
    setArray(newElements);  // volatile 写，对读线程立即可见
}
```

**适合读多写少**（如配置白名单）。代价：写时复制整个数组，内存翻倍；写频繁时性能差。

## 四、核心使用原则

**HashMap**：
1. **线程不安全**：JDK 7 并发扩容（头插法）产生环形链表，CPU 100%；JDK 8 改尾插后无死循环，但仍有数据覆盖、size 不准确问题
2. **内存开销**：每个 Node 除 key-value 外还有 hash 和 next 指针；TreeNode 额外存父/左/右指针
3. **null key**：允许一个 null key（存 index 0），允许多个 null value

**ConcurrentHashMap**：
1. **不允许 null key/value**：并发下 `get(key)` 返回 null 存在二义性（key 不存在 vs value 为 null），无法通过 `containsKey()` 消歧（检测过程可能被其他线程修改）；HashMap 单线程可消歧，CHM 直接禁止
2. **弱一致性**：迭代器不保证能看到创建后的修改，是 fail-safe 的代价
3. **size() 近似值**：多线程并发 put 时统计可能滞后，用 `mappingCount()` 稍好

**容量预估**（HashMap/CHM 通用）：若预知元素数量 n，初始化 `new HashMap<>(n / 0.75 + 1)` 避免扩容。

## 五、综合对比

### HashMap vs CHM vs Hashtable

| 维度 | HashMap | ConcurrentHashMap | Hashtable |
|------|---------|-------------------|-----------|
| 线程安全 | 否 | 是 | 是 |
| 锁粒度 | 无 | 桶级（CAS + 节点锁）| 方法级（整表）|
| null key/value | 允许 | 不允许 | 不允许 |
| 并发度 | 单线程 | 高 | 低（= 1）|
| 迭代 | fail-fast | fail-safe（弱一致）| fail-fast |
| 适用场景 | 单线程 | 多线程 | 不推荐使用 |

### CHM vs CopyOnWriteArrayList

| 维度 | ConcurrentHashMap | CopyOnWriteArrayList |
|------|-------------------|----------------------|
| 数据结构 | 哈希表 | 数组 |
| 读 | 无锁 | 无锁 |
| 写 | 桶级锁 / CAS | 复制整个数组 |
| 写开销 | 低（只锁一个桶）| 高（O(n) 复制）|
| 适合 | 高并发读写 | 读极多、写极少 |

## 六、与其他概念的关系

- 底层节点退化为 [[机制-红黑树]]：链表长度 ≥ 8 触发树化，O(log n) 保底
- 依赖 [[概念-Java集合]]：数组（主桶）+ 链表（冲突链）是基本结构
- ConcurrentHashMap 底层用 [[机制-CAS]]：空桶插入、`transferIndex` 分配均用 CAS 无锁
- ConcurrentHashMap 低竞争下用 [[机制-synchronized]]：节点锁在偏向锁阶段几乎无开销
- 在 L5 MySQL：B+ 树索引和 HashMap 都是 hash 思想的体现，但场景不同（磁盘 IO 友好 vs 内存 O(1)）

## 七、应用边界

**适合 HashMap**：单线程 KV 缓存；key 均匀分布；不需要线程安全。

**适合 ConcurrentHashMap**：多线程高并发读写 Map；key 分布均匀（避免桶集中）。

**适合 CopyOnWriteArrayList**：读极多、写极少；数据量小；可容忍读到旧数据（如配置白名单）。

**不要用 Hashtable / synchronizedMap**：全局锁，并发度 = 1，无任何优势场景。

**其他 Map 选型**：需要按 key 排序 → TreeMap；需要保持插入顺序 → LinkedHashMap。
