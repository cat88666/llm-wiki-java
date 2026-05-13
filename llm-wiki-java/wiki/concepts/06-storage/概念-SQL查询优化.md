---
type: concept
status: active
name: "SQL查询优化"
layer: L5
aliases: ["慢SQL", "执行计划", "EXPLAIN", "索引失效", "深分页", "慢查询日志", "MySQL大表与查询优化"]
related:
  - "[[机制-InnoDB索引]]"
  - "[[机制-MVCC]]"
  - "[[机制-InnoDB锁]]"
  - "[[机制-MySQL三种日志]]"
sources:
  - "../../../raw/note/Hollis/MySQL/✅如何进行SQL调优？.md"
  - "../../../raw/note/Hollis/MySQL/✅慢SQL的问题如何排查？.md"
  - "../../../raw/note/Hollis/MySQL/✅SQL执行计划分析的时候，要关注哪些信息？.md"
  - "../../../raw/note/Hollis/MySQL/✅什么是最左前缀匹配？为什么要遵守？.md"
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
created: 2026-05-02
updated: 2026-05-13
lint_notes: ""
---

# SQL 查询优化

> SQL 优化的核心路径：发现慢 SQL（慢查询日志/监控）→ 用 EXPLAIN 分析执行计划 → 定位瓶颈（索引失效/回表/深分页/大表/锁竞争/长事务）→ 针对性处理；根本原则是减少磁盘 IO、减少数据扫描量。

## 第一性原理

数据库的性能瓶颈本质是磁盘 IO（随机 > 顺序）和 CPU（排序/聚合）。SQL 慢的根本原因是**扫描了过多的数据行**（全表扫描或大范围索引扫描），或者**做了太多不必要的随机 IO**（大量回表）。优化的方向是：尽量让查询走高效的索引路径，扫描最少的数据。

---

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
| 对索引列用函数 | `WHERE YEAR(create_time) = 2024` | 改为范围查询 |
| 类型隐式转换 | `WHERE phone = 13800000000`（phone 是 varchar）| 改为字符串比较 |
| LIKE 前导 % | `LIKE '%abc'` | 改为 `LIKE 'abc%'`，或用全文索引 |
| OR 连接无索引列 | `WHERE a=1 OR b=2`（b 无索引）| 改为 UNION ALL 或给 b 建索引 |
| 违反最左前缀 | 联合索引 (a,b)，`WHERE b=?` | 调整索引或 WHERE 条件 |
| 索引区分度低 | 优化器认为全表扫描更快 | 评估是否值得建索引 |

### 深分页优化

```sql
-- 低效
SELECT * FROM t ORDER BY id LIMIT 10000000, 10;

-- 方案1：子查询定位主键
SELECT * FROM t WHERE id >= (
    SELECT id FROM t ORDER BY id LIMIT 10000000, 1
) LIMIT 10;

-- 方案2：游标分页
SELECT * FROM t WHERE id > last_max_id ORDER BY id LIMIT 10;
```

### 多表 JOIN 优化

- **小表驱动大表**：让结果集小的表作为驱动表
- **JOIN 字段必须有索引**：否则被驱动表每轮都可能全表扫描
- **大厂建议**：超过 3 表 JOIN 拆成多次查询 + 业务层组装

---

## 大表场景优化

### 千万级查询优化：先索引，再归档

**单表 2000w 以内甚至更高，索引用对时通常无需第一时间分库分表。**

**为什么单表 2000w 常被当成阈值**：
- InnoDB 页大小 16KB，每页能存约 1000 个索引条目
- 理论上 B+ 树 3 层可支撑极大量数据
- 2000w 更像工程经验阈值：索引文件变大、维护成本上升、热点和归档压力增加

**优先级排序**：
```
1. 索引命中
2. 覆盖索引
3. 联合索引
4. 避免深分页
5. 数据归档（冷热分离）
6. 分库分表（最后手段）
```

### 5000w 后四位模糊查询：冗余字段分段存储

`LIKE '%1234'` 不走索引，常见解法是冗余后缀字段：

```sql
ALTER TABLE users ADD COLUMN phone_part3 CHAR(4), ADD INDEX idx_phone_part3(phone_part3);
SELECT * FROM users WHERE phone_part3 = '1234';
```

### 扫表任务跳页问题

```sql
-- 错误：页码受数据状态变化影响
SELECT * FROM table WHERE state='INIT' LIMIT 100, 200;

-- 正确：游标分页
SELECT * FROM table WHERE state='INIT' AND id > ${last_max_id}
ORDER BY id LIMIT 100;
```

---

## 大表结构变更

### 不锁表的三种方案

| 方案 | 条件 | 原理 | 风险 |
|------|------|------|------|
| **Online DDL** | MySQL ≥5.6 + InnoDB | `ALGORITHM=INPLACE` 允许并发 DML | 仍占 CPU/IO |
| **pt-online-schema-change** | Percona Toolkit | 新表 + 触发器同步 + 原子改名 | 额外存储；主从延迟 |
| **gh-ost** | GitHub 开源 | 基于 binlog 同步，不用触发器 | 配置复杂 |

```sql
ALTER TABLE big_table ADD COLUMN new_col VARCHAR(100),
  ALGORITHM=INPLACE, LOCK=NONE;
```

---

## 大表数据清理

### 分批按主键删除

直接 `DELETE WHERE create_time < xxx` 的问题：
- 可能全表扫描
- 锁住大量行
- 产生大量 binlog
- 索引更新开销大

**正确方式**：

```sql
DELETE FROM table_hollis
WHERE id BETWEEN #{start_id} AND #{end_id}
  AND gmt_create < DATE_SUB(NOW(), INTERVAL 300 DAY);
```

实践要点：
- 每批删除小数量
- 批次间 sleep
- 只在低峰期执行
- 用主键范围推进，不用 offset

---

## 热点行与长事务

### 热点行更新的危害

1. 锁竞争，吞吐下降
2. 连接池耗尽
3. CPU 被锁等待、自旋、死锁检测消耗
4. 死锁风险升高
5. 索引页维护压力变大
6. 主从延迟放大

**常见解法**：
```
Redis 预扣 -> DB 最终落盘
库存拆分 -> 1 行拆 N 行
队列串行 -> 彻底消除并发
```

### 长事务危害

1. 占用数据库连接
2. 长时间持有行锁
3. undo log 膨胀
4. 其他事务重建旧版本开销增大

**高风险写法**：

```java
@Transactional
public void process() {
    dbOperation1();
    httpCallToThirdParty();
    dbOperation2();
    mqSend();
}
```

**建议写法**：事务只包住必要的 DB 操作，外部调用放到事务外。

### 分布式锁与事务的位置

**原则**：锁应包住事务，锁粒度大于事务粒度。

```java
lock.lock();
try {
    transactionTemplate.execute(status -> { ... });
} finally {
    lock.unlock();
}
```

如果事务包锁，可能出现锁提前释放但事务未提交，破坏互斥语义。

---

## 特殊场景技巧

### 逻辑删除唯一性约束

`UNIQUE(email, is_deleted)` 往往不理想，推荐用 `del_time`：

```sql
UNIQUE KEY uk_email_del (email, del_time)
```

- 未删除：`del_time = 0`
- 已删除：`del_time = 删除时间戳`

### 主要慢 SQL 原因清单

| 原因 | 解决方向 |
|------|---------|
| 索引失效 | 改 SQL / 加索引 / 调整查询条件 |
| 大量回表 | 覆盖索引 / 索引下推 |
| 深分页 | 游标分页 / 子查询定位主键 |
| 多表 JOIN | 拆查询 / 确保 JOIN 列有索引 |
| 单表数据量大 | 归档 / 分库分表 / 换存储 |
| 长事务 / 锁等待 | 缩短事务，排查锁竞争 |
| `SELECT *` | 只查必要字段 |

---

## 关键权衡

1. **FORCE INDEX 是双刃剑**：当前快，不代表以后也快
2. **覆盖索引 vs 写入成本**：读更快，但索引维护成本更高
3. **分库分表不是第一解**：先把索引、分页、归档、锁问题解决
4. **DDL 方案选型**：Online DDL 简单但有限制；pt-osc/gh-ost 更强但更复杂

---

## 与其他概念的关系

- 直接依赖 [[机制-InnoDB索引]]：所有优化手段都围绕 B+ 树索引
- 需理解 [[机制-MVCC]]：长事务和 undo log 直接影响查询性能
- 需理解 [[机制-InnoDB锁]]：锁等待是慢 SQL 的常见隐性原因
- 需理解 [[机制-MySQL三种日志]]：大删除、大事务、大 DDL 会放大日志成本

---

## 应用边界

**SQL 调优流程**（面试标准答法）：
1. 发现慢 SQL（慢查询日志 / 监控平台）
2. EXPLAIN 分析执行计划
3. 判断是索引问题还是数据量问题
4. 索引问题：建索引 / 改 SQL / 覆盖索引 / 索引下推
5. 数据量问题：归档 / 分库分表 / 换存储

**不要的坏习惯**：
- `SELECT *`
- 大 offset 深分页
- 在事务里做慢外部调用
- 大批量单语句删除
