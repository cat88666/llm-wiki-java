---
type: concept
status: active
name: "配置中心与Nacos"
layer: L7
aliases: ["Nacos", "配置中心", "注册中心", "服务发现", "长轮询", "Distro协议", "JRaft", "Apollo", "动态配置", "推拉结合", "gRPC长连接", "雪崩保护", "保护阈值", "Namespace", "Group", "DataId", "RefreshScope", "配置优先级"]
tags: ["#distributed"]
related:
  - "[[机制-SpringCloud]]"
  - "[[机制-Zookeeper]]"
  - "[[概念-分布式理论]]"
sources:
  - "../../../raw/note/Hollis/配置中心/✅什么是Nacos，主要用来作什么？.md"
  - "../../../raw/note/Hollis/配置中心/✅配置中心的作用是什么？如何选型？.md"
  - "../../../raw/note/Hollis/配置中心/✅Nacos是AP的还是CP的？.md"
  - "../../../raw/note/Hollis/配置中心/✅Nacos能同时实现AP和CP的原理是什么？.md"
  - "../../../raw/note/Hollis/配置中心/✅Nacos如何实现的配置变化客户端可以感知到？.md"
  - "../../../raw/note/Hollis/配置中心/✅Nacos的服务注册和服务发现的过程是怎么样的？.md"
  - "../../../raw/note/Hollis/配置中心/✅Nacos 2 x为什么新增了RPC的通信方式？.md"
  - "../../../raw/note/Hollis/配置中心/✅注册中心如何选型？.md"
  - "../../../raw/note/tuling/09-微服务/11-nacos.md"
created: 2026-05-08
updated: 2026-05-16
lint_notes: ""
---

# 配置中心与 Nacos

> 配置中心将应用配置从代码中剥离、集中管理，并支持动态感知变更；Nacos 是阿里开源的配置中心 + 注册中心二合一平台，通过 CP（JRaft）+ AP（Distro）双模式同时满足配置一致性和注册高可用。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 配置中心 vs 注册中心各自的根本问题 |
| [二、配置中心](#二配置中心) | 动态 vs 静态配置分层管理；Namespace/Group/DataId；vs SpringCloud Config；配置优先级；@RefreshScope |
| [三、Nacos 注册中心机制](#三nacos-注册中心机制) | 服务注册/心跳/推拉结合发现；雪崩保护（保护阈值）；临时实例 vs 持久实例 |
| [四、配置动态感知](#四配置动态感知) | 1.x 长轮询 vs 2.x gRPC 长连接 |
| [五、AP + CP 双模式](#五ap--cp-双模式) | Distro（AP）+ JRaft（CP）各管一摊 |
| [六、注册中心选型对比](#六注册中心选型对比) | Nacos/Eureka/Consul/ZK 四维对比 |
| [七、关键权衡](#七关键权衡) | AP vs CP 实际影响，gRPC vs HTTP |
| [八、与其他概念的关系](#八与其他概念的关系) | SpringCloud、Zookeeper、分布式理论 |
| [九、应用边界](#九应用边界) | 配置中心适用范围，Nacos 适用规模 |

## 一、第一性原理

**配置中心解决的根本问题**：业务配置（开关、阈值、灰度比例）写死在代码或本地文件中，每次变更都需要发版，代价高且风险大。配置中心的核心价值是让配置**独立于代码、动态生效、集中可见**。

**注册中心解决的根本问题**：微服务集群中服务实例 IP/端口动态变化，调用方无法静态维护地址列表。注册中心充当"通讯录"，服务上下线时自动更新，调用方通过名字查地址。

Nacos 的洞察：**配置管理要 CP（不同节点配置不一致是灾难），注册中心要 AP（注册中心不可用比注册信息短暂不一致更致命）**，所以用两种协议各自保障。

## 二、配置中心

| 配置类型 | 推荐位置 | 原因 |
|---------|---------|------|
| 业务开关/灰度比例 | **配置中心** | 需要动态变更，不发版 |
| 运行时调优参数（线程池大小、超时时间）| **配置中心** | 需要在线调整 |
| 日志级别/采样率 | **配置中心** | 生产排查时动态调整 |
| 数据库连接 URL/密码 | 配置文件（或 Vault）| 基本不变，发版时固定 |
| 环境相关配置（dev/prod）| 配置文件（profiles）| Spring profiles 已支持 |

### 数据模型：Namespace / Group / DataId 三元组

Nacos 用三级维度唯一定位一份配置：

| 维度 | 含义 | 典型用法 |
|------|------|---------|
| **Namespace** | 租户/环境隔离 | 开发/测试/生产各一个 Namespace |
| **Group** | 项目/业务隔离 | 同一环境内区分不同项目（如 ORDER_GROUP） |
| **DataId** | 微服务配置文件 | 通常是 `${spring.application.name}.${file-extension}` |

默认值：Namespace = `public`，Group = `DEFAULT_GROUP`。三者共同确保不同环境/项目/服务间配置互不干扰。

### vs Spring Cloud Config

| 维度 | Nacos Config | Spring Cloud Config |
|------|-------------|---------------------|
| 配置存储 | 内置 DB（Derby/MySQL）| 依赖 Git 仓库 |
| 动态感知 | 长轮询（1.x）/ gRPC 推送（2.x），变更秒级生效 | 需配合 Spring Cloud Bus 通过 MQ 广播刷新 |
| 可视化界面 | 内置 UI，可在线编辑 | 无 |
| 复杂度 | 低，开箱即用 | 高，需 Git + Bus + MQ |

### 配置优先级（从高到低）

```
C：应用名 + Profile 自动生成的 DataId（nacos-config-product.yaml）
B：ext-config[n].data-id 扩展 DataId（不同工程扩展配置）
A：shared-configs 共享 DataId（不同工程通用配置）
```

优先级：**C（精准）> B（扩展）> A（共享）**，即 profile 精准配置 > 同工程通用 > 跨工程共享。同一个 key 多处定义时，高优先级覆盖低优先级。

### @RefreshScope — 动态刷新

`@Value` 注解读取配置中心的值后**无法感知后续变更**；需在 Controller/Service 类上加 `@RefreshScope`，配置变更后 Nacos 触发刷新，该 Bean 重新创建，`@Value` 字段获取到新值。

```java
@RestController
@RefreshScope  // 加上此注解，@Value 字段在配置变更时自动刷新
public class TestController {
    @Value("${common.age}")
    private String age;
}
```

> 注意：`@RefreshScope` 会重新创建 Bean，代价是一次 Bean 销毁+重建；高并发时可能短暂出现请求使用旧 Bean 的情况（Bean 刷新期间的并发窗口）。

## 三、Nacos 注册中心机制

**服务注册**：
```
服务实例启动 → Nacos Client 向 Server 发注册请求
  （服务名、IP、端口、集群名、元数据）
  → Server 存入 Derby/MySQL，同步到集群其他节点
  → 实例定期发心跳（默认 5s）
  → 15s 无心跳 → 标记不健康；30s 未收到 → 删除实例
```

**服务发现（推拉结合）**：
- **拉**：客户端首次全量拉取 + 每隔 10s 定时拉取（更新本地缓存）
- **推**：服务变更时 Server 主动推送（UDP / gRPC），客户端秒级感知
- **本地缓存**：即使 Nacos Server 宕机，客户端仍可用缓存提供服务（高可用兜底）

**雪崩保护（保护阈值）**：

当大量服务实例宕机，若只将健康实例返回给消费者，剩余健康实例会因流量激增被压垮，引发雪崩。保护阈值机制解决此问题：

```
保护阈值（0~1 之间的值，如 0.6）
健康实例数 / 总实例数 < 保护阈值（如 1/2 = 0.5 < 0.6）
→ 触发保护：不健康实例也加入服务列表
→ 消费者可能访问到不健康实例并快速失败
→ 但避免了剩余健康实例被打垮的更大灾难
```

**临时实例 vs 持久实例**：

| | 临时实例（默认）| 持久实例 |
|--|------|------|
| 配置 | `ephemeral=true`（默认）| `spring.cloud.nacos.discovery.ephemeral=false` |
| 宕机后 | 心跳超时 → 自动剔除 | 宕机不从服务列表删除，标记为不健康 |
| 适用 | 普通微服务实例 | 需要保留记录的服务（如基础设施节点）|

## 四、配置动态感知

**Nacos 1.x：长轮询（Long Polling）**
```
Client 发送 HTTP 请求（hold 30s）
  → Server 无变化则挂起
  → 配置变更时立即响应
  → Client 收到响应后立即再发下一次长轮询
```
优于短轮询（减少空轮询），优于长连接（减少连接维护成本）。但本质仍是 HTTP，每 30s 一次上下文切换，频繁 GC。

**Nacos 2.x：gRPC 长连接**
- 真实长连接，消除 TIME_WAIT 堆积和频繁 GC
- 客户端不再需要定期心跳，只需 keepalive
- 服务端变更后通过 Stream 实时推送，延迟从 30s 降至毫秒级

## 五、AP + CP 双模式

| 场景 | 协议 | 原因 |
|------|------|------|
| 注册中心（临时实例）| Distro（AP）| 可用性优先，注册信息短暂不一致可接受 |
| 配置中心（持久实例）| JRaft（CP）| 强一致优先，不同节点配置不同是灾难 |

**Distro**（自研 AP 协议）：每个节点平等、负责部分数据、互相同步校验值，读请求本地响应，类似 Dynamo 风格。

**JRaft**（基于 Raft）：Leader 写、多数派确认、强一致但写入有延迟；与 [[机制-Zookeeper]] 的 ZAB 协议思想相同，都是 Paxos 变体。

## 六、注册中心选型对比

| 特性 | Nacos | Eureka | Consul | Zookeeper |
|------|-------|--------|--------|-----------|
| CAP | CP+AP | AP | CP | CP |
| 健康检查 | TCP/HTTP/Client Beat | Client Beat | TCP/HTTP/gRPC | Keep Alive |
| 配置中心 | 内置 | 无 | KV 弱支持 | 需自研 |
| 可视化 UI | 有 | 有 | 有 | 无 |
| Spring Cloud 集成 | 支持 | 支持 | 支持 | 支持 |
| Dubbo 集成 | 支持 | 不支持 | 支持 | 支持 |
| 维护状态 | 活跃（阿里）| 停维（Netflix 2.0）| 活跃 | 活跃 |

**选型建议**：Spring Cloud Alibaba / Dubbo 体系 → Nacos；需要强健康检查和多数据中心 → Consul；已有 Zookeeper 基础设施 → ZK；避免继续新建 Eureka（已停维）。

## 七、关键权衡

1. **AP vs CP 的实际影响**：注册中心用 AP，意味着某节点短暂宕机后还会被调用方发现并调用，调用失败后触发重试——可接受；配置中心用 CP，意味着网络分区时配置写入会被拒绝——可接受（配置变更远少于读操作）
2. **Nacos 2.x gRPC vs 1.x 长轮询**：gRPC 减少了 GC 和 TIME_WAIT，但可观测性不如 HTTP，调试相对困难；2.x 是推荐版本
3. **所有配置放配置中心 vs 分层管理**：全部放配置中心简化运维，但静态配置（如密码）频繁读配置中心增加无谓网络开销；最佳实践：动态配置用配置中心，静态配置用 Spring profiles

## 八、与其他概念的关系

- 支撑 [[机制-SpringCloud]]：Nacos 是 Spring Cloud Alibaba 的注册中心和配置中心核心组件，替代 Eureka + Spring Cloud Config 的组合
- 关联 [[机制-Zookeeper]]：Nacos CP 模式（JRaft）和 Zookeeper（ZAB）都基于 Raft/Paxos 思想实现强一致性，但 Nacos 同时支持 AP，场景更全面
- 依赖 [[概念-分布式理论]]：AP/CP 选择是 CAP 定理的直接应用，Distro 协议体现了最终一致性设计

## 九、应用边界

**适合用配置中心**：需要动态变更且不发版的参数（开关、比例、阈值）；多服务共享配置（公共数据源、MQ地址）。

**不适合放配置中心**：基本不变的环境相关配置（用 Spring profiles 更简单）；含高度敏感信息的密钥（考虑专用 Secret 管理工具如 Vault、K8s Secret）。

**Nacos 适用规模**：注册中心支持百万级实例；单集群建议 3 或 5 节点（奇数 Raft 要求）；超大规模可考虑多集群联邦。
