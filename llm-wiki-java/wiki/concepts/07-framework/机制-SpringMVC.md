---
type: concept
status: active
name: "SpringMVC"
layer: L6
aliases: ["SpringMVC", "DispatcherServlet", "HandlerMapping", "HandlerAdapter", "MappingRegistry", "请求路由", "拦截器", "ViewResolver", "ExceptionResolver"]
related:
  - "[[机制-Spring]]"
  - "[[机制-SpringBoot]]"
---

# SpringMVC

> SpringMVC 以 DispatcherServlet 为核心前端控制器，通过 HandlerMapping 路由、HandlerAdapter 调用、Interceptor 前后置处理、ViewResolver 渲染四级流水线，将 HTTP 请求分发到对应 Controller 方法并返回响应。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | URL→Handler 路由、调用、增强三关注点分离 |
| [二、核心机制](#二核心机制) | 继承结构、路由注册、请求处理流水线、异常处理 |
| [三、综合对比](#三综合对比) | 设计模式全景、Interceptor vs AOP |
| [四、关键权衡](#四关键权衡) | HandlerAdapter 必要性、参数解析器链、@ResponseBody 互斥 |
| [五、与其他概念的关系](#五与其他概念的关系) | Spring IoC、SpringBoot 自动装配 |
| [六、应用边界](#六应用边界) | 全局异常处理、自定义参数解析、Interceptor 边界 |

## 一、第一性原理

Web 框架的核心问题：**不同的 URL 请求如何准确找到对应的处理逻辑，并以统一方式前后置增强？** 朴素解法是一个大 Map（URL → Handler），但现实需求更复杂——同一 URL 的 GET 和 POST 要路由不同方法；请求前需要鉴权；返回值可能是 JSON 也可能是模板视图。

SpringMVC 的答案：**将路由、调用、增强三个关注点分离**，分别由 HandlerMapping、HandlerAdapter、Interceptor/ViewResolver 负责，DispatcherServlet 作为协调者串联整条链路。

## 二、核心机制

### 2.1 继承结构

```
HttpServlet
  └── FrameworkServlet（doService 入口）
        └── DispatcherServlet（核心分发逻辑）
```

HTTP 请求从 Servlet 容器（Tomcat）进入 `HttpServlet.service()` → `FrameworkServlet.doService()` → `DispatcherServlet.doDispatch()`。

### 2.2 路由注册（启动期）

```
@RequestMapping 注解
  └── 启动时由 RequestMappingHandlerMapping 扫描
        ↓
  RequestMappingInfo（封装 URL/Method/Header 等条件）
    + HandlerMethod（封装目标 Method + 持有 Bean）
        ↓
  注册到 MappingRegistry（本质是多维度的 Map）
```

**RequestMappingInfo** 聚合多个 `RequestCondition`（URL条件、HTTP Method条件、Header条件、Params条件），这是**门面模式**（Facade）——用一个对象统一外部复杂条件的比较。

匹配时通过 `AbstractHandlerMethodMapping#lookupHandlerMethod` 遍历 MappingRegistry，利用各 Condition 的 `compare()` 方法选出最优 HandlerMethod，这是**组合模式**（Composite）。

### 2.3 请求处理流水线（运行期）

```
HTTP Request
  ↓
1. HandlerMapping.getHandler()
   → HandlerExecutionChain（HandlerMethod + Interceptor列表）
   → HandlerAdapter（根据 Handler 类型适配）
  ↓
2. Interceptor.preHandle()（责任链，任一返回false则短路）
  ↓
3. HandlerAdapter.handle()
   ├── HandlerMethodArgumentResolver（参数解析，@RequestBody → 反序列化）
   ├── 反射调用 HandlerMethod.invoke()
   └── HandlerMethodReturnValueHandler（返回值处理，@ResponseBody → 序列化）
  ↓
4. Interceptor.postHandle()（逆序执行）
  ↓
5. ViewResolver.resolveViewName() → View.render(ModelAndView → Response)
  ↓
6. Interceptor.afterCompletion()（逆序，无论异常均执行）
```

### 2.4 异常处理（ExceptionResolver）

```
业务方法抛出异常
  ↓
HandlerExceptionResolver 责任链（@ExceptionHandler → ResponseStatus → DefaultHandler）
  ↓
ExceptionHandlerMethodResolver（启动时扫描 @ExceptionHandler，key=异常类型, value=处理方法）
  ↓
参数解析 + 反射调用（与正常 Handler 调用路径几乎相同）→ 响应
```

## 三、综合对比

### 3.1 设计模式全景

| 模式 | 体现 |
|------|------|
| **前端控制器** | DispatcherServlet 统一接收所有请求 |
| **门面（Facade）** | RequestMappingInfo 聚合多个 Condition |
| **适配器（Adapter）** | HandlerAdapter 统一调用不同类型 Handler（Method/Servlet/...）|
| **组合（Composite）** | RequestCondition 嵌套组合多种匹配条件 |
| **责任链（Chain of Responsibility）** | Interceptor 链、ExceptionResolver 链 |
| **策略（Strategy）** | HandlerMethodArgumentResolver / ReturnValueHandler 多策略按需选择 |

### 3.2 Interceptor vs AOP

| 维度 | Interceptor | AOP |
|------|-------------|-----|
| 工作层 | DispatcherServlet 层，只拦截 Controller 请求 | Bean 方法层，可拦截任意 Bean 调用 |
| 典型场景 | 登录鉴权、请求日志、跨域处理 | 事务注解、缓存注解、方法级权限 |
| 执行顺序 | preHandle → Controller → postHandle → afterCompletion | 包裹在 Bean 方法调用上 |

## 四、关键权衡

| 权衡点 | 说明 |
|--------|------|
| HandlerAdapter 的必要性 | Controller 方法可以是 @RequestMapping 方法、实现 Controller 接口的 Bean、原生 HttpServlet——HandlerAdapter 将这三种接口不同的 Handler 统一适配，DispatcherServlet 无需关心具体类型 |
| 参数解析器链的代价 | `HandlerMethodArgumentResolver` 列表按顺序匹配，参数复杂时匹配开销增大；框架内置 26+ 个解析器，自定义时需注意注册顺序 |
| @ResponseBody 与视图解析互斥 | 方法标注 `@ResponseBody` 时 `ReturnValueHandler` 直接写 Response，不走 ViewResolver 渲染流程 |

## 五、与其他概念的关系

- 依赖 [[机制-Spring]]：DispatcherServlet 通过 `WebApplicationContext` 获取 HandlerMapping/Adapter Bean；Controller 由 IoC 管理，`@Transactional`、`@Cacheable` 等注解依赖 AOP 代理生效
- 由 [[机制-SpringBoot]] 装配：`DispatcherServletAutoConfiguration` 自动注册 DispatcherServlet；`WebMvcAutoConfiguration` 自动注册 HandlerMapping/Adapter/ViewResolver

## 六、应用边界

**全局异常处理**：实现 `@ControllerAdvice` + `@ExceptionHandler`，统一处理 Controller 层异常，不要在每个 Controller 里单独 try-catch。

**自定义参数解析**：实现 `HandlerMethodArgumentResolver` 注册到 `WebMvcConfigurer#addArgumentResolvers`，注意不要破坏内置解析器顺序。

**不适合用 Interceptor 的场景**：需要在 Service 层做横切逻辑（如事务、缓存）——用 AOP；需要精确控制 Bean 初始化顺序——用 BeanPostProcessor。
