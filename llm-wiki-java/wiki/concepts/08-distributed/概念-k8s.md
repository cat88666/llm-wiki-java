---
type: concept
status: active
name: "Kubernetes"
layer: L7
aliases: ["K8s", "Kubernetes", "Pod", "Deployment", "ReplicaSet", "Service", "Ingress", "HPA", "PV", "PVC", "StorageClass", "CSI", "CNI", "kube-proxy", "kubelet", "etcd", "Helm"]
tags: ["#distributed", "#devops"]
related:
  - "[[机制-Docker]]"
  - "[[概念-分布式理论]]"
  - "[[概念-可观测性]]"
  - "[[机制-ElasticSearch]]"
created: 2026-05-16
updated: 2026-05-16
lint_notes: ""
---

# Kubernetes

> Kubernetes 是容器集群编排平台：它把多台机器抽象成一个资源池，以声明式 API 管理 Pod、Service、Deployment、存储、网络与发布流程，解决容器在生产环境中的调度、服务发现、故障自愈、弹性伸缩和滚动升级问题。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | K8s 解决多机容器编排，而不是单机容器运行 |
| [二、整体架构](#二整体架构) | 控制面、工作节点、etcd、API Server、Scheduler、Controller、kubelet |
| [三、核心对象](#三核心对象) | Pod、Label、Deployment、ReplicaSet、DaemonSet、Job、Namespace |
| [四、Pod 生命周期](#四pod-生命周期) | 创建流程、状态、重启策略、探针、init container |
| [五、调度与弹性伸缩](#五调度与弹性伸缩) | Request/Limit、nodeSelector、亲和性、污点容忍、HPA |
| [六、服务发现与网络](#六服务发现与网络) | Service、Ingress、kube-proxy、CNI、NetworkPolicy、Flannel、Calico |
| [七、存储](#七存储) | Volume、PV/PVC、StorageClass、CSI |
| [八、安全](#八安全) | Authentication、Authorization、Admission、RBAC、Secret、Pod 安全策略 |
| [九、发布、运维与生态](#九发布运维与生态) | 滚动更新、DaemonSet、Metrics Server、EFK、drain、Helm |
| [十、关键权衡](#十关键权衡) | K8s 引入成本、Service 暴露方式、iptables vs IPVS、存储动态供给 |
| [十一、应用边界](#十一应用边界) | 适合场景与不适合场景 |

## 一、第一性原理

Docker 解决的是**单机容器标准化运行**：镜像怎么构建、容器怎么启动、依赖怎么隔离。生产环境真正难的是多机问题：

| 生产问题 | K8s 的抽象 |
| --- | --- |
| 容器挂了谁来拉起 | kubelet + Controller |
| 多副本怎么维持 | ReplicaSet / Deployment |
| 请求怎么找到后端 Pod | Service / Endpoints / CoreDNS |
| 节点资源怎么分配 | Scheduler + Requests |
| 发布怎么逐步替换 | Deployment RollingUpdate |
| 负载变高怎么扩容 | HPA |
| 配置、密钥、存储怎么管理 | ConfigMap / Secret / PV / PVC |

K8s 的核心思想是**声明式控制循环**：用户提交期望状态，控制器持续观察实际状态，并不断把实际状态修正到期望状态。

```
用户声明 desired state
        |
        v
API Server -> etcd
        |
        v
Controller / Scheduler / kubelet 持续协调
        |
        v
实际集群状态逐步逼近期望状态
```

## 二、整体架构

### 控制面组件

| 组件 | 作用 |
| --- | --- |
| **API Server** | 集群统一入口，提供 REST API；所有组件都通过它读写资源 |
| **etcd** | 高可用强一致 KV 存储，保存集群资源与状态，底层基于 Raft |
| **Scheduler** | 监听未绑定 Node 的 Pod，按过滤和打分策略选择合适节点 |
| **Controller Manager** | 运行多种控制器，维护副本数、节点状态、Endpoint、PV/PVC 等 |

API Server 是通信中心：kubelet 上报节点和 Pod 状态，Scheduler 监听待调度 Pod，Controller 监听资源变化并发起修正动作，所有状态最终写入 etcd。

### 工作节点组件

| 组件 | 作用 |
| --- | --- |
| **kubelet** | 每个 Node 上的代理，负责创建和管理本机 Pod，并上报节点状态 |
| **kube-proxy** | 监听 Service/Endpoint 变化，在节点上维护转发规则 |
| **容器运行时** | 负责拉取镜像、启动容器，现代集群常用 containerd |
| **cAdvisor** | 集成在 kubelet 中，采集节点和容器资源指标 |

## 三、核心对象

| 对象 | 本质 | 高频考点 |
| --- | --- | --- |
| **Pod** | K8s 最小调度单元，内部容器共享网络命名空间和 Volume | 一个 Pod 可含多个强相关容器 |
| **Label / Selector** | 资源分组与筛选机制 | Service、Deployment 通过 selector 管理 Pod |
| **ReplicaSet** | 保证指定数量 Pod 副本存在 | 使用集合式 selector |
| **ReplicationController** | 早期副本控制器 | 能力弱于 ReplicaSet，常被 Deployment 间接替代 |
| **Deployment** | 无状态应用发布与副本管理 | 通过 ReplicaSet 实现滚动更新和回滚 |
| **DaemonSet** | 每个符合条件的 Node 运行一个 Pod | 日志采集、监控 Agent、网络插件 |
| **Job / CronJob** | 一次性任务 / 定时任务 | 适合批处理，不适合常驻服务 |
| **StatefulSet** | 有状态应用管理 | 稳定网络标识、稳定存储 |
| **Namespace** | 逻辑隔离边界 | 多团队/多环境资源隔离 |

Pod 内的多个容器共享同一个 Pod IP，可以通过 `localhost` 通信；这也是 Sidecar 模式的基础。

## 四、Pod 生命周期

### 创建流程

1. 用户通过 `kubectl` 或 API 提交 Pod/Deployment YAML 到 API Server。
2. API Server 校验请求并写入 etcd。
3. Controller 根据 Deployment 等控制器对象创建 Pod。
4. Scheduler 监听到未调度 Pod，过滤和打分后绑定到某个 Node。
5. 目标 Node 上的 kubelet 监听到绑定事件，拉取镜像并启动容器。
6. kubelet 持续上报 Pod 状态，Controller 根据实际状态继续修正。

### Pod 状态

| 状态 | 含义 |
| --- | --- |
| **Pending** | Pod 已创建，但容器尚未全部启动，可能在调度或拉镜像 |
| **Running** | Pod 已绑定节点，至少一个容器正在运行或启动/重启 |
| **Succeeded** | 所有容器成功退出，不再重启 |
| **Failed** | 所有容器退出，至少一个失败 |
| **Unknown** | 控制面无法获取 Pod 状态，常见于节点通信异常 |

### 重启策略

| 策略 | 含义 | 常见控制器 |
| --- | --- | --- |
| **Always** | 容器退出即重启 | Deployment、DaemonSet |
| **OnFailure** | 非 0 退出码才重启 | Job |
| **Never** | 不重启 | Job、调试任务 |

### 健康检查

| 探针 | 判断问题 | 失败后果 |
| --- | --- | --- |
| **startupProbe** | 应用是否完成启动 | 启动期失败不会被 liveness 误杀 |
| **readinessProbe** | 是否可以接流量 | 从 Service Endpoints 移除 |
| **livenessProbe** | 是否仍然存活 | kubelet 杀掉容器并按策略重启 |

探针方式包括 `exec`、`tcpSocket`、`httpGet`。Java 服务通常使用 `/actuator/health/readiness` 和 `/actuator/health/liveness` 分离接流量与存活判断。

### Init Container

Init Container 在业务容器前串行执行，全部成功后才启动业务容器。常用于等待依赖、初始化配置、迁移轻量数据。不要在 Init Container 中放长时间常驻逻辑。

## 五、调度与弹性伸缩

### 调度过程

Scheduler 的调度可以理解为两阶段：

| 阶段 | 作用 |
| --- | --- |
| **过滤** | 排除资源不足、不满足 selector/亲和性/污点约束的节点 |
| **打分** | 对剩余节点按资源、拓扑、亲和性等策略评分，选最高分节点 |

### 调度约束

| 机制 | 作用 | 适用场景 |
| --- | --- | --- |
| **nodeSelector** | 通过 Node 标签定向调度 | 简单固定节点选择 |
| **nodeAffinity** | 节点亲和性，支持硬规则和软规则 | 多条件、带权重的调度 |
| **podAffinity / podAntiAffinity** | Pod 间亲和/反亲和 | 同机部署或打散部署 |
| **Taint / Toleration** | 节点排斥 Pod，Pod 声明可容忍 | 专用节点、故障隔离 |

### Request 与 Limit

| 字段 | 作用 |
| --- | --- |
| **requests** | 调度依据，Scheduler 保证节点 requests 总和不超过可分配资源 |
| **limits** | 运行上限，限制容器最大 CPU/内存使用 |

生产中不能只写 limits 不写 requests，否则调度和容量评估会失真。内存超过 limit 通常触发 OOMKill；CPU 超过 limit 通常被 throttling。

### HPA

HPA 通过 Metrics Server 或自定义指标周期性获取负载数据，计算目标副本数，并对 Deployment/ReplicaSet 发起 scale 操作。常见指标是 CPU、内存，也可以接 Prometheus Adapter 使用 QPS、队列积压等业务指标。

## 六、服务发现与网络

### Service 类型

| 类型 | 访问范围 | 典型用途 |
| --- | --- | --- |
| **ClusterIP** | 集群内访问 | 内部微服务 |
| **NodePort** | 通过任意 Node IP + 端口访问 | 测试、简单暴露 |
| **LoadBalancer** | 云厂商 LB 暴露 | 公有云生产入口 |
| **Headless Service** | 不分配 ClusterIP，DNS 返回 Pod 列表 | StatefulSet、客户端自定义负载均衡 |

Service 通过 selector 关联 Pod，Endpoint/EndpointSlice 记录真实后端。默认分发通常近似轮询，也可通过 SessionAffinity 做基于客户端 IP 的会话保持。

### Ingress

Ingress 提供七层 HTTP/HTTPS 路由规则，Ingress Controller 负责把规则转化为真实转发配置。典型链路：

```
Client -> LoadBalancer/NodePort -> Ingress Controller -> Service -> Pod
```

Ingress 适合统一域名、路径路由、TLS 终止；非 HTTP 流量通常需要 LoadBalancer、NodePort 或网关类产品。

### kube-proxy

| 模式 | 原理 | 特点 |
| --- | --- | --- |
| **iptables** | 监听 Service/Endpoint，生成 NAT 规则 | 简单稳定，但规则量大时线性匹配成本高 |
| **IPVS** | 基于 Netfilter + IPVS 负载均衡 | 性能和扩展性更好，支持更多调度算法 |

IPVS 面向高性能负载均衡，使用更高效的数据结构；大规模集群通常更偏向 IPVS 或 eBPF 类网络方案。

### K8s 网络模型

K8s 假设每个 Pod 有独立 IP，所有 Pod 处于可互通的扁平网络中。用户不需要手动为 Pod 做端口映射，也不需要关心跨节点 Pod 如何互通，这部分由 CNI 插件实现。

| 组件/机制 | 作用 |
| --- | --- |
| **CNI** | 容器网络接口规范，负责创建/删除容器网络资源 |
| **IPAM** | IP 地址分配与管理 |
| **NetworkPolicy** | Pod 间网络访问控制，需要 CNI 支持 |
| **Flannel** | 常见 Overlay 网络，为跨节点容器分配和封装转发网络 |
| **Calico** | 基于三层路由/BGP 的网络方案，也支持 NetworkPolicy |

## 七、存储

### Volume 类型

| 类型 | 生命周期 | 适用场景 |
| --- | --- | --- |
| **emptyDir** | 跟随 Pod，Pod 删除则删除 | 临时文件、同 Pod 多容器共享 |
| **hostPath** | 绑定到宿主机目录 | 节点级 Agent，谨慎用于业务数据 |
| **PV/PVC** | 独立于 Pod | 有状态数据持久化 |

hostPath 会增加 Pod 与节点耦合，业务数据更推荐通过 PV/PVC 使用网络存储或云盘。

### PV / PVC / StorageClass

| 对象 | 作用 |
| --- | --- |
| **PV** | 集群级存储资源抽象，由管理员或动态供给创建 |
| **PVC** | 用户对存储资源的申请 |
| **StorageClass** | 描述动态供给的存储类型和参数 |

PV 生命周期包括 `Available`、`Bound`、`Released`、`Failed`。存储供给分静态和动态两种：静态由管理员预建 PV，动态由 StorageClass 自动创建和绑定。

### CSI

CSI 是容器存储接口标准，使存储厂商插件与 Kubernetes 核心解耦。常见组件：

| 组件 | 作用 |
| --- | --- |
| **CSI Controller** | 管理卷创建、删除、挂载控制面动作 |
| **CSI Node** | 在具体 Node 上执行挂载、卸载等节点动作 |

## 八、安全

K8s 安全链路通常是：

```
Authentication -> Authorization -> Admission -> 持久化/执行
```

| 机制 | 作用 |
| --- | --- |
| **Authentication** | 认证调用者身份，常见方式有证书、Token、ServiceAccount |
| **Authorization** | 判断是否有权限，生产常用 RBAC |
| **Admission Controller** | 准入控制，在持久化前校验或修改对象 |
| **Secret** | 保存密码、Token、镜像拉取凭据等敏感信息 |
| **ServiceAccount** | Pod 访问 API Server 的身份 |

RBAC 由 Role/ClusterRole 与 RoleBinding/ClusterRoleBinding 组成，优势是覆盖资源和非资源权限，可通过 API 动态调整，不需要重启 API Server。

Secret 可通过环境变量、Volume 挂载、`imagePullSecrets` 使用。敏感配置不要写入镜像或普通 ConfigMap。

## 九、发布、运维与生态

### Deployment 发布策略

| 策略 | 行为 | 适用场景 |
| --- | --- | --- |
| **Recreate** | 先删除旧 Pod，再创建新 Pod | 不允许新旧版本共存 |
| **RollingUpdate** | 逐步扩新缩旧 | 默认策略，适合大多数无状态服务 |

RollingUpdate 通过 `maxSurge` 和 `maxUnavailable` 控制最多新增和最多不可用副本数。发布失败时可以回滚到旧 ReplicaSet。

### DaemonSet 场景

DaemonSet 保证每个符合条件的节点运行一个 Pod，常用于：

- 日志采集：Fluentd、Filebeat
- 监控采集：Node Exporter
- 网络插件：Calico、Flannel
- 存储节点插件：CSI Node

### 运维组件与动作

| 项 | 作用 |
| --- | --- |
| **Metrics Server** | 提供 Node/Pod CPU、内存核心指标，支撑 HPA |
| **EFK** | Elasticsearch 存储检索日志，Fluentd 采集日志，Kibana 展示日志 |
| **kubectl drain** | 节点维护前驱逐 Pod，避免直接关机导致服务抖动 |
| **Helm** | K8s 包管理工具，用 Chart 管理一组资源模板、版本、升级和回滚 |

EFK 通常通过 DaemonSet 在每个 Node 部署 Fluentd，采集容器日志并写入 Elasticsearch。

## 十、关键权衡

| 问题 | 取舍 |
| --- | --- |
| **是否引入 K8s** | 多服务、多副本、多环境、弹性和发布治理需求强时收益明显；简单单体应用可能运维成本大于收益 |
| **ClusterIP / NodePort / LoadBalancer / Ingress** | 内部服务用 ClusterIP；临时暴露可用 NodePort；云上四层入口用 LoadBalancer；HTTP 多域名/路径路由用 Ingress |
| **iptables vs IPVS** | iptables 简单成熟；IPVS 更适合大规模 Service/Endpoint 和复杂负载均衡 |
| **requests/limits 怎么设** | requests 决定调度与容量，limits 决定运行上限；Java 服务需特别关注内存 limit 与 JVM 参数匹配 |
| **静态 PV vs 动态 StorageClass** | 静态 PV 可控但运维重；动态供给自动化更强，适合云盘/NFS/CSI 标准化环境 |
| **Deployment vs StatefulSet** | 无状态服务用 Deployment；需要稳定身份、稳定存储、顺序发布的服务用 StatefulSet |
| **Readiness vs Liveness** | readiness 控制是否接流量；liveness 控制是否重启，不能把慢启动误判成死亡 |

## 十一、应用边界

适合引入 K8s 的场景：

- 服务数量多，需要统一部署、扩缩容、滚动发布和回滚。
- 多副本高可用，节点故障时需要自动迁移和自愈。
- 需要统一服务发现、配置、密钥、日志、监控和网络策略。
- 团队具备 DevOps 能力，能够维护集群、镜像仓库、CI/CD 和观测体系。

谨慎引入的场景：

- 单体应用、实例数量少、发布频率低。
- 团队缺少容器和集群运维能力。
- 有状态组件没有成熟存储方案。
- 只是为了“上云原生”而迁移，缺少明确收益指标。

面试回答可以收束为一句话：**K8s 不是 Docker 的替代品，而是在容器之上提供集群级声明式编排、服务发现、自愈、伸缩、发布和资源治理的平台。**
