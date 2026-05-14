---
layer: L7
tags: [distributed, protocol, network]
aliases: [WebSocket, WSS, 长连接]
sources:
  - ../../raw/note/Interview/Eson.md
---

# 机制-WebSocket协议

> WebSocket 是基于 TCP 的全双工持久连接协议，通过 HTTP Upgrade 握手建立，之后服务端可主动向客户端推送消息，无需客户端轮询。

## 第一性原理

HTTP 请求-响应模型的根本限制：**只有客户端能发起通信**。实现服务端推送的三种方案：

| 方案 | 原理 | 缺陷 |
|------|------|------|
| 短轮询 | 客户端定期请求 | 大量无效请求，延迟高 |
| 长轮询 | 服务端 hold 请求直到有数据 | 每次响应后需重新建立请求，资源浪费 |
| SSE | HTTP 流，服务端单向推送 | 半双工，客户端不能通过同一连接发送消息 |
| **WebSocket** | TCP 全双工，一次握手，持久连接 | 有连接数限制；水平扩展复杂 |

IM、游戏实时对战、行情推送等场景需要**真正的全双工**，WebSocket 是唯一合适的协议。

## 核心机制

### 1. HTTP Upgrade 握手

```http
# 客户端发送（HTTP/1.1）
GET /ws HTTP/1.1
Host: game.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

# 服务端响应
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`Sec-WebSocket-Accept = Base64(SHA1(Sec-WebSocket-Key + "258EAFA5..."))` — 防止恶意 HTTP 请求意外升级。握手完成后，HTTP 协议让位，TCP 连接进入 WebSocket 帧模式。

### 2. 帧结构（Frame）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
```

关键字段：
- **FIN**：1 表示此帧是消息的最后一帧
- **Opcode**：`0x1`=文本, `0x2`=二进制, `0x8`=关闭, `0x9`=Ping, `0xA`=Pong
- **MASK**：客户端到服务端必须掩码（防止代理缓存污染），服务端到客户端无需掩码

### 3. 心跳机制

WebSocket 协议自带 Ping/Pong 帧，但通常由应用层实现心跳：

```java
// Netty 服务端 IdleStateHandler 触发心跳
pipeline.addLast(new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS)); // 60s 无读写
pipeline.addLast(new HeartbeatHandler());

// 客户端每 30s 发送心跳包
// 服务端收到心跳回复 PONG；超过 N 次未收到心跳则断开连接
```

**双向心跳**：服务端检测客户端是否存活（IdleStateHandler）；客户端检测服务端是否存活（收不到 PONG 则重连）。

### 4. 认证与安全

**问题**：WebSocket 握手是 HTTP 请求，浏览器不能设置自定义 Header（如 Authorization）。

**方案一**：Token 放 URL Query Param（`/ws?token=xxx`） — 会出现在日志，安全性低。  
**方案二**：连接建立后第一条消息发送 Token，服务端验证通过后才允许后续通信（推荐）。  
**WSS**：WebSocket over TLS，防止中间人攻击，生产必用。

```java
// Netty WebSocket 服务端 Pipeline
pipeline.addLast(new HttpServerCodec());
pipeline.addLast(new HttpObjectAggregator(65536));
pipeline.addLast(new WebSocketServerProtocolHandler("/ws", null, true));
pipeline.addLast(new AuthHandler());     // 验证第一条消息中的 Token
pipeline.addLast(new BusinessHandler()); // 业务处理
```

### 5. 水平扩展挑战

WebSocket 连接是**有状态的**（绑定到具体机器）。多节点扩容需要解决跨节点消息路由：

```
                    ┌─ Node A ─ userId: 101, 203
Client ─ LB ────────┤
                    └─ Node B ─ userId: 102, 204

// 问题：101 给 102 发消息，消息到了 Node A，102 在 Node B
```

**解决方案**：
1. **粘滞会话（Sticky Session）**：LB 按 userId 固定路由到同一节点。节点宕机则连接全断。
2. **Redis 连接注册表 + MQ 消息路由**（推荐）：
   - 连接建立时：`HSET ws:conn userId nodeId`
   - 消息路由：查 Redis 找目标 userId 所在 nodeId → 发 MQ（topic = `ws:node:{nodeId}`）→ 目标节点从 MQ 消费并推送给客户端

### 6. 重连机制

WebSocket 断开后客户端必须实现重连，建议指数退避：
```
重连延迟 = min(基础延迟 × 2^重试次数, 最大延迟)
例：1s → 2s → 4s → 8s → 16s → 30s（上限）
重连时附带 lastMsgId，服务端补发期间的离线消息
```

## 关键权衡

| 维度 | 说明 |
|------|------|
| 连接资源 | 每条连接占一个文件描述符；单机 C10K 需 epoll + 异步（Netty 轻松支持） |
| 水平扩展 | 有状态连接扩容比 REST 复杂，需要连接注册表 |
| 负载均衡 | 传统 HTTP LB 需配置支持 Upgrade；K8s Ingress 需开 WebSocket 支持 |
| 消息可靠性 | TCP 只保证传输可靠，应用层需要 ACK + 重试保证消息可靠 |

## 与其他概念的关系

- 依赖 [[机制-Netty]]：Java 生产级 WebSocket 几乎都用 Netty（`WebSocketServerProtocolHandler`）。
- 依赖 [[机制-Redis分布式锁]]：跨节点消息路由时，Redisson 锁保证同一用户的消息顺序处理。
- 区别于 [[机制-Kafka]]：WebSocket 是实时推送协议，Kafka 是持久化消息队列；IM 中 Kafka 用于离线消息积压，WebSocket 用于在线实时推送。
- 区别于 [[机制-gRPC与Protobuf]]：gRPC 双向流也能实现类似能力，但 gRPC 不适合浏览器直连；WebSocket 对浏览器友好。
- 支撑了 [[系统设计-IM系统]]：IM 系统接入层的核心协议。

## 应用边界

**用 WebSocket：**
- IM / 聊天系统（Eson 简历中谁信科技 IM 系统）
- 游戏实时对战（Eson 简历当前东京游戏公司）
- 金融行情实时推送
- 协作工具（在线文档实时编辑）

**不用 WebSocket：**
- 只需服务端单向推送（SSE 更简单）
- 请求-响应式 API（REST 或 gRPC 足够）
- 消息频率很低（轮询成本可接受时，WebSocket 维护连接反而复杂）
