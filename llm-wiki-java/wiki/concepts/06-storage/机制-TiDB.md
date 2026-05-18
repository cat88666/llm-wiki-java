---
type: concept
status: active
name: "TiDB"
layer: L5
aliases: ["TiDB", "TiKV", "PD", "Placement Driver", "NewSQL", "HTAP", "分布式数据库", "分布式SQL", "Region", "Raft", "TSO", "MVCC", "Percolator", "TiFlash"]
tags: ["#storage", "#distributed"]
related:
  - "[[概念-MySQL]]"
  - "[[机制-RocksDB]]"
  - "[[概念-分布式理论]]"
  - "[[概念-分布式事务]]"
sources:
  - "../../../raw/note/Hollis/分布式/✅什么是分布式数据库，有什么优势？.md"
  - "../../../raw/note/大数据/01-简历.md"
updated: 2026-05-18
---

# TiDB

> TiDB 是兼容 MySQL 协议和 SQL 语法的分布式 NewSQL 数据库：TiDB Server 负责 SQL 解析与优化，PD 负责元数据、调度和全局时间戳，TiKV 负责基于 Raft 的分布式 KV 存储，TiFlash 提供列式分析副本。它的核心目标不是“把 MySQL 拆成很多库”，而是在 SQL、事务、一致性和弹性扩缩容之间做系统级平衡。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、TiDB 解决的 MySQL 扩展问题](#一tidb-解决的-mysql-扩展问题) | 单机瓶颈、分库分表痛点、NewSQL 的定位 |
| [二、TiDB Server、PD、TiKV、TiFlash 架构](#二tidb-serverpdtikvtiflash-架构) | 四类核心组件的职责边界 |
| [三、Region、Raft 与自动调度](#三regionraft-与自动调度) | 数据切片、副本一致性、Leader 调度、热点迁移 |
| [四、SQL 到 KV 的执行链路](#四sql-到-kv-的执行链路) | SQL 层优化、执行计划、谓词下推、分布式执行 |
| [五、TSO、MVCC 与分布式事务](#五tsomvcc-与分布式事务) | 全局时间戳、快照读、两阶段提交、故障恢复 |
| [六、HTAP 与 TiFlash](#六htap-与-tiflash) | 行存 OLTP 与列存 OLAP 的协同 |
| [七、TiDB vs MySQL 分库分表 / OceanBase](#七tidb-vs-mysql-分库分表--oceanbase) | 选型边界和架构差异 |
| [八、生产风险与调优](#八生产风险与调优) | 热点写、统计信息、跨机房延迟、长事务、资源隔离 |
| [九、TiDB 在存储体系的位置](#九tidb-在存储体系的位置) | 与 MySQL、RocksDB、分布式理论的关系 |
| [十、适用边界](#十适用边界) | 适合和不适合的业务场景 |
| [十一、面试速答](#十一面试速答) | 高频问题的压缩回答 |

## 一、TiDB 解决的 MySQL 扩展问题

单机 MySQL 的强项是成熟、稳定、事务能力强；瓶颈是容量、写吞吐、单表规模和跨库治理。数据量和并发继续上升时，常见做法是分库分表、读写分离和缓存，但这些方案会把复杂度暴露给业务层。

| 问题 | MySQL 常见方案 | 暴露的复杂度 | TiDB 的思路 |
| --- | --- | --- | --- |
| 单机容量不够 | 分库分表 | 分片键选择、扩容迁移、跨分片查询 | 自动切分 Region，存储层横向扩展 |
| 写吞吐不够 | 多主键空间拆分 | 热点表、热点分片、全局 ID | Region 调度、Leader 迁移、热点打散 |
| 跨分片事务 | XA/TCC/Saga/最终一致 | 业务补偿和异常流程复杂 | 数据库层提供分布式事务 |
| SQL 查询 | 中间件路由 | 跨分片 JOIN/聚合/排序困难 | SQL 层统一优化和下推 |
| 运维扩容 | 手工迁移 | 停机、双写、校验、回滚 | PD 调度副本和 Region 迁移 |

TiDB 属于 NewSQL：尽量保留传统关系型数据库的 SQL、事务和一致性，同时把存储与计算拆开，用分布式一致性协议承载水平扩展。它不是缓存、不是 NoSQL，也不是简单的 MySQL Proxy。

## 二、TiDB Server、PD、TiKV、TiFlash 架构

```
Client / MySQL Driver
        │
        ▼
TiDB Server（无状态 SQL 层，可横向扩容）
        │  生成执行计划 / 下推算子 / 请求 KV
        ├──────────────┐
        ▼              ▼
PD（元数据、调度、TSO） TiKV（行存 KV，Raft 多副本，RocksDB 落盘）
                       │
                       ▼
                  TiFlash（列式副本，分析查询）
```

| 组件 | 核心职责 | 关键点 |
| --- | --- | --- |
| TiDB Server | 兼容 MySQL 协议，解析 SQL，生成执行计划，执行分布式查询 | 无状态，扩容相对简单；不存储数据 |
| PD | 管理 Region 元数据、集群拓扑、调度策略、全局时间戳 TSO | 集群“大脑”；调度数据均衡和热点迁移 |
| TiKV | 分布式事务 KV 存储，数据按 Region 切分，多副本 Raft 保一致 | 底层使用 RocksDB，适合写入和持久化 |
| TiFlash | 列式存储副本，承接 OLAP 扫描和聚合 | 通过 Raft Learner 同步数据，降低分析查询对 TiKV 的压力 |

TiDB Server 可以理解为“分布式 SQL 计算层”，TiKV 是“分布式事务 KV 存储层”，PD 是“全局协调与调度层”。三者分工清晰，才使 TiDB 可以做到计算扩容、存储扩容和自动负载均衡。

## 三、Region、Raft 与自动调度

TiKV 不按“库表”存储数据，而是把有序 Key 空间切成多个 Region。每个 Region 是一个连续 Key 范围，默认维护多个副本，并通过 Raft 选出 Leader。

```
Table Row / Index
      │ 编码成有序 Key
      ▼
Key Space:  [k1 ... k999] [k1000 ... k1999] [k2000 ...]
              Region A       Region B         Region C
                 │              │                │
          Leader + Follower + Follower（Raft 副本组）
```

### Region 切分

数据写入后 Region 会随大小增长而 split。切分后，新的 Region 可以被 PD 调度到其他 TiKV 节点，避免单机容量成为上限。

| 机制 | 作用 |
| --- | --- |
| Region split | 把大 Key 范围拆成小范围，便于并行调度 |
| Region merge | 合并小 Region，减少元数据和调度开销 |
| Leader transfer | 迁移 Region Leader，均衡写入和读请求压力 |
| Peer add/remove | 增删副本，完成扩容、缩容和故障恢复 |

### Raft 一致性

每个 Region 的多个副本组成一个 Raft Group。写请求先到 Leader，Leader 把日志复制到多数派，提交后再应用到 RocksDB。这样即使少数副本故障，已提交数据仍不会丢。

| 场景 | Raft 行为 | 对业务的影响 |
| --- | --- | --- |
| 单副本宕机 | 多数派仍可提交 | 业务基本无感，后台补副本 |
| Leader 宕机 | Follower 重新选主 | 短暂抖动后恢复 |
| 多数派不可用 | 无法提交写入 | 保护一致性，牺牲可用性 |

### PD 自动调度

PD 持续采集 Region 分布、Leader 分布、磁盘容量和热点信息，再生成调度计划。它能处理三类问题：容量不均、Leader 不均、热点不均。

| 调度目标 | 典型表现 | 调度动作 |
| --- | --- | --- |
| 容量均衡 | 某些 TiKV 磁盘占用高 | 迁移 Region Peer |
| Leader 均衡 | 某台 TiKV 承担大量写入 | 转移 Region Leader |
| 热点打散 | 自增主键或时间序写入集中 | 分裂热点 Region、转移 Leader、调整分布 |

## 四、SQL 到 KV 的执行链路

TiDB 的 SQL 执行不是把 SQL 原样发给 TiKV。TiDB Server 会先解析、优化，再把能下推的过滤、聚合、TopN 等算子下推到 TiKV 或 TiFlash。

```
SQL
  → Parser 解析语法
  → Logical Plan 逻辑计划
  → Optimizer 基于统计信息选择索引、Join 顺序、下推策略
  → Physical Plan 物理计划
  → 按 Region 拆分请求
  → TiKV / TiFlash 并行执行
  → TiDB Server 汇总、排序、返回结果
```

### 示例：按索引查询

```sql
SELECT id, name
FROM user
WHERE phone = '13800000000';
```

执行链路：

1. TiDB Server 解析 SQL，根据统计信息选择 `phone` 索引。
2. 索引 key 被编码成 TiKV 的有序 KV key。
3. TiDB Server 根据 Region 元数据找到目标 Region Leader。
4. TiKV 在 RocksDB 中读取索引记录，拿到主键 handle。
5. 如果不是覆盖索引，再读取行数据 key。
6. TiDB Server 组装 MySQL 协议结果返回客户端。

### 示例：范围扫描与下推

```sql
SELECT city, COUNT(*)
FROM order_record
WHERE created_at >= '2026-05-01'
GROUP BY city;
```

如果统计信息准确，TiDB 会把范围扫描拆到多个 Region 并行执行，并尽量把过滤和局部聚合下推到 TiKV 或 TiFlash。这样网络上传输的不是全量原始行，而是局部计算后的中间结果。

## 五、TSO、MVCC 与分布式事务

TiDB 的事务核心由三块组成：PD 提供全局单调递增时间戳 TSO，TiKV 用 MVCC 保存多个版本，提交协议采用 Percolator 思路的两阶段提交。

### TSO：全局事务时间

分布式数据库不能只依赖单机本地时间，否则不同节点对事务先后顺序会产生歧义。TiDB 通过 PD 分配全局时间戳：

| 时间戳 | 含义 |
| --- | --- |
| `start_ts` | 事务开始时间，决定读快照 |
| `commit_ts` | 事务提交时间，决定版本可见性 |

读请求只读取 `commit_ts <= start_ts` 的版本，因此可以获得一致性快照。

### MVCC：同一 Key 多版本

TiKV 中同一个逻辑行会编码成多个版本。新事务写入不会覆盖旧版本，而是产生新版本；旧版本用于快照读和回滚。

```
user:1
  version commit_ts=120  value={name: "A"}
  version commit_ts=150  value={name: "B"}
  version commit_ts=180  value={name: "C"}

事务 start_ts=160 读取到 B
事务 start_ts=200 读取到 C
```

这和 MySQL InnoDB 的 MVCC 思想类似：读不阻塞写，写不阻塞普通快照读。区别在于 TiDB 要在多个 TiKV 节点之间维持全局快照。

### 两阶段提交：跨 Region 原子提交

一次事务可能修改多个 Region。TiDB 需要保证这些修改要么全部成功，要么全部失败。

```sql
BEGIN;
INSERT INTO account(id, balance) VALUES (1, 100);
INSERT INTO account_log(id, account_id, amount) VALUES (10, 1, 100);
COMMIT;
```

简化流程：

1. TiDB 获取 `start_ts`，后续读基于该快照。
2. 两条 `INSERT` 被编码成多个 KV mutation，可能落在不同 Region。
3. TiDB 选择一个 key 作为 primary key，其余为 secondary key。
4. Prewrite 阶段：向相关 Region 写入锁和临时版本；如果发现写写冲突，事务失败。
5. 获取 `commit_ts`。
6. Commit primary key：提交主 key，事务从逻辑上成功。
7. Commit secondary keys：异步或批量提交其他 key。
8. 其他事务根据 primary key 状态判断 secondary key 是提交还是回滚。

如果 prewrite 期间失败，所有已写锁会被清理，临时版本不可见；如果 primary 已提交但 secondary 未全部提交，后续读写会通过 primary 状态完成提交或回滚判定。这是 TiDB 能在多 Region 上提供事务原子性的关键。

### 与 MySQL 本地事务的差异

| 维度 | MySQL InnoDB | TiDB |
| --- | --- | --- |
| 时间顺序 | 单机事务系统内部维护 | PD 分配全局 TSO |
| 数据位置 | 单机表空间/页 | 多 TiKV、多 Region |
| 提交流程 | 本地 redo/undo/binlog 协作 | 跨 Region prewrite/commit |
| 一致性协议 | 单机锁和日志 | MVCC + 2PC + Raft |
| 主要开销 | 磁盘 IO、锁冲突、日志刷盘 | 网络 RTT、Raft 复制、跨 Region 事务 |

## 六、HTAP 与 TiFlash

传统架构常把 OLTP 和 OLAP 拆成两套系统：MySQL 承接交易，离线数仓或 ClickHouse 承接分析。TiDB 的 HTAP 思路是在同一套数据上同时提供行存和列存能力。

| 存储 | 形态 | 适合 |
| --- | --- | --- |
| TiKV | 行存 KV，基于 RocksDB | 点查、短事务、OLTP 写入 |
| TiFlash | 列式副本 | 大范围扫描、聚合、报表分析 |

TiFlash 通过 Raft Learner 机制从 TiKV 同步数据，不参与多数派投票，避免影响 OLTP 写入多数派提交。优化器可以根据查询代价选择 TiKV 或 TiFlash。

适合走 TiFlash 的查询：

```sql
SELECT product_id, SUM(amount)
FROM order_record
WHERE created_at >= '2026-01-01'
GROUP BY product_id;
```

不适合走 TiFlash 的查询：

```sql
SELECT *
FROM order_record
WHERE order_id = 1000001;
```

点查和短事务优先走 TiKV；大扫描、聚合和分析优先走 TiFlash。

## 七、TiDB vs MySQL 分库分表 / OceanBase

| 维度 | MySQL 分库分表 | TiDB | OceanBase |
| --- | --- | --- | --- |
| SQL 兼容 | 原生 MySQL，但跨分片受限 | 兼容 MySQL 协议和大部分语法 | 兼容 MySQL/Oracle 模式 |
| 扩展方式 | 业务或中间件按分片键路由 | Region 自动切分和调度 | 分区和副本调度 |
| 跨分片事务 | 依赖中间件或业务方案 | 数据库层分布式事务 | 数据库层分布式事务 |
| 运维复杂度 | 分片、迁移、路由复杂 | 集群组件多，但扩缩容自动化更强 | 集群体系完整，运维体系偏重 |
| 适合场景 | 分片模型稳定、SQL 简单 | 需要 MySQL 生态 + 水平扩展 + 强一致 | 金融级强一致、超大规模集群 |

选型不是“TiDB 一定替代 MySQL”。如果单机 MySQL 能稳定承载，分库分表规则简单且团队已经有成熟治理，继续使用 MySQL 可能成本更低。TiDB 的价值主要出现在容量、扩容、跨分片 SQL 和跨分片事务已经成为主要矛盾时。

## 八、生产风险与调优

| 风险 | 表现 | 根因 | 应对 |
| --- | --- | --- | --- |
| 热点写 | 某个 TiKV 或 Region 写入延迟高 | 自增主键、时间序 key、单调递增索引集中到右端 Region | 打散 key、预分裂、热点调度、避免强单调写入 |
| 统计信息不准 | 执行计划突然变差、慢 SQL 增多 | 数据分布变化后优化器估算错误 | 定期 analyze、观察执行计划、必要时绑定执行计划 |
| 跨 Region 大事务 | 提交延迟高、锁冲突多 | mutation 多、涉及 Region 多、2PC 和 Raft 开销叠加 | 拆小事务、批量大小控制、避免长事务 |
| 跨机房部署延迟 | 写入 RTT 明显升高 | Raft 多数派复制跨地域 | 同城多 AZ 优先，跨地域需明确 RPO/RTO 与延迟目标 |
| TiFlash 资源争抢 | OLAP 查询影响 OLTP | 大查询占 CPU/IO/网络 | 资源隔离、限流、合理选择 TiFlash 副本 |
| 小集群成本高 | 资源利用率低、组件运维重 | TiDB 是分布式系统，基础组件多 | 小业务优先 MySQL，避免为未来规模过早上 TiDB |
| 兼容性差异 | 某些 SQL、函数、执行行为与 MySQL 不完全一致 | 分布式执行和兼容范围有限 | 上线前兼容性测试，重点验证事务、DDL、复杂 SQL |

TiDB 的慢 SQL 排查要同时看 SQL 层和存储层：执行计划是否合理、统计信息是否准确、是否发生 coprocessor 慢请求、是否存在 Region 热点、TiKV RocksDB 是否有 Compaction 或 IO 抖动。

## 九、TiDB 在存储体系的位置

| 关联概念 | TiDB 中的体现 |
| --- | --- |
| [[概念-MySQL]] | 兼容 MySQL 协议和 SQL 生态，但底层不是 InnoDB |
| [[机制-RocksDB]] | TiKV 使用 RocksDB 作为本地持久化引擎 |
| [[概念-分布式理论]] | Raft、多数派、CAP 权衡、调度和故障恢复 |
| [[概念-分布式事务]] | TSO、MVCC、2PC、事务冲突检测 |
| [[机制-B+树]] | TiDB SQL 索引语义类似 MySQL，但落到分布式 KV key 空间 |

TiDB 可以看作“SQL 入口 + 分布式事务 KV + 一致性复制 + 自动调度”的组合。理解它时，不要只从 MySQL 角度看，也要从分布式系统和 LSM 存储引擎角度看。

## 十、适用边界

### 适合

| 场景 | 原因 |
| --- | --- |
| 数据规模持续增长，单机 MySQL 容量逼近上限 | TiKV 可水平扩展存储容量 |
| 分库分表治理成本过高 | TiDB 屏蔽大量分片和迁移复杂度 |
| 需要强一致事务和 MySQL 生态 | SQL 兼容 + 分布式事务 |
| 需要实时分析但不想维护多套链路 | TiFlash 提供 HTAP 能力 |
| 业务增长不确定，需要弹性扩容 | Region 调度和无状态 SQL 层更利于扩展 |

### 不适合

| 场景 | 原因 |
| --- | --- |
| 小规模业务、单机 MySQL 足够 | TiDB 运维和资源成本更高 |
| 极致低延迟单行写 | 分布式事务、Raft、多网络 RTT 有额外开销 |
| 大量超长事务或超大事务 | 锁、MVCC 版本、2PC 成本高 |
| 强依赖 MySQL 特定语法或插件 | 兼容性需要逐项验证 |
| 简单 KV 缓存场景 | Redis 或专用 KV 更轻量 |

## 十一、面试速答

| 问题 | 回答 |
| --- | --- |
| TiDB 为什么能水平扩展？ | SQL 层无状态，数据按 Key Range 切成 Region，TiKV 节点可增加容量，PD 负责 Region 和 Leader 调度。 |
| TiDB 如何保证强一致？ | 每个 Region 内通过 Raft 多数派复制保证副本一致；跨 Region 事务通过 TSO、MVCC 和两阶段提交保证原子性。 |
| PD 的作用是什么？ | 管理 Region 元数据、调度副本和 Leader、处理热点/容量均衡，并提供全局时间戳 TSO。 |
| TiKV 和 RocksDB 什么关系？ | TiKV 是分布式事务 KV 层，RocksDB 是 TiKV 单机上的本地 LSM 存储引擎。 |
| TiDB 和分库分表最大区别？ | 分库分表把路由、扩容、跨分片 SQL/事务复杂度暴露给应用或中间件；TiDB 在数据库层处理切分、调度、SQL 和事务。 |
| TiFlash 解决什么问题？ | 用列式副本承接大范围扫描和聚合，让分析查询尽量不压垮 TiKV 的 OLTP 读写。 |
| TiDB 的主要代价是什么？ | 分布式事务、Raft 复制和网络 RTT 带来额外延迟；组件更多，运维复杂度高于单机 MySQL。 |
