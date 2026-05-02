---
type: concept
status: active
name: "ThreadLocal"
layer: L3
aliases: ["ThreadLocalMap", "线程局部变量", "弱引用Key", "内存泄漏", "线程隔离"]
related:
  - "[[机制-线程池]]"
  - "[[概念-引用类型]]"
  - "[[概念-JMM]]"
sources:
  - "../../raw/note/📚 Hollis Java/Java并发/✅ThreadLocal为什么会导致内存泄漏？如何解决的？ 30f3673e11388186a780d557d5f8cc3b.md"
  - "../../raw/note/📚 Hollis Java/Java并发/✅ThreadLocal的应用场景有哪些？ 30f3673e113881c68ddafd2dfe5c3f17.md"
  - "../../raw/note/📚 Hollis Java/Java并发/✅线程池中使用ThreadLocal会有哪些潜在风险？ 30f3673e113881dda797decc50fa08f7.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# ThreadLocal

> ThreadLocal 是线程级别的变量隔离机制——每个线程拥有独立副本，读写互不干扰，无需同步；底层由 Thread 持有 ThreadLocalMap，以 ThreadLocal 实例的弱引用为 key，线程独立数据为 value；内存泄漏风险来自 key 被回收后 value 依然存活在线程池线程中。

## 第一性原理

共享变量需要加锁，加锁引入阻塞和上下文切换开销。但有些场景中，不同线程根本不需要共享同一个变量——每个线程只需要自己的独立副本（如 userId、数据库连接、SimpleDateFormat）。ThreadLocal 的存在理由：**通过将变量绑定到线程而非共享内存，从根本上避免竞争——不是更好地加锁，而是不需要锁**。

## 核心机制

### 存储结构

```
Thread 对象
  └── threadLocals: ThreadLocalMap
        └── Entry[] table（数组，线性探测法解决哈希冲突）
              └── Entry extends WeakReference<ThreadLocal<?>>
                    ├── key: WeakReference(ThreadLocal实例)  ← 弱引用！
                    └── value: Object（线程独立数据）         ← 强引用
```

**关键**：`Entry` 的 key 是 `ThreadLocal` 实例的**弱引用**，value 是**强引用**。

### 读写过程

```java
ThreadLocal<String> tl = new ThreadLocal<>();

tl.set("userId-123");
// → Thread.currentThread().threadLocals（ThreadLocalMap）
// → 以 tl（弱引用）为 key，"userId-123" 为 value，存入 Entry 数组

tl.get();
// → 当前线程的 ThreadLocalMap → 用 tl 为 key 查找 → 返回 value
```

### 内存泄漏根因与解决

**两条引用链**：

```
栈上 ThreadLocal 引用 → ThreadLocal 对象 ← Entry.key（弱引用）
Thread 对象 → ThreadLocalMap → Entry → value（强引用）
```

**泄漏场景**：

| 情况 | 发生过程 | 后果 |
|------|---------|------|
| 普通线程 | 方法结束，栈上 tl 引用消失 → Entry.key 被弱引用，GC 时 key 被回收 → key=null，但 value 还在 | JDK 部分解决：`get/set/remove` 时会清理 key=null 的 Entry |
| **线程池线程** | 线程复用，Thread 对象长期存活 → ThreadLocalMap 不被回收 → value 一直存活 | **真正的内存泄漏**：value 随线程池线程永久存在 |

**解决方案（开发者责任）**：

```java
try {
    tl.set(userId);
    // ... 业务逻辑
} finally {
    tl.remove();  // ← 强制清理，不依赖 GC
}
```

`remove()` 会显式删除当前线程 ThreadLocalMap 中对应 Entry，彻底断开 value 的引用链。

### 父子线程传递

普通 `ThreadLocal` 无法跨线程传递。需要跨线程共享时：
- `InheritableThreadLocal`：子线程创建时复制父线程的 ThreadLocalMap（但线程池中线程复用，子线程与任务无关）
- 分布式链路追踪通常用 `TransmittableThreadLocal`（阿里 TTL 开源库）

## 关键权衡

1. **弱引用 key 只解决了一半**：防止 ThreadLocal 实例本身无法被 GC；但 value 是强引用，线程池场景下 value 的泄漏需开发者手动 `remove()`
2. **`get/set/remove` 内置清理的局限**：ThreadLocalMap 在调用这三个方法时会顺带清理 key=null 的 Entry，但如果某个 ThreadLocal 用完后再不调用这些方法，清理就不会发生
3. **线性探测法的遍历成本**：大量 ThreadLocal 时，查找需要扫描 Entry 数组，不如 HashMap 高效

## 与其他概念的关系

- 依赖 [[概念-引用类型]]：Entry.key 是弱引用，是 JDK 的半自动泄漏防护机制
- 与 [[机制-线程池]] 组合有风险：线程复用 + 强引用 value = 内存泄漏，必须 `remove()`
- 与 [[概念-JMM]] 的关系：ThreadLocal 通过隔离副本绕开了 JMM 的可见性问题——根本不共享，无需 happens-before

## 应用边界

**适合用 ThreadLocal**：
- 用户身份信息在请求链路中传递（Web 拦截器 set → 业务层 get → 拦截器 finally remove）
- 线程不安全工具类的线程隔离（`SimpleDateFormat`、`Calendar`）
- 数据库 Session / 事务上下文管理（MyBatis、Hibernate）
- 分布式链路 traceId 传递

**不适合 / 注意事项**：
- 线程池中使用时，**每个任务结束必须 `remove()`**，否则旧值污染下一个任务
- 大对象作为 value 时泄漏代价更高，更需要严格 remove
- JDK 25 引入 `ScopedValue` 作为 ThreadLocal 的改进替代（不可变、生命周期明确、虚拟线程友好）
