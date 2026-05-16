---
type: concept
status: active
name: "RocketMQ"
layer: L7
aliases: ["RocketMQ", "事务消息", "延迟消息", "NameServer", "CommitLog", "ConsumeQueue", "IndexFile", "半消息", "回查机制", "顺序消息", "MessageQueueSelector", "MessageListenerOrderly", "死信队列DLQ", "Dledger", "同步刷盘", "同步双写"]
tags: ["#distributed"]
related:
  - "[[机制-Kafka]]"
  - "[[机制-RabbitMQ]]"
  - "[[概念-幂等设计]]"
  - "[[机制-Seata]]"
sources:
  - "../../raw/note/Interview/Eson.md"
  - "../../../raw/note/tuling/08-mq.md"
created: 2026-05-08
updated: 2026-05-16
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
| [五、可靠性与高性能设计](#五可靠性与高性能设计) | 生产/存储/消费三阶段可靠性，CommitLog/PageCache/零拷贝 |
| [六、顺序消息机制](#六顺序消息机制) | 局部顺序、MessageQueueSelector、三把锁 |
| [七、消费模式与重试](#七消费模式与重试) | 集群消费/广播消费、指数退避、DLQ |
| [八、消息积压治理](#八消息积压治理) | 积压根因、紧急处理、预防 |
| [九、集群部署与生产调优](#九集群部署与生产调优) | 多 Master、多 Slave、Dledger、关键参数 |
| [十、与其他 MQ 的对比](#十与其他-mq-的对比) | vs Kafka vs RabbitMQ |
| [十一、关键权衡](#十一关键权衡) | 事务消息 vs 本地消息表 |
| [十二、与其他概念的关系](#十二与其他概念的关系) | Kafka、RabbitMQ、幂等设计、Seata |
| [十三、应用边界](#十三应用边界) | 适合 vs 不适合场景 |
| [十四、面试速答](#十四面试速答) | 不丢、不重、有序、堆积、事务 |

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
| **IndexFile** | 按消息 Key 和时间建立索引 | 支持按 Key/时间快速查询消息 |
| **Topic** | 消息逻辑分类单位 | Producer 发到 Topic，Consumer 订阅 Topic |
| **MessageQueue** | Topic 的物理队列 | 一个 Topic 可有多个 Queue，顺序消息依赖同业务 ID 落同 Queue |

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

### 延迟消息实现原理

1. Producer 发送消息时指定延迟级别。
2. Broker 将消息先写入内部延迟 Topic：`SCHEDULE_TOPIC_XXXX`。
3. 定时任务扫描到期消息。
4. 到期后重新投递到真实 Topic。

如果需要任意时间延迟，常见方案是 RocketMQ 5.x Timer Message、DB 定时任务二次投递，或时间轮。

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

### 事务消息异常追问

| 问题 | 回答 |
| --- | --- |
| 半消息第一次发送失败怎么办 | 本地事务不执行，不会产生不一致；由调用方重试即可 |
| 本地事务成功但 Commit 失败怎么办 | Broker 定时回查 Producer，根据本地事务状态推进 Commit/Rollback |
| 一直回查不到明确状态怎么办 | 超过回查次数或过期后按策略回滚/进入异常处理，业务侧需有补偿表兜底 |
| 为什么不用“先本地事务后发消息” | 本地事务提交后消息发送失败会导致下游永远收不到事件 |

## 五、可靠性与高性能设计

### 5.1 三阶段可靠性

| 阶段 | 风险 | 保障手段 |
| --- | --- | --- |
| 生产阶段 | Producer 发送失败、Broker ACK 丢失 | 同步发送、失败重试、发送结果检查 |
| 存储阶段 | Broker 宕机、磁盘未刷入 | 同步刷盘 `SYNC_FLUSH`、主从同步双写 `SYNC_MASTER` |
| 消费阶段 | Consumer 处理失败、ACK 丢失 | 手动 ACK、失败重试、死信队列、消费端幂等 |

一句话：**可靠性 = 同步发送 + 同步刷盘 + 同步复制 + 手动 ACK + 幂等消费**。

### 5.2 存储模型为什么快

RocketMQ 的高性能来自：

- **CommitLog 顺序写**：所有 Topic 共用 CommitLog，顺序追加写磁盘，避免随机写。
- **ConsumeQueue 索引**：Consumer 不扫大文件，只读轻量索引定位 CommitLog。
- **PageCache**：消息写入先进入 OS PageCache，提升写入吞吐。
- **零拷贝**：通过 mmap/sendfile 等减少用户态/内核态拷贝。
- **批量拉取/批量消费**：减少网络往返和线程切换。

```
消息写入 CommitLog
       |
       v
构建 ConsumeQueue 索引（offset + size + tagCode）
       |
       v
Consumer 读 ConsumeQueue -> 定位 CommitLog -> 拉取消息
```

### 5.3 消息过滤

| 过滤方式 | 位置 | 说明 |
| --- | --- | --- |
| Tag 过滤 | Broker 端 | ConsumeQueue 保存 TagCode，适合简单分类过滤 |
| SQL92 过滤 | Broker 端 | 基于消息属性做复杂条件过滤 |
| 类过滤 | Broker 端扩展 | 自定义 Java 代码，运维复杂，谨慎使用 |

Broker 端过滤可以减少 Consumer 无效拉取，但复杂过滤会增加 Broker CPU 成本。

### 5.4 重复消费与幂等

重复消费常见原因：

- 网络抖动导致 ACK 丢失。
- Consumer 重启或 Rebalance。
- Broker 宕机切换。
- 消费成功但位点提交失败。

解决方案一定在业务层做幂等：

1. 数据库唯一约束：消息 ID 或业务流水号做唯一键。
2. Redis 去重：`SETNX msgId`，设置合理过期时间。
3. 业务状态机：订单从待支付到已支付不可逆，重复事件不重复处理。
4. Token/流水机制：每次操作有唯一业务凭证。

## 六、顺序消息机制

### 6.1 全局顺序 vs 局部顺序

| 类型 | 实现 | 代价 | 场景 |
| --- | --- | --- | --- |
| 全局顺序 | 一个 Topic 只设一个 Queue，一个 Consumer 线程 | 吞吐极低 | Binlog 同步、强顺序审计 |
| 局部顺序 | 同业务 ID 路由到同一个 Queue | 只保证同 key 有序 | 订单状态流转、支付状态流转 |

生产端用 `MessageQueueSelector` 将相同业务 ID 路由到同一 Queue：

```java
producer.send(msg, new MessageQueueSelector() {
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        return mqs.get(Math.abs(arg.hashCode()) % mqs.size());
    }
}, orderId);
```

消费端使用 `MessageListenerOrderly`：

```java
consumer.registerMessageListener(new MessageListenerOrderly() {
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext ctx) {
        // 按 Queue 内顺序消费
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

### 6.2 顺序消费的三把锁

RocketMQ 顺序消费不只是“发到同一个 Queue”，消费端还要控制并发：

| 锁 | 目的 |
| --- | --- |
| Broker 上的 MessageQueue 锁 | 确保同一个 Queue 同一时间只分配给一个 Consumer |
| 本地 MessageQueue 锁 | 确保本客户端同一个 Queue 只有一个线程处理 |
| ProcessQueue 锁 | Rebalance 时避免一边消费一边释放队列，降低重复消费风险 |

第三把锁主要处理 Rebalance：当队列从 Consumer A 转移到 Consumer B 时，A 必须确认本地没有正在处理的消息，才能安全释放 Broker 侧锁。否则 B 可能立刻接手并重复消费未提交位点的消息。

**顺序消息代价**：吞吐下降；前一条消息阻塞会拖住后续消息；因此只对确实需要顺序的业务 ID 做局部顺序，不要滥用全局顺序。

## 七、消费模式与重试

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

## 八、消息积压治理

```
积压量 = 生产速率 - 消费速率

治理策略（按严重程度递进）：
1. 扩 Consumer 实例（受限于 MessageQueue 数量，Consumer ≤ Queue）
2. 扩 MessageQueue（需提前规划，不能动态缩减）
3. 降级：跳过非关键消息（需业务支持，谨慎操作）
4. 清理：直接跳过指定 offset（数据丢失风险，仅限非关键消息）
5. 重建：新 Topic 扩容，老 Topic 消息迁移消费（有序消费场景复杂）
```

### 8.1 快速定位

```bash
# 查看 ConsumerGroup 消费进度和堆积量
./mqadmin consumerProgress -g GroupName
```

排查顺序：

1. 生产速率是否突增。
2. Consumer 实例数是否小于 Queue 数。
3. 消费线程数是否过小。
4. 单条业务逻辑是否耗时过长。
5. 下游 DB/Redis/HTTP 是否变慢。

### 8.2 紧急处理

- 扩容消费者实例或线程数。
- 调大 `pullBatchSize`、`consumeMessageBatchMaxSize` 做批量消费。
- 新建临时 Topic 分流，快速拉平后异步补处理。
- 非关键消息可重置消费位点跳过，必须经过业务确认。
- 对慢逻辑做异步化、批量写、降级或旁路。

### 8.3 预防措施

- 设置消息保留时长 `fileReservedTime`。
- 监控堆积量、消费 TPS、消费 RT、失败重试次数、DLQ 数量。
- 对热点 Topic 做限流和削峰。
- 为核心 Topic 提前规划足够 Queue 数。

## 九、集群部署与生产调优

### 9.1 集群部署模式

| 模式 | 特点 | 适用场景 |
| --- | --- | --- |
| 单 Master | 简单但有单点风险 | 本地/测试 |
| 多 Master | 吞吐高，无 Slave 容灾 | 可接受少量丢失的非核心业务 |
| 多 Master 多 Slave（异步复制） | 高可用，延迟低，主宕机可能丢少量数据 | 常规生产 |
| 多 Master 多 Slave（同步复制） | 数据可靠性更高，写延迟增加 | 金融/支付 |
| Dledger | 基于 Raft 自动主从切换 | 对自动故障转移要求高的生产集群 |

### 9.2 关键参数

Broker：

```properties
sendMessageThreadPoolNums=128
pullMessageThreadPoolNums=128
diskMaxUsedSpaceRatio=88
flushDiskType=ASYNC_FLUSH
brokerRole=ASYNC_MASTER
```

Consumer：

```properties
consumeThreadMin=20
consumeThreadMax=64
pullBatchSize=32
consumeMessageBatchMaxSize=10
```

调参原则：

- 核心交易消息优先可靠性：同步刷盘、同步复制、消费端手动 ACK。
- 日志/行为消息优先吞吐：异步刷盘、批量消费、允许短暂堆积。
- 线程数不是越大越好，下游 DB/Redis 扛不住时会把压力转移下去。

## 十、与其他 MQ 的对比

| 维度 | RocketMQ | Kafka | RabbitMQ |
|------|---------|-------|----------|
| 事务消息 | ✅ 原生半消息机制 | ❌ 需外部协调 | ✅ 基础事务 |
| 延迟消息 | ✅ 固定 Level（4.x）/ 任意（5.x）| ❌ 需自实现 | ✅ 插件支持任意 |
| 吞吐量 | 中高（CommitLog 顺序写）| 极高（分区顺序写 + 零拷贝）| 中（万级 TPS）|
| 消息积压 | 中等（ConsumeQueue 设计）| 极强（适合超大规模积压）| 弱（内存压力大）|
| 路由能力 | 弱（Tag 过滤）| 无（Partition 路由）| 强（多种 Exchange 类型）|
| 适用场景 | 金融支付、电商、事务一致性 | 日志收集、流计算、大数据管道 | 复杂路由、AMQP 兼容 |

### 10.1 Pull vs Push

| 模式 | 代表 | 优点 | 缺点 |
| --- | --- | --- | --- |
| Pull | Kafka / RocketMQ | 消费者自控节奏，可批量拉取，不易压垮消费者 | 可能拉空数据，实时性依赖长轮询 |
| Push | RabbitMQ | 有消息即推送，实时性好 | 消费端节奏不可控，容易被高峰流量压垮 |

### 10.2 RocketMQ vs Kafka

| 维度 | Kafka | RocketMQ |
| --- | --- | --- |
| 存储模型 | Topic-Partition | CommitLog + ConsumeQueue |
| 写入方式 | 每 Partition 一组文件 | 所有 Topic 共用 CommitLog |
| 刷盘 | 常见异步刷盘 | 同步/异步可选 |
| 副本 | ISR 副本机制 | 主从复制 / Dledger |
| 顺序消息 | Partition 内有序 | 单 Queue 严格有序 |
| 延时消息 | 非原生能力 | 原生支持 |
| 消息过滤 | 消费端过滤为主 | Broker 端 Tag/SQL 过滤 |
| 重试/死信 | 需自行设计更多逻辑 | 原生重试和 DLQ |
| 事务 | Kafka 事务偏流处理语义 | 半消息适合本地事务 + 消息一致 |
| 服务发现 | ZooKeeper / KRaft | NameServer |

选型口径：**Kafka 适合日志、大数据、流处理和超高吞吐；RocketMQ 适合金融、电商、事务消息、延迟消息和业务可靠性要求更强的场景**。

## 十一、关键权衡

| 权衡点 | 结论 |
|--------|------|
| RocketMQ vs Kafka | RocketMQ 功能更强（事务/延迟），Kafka 吞吐更高；支付场景选 RocketMQ，日志采集选 Kafka |
| 同步 vs 异步复制 | SYNC_MASTER 数据不丢但写入延迟高；ASYNC_MASTER 性能好但主宕机有少量数据丢失风险 |
| NameServer AP vs ZK CP | NameServer 无选举开销，简单高可用；代价是 Producer/Consumer 需定期拉取路由表（最多 30s 延迟感知 Broker 变化）|
| 延迟消息 Level 限制 | 4.x 固定 Level 是历史设计，5.x 任意精度是改进；生产建议升级 5.x |
| 顺序消息 vs 吞吐 | 顺序消费通过锁牺牲并发，只应对同一业务 ID 做局部顺序 |
| 同步刷盘/同步双写 | 数据可靠性高但延迟增加，支付核心链路可用，日志类消息不必强求 |
| Broker 端过滤 | 能减少 Consumer 无效拉取，但复杂 SQL 过滤会增加 Broker CPU 压力 |

## 十二、与其他概念的关系

- 区别于 [[机制-Kafka]]：Kafka 吞吐更高，RocketMQ 事务/延迟消息更强，支付系统选 RocketMQ，日志埋点选 Kafka
- 区别于 [[机制-RabbitMQ]]：RabbitMQ 协议更标准（AMQP），适合企业集成；RocketMQ 针对大规模互联网场景优化
- 依赖 [[概念-幂等设计]]：无论哪种 MQ，消费端都必须做幂等（消息可能重投，重试机制本身就是幂等性的来源）
- 与 [[机制-Seata]] 协作：Seata 的 Saga 模式可以用 RocketMQ 事务消息替代本地消息表实现可靠消息

## 十三、应用边界

**用 RocketMQ**：
- 支付/金融场景（转账成功通知、对账消息）— 事务消息保证本地事务与消息原子
- 订单超时关闭（延迟消息，30min/1h 后触发关闭逻辑）
- 需要精确控制消费速率、有重试和死信需求的业务流

**不用 RocketMQ（选 Kafka）**：
- 日志收集（量极大，不需要事务消息）
- 流计算（Flink/Spark 的 Kafka 连接器生态更成熟）
- 需要超长消息保留（Kafka 的时间索引保留效率更高）

## 十四、面试速答

| 问题 | 速答 |
| --- | --- |
| 如何保证消息不丢失 | 同步发送 + 同步刷盘 + 主从同步双写 + 消费端手动 ACK |
| 如何保证消息不重复 | RocketMQ 只能尽量减少重复，业务层必须用 Redis/DB 去重和状态机幂等 |
| 如何保证消息顺序 | 相同业务 ID 通过 MessageQueueSelector 路由到同一 Queue，消费端使用 MessageListenerOrderly |
| 百万级消息堆积怎么处理 | 扩容 Consumer、批量消费、优化慢逻辑、临时 Topic 分流、必要时跳过非关键消息 |
| RocketMQ 为什么快 | CommitLog 顺序写 + PageCache + ConsumeQueue 索引 + 零拷贝 + 批量拉取 |
| RocketMQ 为什么适合事务 | 半消息 + 本地事务 + Commit/Rollback + Broker 回查，可推进本地事务和消息最终一致 |
| RocketMQ vs Kafka 怎么选 | 业务事务/延迟/可靠性选 RocketMQ；日志/流计算/超高吞吐选 Kafka |
| 顺序消息有什么坑 | 加锁降低吞吐，前序消息阻塞会拖住后续消息，只能谨慎做局部顺序 |

核心记忆口诀：

- 可靠性 = 同步发送 + 同步刷盘 + 同步复制
- 幂等性 = Redis/DB 去重 + 状态机
- 顺序性 = 同业务 ID -> 同 Queue -> 顺序监听器
- 高性能 = 顺序写 + PageCache + 零拷贝 + 批量
