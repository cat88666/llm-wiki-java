---
type: concept
status: active
name: "Java异常体系"
layer: L1
aliases: ["Exception", "异常", "Checked Exception", "RuntimeException", "Error", "try-catch-finally"]
related:
  - "[[机制-Java序列化]]"
  - "[[概念-OOP特征]]"
sources:
  - "../../../raw/note/Hollis/Java基础/✅Java中异常分哪两类，有什么区别？.md"
  - "../../../raw/note/Hollis/Java基础/✅final、finally、finalize有什么区别.md"
  - "../../../raw/note/Hollis/Java基础/✅finally中代码一定会执行吗？.md"
  - "../../../raw/note/Hollis/Java基础/✅为什么不建议使用异常控制业务流程.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# Java 异常体系

> Java 通过类型化的异常继承树，将运行失败分为"编译期必须处理的可预期失败"和"运行期暴露的代码缺陷"两类，强制调用方在编译期明确应对策略。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 错误码缺陷、编译期失败感知 |
| [二、核心机制](#二核心机制) | Throwable 继承树、Checked vs Unchecked、finally 规则 |
| [三、Java 核心使用](#三java-核心使用) | 关键字语义、自定义异常、全局异常处理 |
| [四、核心使用原则](#四核心使用原则) | 异常选型、catch 精度、业务异常设计 |
| [五、综合对比](#五综合对比) | Checked vs Unchecked vs Error 全维度对比 |
| [六、生产风险](#六生产风险) | 异常控流性能、finally 陷阱、异常吞没 |
| [七、与其他概念的关系](#七与其他概念的关系) | 序列化、JVM、Spring |
| [八、应用边界](#八应用边界) | 适用场景与反模式 |

## 一、第一性原理

不区分异常类型时，所有错误要么显式检查（C 风格错误码，代码充斥 if-check），要么完全不处理（错误静默失败）。Java 异常体系的根本目的：**让调用方在编译期就知道哪些失败场景必须明确应对，哪些属于代码 Bug 应在开发期消灭**。

## 二、核心机制

### 2.1 Throwable 继承树

```
Throwable
├── Error                ← JVM/系统级，程序无法恢复
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── ...
└── Exception
    ├── IOException      ← Checked：编译器强制处理
    ├── SQLException
    ├── ...
    └── RuntimeException ← Unchecked：代码缺陷，运行期暴露
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── ClassCastException
        ├── IndexOutOfBoundsException
        └── ...
```

### 2.2 Checked vs Unchecked 核心区别

| 维度 | Checked Exception | Unchecked (RuntimeException) | Error |
|------|-------------------|------------------------------|-------|
| 编译期 | 必须 try-catch 或 throws 声明 | 无需声明 | 无需声明 |
| 语义 | 可预期的外部失败 | 代码逻辑错误 | JVM/系统不可恢复 |
| 调用方责任 | 必须处理（重试/降级/上抛） | 自行决定 | 不应捕获 |
| 典型例子 | IOException、SQLException | NPE、IAE、ClassCastException | OOM、SOF |

### 2.3 finally 执行规则

- **正常情况**：无论是否有异常，finally 块都会执行
- **不执行的例外**：`System.exit()` 调用、JVM 崩溃、线程被 kill
- **finally-return 覆盖**：finally 中的 return 会覆盖 try/catch 中的 return 值

**面试陷阱**：finally 中不要写 return，否则 try 中的返回值被吞没且无编译警告。

## 三、Java 核心使用

### 3.1 五个关键字

| 关键字 | 作用 |
|--------|------|
| `try` | 包裹可能抛异常的代码块 |
| `catch` | 捕获并处理特定异常类型 |
| `finally` | 无论是否异常都执行的清理逻辑 |
| `throw` | 方法内抛出一个异常实例 |
| `throws` | 方法签名声明可能抛出的异常类型 |

### 3.2 try-with-resources（JDK 7+）

```java
try (InputStream is = new FileInputStream("f.txt")) {
    // 使用资源
}  // 自动调用 is.close()，无需手写 finally
```

实现 `AutoCloseable` 接口的对象均可使用，替代手写 finally 关闭资源。

### 3.3 自定义业务异常

```java
// 继承 RuntimeException，避免调用方被迫 try-catch
public class BizException extends RuntimeException {
    private final String errorCode;
    public BizException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}
```

配合 Spring `@ControllerAdvice` + `@ExceptionHandler` 全局统一翻译为 HTTP 响应码。

## 四、核心使用原则

| 原则 | 说明 |
|------|------|
| 业务异常继承 RuntimeException | 避免调用方被迫 try-catch，全局处理器统一处理 |
| catch 要精确 | 禁止 `catch(Exception e)` 吞掉所有异常，隐藏真实问题 |
| 不要捕获 Error | OOM/SOF 后系统已不稳定，捕获无意义 |
| 不用异常控制业务流程 | 异常创建需 `fillInStackTrace()`，开销远大于 if-else |
| 异常信息要完整 | `throw new XxxException("orderId=" + id, cause)` 保留上下文和原始异常链 |
| finally 不要 return | finally 的 return 会覆盖 try 的返回值 |

## 五、综合对比

### 5.1 final / finally / finalize

| 关键字 | 类别 | 作用 |
|--------|------|------|
| `final` | 修饰符 | 类不可继承、方法不可重写、变量不可重赋值 |
| `finally` | 异常处理 | try-catch 后必执行的清理块 |
| `finalize` | Object 方法 | GC 前回调（JDK 9+ 已废弃，不可依赖） |

### 5.2 异常处理策略选型

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 文件/网络 IO | Checked Exception + 重试/降级 | 外部失败可预期，调用方有能力处理 |
| 参数校验失败 | IllegalArgumentException（Unchecked） | 代码 Bug，应在开发期修复 |
| 业务规则违反（余额不足） | 自定义 RuntimeException | 全局处理器统一翻译 |
| 正常流程分支 | if-else / Optional | 异常不是流程控制工具 |

## 六、生产风险

| 风险 | 触发场景 | 后果 | 防御 |
|------|---------|------|------|
| 异常控流性能差 | 用 throw/catch 替代 if-else | `fillInStackTrace()` 开销大，比条件判断慢数百倍 | 正常分支用 if/Optional |
| 异常被吞没 | `catch(Exception e) {}` 空块 | 问题隐藏，排查困难 | 至少 log.error，最好重抛 |
| finally-return 覆盖 | finally 中写 return | try 中返回值被静默覆盖 | finally 只做清理，不要 return |
| 异常链丢失 | `throw new XxxException(msg)` 不传 cause | 原始根因丢失 | 构造时传入 cause |
| catch 范围过大 | `catch(Exception e)` | 意外捕获 NPE 等 Bug 类异常 | 精确 catch 具体异常类型 |

## 七、与其他概念的关系

- [[机制-Java序列化]]：序列化相关方法抛 `IOException`（Checked），提醒调用方处理 IO 失败
- JVM 层：`StackOverflowError`、`OutOfMemoryError` 是 Error，由 JVM 抛出，与 GC 和内存结构直接相关
- Spring 框架：`@ExceptionHandler` + `@ControllerAdvice` 实现全局异常处理，将业务异常翻译为统一响应格式
- [[概念-OOP特征]]：异常体系本身是多态的典型应用 -- catch 父类型可捕获所有子类型

## 八、应用边界

| 场景 | 适用 | 不适用 |
|------|------|--------|
| Checked Exception | API 边界处，调用方确实能恢复（重试、降级） | 调用方无法处理只能上抛的场景（退化为模板代码） |
| RuntimeException | 业务规则违反、参数校验失败 | 需要调用方强制感知的外部失败 |
| 异常机制 | 异常/错误场景的处理 | 正常业务流程跳转、性能热路径 |
