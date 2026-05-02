---
type: concept
status: active
name: "CAS"
layer: L3
aliases: ["Compare And Swap", "乐观锁", "ABA问题", "AtomicStampedReference", "自旋", "cmpxchg"]
related:
  - "[[机制-volatile]]"
  - "[[机制-synchronized]]"
  - "[[机制-AQS]]"
sources:
  - "../../raw/note/📚 Hollis Java/Java并发/✅什么是CAS？存在什么问题？ 30f3673e11388159a4a6fe8bead80ccd.md"
  - "../../raw/note/📚 Hollis Java/Java并发/✅CAS在操作系统层面是如何保证原子性的？ 30f3673e113881489549dc73299871e9.md"
  - "../../raw/note/📚 Hollis Java/Java并发/✅CAS一定有自旋吗？ 30f3673e11388121a0f8f78d65537fd8.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# CAS（Compare And Swap）

> CAS 是硬件级原子操作——包含内存位置 V、预期值 A、新值 B 三个操作数，当 V 的当前值等于 A 时才将 V 更新为 B，否则不操作；失败线程不阻塞而是重试（自旋）；这是 Java JUC 包所有 Atomic 类和 AQS 的底层基础。

## 第一性原理

synchronized 用互斥锁保证原子性，代价是线程阻塞/唤醒（OS 系统调用），竞争激烈时开销大。CAS 的存在理由：**不通过锁，而是通过"比较再替换"的乐观假设，在 CPU 硬件指令层面保证原子性，失败时线程继续自旋而非阻塞**——这是无锁（lock-free）算法的核心思想。

## 核心机制

### 硬件保证：cmpxchg 指令

```
CAS(V, A, B) 在 x86 上编译为 cmpxchg 指令
  → CPU 执行时自动锁总线（或缓存行锁）
  → 中间不允许其他 CPU 访问该内存地址
  → 比较 V == A？成立则写 B；不成立则不写
  → 解锁总线

多核场景：基于缓存一致性协议（MESI）传播变更
```

**关键点**：原子性由 CPU 硬件保证，不是软件锁。

### Java 中的 CAS：Unsafe + AtomicXxx

```java
// AtomicInteger 内部：
private volatile int value;           // volatile 保证可见性

// getAndIncrement 等价于 i++，但线程安全：
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
    // 内部是 CAS 自旋：
    // do { old = get(); } while (!compareAndSet(old, old+1));
}
```

**`value` 必须是 volatile**：CAS 读到旧值时必须是主内存最新值，volatile 保证这一点。

### 三大问题

#### 1. ABA 问题

```
线程1 读到 V=A
线程2 把 V: A→B→A（中间经历了变化）
线程1 CAS(V, A, B) 成功——但它"看到的 A"已经历了隐藏的变化
```

**危害**：被忽略的中间操作可能导致业务状态不一致（如账户余额经过借出+归还的过程被抹掉）。

**解决**：`AtomicStampedReference` — 同时维护引用 + 版本号（stamp）。CAS 时两者都必须匹配：

```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
// compareAndSet(期望引用, 新引用, 期望stamp, 新stamp)
ref.compareAndSet("A", "B", 0, 1);  // stamp: 0→1
```

版本号只增不减，彻底解决 ABA。

#### 2. 忙等待（Busy Waiting）

并发冲突高时，CAS 不断失败重试，线程一直占用 CPU 但无进展——进入忙等待状态，造成 CPU 浪费。

**解决**：
- `LongAdder` 替代 `AtomicLong`（分段 CAS，减少冲突）
- 高竞争场景改用 `synchronized` 或 `ReentrantLock`

#### 3. 只能保证一个变量的原子性

CAS 一次只能操作一个内存位置，多变量的原子更新需要：
- `AtomicReference`（把多个变量封装成对象）
- `synchronized` 或 `Lock`

### 乐观锁 vs 悲观锁

| 维度 | 乐观锁（CAS） | 悲观锁（synchronized）|
|------|------------|---------------------|
| 假设 | 冲突很少，先操作再检查 | 冲突必然发生，先加锁 |
| 冲突处理 | 失败重试（自旋）| 阻塞等待（OS调度）|
| 适用 | 低竞争、短操作 | 高竞争、长临界区 |
| 开销 | 无系统调用，但自旋耗 CPU | 有上下文切换，但不占 CPU |

## 关键权衡

1. **CAS 不等于自旋**：CAS 是"比较替换"的原子操作本身；自旋是 CAS 失败后的重试策略。`compareAndSet` 返回 false 后是否继续重试，由调用者决定（例如 AQS 中失败后会 park 挂起而非自旋）
2. **CAS 性能优于 synchronized 的前提**：竞争不激烈。高竞争下自旋成为 CPU 浪费，synchronized 反而更好（JVM 会把线程挂起不占 CPU）
3. **原子引用比原子基本类型复杂**：引用类型的 ABA 问题更隐蔽，默认优先用 `AtomicStampedReference`

## 与其他概念的关系

- 依赖 [[机制-volatile]]：`AtomicInteger.value` 是 volatile，CAS 读写结合 volatile 实现可见性
- 支撑 [[机制-AQS]]：AQS 的 `compareAndSetState` 就是 CAS，是 AQS 的核心操作
- 相对于 [[机制-synchronized]]：CAS 是乐观非阻塞，synchronized 是悲观阻塞；两者互补

## 应用边界

**适合用 CAS / AtomicXxx**：
- 计数器（`AtomicInteger`、`LongAdder`）
- 无锁数据结构（`ConcurrentLinkedQueue`）
- 简单状态标志的原子切换

**不适合用 CAS**：
- 需要同时原子更新多个字段 → `synchronized` 或 `Lock`
- 高并发竞争激烈场景 → `LongAdder` 或 `synchronized`
- 有 ABA 敏感业务逻辑 → `AtomicStampedReference`
