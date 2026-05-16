---
type: concept
status: active
name: "JMM"
layer: L3
aliases: ["Java内存模型", "Java Memory Model", "happens-before", "内存屏障", "主内存", "工作内存", "StoreStore", "StoreLoad", "LoadLoad", "Store Buffer", "可见性", "有序性", "as-if-serial"]
related:
  - "[[机制-Volatile]]"
  - "[[机制-Synchronized]]"
  - "[[机制-JVM内存模型]]"
  - "[[机制-CAS]]"
---

# JMM（Java 内存模型）

> JMM 是 JDK 5（JSR-133）起定义的并发规范——规定主内存与线程工作内存的交互规则，提供 happens-before 作为可见性推理工具，通过内存屏障指令屏蔽不同硬件架构的差异，统一 Java 多线程的内存语义。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 三大并发问题、JMM 存在理由、JMM vs JVM内存结构 |
| [二、核心机制](#二核心机制) | 主内存/工作内存、happens-before 8条规则、内存屏障 |
| [三、硬件层真相](#三硬件层真相) | Store Buffer、Invalidation Queue、MESI 不够用的原因 |
| [四、核心使用原则](#四核心使用原则) | DCL 双重检验锁、as-if-serial 对比 |
| [五、综合对比](#五综合对比) | JMM vs JVM 内存结构、JMM vs MESI |
| [六、与其他概念的关系](#六与其他概念的关系) | volatile、synchronized、CAS |
| [七、应用边界](#七应用边界) | 何时需要理解 JMM、何时不需要直接操作 |

## 一、第一性原理

多核 CPU 各有 L1/L2 缓存，同一变量的副本可以不一致（**可见性**）；CPU 时间片切割使"读-改-写"可被中断（**原子性**）；编译器/CPU 指令重排让实际执行顺序与代码顺序不同（**有序性**）。

JMM 的存在理由：**在屏蔽不同硬件差异的前提下，给 Java 程序员一套统一的多线程内存语义**，不依赖底层 CPU 具体实现。

> 面试必辨：**JVM 内存结构**（堆/栈/方法区，描述物理内存布局）vs **JMM**（并发规范，定义操作可见性规则）——两者描述维度完全不同。

| 问题 | 根因 | JMM 解决手段 |
|------|------|-------------|
| **可见性** | CPU 多级缓存，各线程工作内存副本不同步 | `volatile` 写回主内存；`synchronized` 释放锁前刷新 |
| **原子性** | CPU 时间片，"读-改-写"可被中断 | `synchronized`（monitorenter/monitorexit）；CAS |
| **有序性** | 编译器/CPU 指令重排 | `volatile`（内存屏障禁止重排）；`synchronized`（单线程串行）|

## 二、核心机制

### 主内存与工作内存

```
主内存（Main Memory）—— 所有共享变量
  ↑↓ load / store（8 种原子操作）
线程 A 工作内存（CPU L1/L2 Cache / 寄存器）
线程 B 工作内存（CPU L1/L2 Cache / 寄存器）
```

- 所有**共享变量**存于主内存（实例字段、静态字段、数组元素）
- 每个线程有自己的**工作内存**（主内存变量的副本，对应 CPU 缓存/寄存器）
- 线程只能操作工作内存，不能直接读写主内存

### happens-before 规则

happens-before 是 JMM 的核心推理工具：**若操作 A happens-before 操作 B，则 A 的结果对 B 可见，且 A 的执行不会被重排到 B 之后**。

注意：happens-before 只要求可见性和有序性，不要求物理上 A 先执行——编译器可以重排，但不能让 B 看不到 A 的结果。

| 规则 | 含义 | 应用场景 |
|------|------|---------|
| **程序次序规则** | 单线程内，前一行 hb 后一行 | 单线程代码顺序保证 |
| **监视器锁规则** | 解锁 hb 后续加锁 | synchronized 可见性基础 |
| **volatile 变量规则** | volatile 写 hb 后续读 | volatile 可见性基础 |
| **线程启动规则** | `start()` hb 新线程所有操作 | 主线程写 → 子线程读 |
| **线程终止规则** | 线程所有操作 hb `join()` 返回 | 子线程写 → 主线程读 |
| **线程中断规则** | `interrupt()` hb 被中断线程检测中断 | 中断通知 |
| **对象终结规则** | 构造函数结束 hb `finalize()` | 对象生命周期 |
| **传递性** | A hb B，B hb C → A hb C | 组合推导 |

**高频面试考点**：监视器锁规则 + volatile 变量规则 + 传递性是 DCL 双重检验锁能正确工作的理论基础。

### 内存屏障

`volatile` 和 `synchronized` 的有序性通过**内存屏障（Memory Barrier）**实现：

| 操作 | 插入屏障 | 效果 |
|------|---------|------|
| volatile **写**之前 | `StoreStore` | 禁止该写前的普通写被重排到 volatile 写之后 |
| volatile **写**之后 | `StoreLoad` | 确保 volatile 写对后续读可见（最重量级屏障）|
| volatile **读**之后 | `LoadLoad` | 禁止后续普通读被重排到 volatile 读之前 |
| volatile **读**之后 | `LoadStore` | 禁止后续普通写被重排到 volatile 读之前 |

**StoreLoad 是四种屏障中开销最大的**（刷新 Store Buffer），x86 上对应 `LOCK; ADDL $0,0(%rsp)` 或 `MFENCE`。

## 三、硬件层真相

### 为什么 MESI 协议不够用

MESI 是 CPU 缓存一致性协议，能保证多 CPU 对同一缓存行的值一致。但 CPU 还引入了：

**Store Buffer（写缓冲区）**：CPU 写操作先写入 Store Buffer，异步刷到缓存行，不等待其他 CPU 的 invalidate 确认——提升写性能，但制造了可见性延迟。

**Invalidation Queue（失效队列）**：收到其他 CPU 的 invalidate 请求，不立即使缓存行失效，而是放入队列异步处理——提升 CPU 响应速度，但读方可能读到尚未失效的旧值。

```
CPU A                       CPU B
  │ write x=1               │
  │ → Store Buffer（异步）   │
  │                         │ read x → 读到 0（旧值！）
  │                         │ Invalidation Queue 还没处理
```

**结论**：MESI 在有 Store Buffer + Invalidation Queue 的现代 CPU 上不足以保证 JMM 语义，内存屏障的作用就是**强制刷新 Store Buffer、强制清空 Invalidation Queue**。

### as-if-serial vs happens-before

| 维度 | as-if-serial | happens-before |
|------|-------------|---------------|
| 范围 | 单线程 | 跨线程 |
| 核心保证 | 单线程内重排不改变可观测结果 | 定义跨线程操作的可见性约束 |
| 是否允许重排 | 允许，只要单线程结果不变 | 允许，只要满足规则对可见性的约束 |
| 目的 | 让编译器/CPU 优化单线程代码 | 让 JVM 规范多线程可见性 |

## 四、核心使用原则

### 双重检验锁（DCL）为何必须加 volatile

```java
private volatile static Singleton instance;  // 必须 volatile！

public static Singleton getInstance() {
    if (instance == null) {                    // ① 第一次检查（无锁，快路径）
        synchronized (Singleton.class) {
            if (instance == null) {            // ② 第二次检查（有锁）
                instance = new Singleton();    // ③ 这里是关键
            }
        }
    }
    return instance;
}
```

**③ `new Singleton()` 编译为三步字节码**：
1. `new`：在堆上分配内存
2. `invokespecial <init>`：初始化对象
3. `astore`：将引用写入 `instance`

JIT 可能将步骤 2、3 重排为 3、2（对象未初始化但引用已赋值）。另一线程在步骤①看到 `instance != null`，拿到未初始化的对象，调用方法时触发 NPE。

volatile 的 **StoreStore 屏障**禁止②③之间重排，**StoreLoad 屏障**保证 ③ 写后对其他线程立即可见。

### 不需要 JMM 的场景

日常业务开发用 `synchronized`/`volatile`/`java.util.concurrent` 包即可。JMM 是这些工具的理论基础，不需要手动插入内存屏障或管理 happens-before。

## 五、综合对比

| 维度 | JMM | JVM 内存结构 |
|------|-----|------------|
| 描述对象 | 线程间的内存操作规则 | Java 进程的内存布局（堆/栈/方法区）|
| 解决问题 | 并发可见性/有序性 | 对象存储/GC/栈帧管理 |
| 主内存 | 所有线程共享变量的逻辑存储位置 | 对应堆（及方法区）|
| 工作内存 | 线程私有的缓存副本（逻辑概念）| 对应栈 + CPU 寄存器/缓存 |
| 面试混淆点 | 经常被误认为是 JVM 堆栈结构 | 经常被误用来解释并发可见性 |

| 维度 | JMM | MESI 协议 |
|------|-----|---------|
| 层次 | 软件规范（语言级）| 硬件协议（CPU 级）|
| 作用 | 定义 Java 并发的可见性语义 | 保证多核缓存一致性 |
| 是否足够 | 需要内存屏障补充 | 有 Store Buffer 等绕过机制 |

## 六、与其他概念的关系

- [[机制-Volatile]]：JMM "volatile 变量规则"的具体实现，通过 LOCK 前缀指令 + 内存屏障落地
- [[机制-Synchronized]]：JMM "监视器锁规则"的具体实现，monitorenter/exit 附带 happens-before 保证
- 与 [[机制-JVM内存模型]] 的区别：JVM 内存结构是物理分区（堆/栈），JMM 是并发语义规范
- 支撑 [[机制-CAS]]：`AtomicInteger` 的 `volatile value` 字段结合 CAS，是 JMM 可见性 + 原子性的双重利用

## 七、应用边界

**需要理解 JMM 的场景**：
- 排查多线程变量可见性 bug（某线程写了但另一线程看不到）
- 理解 DCL 为何必须加 volatile
- 设计无锁数据结构时验证操作的可见性约束
- 理解 synchronized 为何能保证可见性（而不只是互斥）

**不直接操作 JMM**：日常 CRUD 开发使用标准并发工具即可，JMM 是底层语义，不是编程 API。
