---
type: concept
status: active
name: "HashMap"
layer: L4
aliases: ["哈希表", "散列表", "HashMap扩容", "HashMap树化"]
related:
  - "[[概念-Java集合框架]]"
  - "[[机制-红黑树]]"
  - "[[机制-ConcurrentHashMap]]"
sources:
  - "../../raw/note/Hollis/集合类/✅HashMap的数据结构是怎样的？.md"
  - "../../raw/note/Hollis/集合类/✅HashMap在get和put时经过哪些步骤？.md"
  - "../../raw/note/Hollis/集合类/✅HashMap是如何扩容的？.md"
  - "../../raw/note/Hollis/集合类/✅HashMap的hash方法是如何实现的？.md"
  - "../../raw/note/Hollis/集合类/✅为什么HashMap的Cap是2^n，如何保证？.md"
  - "../../raw/note/Hollis/集合类/✅为什么HashMap的默认负载因子设置成0 75.md"
  - "../../raw/note/Hollis/集合类/✅HashMap用在并发场景中有什么问题？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# HashMap

> HashMap 是基于哈希定位的 key-value 容器，JDK 8 后使用数组 + 链表 + 红黑树来兼顾平均 O(1) 访问和冲突退化控制。

## 第一性原理

Map 的根本目标是按 key 快速定位 value。数组下标访问最快，但 key 通常不是整数下标；HashMap 用 hash 函数把 key 映射到桶下标，再用链表或红黑树解决多个 key 落到同一桶的冲突。

## 核心机制

1. **定位桶**：先计算 `hashCode`，再做高低 16 位扰动，最后用 `(table.length - 1) & hash` 定位桶。
2. **容量为 2 的幂**：当容量是 `2^n` 时，`hash % capacity` 可等价为位运算，扩容后节点只需按 `hash & oldCap` 分成原位置或原位置 + oldCap。
3. **put 流程**：空桶直接插入；非空桶按 key 的 hash 和 `equals` 查找；存在则覆盖，不存在则插入链表或红黑树。
4. **扩容机制**：`size > threshold` 时扩为 2 倍，`threshold = capacity * loadFactor`，默认负载因子 0.75 是空间和冲突概率的折中。
5. **树化与退化**：桶链表过长时树化为红黑树；扩容或删除后节点过少会退化回链表。

## 关键权衡

- **平均 O(1) 依赖 hash 分布**：hash 冲突严重时，链表会退化到 O(n)，树化把最坏情况压到 O(log n)。
- **容量与空间浪费**：容量越大冲突越少，但空桶越多；负载因子越高空间越省，冲突越多。
- **允许 null 带来语义二义性**：`get(key) == null` 可能表示不存在，也可能表示 value 为 null。
- **非线程安全**：并发 put、resize、遍历修改都可能产生不一致或异常，应使用并发容器或外部同步。

## 与其他概念的关系

- 依赖 [[机制-红黑树]]：JDK 8+ 用红黑树控制单桶冲突退化。
- 依赖 [[概念-线性数据结构]]：桶数组提供 O(1) 下标访问，链表处理冲突。
- 支撑 [[概念-Java集合框架]]：`HashSet` 通过 `HashMap` 的 key 去重。
- 区别于 [[机制-ConcurrentHashMap]]：HashMap 不做并发控制，ConcurrentHashMap 针对初始化、put、扩容做 CAS 和锁控制。

## 应用边界

**适合**：
- 单线程或外部已同步场景下的 key-value 快速查找
- key 的 `hashCode` / `equals` 稳定且分布较好的对象

**不适合**：
- 多线程共享写入
- key 可变且参与 `hashCode` / `equals`
- 需要排序或范围查询，应选 `TreeMap`

