---
type: concept
status: active
name: "CompletableFuture"
layer: L3
aliases: ["CompletableFuture", "异步编程", "ForkJoinPool", "Completion链", "thenApply", "thenCompose", "allOf", "anyOf", "thenCombine", "exceptionally", "事件驱动", "链式编排"]
related:
  - "[[机制-线程池]]"
  - "[[概念-JMM]]"
  - "[[机制-AQS]]"
---

# CompletableFuture

> CompletableFuture 是 Java 8 引入的异步编程模型——以**链式 Completion 阶段 + 事件驱动**替代阻塞式 `Future.get()`，允许任务串行/并行/聚合编排，且不阻塞调用线程；底层通过 Completion 链表实现回调，上一阶段完成时自动触发下一阶段。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、从阻塞 Future 到事件驱动](#一从阻塞-future-到事件驱动) | Future.get() 的局限、Completion 回调模型 |
| [二、Completion 链与默认线程池](#二completion-链与默认线程池) | Completion 链、result 字段、ForkJoinPool |
| [三、异步编排 API 分类](#三异步编排-api-分类) | 创建、串行编排、并行聚合、异常处理 |
| [四、任务聚合与超时场景](#四任务聚合与超时场景) | 并行查询聚合、批量异步、超时控制 |
| [五、CompletableFuture 语义对比](#五completablefuture-语义对比) | thenApply vs thenApplyAsync、CompletableFuture vs CountDownLatch |
| [六、异步编排生产风险](#六异步编排生产风险) | ForkJoinPool 陷阱、异常吞噬、join() vs get() |
| [七、CompletableFuture 的依赖关系](#七completablefuture-的依赖关系) | 线程池、JMM、AQS |
| [八、CompletableFuture 适用边界](#八completablefuture-适用边界) | 适合/不适合、虚拟线程趋势 |

## 一、从阻塞 Future 到事件驱动

Java 5 引入 `Future`，可提交异步任务但只能通过 `get()` 阻塞等待结果——本质上是把异步变回同步，浪费了异步的价值。若有 N 个并行查询，`Future.get()` 只能串行等待。

CompletableFuture 的核心洞察：**结果到来时通知我（回调），而不是我阻塞等它**。实现机制：每个 CompletableFuture 内部维护一个 Completion 链，上一阶段完成时自动触发下一阶段——这正是事件驱动的本质。

## 二、Completion 链与默认线程池

### 底层结构

```
CompletableFuture<T>
├── volatile Object result      // 计算结果（null=未完成，AltResult=异常）
├── volatile Completion stack   // Completion 链的栈顶（单向链表）
└── 使用 ForkJoinPool.commonPool()（默认）或自定义 Executor

Completion（内部类，代表一个后续阶段）
├── Executor executor          // 执行该阶段的线程池
├── CompletableFuture dep      // 该阶段完成后产生的 CF
├── BiConsumer/Function fn     // 该阶段要执行的操作
└── Completion next            // 链表下一节点
```

### 事件驱动流程

```
阶段 A 完成
  → CAS 设置 result（volatile 写，保证对后续线程可见）
  → 遍历 stack 中的 Completion 列表
  → 每个 Completion 检查自身依赖是否就绪
  → 就绪则提交 fn 到 executor 执行
  → fn 完成后更新阶段 B 的 result
  → 触发阶段 B 的 Completion 列表……
```

**result 字段 volatile 的意义**：`CompletableFuture.result` 的 volatile 写-读构成 happens-before，保证阶段 A 的所有写操作对阶段 B 可见（参见 [[概念-JMM]]）。

### ForkJoinPool 默认线程池

```java
// 默认使用
ForkJoinPool.commonPool()  // 并行度 = CPU核心数 - 1

// 方法后缀 Async 可指定线程池：
cf.thenApplyAsync(fn, customExecutor)
```

## 三、异步编排 API 分类

### 创建

```java
// 有返回值的异步任务
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> fetchData());

// 无返回值
CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> sendEmail());

// 已完成（用于测试或直接包装同步值）
CompletableFuture<String> cf3 = CompletableFuture.completedFuture("value");

// 手动完成（供外部触发）
CompletableFuture<String> cf4 = new CompletableFuture<>();
cf4.complete("result");     // 手动触发完成
cf4.completeExceptionally(new Exception()); // 手动触发异常
```

### 串行编排（前一阶段完成后触发下一阶段）

```java
// thenApply：同线程执行转换（T → U）
cf.thenApply(r -> r.toUpperCase());

// thenApplyAsync：异步线程执行转换
cf.thenApplyAsync(r -> r.toUpperCase());   // ForkJoinPool
cf.thenApplyAsync(r -> r.toUpperCase(), myPool);  // 自定义线程池

// thenCompose：扁平化嵌套 CF（类似 flatMap，避免 CF<CF<T>>）
cf.thenCompose(r -> anotherCF(r));

// thenAccept：消费结果，无返回值
cf.thenAccept(r -> save(r));

// thenRun：忽略结果，执行后续动作
cf.thenRun(() -> log("done"));
```

### 并行聚合

```java
// 全部完成后继续（AND 语义）
CompletableFuture.allOf(cf1, cf2, cf3)
    .thenRun(() -> {
        String r1 = cf1.join();   // 此时已完成，join() 不阻塞
        String r2 = cf2.join();
    });

// 任意一个完成后继续（OR 语义）
CompletableFuture.anyOf(cf1, cf2, cf3)
    .thenAccept(r -> handleFirst(r));

// 两个都完成后合并（BiFunction）
cf1.thenCombine(cf2, (r1, r2) -> merge(r1, r2));

// 两个任意一个完成（applyToEither）
cf1.applyToEither(cf2, r -> transform(r));
```

### 异常处理

```java
// 异常时返回默认值（正常时不触发）
cf.exceptionally(ex -> defaultValue);

// 正常 + 异常均处理（可修改结果）
cf.handle((result, ex) -> ex != null ? fallback(ex) : result);

// 回调但不改变结果（类似 finally，常用于清理/日志）
cf.whenComplete((result, ex) -> {
    if (ex != null) log.error("failed", ex);
});
```

## 四、任务聚合与超时场景

### 并行查询多服务聚合

```java
// 三个查询并行执行，总耗时 ≈ max(t1, t2, t3)，而非 t1+t2+t3
CompletableFuture<User>  userCF  = CompletableFuture.supplyAsync(() -> userService.get(id), pool);
CompletableFuture<Order> orderCF = CompletableFuture.supplyAsync(() -> orderService.get(id), pool);
CompletableFuture<Score> scoreCF = CompletableFuture.supplyAsync(() -> scoreService.get(id), pool);

CompletableFuture.allOf(userCF, orderCF, scoreCF).join();  // 等所有完成
UserDetail detail = new UserDetail(userCF.join(), orderCF.join(), scoreCF.join());
```

### 批量异步处理

```java
List<CompletableFuture<Result>> futures = items.stream()
    .map(item -> CompletableFuture.supplyAsync(() -> process(item), pool))
    .collect(toList());

List<Result> results = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).collect(toList()))
    .join();
```

### 超时控制（Java 9+）

```java
// 超时后抛 TimeoutException
cf.orTimeout(3, TimeUnit.SECONDS);

// 超时后返回默认值（不抛异常）
cf.completeOnTimeout("default", 3, TimeUnit.SECONDS);
```

### 链式串行执行（带异常回退）

```java
CompletableFuture.supplyAsync(() -> fetchFromDB(id))
    .thenApply(data -> enrich(data))
    .thenApply(data -> transform(data))
    .exceptionally(ex -> {
        log.error("pipeline failed", ex);
        return fallbackData(id);    // 降级数据
    })
    .thenAccept(result -> response.write(result));
```

## 五、CompletableFuture 语义对比

### thenApply vs thenApplyAsync

| 维度 | `thenApply` | `thenApplyAsync` |
|------|------------|----------------|
| 执行线程 | 完成上一阶段的线程（可能是主线程或工作线程）| 提交到线程池（默认 ForkJoinPool）|
| 适用 | 轻量级转换（几纳秒到几微秒）| 重计算或有阻塞的操作 |
| 风险 | 若上游是 `commonPool` 线程，回调也占用 commonPool 线程 | 需要额外线程调度开销 |

**经验**：IO 密集回调（DB查询、HTTP）必须用 `thenApplyAsync` + 自定义线程池；否则 `commonPool` 很快耗尽。

### CompletableFuture vs CountDownLatch

| 维度 | CompletableFuture.allOf | CountDownLatch |
|------|------------------------|---------------|
| 链式编排 | ✓（thenApply、thenCompose 等）| ✗（只能等待）|
| 异常处理 | ✓（exceptionally/handle）| ✗（异常需手动传播）|
| 返回结果 | ✓（各 CF.join() 获取）| ✗（需额外容器收集结果）|
| 可重用 | ✗（一次性）| ✗（一次性）|
| 代码简洁性 | 高 | 较低 |

## 六、异步编排生产风险

| 风险 | 场景 | 解决方案 |
|------|------|---------|
| **ForkJoinPool IO 阻塞** | DB/HTTP 请求使用默认 commonPool | 为 IO 任务指定独立线程池（IO 线程池）|
| **异常吞噬** | `thenApply` 抛异常，链末无 `exceptionally` | 链末必须加 `exceptionally` 或 `handle` |
| **allOf 不收集结果** | `allOf().thenApply(v -> ...)` 中 v 是 Void | 单独调用每个 CF.join() 收集结果 |
| **join() 阻塞调用线程** | 在 Tomcat/Netty 线程中 join()，耗尽工作线程 | join() 不应在 IO 线程上调用 |
| **超时不取消** | `orTimeout` 触发后任务仍在执行 | 配合 `cancel()` 或自行检查 interrupt |
| **链过长难调试** | 十几个 thenApply 链，异常 stacktrace 难定位 | 适当拆分，关键节点加日志 |

### join() vs get()

| 维度 | `join()` | `get()` |
|------|---------|--------|
| 异常类型 | `CompletionException`（unchecked）| `ExecutionException`（checked）|
| 中断处理 | 不响应中断 | 响应中断（抛 InterruptedException）|
| 链式使用 | 推荐（无需 try-catch）| 不推荐（强迫写 try-catch）|

## 七、CompletableFuture 的依赖关系

- 依赖 [[机制-线程池]]：默认使用 `ForkJoinPool.commonPool()`；IO 密集场景必须自定义独立线程池（`supplyAsync(fn, executor)`）
- 依赖 [[概念-JMM]]：`result` 字段是 volatile，阶段间结果传递满足 happens-before，阶段 B 能看到阶段 A 的所有写操作
- 相关 [[机制-AQS]]：`CountDownLatch` 可实现类似 `allOf` 的等待，但缺乏链式编排；CompletableFuture 是更高层的异步抽象

## 八、CompletableFuture 适用边界

**适合 CompletableFuture**：
- 微服务中并行查询多个下游服务并聚合结果
- 批量任务的并行化处理（不要求严格顺序）
- 需要超时控制 + 异常回退的异步操作
- 多步骤异步流水线（fetch → transform → save）

**不适合 CompletableFuture**：
- **响应式流控（背压）**：数据源快于消费者时，无法背压 → 用 Project Reactor（WebFlux）或 RxJava
- **简单独立异步任务**：`ExecutorService.submit()` 更简洁
- **JDK 21+ 虚拟线程场景**：虚拟线程让阻塞式 IO 成本接近零，传统线程池 + CF 的复杂度可被简化为"直接阻塞等待"
