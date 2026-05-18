---
type: concept
status: active
name: "Lambda与函数式编程"
layer: L1
aliases: ["Lambda", "函数式接口", "Stream", "invokedynamic", "FunctionalInterface"]
related:
  - "[[机制-泛型]]"
  - "[[概念-OOP特征]]"
  - "[[机制-动态代理]]"
sources:
  - "../../../raw/note/Hollis/Java基础/"
updated: 2026-05-18
---

# Lambda表达式

> Lambda 把“行为”变成一等值，核心收益是减少样板代码与提升组合能力，核心代价是调试可读性和并发误用风险。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、Lambda 解决的核心问题](#sec-2) | 行为参数化替代匿名内部类 |
| [二、底层执行机制](#sec-3) | invokedynamic、LambdaMetafactory、变量捕获 |
| [三、函数式接口体系](#sec-4) | Supplier/Consumer/Function/Predicate 选型 |
| [四、Stream 与方法引用实战](#sec-5) | 流水线、短路、并行流边界 |
| [五、设计与编码原则](#sec-6) | 纯函数优先、副作用收敛、表达式长度控制 |
| [六、Lambda 与替代方案对比](#sec-7) | Lambda vs 匿名类 vs 命名方法 |
| [七、生产风险与排查](#sec-8) | 并行流共享状态、commonPool 争用、序列化陷阱 |
| [八、关系与边界](#sec-9) | 与泛型、OOP、并发模型的关系 |
| [九、面试速答口径](#sec-10) | 高频追问答案 |

<a id="sec-2"></a>
## 一、Lambda 解决的核心问题

Java 8 之前，传递行为通常要写匿名内部类，代码噪音大、组合困难。Lambda 的本质是行为参数化。

```java
// 匿名内部类
executor.submit(new Runnable() {
    @Override
    public void run() { doWork(); }
});

// Lambda
executor.submit(() -> doWork());
```

收益链路：

- 更少样板代码
- 更强函数组合（`map/filter/reduce`）
- 更高可读性（前提是表达式短小）

<a id="sec-3"></a>
## 二、底层执行机制

### 2.1 Lambda 不是匿名内部类语法糖

| 维度 | 匿名内部类 | Lambda |
| --- | --- | --- |
| 编译产物 | 额外 `Outer$1.class` | 无额外类文件 |
| 字节码调用 | `new + invokespecial` | `invokedynamic` |
| 运行时绑定 | 类加载器加载匿名类 | `LambdaMetafactory` 动态生成目标实现 |
| `this` 指向 | 匿名类实例 | 外围类实例 |

### 2.2 执行流程

```text
源码 lambda
  -> 编译为 invokedynamic 调用点
  -> 首次触发 Bootstrap 方法
  -> LambdaMetafactory 生成函数式接口实现
  -> CallSite 缓存，后续复用
```

### 2.3 变量捕获（effectively final）

Lambda 捕获外部局部变量时，变量必须“事实不可变”。原因是捕获值会在编译后转为参数快照；若允许修改，会产生语义分裂。

<a id="sec-4"></a>
## 三、函数式接口体系

| 接口 | 签名 | 语义 | 常见场景 |
| --- | --- | --- | --- |
| `Supplier<T>` | `() -> T` | 供给 | 延迟构建、默认值 |
| `Consumer<T>` | `T -> void` | 消费 | 日志、遍历副作用 |
| `Function<T,R>` | `T -> R` | 转换 | DTO 映射、管道转换 |
| `Predicate<T>` | `T -> boolean` | 断言 | 条件过滤 |
| `BiFunction<T,U,R>` | `(T,U)->R` | 双输入转换 | 合并逻辑 |
| `UnaryOperator<T>` | `T -> T` | 同类型变换 | 归一化处理 |

反直觉点：`@FunctionalInterface` 只限制抽象方法数量，不限制 `default/static` 方法数量。

<a id="sec-5"></a>
## 四、Stream 与方法引用实战

### 4.1 Stream 流水线

```java
List<String> result = list.stream()
    .filter(s -> s.length() > 2)
    .map(String::toUpperCase)
    .sorted()
    .toList();
```

- 中间操作惰性执行
- 终止操作才触发计算
- 单个 Stream 只能消费一次

### 4.2 并行流边界

| 场景 | `parallelStream` 建议 | 结论 |
| --- | --- | --- |
| CPU 密集 + 数据量大 | 可尝试 | 可能获益 |
| IO 密集 | 不建议 | 线程切换开销高 |
| 有共享可变状态 | 禁止 | 竞态风险高 |
| 服务端混用其他线程池 | 谨慎 | `commonPool` 争用 |

### 4.3 方法引用选择

- 逻辑简单、只转发调用时优先方法引用
- 涉及多步逻辑时仍用 Lambda，可读性更高

<a id="sec-6"></a>
## 五、设计与编码原则

1. Lambda 函数体尽量单一职责，超过 3 行可提取命名方法。
2. 业务核心链路避免在 Lambda 里做复杂副作用（写库、远程调用、锁）。
3. `Optional.orElseGet` 优先于 `orElse`（高成本默认值时）。
4. 并行流默认不替代线程池设计，吞吐优化先看 profiling 证据。

<a id="sec-7"></a>
## 六、Lambda 与替代方案对比

| 方案 | 优势 | 劣势 | 适用场景 |
| --- | --- | --- | --- |
| Lambda | 简洁、组合强 | 调试堆栈可读性一般 | 集合转换、轻量行为传递 |
| 匿名内部类 | 语义直观 | 样板多 | 需要维护内部状态时 |
| 命名方法 + 方法引用 | 可读性强、可复用 | 代码分散 | 公共策略逻辑 |

<a id="sec-8"></a>
## 七、生产风险与排查

| 风险 | 触发条件 | 现象 | 应对 |
| --- | --- | --- | --- |
| 并行流数据竞争 | Lambda 写共享变量 | 结果不稳定 | 改为无副作用聚合 |
| `commonPool` 饥饿 | 长任务占满公共池 | 请求 RT 抖动 | 使用专用线程池 |
| 可读性下降 | 链式调用过长 | 维护困难 | 提取中间变量/命名方法 |
| Lambda 序列化问题 | 框架要求可序列化闭包 | 兼容异常 | 明确协议或改命名类 |

<a id="sec-9"></a>
## 八、关系与边界

- 依赖 [[机制-泛型]]：函数式接口广泛使用泛型承载输入输出。
- 关联 [[概念-OOP特征]]：Lambda 强化“行为抽象”，但不替代面向对象建模。
- 关联 [[机制-动态代理]]：两者都在运行期处理行为，关注点不同（函数组合 vs 调用拦截）。

边界：Lambda 提升表达力，但不是性能银弹，更不是并发模型替代品。

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| Lambda 是匿名内部类语法糖吗 | 不是；底层是 `invokedynamic + LambdaMetafactory` |
| 为什么捕获变量要 effectively final | 捕获的是值快照，允许修改会破坏语义一致性 |
| 并行流为什么常翻车 | `commonPool` 争用 + 共享状态竞态 + 拆分成本 |
| 什么时候不用 Lambda | 逻辑复杂、需调试可读性、强副作用流程 |
