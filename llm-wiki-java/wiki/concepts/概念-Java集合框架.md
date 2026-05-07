---
type: concept
status: active
name: "Java集合框架"
layer: L4
aliases: ["Collection", "Collections", "List", "Set", "Queue", "Map"]
related:
  - "[[概念-线性数据结构]]"
  - "[[机制-HashMap]]"
  - "[[概念-集合迭代一致性]]"
sources:
  - "../../raw/note/Hollis/集合类/✅Java中的集合类有哪些？如何分类的？.md"
  - "../../raw/note/Hollis/集合类/✅ArrayList、LinkedList与Vector的区别？.md"
  - "../../raw/note/Hollis/集合类/✅Set是如何保证元素不重复的.md"
  - "../../raw/note/Hollis/集合类/✅你能说出几种集合的排序方式？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# Java集合框架

> Java 集合框架是一组围绕元素容器、键值映射、遍历协议和工具方法组织的通用数据结构抽象。

## 第一性原理

业务代码不断遇到同一类问题：如何保存一批对象、按某种规则查找、遍历、去重、排序或映射。集合框架把这些问题拆成接口语义（`List`、`Set`、`Queue`、`Map`）和底层结构（数组、链表、哈希表、红黑树），让调用方按访问模式选容器。

## 核心机制

1. **Collection 体系**：`List`、`Set`、`Queue` 都继承 `Collection`，并通过 `Iterable` 暴露统一遍历能力。
2. **Map 体系**：`Map` 存储 key-value，不属于 `Collection`，核心能力是按 key 定位 value。
3. **List 取舍**：`ArrayList` 基于数组，随机访问 O(1)，中间插删要搬移元素；`LinkedList` 基于双向链表，头尾操作更自然，但随机访问 O(n)。
4. **Set 去重**：`HashSet` 基于 `HashMap` 的 key 去重，依赖 `hashCode` + `equals`；`TreeSet` 基于 `TreeMap`，依赖 `compareTo` 并保持排序。
5. **Collections 工具类**：`Collections` 是静态工具类，不是集合接口，可提供排序、同步包装等操作。

## 关键权衡

- **接口语义优先于实现类**：先判断是否需要有序、去重、队列语义或 key-value，再选具体实现。
- **线程安全不是默认能力**：`ArrayList`、`HashMap` 等普通容器默认不保证并发安全。
- **同步容器不等于高并发容器**：`Vector`、`Hashtable` 单方法同步，但复合操作仍需额外同步，且读写互斥。
- **排序语义要明确**：对象自身排序用 `Comparable`，外部临时排序规则用 `Comparator`。

## 与其他概念的关系

- 依赖 [[概念-线性数据结构]]：`ArrayList` 和 `LinkedList` 分别对应数组与链表。
- 依赖 [[机制-红黑树]]：`TreeMap`、`TreeSet` 和 JDK 8+ `HashMap` 桶树化都使用红黑树。
- 支撑 [[机制-HashMap]]：`HashSet` 本质上复用 `HashMap` 的 key 唯一性。
- 关联 [[概念-集合迭代一致性]]：集合遍历时是否允许结构变化，决定 fail-fast 或 fail-safe 行为。

## 应用边界

**适合**：
- 读多、按下标访问：`ArrayList`
- 头尾队列语义：`ArrayDeque` 或 `LinkedList`
- key-value 定位：`HashMap`
- 去重：`HashSet` / `TreeSet`

**不适合**：
- 高并发写入时直接使用普通集合
- 用 `Vector` 作为默认线程安全 List
- 在排序规则不稳定时放入 `TreeSet` / `TreeMap`

