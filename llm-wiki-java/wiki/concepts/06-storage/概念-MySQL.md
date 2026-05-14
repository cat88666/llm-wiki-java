---
type: concept
status: active
name: "MySQL体系"
layer: L5
tags: ["#storage"]
aliases: ["MySQL", "InnoDB", "B+树索引", "聚簇索引", "二级索引", "回表", "覆盖索引", "索引下推", "最左前缀", "联合索引", "共享锁", "排他锁", "意向锁", "记录锁", "间隙锁", "临键锁", "Next-Key Lock", "Gap Lock", "死锁", "MVCC", "多版本并发控制", "快照读", "当前读", "ReadView", "undo log版本链", "可重复读", "读已提交", "binlog", "redo log", "undo log", "WAL", "两阶段提交", "崩溃恢复", "主从复制", "慢SQL", "执行计划", "EXPLAIN", "索引失效", "深分页", "慢查询日志", "大表优化", "Online DDL", "热点行", "Canal", "binlog订阅", "CDC", "数据变更捕获", "分库分表", "雪花算法", "ShardingJDBC", "基因法", "时钟回拨", "数据倾斜", "扩容迁移"]
related:
  - "[[机制-B+树]]"
  - "[[机制-CAS]]"
  - "[[概念-分布式事务]]"
sources:
  - "../../../raw/note/Hollis/MySQL/✅InnoDB为什么使用B+树实现索引？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是聚簇索引和非聚簇索引？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是回表，怎么减少回表的次数？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是索引覆盖、索引下推？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是最左前缀匹配？为什么要遵守？.md"
  - "../../../raw/note/Hollis/MySQL/✅介绍下InnoDB的锁机制？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是意向锁？.md"
  - "../../../raw/note/Hollis/MySQL/✅MySQL的行级锁锁的到底是什么？.md"
  - "../../../raw/note/Hollis/MySQL/✅如何理解MVCC？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是ReadView，什么样的ReadView可见？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是脏读、幻读、不可重复读？.md"
  - "../../../raw/note/Hollis/MySQL/✅MySQL如何实现不同隔离级别？.md"
  - "../../../raw/note/Hollis/MySQL/✅binlog、redolog和undolog区别？.md"
  - "../../../raw/note/Hollis/MySQL/✅MySQL事务ACID是如何实现的？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是事务的2阶段提交？.md"
  - "../../../raw/note/Hollis/MySQL/✅如何进行SQL调优？.md"
  - "../../../raw/note/Hollis/MySQL/✅慢SQL的问题如何排查？.md"
  - "../../../raw/note/Hollis/MySQL/✅SQL执行计划分析的时候，要关注哪些信息？.md"
  - "../../../raw/note/Hollis/MySQL/✅MySQL的深度分页如何优化.md"
  - "../../../raw/note/Hollis/场景题/✅MySQL千万级大表中如何增加字段？.md"
  - "../../../raw/note/Hollis/场景题/✅MySQL千万级大表如何做数据清理？.md"
  - "../../../raw/note/Hollis/场景题/✅MySQL千万级数据量，查询如何做优化？.md"
  - "../../../raw/note/Hollis/场景题/✅MySQL单表一千万条数据怎么做分页查询？.md"
  - "../../../raw/note/Hollis/场景题/✅MySQL热点数据更新会带来哪些问题？.md"
  - "../../../raw/note/Hollis/场景题/✅5000w数据查询电话号码后4位，如何优化？.md"
  - "../../../raw/note/Hollis/场景题/✅代码中使用长事务，会带来哪些问题？.md"
  - "../../../raw/note/Hollis/场景题/✅使用分布式锁时，分布式锁加在事务外面还是里面，有什么区别？.md"
  - "../../../raw/note/Hollis/场景题/✅为啥不要在事务中做外部调用？.md"
  - "../../../raw/note/Hollis/场景题/✅扫表任务，如何写SQL可以避免出现跳页的情况？.md"
  - "../../../raw/note/Hollis/场景题/✅从B+树的角度分析为什么单表2000万要考虑分表？？.md"
  - "../../../raw/note/Hollis/场景题/✅数据库逻辑删除后，怎么做唯一性约束？.md"
  - "../../../raw/note/Hollis/分布式/✅什么是Canal，他的工作原理是什么？.md"
  - "../../../raw/note/Hollis/面经实战/✅6年经验，电商平台，分布式事务，分布式.md"
  - "../../../raw/note/Hollis/面经实战/✅工作4年，供应链相关，分库分表、分布式锁.md"
  - "../../../raw/note/Hollis/面经实战/✅工作7年，2家大厂经验，下单&导购核心开发.md"
  - "../../../raw/note/Hollis/面经实战/✅10年，深圳证券公司，系统架构组架构师，监控平台、电商平台，秒杀.md"
  - "../../../raw/note/Hollis/场景题/✅从B+树的角度分析为什么单表2000万要考虑分表？？.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# MySQL 体系

> InnoDB 用四大机制共同保障事务正确性：**索引**（B+ 树，减少磁盘 IO）控制数据访问路径；**锁**（Record/Gap/Next-Key Lock）控制写-写并发和当前读；**MVCC**（undo log 版本链 + ReadView）控制读-写并发；**三种日志**（undo/redo/binlog）保证 ACID 和主从一致。四者相互依赖，缺一不可。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 四大机制的存在理由、知识地图 |
| [二、InnoDB 索引](#二innodb-索引) | B+树选型、聚簇/二级索引、回表、覆盖索引、ICP、最左前缀 |
| [三、锁机制](#三锁机制) | S/X锁、意向锁、Record/Gap/Next-Key Lock、死锁 |
| [四、MVCC](#四mvcc) | 三大基础设施、ReadView可见性规则、快照读/当前读、RC vs RR |
| [五、三种日志](#五三种日志) | undo/redo/binlog 对比、UPDATE 完整流程、两阶段提交、ACID 映射 |
| [六、综合对比](#六综合对比) | MVCC vs 锁、RC vs RR 选型、四种隔离级别、三日志横向对比 |
| [七、生产风险](#七生产风险) | 长事务、无索引锁退化、死锁、刷盘策略、索引失效 |
| [八、与其他概念的关系](#八与其他概念的关系) | B+树、CAS、分布式事务、SQL优化 |
| [九、应用边界](#九应用边界) | 各机制的适用边界与常见误用 |
| [十、SQL 查询优化](#十sql-查询优化) | 慢SQL发现、EXPLAIN、索引失效、深分页、大表、热点行、长事务 |
| [十一、Canal 数据同步](#十一canal-数据同步) | binlog 订阅、CDC、MySQL→ES/缓存同步、幂等消费 |
| [十二、分库分表与全局ID](#十二分库分表与全局id) | 分片键选择、Hash取模、基因法、雪花算法、扩容迁移 |

## 一、第一性原理

数据库面临三个根本矛盾：

| 矛盾 | 机制 | 核心思路 |
|------|------|---------|
| 磁盘随机 IO 太慢 | InnoDB 索引（B+ 树）| 用有序树结构将随机 IO 降为 O(log n)，叶子链表支持范围扫描 |
| 读-写并发冲突 | MVCC | 保留历史版本，读不加锁，写不阻塞读 |
| 写-写并发冲突 + 当前读 | 锁机制 | 最小粒度加锁，不冲突的并发互不阻塞 |
| 崩溃丢数据 + 主备不一致 | 三种日志 + 两阶段提交 | WAL（顺序 IO 先写日志），2PC 保证 redo/binlog 原子提交 |

**知识依赖图**：

```
         存储物理层
    ┌──────────────────────────────────────────┐
    │  磁盘（B+ 树数据页 16KB）                  │
    │  Buffer Pool（内存缓冲池，减少磁盘 IO）      │
    └──────────────────┬───────────────────────┘
                       │
              InnoDB 索引（二、）
    ┌──────────────────────────────────────────┐
    │  聚簇索引（主键）: 叶子节点存整行数据          │
    │  二级索引（非主键）: 叶子节点存主键值 → 回表   │
    │  减少回表: 覆盖索引 / 索引下推               │
    │  联合索引 → 最左前缀匹配原则                 │
    └─────────┬─────────────────┬──────────────┘
              │                 │
    ┌──────────▼──┐       ┌─────▼────────────┐
    │  MVCC（四）  │       │  锁机制（三）       │
    │  快照读（无锁）│       │  当前读（加锁）     │
    │  ReadView   │       │  S/X/IS/IX       │
    │  undo log   │       │  Record/Gap/     │
    │  版本链可见性 │       │  Next-Key Lock   │
    └──────┬──────┘       └──────────────────┘
           │ 依赖
    ┌───────▼─────────────────────────────────┐
    │           三种日志（五、）                 │
    │  undo log: 原子性 + MVCC 版本链           │
    │  redo log: 持久性 + 崩溃恢复（WAL）        │
    │  binlog:   主从复制 + 数据备份             │
    │  两阶段提交: binlog ↔ redo log 一致性      │
    └─────────────────────────────────────────┘
```

## 二、InnoDB 索引

### B+ 树选型原因

磁盘 IO 代价极高，核心矛盾是**减少 IO 次数**。B+ 树用低树高（通常 3 层）覆盖千万级数据：

| 数据结构 | 缺点 |
|---------|------|
| 红黑树 | 树高随数据量线性增长，百万数据树高 20+，IO 多 |
| B 树 | 叶子节点不连接，范围查询需回溯 |
| Hash 索引 | 不支持范围查询和排序，仅等值查询有优势 |
| **B+ 树** | 非叶子节点不存数据，单节点存更多 key → 树更矮；叶子双向链表 → 范围扫描天然高效 |

### 聚簇索引 vs 二级索引

| 维度 | 聚簇索引（主键索引）| 二级索引（非聚簇索引）|
|------|----------------|-------------------|
| 叶子节点内容 | 整行数据 | 索引列值 + 主键值 |
| 数量 | 每表只有 1 个 | 可有多个 |
| 查询 | 直接得到数据 | 需要回表查聚簇索引 |
| 物理存储 | 数据按主键顺序存放 | 独立 B+ 树，不影响数据顺序 |

**无主键时**：InnoDB 选唯一非空索引作聚簇索引；都没有则用隐藏 `db_row_id`。

### 回表

```
查询: SELECT name FROM t WHERE age = 25;（age 有二级索引，无联合索引覆盖）

① 扫描 age 二级索引树 → 找到 age=25 → 得到主键 id=100
② 回聚簇索引 → 用 id=100 查整行 → 返回 name 字段  ← 这步叫"回表"
```

**减少回表的两种手段**：
- **覆盖索引**：查询所有字段都在索引中 → 直接从索引返回，EXPLAIN Extra: `Using index`
- **索引下推（ICP，MySQL 5.6+）**：在存储引擎层先用索引过滤，减少回聚簇索引次数，EXPLAIN Extra: `Using index condition`

```sql
-- 覆盖索引示例：联合索引 idx_age_name(age, name)
SELECT name FROM t WHERE age = 25;  -- 不回表

-- 索引下推示例：联合索引 idx_zipcode_lastname(zipcode, lastname)
SELECT * FROM t WHERE zipcode='95054' AND lastname LIKE '%etrunia%';
-- ICP：存储引擎层先在索引检查 lastname，不满足直接跳过，不回表
```

### 最左前缀匹配

联合索引 `(col1, col2, col3)`：先按 col1 排序，col1 相同按 col2，依此类推。

| WHERE 条件 | 走索引情况 |
|-----------|---------|
| `col1=?` | 走索引 |
| `col1=? AND col2=?` | 走索引 |
| `col1=? AND col2=? AND col3=?` | 走索引 |
| `col2=?` | **不走**联合索引 |
| `col1=? AND col3=?` | 只走 col1 部分，col3 无法利用 |

注意：WHERE 中条件顺序不影响结果（优化器会重排），但索引列本身的排序顺序不变。

### 主键选型

**自增整数 vs UUID**：自增主键叶子顺序插入，页分裂少；UUID 随机写入导致频繁页分裂，写性能差，B+ 树空间利用率低。

### 索引建立原则

**建索引**：WHERE 高频列、ORDER BY/GROUP BY 列、JOIN 连接列、区分度 > 95% 的列。

**不建/会失效**：
- 索引列使用函数或隐式类型转换（`WHERE year(date) = 2024`）
- LIKE 前导 `%`（`LIKE '%abc'` 走不了索引）
- OR 连接且其中一列无索引
- 区分度极低的列（性别等）
- 数据量极少的表（全表扫描更快）

## 三、锁机制

### 锁的两个基本类型

| 锁类型 | 符号 | 兼容关系 | 语义 |
|-------|------|---------|------|
| 共享锁（读锁）| S | S 与 S 兼容；S 与 X 不兼容 | 可并发读，阻止写 |
| 排他锁（写锁）| X | X 与任何锁不兼容 | 独占读写，阻止一切 |

```sql
SELECT ... LOCK IN SHARE MODE   -- 显式 S 锁
SELECT ... FOR UPDATE           -- 显式 X 锁
UPDATE / DELETE / INSERT        -- 自动加 X 锁
```

### 意向锁（表级）

**问题**：事务 A 对某行加了行级 X 锁，事务 B 想加表级 X 锁，需逐行检查是否有行锁 → 开销大。

**解决**：加行锁前先在表级加意向锁，让表锁申请时 O(1) 判断冲突。

| 意向锁 | 含义 | 触发时机 |
|-------|------|---------|
| IS（意向共享锁）| 表内有行要加 S 锁 | 加行 S 锁前自动加 |
| IX（意向排他锁）| 表内有行要加 X 锁 | 加行 X 锁前自动加 |

**意向锁由 MySQL 自动管理，开发者无需显式操作。**

### 行级锁三种形态

#### Record Lock（记录锁）

锁定具体的一条**索引记录**（锁的是 B+ 树节点上的 key，不是数据行）。

```sql
SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;
-- 对 c1=10 的索引记录加 X 记录锁
```

**无索引时**：退化为锁聚簇索引，实际效果接近锁全表。

#### Gap Lock（间隙锁）

锁定索引记录**之间的间隙**，防止其他事务在间隙中插入新记录。只在 RR 隔离级别生效。

```
索引值: ... 5, 10, 15, ...
Gap Lock (5, 10)：阻止任何事务在此间隙插入 6/7/8/9
```

#### Next-Key Lock（临键锁）

= Record Lock + 左侧 Gap Lock，范围**左开右闭**：`(prev_key, current_key]`

RR 下 InnoDB 的**加锁基本单位**，防止幻读。

**加锁优化规则**：
- 唯一索引等值查询**命中**时：Next-Key Lock 退化为 Record Lock（不锁间隙）
- 等值查询向右遍历且**最后一个值不满足条件**时：退化为 Gap Lock

### 锁的粒度层次

```
全局锁（FLUSH TABLES WITH READ LOCK）
  └── 表级锁
        ├── 意向锁（IS / IX）：自动管理，配合行锁使用
        ├── MDL 锁（元数据锁）：DDL 语句自动加，保护表结构
        └── AUTO-INC 锁：自增主键插入时临时持有
  └── 行级锁
        ├── Record Lock：锁索引记录
        ├── Gap Lock：锁间隙（RR 生效）
        └── Next-Key Lock：Record + Gap（RR 加锁基本单位）
```

### 死锁与解决

```
事务A: 持有 id=1 的 X 锁，等待 id=2 的 X 锁
事务B: 持有 id=2 的 X 锁，等待 id=1 的 X 锁  → 死锁
```

**InnoDB 自动检测死锁**（等待图算法），选代价最小的事务回滚（victim）。

**避免死锁**：所有事务按相同顺序访问资源；缩短事务减少持锁时间；避免大范围 WHERE 条件。

**排查锁问题**：
```sql
SELECT * FROM information_schema.innodb_trx;          -- 查活跃事务
SELECT * FROM information_schema.innodb_lock_waits;   -- 查锁等待
SHOW ENGINE INNODB STATUS;                            -- 查死锁信息
```

## 四、MVCC

### 三大基础设施

**① 行记录的三个隐式字段（聚簇索引叶子节点）**

```
db_row_id    — 隐藏主键（无主键时构建聚簇索引）
db_trx_id    — 最近一次修改该行的事务 ID
db_roll_ptr  — 回滚指针 → 指向 undo log 中的上一个版本
```

**② undo log 版本链**

```
当前版本: {name="Hollis", db_trx_id=8, db_roll_ptr → }
              ↓
     版本 v2: {name="Bob",   db_trx_id=5, db_roll_ptr → }
              ↓
     版本 v1: {name="Alice", db_trx_id=2, db_roll_ptr → null}
```

每次 UPDATE/DELETE 前，旧版本写入 undo log，形成单向链表。

**③ ReadView（事务的"可见性快照"）**

| 字段 | 含义 |
|------|------|
| `trx_ids` | 生成 ReadView 时系统中**活跃（未提交）**的事务 ID 集合 |
| `up_limit_id` | 活跃事务中最小的事务 ID（低水位）|
| `low_limit_id` | 下一个将分配的事务 ID（高水位）|
| `creator_trx_id` | 创建这个 ReadView 的事务 ID |

### 可见性判断规则

读一行数据时，取其 `db_trx_id` 与 ReadView 比较：

```
db_trx_id < up_limit_id
  → 该事务在 ReadView 生成前已提交 → 可见 ✓

db_trx_id >= low_limit_id
  → 该事务在 ReadView 生成后才开启 → 不可见 ✗

up_limit_id ≤ db_trx_id < low_limit_id
  ├── db_trx_id 在 trx_ids 中（仍活跃）       → 不可见 ✗
  ├── db_trx_id 不在 trx_ids 中（已提交）     → 可见 ✓
  └── db_trx_id == creator_trx_id（自己的修改）→ 可见 ✓

不可见时：沿 db_roll_ptr 找 undo log 上一个版本，重复判断
```

### 快照读 vs 当前读

| 类型 | 语句 | 机制 | 场景 |
|------|------|------|------|
| **快照读** | `SELECT`（不加锁）| MVCC，读 undo log 快照 | 普通查询，不阻塞写 |
| **当前读** | `SELECT FOR UPDATE`、`LOCK IN SHARE MODE`、`UPDATE`、`DELETE`、`INSERT` | 加锁，读最新已提交数据 | 需确保读到最新值 |

### RC vs RR：ReadView 生成时机

| 隔离级别 | ReadView 生成时机 | 效果 |
|---------|----------------|------|
| Read Committed（RC）| 每次 SELECT 都生成新 ReadView | 能看到其他事务的最新提交 → 解决脏读，但存在不可重复读 |
| Repeatable Read（RR）| 事务**首次** SELECT 时生成，整个事务复用 | 始终看同一快照 → 解决脏读 + 不可重复读 |

**幻读的解决**：RR 下快照读靠 MVCC 避免幻读；当前读（`SELECT FOR UPDATE`）靠 Next-Key Lock 避免幻读。

### MVCC 生效与不生效

| 场景 | MVCC 状态 |
|------|----------|
| RC / RR 下的普通 SELECT | 生效（快照读）|
| `SELECT FOR UPDATE` / `LOCK IN SHARE MODE` | 不生效，读最新版本 + 加锁（当前读）|
| Serializable | 不生效，全部加锁 |
| Read Uncommitted | 不生效，直接读最新数据 |

## 五、三种日志

### 三种日志对比

| 日志 | 属于 | 主要作用 | 记录内容 | 适用引擎 |
|------|------|---------|---------|---------|
| **binlog** | MySQL Server 层 | 主从复制、数据备份 | 所有 DDL/DML（逻辑操作）| 所有引擎 |
| **redo log** | InnoDB 引擎层 | 崩溃恢复（持久性）| 数据页的物理修改（"第 X 页第 Y 字节改为 Z"）| 仅 InnoDB |
| **undo log** | InnoDB 引擎层 | 事务回滚（原子性）+ MVCC 版本链 | 修改前的旧数据 | 仅 InnoDB |

记忆口诀：undo = Ctrl+Z（撤销旧值）；redo = 重做（新值重放）；binlog = 最全原始记录（备份/复制）。

### 一条 UPDATE 的完整日志写入顺序

```
UPDATE user SET name='Hollis' WHERE id=10

① 读取 id=10 行到 Buffer Pool（若不在内存则磁盘 IO）
② 写 undo log：记录旧值 name='Alice'（原子性 + 版本链）
③ 修改 Buffer Pool 中的数据页（内存中改，未落盘）
④ 写 redo log（prepare 状态）：记录"页X字节Y改为Hollis"
⑤ 写 binlog：记录 UPDATE SQL 操作
⑥ redo log 提交（commit 状态）
⑦ Buffer Pool 的脏页后台异步刷盘（非事务提交时同步）
```

### 两阶段提交（2PC）

**问题**：先写 redo log 再写 binlog——若 redo 写成但 binlog 未写就崩溃，主库恢复有新数据，备库无 → 主备不一致。

**解决**：

```
Phase 1 — Prepare：
  ① 写 redo log，标记为 prepare 状态
  ② 生成 XID（全局事务 ID）

Phase 2 — Commit：
  ③ 写 binlog（含 XID）并 fsync
  ④ redo log 标记为 commit 状态
```

**崩溃恢复规则**：

| 崩溃点 | redo log 状态 | binlog 有 XID？ | 处理 |
|-------|-------------|----------------|------|
| ④ 之后 | commit | 有 | 提交 |
| ③ 之后 ④ 之前 | prepare | 有 | 提交（binlog 已完整）|
| ③ 之前 | prepare | 无 | 回滚 |

### ACID 与日志的映射

| ACID 特性 | 实现日志 | 机制 |
|----------|---------|------|
| **原子性（A）** | undo log | 事务失败时通过 undo log 回滚所有修改 |
| **持久性（D）** | redo log | 事务提交时 redo log 已落盘，崩溃后可重放 |
| **隔离性（I）** | undo log + 锁 | undo log 版本链支持 MVCC；锁保证写隔离 |
| **一致性（C）** | 上三者共同保障 | A+I+D 做好了，C 自然满足 |

### undo log 生命周期

undo log 不会立即删除：当没有任何活跃 ReadView 引用它时，才可被 purge 线程清理。**长事务 = undo log 膨胀 + GC 停滞**。

## 六、综合对比

### MVCC vs 锁

| 维度 | MVCC | 锁 |
|------|------|-----|
| 解决的问题 | 读-写并发（快照读不阻塞写）| 写-写并发 + 当前读 |
| 并发度 | 高（读不加锁）| 低（加锁阻塞）|
| 适用语句 | 普通 SELECT | SELECT FOR UPDATE / DML |
| 实现基础 | undo log 版本链 + ReadView | Record/Gap/Next-Key Lock |

两者**互补**：快照读用 MVCC，当前读用锁；共同实现 InnoDB 的隔离性。

### RC vs RR 选型

| 维度 | RC | RR（默认）|
|------|-----|----------|
| 锁粒度 | 仅 Record Lock（行锁）| Next-Key Lock（行 + 间隙）|
| 并发度 | 高 | 低（锁范围大）|
| 死锁概率 | 低 | 高 |
| 幻读 | 存在 | 大部分解决 |
| 大厂选择 | **RC + binlog row 格式**（推荐）| MySQL 默认，适合强一致要求 |

### 四种隔离级别

| 隔离级别 | 实现方式 | 解决问题 |
|---------|---------|---------|
| Read Uncommitted | 直接读最新数据，无控制 | 无 |
| Read Committed | 每次快照读生成新 ReadView | 脏读 |
| Repeatable Read（默认）| 首次生成 ReadView 复用 + Next-Key Lock | 脏读 + 不可重复读 + 大部分幻读 |
| Serializable | 所有读加 S 锁，写加 X 锁 | 全解决，性能最差 |

### 三种日志横向对比

| 对比维度 | binlog | redo log | undo log |
|---------|--------|---------|---------|
| 所属层 | Server 层 | InnoDB | InnoDB |
| 写入时机 | 事务提交时 | 事务执行中（WAL）| 事务执行中（修改前）|
| 内容形式 | 逻辑（SQL / Row）| 物理（页偏移 + 值）| 逻辑（旧值快照）|
| 存储方式 | 追加写（不覆盖）| 循环写（固定大小）| 追加写（purge 清理）|
| 主要用途 | 主从复制、备份 | 崩溃恢复 | 回滚、MVCC |

## 七、生产风险

| 风险 | 表现 | 应对 |
|------|------|------|
| **长事务** | undo log 膨胀、GC 停滞、锁持有时间长 | 监控 `innodb_trx` 中 > 60s 的事务；强制 `autocommit=1` |
| **无索引锁退化** | `UPDATE t WHERE name='x'`（name 无索引）→ 锁全聚簇索引 ≈ 锁全表 | 所有 DML 的 WHERE 条件必须走索引；上线前 EXPLAIN 验证 |
| **死锁** | 高并发 UPDATE 互相等待，事务被回滚 | 统一资源访问顺序；缩短事务；`innodb_deadlock_detect=ON` |
| **undo log 膨胀** | 磁盘暴涨，purge 线程追不上 | 避免长事务；`innodb_purge_threads` 适当调大 |
| **刷盘策略过松** | `innodb_flush_log_at_trx_commit=0/2` + `sync_binlog=0` → 崩溃丢数据 | 生产设为双 1：`innodb_flush_log_at_trx_commit=1` + `sync_binlog=1` |
| **redo log 太小** | redo log 写满，Buffer Pool 脏页被迫强制刷盘，写性能抖动 | 根据写 TPS 调大 `innodb_log_file_size`；通常 1GB+ |
| **间隙锁阻塞 INSERT** | RR 下范围查询加 Gap Lock，其他事务 INSERT 被阻塞 | 评估是否需要 RR；业务允许可降为 RC |
| **索引失效全表扫** | 慢 SQL + 锁放大 | 开慢查询日志；定期 EXPLAIN；避免函数包裹索引列 |

## 八、与其他概念的关系

- 依赖 [[机制-B+树]]：InnoDB 索引是 B+ 树的工程实现，B+ 树特性（低树高、叶子链表）直接决定索引设计规则
- 类比 [[机制-CAS]]：MVCC 是数据库层的乐观并发控制——先操作后检查版本，而非提前加锁；与 CAS 的"比较后交换"同一思想
- 支撑 [[概念-分布式事务]]：两阶段提交（redo log + binlog 2PC）是单机事务的基础；Seata XA 模式将其扩展到分布式场景
- 指导 [[概念-SQL查询优化]]：索引失效（最左前缀、函数、类型转换）、回表、覆盖索引都是 SQL 调优的直接分析对象
- 被 [[概念-读写分离]] 依赖：binlog 是主从复制的数据通道（主库写 binlog → relay log → 从库重放）

## 九、应用边界

**InnoDB 索引**：
- 适合：B+ 树适合等值查询 + 范围查询 + 排序；自增主键适合高频写入
- 不适合：全文搜索（用 ES）；超高基数的随机字符串主键（UUID 写放大）；区分度极低的列（性别等，加了等于没加）

**锁机制**：
- 需要显式关注：批量 UPDATE/DELETE 是否走索引；`SELECT FOR UPDATE` 的范围；高并发热点行
- 不要用 Serializable：性能代价极高，RC/RR + 业务幂等通常已足够

**MVCC**：
- 生效：RC / RR 下的普通 SELECT，无锁读
- 不生效：Serializable（全加锁）、Read Uncommitted（读最新）、当前读（`FOR UPDATE`）
- 长事务是最大敌人：事务开启越早，undo log 链越长，GC 越慢

**三种日志**：
- binlog 格式选 row（而非 statement），防止非确定性 SQL 在备库重放产生不一致
- redo log 大小和双 1 刷盘是可靠性的底线配置，不要为了性能轻易调低
- 主从延迟根因通常是 binlog 积压或从库 SQL 线程单线程重放，可开启并行复制

## 十、SQL 查询优化

> 优化核心路径：发现慢 SQL → EXPLAIN 分析执行计划 → 定位瓶颈（索引失效/回表/深分页/大表/锁竞争/长事务）→ 针对性处理；根本原则是**减少磁盘 IO，减少数据扫描量**。

### 慢 SQL 发现路径

```
线上报警（RT 高）/ 慢查询日志（slow_query_log=ON, long_query_time=1s）
  → 定位具体 SQL 和表
  → EXPLAIN 分析执行计划
  → 找到瓶颈 → 针对性优化
```

### EXPLAIN 执行计划关键字段

| 字段 | 关注点 |
|------|-------|
| `type` | 访问类型，从好到差：const > eq_ref > ref > range > index > **ALL（全表扫描，必须解决）** |
| `key` | 实际使用的索引（NULL 表示未走索引）|
| `rows` | 预估扫描行数（越小越好）|
| `Extra` | `Using index`（覆盖索引）、`Using index condition`（ICP）、`Using filesort`（需优化）、`Using temporary`（需优化）|

### 常见索引失效场景

| 失效原因 | 示例 | 优化 |
|---------|------|------|
| 对索引列用函数 | `WHERE YEAR(create_time) = 2024` | 改为范围查询 |
| 类型隐式转换 | `WHERE phone = 13800000000`（phone 是 varchar）| 改为字符串比较 |
| LIKE 前导 % | `LIKE '%abc'` | 改为 `LIKE 'abc%'`，或用全文索引 |
| OR 连接无索引列 | `WHERE a=1 OR b=2`（b 无索引）| 改为 UNION ALL 或给 b 建索引 |
| 违反最左前缀 | 联合索引 (a,b)，`WHERE b=?` | 调整索引或 WHERE 条件 |
| 索引区分度低 | 优化器认为全表扫描更快 | 评估是否值得建索引 |

### 深分页优化

```sql
-- 低效：扫描 + 丢弃 1000w 行
SELECT * FROM t ORDER BY id LIMIT 10000000, 10;

-- 方案1：子查询定位主键（先在索引上定位，再回表少量行）
SELECT * FROM t WHERE id >= (
    SELECT id FROM t ORDER BY id LIMIT 10000000, 1
) LIMIT 10;

-- 方案2：游标分页（业务层记录上次最大 id）
SELECT * FROM t WHERE id > last_max_id ORDER BY id LIMIT 10;
```

### 多表 JOIN 优化

- **小表驱动大表**：让结果集小的表作为驱动表（MySQL 优化器通常自动处理，可用 STRAIGHT_JOIN 强制）
- **JOIN 字段必须有索引**：否则被驱动表每轮都可能全表扫描
- **超过 3 表 JOIN**：拆成多次查询 + 业务层组装，避免笛卡尔积风险

### 大表查询优化（千万级）

**单表 2000w 阈值的本质**：不是 B+ 树层高变了（3 层仍可覆盖），而是索引文件变大、维护成本上升、热点和归档压力增加，是工程经验阈值。

**优先级排序（先做低成本的）**：

```
1. 索引命中（最重要）
2. 覆盖索引（消除回表）
3. 联合索引（减少索引数量）
4. 避免深分页
5. 数据归档（冷热分离）
6. 分库分表（最后手段）
```

**5000w 后四位模糊查询**：`LIKE '%1234'` 不走索引，用冗余字段：

```sql
ALTER TABLE users ADD COLUMN phone_suffix CHAR(4),
  ADD INDEX idx_phone_suffix(phone_suffix);
-- 写入时同步写 phone_suffix = RIGHT(phone, 4)
SELECT * FROM users WHERE phone_suffix = '1234';
```

**扫表任务跳页问题**：

```sql
-- 错误：LIMIT offset 受数据状态变化影响，会跳过或重复
SELECT * FROM t WHERE state='INIT' LIMIT 100, 200;

-- 正确：游标分页，用主键推进
SELECT * FROM t WHERE state='INIT' AND id > ${last_max_id}
ORDER BY id LIMIT 100;
```

### 大表结构变更（不锁表方案）

| 方案 | 条件 | 原理 | 风险 |
|------|------|------|------|
| **Online DDL** | MySQL ≥ 5.6 + InnoDB | `ALGORITHM=INPLACE` 允许并发 DML | 仍占 CPU/IO |
| **pt-online-schema-change** | Percona Toolkit | 新表 + 触发器同步 + 原子改名 | 额外存储；主从延迟 |
| **gh-ost** | GitHub 开源 | 基于 binlog 同步，不用触发器 | 配置复杂；更安全 |

```sql
-- Online DDL 示例
ALTER TABLE big_table ADD COLUMN new_col VARCHAR(100),
  ALGORITHM=INPLACE, LOCK=NONE;
```

### 大表数据清理（分批按主键删除）

直接 `DELETE WHERE create_time < xxx` 的问题：可能全表扫描、锁大量行、产生大量 binlog。

```sql
-- 正确：按主键范围分批删除
DELETE FROM t
WHERE id BETWEEN #{start_id} AND #{end_id}
  AND gmt_create < DATE_SUB(NOW(), INTERVAL 300 DAY);
```

要点：每批少量删除；批次间 sleep；低峰期执行；用主键范围推进，不用 offset。

### 热点行更新

**危害**：锁竞争导致吞吐下降；连接池耗尽；死锁风险升高；主从延迟放大。

**常见解法**：

| 方案 | 原理 | 适用 |
|------|------|------|
| Redis 预扣 → DB 最终落盘 | 热点操作走内存，异步写 DB | 库存扣减、点赞计数 |
| 库存拆分（1行 → N行）| 分散热点到多行，最终合并 | 秒杀库存 |
| 队列串行 | 彻底消除并发，顺序处理 | 强一致性场景 |

### 长事务与外部调用

**长事务危害**：占用连接；长时间持行锁；undo log 膨胀；其他事务重建旧版本开销大。

**高风险写法**：

```java
@Transactional
public void process() {
    dbOperation1();
    httpCallToThirdParty();  // ← 外部调用在事务内，可能几秒甚至超时
    dbOperation2();
    mqSend();                // ← MQ 发送在事务内，消息发出但事务可能回滚
}
```

**正确写法**：事务只包必要的 DB 操作，外部调用和 MQ 发送放到事务外。

### 分布式锁与事务的位置

**原则：锁应包住事务，锁粒度大于事务粒度。**

```java
lock.lock();
try {
    transactionTemplate.execute(status -> { ... });
} finally {
    lock.unlock();
}
```

若事务包锁：事务提交前锁已释放，其他线程可能读到未提交数据，破坏互斥语义。

### 逻辑删除的唯一性约束

`UNIQUE(email, is_deleted)` 的问题：`is_deleted=1` 的记录可以有多条，重复删除后唯一约束失效。

推荐用 `del_time` 替代布尔值：

```sql
UNIQUE KEY uk_email_del (email, del_time)
-- 未删除：del_time = 0
-- 已删除：del_time = 删除时间戳（多条已删记录的时间戳各不同）
```

### 慢 SQL 原因清单

| 原因 | 解决方向 |
|------|---------|
| 索引失效 | 改 SQL / 加索引 / 调整查询条件 |
| 大量回表 | 覆盖索引 / 索引下推 |
| 深分页 | 游标分页 / 子查询定位主键 |
| 多表 JOIN | 拆查询 / 确保 JOIN 列有索引 |
| 单表数据量大 | 归档 / 分库分表 / 换存储 |
| 长事务 / 锁等待 | 缩短事务，排查锁竞争 |
| `SELECT *` | 只查必要字段 |
| 热点行 | Redis 预扣 / 行拆分 / 队列串行 |

## 十一、Canal 数据同步

> Canal 是阿里开源的 CDC 工具，通过模拟 MySQL slave 拉取 binlog，将数据库增量变更以实时流推送给下游系统（ES、缓存、异构库）。

### 工作原理

```
Canal Server
  ├── 向 MySQL Master 注册，伪装成 slave
  ├── 发送 dump 请求拉取 binlog（需 ROW 格式）
  ├── 解析 binlog 事件（INSERT/UPDATE/DELETE → 结构化变更数据）
  └── 投递给 Canal Client / MQ（Kafka/RocketMQ）→ 消费端同步到 ES/缓存/异构库
```

Canal 利用 MySQL 主从复制协议（`COM_BINLOG_DUMP`），以零侵入方式实现增量数据捕获。对比轮询（有延迟、有压力）和触发器（侵入性强），Canal 是最优解。

**前置条件**：MySQL 开启 `binlog_format = ROW`，创建有 `REPLICATION SLAVE` 权限的账号。

### 典型应用场景

| 场景 | 说明 |
|------|------|
| **MySQL → Elasticsearch** | 商品/订单变更实时同步到 ES，支持全文检索 |
| **分库分表买卖家表同步** | 按买家 ID 分表 → Canal 解析 → 按卖家 ID 写入卖家分表，实现多维度查询 |
| **缓存与 DB 一致性** | DB 变更后 Canal 通知缓存失效/更新，比业务代码手动删缓存更可靠（参见十一章一致性策略）|
| **数据迁移** | 业务双写过渡期结束后，Canal 补偿存量数据迁移 |
| **审计日志** | 捕获所有数据变更，写入审计系统，无需修改业务代码 |

### 关键权衡

1. **ROW vs STATEMENT binlog**：Canal 要求 ROW 格式；STATEMENT 只记录 SQL，含非确定性函数时无法精确还原每行变更
2. **延迟**：Canal 是近实时（毫秒级）；经过 MQ 环节后延迟叠加可达秒级
3. **消费幂等**：Canal 基于 binlog position 消费，异常重启后可能重复消费，消费端必须实现 [[概念-幂等设计|幂等]]
4. **大事务风险**：单个大事务产生大量 binlog，Canal 解析和投递会有明显延迟峰值
5. **DDL 处理**：Canal 可捕获 DDL，消费端需能处理 schema 变化

**不适合 Canal 的场景**：对延迟要求亚毫秒级；非 MySQL 数据源（用 Debezium 等）；全量初始化（Canal 只处理增量）。

## 十二、分库分表与全局ID

> 分库分表是单表数据量超过 2000 万行（或单库 QPS 达上限）时的水平拆分方案。以分片键限制、跨表查询困难、运维复杂为代价，换取线性可扩展的存储和读写能力。

**优先考虑替代方案**：归档（热冷分离）→ 读写分离 → 分布式数据库（TiDB/OceanBase）→ 最后才考虑分库分表。

### 分片键选择原则

1. **高散列性**：值分布均匀，避免数据倾斜（热点商家/用户）
2. **高关联性**：业务查询最常用的字段（如订单表用 `user_id`）
3. **不可变性**：分片键一旦确定不应修改（修改意味着跨分片迁移数据）
4. **不用自增 ID 作分片键**：全局自增 ID 在多分片中无法保证递增且分片不均匀

### 分表算法

**Hash 取模**（最常用）：`shard_index = hash(user_id) % shard_count`（shard_count 取 2^n）

先 hash 再取模：防止 user_id 本身分布不均，hash 函数（MurmurHash）打散后取模更均匀。

**基因法**（订单号携带分片信息）：

```
user_id % 64 = 10  → 分片 index
订单号 = 时间戳 + 随机数 + "10"  // 最后 2 位是分片基因

查询订单：从订单号提取分片 index，不需要 JOIN user_id
适合：买家用 user_id 分表，订单号携带分片基因，卖家维度另建卖家订单表（双写）
```

### 全局唯一 ID：雪花算法

```
64 bit = 1(符号位) + 41(时间戳ms) + 10(机器码/workerID) + 12(序列号)
                      ↑支持69年        ↑最多1024台机器         ↑同毫秒最多4096个ID
```

**时钟回拨解决方案**：
1. 抛异常，等待时钟追上（简单但影响可用性）
2. sleep 到上次时间戳后再生成（适合回拨时间很短）
3. 切换备用机器码段，避免等待
4. 预生成 ID 缓存内存（Leaf 等方案）

**workerID 分配**（防容器重启冲突）：使用 Redis `INCR worker_id`，服务启动时申请唯一 ID。

### 跨分片查询

| 查询类型 | 解决方案 |
|---------|---------|
| 按分片键精确查询 | 路由到单个分片，无问题 |
| 非分片键单条查询 | Hint 强制指定分片；或建立二级索引表（id→分片映射）|
| 列表分页（无分片键）| 查所有分片后合并（ShardingJDBC 默认）；加全局索引优化 |
| 卖家维度查询（买家分表）| 双写：买家分表 + 卖家冗余表 |

### 扩容与迁移（双写方案）

```
阶段1：开启双写（新写入同时写老分片和新分片）
阶段2：后台任务迁移存量数据（用 updated_at 兜底防漏）
阶段3：切读（切换到新分片，对比验证一致性）
阶段4：停写老分片，清理老数据
```

扩容从 64 → 128 分片时，取 2^n 的好处：一半数据原地不动，另一半迁移到新节点，迁移量减半。

### 分库分表关键权衡

1. **分库 vs 分表**：单库连接数有上限（通常 2000-4000），瓶颈是连接数/写 QPS 时分库；单表数据量大但库级 QPS 可接受时只需分表
2. **ShardingJDBC vs MyCat**：ShardingJDBC 客户端分片，无中间件单点；MyCat 服务端代理，对应用透明但多一跳延迟
3. **分片键不能是随机 ID**：UUID 完全随机，破坏 B+树局部性（页分裂频繁），推荐单调递增的雪花 ID 作主键
