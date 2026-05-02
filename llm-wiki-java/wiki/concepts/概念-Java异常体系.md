---
type: concept
status: active
name: "Java异常体系"
layer: L1
aliases: ["Exception", "异常", "Checked Exception", "RuntimeException"]
related:
  - "[[机制-Java序列化]]"
sources:
  - "../../raw/note/Hollis/Java基础/✅Java中异常分哪两类，有什么区别？.md"
  - "../../raw/note/Hollis/Java基础/✅final、finally、finalize有什么区别.md"
  - "../../raw/note/Hollis/Java基础/✅finally中代码一定会执行吗？.md"
  - "../../raw/note/Hollis/Java基础/✅为什么不建议使用异常控制业务流程.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# Java异常体系

> Java 把运行失败分成"必须处理"和"代码 Bug"两类，强制区分让调用方知道哪些意外是可预期的。

## 第一性原理

不做区分时，所有错误都必须显式处理（像 C 的错误码），导致代码充斥 if-null-check；或者完全不处理，让错误静默失败。Java 的异常体系解决：**让调用方在编译期就知道哪些失败场景需要明确应对**。

## 核心机制

### 继承树

```
Throwable
├── Error          ← JVM/系统级，程序无法恢复（OOM、StackOverflow）
└── Exception
    ├── IOException
    ├── SQLException
    ├── ...           ← Checked Exception：编译器强制 try-catch 或 throws
    └── RuntimeException
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── ClassCastException
        └── ...       ← Unchecked Exception：代码 Bug，运行期才暴露
```

### 两类核心区别

| | Checked Exception | Unchecked / RuntimeException |
|--|-------------------|------------------------------|
| **编译期** | 必须 try-catch 或 throws | 无需声明 |
| **语义** | 可预期的外部失败（文件不存在、网络超时）| 代码逻辑错误（空指针、越界）|
| **调用方** | 必须明确处理 | 自行决定是否 catch |
| **典型例子** | `FileNotFoundException`, `SQLException` | `NPE`, `IOOBE`, `IAE` |

### finally 执行规则

- **正常情况**：不管是否有异常，finally 块都执行
- **例外**：`System.exit()` 被调用；JVM 崩溃；线程被强制 kill
- **try-return + finally-return**：finally 的 return 会覆盖 try 的 return（强烈不推荐在 finally 中 return）

### 异常关键字

```
try      → 可能抛出异常的代码块
catch    → 捕获并处理特定异常
finally  → 总是执行的清理逻辑（关闭资源）
throw    → 在方法内抛出一个具体异常实例
throws   → 在方法签名声明可能抛出的异常类型
```

## 关键权衡

1. **不要用异常控制业务流程**：异常对象的创建需要填充栈轨迹（`fillInStackTrace`），开销大；逻辑用 if-else 判断，性能和可读性均优于 throw/catch
2. **自定义业务异常应继承 RuntimeException**：避免调用方被迫 try-catch 业务异常，用全局异常处理器统一处理
3. **catch 要精确**：不要 `catch(Exception e)` 吞掉所有异常，会隐藏真正的问题
4. **Error 不要捕获**：捕获 OOM 无意义，系统已经不稳定

## 与其他概念的关系

- [[机制-Java序列化]]：`Serializable` 接口相关方法抛 `IOException`（Checked），提醒调用方处理 IO 失败
- L2 JVM 层：`StackOverflowError`、`OutOfMemoryError` 是 Error，由 JVM 抛出，与 GC 和内存结构直接相关
- L6 Spring：`@ExceptionHandler` 和 `@ControllerAdvice` 是业务层统一异常处理的实现

## 应用边界

**适合声明 Checked Exception**：API 边界处，调用方确实需要知道并能从中恢复（如重试、降级）。

**适合用 RuntimeException**：业务规则违反（余额不足、权限不足）、参数校验失败；配合全局异常处理器统一翻译为 HTTP 响应码。

**不适合用异常**：用来做正常流程跳转（用 if/Optional 替代）；在性能热路径中 throw/catch。
