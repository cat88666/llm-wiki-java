---
type: synthesis
status: active
name: "IM系统设计"
layer: L8
aliases: ["IM", "即时通信", "WebSocket", "消息可靠投递", "消息顺序", "在线状态", "群消息扇出", "未读消息数", "消息冷热分层"]
tags: ["#practice", "#distributed"]
related:
  - "[[机制-Netty]]"
  - "[[概念-Redis]]"
  - "[[概念-幂等设计]]"
  - "[[机制-RabbitMQ]]"
sources:
  - "../../raw/note/Hollis/面经实战/"
updated: 2026-05-18
---

# 系统设计-IM系统

> IM 系统的三个核心工程问题：消息可靠投递（双向 ACK 防丢）、会话内有序（Redis INCR 分配 seqId）、百万连接水平扩展（一致性 Hash + MQ 跨节点路由）；选型结论取决于对消息丢失的容忍度和在线规模。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、IM 系统架构](#一im-系统架构) | 整体架构全景、三大核心工程约束及根因 |
| [二、消息可靠投递](#二消息可靠投递) | 双向 ACK、clientMsgId 幂等、超时重传 |
| [三、消息顺序保证](#三消息顺序保证) | seqId 分配、空洞检测与补全 |
| [四、在线状态管理](#四在线状态管理) | Redis 心跳 TTL、连接注册/断开 |
| [五、水平扩展：跨节点路由](#五水平扩展跨节点路由) | 一致性 Hash、MQ 跨节点转发 |
| [六、群消息扇出方案对比](#六群消息扇出方案对比) | 写扩散 vs 读扩散选型 |
| [七、存储与冷热分层](#七存储与冷热分层) | 消息表设计、热7天/冷归档 |
| [八、面试高频追问](#八面试高频追问) | 消息可靠性、顺序、水平扩展 |
| [九、Netty、Redis、幂等设计、MQ 在 IM 架构中的位置](#九netty-redis-幂等设计-mq-在-im-架构中的位置) | 接入层长连接、在线状态/序号、幂等去重、跨节点路由 |

## 一、IM 系统架构

**整体架构**：

```
Client ─── LB ──→ 接入层（Netty + WebSocket，一致性 Hash 路由）
                       │ 业务事件
              业务层（消息路由/在线状态/群组/seq分配）
               │读写          │发布
          MySQL            Redis            MQ             OSS
       （消息持久化）   （在线状态/seq/   （跨节点路由/    （图片/视频）
                         未读数）          离线推送）
```

**三大核心工程问题及约束**：

| 问题 | 根因 | 关键约束 |
|------|------|---------|
| 消息可靠投递 | WebSocket 无内置 ACK | 双向 ACK + 超时重传 + 幂等去重 |
| 会话内有序 | 并发写可能乱序到 DB | Redis INCR 原子分配 seqId |
| 水平扩展 | 有状态连接不能随意漂移 | 一致性 Hash 固定节点 + MQ 跨节点转发 |

## 二、消息可靠投递

```
发送方                  服务端                    接收方
  │── send(msg, clientMsgId) →│                      │
  │                           │ 生成服务端 seqId      │
  │                           │── push(msg, seqId) ──→│
  │←── ServerACK(clientMsgId)─│                      │
  │                           │←── ClientACK(seqId) ─│
  │                           │ 更新已读游标           │
```

**关键设计点**：

| 字段 | 作用 | 说明 |
|------|------|------|
| `clientMsgId` | 客户端幂等 ID（UUID） | 服务端以此去重，防止网络重传重复入库 |
| `seqId` | 服务端单调递增序号 | 客户端靠它排序和检测消息空洞 |
| 超时重传 | 5s 未收到 ServerACK 自动重发 | 服务端必须幂等处理 |
| 空洞补全 | 客户端检测 seqId 不连续时主动拉取 | 防止消息展示乱序 |

```java
// DB 唯一索引兜底幂等（client_msg_id UNIQUE）
CREATE TABLE message (
    id             BIGINT AUTO_INCREMENT,
    conversation_id VARCHAR(64) NOT NULL,
    seq_id         BIGINT NOT NULL,
    sender_id      VARCHAR(64) NOT NULL,
    content        TEXT,
    client_msg_id  VARCHAR(64) UNIQUE,  -- 幂等键
    created_at     DATETIME,
    UNIQUE KEY uk_conv_seq (conversation_id, seq_id)
);
```

## 三、消息顺序保证

**问题根因**：同一会话两条消息可能经由不同路径到达 DB（网络乱序 + 并发写）。

```java
// Redis INCR 原子分配单调递增 seqId，保证会话内有序
Long seqId = redisTemplate.opsForValue()
    .increment("msg:seq:" + conversationId);
message.setSeqId(seqId);
messageRepo.save(message);
pushToClient(receiverId, message);  // 带 seqId 推送给接收方
```

**客户端处理**：收到消息按 seqId 升序展示；发现跳号（如 1,2,4 缺 3）→ 主动拉取 [3,3] 区间补全。

**高并发下的 seqId 分配（需要原子单元时）**：

```java
RLock lock = redisson.getLock("msg:lock:" + conversationId);
lock.lock(5, TimeUnit.SECONDS);
try {
    Long seqId = redisTemplate.opsForValue().increment("msg:seq:" + conversationId);
    message.setSeqId(seqId);
    messageRepo.save(message);
} finally {
    lock.unlock();
}
// 注意：Redis INCR 本身原子，Redisson 锁适合"获取 seq + 写 DB"需要原子单元的场景
```

## 四、在线状态管理

```java
// 连接建立：注册连接
redisTemplate.opsForHash().put("ws:node:map", userId, nodeId);  // userId → nodeId 映射
redisTemplate.expire("ws:heartbeat:" + userId, 90, TimeUnit.SECONDS);  // 在线心跳 key

// 心跳（每 30s）：刷新 TTL
redisTemplate.expire("ws:heartbeat:" + userId, 90, TimeUnit.SECONDS);

// 连接断开：清理
redisTemplate.opsForHash().delete("ws:node:map", userId);
redisTemplate.delete("ws:heartbeat:" + userId);

// 查询在线状态
boolean online = redisTemplate.hasKey("ws:heartbeat:" + userId);
```

**TTL 设计**：心跳周期 30s，TTL = 90s（3 倍），容忍 1-2 次心跳丢失。

## 五、水平扩展：跨节点路由

**一致性 Hash 路由**：userId hash 到固定节点，减少跨节点频率（70-80% 消息本节点处理）。

```java
public void routeMessage(String receiverId, Message msg) {
    String nodeId = (String) redisTemplate.opsForHash().get("ws:node:map", receiverId);
    if (nodeId == null) {
        // 用户离线：存离线消息 + APNs/FCM 推送
        offlineMessageStore.save(receiverId, msg);
        pushNotificationService.sendPush(receiverId, msg);
        return;
    }
    if (nodeId.equals(currentNodeId)) {
        channelManager.push(receiverId, msg);  // 本节点直推
    } else {
        mqTemplate.send("ws:node:" + nodeId, msg);  // 跨节点：发 MQ
    }
}
```

**MQ 跨节点路由架构**：

```
Node A (userId: 101, 203)    Node B (userId: 102, 204)
        │                               │
        └──────────── MQ ───────────────┘
                  (topic 按 nodeId 分区)

流程：101 → Node A → 查 Redis：102 在 Node B → 发 MQ:ws:node:B → Node B 推给 102
```

## 六、群消息扇出方案对比

| 方案 | 适用规模 | 原理 | 优点 | 代价 | 结论 |
|------|---------|------|------|------|------|
| **写扩散（Fan-out Write）** | 小群（≤ 500 人） | 消息写入每个成员 inbox | 读取简单（一次查询）| 写时 N 次操作，写放大 | 小群首选 |
| **读扩散（Fan-in Read）** | 大群（> 500 人） | 只写一条群消息，读时计算 | 节省写放大 | 读取复杂（计算可见消息）| 大群首选 |

```java
// 小群写扩散（N ≤ 500）
public void sendGroupMessage(String groupId, Message msg) {
    List<String> members = groupService.getMembers(groupId);
    for (String memberId : members) {
        inboxService.write(memberId, msg);      // 写入成员 inbox
        routeMessage(memberId, msg);            // 在线推送 or 离线存储
    }
}
```

**未读消息数**：

```java
// HASH 结构：user:{userId}:unread → {conversationId: count}
// 收到消息：HINCRBY（原子加）
redisTemplate.opsForHash().increment("user:" + receiverId + ":unread", conversationId, 1);
// 读消息：HSET 重置为 0
redisTemplate.opsForHash().put("user:" + receiverId + ":unread", conversationId, 0);
```

## 七、存储与冷热分层

```
热数据（近 7 天）：MySQL，索引 (conversation_id, seq_id)
冷数据（7 天以上）：OSS/COS，按月分桶
归档：定时任务每天将超期消息迁移至冷存储，MySQL 软删
```

**消息表核心设计**：

```sql
CREATE TABLE message (
    id             BIGINT AUTO_INCREMENT,
    conversation_id VARCHAR(64) NOT NULL,
    seq_id         BIGINT NOT NULL,          -- 会话内单调递增
    sender_id      VARCHAR(64) NOT NULL,
    msg_type       TINYINT,                  -- 1=文字, 2=图片, 3=语音
    content        TEXT,
    client_msg_id  VARCHAR(64) UNIQUE,       -- 幂等键，防重复入库
    created_at     DATETIME,
    UNIQUE KEY uk_conv_seq (conversation_id, seq_id),  -- 会话内唯一序号
    KEY idx_created (created_at)                        -- 冷热归档依据
);
```

## 八、面试高频追问

### 1. IM 系统如何保证消息不丢失？

**双向 ACK 机制**：①服务端收到消息后立即返回 ServerACK（告知发送方消息已到服务端）；②消息推送给接收方后，接收方返回 ClientACK（告知服务端消息已到客户端）。客户端维护超时重传（5s 无 ServerACK 则重发），服务端以 clientMsgId 做幂等去重。离线消息存储 + 重连拉取保证离线期间不丢。

---

### 2. 如何保证会话内消息有序？

服务端为每个 conversationId 用 Redis INCR 分配单调递增 seqId，Push 给客户端时携带 seqId。客户端按 seqId 升序展示；发现跳号（消息空洞）时主动拉取缺失区间。Redis INCR 本身是原子操作，保证不同并发消息 seqId 不重复不乱序。

---

### 3. 百万连接如何水平扩展？

接入层多节点部署，用一致性 Hash 将 userId 路由到固定节点（减少跨节点路由频率）。跨节点路由通过 MQ 实现：Node A 查 Redis 得到接收方所在 Node B，将消息发到 MQ 的 Node B topic，Node B 消费后通过 WebSocket 推给接收方。用户在线状态和节点映射存 Redis，所有节点共享。

---

### 4. 小群和大群的扇出策略为什么不同？

小群（≤500人）用写扩散：消息写入每个成员 inbox，读取时一次查询即可，实现简单。大群（>500人）写扩散导致写放大（一条消息写 5000 次），改用读扩散：只写一条群消息，成员拉取时按游标查询，计算可见消息列表，节省写操作但增加读复杂度。分界点因业务而异，通常 100-1000 人之间做切换。

## 九、Netty、Redis、幂等设计、MQ 在 IM 架构中的位置

- 接入层长连接管理依赖 [[机制-Netty]]：NIO 主从 Reactor 模型支撑百万级 WebSocket 连接，IdleStateHandler 检测僵尸连接
- 在线状态/seqId/未读数全部依赖 [[概念-Redis]]：Hash 存节点映射、String INCR 分配序号、Hash 存未读计数
- 幂等去重机制属于 [[概念-幂等设计]]：clientMsgId UNIQUE 索引 + Redis SETNX 双重幂等防止重复入库
- 跨节点路由和离线推送依赖 MQ（见 [[机制-RabbitMQ]]）：topic 按 nodeId 分区，保证消息路由到正确节点
