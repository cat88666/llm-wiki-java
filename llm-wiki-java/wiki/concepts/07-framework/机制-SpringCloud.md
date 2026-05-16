---
type: concept
status: active
name: "微服务与SpringCloud"
layer: L7
aliases: ["SpringCloud", "微服务", "Eureka", "Feign", "OpenFeign", "SpringCloudOpenFeign", "FeignClient", "Feign超时", "Hystrix", "Sentinel", "Gateway", "SpringCloudGateway", "Route", "Predicate", "GatewayFilter", "GlobalFilter", "WebFlux", "Reactor", "Ribbon", "LoadBalancer", "Nacos", "熔断器", "服务注册", "服务发现", "蓝绿部署", "灰度发布", "金丝雀发布", "ServiceMesh", "Service Mesh", "Istio", "CI/CD", "DevOps", "康威定律", "限流降级熔断", "循环依赖"]
related:
  - "[[机制-Dubbo]]"
  - "[[机制-Zookeeper]]"
  - "[[机制-动态代理]]"
  - "[[机制-Spring]]"
  - "[[机制-SpringBoot]]"
  - "[[主题-三高架构]]"
sources:
  - "../../../raw/note/Hollis/SpringCloud/✅什么是SpringCloud，有哪些组件？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅SpringCloud和Dubbo有什么区别？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅Eureka和Zookeeper有什么区别？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅Hystrix熔断器的工作原理是什么？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅Hystrix和Sentinel的区别是什么？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅OpenFeign 是如何实现负载均衡的？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅Feign 和 RestTemplate 有什么不同？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅LoadBalancer和Ribbon的区别是什么？为什么用他替代Ribbon？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅为什么需要SpringCloud Gateway，他起到了什么作用？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅Zuul、Gateway和Nginx有什么区别？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅Dubbo和Feign有什么区别？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅介绍一下 Hystrix 的隔离策略，你用哪个？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅什么是Eureka的自我保护模式？.md"
  - "../../../raw/note/Hollis/SpringCloud/✅在 Spring Cloud 中，服务间的通信有哪些方式？.md"
  - "../../../raw/note/Hollis/架构设计/✅微服务的拆分有哪些原则？.md"
  - "../../../raw/note/Hollis/微服务/✅灰度发布、蓝绿部署、金丝雀部署都是什么？.md"
  - "../../../raw/note/Hollis/微服务/✅听说过ServiceMesh吗？是什么？.md"
  - "../../../raw/note/Hollis/微服务/✅限流、降级、熔断有什么区别？.md"
  - "../../../raw/note/Hollis/微服务/✅什么是微服务的循环依赖？.md"
  - "../../../raw/note/Hollis/微服务/✅微服务中的CI CD了解吗？.md"
  - "../../../raw/note/Hollis/微服务/✅什么是DevOps？.md"
  - "../../../raw/note/Hollis/微服务/✅什么是康威定律？.md"
  - "../../../raw/note/Hollis/微服务/✅SOA和微服务之间的主要区别是什么？.md"
  - "../../../raw/note/Hollis/微服务/✅分布式和微服务的区别是什么？.md"
created: 2026-05-06
updated: 2026-05-16
lint_notes: ""
---

# 微服务与SpringCloud

> Spring Cloud 是基于 Spring Boot 的微服务开发工具集：通过一系列开箱即用的组件（注册中心、负载均衡、熔断器、网关、配置中心），解决分布式系统中服务注册发现、流量管控和容错等共性问题，核心理念是"专业的组件做专业的事"。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 服务发现、负载均衡、容错熔断 |
| [二、核心机制](#二核心机制) | 组件全景、注册中心、OpenFeign、熔断器、API 网关、Nacos |
| [三、OpenFeign 机制详解](#三openfeign-机制详解) | 声明式 HTTP、动态代理、服务发现、超时重试 |
| [四、Spring Cloud Gateway 机制详解](#四spring-cloud-gateway-机制详解) | Route/Predicate/Filter、执行流程、动态路由、网关边界 |
| [五、综合对比](#五综合对比) | SpringCloud vs Dubbo、Eureka vs ZK、Hystrix vs Sentinel、Gateway vs Nginx |
| [六、核心使用原则](#六核心使用原则) | 微服务拆分原则、限流降级熔断、循环依赖解法 |
| [七、使用案例](#七使用案例) | 部署策略、Service Mesh、CI/CD、DevOps |
| [八、关键权衡](#八关键权衡) | AP vs CP、客户端 vs 服务端 LB、Feign/Gateway 使用边界 |
| [九、与其他概念的关系](#九与其他概念的关系) | Dubbo、Zookeeper、Spring、SpringBoot |
| [十、应用边界](#十应用边界) | 适用与不适用场景 |

## 一、第一性原理

单体应用拆分为微服务后，原本的进程内调用变为跨网络调用，带来三个根本问题：**如何找到对方**（服务发现）、**如何选择对方**（负载均衡）、**如何在对方出问题时自我保护**（容错/熔断）。Spring Cloud 的每个组件都在解决其中一个或多个问题。

## 二、核心机制

### 2.1 组件全景（现代选型）

| 问题领域 | 老版本组件 | 现代推荐替代 |
|---------|-----------|------------|
| 服务注册/发现 | Eureka | **Nacos**（AP/CP 可切，含配置中心）|
| 负载均衡 | Ribbon（停维）| **Spring Cloud LoadBalancer** |
| 服务间调用 | RestTemplate | **OpenFeign**（声明式 HTTP）|
| 熔断/限流/降级 | Hystrix（停维）| **Sentinel**（多维度流控 + 控制台）|
| API 网关 | Zuul（阻塞）| **Spring Cloud Gateway**（响应式）|
| 配置中心 | Spring Cloud Config | **Nacos Config** |

### 2.2 服务注册：Eureka vs ZooKeeper

| 维度 | Eureka | ZooKeeper（见 [[机制-Zookeeper]]）|
|------|--------|--------------------------------------|
| CAP 选择 | **AP**（可用性优先）| **CP**（一致性优先）|
| 设计目标 | 专为服务注册发现设计 | 通用分布式协调组件 |
| 健康检查 | Client 心跳（定期发送）| ZNode 临时节点（Session 断开自动删除）|
| 一致性协议 | Gossip（最终一致）| ZAB（强一致）|
| 自我保护 | **有**（低心跳率时不摘除节点）| 无 |
| 结论 | 注册中心场景优选 AP（高可用更重要）| ZK 更适合分布式锁/协调 |

**Nacos 优势**：可在 AP/CP 间切换，同时提供服务注册 + 动态配置，一个组件替代 Eureka + Config。

### 2.3 OpenFeign：声明式 HTTP 调用

```java
@FeignClient(name = "order-service")  // 服务名（从注册中心发现）
public interface OrderServiceClient {
    @PostMapping("/orders")
    OrderVO createOrder(@RequestBody CreateOrderReq req);
}
// 调用方直接注入接口调用，无需手动拼装 URL 和处理 HTTP 细节
```

- OpenFeign = Feign（声明式 HTTP 客户端）+ Spring Cloud 集成（服务发现 + LoadBalancer）
- 底层通过 JDK 动态代理（见 [[机制-动态代理]]）生成接口实现，拦截方法调用后转为 HTTP 请求
- 负载均衡：OpenFeign 自身无 LB，委托给 Spring Cloud LoadBalancer（服务名 → 实例列表 → 选择）
- vs RestTemplate：Feign 声明式更简洁，内置 LB 集成；RestTemplate 命令式更灵活，适合调用外部接口

### 2.4 熔断器：Hystrix 三态模型

```
关闭（Closed）
    ↓ 错误率 > 阈值 且 请求数 > 最小阈值
开启（Open）→ 所有请求直接 fallback，不调用下游
    ↓ 等待 sleep window（默认 5s）
半开（Half-Open）→ 放入少量探测请求
    ↓ 探测成功       ↓ 探测失败
  关闭               重新开启
```

作用：**快速失败**（避免上游被慢/挂的下游拖垮）+ **无缝恢复**（自动检测下游是否恢复）。

### 2.5 Hystrix vs Sentinel

| 维度 | Hystrix（停维）| Sentinel（推荐）|
|------|--------------|----------------|
| 核心关注 | 隔离 + 熔断 | 多维度流控（QPS/并发/热点参数）+ 熔断 + 系统保护 |
| 隔离策略 | 线程池隔离（默认）/ 信号量隔离 | 信号量隔离 |
| 规则持久化 | 需自行实现 | 支持 Nacos/ZK 等多种持久化方案 |
| 实时监控 | Hystrix Dashboard（单独部署）| 自带控制台（实时规则修改 + Metrics）|
| 限流粒度 | 服务级别 | 资源级别（API 级别精细控制）|

### 2.6 API 网关

```
外部请求 → Nginx（反向代理 + 静态资源）→ Spring Cloud Gateway（鉴权 + 路由 + 限流）→ 微服务集群
```

**Spring Cloud Gateway**：
- 基于 Spring WebFlux（Reactor 响应式非阻塞）；取代 Zuul（Zuul 基于 Servlet 阻塞模型）
- 功能：路由转发、负载均衡（集成 LoadBalancer）、统一鉴权（OAuth2/JWT）、限流（集成 Sentinel）、跨域支持
- 瓶颈：所有请求必须经过网关，可能成为单点性能瓶颈

| 维度 | Nginx | Spring Cloud Gateway |
|------|-------|---------------------|
| 定位 | 通用 Web 服务器 + 反向代理 | 微服务 API 网关 |
| 负载均衡 | 服务端 LB（调用方不感知具体实例）| 客户端 LB（与注册中心集成）|
| 生态 | 无限制，适合任何技术栈 | Java 微服务生态 |
| 静态资源 | 支持（高性能）| 不支持 |
| 微服务集成 | 需手动配置，无服务发现 | 天然集成 Spring Cloud 服务发现 |

**生产实践**：Nginx 做外层反向代理 + 静态资源，Gateway 做内层业务路由 + 鉴权，分工明确。

### 2.7 Nacos 注册中心与配置中心

**注册中心生命周期**：
1. Provider 启动 → REST 请求注册到 Nacos Server，元数据存入**双层内存 Map**
2. 心跳维持：Provider 每 **5s** 发一次心跳
3. 健康检查：Server 超 **15s** 未收到心跳 → `healthy=false`（消费者不发现该实例）；超 **30s** → 剔除（恢复后重新注册）
4. Consumer 本地缓存 + 定时轮询更新注册表

**雪崩保护阈值**：设 0~1 之间（如 0.6）。当 `健康实例数/总实例数 < 阈值` 时，Nacos 允许返回不健康实例（防止全流量打挂仅剩的健康节点）。

**配置中心三元组**：Namespace（环境：dev/test/prod）→ Group（项目分组）→ DataId（具体配置文件）。

优先级：精准配置（service-dev.yaml）> 同工程通用 > 扩展配置 > 共享配置。

**动态配置刷新**：`@Value` 不感知变更；加 `@RefreshScope` 后 Bean 在配置变更时重新初始化。Nacos 用**长轮询**推送变更，比 Spring Cloud Config + Bus 快，且内置可视化控制台。

## 三、OpenFeign 机制详解

OpenFeign 是声明式 HTTP 客户端：开发者只定义 Java 接口和 Spring MVC 注解，运行期由 Feign 创建代理对象，把一次方法调用转换成一次 HTTP 请求；Spring Cloud OpenFeign 在此基础上集成服务发现、负载均衡、编码解码、超时和容错能力。

### 3.1 第一性原理：把手写 HTTP 调用抽象成接口

没有 Feign 时，服务间 HTTP 调用通常要手写：

1. 从注册中心或配置中拿目标地址。
2. 拼 URL、Header、Query、Body。
3. 选择实例并处理负载均衡。
4. 发起 HTTP 请求。
5. 反序列化响应。
6. 处理超时、异常和重试。

OpenFeign 的核心价值是把这些模板代码收敛到框架中。业务代码只保留接口语义：

```java
@FeignClient(name = "order-service")
public interface OrderClient {
    @PostMapping("/orders")
    OrderVO create(@RequestBody CreateOrderRequest request);
}
```

调用方注入 `OrderClient` 后直接调用 `create()`，框架负责把方法调用翻译成 HTTP 请求。写法像本地调用，但成本仍然是远程调用，因此超时、降级、监控不能省。

### 3.2 核心用法

```java
@FeignClient(name = "user-service", path = "/users")
public interface UserClient {
    @GetMapping("/{id}")
    UserVO getById(@PathVariable("id") Long id);

    @PostMapping
    UserVO create(@RequestBody CreateUserRequest request);
}
```

| 配置 | 含义 |
| --- | --- |
| `name` | 服务名，通常对应注册中心中的服务 ID |
| `path` | 当前客户端统一路径前缀 |
| `url` | 固定地址，常用于调用第三方服务或本地测试 |
| `fallback` / `fallbackFactory` | 降级实现，需结合熔断组件 |
| `configuration` | 指定当前 FeignClient 的局部配置 |

Spring Cloud OpenFeign 支持 Spring MVC 注解：

| 注解 | 用途 |
| --- | --- |
| `@GetMapping` / `@PostMapping` | 声明 HTTP 方法和路径 |
| `@PathVariable` | 路径变量 |
| `@RequestParam` | Query 参数 |
| `@RequestHeader` | 请求头 |
| `@RequestBody` | 请求体 |

多个复杂参数不要依赖隐式推断，最好显式标注 `@RequestParam` 或封装为 `@RequestBody`，避免 Feign 元数据解析和服务端参数绑定不一致。

### 3.3 执行流程

```
调用 Feign 接口方法
      |
      v
JDK 动态代理拦截方法调用
      |
      v
读取 MethodMetadata（HTTP 方法、路径、参数、Header）
      |
      v
构造 RequestTemplate
      |
      v
服务发现 + 负载均衡选择实例
      |
      v
HTTP Client 执行请求
      |
      v
Decoder 反序列化响应 / ErrorDecoder 处理异常
```

关键点：

- Feign 本身是接口代理框架，Spring Cloud OpenFeign 负责和 Spring MVC 注解、注册中心、负载均衡、编码解码体系集成。
- 接口代理常用 JDK 动态代理完成，调用方法时进入 Feign 的 InvocationHandler。
- `Encoder` 负责把 Java 对象编码成请求体，`Decoder` 负责把响应体转成 Java 对象。
- `RequestInterceptor` 可统一追加 Token、TraceId、租户 ID 等 Header。

### 3.4 负载均衡与服务发现

当 `@FeignClient(name = "order-service")` 未配置固定 `url` 时，`name` 会作为服务名交给注册中心和负载均衡组件处理。

| 组件 | 作用 |
| --- | --- |
| **Nacos/Eureka** | 保存服务实例列表 |
| **Spring Cloud LoadBalancer** | 从实例列表中选择一个实例 |
| **Ribbon** | 老版本负载均衡组件，已逐步被 LoadBalancer 替代 |

老版本 Spring Cloud 中常见说法是 Feign 底层整合 Ribbon；现代版本更准确的说法是 OpenFeign 集成 Spring Cloud LoadBalancer。无论底层是哪一个，Feign 的业务调用模型都是：**服务名 -> 实例列表 -> 选择实例 -> HTTP 请求**。

### 3.5 超时、重试与容错

Feign 的超时通常分两类：

| 参数 | 含义 |
| --- | --- |
| **connectTimeout** | 建立连接的超时时间 |
| **readTimeout** | 请求发出后等待响应的超时时间 |

原始笔记中的要点：Feign `Options` 第一个参数是连接超时，第二个参数是读取超时；常见默认值是连接 2s、读取 5s。若同时存在 Ribbon/LoadBalancer/HTTP Client 相关配置，应以当前 Feign 客户端最终生效配置为准，避免多层超时互相打架。

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connectTimeout: 2000
            readTimeout: 5000
          order-service:
            connectTimeout: 1000
            readTimeout: 3000
```

重试不是免费能力，尤其是写接口：

| 场景 | 建议 |
| --- | --- |
| 查询接口 | 可有限重试，需设置总超时预算 |
| 创建/扣款/发券等写接口 | 默认不要透明重试，除非下游保证幂等 |
| 网络瞬断 | 可在网关或调用方做有限重试 |
| 下游慢 | 优先限流、熔断、降级，不要用无限重试放大故障 |

OpenFeign 可以结合 Sentinel、Resilience4j 等实现熔断降级。推荐用 `fallbackFactory`，因为它可以拿到异常原因，便于区分超时、连接失败、业务错误。

```java
@FeignClient(name = "inventory-service", fallbackFactory = InventoryFallbackFactory.class)
public interface InventoryClient {
    @PostMapping("/inventory/deduct")
    DeductResult deduct(@RequestBody DeductRequest request);
}
```

### 3.6 OpenFeign 应用边界

适合使用 OpenFeign：

- Spring Cloud 微服务之间的 HTTP 调用。
- 接口相对稳定，适合抽象为 Java 接口。
- 需要服务发现、负载均衡、统一拦截器、编码解码集成。
- 对跨语言和 HTTP 生态有要求。

谨慎使用 OpenFeign：

- 超高频、极低延迟的内部调用。
- 大文件上传下载。
- 强事务链路中串联多个远程写调用。
- 下游接口不稳定且缺少幂等能力。

## 四、Spring Cloud Gateway 机制详解

Spring Cloud Gateway 是 Spring Cloud 官方的第二代 API 网关，基于 Spring WebFlux、Project Reactor 和 Netty 构建，用声明式路由、断言和过滤器完成流量入口治理：路由转发、统一鉴权、限流、路径重写、监控和灰度发布。

### 4.1 第一性原理：微服务入口治理

微服务拆分后，外部客户端如果直接访问每个服务，会遇到四类问题：

| 问题 | 网关职责 |
| --- | --- |
| 服务地址暴露且变化频繁 | 统一入口 + 服务发现路由 |
| 每个服务都重复做鉴权、跨域、限流 | 入口层统一横切治理 |
| 对外 API 路径和内部服务路径不一致 | 路径重写和协议适配 |
| 灰度、监控、审计缺少统一入口 | 入口流量打标、采集、分流 |

Gateway 的定位不是替代业务服务，而是作为**微服务 API 流量入口**，把入口层通用能力从各个业务服务中抽离出来。

### 4.2 Route / Predicate / Filter

Route 是网关的基础路由单元，通常包含：

| 字段 | 含义 |
| --- | --- |
| **id** | 路由唯一标识 |
| **uri** | 目标地址，可以是固定 URL，也可以是 `lb://service-name` |
| **predicates** | 路由匹配条件 |
| **filters** | 当前路由生效的过滤器链 |

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
```

Predicate 是路由匹配条件，输入是 `ServerWebExchange`，可以基于路径、方法、Header、Query、Cookie、Host、时间窗口等信息判断是否命中路由。

| Predicate | 用途 |
| --- | --- |
| `Path` | 按 URL 路径匹配 |
| `Method` | 按 HTTP 方法匹配 |
| `Header` | 按请求头匹配 |
| `Query` | 按查询参数匹配 |
| `Host` | 按域名匹配 |
| `Weight` | 按权重做灰度分流 |

Filter 用于处理请求和响应：

| 类型 | 范围 | 用途 |
| --- | --- | --- |
| **GatewayFilter** | 单个 Route | 路径重写、加 Header、限流、熔断 |
| **GlobalFilter** | 全局所有 Route | 统一鉴权、日志、Trace、全局异常处理 |

### 4.3 执行流程

```
Client
  -> HttpWebHandlerAdapter
  -> DispatcherHandler
  -> RoutePredicateHandlerMapping
  -> FilteringWebHandler
  -> Pre Filters
  -> 目标微服务
  -> Post Filters
  -> Response
```

关键步骤：

1. 客户端请求进入 Gateway Server。
2. `HttpWebHandlerAdapter` 将请求组装成 WebFlux 上下文。
3. `DispatcherHandler` 分发请求。
4. `RoutePredicateHandlerMapping` 查找所有 Route，并执行 Predicate 判断是否匹配。
5. 匹配成功后，`FilteringWebHandler` 创建过滤器链。
6. 请求依次经过 Pre Filter，转发到目标微服务。
7. 下游响应返回后，再经过 Post Filter，最终响应客户端。

这条链路的核心是**先匹配路由，再执行过滤器链**。因此，路由配置错误会导致请求根本进不了业务过滤逻辑。

### 4.4 常见能力

**服务发现路由**：Gateway 可以和 Nacos/Eureka 等注册中心集成，用 `lb://服务名` 转发到服务实例，并通过 Spring Cloud LoadBalancer 做客户端负载均衡。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
```

**路径重写**：外部 API 路径通常和内部服务路径不同，常用过滤器做转换。

```yaml
filters:
  - StripPrefix=1
  - RewritePath=/api/(?<segment>.*), /${segment}
```

**统一鉴权**：常见做法是在 GlobalFilter 中解析 JWT/OAuth2 Token。白名单直接放行；其余请求校验 Token 签名、过期时间和权限；再将用户 ID、租户 ID、角色等信息写入 Header。鉴权失败应在网关直接返回 `401/403`，避免无效流量打到业务服务。

**限流与熔断**：Gateway 可集成 Redis RateLimiter、Sentinel 或 Resilience4j。

| 能力 | 典型实现 |
| --- | --- |
| IP/用户/API 限流 | Redis RateLimiter、Sentinel |
| 热点接口保护 | Sentinel 资源维度规则 |
| 下游异常快速失败 | CircuitBreaker filter |
| 降级兜底 | fallbackUri |

限流规则建议按资源拆分，不要只做全局 QPS。否则一个热点 API 可能挤占所有入口流量。

**动态路由**：路由可以来自本地 YAML，也可以来自 Nacos、数据库或配置中心。动态路由变更要能原子刷新，配置要可审计、可回滚，灰度规则要能和服务版本、Header、用户标签联动。

### 4.5 过滤器机制

Pre Filter 在请求转发前执行，常见职责：

- 认证鉴权
- 请求日志和 Trace ID 注入
- Header 增删改
- 参数校验
- 限流和黑白名单
- 路径重写

Post Filter 在下游响应后执行，常见职责：

- 响应 Header 增删改
- 统一审计日志
- 响应耗时统计
- 异常包装

不要在 Gateway 里读取并改写大响应体。响应式流被消费后需要重新包装，容易引入内存和性能问题。

多个过滤器通过 `Ordered` 控制执行顺序。一般原则：

1. Trace、日志上下文最先执行。
2. 黑名单、限流、鉴权尽早执行。
3. 路径重写在转发前完成。
4. 统计、审计在响应后执行。

### 4.6 Gateway 应用边界

适合放在 Gateway 的逻辑：

- 统一认证、Token 解析、租户识别。
- API 路由、路径重写、版本路由。
- 限流、黑白名单、基础风控。
- 灰度发布、流量染色、Trace ID 注入。
- 入口审计日志和指标采集。

不适合放在 Gateway 的逻辑：

- 复杂业务编排。
- 需要访问多个业务数据库的逻辑。
- 大量阻塞 IO 或 CPU 密集计算。
- 需要强一致事务的业务流程。

## 五、综合对比

### 5.1 SpringCloud vs Dubbo

| 维度 | Spring Cloud | Dubbo |
|------|-------------|-------|
| 定位 | 完整微服务框架（含注册/LB/熔断/网关）| RPC 框架（专注服务调用 + 治理）|
| 通信协议 | HTTP/HTTPS（REST 为主）| TCP 自定义协议（高性能）|
| 语言支持 | 多语言（HTTP 天然跨语言）| 主要 Java（Dubbo-go 支持 Go）|
| 适用场景 | 对外 API、异构环境、全栈 Spring | 内部高并发服务调用、强治理需求 |
| 生态 | Spring 官方维护，生态完整 | 阿里/Apache，服务治理能力强 |

**实战：两者也可结合**——用 Dubbo 做内部 RPC，Spring Cloud Gateway 做外部接入网关。

### 5.2 Feign vs RestTemplate

| 维度 | OpenFeign | RestTemplate |
|------|-----------|--------------|
| 风格 | 声明式（接口 + 注解）| 命令式（手动构建请求）|
| LB 集成 | 内置 LoadBalancer 集成 | 需手动配置 |
| 适用 | 内部服务调用 | 调用外部第三方 HTTP 接口（无服务注册）|

### 5.3 OpenFeign vs RestTemplate vs Dubbo

| 维度 | OpenFeign | RestTemplate | Dubbo |
| --- | --- | --- | --- |
| 调用风格 | 声明式接口 | 命令式手写 HTTP | RPC 接口 |
| 协议 | HTTP | HTTP | Dubbo/gRPC/Triple 等 |
| 服务发现 | Spring Cloud 集成 | 需额外配置 | 内建服务治理 |
| 负载均衡 | LoadBalancer/Ribbon | 需额外配置 | 框架内置 |
| 性能 | 中等，跨语言友好 | 中等，灵活 | 通常更高，适合内部高频调用 |
| 适用 | 微服务内部 HTTP 调用 | 第三方 HTTP、特殊请求 | Java 内部高性能 RPC |

OpenFeign 不是 RPC 框架，它仍然是 HTTP 客户端。它的优势是开发效率、Spring 生态集成和跨语言友好；如果内部调用对吞吐、延迟、连接复用和二进制协议治理要求很高，Dubbo 这类 RPC 框架更合适。

### 5.4 Gateway vs Zuul vs Nginx

| 维度 | Spring Cloud Gateway | Zuul 1.x | Nginx |
| --- | --- | --- | --- |
| 定位 | 微服务 API 网关 | 旧一代 Java API 网关 | 通用 Web 服务器/反向代理 |
| IO 模型 | WebFlux + Netty 响应式非阻塞 | Servlet 阻塞模型 | 事件驱动非阻塞 |
| 服务发现 | 原生集成 Spring Cloud | 可集成但体系较旧 | 通常需手工或额外模块 |
| 扩展方式 | Predicate + Filter | Filter | 配置、Lua、模块 |
| 业务治理 | 鉴权、限流、灰度、熔断更贴近 Java 生态 | 能力较旧 | 强代理能力，业务治理需扩展 |
| 静态资源 | 不适合 | 不适合 | 适合 |

常见生产分层：

```
Client -> Nginx/SLB/WAF -> Spring Cloud Gateway -> 微服务
```

Nginx 负责外层四/七层代理、TLS、静态资源和基础防护；Gateway 负责微服务语义下的路由、鉴权、限流和灰度。

## 六、核心使用原则

### 6.1 微服务拆分原则

| 原则 | 说明 |
|------|------|
| **职责单一** | 每个服务聚焦一件事，用户服务 ≠ 交易服务 |
| **业务边界** | 按业务域拆分（DDD 中的 Bounded Context），保险中投保与理赔是不同域 |
| **中台化** | 公共能力（会员积分、消息通知）可独立为平台服务，供多业务线复用 |
| **系统保障级别** | 秒杀链路与日常交易链路隔离，保证秒杀压力不影响正常下单 |
| **在线/离线分离** | 在线任务（实时响应）与离线任务（异步扫表/定时任务）单独部署，独立扩容 |
| **单向依赖** | 服务间不允许循环依赖；依赖必须是 DAG（有向无环图）|
| **康威定律** | 系统架构需匹配组织架构，一个微服务由一个团队负责，避免多团队争一个库 |

**微服务粒度误区**：服务不是越小越好——过细的拆分导致跨服务调用激增、分布式事务激增、运维成本爆炸。**服务粒度 = 能被一个小团队完整负责的最大业务单元**。

### 6.2 限流 / 降级 / 熔断的本质区别

| 策略 | 触发时机 | 作用对象 | 目的 | 典型场景 |
|------|---------|---------|------|---------|
| **限流** | 流量超过阈值 | 当前服务入口 | 保护自身不被压垮 | 双十一大促限制每秒下单数 |
| **降级** | 系统负载高 / 非核心功能 | 主动关闭部分功能 | 保核心、舍次要 | 大促期间关闭退款入口 |
| **熔断** | 下游错误率持续超阈值 | 对下游的调用 | 快速失败，防级联崩溃 | 微信支付超时率激增时断开调用 |

- **限流**通常由**被调用方**对调用方设置（"我最多接受 1000 QPS"）
- **降级**通常由调用方主动执行（"下游慢，我返回兜底数据"）
- **熔断**发生在**调用方**，检测到下游异常后自动断路，一段时间后半开探测

### 6.3 微服务循环依赖与解法

微服务循环依赖（A→B→C→A）危害：流量放大（QPS 倍增）、响应 RT 拉长、级联故障风险、发布顺序无法确定。

**解法优先级**：
1. **重新设计**：循环依赖往往是职责划分不清，先审视是否能合并或重新划分边界
2. **消息解耦**：同步调用改为 MQ 异步通知，断开强依赖
3. **抽取共享服务（中介服务）**：A 和 B 共同依赖 C（台账/数据服务），而不是互相调用

## 七、使用案例

### 7.1 部署策略：蓝绿 / 灰度 / 金丝雀

| 策略 | 机器资源 | 回滚速度 | 验证范围 | 适用场景 |
|------|---------|---------|---------|---------|
| **蓝绿部署** | 双倍（闲置一套）| 极快（切流量）| 全量前验证 | 稳定性高要求、预算充足 |
| **灰度/金丝雀** | 仅额外少量机器 | 较慢（需回退版本）| 小流量真实验证 | 常规发布首选；节省资源 |

**蓝绿部署**：Blue/Green 两套环境，流量全量切换，瞬间切回即回滚，但有 50% 机器常年闲置。

**金丝雀/灰度发布**：从集群中取少量机器先发布新版，逐步扩大流量比例（5% → 20% → 100%）。发现问题只需将问题机器下线即可，无需两套完整环境。

**快速回滚**：记录发布前基线（环境快照/容器镜像），回滚时基于基线重新部署，无需手工操作。

### 7.2 Service Mesh（服务网格）

**核心思想**：将微服务间的通信（服务发现、负载均衡、熔断、限流、TLS、监控）从业务代码中抽离，下沉到每个服务实例旁的 **Sidecar 代理**（通常是 Envoy）。

```
服务 A  →  Sidecar(A)  →  网络  →  Sidecar(B)  →  服务 B
                           ↑
                      控制平面（Istio/Linkerd）统一下发策略
```

**解决的问题**：多语言异构环境下，Java 的 Sentinel、Go 的自研限流、Python 的 requests 各有一套，Service Mesh 提供统一治理，语言无关。

**典型能力**：流量镜像（把线上流量拷贝到测试环境）、精细路由（按 Header/百分比分流）、mTLS 服务间加密、全链路 tracing。

**代价**：每次请求多一跳（本机 loopback，延迟极低 <1ms）；控制平面（Istio）本身运维复杂。

### 7.3 CI/CD 与 DevOps

**CI（持续集成）**：开发者每次提交代码后自动构建 + 单元测试，快速判断本次变更是否破坏现有功能。

**CD（持续交付/部署）**：
- **持续交付**：CI 通过后自动部署到 Staging 环境，人工审批后发生产
- **持续部署**：全自动推到生产，无需人工干预

**DevOps**：一种文化/理念——开发（Dev）和运维（Ops）不再割裂，共同对软件全生命周期负责，由 CI/CD 自动化工具链支撑。核心是"谁构建谁运维"（You build it, you run it）。

## 八、关键权衡

| 权衡点 | 说明 |
|--------|------|
| AP vs CP 注册中心 | 注册中心对可用性要求高（服务发现临时不一致影响有限），选 Eureka/Nacos AP 模式；配置中心对一致性要求高，选 CP 模式 |
| 客户端 LB vs 服务端 LB | 客户端 LB（LoadBalancer/Ribbon）无中间节点，性能好但每个 Consumer 都要感知 Provider 列表；Nginx 服务端 LB 集中管理，但多一跳延迟 |
| 线程池隔离 vs 信号量隔离 | Hystrix 线程池隔离完全隔离故障但线程切换有开销；信号量隔离轻量但无法做超时控制（适合本地快速执行的逻辑）|
| Spring Cloud vs Spring Cloud Alibaba | Alibaba 套件（Nacos+Sentinel+RocketMQ+Seata）更适合国内生态；原版 Spring Cloud 更适合国际化云原生场景 |
| OpenFeign 是否像本地调用 | 写法像本地调用，成本仍是远程调用；必须设置超时、降级、监控和幂等边界 |
| Feign 重试 | 读接口可谨慎开启；写接口必须先保证幂等，否则会放大重复扣款/重复下单风险 |
| Gateway WebFlux 非阻塞模型 | 并发能力好，但过滤器中不能随意写阻塞 IO；Gateway 不能运行在传统 Servlet 容器中，也不应构建成 war |
| 网关职责边界 | 网关做统一认证、路由、限流、灰度；复杂业务编排和强事务逻辑应留在业务服务 |

## 九、与其他概念的关系

- **依赖 [[机制-Dubbo]]**：Dubbo 可作为 Spring Cloud 内部服务间调用的替代方案（OpenFeign HTTP → Dubbo TCP），两者互补而非互斥
- **依赖 [[机制-Zookeeper]]**：Eureka/Nacos 之前，ZooKeeper 也被用作注册中心；ZK 的 CP 特性使其在服务注册场景中略显过重
- **依赖 [[机制-Spring]] / [[机制-动态代理]]**：OpenFeign 底层通过动态代理生成 Stub；Hystrix 通过 AOP 切面拦截方法调用并织入熔断逻辑
- **依赖 [[机制-SpringBoot]]**：Spring Cloud 各组件均通过 Spring Boot AutoConfiguration 自动装配，`@EnableEurekaClient`/`@EnableFeignClients` 等注解触发自动配置链路

## 十、应用边界

**适合 Spring Cloud**：Java 技术栈的微服务全栈解决方案；需要快速搭建注册中心/网关/配置中心；与 Spring Boot 深度集成的团队。

**不适合 Spring Cloud**：极高性能内部调用（考虑 Dubbo，HTTP 比 TCP RPC 多约 30-50% 开销）；非 Java 技术栈（HTTP 可互通，但配套组件均为 Java）；超大规模（Netflix 内部已放弃 Eureka，转向 Kubernetes 原生服务发现）。
