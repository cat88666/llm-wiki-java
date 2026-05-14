---
type: synthesis
status: active
name: "Redis实战场景设计"
layer: L8
aliases: ["Redis排行榜", "Redis附近的人", "Redis点赞", "Redis红包", "Redis UV统计", "Redis购物车"]
tags: ["#practice"]
related:
  - "[[概念-Redis]]"
  - "[[机制-Redis持久化]]"
  - "[[机制-Redis集群与高可用]]"
  - "[[概念-缓存三大问题]]"
sources:
  - "../../raw/note/Hollis/场景题/✅如何实现百万级排行榜功能？.md"
  - "../../raw/note/Hollis/场景题/✅Redis的zset实现排行榜，实现分数相同按照时间顺序排序，怎么做？.md"
  - "../../raw/note/Hollis/场景题/✅如何实现 查找附近的人 功能？.md"
  - "../../raw/note/Hollis/场景题/✅如何用Redis实现朋友圈点赞功能？.md"
  - "../../raw/note/Hollis/场景题/✅如何实现一个抢红包功能？.md"
  - "../../raw/note/Hollis/场景题/✅如何设计一个购物车功能？.md"
created: 2026-05-07
updated: 2026-05-07
---

# Redis 实战场景设计

高频 Redis 系统设计题合集：排行榜、附近的人、点赞系统、抢红包、UV 统计。核心是根据业务特征选择合适的数据结构，同时处理高并发、大 key、热 key 等 Redis 特有的工程问题。

---

## 排行榜设计（ZSet）

### 基础方案

```
ZADD leaderboard <score> <userId>     // 添加/更新分数
ZREVRANK leaderboard <userId>          // 查询排名（从高到低）
ZREVRANGE leaderboard 0 9 WITHSCORES  // 查询前10名
```

ZSet 内部用跳表 + 哈希表，插入/删除/范围查询均为 O(log N)。

### 百万级排行榜的工程挑战

**问题 1：BigKey（成员超过 10w 就要考虑）**

解决：**数据分片**——按维度拆成多个 ZSet：
```
leaderboard:province:guangdong  // 广东榜
leaderboard:province:zhejiang   // 浙江榜
```
查全国前 10：各省取前 10 → 内存合并再排序（34 省 × 10 = 340 条数据做归并排序）

**问题 2：高并发写更新（分数实时变化）**

解决：**异步批量更新**——用户行为写 MQ，消费端异步 `ZINCRBY` 更新 ZSet；不需要每次操作立即同步。

**问题 3：分数相同时按时间顺序排名**

解决：**分数编码合并**——将时间戳和分数编码进一个 double：
```
score = (业务分数 * 1e13) + (Long.MAX_VALUE - 时间戳)
```
分数高的排前，分数相同时时间早的排前（因为 MAX_VALUE - timestamp 越小的越先）。

**问题 4：容灾与持久化**

- Redis 启用 AOF + RDB 双持久化；定期同步排行榜快照到 MySQL
- 重要周期榜（日榜/周榜）在 Redis 过期前备份到 DB，下次从 DB 重建

---

## 附近的人（GEO）

Redis 3.2+ 内置 Geospatial 数据类型，底层是 ZSet（GeoHash 编码为 score）：

```redis
# 写入用户位置
GEOADD users:location <longitude> <latitude> <userId>

# 查询指定坐标附近 1km 的用户，按距离排序
GEORADIUS users:location 116.40 39.90 1 km ASC COUNT 100

# Redis 6.2+ 推荐使用 GEOSEARCH（替代 GEORADIUS）
GEOSEARCH users:location FROMLONLAT 116.40 39.90 BYRADIUS 1 km ASC
```

**工程要点**：
- 精度：GeoHash 精度在 100m 量级内够用，超精度场景用 PostGIS
- BigKey：活跃用户太多时按区域分 key（如 `location:city:shanghai`）
- 实时性：用户位置频繁更新时用 MQ 异步写，避免写热点

---

## 朋友圈点赞（ZSet + Set）

**数据结构选择**：
- ZSet：key=`likes:{postId}`，value=userId，score=点赞时间戳（支持按时间排序、去重、取消点赞）
- Set（简化版）：key=`likes:{postId}`，value=userId（不需要时间顺序时更省内存）

**ZSet 实现操作**：
```redis
ZADD likes:post123 <timestamp> user456   // 点赞
ZREM likes:post123 user456               // 取消赞
ZCARD likes:post123                      // 点赞总数
ZREVRANGEBYSCORE likes:post123 +inf -inf // 按时间倒序查看谁赞了
```

**关键决策**：用 ZSet 而非普通 Hash/Set 的原因是**天然支持时间排序**，"张三、李四刚刚赞过"的展示逻辑不需要额外存储时间字段。

---

## 抢红包（二倍均值法 + Redis 预分配）

**核心算法：二倍均值法**
```
每次随机金额上限 = (剩余金额 / 剩余个数) × 2
随机金额 ∈ [0.01, 上限]
```
保证：所有人都有机会抢到相对公平的金额；最后一个人金额 = 剩余所有金额。

**高并发方案（与秒杀类似）**：
```
1. 红包发出时：预先计算好所有红包金额列表，存入 Redis List
   RPUSH hongbao:123 500 300 200 ...（单位：分）

2. 用户抢：原子弹出一个金额
   LPOP hongbao:123   // 返回 null 表示抢完

3. 到账记录：写入 MQ，异步落库（user_id, amount, timestamp）
```

优势：预分配列表使每次抢红包 O(1) 且原子，无超卖、无并发写冲突。

**注意**：微信红包金额是**实时计算**而非提前预分配，主要原因是提前算好浪费内存（未领取的退回），且实时算效率也够高。

---

## UV 统计（HyperLogLog）

场景：统计每天有多少**不重复的**访客（UV）

**HyperLogLog**：概率性数据结构，12KB 固定内存，允许约 0.81% 误差，支持海量 UV：
```redis
PFADD uv:2026-05-07 user123    // 记录访问
PFADD uv:2026-05-07 user456
PFCOUNT uv:2026-05-07          // 查询今日 UV（约数）
PFMERGE uv:week uv:05-01 uv:05-02 ... uv:05-07  // 合并周 UV
```

**对比 Set 存储 UV 的差异**：
| | HyperLogLog | Set |
|--|--|--|
| 内存 | 固定 12KB | O(n)，千万 UV = 数百 MB |
| 精度 | 约 0.81% 误差 | 精确 |
| 支持删除 | ❌ | ✅ |

精度要求严格（如账单统计）→ Set；可接受误差（如展示 PV/UV）→ HyperLogLog。

---

## 购物车设计

**核心选型：Redis Hash**
```
key: cart:{userId}
field: skuId
value: quantity
```

```redis
HSET cart:user123 sku:456 2       // 加入/更新购物车
HINCRBY cart:user123 sku:456 1    // 数量 +1
HDEL cart:user123 sku:456         // 删除商品
HGETALL cart:user123              // 查看全部购物车
```

**为何用 Hash 而非 String/List**：
- Hash 可以精确操作单个 sku，不需要序列化/反序列化整个购物车
- `HGETALL` O(N)，N 是购物车商品数，通常很小

**持久化策略**：
- 已登录用户：Redis Hash 作为缓存，定时同步到 DB（或重要变更时同步）
- 未登录用户：存浏览器 localStorage，登录后合并到服务端
- Redis 重启：从 DB 重建热点用户购物车（冷数据不需要）

---

## 关键权衡总结

| 场景 | 核心数据结构 | 主要挑战 |
|------|------------|---------|
| 排行榜 | ZSet | BigKey（分片）、并发写（异步）、分数相同排序（编码合并）|
| 附近的人 | GEO（底层 ZSet）| 精度、位置更新频率、大城市 BigKey |
| 点赞 | ZSet / Set | 按时间顺序选 ZSet，仅需计数选 String INCR |
| 抢红包 | List（预分配）| 公平性（二倍均值法）、高并发（原子 LPOP）|
| UV 统计 | HyperLogLog | 误差可接受则省内存，否则用 Set |
| 购物车 | Hash | 未登录合并、持久化策略 |
