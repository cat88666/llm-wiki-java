---
type: concept
status: active
name: "InnoDB索引模型"
layer: L5
aliases: ["B+树索引", "聚簇索引", "非聚簇索引", "二级索引", "回表", "覆盖索引", "索引下推", "最左前缀", "联合索引"]
related:
  - "[[机制-B树与B加树]]"
  - "[[机制-MVCC]]"
  - "[[机制-InnoDB锁机制]]"
  - "[[概念-SQL查询优化]]"
sources:
  - "../../raw/note/📚 Hollis Java/MySQL/✅InnoDB为什么使用B+树实现索引？ 30f3673e11388195a750f5e6f87b0bb0.md"
  - "../../raw/note/📚 Hollis Java/MySQL/✅什么是聚簇索引和非聚簇索引？ 30f3673e11388163ab83fb19a3d1ee06.md"
  - "../../raw/note/📚 Hollis Java/MySQL/✅什么是回表，怎么减少回表的次数？ 30f3673e1138817cb541e388d18f0016.md"
  - "../../raw/note/📚 Hollis Java/MySQL/✅什么是索引覆盖、索引下推？ 30f3673e113881d8af69cdcf2e16946d.md"
  - "../../raw/note/📚 Hollis Java/MySQL/✅什么是最左前缀匹配？为什么要遵守？ 30f3673e1138818b981cf0a1b7d4098d.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# InnoDB 索引模型

> InnoDB 用 B+ 树实现索引——主键索引（聚簇索引）叶子节点存整行数据，二级索引叶子节点存主键值；通过二级索引查询需要先查主键再回聚簇索引（回表）；覆盖索引和索引下推是减少回表的两种核心手段。

## 第一性原理

磁盘随机 IO 代价极高，数据库查询的核心矛盾是**减少磁盘 IO 次数**。B+ 树用低树高（通常 3 层）覆盖千万级数据，且叶子节点双向链表天然支持范围扫描；非叶子节点不存数据，单个节点能存更多索引条目，进一步降低树高。InnoDB 选 B+ 树的本质：**IO 友好的有序数据结构**。

## 核心机制

### 聚簇索引 vs 二级索引

| 维度 | 聚簇索引（主键索引）| 二级索引（非聚簇索引）|
|------|----------------|-------------------|
| 叶子节点内容 | 整行数据 | 索引列值 + 主键值 |
| 数量 | 每表只有 1 个 | 可有多个 |
| 查询 | 直接得到数据 | 需要回表查聚簇索引 |
| 物理存储 | 数据按主键顺序存放 | 单独 B+ 树，不影响数据顺序 |

**无主键时**：InnoDB 选唯一非空索引作聚簇索引；都没有则用隐藏 `db_row_id` 作为隐藏主键。

### 回表

```
查询: SELECT name FROM t WHERE age = 25;（age 有二级索引）

① 扫描 age 二级索引树 → 找到 age=25 的条目 → 得到主键值 id=100
② 回聚簇索引 → 用 id=100 查整行 → 返回 name 字段

这个"第②步"就叫回表
```

**减少回表的手段**：
- **覆盖索引**：查询的所有字段都在索引中 → 不用回表
- **索引下推**（Index Condition Pushdown，ICP）：在存储引擎层先用索引过滤，减少回聚簇索引的次数

### 覆盖索引

```sql
-- 联合索引 idx_age_name(age, name)
SELECT name FROM t WHERE age = 25;
-- age 和 name 都在索引中，直接从索引返回，不回表
-- EXPLAIN extra 列显示: Using index
```

### 索引下推（MySQL 5.6+）

```sql
-- 联合索引 idx_zipcode_lastname(zipcode, lastname)
SELECT * FROM t WHERE zipcode='95054' AND lastname LIKE '%etrunia%';

-- 不用 ICP：用 zipcode 过滤后，每行都回表取 lastname 再过滤
-- 用 ICP：存储引擎先在索引层检查 lastname LIKE '%etrunia%'，不满足直接跳过，不回表
-- EXPLAIN extra 列显示: Using index condition
```

### 最左前缀匹配

联合索引 `(col1, col2, col3)` 的 B+ 树：先按 col1 排序，col1 相同按 col2，col2 相同按 col3。

**能走索引**：`WHERE col1=?`、`WHERE col1=? AND col2=?`、`WHERE col1=? AND col2=? AND col3=?`

**不能走联合索引**（不含最左列）：`WHERE col2=?`、`WHERE col2=? AND col3=?`

**注意**：`WHERE col1=? AND col3=?` 只走 col1 部分索引，col3 无法利用；`WHERE` 中条件顺序不影响结果（优化器会重排）。

### 为什么不用红黑树或 Hash 索引

| 数据结构 | 缺点 |
|---------|------|
| 红黑树 | 树高随数据量线性增长，百万数据树高可达 20+，磁盘 IO 多 |
| B 树 | 叶子节点不连接，范围查询需回溯 |
| Hash 索引 | 不支持范围查询和排序，等值查询才有优势 |

## 关键权衡

1. **主键选型**：自增整数主键 vs UUID——自增主键叶子节点顺序插入，页分裂少；UUID 随机写入导致频繁页分裂，写性能差
2. **索引不是越多越好**：索引维护成本（INSERT/UPDATE/DELETE 需同步所有索引树），适度建索引
3. **索引区分度**：区分度低的字段（如性别）建索引效果差，优化器可能选择全表扫描
4. **前缀索引**：对长字符串取前 N 字符建索引节省空间，但不能做覆盖索引（索引中没有完整值）

## 与其他概念的关系

- 依赖 [[机制-B树与B加树]]：InnoDB 索引是 B+ 树的工程实现
- 支撑 [[机制-MVCC]]：MVCC 的 ReadView 判断也需要通过聚簇索引的隐式字段（`db_trx_id`）进行
- 影响 [[机制-InnoDB锁机制]]：Record Lock 锁的是索引记录，不是数据行，无索引时退化为锁全表
- 指导 [[概念-SQL查询优化]]：索引失效、回表、最左前缀都是 SQL 调优的直接分析对象

## 应用边界

**建索引的场景**：WHERE 条件频繁出现的列；ORDER BY / GROUP BY 列；JOIN 的连接列；区分度高（>95%）的列。

**不建索引 / 索引失效场景**：
- `WHERE` 列使用函数或类型转换（如 `WHERE year(date) = 2024`）
- LIKE 前导 `%`（`LIKE '%abc'` 走不了索引）
- OR 连接且其中一个列无索引
- 数据量极少的表（全表扫描更快）
