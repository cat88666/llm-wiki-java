---
type: concept
status: active
name: "CAS"
layer: L3
aliases: ["Compare And Swap", "乐观锁", "ABA问题", "AtomicStampedReference", "自旋", "cmpxchg", "无锁算法", "LongAdder", "Cell数组", "AtomicInteger", "AtomicReference"]
related:
  - "[[机制-volatile]]"
  - "[[机制-synchronized]]"
  - "[[机制-AQS]]"
  - "[[概念-JMM]]"
sources:
  - "../../../raw/note/Hollis/Java并发/✅什么是CAS？存在什么问题？.md"
  - "../../../raw/note/Hollis/Java并发/✅CAS在操作系统层面是如何保证原子性的？.md"
  - "../../../raw/note/Hollis/Java并发/✅CAS一定有自旋吗？.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# CAS（Compare And Swap）

> CAS 是硬件级原子操作——包含内存位置 V、预期值 A、新值 B 三个操作数，当 V 的当前值等于 A 时才将 V 更新为 B，否则不操作并返回失败；Java JUC 包所有 Atomic 类和 AQS 的 state 修改均构建于其上，是无锁（lock-free）编程的底层基础。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 无锁算法的根本逻辑、与悲观锁的对比 |
| [二、核心机制](#二核心机制) | cmpxchg 指令、Java Unsafe/AtomicXxx、三大问题 |
| [三、LongAdder 高竞争优化](#三longadder-高竞争优化) | Cell 数组分段、base+cells 机制 |
| [四、综合对比](#四综合对比) | 乐观锁 vs 悲观锁、AtomicLong vs LongAdder |
| [五、生产风险](#五生产风险) | ABA 敏感场景、忙等待、多变量原子性 |
| [六、与其他概念的关系](#六与其他概念的关系) | volatile、AQS、synchronized |
| [七、应用边界](#七应用边界) | 适合/不适合场景 |

## 一、第一性原理

synchronized 用互斥锁保证原子性，代价是线程阻塞/唤醒（OS 系统调用），竞争激烈时上下文切换开销可能远超临界区本身的执行时间。

CAS 的存在理由：**不通过锁，而是通过"比较再替换"的乐观假设，在 CPU 硬件指令层面保证原子性——失败时线程自旋重试而非阻塞，避免用户态→内核态切换**。这是无锁（lock-free）算法的核心思想。

关键推论：CAS 性能优于 synchronized 的前提是**冲突不频繁**；高竞争下自旋成为 CPU 浪费，synchronized 反而更好（线程挂起不占 CPU）。

## 二、核心机制

### 硬件保证：cmpxchg 指令

```
CAS(V, A, B)
  → x86 编译为 cmpxchg 指令
  → 单核：cmpxchg 本身是原子的
  → 多核：LOCK 前缀锁缓存行（Cache Line Lock）或总线锁（Bus Lock）
  → 中间不允许其他 CPU 访问该内存地址
  → 比较 V == A？成立则写 B，返回成功；不成立则不写，返回失败
```

**原子性由 CPU 硬件保证**，不是软件模拟。`LOCK CMPXCHG` 指令执行期间，其他 CPU 对该缓存行的读写被暂停。

现代 CPU 更多采用**缓存行锁**（Cache Line Lock）而非总线锁：只锁定目标内存所在的缓存行，粒度更小，吞吐量更高。

### Java 中的 CAS：Unsafe + AtomicXxx

```java
// AtomicInteger 内部简化：
public class AtomicInteger {
    private volatile int value;           // volatile 保证可见性

    // getAndIncrement 线程安全的 i++
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
        // 底层：
        // int v;
        // do {
        //     v = getIntVolatile(obj, offset);  // 读最新值
        // } while (!compareAndSwapInt(obj, offset, v, v + 1));  // CAS
        // return v;
    }
}
```

**`value` 必须是 volatile**：CAS 读到旧值必须是主内存最新值，volatile 的 load 语义保证这一点（结合 [[概念-JMM]] 可见性规则）。

### 三大问题

#### 1. ABA 问题

```
线程1 读到 V = A
线程2 把 V: A → B → A（中间经历了变化，当前又是 A）
线程1 CAS(V, A, B) 成功——但"看到的 A"已历经隐藏的变化
```

**危害**：链表结构中 ABA 可能导致节点被错误操作；账户余额经过借出+归还的过程被抹掉。

**解决**：`AtomicStampedReference<T>`——同时维护值 + 版本号（stamp），CAS 时两者都必须匹配：

```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);

// compareAndSet(期望值, 新值, 期望stamp, 新stamp)
boolean success = ref.compareAndSet("A", "B", 0, 1);  // stamp: 0→1

// 第三个线程改回 A
ref.compareAndSet("B", "A", 1, 2);  // stamp: 1→2，version 变了！

// 原线程的 CAS 失败（期望stamp=0，实际stamp=2）
ref.compareAndSet("A", "B", 0, 1);  // 返回 false
```

stamp 只增不减，彻底解决 ABA 问题。

#### 2. 忙等待（Busy Waiting）

并发冲突高时，CAS 不断失败重试，线程一直占用 CPU 但无进展——**CPU 使用率高，吞吐量低**。

**解决方案**：
- 改用 `LongAdder`（分段 CAS，详见第三节）
- 高竞争场景改用 `synchronized` 或 `ReentrantLock`
- AQS 内部 CAS 失败后 `LockSupport.park()` 挂起，不是纯自旋

#### 3. 只能保证单变量的原子性

CAS 一次只能操作一个内存地址，多变量的原子更新需要：
- `AtomicReference<T>`（把多个变量封装为对象，CAS 整个引用）
- `synchronized` 或 `Lock`

## 三、LongAdder 高竞争优化

`AtomicLong` 在高并发下所有线程竞争同一个 `value`，CAS 失败率高。`LongAdder` 引入**分段竞争**：

```
LongAdder 内部结构：
├── base: long（低竞争时直接 CAS base）
└── Cell[] cells（哈希到不同 Cell，每个 Cell 独立 CAS）

sum() = base + Σ cells[i].value
```

**工作流程**：
1. 无竞争时：CAS 更新 `base`（与 AtomicLong 行为相同）
2. 有竞争时：将当前线程哈希到某个 `Cell`，CAS 更新该 `Cell`
3. `Cell` 也冲突时：扩容 `cells` 数组（最大 = CPU 核心数）

**代价**：`sum()` 不是精确快照（读取 base + 所有 Cell 之间有时间差），适合统计场景（如计数器、求和），不适合需要精确值的场景。

## 四、综合对比

### 乐观锁 vs 悲观锁

| 维度 | 乐观锁（CAS）| 悲观锁（synchronized）|
|------|-----------|---------------------|
| 核心假设 | 冲突很少，先操作再检查 | 冲突必然发生，先加锁再操作 |
| 冲突处理 | 失败重试（自旋/中止）| 阻塞等待（OS 调度）|
| CPU 占用 | 自旋时占 CPU，无进展 | 挂起不占 CPU，但有上下文切换 |
| 延迟特性 | 低竞争下延迟极低 | 高竞争下稳定，低竞争下多一次锁开销 |
| 适用场景 | 低竞争、临界区极短 | 高竞争、临界区较长 |

### AtomicLong vs LongAdder

| 维度 | AtomicLong | LongAdder |
|------|-----------|----------|
| 数据结构 | 单个 volatile long | base + Cell 数组 |
| 低竞争 | 性能相当 | 性能相当 |
| 高竞争 | 大量 CAS 失败，性能下降 | 分段竞争，吞吐量高 |
| `get()` 精确性 | 精确 | 近似（sum() 有时间差）|
| 内存占用 | 小 | 较大（Cell 数组）|
| 适用 | 需要精确原子值 | 高并发统计/累加 |

## 五、生产风险

| 风险 | 场景 | 解决 |
|------|------|------|
| ABA 问题被忽略 | 链表/队列结构用 AtomicReference | 改用 AtomicStampedReference |
| 忙等待 CPU 飙高 | 高并发计数器用 AtomicLong | 改用 LongAdder |
| CAS 认为可以保证多变量原子 | 同时更新两个 Atomic 字段 | 封装对象 + AtomicReference |
| sum() 精度误解 | LongAdder.sum() 用于余额计算 | 余额等精确场景用 AtomicLong + synchronized |
| Unsafe 直接调用 | 绕过 JVM 安全机制 | 不要直接使用 Unsafe，用 AtomicXxx 封装 |

## 六、与其他概念的关系

- 依赖 [[机制-volatile]]：`AtomicInteger.value` 是 volatile，CAS 读写结合 volatile 实现可见性
- 依赖 [[概念-JMM]]：CAS 操作隐含 happens-before 关系，成功的 CAS 对后续读可见
- 支撑 [[机制-AQS]]：AQS 的 `compareAndSetState(expect, update)` 就是 CAS，是 AQS 修改同步状态的核心手段
- 相对于 [[机制-synchronized]]：CAS 是乐观非阻塞，synchronized 是悲观阻塞；两者互补，AQS 内部混用了两者（CAS + LockSupport.park）

## 七、应用边界

**适合 CAS / AtomicXxx**：
- 单变量原子计数器（`AtomicInteger`、`AtomicLong`）
- 高并发累加统计（`LongAdder`）
- 无锁状态标志切换（`AtomicBoolean`）
- 无锁引用替换（`AtomicReference`）
- 无锁数据结构（`ConcurrentLinkedQueue` 底层）

**不适合 CAS**：
- 需要同时原子更新多个字段 → `synchronized` 或 `Lock`
- 高并发竞争激烈 + 操作耗时较长 → `LongAdder` 或 `synchronized`
- 有 ABA 敏感业务逻辑（链表/队列变更追踪）→ `AtomicStampedReference`
- 临界区内有 IO 或复杂逻辑 → `synchronized`（挂起不占 CPU）
