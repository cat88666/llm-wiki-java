---
type: concept
status: active
name: "RocksDB"
layer: L5
aliases: ["LSM树", "LSM-Tree", "RocksDB", "LevelDB", "SSTable", "Memtable", "Compaction", "写放大", "读放大", "空间放大", "WAL预写日志"]
tags: ["#storage"]
related:
  - "[[机制-B+树]]"
  - "[[机制-MySQL三种日志]]"
  - "[[机制-Redis持久化]]"
  - "[[概念-分布式系统理论]]"
sources: []
created: 2026-05-07
updated: 2026-05-07
lint_notes: ""
---

# RocksDB

> LSM-Tree（Log-Structured Merge-Tree）是专为**写密集型**场景设计的存储结构：所有写操作先顺序追加到内存再批量落盘，以**读放大**换取**极致写吞吐**；RocksDB 是 Facebook 在 LevelDB 基础上实现的生产级 LSM 存储引擎，被 TiKV、Flink State Backend、Kafka 等广泛采用。

## 第一性原理

B+ 树的写性能瓶颈：每次写入都需**随机定位**磁盘上的 B+ 树节点并原地更新，加上预写日志，一次写入实际发生 2-3 次随机 IO。磁盘随机 IO 是绝对瓶颈（HDD：~100 IOPS；SSD：数万 IOPS 但仍有延迟上限）。

**LSM-Tree 颠覆**：把随机写变成顺序写。所有写操作只做两件事：① 顺序追加 WAL；② 更新内存中的 Memtable。顺序写比随机写快 10x～100x。代价是读需要查多层结构（读放大）。

---

## 核心机制

### 三层结构

```
Write ──→ WAL（预写日志，顺序追加，崩溃恢复）
       ──→ Memtable（内存有序结构，默认 64MB，SkipList 实现）
              │ 写满 → Immutable Memtable → 后台 flush 到磁盘
              ↓
          L0  ── SSTable 文件（可能 key 范围重叠，读放大高）
              │ 文件数达阈值 → Compaction
              ↓
          L1  ── 有序、无重叠 SSTable（10MB 上限）
          L2  ── 100MB
          L3  ── 1GB
          Ln  ── 每层容量 × 10
```

**SSTable（Sorted String Table）**：不可变的有序磁盘文件，内含：
- Data Block：实际 KV 数据
- Index Block：每个 Data Block 的首 key → 快速定位
- Bloom Filter：概率判断 key 是否存在，快速排除不含该 key 的文件（节省 IO）
- Footer：指向 Index/Bloom Filter 的偏移量

### 写路径（极速）

```
写请求 → WAL 顺序追加（μs 级）→ Memtable 插入（内存 SkipList）→ 返回
  ↓ Memtable 写满（64MB）
Memtable → Immutable → 后台 flush → L0 新 SSTable
  ↓ L0 文件数 ≥ 4
触发 Compaction（L0 → L1 合并排序，消除重复 key，保留最新版本）
```

### 读路径（需查多层）

```
读请求（点查）：
  ① Memtable（内存 SkipList，O(log n)）
  ② Immutable Memtable
  ③ L0 所有 SSTable（需全查，L0 key 可能重叠）
  ④ L1 → L2 → … → Ln
     每层：Bloom Filter 快速排除 → 二分找 SSTable → Index Block 定位 → Data Block 读取
```

最坏情况：需查 L0 所有文件 + 其他每层 1 个文件 ≈ 10-20 次磁盘 IO（读放大）。

**Bloom Filter 是读性能的关键**：以约 10 bits/key 的空间代价，将"不存在的 key"的磁盘 IO 降为 0。

### Compaction 策略对比

| 策略 | 写放大 | 读放大 | 空间放大 | 适用场景 |
|------|--------|--------|----------|---------|
| **Leveled**（默认）| 高（10-30x）| 低 | 低 | 读多写少，生产首选 |
| **Size-Tiered** | 低（3-10x）| 高 | 高 | 写密集，读性能要求低 |
| **FIFO** | 极低 | 极高 | 高 | 时序数据，只追加不更新 |

### 三大放大因子

| | 写放大 (WA) | 读放大 (RA) | 空间放大 (SA) |
|--|--|--|--|
| **定义** | 实际写磁盘字节 / 逻辑写字节 | 一次读的实际 IO 次数 | 存储占用 / 数据实际大小 |
| **LSM（Leveled）** | 10-30x | 5-20x | 1.1-2x（旧版本待 Compaction）|
| **B+ 树（InnoDB）** | 2-3x | 1-2x（树高 IO）| 1.1-1.2x |

### RocksDB 核心特性

| 特性 | 说明 |
|------|------|
| **列族（Column Family）** | 独立的 Memtable/SSTable/Compaction，同一实例逻辑隔离不同数据 |
| **前缀迭代** | Prefix Bloom Filter 高效扫描同前缀 key（如 `userId:*`）|
| **事务** | 悲观事务（写锁）+ 乐观事务（提交时冲突检测）|
| **备份/Checkpoint** | 物理快照，不停写创建一致性备份 |
| **rate_limiter** | 限制 Compaction IO 速率，避免抢占前台读写 |

---

## 关键权衡

1. **写放大 vs 读放大的天平**：Leveled Compaction 写放大大（每 key 平均被写 10-30 次磁盘）但读快；Size-Tiered 写放大小但读慢——根据实际读写比选择
2. **Compaction 占用 IO**：Compaction 是后台任务，会抢占磁盘 IO，影响前台读写延迟；线上必须设置 `rate_limiter`（如限制 100MB/s），宁可 Compaction 慢一些
3. **空间回收延迟**：删除操作写入墓碑（Tombstone）记录，真正释放空间在 Compaction 后；极端情况下大量删除后磁盘反而膨胀（旧数据 + 墓碑同时占用）
4. **L0 文件数是热点**：L0 不做排序，文件多了读放大急剧恶化，且 L0→L1 Compaction 是写入停顿的根源；需严格配置 `level0_slowdown_writes_trigger`（慢写）和 `level0_stop_writes_trigger`（停写）阈值
5. **LSM vs B+ 树选型**：写吞吐优先（时序/消息/日志存储）选 LSM；强一致点查性能优先（OLTP 金融交易）选 B+ 树（InnoDB）

---

## 与其他概念的关系

- **对立于 [[机制-B+树]]**：B+ 树原地更新优化随机读（低读放大），LSM 顺序追加优化随机写（低写延迟）；InnoDB 用 B+树，TiKV/RocksDB 用 LSM 树——两种不同的读写权衡
- **借鉴 [[机制-MySQL三种日志]]**：WAL（Write-Ahead Log）是 LSM 和 InnoDB redo log 共享的崩溃恢复思想；Memtable flush = InnoDB Buffer Pool 刷脏页
- **类比 [[机制-Redis持久化]]**：RDB ≈ SSTable 全量快照；AOF ≈ WAL 顺序追加；LSM Compaction ≈ AOF rewrite 整理
- **支撑 [[概念-分布式系统理论]]**：TiKV（TiDB 底层）、CockroachDB、Cassandra、HBase 均基于 LSM；RocksDB 是工程落地的事实标准

---

## 应用边界

**适合 LSM/RocksDB**：
- 写多读少（消息存储、时序数据、IoT 设备日志）
- 作为**事务兜底存储**（游戏/交易系统异步写入，极端网络下保证最终一致性）
- 嵌入式存储（直接 JNI 调用，无独立进程）
- 分布式存储引擎底层（Flink RocksDB State Backend、TiKV、MyRocks）

**不适合**：
- 高频随机点查 + 复杂 JOIN（B+ 树更优，InnoDB 是正确选择）
- 数据量小（< 1GB），B+ 树简单可靠
- 事务 ACID 要求严格的 OLTP 核心链路
