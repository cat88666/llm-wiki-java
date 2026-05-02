---
type: concept
status: active
name: "JMM"
layer: L3
aliases: ["Java内存模型", "Java Memory Model", "happens-before", "内存屏障", "主内存", "工作内存"]
related:
  - "[[机制-volatile]]"
  - "[[机制-synchronized]]"
  - "[[机制-JVM内存模型]]"
sources:
  - "../../raw/note/📚 Hollis Java/Java并发/✅什么是Java内存模型（JMM）？ 30f3673e113881f09c2af23d5f51d65b.md"
  - "../../raw/note/📚 Hollis Java/Java并发/✅什么是happens-before原则？ 30f3673e113881748a2ff4c6e861a464.md"
  - "../../raw/note/📚 Hollis Java/Java并发/✅到底啥是内存屏障？到底怎么加的？ 30f3673e113881a88c8dc3572b3a9785.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# JMM（Java 内存模型）

> JMM 是 JDK 5+ 起的并发规范——定义主内存与线程工作内存的交互规则，解决多核 CPU 缓存带来的可见性问题、时间片导致的原子性问题、指令重排带来的有序性问题，并提供 happens-before 作为推理工具。

## 第一性原理

多核 CPU 各有 L1/L2 缓存，同一变量的副本可以不一致（可见性）；CPU 时间片切割使"读-改-写"可被中断（原子性）；编译器/CPU 指令重排让实际执行顺序与代码顺序不同（有序性）。JMM 的存在理由：**在屏蔽不同硬件差异的前提下，给 Java 程序员一套统一的多线程内存语义**。

> ⚠️ 面试区分：**JVM 内存结构**（堆/栈/方法区，物理内存布局）vs **JMM**（并发规范，定义操作可见性规则）。

## 核心机制

### 主内存与工作内存

```
主内存（Main Memory）
  ↑↓ load / store（8 种原子操作）
线程 A 工作内存       线程 B 工作内存
（CPU Cache / 寄存器）  （CPU Cache / 寄存器）
```

- 所有**共享变量**存于主内存
- 每个线程有自己的**工作内存**（主内存变量的副本）
- 线程只能操作工作内存，不能直接读写主内存

### 三大并发问题

| 问题 | 根因 | JMM 解决手段 |
|------|------|-------------|
| **可见性** | CPU 多级缓存，各线程工作内存的副本不同步 | `volatile` 写回主内存并刷新；`synchronized` 释放锁前刷新 |
| **原子性** | CPU 时间片，"读-改-写"可被中断 | `synchronized`（monitorenter/monitorexit）；CAS |
| **有序性** | 编译器/CPU 指令重排 | `volatile`（内存屏障禁止重排）；`synchronized`（单线程执行）|

### happens-before 规则

happens-before 是 JMM 的核心推理工具：**若操作 A happens-before 操作 B，则 A 的结果对 B 可见**。

8 条关键规则（节选）：

| 规则 | 含义 |
|------|------|
| 程序次序规则 | 单线程内，前一行 happens-before 后一行 |
| 监视器锁规则 | 解锁 happens-before 后续加锁（synchronized 可见性基础）|
| volatile 变量规则 | volatile 写 happens-before 后续读（volatile 可见性基础）|
| 线程启动规则 | `start()` happens-before 新线程的所有操作 |
| 线程终止规则 | 线程所有操作 happens-before `join()` 返回 |
| 传递性 | A hb B，B hb C → A hb C |

### 内存屏障

`volatile` 的有序性通过**内存屏障（Memory Barrier）**实现：
- **volatile 写**之前插入 StoreStore 屏障（禁止写操作重排到 volatile 写之后）
- **volatile 写**之后插入 StoreLoad 屏障（防止后续读被重排到 volatile 写之前）
- **volatile 读**之后插入 LoadLoad + LoadStore 屏障

**双重校验锁必须用 volatile** 的原因：`new` 操作包含三步（分配内存→初始化对象→赋值引用），后两步可能被重排，不加 volatile 可能导致另一个线程拿到未初始化的对象。

## 关键权衡

1. **JMM vs MESI 协议**：MESI 是硬件级缓存一致性协议；JMM 是软件语言规范，它依赖底层 MESI + 内存屏障指令，但不能完全依赖 MESI（因为 CPU 还有 Store Buffer、无效化队列等）
2. **as-if-serial（单线程）vs happens-before（多线程）**：两者都允许重排序，但都不能改变可观测的结果；区别在于 as-if-serial 只保证单线程视角，happens-before 定义跨线程的可见性约束
3. **JMM 不直接给程序员编程**：程序员用 volatile/synchronized/Lock，JMM 是这些关键字背后的语义保证

## 与其他概念的关系

- [[机制-volatile]] 和 [[机制-synchronized]] 是 JMM 规范的两个主要实现载体
- 与 [[机制-JVM内存模型]] 的区别：JVM 内存结构是物理分区（堆/栈），JMM 是并发语义规范；两者描述维度不同
- 支撑了 L3 并发全部关键字：`volatile`、`synchronized`、`Lock`、`Atomic` 类的语义都建立在 JMM 上

## 应用边界

**需要理解 JMM 的场景**：排查多线程下的变量可见性 bug；理解双重检验锁为何必须加 volatile；理解 synchronized 为何能保证可见性（而不只是互斥）。

**不需要直接操作 JMM**：日常业务开发用 `synchronized`/`volatile`/`java.util.concurrent` 包即可，JMM 是理解这些工具的理论基础。
