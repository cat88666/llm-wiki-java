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
  - "../../../raw/note/Hollis/SpringCloud.md"
  - "../../../raw/note/Hollis/SpringCloud/"
  - "../../../raw/note/Hollis/微服务/"
  - "../../../raw/note/Hollis/配置中心/"
updated: 2026-05-18
---

# 微服务与SpringCloud

> Spring Cloud 是 Spring Boot 生态下的微服务治理工具集，解决“服务怎么发现、请求怎么路由、故障怎么隔离、配置怎么动态变更、发布怎么控风险”这些分布式系统共性问题。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、微服务拆分后的新问题](#sec-2) | 远程调用带来发现、负载、容错、治理 |
| [二、组件全景与现代选型](#sec-3) | Nacos、LoadBalancer、OpenFeign、Sentinel、Gateway |
| [三、注册发现与配置中心](#sec-4) | AP/CP、Nacos 心跳、配置长轮询 |
| [四、OpenFeign 调用链路](#sec-5) | 接口代理、服务发现、负载均衡、超时重试 |
| [五、Gateway 与流量入口](#sec-6) | Route、Predicate、Filter、Nginx 分工 |
| [六、熔断限流与发布治理](#sec-7) | Sentinel、降级、灰度、Service Mesh |
| [七、架构权衡与拆分原则](#sec-8) | 微服务循环依赖、边界、通信选型 |
| [八、生产风险与排查](#sec-9) | 超时风暴、网关瓶颈、配置事故、循环依赖 |
| [九、面试速答口径](#sec-10) | 高频问题答案 |

<a id="sec-2"></a>
## 一、微服务拆分后的新问题

单体拆成微服务后，进程内方法调用变成跨网络调用，系统多了四类问题：

| 问题 | 典型能力 |
| --- | --- |
| 如何找到服务 | 注册中心、服务发现 |
| 如何选择实例 | 负载均衡 |
| 下游慢/挂怎么办 | 超时、重试、熔断、降级、限流 |
| 配置和发布怎么控风险 | 配置中心、灰度、蓝绿、金丝雀 |

Spring Cloud 的价值不是“让调用像本地方法”，而是把远程调用的治理能力标准化。

<a id="sec-3"></a>
## 二、组件全景与现代选型

| 问题领域 | 老组件 | 现代常用 |
| --- | --- | --- |
| 注册发现 | Eureka | Nacos / Consul |
| 负载均衡 | Ribbon | Spring Cloud LoadBalancer |
| HTTP 调用 | Feign | OpenFeign / HttpInterface |
| 熔断限流 | Hystrix | Sentinel / Resilience4j |
| 网关 | Zuul | Spring Cloud Gateway |
| 配置中心 | Spring Cloud Config | Nacos / Apollo |

选型结论：Spring Cloud Alibaba 体系常用 Nacos + Sentinel + OpenFeign + Gateway；新项目要关注 Spring Cloud 版本与 Spring Boot 版本矩阵。

<a id="sec-4"></a>
## 三、注册发现与配置中心

### 3.1 注册中心 AP/CP 取舍

| 组件 | CAP 倾向 | 适用 |
| --- | --- | --- |
| Eureka | AP | 服务注册发现，高可用优先 |
| ZooKeeper | CP | 分布式协调、锁、强一致元数据 |
| Nacos | AP/CP 可切 | 注册中心 + 配置中心统一治理 |

注册中心场景更重视可用性：短时间读到旧实例通常比注册中心不可用更可接受。

### 3.2 Nacos 服务生命周期

```text
Provider 启动注册
  -> 定期心跳
  -> 超时标记 unhealthy
  -> 更长时间未恢复则剔除
Consumer
  -> 本地缓存服务列表
  -> 监听或轮询变更
```

### 3.3 配置中心

Nacos 配置通常按 `Namespace -> Group -> DataId` 管理。动态刷新常见做法是 `@RefreshScope`，配置变更后重建作用域 Bean。

<a id="sec-5"></a>
## 四、OpenFeign 调用链路

```java
@FeignClient(name = "order-service", path = "/orders")
public interface OrderClient {
    @GetMapping("/{id}")
    OrderVO getById(@PathVariable Long id);
}
```

运行链路：

```text
调用接口方法
  -> JDK 动态代理拦截
  -> 解析 MethodMetadata
  -> 构造 RequestTemplate
  -> 服务发现 + LoadBalancer 选实例
  -> HTTP Client 执行
  -> Decoder / ErrorDecoder 处理响应
```

超时必须分层治理：

| 类型 | 含义 |
| --- | --- |
| connectTimeout | 建连超时 |
| readTimeout | 响应读取超时 |
| retry | 重试次数与重试条件 |
| fallback | 熔断或异常后的降级结果 |

反直觉点：Feign 写法像本地调用，但成本和失败模式都是远程调用。

<a id="sec-6"></a>
## 五、Gateway 与流量入口

Spring Cloud Gateway 基于 WebFlux/Reactor，核心模型是：

```text
Route = Predicate + Filter + URI
```

请求链路：

```text
外部请求
  -> Nginx（四七层转发、静态资源、TLS）
  -> Gateway（鉴权、路由、限流、灰度）
  -> 后端微服务
```

Gateway vs Nginx：

| 维度 | Nginx | Gateway |
| --- | --- | --- |
| 定位 | 通用反向代理 | 微服务业务网关 |
| 服务发现 | 需额外配置 | 原生接注册中心 |
| 业务过滤 | 弱 | 强 |
| 静态资源 | 强 | 弱 |

<a id="sec-7"></a>
## 六、熔断限流与发布治理

### 6.1 熔断三态

```text
Closed 正常调用
  -> 错误率/慢调用超阈值
Open 快速失败
  -> 等待窗口后少量探测
Half-Open
  -> 成功关闭 / 失败重新打开
```

### 6.2 Sentinel 治理能力

| 能力 | 说明 |
| --- | --- |
| 限流 | QPS、并发线程、热点参数 |
| 熔断 | 异常比例、慢调用比例 |
| 降级 | 返回兜底结果 |
| 系统保护 | 从整体 load/RT/线程数保护入口 |

### 6.3 发布策略

| 策略 | 结论 |
| --- | --- |
| 蓝绿 | 两套环境切换，回滚快但成本高 |
| 灰度/金丝雀 | 小流量验证，风险低 |
| 滚动发布 | 成本低，但回滚和兼容要求高 |
| Service Mesh | 治理下沉到 sidecar，业务侵入低但平台复杂 |

<a id="sec-8"></a>
## 七、架构权衡与拆分原则

### 7.1 微服务拆分原则

1. 按业务能力和数据所有权拆分，不按技术层拆分。
2. 服务间调用避免循环依赖。
3. 查询组合优先考虑 API 聚合层或 CQRS 视图。
4. 强一致链路减少跨服务同步调用。

### 7.2 循环依赖危害

| 危害 | 说明 |
| --- | --- |
| 流量放大 | A 调 B，B 再调 A，QPS 被放大 |
| RT 拉长 | 服务互相等待 |
| 故障传播 | 一个服务异常沿环路扩散 |
| 发布困难 | 无法判断谁先上线 |

识别方式：链路追踪中同一服务在一次 trace 内反复出现，或发布评审中出现互相等待。

<a id="sec-9"></a>
## 八、生产风险与排查

| 风险 | 现象 | 排查点 |
| --- | --- | --- |
| 超时风暴 | 上游线程堆积 | Feign 超时、重试、线程池、下游 RT |
| 网关瓶颈 | 所有接口 RT 上升 | Gateway filter、连接池、限流配置 |
| 注册表异常 | 请求打到下线实例 | 心跳、健康检查、缓存刷新 |
| 配置事故 | 多服务同时异常 | Namespace/Group/DataId、灰度发布 |
| 循环依赖 | trace 重复服务节点 | 服务边界、共享库、异步事件 |
| 熔断误伤 | 正常请求被 fallback | 规则阈值、慢调用比例、窗口设置 |

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| Spring Cloud 解决什么 | 注册发现、负载均衡、熔断限流、网关、配置治理 |
| OpenFeign 怎么工作 | 接口动态代理把方法调用转成 HTTP 请求，再结合服务发现和负载均衡 |
| Gateway 和 Nginx 区别 | Nginx 是通用反向代理，Gateway 是微服务业务网关 |
| Eureka 和 ZK 注册中心怎么选 | 注册发现更偏 AP，ZK 更适合强一致协调 |
| 微服务循环依赖为什么危险 | 流量放大、RT 变长、故障传播、发布困难 |
