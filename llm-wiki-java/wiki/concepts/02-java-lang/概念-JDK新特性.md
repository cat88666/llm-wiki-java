---
type: concept
status: active
name: "JDK新特性"
layer: L1
aliases: ["JDK新特性", "Java新特性", "虚拟线程", "Virtual Threads", "Record", "Sealed Classes", "Lambda", "Stream API", "Optional", "invokedynamic", "JDK21", "JDK17", "JDK16", "JDK11", "JDK8"]
tags: ["#java-lang"]
related:
  - "[[机制-Lambda表达式]]"
  - "[[机制-线程池]]"
  - "[[概念-JMM]]"
sources:
  - "../../../raw/note/架构体系/一、Java 基础与进阶.md"
created: 2026-05-07
updated: 2026-05-07
lint_notes: ""
---

# JDK 新特性

> Java 8 起每隔 6 个月发版，每 2-3 年出一个 LTS 版本（8/11/17/21）；核心演进方向是**函数式编程**（JDK8）→**语言简化**（JDK14-17）→**高并发模型**（JDK21 虚拟线程）。

## 第一性原理

传统 Java 的三个痛点驱动了新特性的演进：
1. **样板代码太多**：匿名内部类/Bean/DTO 需要大量重复代码 → Lambda / Record 简化
2. **类型体系不够表达力**：子类可以随意扩展导致设计意图无法传达 → Sealed Classes
3. **线程太重**：OS 线程 1:1 对应，百万并发需要百万线程开销不可接受 → 虚拟线程

---

## 核心机制

### JDK8 核心特性

**Lambda 表达式**：函数式接口的语法糖，本质通过 `invokedynamic` 指令在运行期生成匿名类（非编译期），性能优于传统匿名内部类（延迟生成，无额外 class 文件）。

```java
// 传统匿名内部类
Runnable r = new Runnable() {
    public void run() { System.out.println("hello"); }
};
// Lambda
Runnable r = () -> System.out.println("hello");
```

**Stream API**：惰性求值流水线，中间操作（`filter/map/flatMap/sorted`）不立即执行，终端操作（`collect/forEach/count/reduce`）触发整个流水线。并行流 `parallelStream()` 底层使用 ForkJoinPool，注意线程安全和拆分成本。

**Optional**：强制调用者处理 null，常用方法：`orElse / orElseGet / orElseThrow / ifPresent / map`。

**LocalDateTime**：线程安全、不可变，替代 `Date/Calendar`。`DateTimeFormatter` 也是线程安全的。

### JDK11-17 重要特性

| 版本 | 特性 | 说明 |
|------|------|------|
| JDK11 | `var` 局部变量推断 / HTTP Client API / LTS | `var list = new ArrayList<String>()` |
| JDK14 | Switch 表达式（`->`语法） / 有用的 NPE | NPE 报告哪个变量为 null |
| JDK16 | **Record 正式版** / `instanceof` 模式匹配 | 参见下文 |
| JDK17 | **Sealed Classes 正式版** / LTS | 参见下文 |

**Record（JDK16）**：不可变数据类，编译器自动生成构造器、`getter`、`equals`、`hashCode`、`toString`：

```java
record Point(int x, int y) {}
// 等价于带 final 字段的 class + 自动生成上述方法
Point p = new Point(3, 4);
System.out.println(p.x()); // 3（方法名=字段名，无 get 前缀）
```

**适用场景**：DTO、值对象（Value Object）、方法返回多值。**不适合**：需要继承、需要可变状态的场景。

**Sealed Classes（JDK17）**：`permits` 子句限制合法子类，编译器可以穷举所有子类型（配合 `pattern matching switch` 实现类型安全的代数数据类型）：

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
// 使用 pattern matching switch 时，若覆盖所有子类则无需 default
String desc = switch (shape) {
    case Circle c    -> "圆，半径=" + c.radius();
    case Rectangle r -> "矩形";
    case Triangle t  -> "三角形";
};
```

### JDK21：虚拟线程（Virtual Threads）

**核心问题**：传统 `Thread` 是 OS 线程的 1:1 包装，每个线程占用约 1MB 栈内存；创建/切换代价高，高并发 IO 密集型服务（如每个请求一个线程）上限约数万线程。

**虚拟线程模型**：JVM 管理的轻量级线程，挂载在少量 OS 载体线程（Carrier Threads）上。IO 阻塞时，JVM 自动将虚拟线程从载体线程上卸载，让载体线程处理其他虚拟线程；IO 完成后重新挂载。

```
                   ┌── VThread-1 (RUNNING)
OS Thread-1 ───────┤
                   └── VThread-2 (BLOCKED on IO → 卸载)

OS Thread-2 ───────── VThread-2 重新挂载（IO完成后）
```

```java
// 创建方式
Thread vt = Thread.ofVirtual().start(() -> System.out.println("virtual"));
// 或通过线程池（每个任务一个虚拟线程）
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
```

**适用**：IO 密集型，如 HTTP 调用、数据库查询（每请求一线程模型）。
**不适用**：CPU 密集型（虚拟线程不能并行跑 CPU 计算）；持有 `synchronized` 锁时会 pin 住载体线程（应改用 `ReentrantLock`）。

---

## 关键权衡

1. **Lambda 可读性 vs 调试难度**：嵌套 Lambda 难以调试（堆栈信息不直观）；复杂逻辑仍建议抽方法
2. **Stream 并行流 vs 线程安全**：`parallelStream` 共用 ForkJoinPool，CPU 密集型有收益；IO 密集型反而因线程切换变慢；流中不能使用非线程安全的状态
3. **Record 简洁 vs 灵活**：Record 不能继承（隐式 final），字段不可变；适合纯数据容器，不适合富领域对象
4. **虚拟线程 vs 响应式**：虚拟线程让同步风格代码获得异步性能，比响应式（WebFlux/RxJava）学习曲线低很多；但响应式在极低延迟和背压控制上更精细

## 与其他概念的关系

- **依赖 [[机制-Lambda表达式]]**：Lambda/Stream 是 JDK8 的基础，invokedynamic 实现原理见机制-Lambda表达式
- **关联 [[机制-线程池]]**：虚拟线程替代的是 `newCachedThreadPool` 等 IO 密集型场景，不替代 CPU 密集型的固定线程池
- **关联 [[概念-JMM]]**：虚拟线程遵守同样的 JMM 可见性规则；`synchronized` 在虚拟线程上会 pin 载体线程

## 应用边界

**Record** 适合：DTO、响应体、方法多返回值包装、DDD 值对象。
**Sealed + Pattern Matching** 适合：状态机、AST 节点、协议消息类型。
**虚拟线程** 适合：每请求一线程的 IO 密集型服务（替代线程池 + CompletableFuture 的繁琐写法）；JDK21 LTS 新项目首选。
