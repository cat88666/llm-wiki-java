---
type: concept
status: active
name: "SQL查询优化"
layer: L5
aliases: ["慢SQL", "执行计划", "EXPLAIN", "索引失效", "深分页", "慢查询日志"]
related:
  - "[[机制-InnoDB索引]]"
  - "[[机制-MVCC]]"
  - "[[机制-InnoDB锁]]"
sources:
  - "../../../raw/note/Hollis/MySQL/✅如何进行SQL调优？.md"
  - "../../../raw/note/Hollis/MySQL/✅慢SQL的问题如何排查？.md"
  - "../../../raw/note/Hollis/MySQL/✅SQL执行计划分析的时候，要关注哪些信息？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是最左前缀匹配？为什么要遵守？.md"
  - "../../../raw/note/Hollis/MySQL/✅MySQL的深度分页如何优化.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# SQL 查询优化

> SQL 优化的核心路径：发现慢 SQL（慢查询日志/监控）→ 用 EXPLAIN 分析执行计划 → 定位瓶颈（索引失效/回表/深分页/大表/锁竞争）→ 针对性处理；根本原则是减少磁盘 IO、减少数据扫描量。

## 第一性原理

数据库的性能瓶颈本质是磁盘 IO（随机 > 顺序）和 CPU（排序/聚合）。SQL 慢的根本原因是**扫描了过多的数据行**（全表扫描或大范围索引扫描），或者**做了太多不必要的随机 IO**（大量回表）。优化的方向是：尽量让查询走高效的索引路径，扫描最少的数据。

## 核心机制

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
| `Extra` | `Using index`（覆盖索引）、`Using index condition`（索引下推）、`Using filesort`（文件排序，需优化）、`Using temporary`（临时表，需优化）|

### 常见索引失效场景

| 失效原因 | 示例 | 优化 |
|---------|------|------|
| 对索引列用函数 | `WHERE YEAR(create_time) = 2024` | 改为范围查询 `create_time >= '2024-01-01' AND < '2025-01-01'` |
| 类型隐式转换 | `WHERE phone = 13800000000`（phone 是 varchar）| 加引号 `WHERE phone = '13800000000'` |
| LIKE 前导 % | `LIKE '%abc'` | 改为 `LIKE 'abc%'`，或用全文索引 |
| OR 连接无索引列 | `WHERE a=1 OR b=2`（b 无索引）| 改为 UNION ALL 或给 b 建索引 |
| 违反最左前缀 | 联合索引 (a,b)，`WHERE b=?` | 调整索引或 WHERE 条件 |
| 索引区分度低 | 优化器认为全表扫描更快 | 评估是否值得建索引 |

### 深分页优化

```sql
-- 低效（LIMIT 10000000, 10：扫描 10000010 行，只取 10 行）
SELECT * FROM t ORDER BY id LIMIT 10000000, 10;

-- 方案1：子查询定位主键（只对索引列深分页，再回表 10 次）
SELECT * FROM t WHERE id >= (
    SELECT id FROM t ORDER BY id LIMIT 10000000, 1
) LIMIT 10;

-- 方案2：记录上一页最后 ID（游标分页，无性能衰减）
SELECT * FROM t WHERE id > last_max_id ORDER BY id LIMIT 10;
```

### 多表 JOIN 优化

- **小表驱动大表**：让结果集小的表作为驱动表，减少嵌套循环次数
- **JOIN 字段必须有索引**：被驱动表的 JOIN 条件列没索引 → 每次循环都全表扫描
- **大厂建议**：超过 3 表 JOIN 拆分为多次查询 + 业务层组装（减少锁竞争，方便扩展）

### 主要慢 SQL 原因清单

| 原因 | 解决方向 |
|------|---------|
| 索引失效 | 修改 SQL / 加索引 / force index |
| 大量回表 | 覆盖索引 / 索引下推 |
| 深分页 | 游标分页 / 子查询定位主键 |
| 多表 JOIN | 拆查询 / 保证 JOIN 字段有索引 |
| 单表数据量超大（>1000万）| 分库分表 / 数据归档 / 切 ES |
| 长事务 / 锁等待 | 缩短事务，监控锁等待 |
| 查询字段过多（SELECT *）| 只选所需字段 |

## 关键权衡

1. **FORCE INDEX 是双刃剑**：强制走某个索引在当前数据分布下可能更快，但数据变化后优化器可能本该换另一个索引，FORCE INDEX 不随之调整 → 尽量优化 SQL 而非强制索引
2. **覆盖索引 vs 索引维护成本**：加联合索引减少回表，但 INSERT/UPDATE 需同步维护更多索引树
3. **分析 filtered 值**：`EXPLAIN` 的 `filtered` 表示索引过滤后预计返回的行占比，值小说明大量行被索引筛掉（好事）；值大说明索引效果差

## 与其他概念的关系

- 直接依赖 [[机制-InnoDB索引]]：所有优化手段都围绕如何更好地利用 B+ 树索引
- 需理解 [[机制-MVCC]]：长事务导致 undo log 膨胀，影响查询性能
- 需理解 [[机制-InnoDB锁]]：锁等待是慢 SQL 的常见隐性原因，`SHOW ENGINE INNODB STATUS` 排查

## 应用边界

**SQL 调优流程**（面试标准答法）：
1. 发现慢 SQL（慢查询日志 / 监控平台）
2. EXPLAIN 分析执行计划（重点看 type / key / rows / Extra）
3. 判断是索引问题还是数据量问题
4. 索引问题 → 建索引 / 改 SQL / 覆盖索引 / 索引下推
5. 数据量问题 → 分库分表 / 归档 / 换存储

**不要的坏习惯**：
- `SELECT *`（传输多余数据 + 无法利用覆盖索引）
- `ORDER BY` 随机字段（Using filesort）
- 大事务（长时间持锁，阻塞其他查询）
