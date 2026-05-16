---
type: concept
status: active
name: "容器化与Docker"
layer: L7
aliases: ["Docker", "容器", "Dockerfile", "Docker Compose", "容器编排", "云原生", "Serverless", "namespace", "cgroup", "GraalVM", "无状态设计"]
tags: ["#devops"]
related:
  - "[[概念-k8s]]"
  - "[[机制-SpringCloud]]"
  - "[[机制-Dubbo]]"
  - "[[主题-三高架构]]"
sources:
  - "../../../raw/note/Hollis/容器/✅容器和虚拟机的区别是什么？.md"
  - "../../../raw/note/Hollis/容器/✅为什么要使用Docker？.md"
  - "../../../raw/note/Hollis/容器/✅Dockerfile 是什么？它通常包含哪些指令？.md"
  - "../../../raw/note/Hollis/容器/✅什么是 Docker Compose？.md"
  - "../../../raw/note/Hollis/容器/✅Docker 的常用命令有哪些？.md"
  - "../../../raw/note/Hollis/容器/✅有了Docker为啥还需要k8s.md"
  - "../../../raw/note/Hollis/云计算/✅什么是云计算？.md"
  - "../../../raw/note/Hollis/云计算/✅什么是IaaS、PaaS、SaaS？.md"
  - "../../../raw/note/Hollis/云计算/✅什么是公有云、私有云、混合云？.md"
  - "../../../raw/note/Hollis/云计算/✅什么是Serverless？.md"
  - "../../../raw/note/Hollis/云计算/✅啥是无状态，为啥说Serverless是无状态的.md"
  - "../../../raw/note/Hollis/云计算/✅为什么云原生对应用的启动速度要求很高？.md"
created: 2026-05-06
updated: 2026-05-15
---

# 容器化与 Docker

> 容器是共享宿主机 OS 内核的隔离进程组，Docker 提供标准化的容器镜像构建和运行时，K8s 负责在集群中编排大规模容器；进一步演进到云原生时，讨论的是如何将应用以容器/K8s/Serverless 方式运行在云环境中。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 容器 vs VM：namespace + cgroup vs Hypervisor |
| [二、容器 vs 虚拟机对比](#二容器-vs-虚拟机对比) | 隔离实现、启动时间、资源开销 |
| [三、Docker 核心概念与命令](#三docker-核心概念与命令) | Image/Container/Registry，Dockerfile，常用命令 |
| [四、为什么有 Docker 还需要 K8s](#四为什么有-docker-还需要-k8s) | 生产编排问题，K8s 核心对象 |
| [五、云服务交付模型](#五云服务交付模型) | IaaS/PaaS/SaaS/Serverless |
| [六、三种云部署模式](#六三种云部署模式) | 公有云/私有云/混合云 |
| [七、Serverless 与 Java 冷启动](#七serverless-与-java-冷启动) | 冷启动问题，GraalVM Native Image 解法 |
| [八、无状态设计原则](#八无状态设计原则) | 有状态 vs 无状态，云原生改造要点 |
| [九、关键权衡](#九关键权衡) | 容器隔离、Compose vs K8s、无状态性能 |
| [十、与其他概念的关系](#十与其他概念的关系) | SpringCloud、Dubbo、三高架构 |
| [十一、应用边界](#十一应用边界) | 适合 vs 谨慎容器化，K8s 引入时机 |

## 一、第一性原理

虚拟机通过 Hypervisor 为每个 VM 模拟完整硬件并运行独立 OS 内核，隔离彻底但每个 VM 都携带完整 OS，启动慢、资源开销大。

容器的核心洞察：同一宿主机上的进程共享同一 OS 内核，只需用 Linux **namespace**（隔离进程/网络/文件系统视图）和 **cgroup**（限制 CPU/内存资源）就能实现隔离，不需要重复一份 OS。

```
VM:   [App A] [App B]         容器:  [App A] [App B]
      [OS A]  [OS B]                 [容器运行时]
      [Hypervisor]                   [宿主机 OS 内核]（共享）
      [物理硬件]                      [物理硬件]
```

## 二、容器 vs 虚拟机对比

| 维度 | 容器 | 虚拟机 |
|------|------|--------|
| 隔离实现 | namespace + cgroup | Hypervisor + 完整 OS |
| 启动时间 | 秒级 | 分钟级 |
| 镜像大小 | MB~数百 MB | GB 级 |
| 资源开销 | 低（共享内核）| 高（每个 VM 一份 OS）|
| 隔离强度 | 中（内核共享）| 高（内核隔离）|
| 适用场景 | 微服务、CI/CD、弹性扩缩 | 强隔离、不同 OS 环境 |

## 三、Docker 核心概念与命令

```
镜像 (Image)  →  容器 (Container)  →  仓库 (Registry)
  只读层叠            可写运行实例          镜像存储分发
```

**Dockerfile 核心指令**：

```dockerfile
FROM ubuntu:22.04
LABEL maintainer="team"
ENV JAVA_HOME=/usr/lib/jvm
ARG VERSION=1.0
WORKDIR /app
COPY target/*.jar app.jar
RUN apt-get update && apt-get install -y curl
EXPOSE 8080
HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar"]
CMD ["app.jar"]
USER appuser
```

**Docker Compose（多容器编排）**：

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
  mysql:
    image: mysql:8.0
```

**常用命令**：

```bash
docker build -t myapp:1.0 .
docker run -d -p 8080:8080 --name app myapp:1.0
docker ps
docker logs -f app
docker exec -it app bash
```

## 四、为什么有 Docker 还需要 K8s

Docker 解决单机容器运行，K8s 解决生产环境的编排问题：

> K8s 的架构、Pod 生命周期、调度、网络、存储和安全细节见 [[概念-k8s]]。

| 生产问题 | K8s 解决方案 |
|---------|------------|
| 容器宕机需手动重启 | Deployment / Pod 自动重启 |
| 手动扩缩容 | HPA 自动扩缩 |
| 跨主机服务发现 | Service + CoreDNS |
| 流量负载均衡 | Service / Ingress |
| 配置/密钥管理 | ConfigMap + Secret |
| 滚动发布/回滚 | Deployment 滚动更新 |
| 节点故障转移 | 调度器自动迁移 |

**K8s 核心对象**：
- **Pod**：最小调度单元
- **Deployment**：无状态应用副本控制
- **Service**：稳定 VIP + DNS
- **Ingress**：集群入口与七层路由
- **StatefulSet**：有状态应用

## 五、云服务交付模型

| 模型 | 全称 | 用户管理范围 | 典型产品 |
|------|------|------------|---------|
| **IaaS** | 基础设施即服务 | OS + 运行时 + 应用 | ECS / EC2 |
| **PaaS** | 平台即服务 | 应用 + 数据 | EDAS / Heroku |
| **SaaS** | 软件即服务 | 仅配置 | 飞书 / Salesforce |
| **Serverless** | 函数即服务 | 仅业务逻辑 | 阿里云 FC / AWS Lambda |

## 六、三种云部署模式

| 模式 | 特点 | 适用场景 |
|------|------|---------|
| **公有云** | 共享资源，按量付费 | 初创公司、弹性需求 |
| **私有云** | 独享资源，高安全性 | 政府、金融、医疗 |
| **混合云** | 核心数据私有云，峰值流量公有云 | 大型企业、云爆发场景 |

**选型决策树**：
```
需要部署 Java 后端服务
         │
是否有 DevOps 能力自运维基础设施？
    No  → 公有云 PaaS
    Yes → 继续
         │
是否有数据合规/安全要求？
    Yes → 私有云或混合云
    No  → 公有云
         │
流量是否有明显峰谷？
    Yes → Serverless 或 K8s + HPA
    No  → 固定规格 ECS / 容器集群
```

## 七、Serverless 与 Java 冷启动

**为什么云原生对启动速度要求高**：
```
无请求时 → 服务不运行
有请求时 → 按需启动 → 处理请求 → 释放资源
```
如果启动耗时 10 秒，用户就要等 10 秒，Serverless 场景对冷启动极其敏感。

**Java 的冷启动问题**：JVM 启动 + 类加载 + JIT 预热通常需要 3~10 秒；Spring Boot 还会额外增加数秒。

**解决方案对比**：

| 方案 | 原理 | 启动时间 | 代价 |
|------|------|---------|------|
| GraalVM Native Image | AOT 编译成本地程序 | < 100ms | 反射/动态代理受限 |
| Spring Boot 3 + GraalVM | Spring 官方 AOT | ~50ms | 需新版本技术栈 |
| Quarkus | 为云原生设计 | ~100ms | 生态小于 Spring |
| Keep Warm | 保持实例不销毁 | ~10ms | 持续占用资源 |

## 八、无状态设计原则

**无状态**：每次请求独立处理，不依赖之前请求的 JVM 内存状态。

| 有状态（避免）| 无状态（推荐）|
|----------------|------------|
| 本地 JVM 缓存 | Redis 分布式缓存 |
| 本地文件存储 | OSS / S3 |
| Session in JVM | JWT / Redis Session |
| 单机连接状态 | 粘性路由或消息总线 |

无状态的价值：水平扩展简单、实例故障可快速恢复、更适配 K8s 和 Serverless。

**Java 后端云原生改造要点**：

| 改造项 | 传统做法 | 云原生做法 |
|--------|---------|----------|
| 配置管理 | 本地 application.yml | Nacos / Apollo / ConfigMap |
| 日志 | 本地文件 | stdout + ELK / SLS |
| 健康检查 | 无 | `/actuator/health` + probes |
| 链路追踪 | 无 | SkyWalking / OpenTelemetry |
| 优雅停机 | 强制 kill | graceful shutdown + SIGTERM |
| 服务发现 | 配置文件写死 IP | Nacos / K8s Service DNS |

## 九、关键权衡

1. **容器隔离 vs VM 隔离**：容器更轻，但共享内核；高安全场景可用 gVisor 等强隔离方案
2. **Compose vs K8s**：Compose 适合开发；K8s 才是生产编排标准
3. **镜像大小 vs 调试便利**：Alpine 小但不易调试，多阶段构建是常见平衡
4. **无状态 vs 本地性能**：无状态利于扩展，但会牺牲部分本地缓存便利
5. **Serverless 灵活 vs Java 冷启动**：Serverless 运维成本低，但 Java 需要额外优化启动路径

## 十、与其他概念的关系

- **[[机制-SpringCloud]]**：云原生是微服务的基础设施层承载方式，Nacos/Sentinel 等组件在 K8s 中以 Sidecar 模式运行
- **[[机制-Dubbo]]**：容器环境中服务发现和 IP 漂移会影响 RPC 治理，Dubbo 需配合 Nacos/ZK 感知变化
- **[[主题-三高架构]]**：K8s Deployment、HPA、Probe、反亲和性是高可用的基础设施手段

## 十一、应用边界

**适合容器化 / 云原生**：无状态 Web 服务、微服务 API、定时任务、CI/CD 工作负载、推理服务。

**谨慎容器化**：生产数据库（有状态数据管理复杂）、对延迟极敏感的实时系统、需要特殊硬件/驱动访问的应用。

**K8s 引入时机**：服务数量较多（> 5 个）、需要自动扩缩容、需要滚动发布零停机、团队具备 K8s 运维能力。
