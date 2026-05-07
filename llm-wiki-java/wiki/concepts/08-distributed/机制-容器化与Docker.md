---
type: concept
status: active
name: "容器化与Docker"
layer: L7
aliases: ["Docker", "容器", "K8s", "Kubernetes", "Dockerfile", "Docker Compose", "容器编排"]
tags: ["#devops"]
related:
  - "[[机制-微服务与SpringCloud]]"
  - "[[机制-RPC与Dubbo]]"
sources:
  - "../../../raw/note/Hollis/容器/✅容器和虚拟机的区别是什么？.md"
  - "../../../raw/note/Hollis/容器/✅为什么要使用Docker？.md"
  - "../../../raw/note/Hollis/容器/✅Dockerfile 是什么？它通常包含哪些指令？.md"
  - "../../../raw/note/Hollis/容器/✅什么是 Docker Compose？.md"
  - "../../../raw/note/Hollis/容器/✅Docker 的常用命令有哪些？.md"
  - "../../../raw/note/Hollis/容器/✅有了Docker为啥还需要k8s.md"
created: 2026-05-06
updated: 2026-05-06
---

# 容器化与 Docker

**一句话定义**：容器是共享宿主机 OS 内核的隔离进程组，Docker 提供标准化的容器镜像构建和运行时，K8s 负责在集群中编排大规模容器。

---

## 第一性原理

虚拟机通过 Hypervisor 为每个 VM 模拟完整硬件并运行独立 OS 内核——隔离彻底但代价是每个 VM 携带完整 OS（GB 级），启动需分钟级。

容器的核心洞察：同一宿主机上的进程共享同一 OS 内核，只需用 Linux **namespace**（隔离进程/网络/文件系统视图）和 **cgroup**（限制 CPU/内存资源）就能实现隔离，不需要重复一份 OS。

```
VM:   [App A] [App B]         容器:  [App A] [App B]
      [OS A]  [OS B]                 [容器运行时]
      [Hypervisor]                   [宿主机 OS 内核]（共享）
      [物理硬件]                      [物理硬件]

VM 启动时间: 分钟级              容器启动时间: 秒级
VM 镜像大小: GB 级              镜像大小: MB~百 MB 级
VM 隔离强度: 高                 容器隔离强度: 中（逃逸风险更高）
```

---

## 核心机制

### 容器 vs 虚拟机对比

| 维度 | 容器 | 虚拟机 |
|------|------|--------|
| 隔离实现 | namespace + cgroup | Hypervisor + 完整 OS |
| 启动时间 | 秒级 | 分钟级 |
| 镜像大小 | MB~数百 MB | GB 级 |
| 资源开销 | 低（共享内核）| 高（每个 VM 一份 OS）|
| 隔离强度 | 中（内核共享）| 高（内核隔离）|
| 适用场景 | 微服务、CI/CD、弹性扩缩 | 强隔离、不同 OS 环境 |

### Docker 核心概念

```
镜像 (Image)  →  容器 (Container)  →  仓库 (Registry)
  只读层叠            可写运行实例          镜像存储分发
  (UnionFS)          (Copy-on-Write)       (Docker Hub / 私有)
```

**UnionFS 分层**：镜像由多个只读层叠加，容器运行时在最上层加一个可写层。层共享使相同基础镜像的容器复用磁盘空间。

### Dockerfile 核心指令

```dockerfile
FROM    ubuntu:22.04           # 基础镜像（必须第一行）
LABEL   maintainer="team"      # 元数据
ENV     JAVA_HOME=/usr/lib/jvm # 环境变量
ARG     VERSION=1.0            # 构建时变量（不进入镜像）
WORKDIR /app                   # 工作目录（推荐绝对路径）
COPY    target/*.jar app.jar   # 复制文件（COPY 优于 ADD，语义清晰）
ADD     archive.tar.gz /opt    # 支持解压和 URL（仅在需要时用）
RUN     apt-get install -y...  # 构建时执行（每条 RUN 新建一层）
EXPOSE  8080                   # 声明端口（仅文档作用，需 -p 映射）
VOLUME  ["/data"]              # 挂载点声明
HEALTHCHECK CMD curl -f ...    # 容器健康检查
ENTRYPOINT ["java", "-jar"]    # 容器启动入口（不可被 docker run 覆盖）
CMD     ["app.jar"]            # 默认参数（可被 docker run 覆盖）
USER    appuser                # 运行用户（安全实践，非 root）
```

**ENTRYPOINT vs CMD**：ENTRYPOINT 固定可执行文件，CMD 提供默认参数；组合使用 `ENTRYPOINT ["java"] CMD ["-jar", "app.jar"]`，运行时可替换 CMD 但 ENTRYPOINT 不变。

**减少层数**：将多个 RUN 合并为一条（`RUN apt-get update && apt-get install -y ...`），避免每条 RUN 生成一层。

### Docker Compose（多容器编排）

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/db
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - db_data:/var/lib/mysql
volumes:
  db_data:
```

`docker-compose up -d` 一键启动所有服务，自动处理服务依赖和网络互联。适合本地开发和小规模测试环境；生产环境需 K8s。

### 常用 Docker 命令

```bash
# 镜像操作
docker build -t myapp:1.0 .          # 构建镜像
docker pull nginx:alpine              # 拉取镜像
docker images                         # 列出本地镜像

# 容器生命周期
docker run -d -p 8080:8080 --name app myapp:1.0   # 后台启动
docker ps                             # 查看运行中容器
docker stop app && docker rm app      # 停止并删除

# 调试
docker logs -f --tail 100 app         # 实时日志
docker exec -it app bash              # 进入容器
docker inspect app                    # 查看容器详情
```

### 为什么有 Docker 还需要 K8s

Docker 解决了**单机容器运行**问题，但生产环境面临：

| 生产问题 | K8s 解决方案 |
|---------|------------|
| 容器宕机需手动重启 | Pod 控制器自动重启（Deployment）|
| 手动扩缩容 | HPA 根据 CPU/内存自动扩缩 |
| 跨主机服务发现 | Service + CoreDNS 内部 DNS |
| 流量负载均衡 | Service/Ingress 层4/层7负载均衡 |
| 配置/密钥管理 | ConfigMap + Secret |
| 滚动发布/回滚 | Deployment 滚动更新策略 |
| 节点故障转移 | 调度器自动将 Pod 调度到健康节点 |

**K8s 核心对象**：
- **Pod**：最小调度单元，一个或多个容器
- **Deployment**：无状态应用，副本数控制
- **Service**：稳定的 VIP + DNS，屏蔽 Pod IP 变化
- **Ingress**：集群入口，域名/路径路由到 Service
- **StatefulSet**：有状态应用（DB、MQ），稳定网络标识和持久化存储

---

## 关键权衡

1. **容器隔离 vs VM 隔离**：容器共享内核，内核漏洞可能被利用（如容器逃逸）；金融/安全敏感场景用 gVisor 沙箱或 VM 嵌套；大多数应用场景容器隔离足够
2. **Docker Compose vs K8s**：Compose 简单，适合开发和小规模（单机）；K8s 复杂但提供完整生产级特性，5台以上节点的集群才值得引入 K8s 复杂度
3. **镜像大小 vs 功能完整**：Alpine 基础镜像（5MB）适合生产，但缺少调试工具；开发镜像可用 ubuntu/debian；多阶段构建（Multi-stage build）可以在构建阶段用完整镜像，运行阶段只拷贝产物到 Alpine
4. **有状态 vs 无状态容器**：无状态服务（API server）容器化最友好；有状态服务（MySQL）容器化需要 PersistentVolume，实践中生产数据库常在容器外运行

---

## 与其他概念的关系

- **[[机制-微服务与SpringCloud]]**：K8s 是微服务的基础设施层，Service 替代 Eureka 做服务发现，Ingress 替代 Gateway 做流量入口，K8s 提供的 NameSpace 实现环境隔离
- **[[机制-RPC与Dubbo]]**：容器环境下 Pod IP 动态变化，Dubbo 需适配 K8s 服务发现（基于 Service DNS 或 K8s 原生注册中心）
- **[[概念-高可用设计]]**（见 L7）：K8s Deployment 副本数 + HPA + 多节点调度是高可用的基础设施实现；Pod 反亲和性规则确保副本分布在不同节点

---

## 应用边界

**适合容器化**：无状态 Web 服务、微服务 API、定时任务、CI/CD 工作负载、ML 推理服务。

**谨慎容器化**：生产 MySQL/PostgreSQL（数据持久化和性能调优更复杂）、对延迟极敏感的实时系统（容器网络有额外开销）、需要 GPU 直接访问的应用（需要特殊驱动支持）。

**K8s 引入时机**：服务数量 > 5、需要自动扩缩容、需要滚动发布零停机、团队有能力维护 K8s 集群（否则优先考虑云厂商托管 K8s：EKS/AKS/GKE）。
