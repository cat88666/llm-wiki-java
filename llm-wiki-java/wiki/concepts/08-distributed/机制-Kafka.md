---
type: concept
status: active
name: "Kafka"
layer: L7
aliases: ["Kafka", "消息流平台", "Kafka重平衡", "ISR", "高水位", "Leader Epoch", "渐进式重平衡", "Kafka顺序消费", "Kafka存储结构", "KRaft"]
tags: ["#distributed"]
related:
  - "[[机制-消息队列可靠性]]"
  - "[[机制-ZAB协议与Zookeeper]]"
  - "[[概念-分布式系统理论]]"
sources:
  - "../../../raw/note/tuling/08-mq/Kafka.md"
  - "../../../raw/note/tuling/08-mq/Kafka/✅Kafka 消息的发送过程简单介绍一下？.md"
  - "../../../raw/note/tuling/08-mq/Kafka/✅Kafka 中的Offset是什么？.md"
  - "../../../raw/note/tuling/08-mq/Kafka/✅Kafka的消费者数量和分区数量可以不同吗？会发生什么？.md"
  - "../../../raw/note/tuling/08-mq/Kafka/✅Kafka为什么依赖Zookeeper，有什么用？.md"
  - "../../../raw/note/tuling/08-mq/Kafka/✅Kafka支持事务消息吗？如何实现的？.md"
  - "../../../raw/note/tuling/08-mq/Kafka/✅MQ的重平衡会带来哪些问题？.md"
created: 2026-05-07
updated: 2026-05-07
lint_notes: ""
---

# Kafka

> 分布式消息流平台：以**顺序磁盘写 + 零拷贝 + 批处理**为核心，实现百万级 TPS 的持久化消息传输，天然契合日志采集、实时流处理等大吞吐场景。

## 第一性原理

Kafka 快的本质：**磁盘顺序写速度接近内存（约 600 MB/s），远超随机写（约 100 MB/s）**。只要保证消息追加写入（Append-Only），就能以极低成本实现极高吞吐。所有其他优化（批处理减少网络往返、零拷贝减少内核切换、稀疏索引减少内存占用）都是围绕这一点展开。

---

## 核心机制

### 架构全景

```
生产者 Producer
    ↓ 发送 (acks=0/1/-1)
Kafka Broker 集群
  ├── Topic: 逻辑消息分类
  │     └── Partition 1..n: 有序、持久化日志 (Leader + Follower ISR副本)
  │           └── Segment: .log 数据文件 + .index 偏移索引 + .timeindex 时间索引
  └── ZooKeeper（<4.0）/ KRaft（4.0 自治）: 元数据管理、Controller选举
Consumer Group
  └── Consumer 1..n → 每个 Partition 只被组内一个 Consumer 消费
```

- **Partition** = 有序不变消息序列，同一 Partition 内消息按追加顺序有 offset
- **Consumer Group** = 多个消费者共享负载；不同 Group 可独立消费同一 Topic（发布-订阅）
- **Offset** = 分区内每条消息的单调递增位置标识；消费者通过提交 Offset 记录进度

### 为什么 Kafka 这么快

| 优化点 | 原理 |
|--------|------|
| **顺序磁盘写** | Append-Only 日志，磁盘顺序写速度接近内存 |
| **零拷贝** | `sendfile` 系统调用：磁盘→内核缓冲区→Socket，跳过用户空间两次拷贝 |
| **页缓存（Page Cache）**| 读请求先走 OS Page Cache，避免磁盘 IO |
| **批量发送** | 生产者缓存消息，达到阈值（大小/时间）后批量发送，减少网络往返 |
| **稀疏索引** | `.index` 文件每隔 N 条消息建一个索引点，索引小、加载快 |
| **分区并行** | 多 Partition 多 Consumer 并行处理，突破单机 IO 瓶颈 |

### 存储结构

```
Topic: order-events
└── Partition 0 (磁盘目录)
    ├── 00000000000000000000.log   ← Segment：从 offset 0 开始的消息数据
    ├── 00000000000000000000.index ← 稀疏 offset 索引
    ├── 00000000000000000000.timeindex ← 时间戳索引
    └── 00000000000001000000.log   ← 下一个 Segment（从 offset 1000000 开始）
```

**消息读取流程**：Consumer 指定 offset → 二分查找找到对应 Segment（按文件名定位）→ 用 `.index` 定位物理位置 → 顺序扫描 `.log` 找到精确消息。

### 消息可靠性：acks + ISR + HW

**生产者确认机制（acks）**：
| acks | 含义 | 可靠性 | 吞吐 |
|------|------|--------|------|
| `0` | 不等待确认，发完即返回 | 最低，可能丢 | 最高 |
| `1` | Leader 写入磁盘后确认 | 中，Follower 未同步时 Leader 挂则丢 | 中 |
| `-1` | 等 ISR 全部副本写入后确认 | 最高 | 最低 |

**ISR（In-Sync Replicas）**：与 Leader 保持同步的 Follower 集合。Follower 落后超过 `replica.lag.max.ms`（默认 10s）则被移出 ISR。`acks=-1` 需等 ISR 全部确认。

**高水位（HW）与 LEO**：
- **LEO（Log End Offset）**：分区日志末尾下一条待写 offset
- **HW（High Watermark）**：ISR 中所有副本都已确认的最高 offset；消费者只能读取 HW 以内的消息（已提交消息）
- **Leader Epoch**：每次 Leader 切换递增，用于防止副本切换后数据回滚（新 Leader 只接受旧 Epoch ≤ 自身且 HW 一致的数据）

**为什么不能 100% 不丢消息**：
1. 生产者异步发送 + 回调中未处理失败 → 消息静默丢失
2. Broker PageCache 未 fsync 时宕机 → 消息丢失
3. ISR 同步延迟期间 Leader 挂 + acks=1 → Follower 升 Leader 后数据不一致

配合本地消息表或分布式事务可接近 100% 可靠。

### 重平衡（Rebalance）

**触发条件**（3 类 + 2 异常）：
- 消费者组成员变化（加入/退出）
- 订阅的 Topic 数量变化
- Topic 的 Partition 数量变化
- 异常：Group Coordinator 故障 / Leader Consumer 崩溃

**5 种消费者组状态**：Empty → PreparingRebalance → CompletingRebalance → Stable → Dead

**传统重平衡 vs 渐进式重平衡**：

传统方式：所有消费者停止消费（STW）→ 重新分配所有 Partition → 恢复消费。

**CooperativeStickyAssignor（渐进式）**（Kafka 2.4+）：
1. 第一阶段：计算最小变更方案，只让部分消费者释放少数 Partition
2. 第二阶段：将释放的 Partition 分配给新消费者
- 未变动的 Partition 继续消费，**不触发全局 STW**

**Kafka 4.0 改进**：分区分配逻辑从客户端移到服务端（全局视角），单消费者变化不再暂停整个组。

**减少重平衡的手段**：
- 开启**静态成员**（`group.instance.id`）：消费者重启后保留分区分配
- 使用 StickyAssignor 或 CooperativeStickyAssignor 分配策略
- 合理配置心跳超时，避免误判消费者挂掉

### 顺序消费

Kafka 保证**同一 Partition 内消息有序**（追加写 + 偏移量）；跨 Partition 无序。

实现顺序消费的三种方式：
1. **单 Partition**：一个 Topic 只建一个 Partition（吞吐换顺序）
2. **指定 Partition**：`ProducerRecord(topic, partition, key, value)` 直接指定
3. **相同 Key → 相同 Partition**：Kafka 对 key 做 hash 路由，相同 key 保证同一分区

### 消费语义

| 语义 | 机制 | 场景 |
|------|------|------|
| **At-most-once** | 先提交 offset，再处理 | 允许丢失 |
| **At-least-once**（默认）| 先处理，成功后手动提交 offset | 可能重复，需消费端幂等 |
| **Exactly-once** | Kafka 事务（生产+消费 offset 提交原子化）| 金融/精确计算场景 |

---

## 与其他 MQ 的对比

| 维度 | Kafka | RocketMQ | RabbitMQ |
|------|-------|----------|----------|
| **吞吐量** | 极高（百万/s）| 高（十万/s）| 中（万/s）|
| **延迟消息** | 不支持（间接实现）| ✅ 固定等级 | ✅ 插件支持任意 |
| **死信队列** | 无 | ✅ | ✅ |
| **消费模式** | 主要 Pull | Push/Pull | 主要 Push |
| **事务消息** | ✅（限生产端）| ✅ | ✅ 基础事务 |
| **优先级队列** | ❌ | ✅ | ✅ |
| **适合场景** | 日志/大数据管道 | 业务消息/延迟任务 | 复杂路由/AMQP |

---

## 关键权衡

1. **高吞吐 vs 功能丰富**：Kafka 砍掉了延迟队列、死信队列等功能，换取极致吞吐；RocketMQ/RabbitMQ 功能更全但吞吐较低
2. **多分区并发 vs 消息全局顺序**：增加 Partition 提升吞吐，但多 Partition 无全局顺序保证
3. **acks=-1 vs 性能**：最高可靠性需等全 ISR 确认，吞吐降低；通常生产用 acks=1 + 重试是合理折衷
4. **消费者数量 vs Partition 数量**：消费者 ≤ Partition 数才能完全并行；消费者 > Partition 数则多余消费者空闲

## 与其他概念的关系

- **依赖 [[机制-ZAB协议与Zookeeper]]**：Kafka 2.8 之前依赖 ZooKeeper 做 Controller 选举和元数据存储；4.0 通过 KRaft 协议自治，完全移除 ZK 依赖
- **相关 [[机制-消息队列可靠性]]**：Kafka 是消息队列的一种实现，消息可靠性三段（Producer/Broker/Consumer）机制与其他 MQ 通用
- **依赖 [[概念-幂等设计]]**：At-least-once 语义下消费者必须实现幂等处理，防止重复消费带来副作用

## 应用边界

**适合 Kafka**：日志采集（ELK Pipeline）、实时流处理（Flink 数据源）、大数据管道（Hadoop 写入缓冲）、业务事件总线（高吞吐场景）。

**不适合 Kafka**：需要精确延迟消息（订单超时关闭，用 Redisson）；需要死信队列自动重试；需要 AMQP 协议的复杂路由（用 RabbitMQ）；业务量小的简单消息通知（用 RocketMQ 更简单）。

---

## 大流量实战补充

### 生产者关键调优参数

| 参数 | 默认值 | 调优建议 | 说明 |
|------|--------|---------|------|
| `linger.ms` | 0 | 5~50ms | 等待更多消息凑批，为 0 则立即发送（吞吐低）|
| `batch.size` | 16KB | 64KB~512KB | 批次大小上限，配合 linger.ms 提升批量效果 |
| `compression.type` | none | `lz4` 或 `zstd` | 压缩减少网络和磁盘 IO，lz4 速度最快 |
| `buffer.memory` | 32MB | 64MB~256MB | 发送缓冲区，写入峰值时防阻塞 |
| `max.in.flight.requests.per.connection` | 5 | 1（需顺序时）| 设为 1 保证单分区有序，但吞吐下降 |
| `acks` | 1 | -1（金融）/ 1（通用）| 与可靠性权衡 |

**生产者吞吐公式**：`TPS ≈ batch.size × (1000 / linger.ms) × 分区数`

### 消费者关键调优参数

| 参数 | 默认值 | 调优建议 | 说明 |
|------|--------|---------|------|
| `fetch.min.bytes` | 1 | 1KB~64KB | 最小拉取字节，减少空轮询 |
| `fetch.max.wait.ms` | 500ms | 100~500ms | 等待数据达到 fetch.min.bytes 的超时 |
| `max.poll.records` | 500 | 100~2000 | 单次拉取最大条数，根据处理速度调整 |
| `max.poll.interval.ms` | 5min | 按业务处理时间上限 + buffer 设置 | 超时触发 Rebalance（最常见的假性重平衡原因）|
| `session.timeout.ms` | 45s | 30s~90s | 心跳超时，过小误判为宕机触发 Rebalance |
| `enable.auto.commit` | true | **false**（推荐）| 手动提交，避免处理中宕机导致丢消息 |

### 消息积压处理

彩票/交易系统开奖结算瞬间产生大量消息，消费速率跟不上时：

```
积压判断：consumer_lag = LEO - committed_offset > 告警阈值

处理策略（按严重程度）：
  1. 扩 Consumer 实例（前提：Consumer数 ≤ Partition数）
  2. 扩 Partition 数（注意：已有消息按旧分区路由，扩后路由变化）
  3. 单次拉取后并行处理（线程池）+ 批量提交 offset
  4. 降级：临时跳过非核心处理逻辑，只保障核心结算

// 并行消费示意（保证同用户串行）
consumer.poll(Duration.ofMillis(100)).forEach(record -> {
    String userId = record.key();
    // 按 userId 分桶，同一 userId 进同一线程
    int bucket = Math.abs(userId.hashCode()) % POOL_SIZE;
    executorPools[bucket].submit(() -> process(record));
});
// 注意：并行消费后 offset 提交要等所有任务完成，防止丢消息
```

### 开奖广播（百万用户实时推送）

开奖结果需要通知所有在线用户（1 Topic → N Consumer Group）：

```
lottery.draw.result topic（1条开奖消息）
    ├── settlement-group    → 结算 Consumer 触发批量结算
    ├── push-group          → 推送 Consumer 触发 WebSocket/App 推送
    ├── risk-group          → 风控 Consumer 分析异常投注
    └── log-group           → 日志 Consumer 写 ES 存档

每个 Consumer Group 独立消费，互不影响
→ 一条消息广播给 N 个系统，天然 Fanout 模式
```

### 重平衡期间不丢消息

```java
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Rebalance 前：提交当前已处理完的 offset（关键！）
        consumer.commitSync(currentOffsets);
    }
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Rebalance 后：从上次提交的 offset 继续消费（默认行为）
    }
});
```
