---
type: concept
status: active
name: "InnoDB锁机制"
layer: L5
aliases: ["共享锁", "排他锁", "意向锁", "记录锁", "间隙锁", "临键锁", "Next-Key Lock", "Gap Lock", "死锁"]
related:
  - "[[机制-MVCC]]"
  - "[[机制-InnoDB索引]]"
  - "[[机制-MySQL三种日志]]"
sources:
  - "../../../raw/note/Hollis/MySQL/✅介绍下InnoDB的锁机制？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是意向锁？.md"
  - "../../../raw/note/Hollis/MySQL/✅MySQL的行级锁锁的到底是什么？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# InnoDB 锁机制

> InnoDB 的锁体系分两个维度：按级别分为共享锁（S）和排他锁（X）；按粒度分为表级的意向锁（IS/IX）+ 行级的记录锁/间隙锁/临键锁；行级锁锁的是**索引记录**而非数据行，RR 隔离级别下以 Next-Key Lock 为加锁基本单位防止幻读。

## 第一性原理

并发写操作会产生数据竞争（写-写并发）；当前读（`SELECT FOR UPDATE`）需要读到最新数据且阻止并发修改（读-写并发）。纯 MVCC 只解决快照读的并发，写冲突和当前读必须用锁。InnoDB 锁的存在理由：**在不同粒度上以最小代价阻止有冲突的并发操作，同时尽量不阻塞无关并发**。

## 核心机制

### 锁的两个基本类型（级别）

| 锁类型 | 符号 | 兼容关系 | 语义 |
|-------|------|---------|------|
| **共享锁（读锁）** | S | S 与 S 兼容；S 与 X 不兼容 | 可并发读，阻止写 |
| **排他锁（写锁）** | X | X 与任何锁不兼容 | 独占读写，阻止一切 |

显式加 S 锁：`SELECT ... LOCK IN SHARE MODE`  
显式加 X 锁：`SELECT ... FOR UPDATE`  
DML（UPDATE/DELETE/INSERT）自动加 X 锁

### 意向锁（表级锁）

**问题**：事务 A 对某行加了行级 X 锁；事务 B 想加表级 X 锁，需要检查表中每行是否有行锁 → 逐行扫描开销大。

**解决**：意向锁 —— 加行锁前，先在表级别加意向锁，让表锁申请时 O(1) 判断冲突。

| 意向锁 | 含义 | 加行 S 锁前 | 加行 X 锁前 |
|-------|------|------------|------------|
| IS（意向共享锁）| 表内有行要加 S 锁 | 自动加 IS | — |
| IX（意向排他锁）| 表内有行要加 X 锁 | — | 自动加 IX |

**意向锁是 MySQL 自动管理的，开发者不需要显式操作。**

### 行级锁三种形态

#### 1. Record Lock（记录锁）

锁定具体的一条**索引记录**（不是数据行，锁的是 B+ 树节点上的 key）。

```sql
SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;
-- 对 c1=10 的索引记录加 X 记录锁
```

**无索引时**：退化为锁聚簇索引，实际效果接近锁全表。

#### 2. Gap Lock（间隙锁）

锁定索引记录**之间的间隙**（不锁记录本身），防止其他事务在间隙中插入新记录。

```
索引值: ... 5, 10, 15, ...
Gap Lock (5, 10)：阻止任何事务在此间隙插入值（如 6, 7, 8, 9）
```

**只在 RR 隔离级别生效**（RC 下没有 Gap Lock，幻读问题由业务自己处理）。

#### 3. Next-Key Lock（临键锁）

= Record Lock + 左侧 Gap Lock，范围**左开右闭**：`(prev_key, current_key]`

RR 下 InnoDB 的**加锁基本单位**，防止幻读。

**加锁优化规则**（丁奇《MySQL实战45讲》）：
- 唯一索引等值查询命中时：Next-Key Lock 退化为 Record Lock（不锁间隙）
- 等值查询向右遍历且最后一个值不满足条件时：Next-Key Lock 退化为 Gap Lock

### 锁的粒度层次

```
全局锁（FLUSH TABLES WITH READ LOCK）
  └── 表级锁
        ├── 意向锁（IS / IX）：自动管理，配合行锁使用
        ├── MDL 锁（元数据锁）：DDL 语句自动加，保护表结构
        └── AUTO-INC 锁：自增主键插入时临时持有
  └── 行级锁
        ├── Record Lock：锁索引记录
        ├── Gap Lock：锁间隙
        └── Next-Key Lock：Record + Gap（RR 默认）
```

### 死锁与解决

**死锁条件**：两个事务互相持有对方需要的锁。

```
事务A: 持有 id=1 的 X 锁，等待 id=2 的 X 锁
事务B: 持有 id=2 的 X 锁，等待 id=1 的 X 锁
→ 死锁
```

**InnoDB 自动检测死锁**（等待图算法），选代价最小的事务回滚（victim）。

**避免死锁**：
- 所有事务按相同顺序访问资源
- 缩短事务，减少持锁时间
- 避免大范围锁（尽量用精确的 WHERE 条件）

## 关键权衡

1. **RR vs RC 的锁代价**：RR 用 Next-Key Lock 防幻读，但锁范围大 → 并发度低，死锁概率高；RC 只用 Record Lock，并发度高，但需接受幻读（大厂通常用 RC + binlog row 格式）
2. **无索引的锁退化**：`UPDATE t WHERE name='Alice'` 若 name 无索引，会锁全聚簇索引（全表），严重影响并发
3. **间隙锁的开销**：间隙锁防止幻读但会阻塞其他事务的 INSERT，OLTP 高并发场景需权衡

## 与其他概念的关系

- 与 [[机制-MVCC]] 互补：MVCC 处理读-写并发（快照读不加锁）；锁机制处理写-写并发和当前读
- 依赖 [[机制-InnoDB索引]]：Record Lock 锁的是索引记录，无索引会锁聚簇索引（等同全表扫描）
- 依赖 [[机制-MySQL三种日志]]：锁信息的获取/释放记录在 redo log 中，保证崩溃后锁状态恢复

## 应用边界

**需要显式关注锁的场景**：
- `SELECT FOR UPDATE` / `LOCK IN SHARE MODE`：明确需要锁保护的临界区
- 批量 UPDATE/DELETE：注意是否走索引，避免锁全表
- 高并发 INSERT 热点：自增主键竞争 AUTO-INC 锁，可调整 `innodb_autoinc_lock_mode`

**排查锁问题**：
- `information_schema.innodb_trx`：查看活跃事务
- `information_schema.innodb_lock_waits`：查看锁等待
- `SHOW ENGINE INNODB STATUS`：查看死锁信息
