---
type: concept
status: active
name: "ThreadLocal"
layer: L3
aliases: ["ThreadLocalMap", "线程局部变量", "弱引用Key", "内存泄漏", "线程隔离", "InheritableThreadLocal", "TransmittableThreadLocal", "TTL", "ScopedValue"]
related:
  - "[[机制-线程池]]"
  - "[[概念-JMM]]"
  - "[[机制-AQS]]"
sources:
  - "../../../raw/note/Hollis/Java并发/✅ThreadLocal为什么会导致内存泄漏？如何解决的？.md"
  - "../../../raw/note/Hollis/Java并发/✅ThreadLocal的应用场景有哪些？.md"
  - "../../../raw/note/Hollis/Java并发/✅线程池中使用ThreadLocal会有哪些潜在风险？.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# ThreadLocal

> ThreadLocal 是线程级变量隔离机制——每个线程拥有独立的变量副本，读写互不干扰，无需同步；底层由 Thread 对象持有 ThreadLocalMap，以 ThreadLocal 实例的弱引用为 key、线程独立数据为强引用 value；内存泄漏风险来自 key 被 GC 后 value 依然随线程池线程长期存活。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 隔离代替同步、不共享就无竞争 |
| [二、核心机制](#二核心机制) | ThreadLocalMap 结构、弱引用key/强引用value、读写流程 |
| [三、内存泄漏根因与解法](#三内存泄漏根因与解法) | 两条引用链、普通线程 vs 线程池、remove() |
| [四、父子线程传递](#四父子线程传递) | InheritableThreadLocal、TTL 原理 |
| [五、典型应用场景](#五典型应用场景) | 请求上下文、事务管理、链路追踪 |
| [六、综合对比](#六综合对比) | ThreadLocal vs synchronized vs ScopedValue |
| [七、生产风险](#七生产风险) | 线程池污染、大对象泄漏、TTL 丢失 |
| [八、与其他概念的关系](#八与其他概念的关系) | 线程池、JMM、AQS |
| [九、应用边界](#九应用边界) | 适合/不适合场景、JDK 21+ 替代方案 |

## 一、第一性原理

共享变量需要加锁（synchronized/Lock），加锁引入阻塞和上下文切换开销。但有些场景中，不同线程根本不需要共享同一变量——每个线程只需要自己的独立副本（如当前用户 ID、数据库连接、`SimpleDateFormat`）。

ThreadLocal 的存在理由：**通过将变量绑定到线程而非共享内存，从根本上避免竞争——不是更好地加锁，而是不需要锁**。代价是每个线程持有独立副本，内存占用是线程数的倍数。

## 二、核心机制

### 存储结构

```
Thread 对象
  └── threadLocals: ThreadLocalMap（Thread 内部字段）
        └── Entry[] table（数组，线性探测法解决哈希冲突）
              └── Entry extends WeakReference<ThreadLocal<?>>
                    ├── key: WeakReference(ThreadLocal实例)  ← 弱引用！
                    └── value: Object（线程独立数据）          ← 强引用
```

**关键设计**：
- **ThreadLocalMap 在 Thread 里**，不在 ThreadLocal 里——ThreadLocal 只是访问当前线程 Map 的入口键
- **key 是弱引用**：ThreadLocal 实例没有外部强引用时，GC 会回收 ThreadLocal 对象，key 变为 null
- **value 是强引用**：value 不会被 GC 自动回收，必须手动清理

### 读写过程

```java
ThreadLocal<String> userId = new ThreadLocal<>();

// set：
userId.set("user-123");
// → Thread.currentThread().threadLocals（ThreadLocalMap）
// → 以 userId 的弱引用为 key，"user-123" 为 value，存入 Entry 数组

// get：
String id = userId.get();
// → 当前线程 ThreadLocalMap → 用 userId 计算哈希找 Entry → 返回 value

// remove：
userId.remove();
// → 找到 Entry，清空 key（置 null）和 value（置 null），彻底断开引用链
```

### 哈希冲突：线性探测法

ThreadLocalMap 不用链表（避免内存开销），用**线性探测**解决冲突：index 冲突时依次向后探测空槽。

**代价**：ThreadLocal 数量多时，`get/set` 需要扫描多个 Entry，性能不如 HashMap。实践中每个线程的 ThreadLocal 数量通常很少（< 10），问题不大。

## 三、内存泄漏根因与解法

### 两条引用链

```
强引用链（不会被 GC）：
Thread 对象 → ThreadLocalMap → Entry → value

弱引用链（可能被 GC 回收）：
栈帧上 threadLocal 变量 → ThreadLocal 实例 ← Entry.key（WeakReference）
```

### 普通线程 vs 线程池的区别

| 场景 | 过程 | 后果 |
|------|------|------|
| **普通线程** | 线程结束 → Thread 对象被 GC → ThreadLocalMap 随之回收 → value 随之回收 | 无泄漏（Thread 生命周期兜底）|
| **线程池线程** | 线程被复用，Thread 对象长期存活 → ThreadLocalMap 不被回收 → key 被 GC（弱引用），变 null → **value 强引用永远存活** | **内存泄漏**（value 随线程池线程永久存在）|

### JDK 的半自动清理（不够用）

`ThreadLocalMap` 在调用 `get`、`set`、`remove` 时，会顺带清理 key=null 的过期 Entry（`expungeStaleEntry`）。

**局限**：如果某个 ThreadLocal 用完后，该线程不再调用这三个方法（或调用的是其他 ThreadLocal），过期 Entry 就不会被清理。线程池场景下这很常见。

### 正确解法：手动 remove()

```java
ThreadLocal<UserContext> ctx = new ThreadLocal<>();

// 推荐写法：
try {
    ctx.set(new UserContext(userId));
    // ... 业务逻辑
} finally {
    ctx.remove();  // ← 强制清理，不依赖 GC 或 JDK 自动清理
}
```

`remove()` 会显式删除当前线程 ThreadLocalMap 中对应的 Entry，彻底断开 value 的引用链。**在任何使用线程池的场景，这都是必须的。**

## 四、父子线程传递

普通 `ThreadLocal` 不跨线程传递，子线程看不到父线程的值。

### InheritableThreadLocal

子线程**创建时**复制父线程的 ThreadLocalMap 到子线程：

```java
InheritableThreadLocal<String> traceId = new InheritableThreadLocal<>();
traceId.set("trace-abc");

new Thread(() -> {
    System.out.println(traceId.get()); // "trace-abc"（子线程看到父线程的值）
}).start();
```

**局限**：线程池中的线程在池创建时就已存在，不是每次提交任务时新建，所以任务提交时无法触发复制，子线程看到的是**线程创建时的旧值**，不是任务提交时的值。

### TransmittableThreadLocal（阿里 TTL）

`com.alibaba:transmittable-thread-local` 解决线程池场景的父子线程传递：

```
任务提交时：捕获（capture）当前线程的 TTL 值快照
任务执行前：重放（replay）快照到工作线程
任务执行后：恢复（restore）工作线程原来的 TTL 值
```

原理：包装 `Runnable`/`Callable`，在任务执行前后做快照捕获和恢复，对工作线程无感知。

## 五、典型应用场景

| 场景 | 示例 | 存入时机 | 清理时机 |
|------|------|---------|---------|
| 用户身份信息传递 | Web 拦截器 set → Controller/Service get | 请求进入拦截器 | 拦截器 finally |
| 事务/DB 连接管理 | Spring `@Transactional` 绑定连接 | 开始事务 | 事务提交/回滚后 |
| 链路追踪 traceId | 分布式链路的请求标识 | 请求进入 | 请求结束 |
| 线程不安全工具隔离 | `SimpleDateFormat`（非线程安全）| 首次 get 时初始化 | 不需要（随线程销毁）|
| 性能统计上下文 | 每个线程独立计时 | 方法开始 | 方法结束 |

## 六、综合对比

| 维度 | ThreadLocal | synchronized | ScopedValue（JDK 21+）|
|------|------------|-------------|----------------------|
| 隔离方式 | 每线程独立副本 | 互斥共享 | 每线程只读绑定值 |
| 是否共享 | 不共享 | 共享（受锁保护）| 不共享（只读）|
| 是否可修改 | 可修改（set/remove）| 可修改（临界区内）| 不可修改（只读）|
| 生命周期 | 手动管理（remove）| 自动（锁范围）| 自动（绑定范围内有效）|
| 线程池安全 | 需手动 remove | 天然安全 | 天然安全（随作用域结束）|
| 虚拟线程兼容 | 兼容但浪费内存 | 兼容 | 专为虚拟线程设计 |

## 七、生产风险

| 风险 | 场景 | 解决方案 |
|------|------|---------|
| **线程污染** | 线程池任务 A 的 ThreadLocal 值被任务 B 读到 | 任务结束必须 `remove()` |
| **内存泄漏** | 大对象（如请求体、大 List）作为 value | 更需严格 `remove()`，避免大对象 value |
| **InheritableThreadLocal 失效** | 线程池中父子线程传递失败 | 改用 TTL（TransmittableThreadLocal）|
| **TTL 快照不及时** | TTL 在任务提交后、执行前父线程修改了值 | TTL 捕获的是提交时的快照，这是预期行为 |
| **弱引用误解** | 误以为弱引用 key 能自动清理 value | value 是强引用，必须手动 remove |

## 八、与其他概念的关系

- 与 [[机制-线程池]] 组合有内存泄漏风险：线程复用 + 强引用 value = 值残留，必须每次任务结束 `remove()`
- 与 [[概念-JMM]] 的关系：ThreadLocal 通过隔离副本绕开了 JMM 可见性问题——根本不共享，不需要 happens-before 保证
- 与 [[机制-AQS]]（ReentrantLock）对比：都解决多线程并发问题，但方式相反——AQS 控制共享访问，ThreadLocal 消除共享

## 九、应用边界

**适合 ThreadLocal**：
- 每个请求/线程需要独立上下文（userId、traceId、事务连接）
- 线程不安全工具类的线程隔离（`SimpleDateFormat`、`Calendar`）
- 数据库 Session / 事务上下文（MyBatis、Spring）

**不适合 / 注意事项**：
- 线程池场景：**每个任务结束必须 `remove()`**，否则旧值污染下一个任务
- 需要父子线程传递：用 `InheritableThreadLocal`（普通线程）或 `TransmittableThreadLocal`（线程池）
- JDK 21+ 虚拟线程场景：考虑 `ScopedValue`——不可变、生命周期明确、性能更好，是 ThreadLocal 的长期替代方案
