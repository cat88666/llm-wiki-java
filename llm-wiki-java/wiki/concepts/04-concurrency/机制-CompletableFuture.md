---
type: concept
status: active
name: "CompletableFuture"
layer: L3
aliases: ["CompletableFuture", "异步编程", "ForkJoinPool", "Completion链", "thenApply", "thenCompose", "allOf", "anyOf"]
tags: ["#concurrency"]
related:
  - "[[机制-线程池]]"
  - "[[概念-JMM]]"
  - "[[机制-AQS]]"
sources:
  - "../../../raw/note/Hollis/Java并发/✅CompletableFuture的底层是如何实现的？.md"
created: 2026-05-08
updated: 2026-05-08
lint_notes: ""
---

# CompletableFuture

> CompletableFuture 是 Java 8 引入的异步编程模型：以**链式 Completion 阶段 + 事件驱动**替代阻塞式 Future.get()，允许任务编排（串行/并行/聚合）且不阻塞调用线程。

## 第一性原理

Java 5 引入 `Future`，可以提交异步任务但只能通过 `get()` 阻塞等待结果——本质上还是把异步变回同步，浪费了异步的价值。

`CompletableFuture` 的核心洞察：**结果到来时通知我，而不是我一直等它**。实现机制是：每个 CompletableFuture 内部维护一个 Completion 链，当上一阶段计算完成时，自动触发下一阶段——这正是事件驱动的本质。

## 核心机制

### 底层结构

```
CompletableFuture<T>
  ├── result: Object         // 计算结果（完成后赋值）
  ├── stack: Completion      // Completion 链的栈顶（单向链表）
  └── ForkJoinPool（默认线程池）

Completion（内部类，代表一个后续阶段）
  ├── fn: Function           // 该阶段要执行的操作
  ├── dep: CompletableFuture // 该阶段完成后产生的 CF
  └── next: Completion       // 链表下一节点
```

**事件驱动流程**：
```
阶段A完成 → 设置 result → 触发 stack 中的 Completion
  → Completion 执行 fn → 完成阶段B → 触发阶段B 的 Completion → ...
```

**线程池**：默认使用 `ForkJoinPool.commonPool()`（并行度 = CPU核心数-1）；可通过方法后缀 `Async(Executor e)` 指定线程池。

### 常用 API 分类

**创建**：
```java
CompletableFuture.supplyAsync(() -> compute())  // 有返回值
CompletableFuture.runAsync(() -> doWork())       // 无返回值
CompletableFuture.completedFuture(value)         // 已完成的 CF
```

**串行编排**（前一阶段完成后触发下一阶段）：
```java
cf.thenApply(r -> transform(r))     // 同步转换（同线程）
cf.thenApplyAsync(r -> transform(r)) // 异步转换（ForkJoinPool 线程）
cf.thenCompose(r -> anotherCF(r))   // 扁平化嵌套 CF（类似 flatMap）
cf.thenAccept(r -> consume(r))       // 消费结果，无返回值
cf.thenRun(() -> afterDone())        // 忽略结果，执行动作
```

**并行聚合**：
```java
// 全部完成后继续（AND 语义）
CompletableFuture.allOf(cf1, cf2, cf3).thenRun(() -> allDone())

// 任意一个完成后继续（OR 语义）
CompletableFuture.anyOf(cf1, cf2, cf3).thenAccept(r -> firstResult(r))

// 两个 CF 都完成后组合结果
cf1.thenCombine(cf2, (r1, r2) -> merge(r1, r2))
```

**异常处理**：
```java
cf.exceptionally(ex -> defaultValue)           // 异常时返回默认值
cf.handle((result, ex) -> ex != null ? fallback : result)  // 正常+异常均处理
cf.whenComplete((result, ex) -> log())          // 回调但不改变结果
```

### 典型应用场景

**场景1：并行查询多个服务后聚合**
```java
CompletableFuture<User> userCF = CompletableFuture.supplyAsync(() -> userService.get(id));
CompletableFuture<Order> orderCF = CompletableFuture.supplyAsync(() -> orderService.get(id));

CompletableFuture.allOf(userCF, orderCF).thenRun(() -> {
    User user = userCF.join();
    Order order = orderCF.join();
    assemble(user, order);
});
```
效果：两个查询**并行**执行，总耗时 ≈ max(查询A, 查询B)，而非 A+B。

**场景2：批量异步处理**
```java
List<CompletableFuture<Result>> futures = items.stream()
    .map(item -> CompletableFuture.supplyAsync(() -> process(item)))
    .collect(toList());

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenRun(() -> futures.stream().map(CompletableFuture::join).forEach(result::add));
```

**场景3：超时控制（Java 9+）**
```java
cf.orTimeout(3, TimeUnit.SECONDS)              // 超时后 TimeoutException
cf.completeOnTimeout(defaultVal, 3, TimeUnit.SECONDS)  // 超时返回默认值
```

## 关键权衡

1. **thenApply vs thenApplyAsync**：`thenApply` 在完成上一阶段的线程中执行（可能是主线程），`thenApplyAsync` 提交到线程池；若回调逻辑轻量用 `thenApply`，重计算用 `thenApplyAsync` 避免阻塞上游线程
2. **ForkJoinPool 默认线程池的陷阱**：`commonPool` 的线程数 = CPU-1，I/O 密集型任务（DB查询、HTTP请求）会导致线程耗尽；**必须为 I/O 任务指定独立线程池**
3. **join() vs get()**：`join()` 抛 `CompletionException`（unchecked），`get()` 抛 `ExecutionException`（checked）；链式调用中推荐 `join()`
4. **异常吞噬问题**：`thenApply` 中的异常会被包装传递到链末，若不处理则静默丢失；务必在链末加 `exceptionally` 或 `handle`

## 与其他概念的关系

- 依赖 [[机制-线程池]]：默认使用 `ForkJoinPool.commonPool()`，I/O 密集场景必须自定义线程池，否则线程池满后任务排队
- 关联 [[概念-JMM]]：Completion 链的结果传递涉及跨线程可见性，CompletableFuture 内部通过 `volatile` 的 `result` 字段保证 happens-before
- 相关 [[机制-AQS]]：`CountDownLatch` 可实现类似 `allOf` 的等待语义，但缺乏链式编排能力；CompletableFuture 是更高层的抽象

## 应用边界

**适合 CompletableFuture**：微服务中需并行查询多个下游服务后聚合结果；批量异步处理（不需要顺序保证）；需要超时控制和异常回退的异步操作。

**不适合 CompletableFuture**：需要响应式流控（背压）→ 用 Project Reactor/RxJava；极高并发的 I/O 场景 → 考虑虚拟线程（JDK 21 Project Loom）；简单的独立异步任务 → `ExecutorService.submit()` 更简洁。
