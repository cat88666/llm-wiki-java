---
type: concept
status: active
name: "SpringMVC"
layer: L6
aliases: ["SpringMVC", "DispatcherServlet", "HandlerMapping", "HandlerAdapter", "MappingRegistry", "请求路由", "拦截器", "ViewResolver", "ExceptionResolver", "HandlerMethodArgumentResolver"]
related:
  - "[[机制-Spring]]"
  - "[[机制-SpringBoot]]"
  - "[[机制-动态代理]]"
sources:
  - "../../../raw/note/Hollis/Spring/✅SpringMVC是如何将不同的Request路由到不同Controller中的？.md"
  - "../../../raw/note/Hollis/Spring/✅SpringMVC中如何实现流式输出.md"
updated: 2026-05-18
---

# SpringMVC

> SpringMVC 用 `DispatcherServlet` 统一接收 HTTP 请求，再通过 HandlerMapping 路由、HandlerAdapter 调用、Interceptor 增强、ExceptionResolver 兜底，把 Web 请求处理拆成可扩展流水线。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、SpringMVC 解决的问题](#sec-2) | URL 到 Controller 方法的统一分发 |
| [二、启动期路由注册](#sec-3) | `RequestMappingInfo`、`HandlerMethod`、`MappingRegistry` |
| [三、运行期请求流水线](#sec-4) | `doDispatch` 主链路 |
| [四、参数解析与返回值处理](#sec-5) | Resolver/Handler 策略链 |
| [五、拦截器、异常与流式输出](#sec-6) | Interceptor、ExceptionResolver、ResponseBodyEmitter |
| [六、设计模式与对比](#sec-7) | 前端控制器、适配器、责任链、AOP 对比 |
| [七、生产风险与排查](#sec-8) | 路由冲突、参数解析失败、异常未统一 |
| [八、关系与边界](#sec-9) | 与 Spring、Boot、AOP 的关系 |
| [九、面试速答口径](#sec-10) | 高频问题答案 |

<a id="sec-2"></a>
## 一、SpringMVC 解决的问题

Web 框架要解决三件事：

1. HTTP 请求如何匹配到一个业务方法。
2. 请求参数如何转换成 Java 方法参数。
3. 返回值、异常和前后置逻辑如何统一处理。

SpringMVC 的答案是把分发、调用、增强拆开，`DispatcherServlet` 只负责协调。

<a id="sec-3"></a>
## 二、启动期路由注册

```text
扫描 @Controller / @RequestMapping
  -> 生成 RequestMappingInfo（URL、Method、Header、Param 条件）
  -> 生成 HandlerMethod（目标 Bean + Method）
  -> 注册到 MappingRegistry
```

`RequestMappingInfo` 聚合多种匹配条件，匹配请求时再按条件比较选出最优 `HandlerMethod`。

<a id="sec-4"></a>
## 三、运行期请求流水线

```text
HTTP Request
  -> DispatcherServlet#doDispatch
  -> HandlerMapping#getHandler
  -> HandlerExecutionChain（Handler + Interceptors）
  -> HandlerAdapter#handle
  -> Controller 方法调用
  -> ReturnValueHandler 写响应或 ModelAndView
  -> ViewResolver 渲染视图（非 @ResponseBody 场景）
  -> afterCompletion 收尾
```

核心结论：`HandlerMapping` 负责找谁处理，`HandlerAdapter` 负责怎么调用。

<a id="sec-5"></a>
## 四、参数解析与返回值处理

| 扩展点 | 作用 | 示例 |
| --- | --- | --- |
| `HandlerMethodArgumentResolver` | 把请求内容转成方法参数 | `@RequestBody`、`@PathVariable` |
| `HandlerMethodReturnValueHandler` | 把方法返回值写入响应 | `@ResponseBody`、`ModelAndView` |
| `HttpMessageConverter` | Java 对象与 HTTP Body 转换 | JSON 序列化/反序列化 |

自定义登录用户参数、租户上下文、灰度标记时，优先用参数解析器，而不是在每个 Controller 重复解析。

<a id="sec-6"></a>
## 五、拦截器、异常与流式输出

### 5.1 Interceptor 执行顺序

```text
preHandle 正序
  -> Controller
postHandle 逆序
  -> View 渲染
afterCompletion 逆序
```

### 5.2 异常处理链

`HandlerExceptionResolver` 负责把 Controller 抛出的异常翻译成响应，常见实践是 `@ControllerAdvice + @ExceptionHandler`。

### 5.3 流式输出

SpringMVC 可以用 `StreamingResponseBody`、`ResponseBodyEmitter` 做流式响应，但传统 MVC 仍占用 Servlet 请求线程；高并发长连接场景要评估 WebFlux 或异步化方案。

<a id="sec-7"></a>
## 六、设计模式与对比

| 模式 | 体现 |
| --- | --- |
| 前端控制器 | `DispatcherServlet` |
| 适配器 | `HandlerAdapter` |
| 责任链 | Interceptor、ExceptionResolver |
| 策略 | ArgumentResolver、ReturnValueHandler |
| 门面 | `RequestMappingInfo` 聚合匹配条件 |

Interceptor vs AOP：

| 维度 | Interceptor | AOP |
| --- | --- | --- |
| 层级 | Web 请求层 | Bean 方法层 |
| 适合 | 登录、鉴权、请求日志 | 事务、缓存、方法权限 |
| 生效范围 | Controller 请求 | Spring Bean 方法调用 |

<a id="sec-8"></a>
## 七、生产风险与排查

| 风险 | 现象 | 排查点 |
| --- | --- | --- |
| 路由冲突 | 启动失败或请求命中不稳定 | URL、Method、Header 条件 |
| 参数绑定失败 | 400 / 类型转换异常 | 参数注解、Converter、JSON 字段 |
| 异常未统一 | 响应格式不一致 | `@ControllerAdvice` 是否扫描 |
| 拦截器误用 | 静态资源/健康检查被拦截 | path pattern 与排除规则 |
| 流式输出阻塞 | 线程被长期占用 | Servlet 线程池与连接时长 |

<a id="sec-9"></a>
## 八、关系与边界

- 依赖 [[机制-Spring]]：Controller、HandlerMapping、Adapter 都是容器 Bean。
- 由 [[机制-SpringBoot]] 自动装配：Boot 注册 `DispatcherServlet` 与 MVC 默认组件。
- 与 [[机制-动态代理]]：Controller 上的事务或缓存注解仍依赖 Spring AOP 代理。

边界：SpringMVC 是 Servlet 栈 Web 框架，不适合把所有 Service 层横切能力放进 Interceptor。

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| SpringMVC 核心流程 | `DispatcherServlet -> HandlerMapping -> HandlerAdapter -> Controller -> ReturnValueHandler/ViewResolver` |
| HandlerAdapter 为什么需要 | 不同 Handler 类型调用方式不同，Adapter 统一调用接口 |
| Interceptor 和 AOP 区别 | Web 请求层 vs Bean 方法层 |
| 全局异常怎么做 | `@ControllerAdvice + @ExceptionHandler` |
