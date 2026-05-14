---
type: concept
status: active
name: "Canal数据同步"
layer: L7
aliases: ["Canal", "binlog订阅", "CDC", "数据变更捕获"]
tags: ["#distributed"]
related:
  - "[[机制-MySQL三种日志]]"
  - "[[机制-倒排索引与ElasticSearch]]"
  - "[[概念-分库分表]]"
sources:
  - "../../../raw/note/Hollis/分布式/✅什么是Canal，他的工作原理是什么？.md"
created: 2026-05-07
updated: 2026-05-07
---

# Canal 数据同步

**一句话定义**：Canal 是阿里开源的数据变更捕获（CDC）工具，通过模拟 MySQL slave 拉取 binlog，将数据库的增量变更以实时流的方式推送给下游系统（ES、缓存、异构库等）。

---

## 第一性原理

数据库本身不直接支持"有变更时通知外部系统"。轮询方案（定时 SELECT）有延迟、有压力；触发器方案侵入性强、难维护。Canal 利用 MySQL 主从复制的现有协议——**binlog 就是数据库的变更流水账**，伪装成 slave 来订阅它，以零侵入的方式实现增量数据捕获。

---

## 核心机制

### MySQL 主从复制原理（基础）

```
Master                    Slave
  │  binlog（变更事件）      │
  ├─────────── dump ──────► │  IO Thread 拉取 binlog → Relay Log
                            │  SQL Thread 重放 Relay Log → 数据同步
```

MySQL slave 通过向 master 发送 `COM_BINLOG_DUMP` 请求，持续拉取 binlog 事件并重放。

### Canal 工作原理

```
Canal Server
  │
  ├── 向 MySQL Master 注册，伪装成 slave
  ├── 发送 dump 请求拉取 binlog
  ├── 解析 binlog 事件（INSERT/UPDATE/DELETE → 结构化变更数据）
  └── 投递给 Canal Client / MQ（Kafka/RocketMQ）→ 消费端同步到 ES/缓存/异构库
```

**Canal 消费端可以是**：
- 直连 Canal Server 的 Java Client
- 通过 MQ 解耦（Canal Adapter 投递到 Kafka，消费者处理后写入 ES）

### 核心配置

Canal 需要 MySQL 开启 binlog（`binlog_format = ROW`），且创建有 REPLICATION SLAVE 权限的账号：

```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
```

ROW 格式 binlog 记录每一行的完整前后镜像，Canal 解析后可得到精确的字段级变更。

---

## 典型应用场景

| 场景 | 说明 |
|------|------|
| **MySQL → Elasticsearch** | Canal 订阅 binlog，商品/订单变更实时同步到 ES，支持全文检索 |
| **分库分表买卖家表同步** | 按买家 ID 分表 → Canal 解析 → 按卖家 ID 写入卖家分表，实现多维度查询 |
| **缓存与 DB 一致性** | DB 变更后 Canal 通知缓存失效/更新，比在业务代码中手动删缓存更可靠 |
| **数据迁移** | 业务双写过渡期结束后，Canal 补偿存量数据迁移 |
| **审计日志** | 捕获所有数据变更，写入审计系统，无需修改业务代码 |

---

## 关键权衡

1. **ROW vs STATEMENT binlog**：Canal 要求 ROW 格式；STATEMENT 格式只记录 SQL，无法精确还原每行变更（含非确定性函数时会出错）
2. **延迟问题**：Canal 是近实时（毫秒级）而非零延迟；下游若有 MQ 环节，延迟叠加可达秒级
3. **消费幂等**：Canal 基于 binlog position 消费，canal client 需记录消费位点（checkpoint）；异常重启后可能重复消费，消费端必须实现[[概念-幂等设计|幂等]]
4. **大事务风险**：单个大事务产生大量 binlog，Canal 解析和投递会有明显延迟峰值
5. **DDL 变更处理**：Canal 可捕获 DDL（ALTER TABLE 等），消费端需要能处理 schema 变化，否则会解析失败

---

## 与其他概念的关系

- **[[机制-MySQL三种日志]]**：Canal 依赖 binlog（WAL 日志中的逻辑日志），ROW 格式 binlog 是 Canal 工作的前提
- **[[机制-倒排索引与ElasticSearch]]**：MySQL → Canal → ES 是最常见的 ES 数据同步方案，替代全量同步（效率低）和业务代码双写（侵入性强）
- **[[概念-分库分表]]**：买卖家表双向维护是 Canal 的经典使用场景；分库分表扩容期间也可用 Canal 做增量数据迁移

---

## 应用边界

**适合用 Canal 的场景**：
- 需要实时感知数据库变更，但不想修改业务代码（非侵入）
- 异构数据源同步（MySQL → ES/Redis/其他 DB）
- 分库分表中跨维度查询的数据冗余维护

**不适合的场景**：
- 对延迟要求极低（亚毫秒级）：binlog 传输本身有延迟
- 非 MySQL 数据源：Canal 专为 MySQL binlog 设计，其他 DB 需用对应的 CDC 工具（如 Debezium）
- 全量初始化：Canal 只处理增量，全量数据需额外方案（mysqldump 或全量扫描）
