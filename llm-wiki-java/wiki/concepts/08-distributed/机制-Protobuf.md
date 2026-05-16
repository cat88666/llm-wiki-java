---
type: concept
status: active
name: "Protobuf"
layer: L7
aliases: ["gRPC", "Protobuf", "Protocol Buffers", "Varint编码", "HTTP/2多路复用", "Deadline传播", "双向流", "gRPC拦截器"]
tags: ["#distributed"]
related:
  - "[[机制-Netty]]"
  - "[[机制-Dubbo]]"
  - "[[机制-SpringCloud]]"
  - "[[机制-RabbitMQ]]"
sources:
  - "../../raw/note/Interview/Eson.md"
created: 2026-05-07
updated: 2026-05-15
lint_notes: ""
---

# Protobuf

> gRPC 是 Google 开源的、基于 HTTP/2 + Protocol Buffers 的高性能跨语言 RPC 框架；Protobuf 是其二进制序列化协议，相比 JSON 体积缩减 3-10×，解析速度快 5-10×。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | REST/JSON 的两个本质瓶颈 |
| [二、Protobuf 二进制编码](#二protobuf-二进制编码) | Field Tag、Varint、向后兼容 |
| [三、HTTP/2 多路复用](#三http2-多路复用) | vs HTTP/1.1：Stream 并发、HPACK 头压缩 |
| [四、四种通信模式](#四四种通信模式) | Unary/Server Stream/Client Stream/双向流 |
| [五、拦截器（Interceptor）](#五拦截器interceptor) | AOP 切面：认证、链路追踪、重试 |
| [六、Deadline 传播](#六deadline-传播) | 调用链级联超时控制 |
| [七、错误模型](#七错误模型) | 17种 Status Code |
| [八、gRPC vs REST 关键权衡](#八grpc-vs-rest-关键权衡) | 性能/可读性/浏览器支持/契约管理 |
| [九、与其他概念的关系](#九与其他概念的关系) | Netty、Dubbo、SpringCloud、MQ |
| [十、应用边界](#十应用边界) | 选 gRPC vs 选 REST 的决策信号 |

## 一、第一性原理

REST/JSON 的两个本质瓶颈：
1. **文本编码低效**：JSON 字段名重复传输、数字以字符串形式传输，体积大、解析慢
2. **HTTP/1.1 连接利用率低**：Head-of-Line Blocking，每个连接同时只能处理一个请求；微服务间高频调用下线程和连接被大量占用

gRPC 用 Protobuf 解决序列化，用 HTTP/2 解决连接复用，两者合力将内部微服务通信的性能提升一个数量级。

## 二、Protobuf 二进制编码

```protobuf
// 定义消息与服务 (.proto)
message UserRequest {
  int64 user_id = 1;
  string region  = 2;
}
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc StreamUsers (UserRequest) returns (stream UserResponse); // 服务端流
}
```

- **字段编号（Field Tag）**：编码为 `(field_number << 3) | wire_type`，不传字段名，比 JSON `"user_id":` 节省大量字节
- **Varint 编码**：小整数用更少字节（1-10 内只需 1 字节）；负数统一用 sint32/sint64（ZigZag 编码）
- **字段可选性**：字段不存在时不占字节；新增字段不影响旧客户端解码（向后兼容）
- **代码生成**：`protoc` 插件生成 Stub（客户端代理）和 Base（服务端实现接口），支持 20+ 语言

**Protobuf vs JSON 量化对比**：
- 体积：Protobuf 通常是 JSON 的 1/3 到 1/10
- 解析速度：Protobuf 比 JSON 快 5-10×（二进制 vs 文本）
- 代价：需要 .proto 文件和代码生成工具链，可读性差

## 三、HTTP/2 多路复用

| 特性 | HTTP/1.1 | HTTP/2 |
|------|---------|--------|
| 连接复用 | 单请求/连接（Keep-Alive 勉强）| 多 Stream 并发共享一个 TCP 连接 |
| 头部压缩 | 无（每次重传全量头）| HPACK 静态表 + 动态表，增量传输 |
| 流量控制 | TCP 级别 | Stream 级别（精细控制）|
| 帧优先级 | 无 | 有（Stream Priority）|

gRPC 的每次 RPC 调用对应一个 HTTP/2 Stream，多个并发调用共享一个 TCP 连接，彻底消除 HTTP/1.1 的连接开销。

## 四、四种通信模式

```
Unary（一元）:       Client →  Request  → Server → Response → Client
Server Streaming:   Client →  Request  → Server → Response × N → Client
Client Streaming:   Client →  Request × N → Server → Response → Client
Bidirectional:      Client ←→ Request/Response × N ←→ Server（全双工）
```

双向流是 gRPC 最核心的差异化能力，适合实时游戏对战、行情推送等场景。

## 五、拦截器（Interceptor）

```java
// 客户端拦截器：统一添加 Auth Token
ClientInterceptor authInterceptor = new ClientInterceptor() {
    public <Req, Resp> ClientCall<Req, Resp> interceptCall(
            MethodDescriptor<Req, Resp> method, CallOptions options, Channel next) {
        return new ForwardingClientCall.SimpleForwardingClientCall<>(next.newCall(method, options)) {
            public void start(Listener<Resp> responseListener, Metadata headers) {
                headers.put(AUTH_KEY, token);
                super.start(responseListener, headers);
            }
        };
    }
};
```

拦截器是 gRPC 的 AOP 切面，用于认证、链路追踪（TraceId 注入）、重试、监控。

## 六、Deadline 传播

gRPC 的 Deadline 会在调用链中级联传播：A 调用 B（剩余 500ms），B 调用 C 时也只有剩余时间，C 不会无限等待。这是 REST 做不到的。

```java
stub.withDeadlineAfter(500, TimeUnit.MILLISECONDS).getUser(request);
```

**价值**：防止慢速下游因无 Deadline 而占用上游线程；在调用链中自然实现超时预算分配。

## 七、错误模型

gRPC 有 17 种 Status Code（非 HTTP Status Code）：

| Code | 含义 | 典型场景 |
|------|------|---------|
| `OK(0)` | 成功 | — |
| `CANCELLED(1)` | 调用方取消 | 客户端超时主动取消 |
| `DEADLINE_EXCEEDED(4)` | 截止时间超过 | Deadline 传播超时 |
| `NOT_FOUND(5)` | 资源不存在 | 查询不到记录 |
| `RESOURCE_EXHAUSTED(8)` | 资源耗尽/限流 | 限流/配额超限 |
| `UNAVAILABLE(14)` | 服务不可用 | 服务未启动/网络不通 |

## 八、gRPC vs REST 关键权衡

| 维度 | gRPC | REST/JSON |
|------|------|-----------|
| 性能 | 高（二进制 + HTTP/2）| 低（文本 + HTTP/1.1）|
| 可读性 | 差（二进制不可读，需工具）| 好（直接 curl）|
| 浏览器支持 | 差（需 gRPC-Web + 代理）| 好 |
| 双向流 | 原生支持 | 需 WebSocket |
| 契约管理 | 强制（.proto 是单一来源）| 靠 OpenAPI，非强制 |
| 学习成本 | 高（需掌握 proto + 代码生成）| 低 |

**选 gRPC 的信号**：内部微服务高频调用、跨语言团队、需要流式通信、性能敏感。
**选 REST 的信号**：对外 API、需要浏览器直接调用、团队 HTTP 生态成熟。

## 九、与其他概念的关系

- **依赖 [[机制-Netty]]**：Java gRPC 底层传输层使用 Netty，EventLoop 管理 HTTP/2 连接
- **区别于 [[机制-Dubbo]]**：Dubbo 是 Java-first 的服务治理框架，内置注册中心/负载均衡；gRPC 只是通信协议，需要外部治理（Consul/Nacos）
- **与 [[机制-SpringCloud]] 配合**：Spring Cloud 生态通过 `grpc-spring-boot-starter` 集成 gRPC，Nacos 作为注册中心
- **优于 [[机制-RabbitMQ]] 的场景**：同步请求-响应（RPC），而非异步消息投递

## 十、应用边界

**用 gRPC**：
- 内部微服务之间高频 RPC（延迟降低 15%+）
- 跨语言系统集成（Java 后端 + Go 前端 + Python ML 服务）
- 实时流数据推送（游戏战斗数据、行情推送）

**不用 gRPC**：
- 对外公开 API（REST 可读性和工具生态更好）
- 浏览器直接调用（需 gRPC-Web 代理，不如 REST 简单）
- 简单场景（引入 proto 代码生成工具链成本超过收益）
