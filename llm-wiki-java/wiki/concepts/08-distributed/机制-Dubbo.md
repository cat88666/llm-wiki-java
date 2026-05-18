---
type: concept
status: active
name: "Dubbo"
layer: L7
aliases: ["Dubbo", "RPC", "远程过程调用", "服务治理", "负载均衡", "集群容错", "优雅停机", "Dubbo SPI", "gRPC", "Protobuf", "序列化协议", "HTTP/2", "服务导出", "服务引入", "平滑加权轮询", "Directory", "Invoker", "Exporter"]
related:
  - "[[机制-Zookeeper]]"
  - "[[机制-动态代理]]"
  - "[[机制-Java序列化]]"
  - "[[机制-SPI]]"
  - "[[机制-SpringBoot]]"
---

# Dubbo

> RPC（Remote Procedure Call）让调用方像调用本地方法一样调用远程服务，屏蔽了序列化、网络传输、负载均衡等细节；Dubbo 是阿里开源的高性能 Java RPC 框架，在 RPC 基础上提供了完整的服务治理体系。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、RPC 透明调用的本质](#一rpc-透明调用的本质) | RPC 存在的根本原因：远程调用透明化 |
| [二、RPC vs HTTP](#二rpc-vs-http) | 协议差异、适用场景边界 |
| [三、Dubbo 整体架构](#三dubbo-整体架构) | 9层分层架构（各层关键组件）、三阶段流程、Protocol/Invoker/Exporter 三件套 |
| [四、服务导出与引入流程](#四服务导出与引入流程) | Provider 端 5 步导出、Consumer 端 4 步引入 |
| [五、透明代理机制](#五透明代理机制) | 动态代理 Stub 生成，像本地方法一样调用 |
| [六、负载均衡策略](#六负载均衡策略) | 5种策略及选型场景；平滑加权轮询 |
| [七、通信协议与序列化](#七通信协议与序列化) | dubbo/Hessian2/Protobuf/Fury 对比 |
| [八、Dubbo SPI 增强](#八dubbo-spi-增强) | vs JDK SPI：懒加载、AOP、@Activate |
| [九、服务治理体系](#九服务治理体系) | 集群容错、流量管控、降级、动态配置 |
| [十、优雅停机](#十优雅停机) | SIGTERM → JVM hook → Dubbo 有序收尾 |
| [十一、gRPC 与序列化协议对比](#十一grpc-与序列化协议对比) | gRPC vs Dubbo，4种通信模式 |
| [十二、Dubbo 工程权衡](#十二dubbo-工程权衡) | RPC vs HTTP、序列化、LB 方式 |
| [十三、Dubbo 的生态位置](#十三dubbo-的生态位置) | Zookeeper、动态代理、SPI、序列化 |
| [十四、Dubbo 适用边界](#十四dubbo-适用边界) | 适合 vs 不适合场景 |

## 一、RPC 透明调用的本质

分布式系统中，不同服务部署在不同机器上，服务间通信必然涉及网络传输——如果每次调用都要开发者手动处理序列化、连接管理、重试、负载均衡，开发效率极低且极易出错。**RPC 框架的存在理由：让远程调用像本地调用一样透明，同时把分布式通信的复杂性（连接池、序列化、服务发现、容错）统一在框架层解决。**

## 二、RPC vs HTTP

RPC 和 HTTP 不是同一维度的概念：HTTP 是传输协议，RPC 是远程调用范式（需要传输协议 + 序列化协议）。RPC 可以基于 TCP 也可以基于 HTTP 实现。

| 维度 | RPC（如 Dubbo） | HTTP（如 REST） |
|------|----------------|----------------|
| 适用场景 | 内部服务间调用 | 对外异构环境、浏览器、第三方 |
| 连接方式 | 长连接（复用） | 短连接（每次握手） |
| 协议头 | 精简二进制头 | 文本头（HTTP Header，较重）|
| 序列化 | 高效二进制（Hessian2/Protobuf）| JSON/XML（可读但体积大）|
| 服务治理 | 内建（注册中心、LB、容错）| 需外部组件（Nginx、熔断器）|

**只能用 HTTP 不能用 RPC 的场景**：对外开放的 OpenAPI、跨语言异构系统（浏览器/APP）、需要防火墙穿透（80/443 端口）。

## 三、Dubbo 整体架构

```
Consumer（消费者）
    ↓ 订阅服务列表
Registry（注册中心：ZK / Nacos）← Provider 启动时注册
    ↓ 推送变更通知
Consumer → Cluster（聚合 Invoker 列表）→ LoadBalance → Invoker
    ↓ 序列化 + Netty
Provider（服务端）→ 反序列化 → 业务方法执行 → 结果返回
```

**三阶段**：
1. **服务注册**：Provider 启动时向注册中心注册（服务名、IP、Port、协议、权重等）
2. **服务发现**：Consumer 订阅服务，获取 Provider 列表，列表变更时收到推送
3. **服务调用**：Consumer 按负载均衡策略选择 Provider，发起 RPC 调用

**Dubbo 分层架构（由上到下）**：

| 层 | 职责 | 关键组件 |
|----|------|---------|
| config | ServiceConfig / ReferenceConfig 配置解析，Spring/XML 参数初始化 | ServiceConfig, ReferenceConfig |
| proxy | 动态代理生成 Stub（JDK 动态代理 / Javassist），Invoker ↔ 接口互转 | ProxyFactory |
| registry | 服务注册、发现、订阅、变更通知 | RegistryFactory, Registry, RegistryService |
| cluster | 多 Provider 路由 + 负载均衡 + 容错；将多 Invoker 聚合成一个 | Cluster, **Directory**, Router, LoadBalance |
| monitor | 调用次数/耗时统计（类似 Prometheus） | MonitorFactory, Monitor |
| protocol | **核心**：RPC 调用封装，只需 Protocol + Invoker + Exporter 即可完成 RPC | **Protocol, Invoker, Exporter** |
| exchange | 封装 request/response 语义，同步转异步 | Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer |
| transport | 统一网络框架抽象（Netty / Mina / Grizzly 可插拔） | Transporter, Client, Server, Codec |
| serialize | 序列化/反序列化（Hessian2 / JSON / FastJson2 / Kryo / FST）| Serialization, ObjectInput, ObjectOutput |

**RPC 最小三件套**：Protocol + Invoker + Exporter 是完成 RPC 调用的核心，其余各层是增强功能（容错/监控/路由等）。Cluster 层将多个 Invoker 聚合成一个，Proxy 层将 Invoker 转换成透明的本地 Java 接口。

## 四、服务导出与引入流程

### Provider 端：服务导出（5步）

```
1. 解析 @DubboService / @Service 注解
   → 读取服务名、接口、协议、超时、版本等 → 构建 ServiceBean

2. 调用 ServiceBean.export() 进入服务暴露流程

3. 注册服务到注册中心（ZK / Nacos / Redis 等）
   → 若配置多个协议/多个注册中心 → 对每个协议×每个注册中心分别注册

4. 绑定动态配置监听器
   → 监听配置中心下发的路由规则/限流/超时变更

5. 启动协议对应的网络服务器
   → dubbo 协议 → Netty Server
   → HTTP 协议  → Tomcat / Jetty
```

### Consumer 端：服务引入（4步）

```
1. 解析 @Reference 注解
   → 读取服务名、接口、负载均衡策略、超时等配置

2. 从注册中心查询 Provider 列表
   → 将 Provider 列表封装为 Directory（服务目录，动态维护可用节点）

3. 注册监听器
   → 监听路由变化、动态配置变更、Provider 上下线通知

4. 基于 Provider 列表创建代理对象
   → ProxyFactory 生成接口代理 → 注入 Spring 容器供业务代码调用
```

## 五、透明代理机制

Dubbo 使用**动态代理**（见 [[机制-动态代理]]）生成 Consumer 端的 Stub：

```
@Reference OrderService orderService;  // 注入的是代理对象
orderService.createOrder(req);
    ↓ 代理拦截
序列化请求（方法名 + 参数） → Netty 长连接发送
    ↓ Provider 端
反序列化 → 调用真实实现类 → 序列化结果 → 返回
```

## 六、负载均衡策略

Dubbo 是**客户端负载均衡**，Consumer 自行选择 Provider。

| 策略 | 算法 | 适用场景 |
|------|------|---------|
| Random（默认）| 加权随机 | 各节点处理能力相近 |
| RoundRobin | 加权轮询（底层采用**平滑加权轮询算法**，避免权重高的节点连续命中）| 稳定负载环境 |
| LeastActive | 活跃调用数最少者优先 | 响应时间差异大，防慢节点堆积 |
| ShortestResponse | 最近响应时间最短者优先 | 资源消耗波动大的服务 |
| ConsistentHash | 一致性哈希，同参数→同节点 | 有状态服务、会话亲和性（高频用户/同会话绑定/游戏分区）|

## 七、通信协议与序列化

**默认协议**：`dubbo` 协议（TCP 长连接 + Hessian2 序列化 + NIO 异步），适合高并发小数据量（< 100KB/次）。

| 协议 | 传输 | 适用场景 |
|------|------|---------|
| dubbo（默认）| TCP 长连接 | 内部高并发调用 |
| HTTP | HTTP 短连接 | 异构系统互通 |
| Hessian | HTTP | 大文件传输 |
| RESTful | HTTP/HTTPS | OpenAPI 对外暴露 |

**序列化对比**：

| 协议 | 性能 | 跨语言 | 可读性 | 典型场景 |
|------|------|--------|--------|---------|
| JSON | 低 | ✓ | 高 | HTTP API、配置文件 |
| Hessian2 | 中 | 部分 | 低 | Dubbo 默认 |
| **Protobuf** | 高 | ✓ | 低（需 IDL）| gRPC、Kafka Schema |
| Kryo | 极高 | ✗（Java 专用）| 低 | Java 内部高性能缓存序列化 |
| Fury | 极高 | ✓ | 低 | 蚂蚁开源，号称比 Hessian2 快 100 倍 |
| Avro | 高 | ✓ | 低 | Kafka、Hadoop 大数据 |

**选型原则**：对外接口用 JSON；内部 Java 服务用 Hessian2/Fury；跨语言高性能用 Protobuf；大数据管道用 Avro。

## 八、Dubbo SPI 增强

Dubbo 对 JDK SPI（见 [[机制-SPI]]）做了增强：

| 特性 | JDK SPI | Dubbo SPI |
|------|---------|-----------|
| 加载方式 | 预加载全部实现 | **懒加载**（按需加载） |
| 依赖注入 | 不支持 | 支持（自动注入其他扩展点）|
| AOP | 不支持 | 支持 Wrapper 包装（类似装饰器）|
| 条件激活 | 不支持 | `@Activate` 按条件自动激活 |
| 配置格式 | 类名列表 | `key=value`（键值对，可按名获取）|
| 配置路径 | `META-INF/services/` | `META-INF/dubbo/` |

Dubbo 内部所有扩展点（Protocol/LoadBalance/Cluster/Serialization 等）均通过 `ExtensionLoader` 实现。

## 九、服务治理体系

- **集群容错**：Failover（失败重试其他节点，默认）、Failfast（失败立即报错）、Failsafe（失败忽略）、Failback（失败后台重试）
- **流量管控**：A/B 测试、金丝雀发布、多版本按比例分流、黑白名单
- **服务降级**：Provider 异常时返回 mock 结果，保证 Consumer 可用
- **动态配置**：运行期通过配置中心调整超时时间、重试次数、权重等
- **可视化**：Dubbo Admin 控制台监控集群流量和服务状态

## 十、优雅停机

```
JVM kill -15 (SIGTERM)
    ↓ JVM shutdown hook
Spring ContextClosedEvent（Spring 监听到容器关闭）
    ↓ Dubbo 监听 Spring 事件
Provider: 标记不接受新请求，等待处理中请求完成（默认等待 10s）
Consumer: 不再发起新调用，等待响应中的请求返回
```

核心：利用 JVM shutdown hook → Spring 事件机制 → Dubbo 有序收尾，避免强杀（kill -9）导致请求丢失。

## 十一、gRPC 与序列化协议对比

Google 开源的高性能 RPC 框架，基于 **HTTP/2 + Protobuf**：

| 特性 | gRPC | Dubbo |
|------|------|-------|
| 传输协议 | HTTP/2（多路复用/头部压缩）| TCP 长连接（dubbo 协议）|
| 序列化 | Protobuf（二进制，强类型 IDL）| Hessian2（默认），可插拔 |
| 跨语言 | 一流（Go/Python/C++/Java/...）| Java 为主，多语言弱 |
| 服务治理 | 无（需自建或 Service Mesh）| 丰富（路由/降级/监控）|
| 通信模式 | Unary/Server Streaming/Client Streaming/双向 Streaming | 请求-响应为主 |

**gRPC 四种通信模式**：
- **Unary**：一次请求 + 一次响应（普通 RPC）
- **Server Streaming**：一次请求 + 多次响应（如实时推送）
- **Client Streaming**：多次请求 + 一次响应（如文件上传）
- **Bidirectional Streaming**：双向流式（如即时聊天）

**适用场景**：微服务内部跨语言通信；对性能极致要求（Protobuf 比 JSON 小 3-10倍）；需要流式传输。

## 十二、Dubbo 工程权衡

1. **RPC vs HTTP**：内部服务优先 RPC（性能高、治理能力强）；对外接口用 HTTP（互通性好、防火墙友好）
2. **序列化性能 vs 跨语言**：Hessian2 跨语言性能好是默认选择；Protobuf 更高性能但需 IDL；Fury 性能极致但生态较新
3. **客户端 LB vs 服务端 LB**：Dubbo 是客户端 LB，无中心，性能好但 Consumer 需感知 Provider 列表；Nginx 是服务端 LB，Consumer 无感知但多一跳延迟
4. **Failover vs Failfast**：Failover 提高可用性但可能放大写操作副作用（重试导致重复写）；幂等操作用 Failover，非幂等用 Failfast
5. **长连接复用 vs 连接数**：dubbo 协议单一长连接高并发性能好，但高权重节点需调整连接数

## 十三、Dubbo 的生态位置

- **依赖 [[机制-Zookeeper]]**：Dubbo 默认使用 ZooKeeper 作为注册中心（利用 ZK 临时节点做服务上下线感知，Watch 机制做变更通知）
- **依赖 [[机制-动态代理]]**：Consumer 端 Stub 通过 JDK 动态代理或 Javassist 字节码生成，实现调用透明化
- **依赖 [[机制-SPI]]**：Dubbo SPI 是 JDK SPI 的增强版，支撑了 Dubbo 所有扩展点（Protocol/LB/Serialization 等）
- **依赖 [[机制-Java序列化]]**：RPC 跨进程调用必须序列化，Dubbo 默认 Hessian2，可插拔替换
- **与 [[机制-SpringBoot]] 协作**：Dubbo Spring Boot Starter 借助 AutoConfiguration 自动注册 DubboConfig Bean

## 十四、Dubbo 适用边界

**适合 Dubbo**：Java 技术栈的内部微服务间调用；需要丰富服务治理（路由、降级、监控）；ZooKeeper/Nacos 注册中心已有沉淀。

**不适合 Dubbo**：对外 OpenAPI（用 Spring MVC + HTTP）；跨语言异构系统（考虑 gRPC + Protobuf）；前端/移动端直接调用（HTTP/HTTPS 更合适）；团队技术栈为非 Java（Dubbo 多语言支持弱于 gRPC）。
