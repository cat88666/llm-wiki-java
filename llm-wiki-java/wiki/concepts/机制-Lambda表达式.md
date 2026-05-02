---
type: concept
status: active
name: "Lambda表达式"
layer: L1
aliases: ["Lambda", "函数式接口", "Stream", "invokedynamic"]
related:
  - "[[机制-泛型类型擦除]]"
  - "[[概念-OOP三大特征]]"
sources:
  - "../../raw/note/📚 Hollis Java/Java基础/✅Lambda表达式是如何实现的？ 30f3673e11388107b5d2d6a2b4bac2af.md"
  - "../../raw/note/📚 Hollis Java/Java基础/✅Stream的并行流一定比串行流更快吗？ 30f3673e113881e187cac19bf80b15b3.md"
  - "../../raw/note/📚 Hollis Java/Java基础/✅说几个常见的语法糖？ 30f3673e113881ad9679f220d9e3bb28.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# Lambda表达式

> Lambda 是函数式接口的匿名实现——把"一段行为"当作值传递，底层通过 `invokedynamic` 实现，非匿名内部类的语法糖。

## 第一性原理

在 Lambda 之前，传递"行为"只能用匿名内部类，每次都需要定义类、实现接口，代码冗余。Lambda 的存在理由：**让行为像值一样传递，消除仅为传递单一方法而写的样板类**。

## 核心机制

### Lambda 不是匿名内部类的语法糖

- 匿名内部类编译后生成 `Outer$1.class`（额外 class 文件）
- Lambda 编译后**不生成额外 class 文件**，而是在调用处生成 `lambda$main$0` 等私有静态方法

底层实现：`invokedynamic` 指令 → `LambdaMetafactory.metafactory()` → 运行时生成函数式接口的实现

```java
// 源码
list.forEach(s -> System.out.println(s));

// 反编译等价
// 1. 编译器生成私有方法：
private static void lambda$main$0(String s) { System.out.println(s); }
// 2. 调用处用 invokedynamic 链接到该方法
```

### 函数式接口

Lambda 只能赋值给**函数式接口**（`@FunctionalInterface`）：只有一个抽象方法的接口。

常用内置函数式接口：

| 接口 | 签名 | 用途 |
|------|------|------|
| `Runnable` | `() -> void` | 无参无返回 |
| `Supplier<T>` | `() -> T` | 无参有返回（生产者）|
| `Consumer<T>` | `T -> void` | 有参无返回（消费者）|
| `Function<T,R>` | `T -> R` | 转换 |
| `Predicate<T>` | `T -> boolean` | 判断 |

### Stream API 与并行流

```java
list.stream()
    .filter(s -> s.startsWith("A"))   // 中间操作，懒执行
    .map(String::toUpperCase)          // 中间操作
    .collect(Collectors.toList());     // 终止操作，触发计算
```

**并行流不一定更快**：
- `parallelStream()` 使用 `ForkJoinPool.commonPool()` 分配线程
- 元素少（< 1000）：线程切换开销 > 计算收益
- 有状态操作（`sorted`、`limit`）：并行反而需要同步，更慢
- IO 密集型：CPU 并行无法加速 IO 等待
- 适合场景：大数据量纯 CPU 计算、元素间无依赖

## 关键权衡

1. **性能**：Lambda 首次调用有 `invokedynamic` 链接开销（后续正常）；比匿名内部类略快（无 class 加载）
2. **调试困难**：Lambda 栈帧显示为 `lambda$main$0`，不直观
3. **可读性**：链式 Stream 适度（3-5步），过长难以理解；复杂逻辑抽成命名方法
4. **并行流风险**：共享可变状态在并行流中是并发 Bug 的来源

## 与其他概念的关系

- 依赖 [[机制-泛型类型擦除]]：`Function<T,R>`、`Predicate<T>` 等函数式接口都是泛型的
- 与 [[概念-OOP三大特征]] 互补：Lambda 引入函数式思想，面向对象之外的另一种抽象方式
- 支撑了 L3 并发：`CompletableFuture.thenApply(Function)` 等异步 API 全面依赖 Lambda

## 应用边界

**适合用 Lambda**：集合操作（filter/map/reduce）；事件回调；Builder 模式中的配置；简单的行为传递。

**不适合用 Lambda**：复杂逻辑（超过 5 行的 Lambda 应抽成命名方法）；需要调试的关键路径；并行流处理共享可变状态。
