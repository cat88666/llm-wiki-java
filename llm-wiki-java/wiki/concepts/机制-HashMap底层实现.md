---
type: concept
status: active
name: "HashMap底层实现"
layer: L4
aliases: ["HashMap", "哈希表", "散列表", "hash冲突", "扩容"]
related:
  - "[[机制-红黑树]]"
  - "[[概念-线性数据结构]]"
  - "[[机制-ConcurrentHashMap并发设计]]"
sources:
  - "../../raw/note/Hollis/集合类/✅HashMap的数据结构是怎样的？.md"
  - "../../raw/note/Hollis/集合类/✅HashMap的hash方法是如何实现的？.md"
  - "../../raw/note/Hollis/集合类/✅HashMap是如何扩容的？.md"
  - "../../raw/note/Hollis/集合类/✅JDK1 8中HashMap有哪些改变？.md"
  - "../../raw/note/Hollis/集合类/✅为什么HashMap的Cap是2^n，如何保证？.md"
  - "../../raw/note/Hollis/集合类/✅为什么HashMap的默认负载因子设置成0 75.md"
  - "../../raw/note/Hollis/集合类/✅为什么在JDK8中HashMap要转成红黑树.md"
  - "../../raw/note/Hollis/集合类/✅HashMap用在并发场景中有什么问题？.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# HashMap底层实现

> 基于"数组+链表+红黑树"的散列表，通过 hash 函数将 key 映射到数组下标，平均 O(1) 增删查，代价是哈希冲突和扩容。

## 第一性原理

HashMap 要解决的根本问题：**在无限 key 空间中实现接近 O(1) 的查找**。核心手段是 hash 函数——将任意 key 压缩成有限整数（数组下标），冲突时用链表/红黑树兜底。整个设计的所有细节（2^n 容量、0.75 负载因子、红黑树阈值）都是围绕"尽量少冲突、冲突了尽量快"这一目标展开的。

## 核心机制

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

## 关键权衡

1. **线程不安全**：JDK 7 并发扩容（头插法）产生环形链表，CPU 100%；JDK 8 改尾插后无死循环，但仍有数据覆盖、size 不准确等问题，并发场景用 [[机制-ConcurrentHashMap并发设计]]
2. **内存开销**：每个 Entry/Node 除 key-value 外还有 hash 和 next 指针；TreeNode 额外存父/左/右指针
3. **null key**：允许一个 null key（存在 index 0），允许多个 null value；与 [[机制-ConcurrentHashMap并发设计]] 不同

## 与其他概念的关系

- 底层节点退化为 [[机制-红黑树]]：链表长度 ≥ 8 触发树化，O(log n) 保底
- 依赖 [[概念-线性数据结构]]：数组（主桶）+ 链表（冲突链）是基本结构
- 被 [[机制-ConcurrentHashMap并发设计]] 继承演化：CHM 在 HashMap 基础上加并发控制
- 在 L5 MySQL：B+ 树索引（[[机制-InnoDB索引模型]]）和 HashMap 都是 hash 思想的体现，但场景不同（磁盘 vs 内存）

## 应用边界

**适合 HashMap**：单线程或用 `synchronized` 包裹的简单 KV 缓存；key 均匀分布的场景。

**不适合**：多线程并发读写（用 ConcurrentHashMap）；需要按 key 排序（用 TreeMap）；需要保持插入顺序（用 LinkedHashMap）。

**容量预估**：若预知元素数量 n，初始化 `new HashMap<>(n / 0.75 + 1)` 避免扩容。
