---
type: concept
status: active
name: "RocketMQ"
layer: L7
aliases: ["RocketMQ", "事务消息", "延迟消息", "NameServer", "CommitLog", "ConsumeQueue", "半消息", "回查机制", "死信队列DLQ"]
tags: ["#distributed"]
related:
  - "[[机制-Kafka]]"
  - "[[机制-RabbitMQ]]"
  - "[[概念-幂等设计]]"
  - "[[机制-Seata]]"
sources:
  - "../../raw/note/Interview/Eson.md"
created: 2026-05-08
updated: 2026-05-15
lint_notes: ""
---

# RocketMQ

> RocketMQ 是阿里开源的分布式消息中间件，核心差异是原生支持**事务消息**和**延迟消息**，通过 NameServer + CommitLog + ConsumeQueue 架构实现高吞吐，在金融/支付场景下提供比 Kafka 更强的消息可靠性语义。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | Kafka 的缺失能力，支付场景的需求 |
| [二、架构组件](#二架构组件) | NameServer/CommitLog/ConsumeQueue |
| [三、消息类型](#三消息类型) | 普通/延迟/顺序/事务消息 |
| [四、事务消息（重点）](#四事务消息重点) | 半消息机制、本地事务、回查 |
| [五、消费模式与重试](#五消费模式与重试) | 集群消费/广播消费、指数退避、DLQ |
| [六、消息积压治理](#六消息积压治理) | 积压策略递进 |
| [七、与其他 MQ 的对比](#七与其他-mq-的对比) | vs Kafka vs RabbitMQ |
| [八、关键权衡](#八关键权衡) | 事务消息 vs 本地消息表 |
| [九、与其他概念的关系](#九与其他概念的关系) | Kafka、RabbitMQ、幂等设计、Seata |
| [十、应用边界](#十应用边界) | 适合 vs 不适合场景 |

## 一、第一性原理

Kafka 在海量日志场景下吞吐极高，但两个核心能力缺失：
1. **事务消息**：本地事务与消息发送的原子性（Kafka 事务语义需要 Exactly-Once + 外部协调，复杂度高）
2. **精确延迟消息**：定时投递（Kafka 需要自己实现，缺乏原生支持）

支付/金融场景需要"消息不丢 + 本地事务一致 + 延迟重试"，RocketMQ 针对这些场景做了原生优化。

## 二、架构组件

```
Producer ─── NameServer（路由注册中心，AP，无状态）
           ─── Broker Cluster（Master/Slave，存消息）
Consumer ─── NameServer（拉取路由表）
           ─── Broker（拉消息）
```

| 组件 | 职责 | 关键设计 |
|------|------|---------|
| **NameServer** | 轻量级注册中心，Broker 每 30s 心跳注册，Producer/Consumer 每 30s 拉取路由表 | AP 设计，各节点不互通（区别于 Kafka 的 ZK/KRaft）|
| **Broker Master** | 读写服务；Dledger 模式（4.x+）支持 Raft 主从切换 | Dledger 保证 Master 宕机时 Slave 自动接管 |
| **Broker Slave** | 只读（可承接部分消费流量）| 主从数据异步复制（默认）或同步复制（SYNC_MASTER）|
| **CommitLog** | 所有 Topic 的消息顺序写入同一文件（1GB 滚动），是 RocketMQ 高吞吐的关键 | 顺序追加，写性能与 Kafka Partition 相当 |
| **ConsumeQueue** | 每个 Topic/Queue 一个索引文件，记录消息在 CommitLog 的 offset | Consumer 通过 ConsumeQueue 定位消息，避免扫描 CommitLog |

**CommitLog vs Kafka Segment 的差异**：Kafka 每个 Partition 独立一组文件；RocketMQ 所有 Topic 共享同一 CommitLog，多 Topic 下写放大更小，吞吐更均匀。

## 三、消息类型

| 类型 | 场景 | 核心机制 |
|------|------|---------|
| 普通消息 | 日志、通知 | Producer → Broker → Consumer |
| **延迟消息** | 订单超时关闭 | 固定 Level 存入 `SCHEDULE_TOPIC`，到期后投递到目标 Topic |
| 顺序消息 | 同一订单的创建/支付/完成 | 同一 MessageQueue 内严格有序；全局有序需单 Queue（吞吐降低）|
| **事务消息** | 支付 + 消息原子性 | 半消息机制（见下节）|

**延迟消息局限**：
- RocketMQ 4.x 只支持固定 18 个 Level（1s/5s/10s/30s/1m/2m/3m/4m/5m/6m/7m/8m/9m/10m/20m/30m/1h/2h），无法任意延迟
- RocketMQ 5.x 支持任意精度延迟消息（TimerWheel 实现）

## 四、事务消息（重点）

**解决问题**：本地事务执行成功但消息发送失败（或反之），导致数据不一致。

### 半消息机制流程

```
Producer                    Broker                    Consumer
   │                           │                          │
   │── 发送 Half Message ──────>│（存入 Half Topic，不可见）│
   │<─ Broker ACK ─────────────│                          │
   │                           │                          │
   │── 执行本地事务             │                          │
   │   (转账/扣款/下单)         │                          │
   │                           │                          │
   │── Commit/Rollback ────────>│                          │
   │   （本地事务成功 → Commit）  │── 移入正常 Topic ───────>│ 消费
   │   （本地事务失败 → Rollback）│── 删除消息              │
   │                           │                          │
   │   ← 如果未收到 Commit/Rollback（超时）               │
   │   Broker 回查 Producer 本地事务状态（Check-Back）     │
```

### Spring 集成代码

```java
@RocketMQTransactionListener
public class PaymentTransactionListener implements RocketMQLocalTransactionListener {
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            paymentService.deductBalance(msg);  // 本地事务
            return COMMIT;
        } catch (Exception e) {
            return ROLLBACK;
        }
    }
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        // Broker 回查：根据消息中的 orderId 查 DB 判断事务是否成功
        return paymentService.isDeducted(msg) ? COMMIT : ROLLBACK;
    }
}
```

**事务消息 vs 本地消息表**：

| 维度 | 事务消息 | 本地消息表 |
|------|---------|-----------|
| 实时性 | 秒级 | 分钟级（定时任务扫描）|
| 开发复杂度 | 中（需实现回查接口）| 低（消息表 + 定时任务）|
| MQ 依赖 | 强（仅 RocketMQ 支持）| 弱（任意 MQ）|
| 适用场景 | 高实时性，已用 RocketMQ | 已有定时任务体系，消息量小 |

## 五、消费模式与重试

**集群消费（默认）**：同一 ConsumerGroup 的多个实例，每条消息只被一个实例消费（负载均衡）。

**广播消费**：每个实例都消费所有消息（如配置下发、本地缓存刷新）。

### 消费失败重试

Consumer 消费失败返回 `RECONSUME_LATER`，Broker 按**指数退避**重投：

```
第 1 次失败 → 10s 后重投
第 2 次失败 → 30s 后重投
第 3 次失败 → 1min 后重投
...
第 16 次失败 → 2h 后重投
第 17 次（超过 16 次）→ 进入死信队列（DLQ）
```

**死信队列**：`%DLQ%{consumerGroup}`，需单独 Consumer 订阅处理，可触发报警、人工介入或自动补偿。

### 消费者推拉模式

RocketMQ 是**长轮询 Pull 伪装成 Push**：Consumer 主动拉取 Broker，Broker 若无消息则 hold 请求 15s，有消息立即返回。相比真正的 Push，Consumer 可以自控消费速率，避免消息积压时被压垮。

## 六、消息积压治理

```
积压量 = 生产速率 - 消费速率

治理策略（按严重程度递进）：
1. 扩 Consumer 实例（受限于 MessageQueue 数量，Consumer ≤ Queue）
2. 扩 MessageQueue（需提前规划，不能动态缩减）
3. 降级：跳过非关键消息（需业务支持，谨慎操作）
4. 清理：直接跳过指定 offset（数据丢失风险，仅限非关键消息）
5. 重建：新 Topic 扩容，老 Topic 消息迁移消费（有序消费场景复杂）
```

## 七、与其他 MQ 的对比

| 维度 | RocketMQ | Kafka | RabbitMQ |
|------|---------|-------|----------|
| 事务消息 | ✅ 原生半消息机制 | ❌ 需外部协调 | ✅ 基础事务 |
| 延迟消息 | ✅ 固定 Level（4.x）/ 任意（5.x）| ❌ 需自实现 | ✅ 插件支持任意 |
| 吞吐量 | 中高（CommitLog 顺序写）| 极高（分区顺序写 + 零拷贝）| 中（万级 TPS）|
| 消息积压 | 中等（ConsumeQueue 设计）| 极强（适合超大规模积压）| 弱（内存压力大）|
| 路由能力 | 弱（Tag 过滤）| 无（Partition 路由）| 强（多种 Exchange 类型）|
| 适用场景 | 金融支付、电商、事务一致性 | 日志收集、流计算、大数据管道 | 复杂路由、AMQP 兼容 |

## 八、关键权衡

| 权衡点 | 结论 |
|--------|------|
| RocketMQ vs Kafka | RocketMQ 功能更强（事务/延迟），Kafka 吞吐更高；支付场景选 RocketMQ，日志采集选 Kafka |
| 同步 vs 异步复制 | SYNC_MASTER 数据不丢但写入延迟高；ASYNC_MASTER 性能好但主宕机有少量数据丢失风险 |
| NameServer AP vs ZK CP | NameServer 无选举开销，简单高可用；代价是 Producer/Consumer 需定期拉取路由表（最多 30s 延迟感知 Broker 变化）|
| 延迟消息 Level 限制 | 4.x 固定 Level 是历史设计，5.x 任意精度是改进；生产建议升级 5.x |

## 九、与其他概念的关系

- 区别于 [[机制-Kafka]]：Kafka 吞吐更高，RocketMQ 事务/延迟消息更强，支付系统选 RocketMQ，日志埋点选 Kafka
- 区别于 [[机制-RabbitMQ]]：RabbitMQ 协议更标准（AMQP），适合企业集成；RocketMQ 针对大规模互联网场景优化
- 依赖 [[概念-幂等设计]]：无论哪种 MQ，消费端都必须做幂等（消息可能重投，重试机制本身就是幂等性的来源）
- 与 [[机制-Seata]] 协作：Seata 的 Saga 模式可以用 RocketMQ 事务消息替代本地消息表实现可靠消息

## 十、应用边界

**用 RocketMQ**：
- 支付/金融场景（转账成功通知、对账消息）— 事务消息保证本地事务与消息原子
- 订单超时关闭（延迟消息，30min/1h 后触发关闭逻辑）
- 需要精确控制消费速率、有重试和死信需求的业务流

**不用 RocketMQ（选 Kafka）**：
- 日志收集（量极大，不需要事务消息）
- 流计算（Flink/Spark 的 Kafka 连接器生态更成熟）
- 需要超长消息保留（Kafka 的时间索引保留效率更高）
