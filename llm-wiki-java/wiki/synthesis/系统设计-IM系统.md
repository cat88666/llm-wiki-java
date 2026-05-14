# 系统设计-IM系统

> IM（即时通信）系统的核心挑战是：消息可靠投递、顺序保证、百万连接下的水平扩展。
>
> 来源：`../../raw/note/Interview/Eson.md`（谁信科技 IM 系统 + WebSocket 长连接 + Redisson 分布式锁）

---

## 一、整体架构

```
                         ┌──────────────────────────────┐
                         │         接入层（L1）           │
Client ─── LB ─────────┤  Netty + WebSocket             │
(Mobile/PC)             │  连接管理 / 心跳检测            │
                         │  一致性 Hash 路由               │
                         └──────────────────────────────┘
                                        │ 业务事件
                         ┌──────────────────────────────┐
                         │         业务层（L2）           │
                         │  消息路由 / 在线状态           │
                         │  群组管理 / 离线消息           │
                         │  消息 seq 分配                │
                         └──────────────────────────────┘
                          │ 读写           │ 发布
             ┌────────────┤                ├──────────────┐
             ▼            ▼                ▼              ▼
          MySQL         Redis            MQ           对象存储
       （消息持久化）  （在线状态/      （跨节点路由/  （图片/视频）
                        seq/未读数）     离线推送）
```

---

## 二、消息可靠投递

HTTP 的 ACK 由 TCP 保证，但 WebSocket 应用层没有内置的消息送达确认。

```
发送方                     服务端                    接收方
  │                          │                         │
  │── send(msg, clientMsgId)→ │                         │
  │                          │ 生成服务端 seqId          │
  │                          │── push(msg, seqId) ────> │
  │<─ ServerACK(clientMsgId)──│                         │
  │                          │                         │
  │                          │<── ClientACK(seqId) ────-│
  │                          │ 更新已读游标              │
```

**关键设计**：
- `clientMsgId`：客户端生成的幂等 ID（UUID），服务端以此去重，防止网络重传导致重复入库。
- `seqId`：服务端为每个会话（conversation）分配的单调递增序号，客户端靠它排序和检测消息空洞。
- 超时重传：客户端发送后 5s 未收到 ServerACK，自动重发（服务端幂等）。
- 消息空洞补全：客户端检测到 `seqId` 不连续，主动拉取缺失区间。

---

## 三、消息顺序保证

**问题根因**：同一会话的两条消息可能经由不同路径到达 DB（网络乱序 + 并发写）。

```java
// 服务端为每个 conversationId 分配单调递增 seq
// 使用 Redis INCR 保证原子性
Long seqId = redisTemplate.opsForValue()
    .increment("msg:seq:" + conversationId);
message.setSeqId(seqId);
// 存 DB
messageRepo.save(message);
// 推送给接收方（带 seqId）
pushToClient(receiverId, message);
```

客户端收到消息后按 `seqId` 升序展示，若发现跳号则拉取补全。

---

## 四、在线状态管理

```java
// 连接建立：注册连接
redisTemplate.opsForHash().put("ws:node:map", userId, nodeId);
redisTemplate.expire("ws:heartbeat:" + userId, 90, TimeUnit.SECONDS);

// 心跳（每30s）：刷新 TTL
redisTemplate.expire("ws:heartbeat:" + userId, 90, TimeUnit.SECONDS);

// 连接断开：清理
redisTemplate.opsForHash().delete("ws:node:map", userId);
redisTemplate.delete("ws:heartbeat:" + userId);

// 查询在线状态
boolean online = redisTemplate.hasKey("ws:heartbeat:" + userId);
```

**TTL 设计**：心跳周期 30s，TTL = 90s（3倍心跳周期），容忍网络抖动导致的1-2次心跳丢失。

---

## 五、水平扩展：跨节点消息路由

```
Node A (userId: 101, 203)    Node B (userId: 102, 204)
         │                              │
         │                              │
         └─────────── MQ ──────────────┘
                   (topic 按 nodeId 分)

流程：
1. 101 发消息给 102
2. 消息到 Node A（101 所在节点）
3. Node A 查 Redis：102 在 Node B
4. Node A 发消息到 MQ topic: ws:node:B
5. Node B 消费 MQ，通过 WebSocket 推送给 102
```

```java
public void routeMessage(String receiverId, Message msg) {
    String nodeId = (String) redisTemplate.opsForHash().get("ws:node:map", receiverId);
    if (nodeId == null) {
        // 用户离线，存离线消息
        offlineMessageStore.save(receiverId, msg);
        pushNotificationService.sendPush(receiverId, msg); // APNs/FCM
        return;
    }
    if (nodeId.equals(currentNodeId)) {
        // 本节点直接推送
        channelManager.push(receiverId, msg);
    } else {
        // 跨节点：发 MQ
        mqTemplate.send("ws:node:" + nodeId, msg);
    }
}
```

**一致性 Hash 路由**：userId 哈希到固定节点，减少跨节点路由频率（70-80% 消息本节点处理）。

---

## 六、群消息扇出策略

| 群规模 | 策略 | 原因 |
|--------|------|------|
| 小群（≤ 500 人） | **写扩散（Fan-out Write）** | 消息写入每个成员的 inbox；读取简单（一次查询），写时有 N 次操作 |
| 大群（> 500 人） | **读扩散（Fan-in Read）** | 只写一条群消息；成员拉取时计算可见消息列表；节省写放大 |

```java
// 小群写扩散
public void sendGroupMessage(String groupId, Message msg) {
    List<String> members = groupService.getMembers(groupId);
    for (String memberId : members) {
        // 批量写入每个成员的会话收件箱
        inboxService.write(memberId, msg);
        routeMessage(memberId, msg); // 在线则推送，离线则存离线消息
    }
}
```

---

## 七、未读消息数

```java
// HASH 结构：user:{userId}:unread → {conversationId: count}
// 收到消息时原子 HINCRBY
redisTemplate.opsForHash().increment(
    "user:" + receiverId + ":unread", conversationId, 1);

// 用户读消息时 HSET 重置为 0
redisTemplate.opsForHash().put(
    "user:" + receiverId + ":unread", conversationId, 0);

// 获取总未读
Map<Object, Object> unread = redisTemplate.opsForHash()
    .entries("user:" + receiverId + ":unread");
long total = unread.values().stream().mapToLong(v -> (Long) v).sum();
```

---

## 八、Redisson 锁的使用场景

```java
// 场景：同一用户并发发送消息时，seq 分配必须串行
RLock lock = redisson.getLock("msg:lock:" + conversationId);
lock.lock(5, TimeUnit.SECONDS);
try {
    Long seqId = redisTemplate.opsForValue().increment("msg:seq:" + conversationId);
    message.setSeqId(seqId);
    messageRepo.save(message);
} finally {
    lock.unlock();
}
```

**注意**：seq 分配使用 Redis INCR 本身是原子的，Redisson 锁适合需要"获取 seq + 写 DB"作为一个原子单元的场景。高吞吐下可以改用 Redis 单 key 的原子 INCR + 异步写 DB，减少锁持有时间。

---

## 九、消息存储冷热分层

```
热数据（近 7 天）：MySQL，索引 (conversationId, seqId)
冷数据（7 天以上）：对象存储（OSS/COS），按月分桶
归档：定时任务每天将超期消息迁移至冷存储，MySQL 删除或软删
```

```sql
-- 消息表设计
CREATE TABLE message (
    id            BIGINT AUTO_INCREMENT,
    conversation_id VARCHAR(64) NOT NULL,
    seq_id        BIGINT NOT NULL,          -- 会话内单调递增
    sender_id     VARCHAR(64) NOT NULL,
    msg_type      TINYINT,                  -- 1=文字, 2=图片, 3=语音
    content       TEXT,
    client_msg_id VARCHAR(64) UNIQUE,       -- 幂等键
    created_at    DATETIME,
    UNIQUE KEY uk_conv_seq (conversation_id, seq_id),
    KEY idx_created (created_at)
);
```

---

## 十、面试 STAR 模板

**Q：IM 系统最难的技术挑战是什么？**

> **S**：谁信科技 IM 系统日活 50 万，高峰期万级并发消息，还要保证消息不丢、顺序正确。
>
> **T**：解决有状态 WebSocket 连接水平扩展，同时保证消息可靠性。
>
> **A**：①接入层 Netty + WebSocket，一致性 Hash 按 userId 路由，减少跨节点路由；②Redis INCR 分配单调 seqId 保证会话内有序；③跨节点消息路由通过 MQ（topic = nodeId）转发；④Redisson 分布式锁保障高并发下 seq 分配原子性；⑤离线用户走 APNs/FCM 推送，消息存 Redis 离线队列，重连时补发。
>
> **R**：消息送达率 99.99%，P99 推送延迟 < 100ms，单机支撑 5 万并发连接。
