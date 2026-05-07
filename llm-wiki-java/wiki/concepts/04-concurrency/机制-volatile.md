---
type: concept
status: active
name: "volatile"
layer: L3
aliases: ["内存屏障", "可见性", "有序性", "指令重排"]
related:
  - "[[概念-JMM]]"
  - "[[机制-synchronized]]"
  - "[[机制-CAS]]"
sources:
  - "../../../raw/note/Hollis/Java并发/✅volatile是如何保证可见性和有序性的？.md"
  - "../../../raw/note/Hollis/Java并发/✅volatile能保证原子性吗？为什么？.md"
  - "../../../raw/note/Hollis/Java并发/✅有了synchronized为什么还需要volatile.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# volatile

> volatile 是 Java 的轻量级同步关键字——保证**可见性**（写后立即刷主内存，读前总从主内存取）和**有序性**（内存屏障禁止指令重排），但**不保证原子性**（复合操作如 i++ 仍需额外同步）。

## 第一性原理

synchronized 是重量级互斥锁，每次读写都需要完整的加解锁。但许多场景只需要"一写多读"的可见性保证，不需要互斥。volatile 的存在理由：**比 synchronized 更轻量——不需要加解锁，只需要在写时刷新主内存、在读时加内存屏障禁止重排**。

## 核心机制

### 可见性保证

```
线程 A 写 volatile 变量
  → JVM 发出 LOCK 前缀指令（x86）
  → 本线程 CPU 缓存中的值写回主内存
  → 通过缓存一致性协议（MESI），其他 CPU 缓存的该变量失效
线程 B 读 volatile 变量
  → 从主内存重新加载，得到最新值
```

### 有序性保证（内存屏障）

volatile 禁止指令重排的实现原理——JMM 规定在 volatile 读写两侧插入内存屏障：

| 操作 | 屏障 | 效果 |
|------|------|------|
| volatile **写**之前 | StoreStore | 禁止普通写被重排到 volatile 写之后 |
| volatile **写**之后 | StoreLoad | 禁止 volatile 写被重排到后续读之前 |
| volatile **读**之后 | LoadLoad | 禁止 volatile 读被重排到后续普通读之前 |
| volatile **读**之后 | LoadStore | 禁止 volatile 读被重排到后续普通写之前 |

### 不保证原子性

```java
// volatile 不能保证 i++ 的原子性
volatile int i = 0;
// i++ = read(i) + add + write(i)，三步不是原子的
// 多线程下结果仍然不正确
```

`i++` 是三步操作（load-add-store），时间片可在任意步骤之间切换，volatile 仅保证 load 读最新值、store 写回主内存，但不阻止被中断。

### 经典应用：双重检验锁（DCL）

```java
private volatile static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {                    // 第一次检查（无锁）
        synchronized (Singleton.class) {
            if (instance == null) {            // 第二次检查（有锁）
                instance = new Singleton();    // ← 必须加 volatile！
            }
        }
    }
    return instance;
}
```

**为什么必须加 volatile**：`new Singleton()` 编译成字节码有三步——①分配内存②初始化对象③将引用写入 instance。JIT 可能重排为①③②，此时另一个线程拿到非 null 的 instance 但对象未初始化，使用时 NPE。volatile 的内存屏障禁止②③重排。

### volatile vs synchronized 的选择

| 需求 | 选 volatile | 选 synchronized |
|------|------------|----------------|
| 只需可见性（一写多读）| ✓ | 可以，但重了 |
| 需要原子性（i++、复合操作）| ✗ | ✓（或 AtomicInteger）|
| 需要互斥 | ✗ | ✓ |
| 状态标志（boolean flag）| ✓ 最适合 | 可以，但重了 |

## 关键权衡

1. **volatile 写比普通写慢**：需要 LOCK 前缀指令，但比 synchronized 的 OS 级阻塞轻量得多
2. **volatile 读与普通读性能接近**：只是多了内存屏障，不需要内核态切换
3. **volatile 不能替代锁**：凡是有"读-改-写"的场景都不能用 volatile 代替 synchronized

## 与其他概念的关系

- 是 [[概念-JMM]] 中"volatile 变量规则"（happens-before）的具体实现
- 与 [[机制-synchronized]] 互补：volatile 保证可见性+有序性（轻量）；synchronized 额外保证原子性（重量）
- 支撑 [[机制-CAS]]：`AtomicInteger` 内部的 `value` 字段是 volatile，保证 CAS 读到最新值

## 应用边界

**适合用 volatile**：
- 状态标志（`volatile boolean shutdown`）
- 双重检验锁的单例模式
- 发布对象（让其他线程看到已完全初始化的对象）

**不适合用 volatile**：
- 计数器（`i++`）→ 用 `AtomicInteger`
- 需要多个变量的原子更新 → 用 `synchronized` 或 `Lock`
- 高竞争下的状态转换 → 用 `AtomicReference` + CAS
