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
