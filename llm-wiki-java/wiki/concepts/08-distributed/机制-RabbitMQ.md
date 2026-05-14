---
type: concept
status: active
name: "RabbitMQ"
layer: L7
aliases: ["RabbitMQ", "AMQP", "消息队列", "死信队列", "延迟消息", "消费端限流", "镜像集群", "Publisher Confirm"]
tags: ["#distributed"]
related:
  - "[[机制-Kafka]]"
  - "[[机制-消息队列可靠性]]"
  - "[[概念-幂等设计]]"
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
created: 2026-05-08
updated: 2026-05-08
lint_notes: ""
---

# RabbitMQ

> 基于 AMQP 协议的消息中间件：通过 Exchange → Binding → Queue 三层路由模型，实现灵活的消息分发，以持久化 + Confirm + ACK 三道防线保障消息可靠投递。

## 第一性原理

消息队列的核心问题是：**生产者和消费者如何解耦，同时保证消息不丢失、不重复**。RabbitMQ 的答案是 AMQP 协议——引入 Exchange 作为路由中心，生产者只管发到 Exchange，Exchange 按规则路由到 Queue，消费者只管从 Queue 消费，双方完全不感知对方。

与 Kafka "顺序写磁盘换吞吐量"不同，RabbitMQ 的核心诉求是**灵活路由 + 多种消费模式**，以吞吐量为代价换来更丰富的消息语义。

## 核心机制

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
| 工作队列 | 1 Queue → N Consumer（竞争消费） | 任务并发处理 |
| 发布/订阅 | Fanout Exchange 广播到所有绑定 Queue | 日志、事件通知 |
| 路由模式 | Direct Exchange 精确匹配 RoutingKey | 按类型分流 |
| 主题模式 | Topic Exchange 通配符匹配（`*`/`#`） | 复杂路由规则 |
| RPC 模式 | 通过 MQ 实现请求-响应 | 异步 RPC |

### 消息可靠性三道防线

**① 生产者侧：Publisher Confirm + Publisher Returns**
- `Publisher Confirm`：消息到达 Exchange 后，MQ 回调 ACK/NACK
- `Publisher Returns`：消息无法路由到任何 Queue 时回调（配合 `mandatory=true`）
- 备选：事务机制（`txSelect/txCommit/txRollback`），性能差，不推荐

**② 存储侧：持久化**
- Queue `durable=true`：队列元数据持久化（重启不丢队列）
- Message `deliveryMode=2`：消息持久化到磁盘（重启不丢消息）
- 注意：持久化是异步的，极端宕机场景仍有丢消息风险；100% 不丢需引入**本地消息表**兜底

**③ 消费者侧：ACK 机制**
- 消费成功后手动发送 ACK，MQ 收到后才删除消息
- 消费失败发送 NACK，MQ 重新投递（可配置重试次数和死信路由）

### 死信队列（DLQ）

消息成为死信的三种情况：消息被拒绝（NACK + requeue=false）、消息 TTL 过期、队列长度超限。

死信流向：原 Queue → 死信 Exchange（x-dead-letter-exchange）→ 死信 Queue。

**延迟消息实现**：
- **方案一：TTL + 死信队列**：消息设置 TTL 投入正常队列（不消费），过期后流入死信队列再消费。缺点：队头阻塞（仅检查队首消息是否过期）
- **方案二：rabbitmq-delayed-message-exchange 插件**：消息先存 Mnesia DB，定时器触发后投递 Exchange。无队头阻塞，但单点存储存在丢失风险，不适合百万级大量延迟消息

### 消费端限流（basicQos）

```java
// 最多 10 条未确认消息，消费完再拉新的
channel.basicQos(10);
channel.basicConsume(queue, false, consumer); // autoAck=false
// 处理完后手动 ACK
channel.basicAck(deliveryTag, false);
```

防止消息积压时消费者被打垮。

### 高可用模式

| 模式 | 数据分布 | 高可用程度 | 性能 |
|------|---------|-----------|------|
| 单机模式 | 单节点 | 无 | 最高 |
| 普通集群 | 元数据同步，消息只在一个节点 | 节点故障消息不可读 | 高 |
| 镜像集群 | 消息在所有节点同步一份 | 节点故障不影响服务 | 低（同步开销大）|

生产推荐**镜像集群**：队列元数据和消息数据在所有节点保存副本，任意节点宕机不影响消息读写。

## 关键权衡

1. **Confirm 异步 vs 事务同步**：Confirm 性能高（异步回调），事务严格但吞吐量仅约为 Confirm 的 1/3，优先用 Confirm
2. **持久化 vs 性能**：`deliveryMode=2` 触发磁盘 I/O，高吞吐场景可考虑内存队列 + 定期 checkpoint，以性能换可靠性
3. **镜像集群 vs 普通集群**：镜像同步写放大导致性能下降，规模小/可靠性要求高时用镜像；规模大/追求性能时考虑 Kafka
4. **死信 TTL vs 延迟插件**：TTL 方案有队头阻塞缺陷；插件方案稳定但数量级限制（百万以下），超大规模延迟消息考虑 RocketMQ

## 与其他概念的关系

- 区别于 [[机制-Kafka]]：Kafka 面向高吞吐流处理（顺序写、Consumer Group offset 管理）；RabbitMQ 面向灵活路由和任务队列（AMQP、Exchange 路由、消费后删除）；金融/订单业务倾向 RabbitMQ/RocketMQ，日志/大数据倾向 Kafka
- 依赖 [[概念-幂等设计]]：网络延迟可能导致重复消费，消费者必须结合唯一标识实现幂等（"一锁、二判、三更新"）
- 支撑了 [[机制-消息队列可靠性]]：三道防线（Confirm + 持久化 + ACK）是消息队列可靠性的标准实现模式
- 关联 [[系统设计-支付系统]]：死信队列 + TTL 是实现订单超时关闭的常见方案

## 应用边界

**适合 RabbitMQ**：需要灵活路由规则（多种 Exchange 类型）、任务队列（工作队列模式）、延迟消息（死信/插件）、消息量中等（万级 TPS）。

**不适合 RabbitMQ**：超高吞吐日志流水线（用 Kafka）、需要消息回放/时间窗口消费（Kafka offset 模型更合适）、延迟消息量超百万（考虑 RocketMQ 或时间轮方案）。
