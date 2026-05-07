---
type: concept
status: active
name: "MySQL三种日志"
layer: L5
aliases: ["binlog", "redo log", "undo log", "WAL", "两阶段提交", "崩溃恢复", "主从复制"]
related:
  - "[[机制-MVCC]]"
  - "[[机制-InnoDB锁机制]]"
sources:
  - "../../../raw/note/Hollis/MySQL/✅binlog、redolog和undolog区别？.md"
  - "../../../raw/note/Hollis/MySQL/✅MySQL事务ACID是如何实现的？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是事务的2阶段提交？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# MySQL 三种日志（binlog / redo log / undo log）

> MySQL 用三种日志共同保障事务 ACID：undo log 记录修改前快照（原子性 + MVCC），redo log 记录修改后操作（持久性 + 崩溃恢复），binlog 记录所有 DDL/DML（主从复制 + 数据备份）；binlog 与 redo log 通过两阶段提交保证一致性。

## 第一性原理

数据库面临两个对立需求：①高性能（尽量减少磁盘随机写）②高可靠（崩溃后数据不丢）。WAL（Write-Ahead Logging）是解决方案：**先写日志（顺序 IO）再更新数据页（随机 IO），崩溃时用日志重放**。三种日志分别承担不同职责，相互补充而非替代。

## 核心机制

### 三种日志对比

| 日志 | 属于 | 主要作用 | 记录内容 | 适用引擎 |
|------|------|---------|---------|---------|
| **binlog** | MySQL Server 层 | 主从复制、数据备份、崩溃恢复 | 所有 DDL/DML（逻辑操作）| 所有引擎 |
| **redo log** | InnoDB 引擎层 | 崩溃恢复（保证持久性）| 数据页的物理修改（"第 X 页第 Y 字节改为 Z"）| 仅 InnoDB |
| **undo log** | InnoDB 引擎层 | 事务回滚（保证原子性）+ MVCC 版本链 | 修改前的旧数据 | 仅 InnoDB |

**记忆口诀**：
- undo = "撤销"（Ctrl+Z），存操作前的旧值，用于回滚
- redo = "重做"，存操作后的新值，用于崩溃重放
- bin = 最全最原始的二进制记录，有它就有一切（备份/复制）

### 一条 UPDATE 的完整日志写入顺序

```
UPDATE user SET name='Hollis' WHERE id=10

① 读取 id=10 行到 Buffer Pool（若不在内存则磁盘 IO）
② 写 undo log：记录旧值 name='Alice'（原子性 + 版本链）
③ 修改 Buffer Pool 中的数据页（内存中改，未落盘）
④ 写 redo log（prepare 状态）：记录"页X字节Y改为Hollis"
⑤ 写 binlog：记录 UPDATE SQL 操作
⑥ redo log 提交（commit 状态）
⑦ Buffer Pool 的脏页后台异步刷盘（非事务提交时）
```

### 两阶段提交（2PC）— 保证 binlog 与 redo log 一致

**问题**：若先写 redo log 再写 binlog，redo log 写成但 binlog 未写就崩溃 → 主库恢复后有新数据，备库通过 binlog 同步没有新数据 → **主备数据不一致**。反之亦然。

**解决**：两阶段提交

```
Phase 1 — Prepare：
  ① 写 redo log，标记为 prepare 状态
  ② 生成 XID（全局事务 ID）

Phase 2 — Commit：
  ③ 写 binlog（含 XID）并刷盘（sync_binlog=1）
  ④ redo log 标记为 commit 状态
```

**崩溃恢复规则**：
- prepare 状态的 redo log + binlog 有对应 XID → **提交**（binlog 已完整，主备已一致）
- prepare 状态的 redo log + binlog **无**对应 XID → **回滚**（binlog 未写，主备不知道此事务）

### ACID 与日志的映射

| 特性 | 实现日志 | 机制 |
|------|---------|------|
| **原子性** | undo log | 事务失败时，通过 undo log 回滚所有修改 |
| **持久性** | redo log | 事务提交时 redo log 已落盘，崩溃后可重放 |
| **隔离性** | undo log + 锁 | undo log 版本链支持 MVCC；锁保证写隔离 |
| **一致性** | 上三者共同保障 | A+I+D 做好了，C 自然满足 |

### undo log 的生命周期

undo log 不会立即删除：当没有任何活跃 ReadView 引用它时，才可被 purge 线程清理。这就是**长事务会撑大 undo log 段**的原因。

## 关键权衡

1. **redo log 有固定大小**：InnoDB 的 redo log 是循环写的（类似环形缓冲区），写满后必须等待脏页刷盘。redo log 太小会导致频繁刷盘，影响写性能；太大则崩溃恢复时间长
2. **sync_binlog 和 innodb_flush_log_at_trx_commit 的取舍**：
   - 两者都设为 1（最安全）：每次事务提交都 fsync → 最安全但最慢
   - binlog 设 0/N，redo log 设 2（最快）：有丢数据风险
3. **binlog 格式选择**：row 格式（记录行变更）比 statement 格式（记录 SQL）更安全，大厂通常用 row 格式

## 与其他概念的关系

- undo log 是 [[机制-MVCC]] 的版本链存储基础；redo log 保证 undo log 本身的持久性
- 两阶段提交解决了 binlog + redo log 双写的一致性问题（类似分布式事务）
- 与 [[机制-InnoDB锁机制]]：锁机制保证写-写隔离；日志机制保证写的持久性和原子性

## 应用边界

**需要理解三日志的场景**：
- 排查主从延迟（binlog 同步速度）
- 分析长事务影响（undo log 堆积）
- 调优磁盘 IO（redo log 大小、刷盘策略）
- 主从复制原理（binlog → relay log → SQL thread 重放）

**运维注意**：
- 开启 `sync_binlog=1` + `innodb_flush_log_at_trx_commit=1` 保证零数据丢失
- 避免长事务（`innodb_trx` 表监控 > 60s 的活跃事务）
