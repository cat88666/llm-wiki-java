---
type: concept
status: active
name: "Kafka"
layer: L7
aliases: ["Kafka", "消息流平台", "Kafka重平衡", "ISR", "高水位", "Leader Epoch", "渐进式重平衡", "Kafka顺序消费", "Kafka存储结构", "KRaft", "acks", "HW", "LEO", "消费者组状态", "批量消费", "Controller选举", "Partition Leader选举", "Topic vs Partition"]
tags: ["#distributed"]
related:
  - "[[机制-RabbitMQ]]"
  - "[[机制-Zookeeper]]"
  - "[[概念-分布式理论]]"
  - "[[概念-幂等设计]]"
---

# Kafka

> 分布式消息流平台：以**顺序磁盘写 + 零拷贝 + 批处理**为核心，实现百万级 TPS 的持久化消息传输，天然契合日志采集、实时流处理等大吞吐场景；以牺牲功能丰富性换取极致吞吐。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 顺序磁盘写为什么这么快 |
| [二、核心架构](#二核心架构) | Topic/Partition/ConsumerGroup/Offset |
| [三、为什么 Kafka 快](#三为什么-kafka-快) | 六大优化维度 |
| [四、消息可靠性：acks + ISR + HW](#四消息可靠性acks--isr--hw) | 生产者确认、ISR 同步（版本演进）、高水位、消息丢失排查 |
| [五、重平衡（Rebalance）](#五重平衡rebalance) | 触发条件、消费者组5种状态、渐进式重平衡、减少重平衡手段 |
| [六、选举机制](#六选举机制) | Partition Leader 选举、Controller 选举 |
| [七、顺序消费与消费语义](#七顺序消费与消费语义) | 同 Partition 有序、三种消费语义、批量消费实践 |
| [八、与其他 MQ 的对比](#八与其他-mq-的对比) | Kafka vs RocketMQ vs RabbitMQ |
| [九、生产者与消费者调优参数](#九生产者与消费者调优参数) | 关键参数与调优建议 |
| [十、大流量实战](#十大流量实战) | 消息积压治理、广播模式 |
| [十一、关键权衡与应用边界](#十一关键权衡与应用边界) | 高吞吐 vs 功能，适合 vs 不适合场景 |

## 一、第一性原理

Kafka 快的本质：**磁盘顺序写速度接近内存（约 600 MB/s），远超随机写（约 100 MB/s）**。只要保证消息追加写入（Append-Only），就能以极低成本实现极高吞吐。所有其他优化（批处理减少网络往返、零拷贝减少内核切换、稀疏索引减少内存占用）都是围绕这一点展开。

Kafka 的设计哲学：**用简单和克制换取极致性能**。砍掉死信队列、延迟消息、复杂路由，只做分布式日志（Append-Only 分区）这一件事，做到极致。

## 二、核心架构

```
生产者 Producer
    ↓ 发送 (acks=0/1/-1)
Kafka Broker 集群
  ├── Topic: 逻辑消息分类
  │     └── Partition 1..n: 有序、持久化日志 (Leader + Follower ISR 副本)
  │           └── Segment: .log 数据文件 + .index 偏移索引 + .timeindex 时间索引
  └── ZooKeeper（<4.0）/ KRaft（4.0 自治）: 元数据管理、Controller 选举
Consumer Group
  └── Consumer 1..n → 每个 Partition 只被组内一个 Consumer 消费
```

**核心概念**：
- **Topic** = 逻辑消息分类容器，区分业务（类似消息队列名）
- **Partition** = Topic 的物理分区，有序不变消息序列；同一 Partition 内消息按追加顺序有 offset
- **Consumer Group** = 多个消费者共享负载；不同 Group 可独立消费同一 Topic（发布-订阅）
- **Offset** = 分区内每条消息的单调递增位置标识；消费者通过提交 Offset 记录消费进度

**为什么有了 Topic 还要有 Partition（加一层分区）**：Topic 是逻辑概念，Partition 是物理实现。多 Partition 带来三个收益：① **提升吞吐**：多 Partition 可由不同消费者并行消费；② **负载均衡**：Partition 数 > Consumer 数时自动均匀分配；③ **水平扩展**：增加 Partition 数即可扩容并发处理能力，无需改动业务代码。

**存储结构**：

```
Topic: order-events
└── Partition 0 (磁盘目录)
    ├── 00000000000000000000.log      ← Segment：从 offset 0 开始的消息数据
    ├── 00000000000000000000.index    ← 稀疏 offset 索引
    ├── 00000000000000000000.timeindex ← 时间戳索引
    └── 00000000000001000000.log      ← 下一个 Segment（从 offset 1000000 开始）
```

消息读取：Consumer 指定 offset → 二分查找找到 Segment → 用 `.index` 定位物理位置 → 顺序扫描 `.log` 找精确消息。

## 三、为什么 Kafka 快

| 优化点 | 原理 |
|--------|------|
| **顺序磁盘写** | Append-Only 日志，磁盘顺序写速度（600 MB/s）≈ 内存写速度 |
| **零拷贝** | `sendfile` 系统调用：磁盘→内核缓冲区→Socket，跳过用户空间两次拷贝 |
| **页缓存（Page Cache）**| 读请求先走 OS Page Cache，避免磁盘 IO |
| **批量发送** | 生产者缓存消息，达到阈值（大小/时间）后批量发送，减少网络往返 |
| **稀疏索引** | `.index` 文件每隔 N 条消息建一个索引点，索引小、加载快 |
| **分区并行** | 多 Partition 多 Consumer 并行处理，突破单机 IO 瓶颈 |

## 四、消息可靠性：acks + ISR + HW

### 生产者确认机制（acks）

| acks | 含义 | 可靠性 | 吞吐 |
|------|------|--------|------|
| `0` | 不等待确认，发完即返回 | 最低，可能丢 | 最高 |
| `1` | Leader 写入磁盘后确认 | 中，Follower 未同步时 Leader 挂则丢 | 中 |
| `-1` | 等 ISR 全部副本写入后确认 | 最高 | 最低 |

### ISR（In-Sync Replicas）

与 Leader 保持同步的 Follower 集合。**ISR 判断标准版本演进**：

| 版本 | 参数 | 判断方式 | 缺陷 |
|------|------|---------|------|
| < 0.9.x | `replica.lag.max.messages` | Follower 落后消息数 > 阈值则移出 | 瞬间高并发时所有 Follower 会被批量踢出（抖动触发大量 ISR 变更）|
| ≥ 0.9.x | `replica.lag.max.ms`（默认 10s）| Follower LEO 落后 Leader 超过时间阈值才移出 | 瞬间流量只要在时间窗口内追上就不会被踢出，更稳定 |

`acks=-1` 需等 ISR 全部确认。若 ISR 只剩 Leader 一个（其他 Follower 落后），`acks=-1` 退化为 `acks=1`。

### 高水位（HW）与 LEO

- **LEO（Log End Offset）**：分区日志末尾下一条待写 offset
- **HW（High Watermark）**：ISR 中所有副本都已确认的最高 offset；消费者只能读取 HW 以内的消息（已提交消息）
- **Leader Epoch**：每次 Leader 切换递增，用于防止副本切换后数据回滚

**为什么不能 100% 不丢消息（3端分析）**：

| 端 | 常见丢失原因 | 最佳实践 |
|----|------------|---------|
| **生产者** | `producer.send(msg)` 异步不等待，未注册 callback 处理失败；或 Producer 进程在消息未发出时崩溃 | 使用 `send(msg, callback)` 处理失败，必要时结合本地消息表 |
| **Broker** | PageCache 未 fsync 时宕机；acks=1 时 Leader 同步给 Follower 前挂掉；消息超 72h 过期未被消费 | acks=-1 + min.insync.replicas ≥ 2；引入分布式事务/本地消息表做终极保障 |
| **消费者** | 自动提交 offset（消息处理中宕机，下次从已提交 offset 开始丢了这批）；finally 中提交 offset（无论成功失败都提交）| `enable.auto.commit=false`；只在所有消息**成功处理后**手动提交 |

只靠 Kafka 自身无法 100% 保证不丢；配合**本地消息表**或**分布式事务**才能实现极端可靠性。

## 五、重平衡（Rebalance）

### 触发条件

- 消费者组成员变化（加入/退出/心跳超时）
- 订阅的 Topic 数量变化
- Topic 的 Partition 数量变化
- Group Coordinator 故障 / Leader Consumer 崩溃

### 传统 vs 渐进式重平衡

**传统方式（STW 式）**：所有消费者停止消费 → 重新分配所有 Partition → 恢复消费。期间积压上升，延迟飙高。

**CooperativeStickyAssignor（渐进式，Kafka 2.4+）**：
1. 第一阶段：计算最小变更方案，只让部分消费者释放少数 Partition
2. 第二阶段：将释放的 Partition 分配给新消费者
3. 未变动的 Partition 继续消费，**不触发全局 STW**

**Kafka 4.0 改进**：分区分配逻辑从客户端移到服务端（全局视角），单消费者变化不再暂停整个组。

### 消费者组状态机（5种状态）

| 状态 | 描述 |
|------|------|
| **Empty** | 组内无成员，但可能有未过期的已提交 offset |
| **Dead** | 组内无成员，且元数据已被 Group Coordinator 移除 |
| **PreparingRebalance** | 准备重平衡，所有成员需重新申请加入 |
| **CompletingRebalance** | 所有成员已加入，等待分区分配方案下发 |
| **Stable** | 稳定态，重平衡完成，成员可正常消费 |

状态流转：`Empty → PreparingRebalance → CompletingRebalance → Stable`；成员变动时 `Stable → PreparingRebalance`；所有成员退出时 `→ Empty/Dead`。

### 减少重平衡的手段

- 开启**静态成员**（`group.instance.id`）：消费者重启后保留分区分配，不触发 Rebalance
- 使用 CooperativeStickyAssignor 分配策略（非 STW）
- 合理配置 `max.poll.interval.ms`（最常见重平衡原因：消费者处理慢超时被踢出）
- 合理配置 `session.timeout.ms`，避免因网络抖动误判为宕机

### 重平衡期间不丢消息

```java
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Rebalance 前：提交当前已处理完的 offset（关键！防止重复消费后 offset 丢失）
        consumer.commitSync(currentOffsets);
    }
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Rebalance 后：从上次提交的 offset 继续消费（默认行为）
    }
});
```

## 六、选举机制

### Partition Leader 选举

每个 Partition 有一个 Leader 处理读写请求。Leader 故障时，从 ISR 中选出新 Leader（保证数据完整性——ISR 中的副本都有最新已提交数据）。

**选举过程（旧版，依赖 ZooKeeper）**：
1. 候选副本向 ZooKeeper 写入带递增序列号的临时节点（election contender）
2. 所有副本比较序列号，**序列号最小者成为新 Leader**（ZK 序列号唯一递增，保证只有一个赢家）
3. 新 Leader 信息写入 `/broker/.../leader` 节点，其他副本监听并更新元数据

**KRaft 模式（Kafka 4.0+）**：移除 ZK 依赖，由 Controller Quorum 通过 Raft 协议直接管理 Partition 元数据，选举逻辑内置于 Broker 层。

### Controller 选举

Kafka 集群只有一个 **Controller** Broker，负责管理分区副本分配、Leader 选举调度等任务。

**选举过程（ZooKeeper 模式）**：
1. 所有 Broker 启动时向 ZooKeeper 注册自身，并监听 `/controller` 节点
2. Controller 故障 → ZK 删除 `/controller` 临时节点 → 所有 Broker 感知到事件
3. Broker 们竞争向 ZK 写入 `/controller` 节点（只有一个能成功）→ 成功者成为新 Controller
4. 其他 Broker 监听到 `/controller` 变化 → 更新元数据，与新 Controller 建立连接

**KRaft 模式（Kafka 4.0+）**：Controller 由 Controller Quorum 选举产生（Raft Leader），不再依赖 ZK。

## 七、顺序消费与消费语义

### 顺序消费

Kafka 保证**同一 Partition 内消息有序**（追加写 + 偏移量）；跨 Partition 无序。

| 实现方式 | 方法 | 代价 |
|---------|------|------|
| 单 Partition | 一个 Topic 只建一个 Partition | 吞吐量极低 |
| 指定 Partition | `ProducerRecord(topic, partition, key, value)` | 需要业务了解分区 |
| 相同 Key → 相同 Partition | Kafka 对 key 做 hash 路由 | Key 分布要均匀，防数据倾斜 |

### 消费语义

| 语义 | 机制 | 场景 |
|------|------|------|
| **At-most-once** | 先提交 offset，再处理 | 允许丢失 |
| **At-least-once**（默认）| 先处理，成功后手动提交 offset | 可能重复，需消费端幂等 |
| **Exactly-once** | Kafka 事务（生产+消费 offset 提交原子化）| 金融/精确计算场景 |

**手动提交 offset 最佳实践**：`enable.auto.commit=false`，业务处理完成后同步提交，防止消费中宕机导致 offset 丢失。

## 八、与其他 MQ 的对比

| 维度 | Kafka | RocketMQ | RabbitMQ |
|------|-------|----------|----------|
| **吞吐量** | 极高（百万/s）| 高（十万/s）| 中（万/s）|
| **延迟消息** | 不支持（间接实现）| ✅ 固定等级（4.x）/ 任意（5.x）| ✅ 插件支持任意 |
| **死信队列** | 无 | ✅ | ✅ |
| **消费模式** | 主要 Pull（长轮询）| Push/Pull | 主要 Push |
| **事务消息** | ✅（限生产端原子性）| ✅ 半消息机制 | ✅ 基础事务 |
| **优先级队列** | ❌ | ✅ | ✅ |
| **适合场景** | 日志/大数据管道 | 业务消息/延迟任务 | 复杂路由/AMQP |

## 九、生产者与消费者调优参数

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
| `max.poll.interval.ms` | 5min | 按业务处理时间上限 + buffer | 超时触发 Rebalance（最常见的假性重平衡原因）|
| `session.timeout.ms` | 45s | 30s~90s | 心跳超时，过小误判为宕机触发 Rebalance |
| `enable.auto.commit` | true | **false**（推荐）| 手动提交，避免处理中宕机导致丢消息 |

## 十、大流量实战

### 消息积压治理

```
积压判断：consumer_lag = LEO - committed_offset > 告警阈值

治理策略（按严重程度递进）：
  1. 扩 Consumer 实例（前提：Consumer 数 ≤ Partition 数）
  2. 扩 Partition 数（注意：已有消息按旧分区路由，新 Partition 无历史数据）
  3. 单次拉取后并行处理（线程池）+ 批量提交 offset
  4. 降级：临时跳过非核心处理逻辑，只保障核心结算
```

**并行消费示意（保证同用户串行）**：

```java
consumer.poll(Duration.ofMillis(100)).forEach(record -> {
    String userId = record.key();
    int bucket = Math.abs(userId.hashCode()) % POOL_SIZE;
    // 同 userId 进同一线程，保证该用户消息有序处理
    executorPools[bucket].submit(() -> process(record));
});
// 注意：并行消费后 offset 提交要等所有任务完成，防止丢消息
```

### 批量消费最佳实践

Spring Kafka 批量消费通过 `@KafkaListener` + 工厂配置实现：

```java
// 1. 工厂配置
factory.setBatchListener(true);
factory.getContainerProperties().setAckMode(AckMode.MANUAL);  // 必须手动提交

// 2. 批量监听
@KafkaListener(topics = "order", containerFactory = "batchFactory")
public void listen(List<ConsumerRecord<?, ?>> records, Acknowledgment ack) {
    // 3. 并发处理（同 key 串行）
    CompletionService<Void> cs = new ExecutorCompletionService<>(pool);
    for (ConsumerRecord<?, ?> r : records) cs.submit(() -> { process(r); return null; });
    for (int i = 0; i < records.size(); i++) cs.take();  // 等待全部完成
    // 4. 所有消息成功后才提交
    ack.acknowledge();
}
```

**常见陷阱**：
- `finally { ack.acknowledge(); }` 错误：无论成功失败都提交，会丢失失败的消息
- 自动提交模式下批量消费：消息处理中宕机，已自动提交的 offset 导致这批消息丢失
- 正确做法：只有**所有消息都成功**后才提交；一条失败则整批不提交，下次重新投递（需消费端幂等）

### 开奖广播（百万用户实时推送）

```
lottery.draw.result topic（1条开奖消息）
    ├── settlement-group    → 结算 Consumer 触发批量结算
    ├── push-group          → 推送 Consumer 触发 WebSocket/App 推送
    ├── risk-group          → 风控 Consumer 分析异常投注
    └── log-group           → 日志 Consumer 写 ES 存档

每个 Consumer Group 独立消费，互不影响
→ 一条消息广播给 N 个系统，天然 Fanout 模式
```

## 十一、关键权衡与应用边界

### 关键权衡

| 权衡点 | 结论 |
|--------|------|
| 高吞吐 vs 功能丰富 | Kafka 砍掉延迟队列、死信队列等功能换取极致吞吐 |
| 多分区并发 vs 消息全局顺序 | 增加 Partition 提升吞吐，但多 Partition 无全局顺序保证 |
| acks=-1 vs 性能 | 最高可靠性需等全 ISR 确认，吞吐降低；通常 acks=1 + 重试是合理折衷 |
| 消费者数量 vs Partition 数量 | 消费者 ≤ Partition 数才能完全并行；消费者 > Partition 数则多余消费者空闲 |

### 与其他概念的关系

- **依赖 [[机制-Zookeeper]]**：Kafka 2.8 之前依赖 ZooKeeper 做 Controller 选举和元数据存储；4.0 通过 KRaft 协议自治，完全移除 ZK 依赖
- **相关 [[机制-RabbitMQ]]**：Kafka 是消息队列的一种实现，高频面试题是与 RabbitMQ/RocketMQ 对比
- **依赖 [[概念-幂等设计]]**：At-least-once 语义下消费者必须实现幂等处理，防止重复消费带来副作用

### 应用边界

**适合 Kafka**：日志采集（ELK Pipeline）、实时流处理（Flink 数据源）、大数据管道（Hadoop 写入缓冲）、业务事件总线（高吞吐场景）。

**不适合 Kafka**：需要精确延迟消息（订单超时关闭，用 Redisson 或 RocketMQ）；需要死信队列自动重试；需要 AMQP 协议的复杂路由（用 RabbitMQ）；业务量小的简单消息通知（用 RocketMQ 更简单）。
