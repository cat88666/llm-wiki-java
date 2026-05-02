---
type: concept
status: active
name: "MVCC"
layer: L5
aliases: ["多版本并发控制", "快照读", "当前读", "ReadView", "undo log版本链", "可重复读", "读已提交"]
related:
  - "[[机制-MySQL三种日志]]"
  - "[[机制-InnoDB锁机制]]"
  - "[[机制-InnoDB索引模型]]"
  - "[[机制-CAS]]"
sources:
  - "../../raw/note/📚 Hollis Java/MySQL/✅如何理解MVCC？ 30f3673e113881aca193c5642fe7ad5d.md"
  - "../../raw/note/📚 Hollis Java/MySQL/✅什么是ReadView，什么样的ReadView可见？ 30f3673e1138817a929ac16740b136e4.md"
  - "../../raw/note/📚 Hollis Java/MySQL/✅什么是脏读、幻读、不可重复读？ 30f3673e1138814f985ce7e54b00237e.md"
  - "../../raw/note/📚 Hollis Java/MySQL/✅MySQL如何实现不同隔离级别？ 30f3673e11388141b7c7c9dc8f8f041d.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# MVCC（多版本并发控制）

> MVCC 是解决读-写并发的乐观并发控制方案——不加锁，而是通过 undo log 保存数据历史版本链，结合 ReadView 的可见性规则，让读操作读到"应该看到的"快照版本；InnoDB 在 RC 和 RR 隔离级别下使用 MVCC，核心是 ReadView 的生成时机不同。

## 第一性原理

并发读写有三个经典问题：脏读（读到未提交数据）、不可重复读（同一事务两次读结果不同）、幻读（范围查询行数变化）。解决方法有两个方向：①悲观锁——读也加锁（Serializable 用此方案，性能差）；②乐观锁——保留历史版本，读时选合适的版本。MVCC 的存在理由：**让读不阻塞写、写不阻塞读，通过版本选择代替锁**。

## 核心机制

### 三大基础设施

**① 行记录的三个隐式字段（聚簇索引叶子节点）**

```
每行数据隐含：
  db_row_id    — 隐藏主键（无主键时用于构建聚簇索引）
  db_trx_id    — 最近一次修改该行的事务 ID
  db_roll_ptr  — 回滚指针 → 指向 undo log 中的上一个版本
```

**② undo log 版本链**

```
当前版本: {name="Hollis", db_trx_id=8, db_roll_ptr → }
              ↓
     版本 v2: {name="Bob", db_trx_id=5, db_roll_ptr → }
              ↓
     版本 v1: {name="Alice", db_trx_id=2, db_roll_ptr → null}
```

每次 UPDATE/DELETE 前，旧版本写入 undo log，形成单向链表。

**③ ReadView（事务的"可见性快照"）**

ReadView 包含：
- `trx_ids`：生成 ReadView 时系统中**活跃（未提交）**的事务 ID 列表
- `up_limit_id`：活跃事务中最小的事务 ID（低水位）
- `low_limit_id`：下一个将分配的事务 ID（高水位）
- `creator_trx_id`：创建这个 ReadView 的事务 ID

### 可见性判断规则

当读一行数据时，取其 `db_trx_id` 与 ReadView 比较：

```
db_trx_id < up_limit_id
  → 该事务在 ReadView 生成前已提交 → 可见 ✓

db_trx_id >= low_limit_id
  → 该事务在 ReadView 生成后才开启 → 不可见 ✗

up_limit_id ≤ db_trx_id < low_limit_id
  ├── db_trx_id 在 trx_ids 中（仍活跃）→ 不可见 ✗
  ├── db_trx_id 不在 trx_ids 中（已提交）→ 可见 ✓
  └── db_trx_id == creator_trx_id（自己的修改）→ 可见 ✓

不可见时：沿 db_roll_ptr 找 undo log 上一个版本，重复判断
```

### 快照读 vs 当前读

| 类型 | 语句 | 机制 | 场景 |
|------|------|------|------|
| **快照读** | `SELECT`（不加锁）| MVCC，读 undo log 快照 | 普通查询，不阻塞写 |
| **当前读** | `SELECT FOR UPDATE`、`SELECT LOCK IN SHARE MODE`、`UPDATE`、`DELETE`、`INSERT` | 加锁，读最新已提交数据 | 需要确保读到最新值 |

### RC vs RR 的关键差异：ReadView 生成时机

| 隔离级别 | ReadView 生成时机 | 效果 |
|---------|----------------|------|
| **Read Committed（RC）** | 每次 SELECT 都生成新 ReadView | 能看到其他事务的最新提交 → 解决脏读，但存在不可重复读 |
| **Repeatable Read（RR）** | 事务首次 SELECT 时生成，整个事务复用 | 始终看同一个快照 → 解决脏读 + 不可重复读 |

**幻读的解决**：RR 下快照读靠 MVCC 避免幻读，当前读（`SELECT FOR UPDATE`）靠 Next-Key Lock 避免幻读。

### 四种隔离级别与实现

| 隔离级别 | 实现方式 | 解决 |
|---------|---------|------|
| Read Uncommitted | 直接读最新数据，无控制 | 什么都不解决 |
| Read Committed | 每次快照读生成新 ReadView | 脏读 |
| Repeatable Read（默认）| 首次生成 ReadView 复用 + Next-Key Lock | 脏读 + 不可重复读 + （大部分）幻读 |
| Serializable | 所有读加共享锁，写加排他锁 | 三者全解决，但性能最差 |

## 关键权衡

1. **undo log 不会立即删除**：ReadView 还在引用旧版本时，undo log 不能被 purge 线程回收 → 长事务会导致 undo log 膨胀，影响性能和存储
2. **RR 下 MVCC 没有完全解决幻读**：当前读（`SELECT FOR UPDATE`）仍可能幻读，需要 Gap Lock / Next-Key Lock 补充
3. **RC 改为大厂默认的原因**：RR 的 Next-Key Lock 范围锁粒度大，高并发下死锁概率高；RC 的行锁粒度小，并发度更高（大厂通过 binlog 的 row 格式 + 业务幂等解决主备一致性问题）

## 与其他概念的关系

- 依赖 [[机制-MySQL三种日志]]：undo log 是版本链的物理存储；redo log 保证 undo log 本身的持久性
- 依赖 [[机制-InnoDB索引模型]]：隐式字段（`db_trx_id`、`db_roll_ptr`）存在于聚簇索引叶子节点
- 与 [[机制-InnoDB锁机制]] 互补：快照读用 MVCC，当前读用锁；两者共同实现隔离性
- 类比 [[机制-CAS]]：都是乐观并发控制思想——先操作后检查，而非提前加锁

## 应用边界

**MVCC 生效的场景**：RC 和 RR 隔离级别下的普通 SELECT（快照读）。

**MVCC 不生效的场景**：
- `Serializable`：全部加锁，不用 MVCC
- `Read Uncommitted`：直接读最新数据，不用 MVCC
- 当前读（`SELECT FOR UPDATE`）：读最新版本 + 加锁

**长事务风险**：事务开启越早，undo log 版本链越长，GC 越慢，尽量保持短事务。
