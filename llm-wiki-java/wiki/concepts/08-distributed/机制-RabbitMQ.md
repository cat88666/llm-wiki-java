---
type: concept
status: active
name: "RabbitMQ"
layer: L7
aliases: ["RabbitMQ", "AMQP", "消息队列", "死信队列", "延迟消息", "消费端限流", "镜像集群", "Publisher Confirm", "本地消息表", "消息可靠性", "幂等消费", "Exchange", "RoutingKey"]
tags: ["#distributed"]
related:
  - "[[机制-Kafka]]"
  - "[[概念-幂等设计]]"
  - "[[概念-Redis]]"
sources:
  - "../../../raw/note/Hollis/RabbitMQ/✅rabbitMQ的整体架构是怎么样的？.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅RabbitMQ是怎么做消息分发的？.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅RabbitMQ如何保证消息不丢.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅如何保障消息一定能发送到RabbitMQ.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅RabbitMQ如何防止重复消费.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅RabbitMQ 是如何保证高可用的.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅rabbitMQ如何实现延迟消息？.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅什么是RabbitMQ的死信队列？.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅RabbitMQ如何实现消费端限流.md"
  - "../../../raw/note/Hollis/RabbitMQ/✅介绍下RabbitMQ的事务机制.md"
  - "../../../raw/note/Hollis/场景题/✅MQ出现消息乱序了如何解决？.md"
  - "../../../raw/note/Hollis/场景题/✅Spring Event和MQ有什么区别？各自适用场景是什么？.md"
  - "../../../raw/note/Hollis/场景题/✅消息队列使用拉模式好还是推模式好？为什么？.md"
  - "../../../raw/note/Hollis/场景题/✅为什么不建议使用MQ实现订单到期关闭？.md"
  - "../../../raw/note/Hollis/场景题/✅用了本地消息表的方案，如果下游执行失败了上游如何回滚？.md"
  - "../../../raw/note/Hollis/场景题/✅为了避免丢消息问题需要落表，如何设计这张消息表？.md"
created: 2026-05-08
updated: 2026-05-15
lint_notes: ""
---

# RabbitMQ

> 基于 AMQP 协议的消息中间件：通过 Exchange → Binding → Queue 三层路由模型，实现灵活的消息分发，以持久化 + Confirm + ACK 三道防线保障消息可靠投递；面向复杂路由和灵活消费模式，以吞吐量为代价换来丰富的消息语义。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 生产消费解耦、路由 Exchange 核心思想 |
| [二、架构模型与 6 种工作模式](#二架构模型与-6-种工作模式) | AMQP 三层路由、模式选型 |
| [三、消息可靠性三道防线](#三消息可靠性三道防线) | Confirm/持久化/ACK |
| [四、死信队列](#四死信队列) | 死信触发条件、延迟消息实现 |
| [五、消费端限流](#五消费端限流) | basicQos 防压垮 |
| [六、高可用模式](#六高可用模式) | 单机/普通集群/镜像集群 |
| [七、场景综合](#七场景综合) | Spring Event vs MQ、拉推模式、乱序解法 |
| [八、本地消息表](#八本地消息表) | 最终一致性核心实现 |
| [九、关键权衡](#九关键权衡) | Confirm vs 事务、持久化 vs 性能 |
| [十、与其他概念的关系](#十与其他概念的关系) | Kafka、幂等设计、Redis |
| [十一、应用边界](#十一应用边界) | 适合 vs 不适合场景 |

## 一、第一性原理

消息队列的核心问题是：**生产者和消费者如何解耦，同时保证消息不丢失、不重复**。RabbitMQ 的答案是 AMQP 协议——引入 Exchange 作为路由中心，生产者只管发到 Exchange，Exchange 按规则路由到 Queue，消费者只管从 Queue 消费，双方完全不感知对方。

与 Kafka "顺序写磁盘换吞吐量"不同，RabbitMQ 的核心诉求是**灵活路由 + 多种消费模式**，以吞吐量为代价换来更丰富的消息语义。

## 二、架构模型与 6 种工作模式

### 架构模型

```
Producer
  ↓ publish（指定 Exchange + RoutingKey）
Exchange（fanout / direct / topic / headers）
  ↓ 按 Binding 规则路由
Queue（持久化，存真正的消息）
  ↓ push/pull
Consumer（ACK 确认后消息删除）
```

**VHost**：命名空间隔离，不同应用使用不同 VHost，Exchange/Queue/权限互不干扰。

### 6 种工作模式

| 模式 | 特点 | 适用场景 |
|------|------|---------|
| 简单模式 | 1 Producer → 1 Queue → 1 Consumer | 最基础场景 |
| 工作队列 | 1 Queue → N Consumer（竞争消费）| 任务并发处理（如异步邮件发送）|
| 发布/订阅 | Fanout Exchange 广播到所有绑定 Queue | 日志、事件通知 |
| 路由模式 | Direct Exchange 精确匹配 RoutingKey | 按类型分流（如 error/info 日志）|
| 主题模式 | Topic Exchange 通配符匹配（`*`/`#`）| 复杂路由规则（`*.usa.#`）|
| RPC 模式 | 通过 MQ 实现请求-响应 | 异步 RPC（少用，不如直接 HTTP）|

## 三、消息可靠性三道防线

### ① 生产者侧：Publisher Confirm + Publisher Returns

```
Publisher Confirm：消息到达 Exchange 后，MQ 回调 ACK/NACK
Publisher Returns：消息无法路由到任何 Queue 时回调（需设 mandatory=true）
```

- **Confirm 异步模式**（推荐）：发送后不阻塞，通过回调处理 ACK/NACK；性能是同步事务的 3 倍
- **事务机制**（`txSelect/txCommit/txRollback`）：同步阻塞，性能差（仅约 Confirm 的 1/3），不推荐

### ② 存储侧：持久化

- Queue `durable=true`：队列元数据持久化（重启不丢队列）
- Message `deliveryMode=2`：消息持久化到磁盘（重启不丢消息）
- **注意**：持久化是异步的，极端宕机场景仍有丢消息风险；100% 不丢需引入**本地消息表**兜底

### ③ 消费者侧：ACK 机制

```java
// 消费成功后手动发送 ACK，MQ 收到后才删除消息
channel.basicAck(deliveryTag, false);

// 消费失败发送 NACK（requeue=true 重新入队，false 进死信队列）
channel.basicNack(deliveryTag, false, false);
```

**autoAck=false** 是生产必须配置：自动 ACK 时消息被推送就确认，消费者崩溃后消息丢失。

## 四、死信队列

**消息成为死信的三种情况**：
1. 消息被拒绝（NACK + requeue=false）
2. 消息 TTL 过期
3. 队列长度超限

**死信流向**：原 Queue → 死信 Exchange（`x-dead-letter-exchange`）→ 死信 Queue → 死信 Consumer 处理。

### 延迟消息实现

**方案一：TTL + 死信队列**：
```
消息设置 TTL → 投入不消费的 Queue（只设 TTL 不消费）→ 过期后流入死信 Queue → 消费
```
**缺点**：队头阻塞（仅检查队首消息是否过期，队头未过期则后面的消息一直等待）。

**方案二：rabbitmq-delayed-message-exchange 插件**：
```
消息先存 Mnesia DB → 定时器触发后投递 Exchange → 正常队列消费
```
**优点**：无队头阻塞，任意延迟时间。  
**缺点**：单点存储存在丢失风险；百万级以上大量延迟消息性能差（考虑 RocketMQ 延迟消息）。

**为什么不建议用 MQ 实现订单超时关闭**：延迟不够精准（网络抖动影响）、消息可靠性要求高、重复消费需幂等、已投递消息难取消。更推荐：Redisson `RDelayedQueue`、调度框架扫表。

## 五、消费端限流

```java
// 最多 10 条未确认消息，消费完再拉新的（防止消息积压时消费者被打垮）
channel.basicQos(10);
channel.basicConsume(queue, false, consumer); // autoAck=false
// 处理完后手动 ACK
channel.basicAck(deliveryTag, false);
```

`basicQos` 是消费端背压机制：限制 Broker 未确认消息的最大数量，消费者处理慢时 Broker 不会继续推送。

## 六、高可用模式

| 模式 | 数据分布 | 高可用程度 | 性能 |
|------|---------|-----------|------|
| 单机模式 | 单节点 | 无 | 最高 |
| 普通集群 | 元数据同步，消息只在一个节点 | 节点故障消息不可读 | 高 |
| **镜像集群**（推荐）| 消息在所有节点同步一份 | 节点故障不影响服务 | 低（同步开销大）|

生产推荐**镜像集群**：队列元数据和消息数据在所有节点保存副本，任意节点宕机不影响消息读写。

## 七、场景综合

### Spring Event vs MQ

| 维度 | Spring Event | MQ |
|------|-------------|----|
| 作用范围 | 同 JVM | 跨服务/跨进程 |
| 可靠性 | 进程崩溃即丢失 | 可持久化 |
| 延迟 | 极低（同线程/异步线程）| 毫秒级 |
| 运维成本 | 低 | 高 |
| 典型场景 | 同服务模块解耦 | 跨服务异步通信、削峰 |

**原则**：同服务内解耦优先 Spring Event；跨服务、持久化、削峰才用 MQ。

### 拉模式 vs 推模式

| | 拉模式（Kafka）| 推模式（RabbitMQ）|
|--|---|---|
| 控制权 | 消费者主动拉 | Broker 主动推 |
| 实时性 | 略低（长轮询等待数据）| 更高 |
| 过载风险 | 不易压垮消费者（消费者控速）| 需配合 basicQos 限流 |
| 批量消费 | 天然适合 | 需额外控制 |

### 消息乱序：四种解法

1. **顺序消息**：同一业务 key 进同一 Partition / Queue（路由保证有序）
2. **前置状态判断**：消息携带 `beforeStatus`，消费时校验状态机（判断前置状态是否符合预期）
3. **事件表 + 异步重试**：先落事件表，再异步按序处理（通过 offset/sequence_no 排序）
4. **业务容忍乱序**：通过最终状态覆盖或幂等推进规避顺序依赖

## 八、本地消息表

### 消息表设计

```sql
CREATE TABLE message_record (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    msg_id          VARCHAR(64) NOT NULL UNIQUE,
    biz_id          VARCHAR(64) NOT NULL,
    biz_type        VARCHAR(32) NOT NULL,
    topic           VARCHAR(64),
    payload         TEXT,
    state           TINYINT DEFAULT 0,   -- 0:PENDING, 1:SENT, 2:FAILED
    retry_count     INT DEFAULT 0,
    next_retry_time DATETIME,
    create_time     DATETIME,
    update_time     DATETIME,
    INDEX idx_state_retry (state, next_retry_time)
);
```

**双保险**：业务操作和消息表写入同一事务；定时任务扫描 state=PENDING 的消息重投。

### 下游失败时上游如何处理

原则：**不回滚上游已提交业务，只保证下游最终一致**。

流程：上游业务提交成功 → 消息表写入 PENDING → 定时任务扫描 → 下游失败则重试 → 超阈值进入 FAILED + 报警 → 人工补偿或自动降级。

## 九、关键权衡

| 权衡点 | 结论 |
|--------|------|
| Confirm 异步 vs 事务同步 | Confirm 性能高（异步回调），事务严格但吞吐量仅约 Confirm 的 1/3，优先用 Confirm |
| 持久化 vs 性能 | deliveryMode=2 触发磁盘 I/O，高吞吐场景可考虑内存队列 + 定期 checkpoint |
| 镜像集群 vs 普通集群 | 镜像同步写放大导致性能下降，可靠性要求高时用镜像；规模大/追求性能时考虑 Kafka |
| 死信 TTL vs 延迟插件 | TTL 方案有队头阻塞缺陷；插件方案稳定但数量级限制，超大规模延迟消息考虑 RocketMQ |

## 十、与其他概念的关系

- 区别于 [[机制-Kafka]]：Kafka 面向高吞吐流处理（顺序写、Consumer Group offset 管理）；RabbitMQ 面向灵活路由和任务队列（AMQP、Exchange 路由、消费后删除）；金融/订单业务倾向 RabbitMQ/RocketMQ，日志/大数据倾向 Kafka
- 依赖 [[概念-幂等设计]]：网络延迟可能导致重复消费，消费者必须结合唯一标识实现幂等（"一锁、二判、三更新"）
- 关联 [[概念-Redis]]：分布式锁用于幂等控制；Redis 也可以替代本地消息表记录消息处理状态

## 十一、应用边界

**适合 RabbitMQ**：需要灵活路由规则（多种 Exchange 类型）、任务队列（工作队列模式）、延迟消息（死信/插件）、消息量中等（万级 TPS）、已有 Spring AMQP 生态。

**不适合 RabbitMQ**：超高吞吐日志流水线（用 Kafka）；需要消息回放/时间窗口消费（Kafka offset 模型更合适）；延迟消息量超百万（考虑 RocketMQ 或时间轮方案）；对 MQ 吞吐有极端要求。
