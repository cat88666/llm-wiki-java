---
type: concept
status: active
name: "Java异常"
layer: L1
aliases: ["Exception", "异常", "Checked Exception", "RuntimeException", "Error", "try-catch-finally"]
related:
  - "[[机制-Java序列化]]"
  - "[[概念-OOP特征]]"
  - "[[概念-JMM]]"
sources:
  - "../../../raw/note/Hollis/Java基础/✅Java中异常分哪两类，有什么区别？.md"
updated: 2026-05-18
---

# Java 异常体系

> Java 异常体系通过类型化错误模型把故障分层：可恢复故障强制声明处理，不可恢复故障快速暴露并中断错误路径。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、异常模型的目标](#sec-2) | 从错误码到类型化失败治理 |
| [二、Throwable 分层机制](#sec-3) | Error/Checked/Unchecked 语义边界 |
| [三、关键语义与执行规则](#sec-4) | try-catch-finally、throws、异常链 |
| [四、业务异常设计实践](#sec-5) | 统一错误码、全局处理、降级策略 |
| [五、异常策略选择原则](#sec-6) | 何时 checked，何时 runtime |
| [六、对比与结论](#sec-7) | Checked vs Runtime vs Error |
| [七、生产风险与排查](#sec-8) | 吞异常、控流滥用、日志污染 |
| [八、关系与边界](#sec-9) | 与事务、并发、序列化关系 |
| [九、面试速答口径](#sec-10) | 高频问答 |

<a id="sec-2"></a>
## 一、异常模型的目标

异常机制的目标不是“打印错误”，而是让失败成为显式流程的一部分，且在调用边界可追踪、可治理。

<a id="sec-3"></a>
## 二、Throwable 分层机制

```text
Throwable
  -> Error
  -> Exception
     -> Checked Exception
     -> RuntimeException
```

| 类型 | 代表意义 | 处理策略 |
| --- | --- | --- |
| Error | JVM/系统级不可恢复 | 不捕获，快速失败 |
| Checked Exception | 外部可预期失败 | 显式处理或继续上抛 |
| RuntimeException | 编程缺陷或非法状态 | 修代码为主，必要时兜底 |

<a id="sec-4"></a>
## 三、关键语义与执行规则

1. `throw` 抛出异常实例，`throws` 声明方法可能抛出的异常。
2. finally 通常都会执行，但 `System.exit`/进程崩溃是例外。
3. finally 中 return 会覆盖 try/catch 的 return，属于高危反模式。
4. 异常链要保留 cause，避免根因丢失。

<a id="sec-5"></a>
## 四、业务异常设计实践

```java
public class BizException extends RuntimeException {
    private final String code;
    public BizException(String code, String msg, Throwable cause) {
        super(msg, cause);
        this.code = code;
    }
}
```

- 业务层通常使用 RuntimeException + 错误码。
- Web 层通过 `@ControllerAdvice` 统一翻译响应。
- 基础设施异常转换为领域异常再上抛。

<a id="sec-6"></a>
## 五、异常策略选择原则

| 场景 | 建议 |
| --- | --- |
| 外部系统可恢复故障（IO/网络） | checked 或包装后带重试语义 |
| 参数非法、状态不一致 | runtime，快速失败 |
| 系统不可恢复（OOM） | 不捕获 |
| 批处理任务局部失败 | 收集异常并继续，最终汇总 |

<a id="sec-7"></a>
## 六、对比与结论

| 维度 | Checked | Runtime | Error |
| --- | --- | --- | --- |
| 编译期约束 | 强制处理 | 不强制 | 不强制 |
| 可恢复性 | 通常可恢复 | 视场景 | 不可恢复 |
| 主要动作 | 重试/补偿/降级 | 修复缺陷 + 局部兜底 | 保护现场并重启 |

<a id="sec-8"></a>
## 七、生产风险与排查

| 风险 | 现象 | 处理 |
| --- | --- | --- |
| 吞异常 | 问题难定位 | 禁止空 catch，统一日志规范 |
| 异常控流 | CPU 抖动 | 改为条件分支 |
| 过度包装 | 堆栈冗长 | 保留根因，不重复包裹 |
| 事务误回滚 | 捕获后未继续抛出 | 明确事务边界和回滚规则 |

<a id="sec-9"></a>
## 八、关系与边界

- 与 [[机制-Java序列化]]：远程调用中的异常对象传输依赖序列化策略。
- 与 [[概念-OOP特征]]：异常是跨层契约的一部分。
- 与 [[概念-JMM]]：并发异常（可见性竞态）常表现为 runtime 异常。

边界：异常体系不替代监控告警与容量治理。

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| Checked 和 Runtime 本质差异 | 是否强制调用方在编译期处理 |
| 为什么不建议 `catch(Exception)` | 会吞掉语义边界，导致误处理 |
| finally 一定执行吗 | 通常会，`System.exit`/崩溃例外 |
