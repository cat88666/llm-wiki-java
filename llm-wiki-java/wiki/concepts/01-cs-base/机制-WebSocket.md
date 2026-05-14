---
type: concept
status: active
name: "WebSocket"
layer: L1
tags: ["#cs-base", "#network", "#protocol"]
aliases: ["WebSocket", "WSS", "长连接", "全双工推送", "WebSocket握手"]
related:
  - "[[机制-Netty]]"
  - "[[机制-Kafka]]"
  - "[[机制-RPC与Protobuf]]"
  - "[[概念-幂等设计]]"
sources:
  - "../../raw/note/Interview/Eson.md"
created: 2026-05-14
updated: 2026-05-14
lint_notes: ""
---

# WebSocket

> 基于 TCP 的全双工持久连接协议：通过一次 HTTP Upgrade 握手将连接升级，此后服务端可主动推送消息，彻底消除 HTTP 请求-响应模型中"只有客户端能发起通信"的根本限制。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | HTTP 限制、四种推送方案对比 |
| [二、核心机制](#二核心机制) | HTTP Upgrade 握手、帧结构、心跳 |
| [三、Java 核心使用](#三java-核心使用) | Netty Pipeline、Spring WebSocket |
| [四、认证与安全](#四认证与安全) | Token 方案、WSS |
| [五、水平扩展](#五水平扩展) | 粘滞会话、Redis 注册表 + MQ 路由 |
| [六、综合对比](#六综合对比) | WebSocket vs SSE vs 长轮询 vs gRPC 流 |
| [七、生产风险](#七生产风险) | 连接数、消息可靠性、重连、LB 兼容 |
| [八、与其他概念的关系](#八与其他概念的关系) | Netty、Kafka、gRPC |
| [九、应用边界](#九应用边界) | 适用场景、不适用场景 |

## 一、第一性原理

HTTP 请求-响应模型的根本限制：**只有客户端能发起通信，服务端无法主动推送**。

| 方案 | 原理 | 核心缺陷 |
|------|------|---------|
| 短轮询 | 客户端定期发请求 | 大量无效请求，延迟高 |
| 长轮询 | 服务端 hold 请求直到有数据 | 每次响应后必须重建连接，资源浪费 |
| SSE | HTTP 流，服务端单向推送 | 半双工，客户端不能通过同连接回发消息 |
| **WebSocket** | TCP 全双工，一次握手，持久连接 | 有连接数上限；水平扩展有状态 |

IM、实时游戏、行情推送等场景需要**真正的全双工**，WebSocket 是协议层唯一合适的选择。

## 二、核心机制

### HTTP Upgrade 握手

```http
# 客户端（HTTP/1.1）
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

# 服务端
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`Sec-WebSocket-Accept = Base64(SHA1(Key + "258EAFA5..."))` — 防止恶意 HTTP 请求意外触发升级。握手完成后 HTTP 协议让位，TCP 连接进入 WebSocket 帧模式。

### 帧结构（Frame）

关键字段：

| 字段 | 含义 |
|------|------|
| FIN | 1 = 此帧是消息最后一帧 |
| Opcode | `0x1`=文本、`0x2`=二进制、`0x8`=关闭、`0x9`=Ping、`0xA`=Pong |
| MASK | 客户端→服务端必须掩码（防代理缓存污染）；服务端→客户端无需 |

### 心跳机制

协议自带 Ping/Pong 帧，但生产通常由应用层驱动：

- **服务端**：`IdleStateHandler` 检测连接空闲，超时触发 `userEventTriggered`，发 Ping 或直接断链
- **客户端**：每 30s 发心跳包，连续 N 次收不到 Pong 则主动重连
- **双向检测**是关键：只有服务端检测会漏掉"客户端假死"场景

## 三、Java 核心使用

### Netty Pipeline（生产主流）

```java
pipeline.addLast(new HttpServerCodec());
pipeline.addLast(new HttpObjectAggregator(65536));
pipeline.addLast(new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS));
pipeline.addLast(new WebSocketServerProtocolHandler("/ws", null, true));
pipeline.addLast(new AuthHandler());      // 验证首条消息 Token
pipeline.addLast(new HeartbeatHandler()); // 处理 IdleStateEvent
pipeline.addLast(new BusinessHandler());  // 业务逻辑
```

### Spring WebSocket（适合非极致性能场景）

```java
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/ws")
                .setAllowedOrigins("*");
    }
}
```

Spring WebSocket 底层可切换 Jetty/Tomcat NIO，不需要 Netty 时可用，但连接数量大时优先选 Netty。

## 四、认证与安全

**问题**：WebSocket 握手是 HTTP 请求，浏览器的 JS 无法自定义 `Authorization` Header。

| 方案 | 做法 | 安全性 |
|------|------|--------|
| URL Query Param | `/ws?token=xxx` | 低，Token 会出现在日志和 nginx access log |
| 首条消息认证 | 连接后第一条消息携带 Token，服务端验证通过才放行 | **推荐**，Token 不暴露于 URL |
| Cookie | 握手时浏览器自动携带 Cookie | 适合同域场景，跨域受限 |

**WSS**（WebSocket over TLS）：生产必须启用，防止中间人劫持。

## 五、水平扩展

WebSocket 连接是**有状态的**（绑定到具体机器），横向扩容需解决跨节点路由：

```
Client ─ LB ──┬── Node A（userId: 101, 203）
              └── Node B（userId: 102, 204）
// 101 给 102 发消息：消息到达 Node A，但 102 的连接在 Node B
```

| 方案 | 原理 | 代价 |
|------|------|------|
| 粘滞会话（Sticky Session） | LB 按 userId hash 固定路由到同一节点 | 节点宕机则该节点所有连接全断 |
| Redis 注册表 + MQ 路由（推荐） | 连接建立时写 `HSET ws:conn userId nodeId`；消息先查 Redis 找目标节点，再发 MQ，目标节点消费后推送 | 复杂度高，但弹性好 |

## 六、综合对比

| 维度 | WebSocket | SSE | 长轮询 | gRPC 双向流 |
|------|-----------|-----|--------|------------|
| 双工 | 全双工 | 半双工（服务端→客户端） | 半双工 | 全双工 |
| 浏览器支持 | 原生支持 | 原生支持 | 原生支持 | 需 grpc-web |
| 协议开销 | 帧头极小 | HTTP 头每次推送 | HTTP 连接重建开销 | 二进制，高效 |
| 适用场景 | IM、游戏、协作 | 通知、进度推送 | 兼容性要求高场景 | 服务间双向流 |
| 水平扩展难度 | 高（有状态） | 中 | 低（无状态） | 高（有状态） |

**结论**：只需服务端单向推送用 SSE；需要浏览器全双工用 WebSocket；服务间双向流考虑 gRPC。

## 七、生产风险

| 风险 | 表现 | 应对 |
|------|------|------|
| 连接数上限 | 文件描述符耗尽（默认 1024） | 调大 `ulimit -n`；用 epoll 事件驱动（Netty 默认） |
| 消息可靠性 | TCP 保证传输，但应用层无 ACK | 应用层实现消息 ACK + 重试；离线消息落 DB/Kafka |
| 假死连接 | 客户端网络切换后 TCP 状态未清理 | 双向心跳 + IdleStateHandler 超时强制断链 |
| 重连消息丢失 | 断线期间服务端推送的消息丢失 | 重连时携带 `lastMsgId`，服务端补发离线消息 |
| LB 兼容性 | Nginx/K8s Ingress 默认不支持 Upgrade | Nginx 配置 `proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade` |
| 重连风暴 | 大量客户端同时断线重连打垮服务端 | 指数退避重连：`min(基础延迟 × 2^n, 上限)`，加随机抖动 |

## 八、与其他概念的关系

- 依赖 [[机制-Netty]]：Java 生产级 WebSocket 几乎都用 Netty，`WebSocketServerProtocolHandler` 封装了握手全流程
- 区别于 [[机制-Kafka]]：WebSocket 是实时推送协议（在线用户）；Kafka 是持久化消息队列（离线积压）；IM 系统两者都需要
- 区别于 [[机制-RPC与Protobuf]]：gRPC 双向流也可实现类似能力，但不适合浏览器直连；WebSocket 对浏览器天然友好
- 依赖 [[概念-幂等设计]]：重连补发离线消息时，客户端必须幂等处理，防止重复展示

## 九、应用边界

**适合 WebSocket：**
- IM / 聊天系统（在线实时推送）
- 游戏实时对战（双向高频消息）
- 金融行情推送（服务端主动推）
- 协作工具（在线文档实时编辑、多人光标同步）

**不适合 WebSocket：**
- 只需服务端单向推送 → SSE 更简单，无需管理双向状态
- 请求-响应式 API → REST / gRPC 足够，引入 WebSocket 增加复杂度
- 消息频率极低 → 轮询成本可接受时，维护长连接得不偿失
- 极高吞吐服务间通信 → gRPC 二进制协议更高效
