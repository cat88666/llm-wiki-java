---
type: synthesis
status: active
name: "MySQL大表与查询优化"
tags: ["#storage", "#practice"]
related:
  - "[[机制-InnoDB索引模型]]"
  - "[[机制-InnoDB锁机制]]"
  - "[[概念-SQL查询优化]]"
  - "[[机制-MySQL三种日志]]"
sources:
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
  - "../../../raw/note/Hollis/场景题/✅InnoDB为什么不用跳表，Redis为什么不用B+树？.md"
  - "../../../raw/note/Hollis/场景题/✅数据库逻辑删除后，怎么做唯一性约束？.md"
created: 2026-05-08
updated: 2026-05-08
---

# MySQL大表与查询优化

> 千万级以上单表的核心挑战：索引设计 → 避免慢查询；大表结构变更 → 避免锁表；热点行更新 → 避免锁竞争；事务粒度 → 避免长事务。

---

## 千万级查询优化：先索引，再归档

**单表 2000w 以内（甚至 5000w），只要索引用对，性能无需分库分表。**

**为什么单表 2000w 需要考虑分表？（B+树视角）**
- InnoDB 页大小 16KB，每页能存约 1000 个索引条目
- 树高 = 3 时：`1000 × 1000 × 1000 = 10 亿` 条数据
- 但每次索引查询要经历 3 次磁盘 IO，而 InnoDB 的根节点/枝节点常驻内存
- **实际上树高 3 可支撑远超 2000w，2000w 的阈值来自填充率约 2/3 时叶节点数量的经验值**
- 超过 2000w 后，更多的主要是索引维护成本和索引文件体积增长，而非树高变化

**优先级排序**：
```
1. 索引命中（EXPLAIN 验证 key、rows、Extra）
2. 索引覆盖（避免回表）
3. 联合索引（减少二次查询）
4. 避免深分页
5. 数据归档（冷热分离）
6. 分库分表（最后手段）
```

### 深分页优化

```sql
-- 低效：全表扫描 OFFSET 条数据
SELECT * FROM orders LIMIT 10000000, 20;

-- 方案1：游标分页（记录上一页最大 ID）
SELECT * FROM orders WHERE id > ${last_max_id} ORDER BY id LIMIT 20;

-- 方案2：子查询覆盖索引（只用索引找到 id，再回表）
SELECT * FROM orders WHERE id IN (
  SELECT id FROM orders ORDER BY id LIMIT 10000000, 20
);

-- 方案3：倒序查询（查尾部数据时减少扫描量）
SELECT * FROM orders ORDER BY id DESC LIMIT 20;
```

### 5000w 后四位模糊查询：冗余字段分段存储

`LIKE '%1234'` 不走索引（前缀不固定）。解法：

```sql
-- 冗余 phone_part3 字段，存后4位，加普通索引
ALTER TABLE users ADD COLUMN phone_part3 CHAR(4), ADD INDEX idx_phone_part3(phone_part3);

-- 查询改为等值查询，命中索引
SELECT * FROM users WHERE phone_part3 = '1234';
```

### 扫表任务跳页问题

定时扫 INIT 状态数据，处理后状态变 SUCCESS，下一页查询 INIT 数据时会跳过第二页：

```sql
-- 错误方式：页码随处理进展失效
SELECT * FROM table WHERE state='INIT' LIMIT 100, 200;

-- 正确方式：游标分页，状态变化不影响下一批
SELECT * FROM table WHERE state='INIT' AND id > ${last_max_id}
ORDER BY id LIMIT 100;
```

---

## 大表结构变更：不锁表的三种方案

**场景**：千万级大表 ALTER TABLE 增加字段，直接执行可能锁表数十分钟甚至更长。

| 方案 | 条件 | 原理 | 风险 |
|------|------|------|------|
| **Online DDL** | MySQL ≥5.6 + InnoDB | `ALGORITHM=INPLACE` 允许并发 DML | 仍占 CPU/IO，避开高峰期执行 |
| **pt-online-schema-change** | Percona Toolkit | 创建新表 → 触发器同步 → 改名 | 额外存储；主从同步有延迟 |
| **gh-ost** (GitHub) | GitHub 开源 | 基于 binlog 同步，不用触发器 | 配置复杂 |

```sql
-- Online DDL 语法
ALTER TABLE big_table ADD COLUMN new_col VARCHAR(100),
  ALGORITHM=INPLACE, LOCK=NONE;
```

---

## 大表数据清理：分批按主键删除

**直接 DELETE WHERE create_time < xxx 的问题**：
- 全表扫描 → 锁定大量行 → 业务写阻塞
- 单语句产生大量 binlog → 超过 `max_binlog_cache_size` → 报错回滚
- 大量索引更新 → 频繁 IO

**正确方案（阿里云 DMS 同款）**：
```sql
-- 1. 获取范围: min_id=100, max_id=10000000
-- 2. 每次按主键分批删除，默认每批 1000 条
DELETE FROM table_hollis
WHERE id BETWEEN #{start_id} AND #{end_id}
  AND gmt_create < DATE_SUB(NOW(), INTERVAL 300 DAY);

-- 3. 循环，每批间加 sleep(10ms) 让出资源
-- 4. 配置只在业务低峰期执行（如 02:00-06:00）
```

---

## 热点行更新：六大危害与解法

**典型场景**：大促秒杀库存扣减，同一行被高并发更新。

**六大危害**：
1. **锁竞争** → 大量 UPDATE 互斥，吞吐量骤降
2. **连接耗尽** → 等锁的 SQL 持有连接，数据库连接池打满
3. **CPU 耗尽** → 大量自旋 + 死锁检测 + 线程上下文切换
4. **死锁风险** → 高并发下多行锁顺序不一致
5. **索引维护开销** → 频繁更新导致索引页分裂/合并
6. **主从延迟放大** → 热点更新的 binlog 同步压力更大

**解法**（按成本从低到高）：
```
Redis 预扣 → DB 最终落盘（高频扣，低频同步）
库存拆分  → 1 行拆 N 行，压力分散（子库存 + 汇总）
队列串行  → 单线程消费，彻底消除并发（吞吐受限）
```

---

## 长事务危害与规避

**危害**：
1. 占用数据库连接（连接池被耗尽）
2. 持有行锁（阻塞其他 DML）
3. undo log 膨胀（MVCC 旧版本无法清理）
4. 其他事务重建旧数据开销更大

**高风险写法**：
```java
@Transactional  // 声明式事务粒度 = 整个方法
public void process() {
    dbOperation1();       // 数据库操作
    httpCallToThirdParty(); // 外部调用——拖长事务！
    dbOperation2();
    mqSend();             // MQ 发送——外部调用！
}
```

**建议写法**：编程式事务 + 事务外执行外部调用
```java
public void process() {
    // 事务外预查
    Data data = queryBeforeTransaction();
    
    // 缩小事务范围，只包含 DB 操作
    transactionTemplate.execute(status -> {
        dbOperation1(data);
        dbOperation2(data);
        return null;
    });
    
    // 事务提交后，再做外部调用
    httpCallToThirdParty();
    mqSend();
}
```

**在事务中做外部调用的两大危险**：
- 外部调用失败导致事务回滚（MQ 失败让订单回滚，不合理）
- 外部调用超时但实际成功 → 本地回滚但远程未回滚 → **数据不一致**

---

## 分布式锁与事务的位置关系

**结论**：**锁应包住事务（锁粒度 > 事务粒度）**。

```java
// 推荐：锁包事务
lock.lock();
try {
    transactionTemplate.execute(status -> { ... });
} finally {
    lock.unlock();
}

// 不推荐：事务包锁
@Transactional
public void method() {
    lock.lock();           // Redis 锁 → 外部调用 → 拖长事务
    try { ... }
    finally { lock.unlock(); }
    // 事务提交前锁已释放 → 其他线程可能读到未提交数据
}
```

**事务包锁的核心问题**：锁释放 → 事务未提交 → 其他线程读到旧数据 → 锁的互斥语义被破坏。

---

## 逻辑删除唯一性约束

直接用 `is_deleted` (0/1) 后，`UNIQUE(email, is_deleted)` 失效（多条 deleted=0 的记录会冲突）。

**解法**：用 `del_time` 替代 `is_deleted`：
```sql
-- del_time = 0 表示未删除；del_time = 删除时间戳
UNIQUE KEY uk_email_del (email, del_time)
-- 未删除的 del_time=0，唯一约束生效
-- 已删除的 del_time 各不同，不冲突
```

---

## InnoDB 为何不用跳表

| | B+树（InnoDB）| 跳表（Redis ZSet）|
|-|---|---|
| **存储介质** | 磁盘 | 内存 |
| **局部性** | 叶节点相邻，范围查询顺序读磁盘 | 指针跳跃，内存操作无IO代价 |
| **树高** | 极低（3层抗亿级数据）| 跳表层数多，但都是内存操作 |
| **调整代价** | 分裂/合并页（代价高，需最小化） | 内存修改（代价低）|

Redis 不用 B+树：纯内存操作不需要减少 IO 次数，B+树节点管理开销反而更大；跳表实现简单，并发友好。
