---
type: concept
status: active
name: "volatile"
layer: L3
aliases: ["内存屏障", "可见性", "有序性", "指令重排", "LOCK前缀", "StoreStore屏障", "StoreLoad屏障", "DCL", "双重检验锁", "MESI失效"]
related:
  - "[[概念-JMM]]"
  - "[[机制-synchronized]]"
  - "[[机制-CAS]]"
---

# volatile

> volatile 是 Java 的轻量级同步关键字——保证**可见性**（写后立即刷主内存，读前从主内存取最新值）和**有序性**（插入内存屏障禁止指令重排），但**不保证原子性**（复合操作如 i++ 仍需 AtomicInteger 或 synchronized）。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 为何需要 volatile、与 synchronized 的本质区别 |
| [二、核心机制](#二核心机制) | 可见性（LOCK前缀+MESI）、有序性（4种内存屏障）、不保证原子性 |
| [三、经典应用](#三经典应用) | DCL 双重检验锁、状态标志、发布对象 |
| [四、综合对比](#四综合对比) | volatile vs synchronized vs AtomicXxx |
| [五、生产风险](#五生产风险) | i++ 误用、多变量原子性、过度依赖 volatile |
| [六、与其他概念的关系](#六与其他概念的关系) | JMM、synchronized、CAS |
| [七、应用边界](#七应用边界) | 适合/不适合场景 |

## 一、第一性原理

synchronized 是重量级互斥锁，每次读写都需完整的加解锁（可能引发 OS 级阻塞）。但许多场景只需"一写多读"的可见性保证，不需要互斥。

volatile 的存在理由：**比 synchronized 更轻量——不加锁、不阻塞，只需在写时通过硬件指令刷新主内存、在读写两侧插入内存屏障禁止重排**。代价是不能保证原子性，只能替代 synchronized 中对可见性和有序性的那部分语义。

## 二、核心机制

### 可见性实现：LOCK 前缀指令 + MESI 缓存失效

```
线程 A 写 volatile 变量
  → JIT 编译后生成 LOCK 前缀指令（x86: LOCK; ADDL 或 MFENCE）
  → 将 CPU Store Buffer 中的值强制写回缓存行
  → 将修改后的缓存行通过 MESI 协议广播 Invalidate 消息
  → 其他 CPU 的该变量缓存行被标记为 Invalid

线程 B 读 volatile 变量
  → 发现缓存行 Invalid → 从主内存重新加载最新值
```

**为什么不只靠 MESI**：现代 CPU 有 Store Buffer（写缓冲）和 Invalidation Queue（失效队列），LOCK 前缀的本质是强制刷 Store Buffer 并等待 invalidate 确认，确保其他线程看到最新值。

### 有序性实现：4 种内存屏障

JMM 规定在 volatile 读写两侧插入内存屏障：

| 位置 | 屏障类型 | 禁止的重排 |
|------|---------|-----------|
| volatile **写**之前 | `StoreStore` | 普通写重排到 volatile 写之后 |
| volatile **写**之后 | `StoreLoad` | volatile 写重排到后续读之前（最重，刷 Store Buffer）|
| volatile **读**之后 | `LoadLoad` | volatile 读重排到后续普通读之前 |
| volatile **读**之后 | `LoadStore` | volatile 读重排到后续普通写之前 |

**StoreLoad 是所有屏障中开销最大的**，也是 volatile 写比普通写慢的根本原因（约等于一次 cache flush）。

### 不保证原子性（高频误区）

```java
volatile int i = 0;

// 线程A: i++
// 等价于：
// 1. load i（读主内存，拿到 0）
// 2. add i（计算 0+1=1）
// 3. store i（写回主内存）
// 时间片在步骤1和步骤3之间切换 → 其他线程读写 → 结果错误
```

volatile 保证每次 load 读最新值、每次 store 立即可见，但**不阻止 load 和 store 之间被时间片切换打断**。

`i++` 的三步（load-add-store）不是原子的，必须用 `AtomicInteger.incrementAndGet()` 或 `synchronized`。

## 三、经典应用

### 双重检验锁（DCL）——最重要的 volatile 用例

```java
public class Singleton {
    private volatile static Singleton instance;  // ← volatile 必须加！

    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) {            // 第二次检查（有锁）
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**为什么必须加 volatile**：`new Singleton()` 对应三步字节码：
1. 分配内存（堆上 alloc）
2. 初始化对象（调用构造函数）
3. 将引用写入 `instance`

JIT 可能将步骤 2、3 重排为 3、2。步骤 3 执行后，另一线程在第一次检查看到 `instance != null`，拿到**未初始化**的对象，使用时触发 NPE 或状态异常。

volatile 的 **StoreStore 屏障**（写前）禁止 2、3 重排；**StoreLoad 屏障**（写后）保证步骤 3 对所有线程立即可见。

### 状态标志

```java
volatile boolean running = true;

// 工作线程
while (running) {
    doWork();
}

// 控制线程
running = false;  // 立即对工作线程可见，不加 synchronized 也安全
```

这是 volatile 最经典的正确用法：单次写，多线程读，无复合操作。

### 对象安全发布

```java
volatile Config config;  // volatile 保证其他线程看到完整初始化的 Config 对象

// 线程A
config = new Config(/* 复杂初始化 */);

// 线程B
Config c = config;
c.getValue();  // 不加 volatile：可能看到部分初始化的 Config
```

## 四、综合对比

| 维度 | `volatile` | `synchronized` | `AtomicInteger` |
|------|-----------|--------------|----------------|
| 可见性 | ✓ | ✓ | ✓ |
| 有序性 | ✓ | ✓（串行执行）| 部分（CAS 不禁止重排）|
| 原子性 | ✗ | ✓ | ✓（单变量）|
| 阻塞 | 不阻塞 | 阻塞（重量锁）| 不阻塞（自旋）|
| 适用场景 | 状态标志、发布对象、DCL | 复合操作、复杂临界区 | 计数器、单变量原子操作 |
| 性能 | 最轻（写稍慢）| 最重（OS 切换）| 中（CAS 自旋）|

## 五、生产风险

| 风险 | 场景 | 正确做法 |
|------|------|---------|
| 误认为 volatile 保证原子性 | `volatile int count; count++` | 改用 `AtomicInteger` |
| volatile 多变量伪原子性 | 同时更新 `volatile x` 和 `volatile y` 期望两者同步 | 封装为对象 + `AtomicReference` |
| DCL 漏加 volatile | 单例模式双重检验 | 必须加 volatile |
| long/double 的非原子读写 | 64 位变量在 32 位 JVM 上可能被分为两次 32 位操作 | 加 volatile 或 synchronized |
| 过度 volatile 化 | 每个字段都加 volatile 追求"安全感" | volatile 写开销不低，应按需使用 |

## 六、与其他概念的关系

- 是 [[概念-JMM]] 中"volatile 变量规则"（happens-before）的具体实现；volatile 写 happens-before 后续 volatile 读
- 与 [[机制-synchronized]] 互补：volatile 保证可见性+有序性（轻量），synchronized 额外保证原子性（互斥）；两者不互相包含，各有边界
- 支撑 [[机制-CAS]]：`AtomicInteger` 内部的 `value` 字段是 volatile，保证 CAS 操作总能读到主内存最新值，避免基于旧值做比较

## 七、应用边界

**适合 volatile**：
- 状态标志（`volatile boolean shutdown`）——单次写，多线程读
- 双重检验锁单例模式——禁止对象发布时的重排
- 安全发布不可变对象（写完整后再暴露引用）
- `long`/`double` 字段在需要可见性时（防止 32 位 JVM 的字撕裂）

**不适合 volatile**：
- 计数器（`i++`）→ 用 `AtomicInteger`
- 需要多个变量的原子更新 → `synchronized` 或 `Lock`
- 需要互斥临界区 → `synchronized` 或 `ReentrantLock`
- 高竞争下的状态转换 → `AtomicReference` + CAS，或者用锁
