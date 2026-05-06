---
type: concept
status: active
name: "微服务与SpringCloud"
layer: L7
aliases: ["SpringCloud", "微服务", "Eureka", "Feign", "OpenFeign", "Hystrix", "Sentinel", "Gateway", "Ribbon", "LoadBalancer", "Nacos", "熔断器", "服务注册", "服务发现"]
related:
  - "[[机制-RPC与Dubbo]]"
  - "[[机制-ZAB协议与Zookeeper]]"
  - "[[机制-AOP织入]]"
  - "[[机制-SpringBoot自动装配]]"
sources:
  - "../../raw/note/Hollis/SpringCloud/✅什么是SpringCloud，有哪些组件？.md"
  - "../../raw/note/Hollis/SpringCloud/✅SpringCloud和Dubbo有什么区别？.md"
  - "../../raw/note/Hollis/SpringCloud/✅Eureka和Zookeeper有什么区别？.md"
  - "../../raw/note/Hollis/SpringCloud/✅Hystrix熔断器的工作原理是什么？.md"
  - "../../raw/note/Hollis/SpringCloud/✅Hystrix和Sentinel的区别是什么？.md"
  - "../../raw/note/Hollis/SpringCloud/✅OpenFeign 是如何实现负载均衡的？.md"
  - "../../raw/note/Hollis/SpringCloud/✅Feign 和 RestTemplate 有什么不同？.md"
  - "../../raw/note/Hollis/SpringCloud/✅LoadBalancer和Ribbon的区别是什么？为什么用他替代Ribbon？.md"
  - "../../raw/note/Hollis/SpringCloud/✅为什么需要SpringCloud Gateway，他起到了什么作用？.md"
  - "../../raw/note/Hollis/SpringCloud/✅Zuul、Gateway和Nginx有什么区别？.md"
  - "../../raw/note/Hollis/SpringCloud/✅Dubbo和Feign有什么区别？.md"
  - "../../raw/note/Hollis/SpringCloud/✅介绍一下 Hystrix 的隔离策略，你用哪个？.md"
  - "../../raw/note/Hollis/SpringCloud/✅什么是Eureka的自我保护模式？.md"
  - "../../raw/note/Hollis/SpringCloud/✅在 Spring Cloud 中，服务间的通信有哪些方式？.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# 微服务与SpringCloud

> Spring Cloud 是基于 Spring Boot 的微服务开发工具集：通过一系列开箱即用的组件（注册中心、负载均衡、熔断器、网关、配置中心），解决分布式系统中服务注册发现、流量管控和容错等共性问题，核心理念是"专业的组件做专业的事"。

## 第一性原理

单体应用拆分为微服务后，原本的进程内调用变为跨网络调用，带来三个根本问题：**如何找到对方**（服务发现）、**如何选择对方**（负载均衡）、**如何在对方出问题时自我保护**（容错/熔断）。Spring Cloud 的每个组件都在解决其中一个或多个问题。

## 核心机制

### 组件全景（现代选型）

| 问题领域 | 老版本组件 | 现代推荐替代 |
|---------|-----------|------------|
| 服务注册/发现 | Eureka | **Nacos**（AP/CP 可切，含配置中心）|
| 负载均衡 | Ribbon（停维）| **Spring Cloud LoadBalancer** |
| 服务间调用 | RestTemplate | **OpenFeign**（声明式 HTTP）|
| 熔断/限流/降级 | Hystrix（停维）| **Sentinel**（多维度流控 + 控制台）|
| API 网关 | Zuul（阻塞）| **Spring Cloud Gateway**（响应式）|
| 配置中心 | Spring Cloud Config | **Nacos Config** |

### SpringCloud vs Dubbo

| 维度 | Spring Cloud | Dubbo |
|------|-------------|-------|
| 定位 | 完整微服务框架（含注册/LB/熔断/网关）| RPC 框架（专注服务调用 + 治理）|
| 通信协议 | HTTP/HTTPS（REST 为主）| TCP 自定义协议（高性能）|
| 语言支持 | 多语言（HTTP 天然跨语言）| 主要 Java（Dubbo-go 支持 Go）|
| 适用场景 | 对外 API、异构环境、全栈 Spring | 内部高并发服务调用、强治理需求 |
| 生态 | Spring 官方维护，生态完整 | 阿里/Apache，服务治理能力强 |

**实战：两者也可结合**——用 Dubbo 做内部 RPC，Spring Cloud Gateway 做外部接入网关。

### 服务注册：Eureka vs ZooKeeper

| 维度 | Eureka | ZooKeeper（见 [[机制-ZAB协议与Zookeeper]]）|
|------|--------|--------------------------------------|
| CAP 选择 | **AP**（可用性优先）| **CP**（一致性优先）|
| 设计目标 | 专为服务注册发现设计 | 通用分布式协调组件 |
| 健康检查 | Client 心跳（定期发送）| ZNode 临时节点（Session 断开自动删除）|
| 一致性协议 | Gossip（最终一致）| ZAB（强一致）|
| 自我保护 | **有**（低心跳率时不摘除节点）| 无 |
| 结论 | 注册中心场景优选 AP（高可用更重要）| ZK 更适合分布式锁/协调 |

**Nacos 优势**：可在 AP/CP 间切换，同时提供服务注册 + 动态配置，一个组件替代 Eureka + Config。

### OpenFeign：声明式 HTTP 调用

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

### 熔断器：Hystrix 三态模型

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

### Hystrix vs Sentinel

| 维度 | Hystrix（停维）| Sentinel（推荐）|
|------|--------------|----------------|
| 核心关注 | 隔离 + 熔断 | 多维度流控（QPS/并发/热点参数）+ 熔断 + 系统保护 |
| 隔离策略 | 线程池隔离（默认）/ 信号量隔离 | 信号量隔离 |
| 规则持久化 | 需自行实现 | 支持 Nacos/ZK 等多种持久化方案 |
| 实时监控 | Hystrix Dashboard（单独部署）| 自带控制台（实时规则修改 + Metrics）|
| 限流粒度 | 服务级别 | 资源级别（API 级别精细控制）|

### API 网关：Gateway vs Nginx

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

## 关键权衡

1. **AP vs CP 注册中心**：注册中心对可用性要求高（服务发现临时不一致影响有限），选 Eureka/Nacos AP 模式；配置中心对一致性要求高，选 CP 模式
2. **客户端 LB vs 服务端 LB**：客户端 LB（LoadBalancer/Ribbon）无中间节点，性能好但每个 Consumer 都要感知 Provider 列表；Nginx 服务端 LB 集中管理，但多一跳延迟
3. **线程池隔离 vs 信号量隔离**：Hystrix 线程池隔离完全隔离故障但线程切换有开销；信号量隔离轻量但无法做超时控制（适合本地快速执行的逻辑）
4. **Feign vs RestTemplate**：Feign 声明式简洁，适合内部服务调用；RestTemplate 灵活，适合调用外部第三方 HTTP 接口（无服务注册）
5. **Spring Cloud vs Spring Cloud Alibaba**：Alibaba 套件（Nacos+Sentinel+RocketMQ+Seata）更适合国内生态；原版 Spring Cloud 更适合国际化云原生场景

## 与其他概念的关系

- **依赖 [[机制-RPC与Dubbo]]**：Dubbo 可作为 Spring Cloud 内部服务间调用的替代方案（OpenFeign HTTP → Dubbo TCP），两者互补而非互斥
- **依赖 [[机制-ZAB协议与Zookeeper]]**：Eureka/Nacos 之前，ZooKeeper 也被用作注册中心；ZK 的 CP 特性使其在服务注册场景中略显过重
- **依赖 [[机制-AOP织入]]**：OpenFeign 底层通过动态代理生成 Stub；Hystrix 通过 AOP 切面拦截方法调用并织入熔断逻辑
- **依赖 [[机制-SpringBoot自动装配]]**：Spring Cloud 各组件均通过 Spring Boot AutoConfiguration 自动装配，`@EnableEurekaClient`/`@EnableFeignClients` 等注解触发自动配置链路

## 应用边界

**适合 Spring Cloud**：Java 技术栈的微服务全栈解决方案；需要快速搭建注册中心/网关/配置中心；与 Spring Boot 深度集成的团队。

**不适合 Spring Cloud**：极高性能内部调用（考虑 Dubbo，HTTP 比 TCP RPC 多约 30-50% 开销）；非 Java 技术栈（HTTP 可互通，但配套组件均为 Java）；超大规模（Netflix 内部已放弃 Eureka，转向 Kubernetes 原生服务发现）。
