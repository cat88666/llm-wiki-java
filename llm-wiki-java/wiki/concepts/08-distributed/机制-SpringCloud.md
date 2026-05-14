---
type: concept
status: active
name: "微服务与SpringCloud"
layer: L7
aliases: ["SpringCloud", "微服务", "Eureka", "Feign", "OpenFeign", "Hystrix", "Sentinel", "Gateway", "Ribbon", "LoadBalancer", "Nacos", "熔断器", "服务注册", "服务发现", "蓝绿部署", "灰度发布", "金丝雀发布", "ServiceMesh", "Service Mesh", "Istio", "CI/CD", "DevOps", "康威定律", "限流降级熔断", "循环依赖"]
related:
  - "[[机制-RPC与Dubbo]]"
  - "[[机制-ZAB协议与Zookeeper]]"
  - "[[机制-Spring]]"
  - "[[机制-SpringBoot]]"
  - "[[主题-三高体系]]"
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
updated: 2026-05-07
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

### Nacos 注册中心与配置中心机制

**注册中心生命周期**：
1. Provider 启动 → REST 请求注册到 Nacos Server，元数据存入**双层内存 Map**
2. 心跳维持：Provider 每 **5s** 发一次心跳
3. 健康检查：Server 超 **15s** 未收到心跳 → `healthy=false`（消费者不发现该实例）；超 **30s** → 剔除（恢复后重新注册）
4. Consumer 本地缓存 + 定时轮询更新注册表

**雪崩保护阈值**：设 0~1 之间（如 0.6）。当 `健康实例数/总实例数 < 阈值` 时，Nacos 允许返回不健康实例（防止全流量打挂仅剩的健康节点）。

**配置中心三元组**：Namespace（环境：dev/test/prod）→ Group（项目分组）→ DataId（具体配置文件）。

优先级：精准配置（service-dev.yaml）> 同工程通用 > 扩展配置 > 共享配置。

**动态配置刷新**：`@Value` 不感知变更；加 `@RefreshScope` 后 Bean 在配置变更时重新初始化。Nacos 用**长轮询**推送变更，比 Spring Cloud Config + Bus 快，且内置可视化控制台。

## 微服务拆分原则

微服务的核心是"如何划定服务边界"，没有唯一标准，但有以下指导原则：

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

### 限流 / 降级 / 熔断的本质区别

三者都是稳定性策略，但触发时机和目的不同：

| 策略 | 触发时机 | 作用对象 | 目的 | 典型场景 |
|------|---------|---------|------|---------|
| **限流** | 流量超过阈值 | 当前服务入口 | 保护自身不被压垮 | 双十一大促限制每秒下单数 |
| **降级** | 系统负载高 / 非核心功能 | 主动关闭部分功能 | 保核心、舍次要 | 大促期间关闭退款入口 |
| **熔断** | 下游错误率持续超阈值 | 对下游的调用 | 快速失败，防级联崩溃 | 微信支付超时率激增时断开调用 |

- **限流**通常由**被调用方**对调用方设置（"我最多接受 1000 QPS"）
- **降级**通常由调用方主动执行（"下游慢，我返回兜底数据"）
- **熔断**发生在**调用方**，检测到下游异常后自动断路，一段时间后半开探测

### 部署策略：蓝绿 / 灰度 / 金丝雀

| 策略 | 机器资源 | 回滚速度 | 验证范围 | 适用场景 |
|------|---------|---------|---------|---------|
| **蓝绿部署** | 双倍（闲置一套）| 极快（切流量）| 全量前验证 | 稳定性高要求、预算充足 |
| **灰度/金丝雀** | 仅额外少量机器 | 较慢（需回退版本）| 小流量真实验证 | 常规发布首选；节省资源 |

**蓝绿部署**：Blue/Green 两套环境，流量全量切换，瞬间切回即回滚，但有 50% 机器常年闲置。

**金丝雀/灰度发布**：从集群中取少量机器先发布新版，逐步扩大流量比例（5% → 20% → 100%）。发现问题只需将问题机器下线即可，无需两套完整环境。

**快速回滚**：记录发布前基线（环境快照/容器镜像），回滚时基于基线重新部署，无需手工操作。

### Service Mesh（服务网格）

**核心思想**：将微服务间的通信（服务发现、负载均衡、熔断、限流、TLS、监控）从业务代码中抽离，下沉到每个服务实例旁的 **Sidecar 代理**（通常是 Envoy）。

```
服务 A  →  Sidecar(A)  →  网络  →  Sidecar(B)  →  服务 B
                           ↑
                      控制平面（Istio/Linkerd）统一下发策略
```

**解决的问题**：多语言异构环境下，Java 的 Sentinel、Go 的自研限流、Python 的 requests 各有一套，Service Mesh 提供统一治理，语言无关。

**典型能力**：流量镜像（把线上流量拷贝到测试环境）、精细路由（按 Header/百分比分流）、mTLS 服务间加密、全链路 tracing。

**代价**：每次请求多一跳（本机 loopback，延迟极低 <1ms）；控制平面（Istio）本身运维复杂。

### 循环依赖与解法

微服务循环依赖（A→B→C→A）危害：流量放大（QPS 倍增）、响应 RT 拉长、级联故障风险、发布顺序无法确定。

**解法优先级**：
1. **重新设计**：循环依赖往往是职责划分不清，先审视是否能合并或重新划分边界
2. **消息解耦**：同步调用改为 MQ 异步通知，断开强依赖
3. **抽取共享服务（中介服务）**：A 和 B 共同依赖 C（台账/数据服务），而不是互相调用

### CI/CD 与 DevOps

**CI（持续集成）**：开发者每次提交代码后自动构建 + 单元测试，快速判断本次变更是否破坏现有功能。

**CD（持续交付/部署）**：
- **持续交付**：CI 通过后自动部署到 Staging 环境，人工审批后发生产
- **持续部署**：全自动推到生产，无需人工干预

**DevOps**：一种文化/理念——开发（Dev）和运维（Ops）不再割裂，共同对软件全生命周期负责，由 CI/CD 自动化工具链支撑。核心是"谁构建谁运维"（You build it, you run it）。

## 关键权衡

1. **AP vs CP 注册中心**：注册中心对可用性要求高（服务发现临时不一致影响有限），选 Eureka/Nacos AP 模式；配置中心对一致性要求高，选 CP 模式
2. **客户端 LB vs 服务端 LB**：客户端 LB（LoadBalancer/Ribbon）无中间节点，性能好但每个 Consumer 都要感知 Provider 列表；Nginx 服务端 LB 集中管理，但多一跳延迟
3. **线程池隔离 vs 信号量隔离**：Hystrix 线程池隔离完全隔离故障但线程切换有开销；信号量隔离轻量但无法做超时控制（适合本地快速执行的逻辑）
4. **Feign vs RestTemplate**：Feign 声明式简洁，适合内部服务调用；RestTemplate 灵活，适合调用外部第三方 HTTP 接口（无服务注册）
5. **Spring Cloud vs Spring Cloud Alibaba**：Alibaba 套件（Nacos+Sentinel+RocketMQ+Seata）更适合国内生态；原版 Spring Cloud 更适合国际化云原生场景

## 与其他概念的关系

- **依赖 [[机制-RPC与Dubbo]]**：Dubbo 可作为 Spring Cloud 内部服务间调用的替代方案（OpenFeign HTTP → Dubbo TCP），两者互补而非互斥
- **依赖 [[机制-ZAB协议与Zookeeper]]**：Eureka/Nacos 之前，ZooKeeper 也被用作注册中心；ZK 的 CP 特性使其在服务注册场景中略显过重
- **依赖 [[机制-Spring]]**：OpenFeign 底层通过动态代理生成 Stub；Hystrix 通过 AOP 切面拦截方法调用并织入熔断逻辑
- **依赖 [[机制-SpringBoot]]**：Spring Cloud 各组件均通过 Spring Boot AutoConfiguration 自动装配，`@EnableEurekaClient`/`@EnableFeignClients` 等注解触发自动配置链路

## 应用边界

**适合 Spring Cloud**：Java 技术栈的微服务全栈解决方案；需要快速搭建注册中心/网关/配置中心；与 Spring Boot 深度集成的团队。

**不适合 Spring Cloud**：极高性能内部调用（考虑 Dubbo，HTTP 比 TCP RPC 多约 30-50% 开销）；非 Java 技术栈（HTTP 可互通，但配套组件均为 Java）；超大规模（Netflix 内部已放弃 Eureka，转向 Kubernetes 原生服务发现）。
