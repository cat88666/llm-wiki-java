---
type: concept
status: active
name: "JDK新特性"
layer: L1
aliases: ["JDK新特性", "Java新特性", "虚拟线程", "Virtual Threads", "Record", "Sealed Classes", "Lambda", "Stream API", "Optional", "invokedynamic", "JDK21", "JDK17", "JDK16", "JDK11", "JDK8"]
tags: ["#java-lang"]
related:
  - "[[机制-Lambda]]"
  - "[[机制-线程池]]"
  - "[[概念-JMM]]"
sources:
  - "../../../raw/note/架构体系/一、Java 基础与进阶.md"
created: 2026-05-07
updated: 2026-05-14
lint_notes: ""
---

# JDK 新特性

> Java 8 起每 6 个月发版，核心演进路径：**函数式编程**（JDK 8）→ **语言简化与类型表达力**（JDK 11-17）→ **高并发轻量线程模型**（JDK 21）。LTS 版本为 8 / 11 / 17 / 21。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 样板代码、类型表达力、线程模型三大痛点 |
| [二、核心机制 — JDK 8](#二核心机制--jdk-8) | Lambda、invokedynamic、Stream、Optional、LocalDateTime |
| [三、核心机制 — JDK 11-17](#三核心机制--jdk-11-17) | var、Record、Sealed Classes、Pattern Matching |
| [四、核心机制 — JDK 21](#四核心机制--jdk-21) | 虚拟线程模型、挂载/卸载、pin 问题 |
| [五、核心使用原则](#五核心使用原则) | Lambda 简洁度、Stream 适用场景、虚拟线程选型 |
| [六、综合对比](#六综合对比) | LTS 版本选型、虚拟线程 vs 响应式 |
| [七、生产风险](#七生产风险) | parallelStream 陷阱、Record 限制、虚拟线程 pin |
| [八、与其他概念的关系](#八与其他概念的关系) | Lambda 机制、线程池、JMM |
| [九、应用边界](#九应用边界) | 各特性适用与不适用场景 |

## 一、第一性原理

三个根本痛点驱动了 Java 语言演进：

| 痛点 | 特性 | 解决方式 |
|------|------|---------|
| 样板代码过多 | Lambda / Record | 消除匿名内部类、DTO 模板代码 |
| 类型体系表达力不足 | Sealed Classes / Pattern Matching | 限定子类集合，编译器穷举检查 |
| OS 线程太重 | 虚拟线程（JDK 21） | M:N 线程模型，JVM 调度轻量线程 |

## 二、核心机制 — JDK 8

### 2.1 Lambda 表达式

函数式接口（仅含一个抽象方法的接口）的简写形式。底层通过 `invokedynamic` 指令在首次调用时生成匿名类（非编译期生成），性能优于传统匿名内部类：无额外 `.class` 文件、延迟生成、可被 JIT 优化。

```java
// 匿名内部类
Runnable r = new Runnable() { public void run() { doWork(); } };
// Lambda
Runnable r = () -> doWork();
```

### 2.2 Stream API

惰性求值流水线。中间操作（`filter` / `map` / `flatMap` / `sorted` / `distinct`）不立即执行，终端操作（`collect` / `forEach` / `count` / `reduce`）触发整条流水线。

**parallelStream 注意事项**：
- 底层使用公共 `ForkJoinPool`，默认线程数 = CPU 核数
- CPU 密集型有收益；IO 密集型反而因线程切换变慢
- 流中不能使用非线程安全的可变状态
- 数据源拆分成本高（如 `LinkedList`）时并行无收益

### 2.3 Optional

强制调用者显式处理 null。核心方法：`orElse` / `orElseGet` / `orElseThrow` / `map` / `flatMap` / `ifPresent`。

> **面试误区**：`orElse(defaultValue)` 无论 Optional 是否有值都会执行参数表达式（如方法调用），`orElseGet(Supplier)` 仅在值为空时执行——高开销默认值必须用 `orElseGet`。

### 2.4 日期时间 API

`LocalDateTime` / `ZonedDateTime` / `Instant`：不可变、线程安全，替代 `Date` / `Calendar`。`DateTimeFormatter` 也是线程安全的（`SimpleDateFormat` 不是）。

## 三、核心机制 — JDK 11-17

### 3.1 版本特性速查

| 版本 | 核心特性 | 要点 |
|------|---------|------|
| JDK 10 | `var` 局部变量推断 | 仅限局部变量，不改变强类型本质 |
| JDK 11 (LTS) | HTTP Client API / String 增强 | `HttpClient` 替代 `HttpURLConnection`，支持 HTTP/2 |
| JDK 14 | Switch 表达式 / 有用的 NPE | NPE 信息明确指出哪个变量为 null |
| JDK 16 | Record 正式版 / `instanceof` 模式匹配 | 见下文 |
| JDK 17 (LTS) | Sealed Classes 正式版 | 见下文 |

### 3.2 Record（JDK 16）

不可变数据载体，编译器自动生成：全参构造器、字段访问方法（方法名 = 字段名，无 `get` 前缀）、`equals`、`hashCode`、`toString`。

```java
record Point(int x, int y) {}
Point p = new Point(3, 4);
p.x(); // 3
```

- 隐式 `final`，不能被继承
- 字段隐式 `final`，不可变
- 可实现接口，可定义 Compact Constructor 做参数校验

### 3.3 Sealed Classes（JDK 17）

`permits` 子句限定合法子类集合，编译器可穷举所有子类型：

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
// Pattern Matching Switch（JDK 21 正式版）
String desc = switch (shape) {
    case Circle c    -> "圆，半径=" + c.radius();
    case Rectangle r -> "矩形";
    case Triangle t  -> "三角形";
    // 覆盖所有子类，无需 default
};
```

配合 Pattern Matching 构成类型安全的代数数据类型（ADT），适合状态机、AST 节点、协议消息建模。

## 四、核心机制 — JDK 21

### 4.1 虚拟线程（Virtual Threads）

**问题**：传统 `Thread` 是 OS 线程的 1:1 包装，每个线程约占 1MB 栈内存，创建/上下文切换代价高，IO 密集型服务线程上限约数万。

**模型**：JVM 管理的轻量级线程（约几 KB），挂载在少量 OS 载体线程（Carrier Threads）上。IO 阻塞时 JVM 自动卸载虚拟线程，让载体线程处理其他虚拟线程；IO 完成后重新挂载。

```text
OS Carrier Thread-1 ─── VThread-A (RUNNING)
                    └── VThread-B (IO阻塞 → 卸载，载体线程空闲)

OS Carrier Thread-2 ─── VThread-B (IO完成 → 重新挂载)
```

### 4.2 创建方式

```java
// 直接创建
Thread.ofVirtual().start(() -> handleRequest());

// 虚拟线程池（每任务一线程，推荐方式）
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    exec.submit(() -> handleRequest());
}
```

### 4.3 Pin 问题

虚拟线程在以下情况会 **pin 住载体线程**（无法卸载），退化为普通线程：
- 持有 `synchronized` 锁时发生阻塞
- 执行 native 方法 / JNI 调用时

**应对**：将 `synchronized` 替换为 `ReentrantLock`。JDK 24 计划解决 synchronized pin 问题。

## 五、核心使用原则

1. **Lambda 复杂逻辑抽方法**：嵌套 Lambda 堆栈信息不直观，超过 3 行逻辑应提取为命名方法。
2. **Stream 不是万能替代 for 循环**：简单遍历用 for 更直观；需要中间结果、break/continue、checked exception 时 for 更合适。
3. **parallelStream 需评估**：仅 CPU 密集型 + 大数据量 + 数据源易拆分（如 ArrayList）时有收益。
4. **Record 用于纯数据**：DTO、值对象、方法多返回值包装。不适合富领域对象（需要继承/可变状态）。
5. **虚拟线程用于 IO 密集型**：每请求一线程模型，替代 `newCachedThreadPool` + `CompletableFuture` 的繁琐写法。不用于 CPU 密集型。

## 六、综合对比

### 6.1 LTS 版本选型

| 维度 | JDK 8 | JDK 11 | JDK 17 | JDK 21 |
|------|-------|--------|--------|--------|
| Lambda / Stream | 有 | 有 | 有 | 有 |
| var | 无 | 有 | 有 | 有 |
| Record | 无 | 无 | 有（JDK 16 引入） | 有 |
| Sealed Classes | 无 | 无 | 有 | 有 |
| 虚拟线程 | 无 | 无 | 无 | 有 |
| 模块化强封装 | 无（Classpath 模式） | 宽松 | 严格 | 严格 |
| 新项目推荐 | 仅维护老项目 | 过渡期 | 稳健选择 | 首选 |

### 6.2 虚拟线程 vs 响应式（WebFlux / RxJava）

| 维度 | 虚拟线程 | 响应式 |
|------|---------|--------|
| 编程模型 | 同步阻塞风格，代码直观 | 异步回调链，学习曲线陡峭 |
| 调试 | 常规堆栈，易调试 | 堆栈碎片化，难调试 |
| 背压控制 | 无内建背压 | 精细背压（Reactor/RxJava） |
| 生态成熟度 | JDK 21 起，逐步适配 | 成熟，Spring WebFlux 广泛使用 |
| 适用场景 | IO 密集型微服务（新项目首选） | 极低延迟、流式处理、已有响应式代码库 |

## 七、生产风险

| 风险 | 根因 | 应对 |
|------|------|------|
| parallelStream 共用 ForkJoinPool 导致阻塞 | 公共池被慢任务占满 | 自定义 ForkJoinPool 提交任务 |
| Stream 内抛 checked exception 编译报错 | 函数式接口不声明 checked exception | 包装为 unchecked 或使用 Vavr/Try |
| Record 字段不可变导致框架兼容性问题 | 部分框架依赖 setter（如旧版 MyBatis） | 确认框架版本支持 Record |
| Sealed Classes 需所有子类在同一编译单元 | 密封类设计约束 | 子类放同一包或同一模块 |
| 虚拟线程 pin 载体线程 | synchronized + 阻塞 IO | 替换为 ReentrantLock |
| 虚拟线程 + ThreadLocal 内存膨胀 | 百万虚拟线程各持 ThreadLocal 副本 | 使用 ScopedValue（JDK 21 预览） |
| JDK 17+ 强封装破坏旧反射代码 | 模块化限制 `setAccessible` | 添加 `--add-opens` 或迁移到公开 API |

## 八、与其他概念的关系

- 依赖 [[机制-Lambda]]：Lambda / Stream 是 JDK 8 的基础，底层 `invokedynamic` 实现原理详见机制-Lambda
- 关联 [[机制-线程池]]：虚拟线程替代 `newCachedThreadPool` 等 IO 密集型场景的传统线程池方案，不替代 CPU 密集型固定线程池
- 关联 [[概念-JMM]]：虚拟线程遵守同样的 JMM 可见性规则；`synchronized` 在虚拟线程上会 pin 载体线程

## 九、应用边界

| 特性 | 适合 | 不适合 |
|------|------|--------|
| Lambda / Stream | 集合操作、函数式管道、回调简化 | 复杂控制流（break/continue/try-catch）、高嵌套场景 |
| Record | DTO、响应体、方法多返回值、DDD 值对象 | 需要继承、需要可变状态、富领域对象 |
| Sealed + Pattern Matching | 状态机、AST 节点、协议消息类型、有限子类型建模 | 开放扩展的插件体系 |
| 虚拟线程 | IO 密集型每请求一线程服务（JDK 21 新项目首选） | CPU 密集型计算、需要精细背压控制、大量 synchronized 代码 |
