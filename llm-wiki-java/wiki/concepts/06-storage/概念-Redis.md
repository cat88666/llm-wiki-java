---
type: concept
status: active
name: "Redis"
layer: L5
tags: ["#storage"]
aliases: ["Redis", "Redis数据结构", "SDS", "ZipList", "ListPack", "SkipList", "跳表", "Redis快的原因", "RDB", "AOF", "混合持久化", "BGSAVE", "写回策略", "AOF重写", "SETNX", "Redisson", "watchdog", "续期", "可重入锁", "分布式锁", "Redis主从", "哨兵模式", "Redis Cluster", "Sentinel", "脑裂", "槽位", "16384 slots", "Redis排行榜", "Redis附近的人", "Redis点赞", "Redis红包", "Redis UV统计", "Redis购物车", "HyperLogLog", "GEO", "缓存穿透", "缓存击穿", "缓存雪崩", "布隆过滤器", "内存淘汰策略", "缓存预热", "缓存一致性"]
related:
  - "[[机制-B+树]]"
  - "[[机制-CAS]]"
  - "[[机制-AQS]]"
  - "[[概念-MySQL]]"
  - "[[概念-幂等设计]]"
sources:
  - "../../../raw/note/Hollis/Redis/✅Redis 支持哪几种数据类型？.md"
  - "../../../raw/note/Hollis/Redis/✅Redis中的Zset是怎么实现的？.md"
  - "../../../raw/note/Hollis/Redis/✅Redis为什么这么快？.md"
  - "../../../raw/note/Hollis/Redis/✅Redis的持久化机制是怎样的？.md"
  - "../../../raw/note/Hollis/Redis/✅RDB和AOF的写回策略分别是什么？.md"
  - "../../../raw/note/Hollis/Redis/✅如何用SETNX实现分布式锁？.md"
  - "../../../raw/note/Hollis/Redis/✅Redisson的watchdog机制是怎么样的？.md"
  - "../../../raw/note/Hollis/Redis/✅介绍一下Redis的集群模式？.md"
  - "../../../raw/note/Hollis/场景题/✅如何实现百万级排行榜功能？.md"
  - "../../../raw/note/Hollis/场景题/✅Redis的zset实现排行榜，实现分数相同按照时间顺序排序，怎么做？.md"
  - "../../../raw/note/Hollis/场景题/✅如何实现 查找附近的人 功能？.md"
  - "../../../raw/note/Hollis/场景题/✅如何用Redis实现朋友圈点赞功能？.md"
  - "../../../raw/note/Hollis/场景题/✅如何实现一个抢红包功能？.md"
  - "../../../raw/note/Hollis/场景题/✅如何设计一个购物车功能？.md"
  - "../../../raw/note/Hollis/Redis/✅什么是缓存击穿、缓存穿透、缓存雪崩？.md"
  - "../../../raw/note/Hollis/Redis/✅Redis的内存淘汰策略是怎么样的？.md"
  - "../../../raw/note/Hollis/Redis/✅如何解决Redis和数据库的一致性问题？.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# Redis

> Redis 是基于内存的高性能 KV 存储：对外暴露 5 种数据类型（动态切换底层编码以平衡内存与性能），以持久化（RDB/AOF）保障数据安全，以分布式锁（Redisson）提供跨进程互斥，以主从/哨兵/Cluster 实现高可用与水平扩展。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 内存 KV 的核心矛盾、快的根本原因 |
| [二、数据类型与底层编码](#二数据类型与底层编码) | 5 种类型、SDS、二态编码、ZSet 双结构 |
| [三、持久化机制](#三持久化机制) | RDB、AOF 三种写回策略、混合持久化 |
| [四、分布式锁](#四分布式锁) | SET NX EX、Lua 解锁、Redisson watchdog、Hash 可重入、RedLock |
| [五、集群与高可用](#五集群与高可用) | 主从复制、哨兵模式、Cluster 分片、脑裂 |
| [六、综合对比](#六综合对比) | RDB vs AOF、哨兵 vs Cluster、Redis 锁 vs ZK 锁 |
| [七、生产风险](#七生产风险) | 大 Key、热 Key、持久化配置、脑裂、锁失效 |
| [八、与其他概念的关系](#八与其他概念的关系) | MySQL 日志、B+树、CAS、AQS |
| [九、应用边界](#九应用边界) | 类型选型、各机制适用边界、常见误用 |
| [十一、缓存三大问题](#十一缓存三大问题) | 穿透/击穿/雪崩解法、一致性策略、内存淘汰策略 |

## 一、第一性原理

Redis 的核心矛盾：**内存宝贵，但访问必须快**。

| 问题 | 解法 | 机制 |
|------|------|------|
| 内存碎片 + 指针开销大 | 小数据用紧凑连续结构 | ZipList / ListPack |
| 数据量大时紧凑结构太慢 | 超阈值切换高性能结构 | SkipList / Dict / QuickList |
| 进程崩溃数据丢失 | 定期快照 + 追加日志 | RDB + AOF |
| 多进程并发写共享资源 | 基于共享状态实现互斥 | SET NX EX + Redisson |
| 单点故障 + 容量上限 | 冗余 + 分片 | 主从 / 哨兵 / Cluster |

**Redis 快的 5 个根本原因**：

1. **内存存储**：ns 级访问 vs 磁盘 ms 级，数量级差距
2. **单线程处理命令**：消除锁竞争，避免线程切换（6.0+ 网络 IO 多线程，命令执行仍单线程）
3. **IO 多路复用**：epoll 事件驱动，单线程处理大量并发连接
4. **高效数据结构**：为缓存场景定制（SDS O(1) 获取长度 / Dict O(1) 查找 / SkipList O(log n) 有序）
5. **二态编码**：小数据紧凑结构减少内存分配和 GC 压力

## 二、数据类型与底层编码

### 5 种基本数据类型

| 类型 | 典型用途 | 底层编码（小 → 大）|
|------|---------|----------------|
| **String** | 计数器、缓存对象、分布式锁 | int（整数）/ embstr（≤44 字节）/ raw（SDS）|
| **List** | 消息队列、最新 N 条 | ListPack（小）→ QuickList（大）|
| **Hash** | 对象字段存储 | ListPack（小）→ Dict 哈希表（大）|
| **Set** | 去重、共同好友、标签 | ListPack（小）→ Dict（大）|
| **ZSet** | 排行榜、带权重队列 | ListPack（小）→ SkipList + Dict（大）|

> 切换阈值（默认）：元素数 ≤ 128 且单个元素 ≤ 64 字节 → 紧凑结构；超出 → 切换高性能结构

### SDS（Simple Dynamic String）

Redis 字符串底层不是 C 字符串，而是 SDS：

```c
struct sdshdr {
    int len;    // 已用长度，O(1) 获取（C 字符串需 O(n) 遍历）
    int free;   // 剩余空间，支持预分配，减少 realloc 次数
    char buf[]; // 实际数据，兼容 C 函数
};
```

SDS 优势：O(1) 获取长度 / 预分配减少 realloc / 二进制安全（可存 `\0` 字节）。

### ZSet 双结构（二态编码最典型案例）

**小数据（ListPack，7.0+ 替代 ZipList）**：

```
连续内存块: [element1][score1][element2][score2]...
优点: 节省内存（无指针）
缺点: 查找 O(n)
```

**大数据（SkipList + Dict）**：

```
SkipList（跳表）：
  Level 4: [head] ─────────────────────── [tail]
  Level 3: [head] ──────── [B:50] ──────── [tail]
  Level 2: [head] ── [A:20] ── [B:50] ──── [tail]
  Level 1: [head] ── [A:20] ── [B:50] ── [C:80]

Dict: {element → score} 映射，O(1) 按元素查分数
```

**为什么 ZSet 用跳表而非 B+ 树**：跳表实现更简单，范围查询（`ZRANGEBYSCORE`）性能与 B+ 树相当，且 Redis 是内存操作，磁盘 IO 不是瓶颈，B+ 树的 IO 友好特性无优势。

## 三、持久化机制

### RDB（全量快照）

将某一时刻内存的完整数据以二进制压缩格式存为 `.rdb` 文件。

**触发方式**：

| 方式 | 说明 |
|------|------|
| `BGSAVE` | `fork()` 子进程写快照，父进程继续处理命令（**主要方式**）|
| `SAVE` | 阻塞主线程，**生产禁用** |
| 配置触发 | `save 900 1`：900 秒内至少 1 次写操作则触发 BGSAVE |
| 主从全量同步 | 新从节点接入时自动触发 |

**fork + COW（Copy-On-Write）原理**：

```
父进程 fork → 子进程获得相同页表（共享物理内存，页面只读标记）
父进程收到写请求 → OS 触发 COW，复制该页给父进程修改
子进程继续读原始数据 → 生成一致性快照
```

**优点**：文件紧凑（二进制压缩），恢复速度快；fork 后主进程不阻塞。

**缺点**：最多丢失上次快照到故障点之间的数据（分钟级）；大内存实例 fork 慢，可能瞬间卡顿。

### AOF（追加写命令日志）

每次写操作后将命令以文本格式追加到 `.aof` 文件，恢复时重放所有命令。

**三种写回策略（`appendfsync` 配置）**：

| 策略 | 触发时机 | 数据安全 | 性能 | 适用场景 |
|------|---------|---------|------|---------|
| **Always** | 每条命令写后立即 fsync | 最高（最多丢 1 条）| 最低 | 金融，不能丢数据 |
| **Everysec**（默认）| 每秒 fsync 一次 | 高（最多丢 1 秒）| 中 | 大多数场景 |
| **No** | 由 OS 决定 fsync 时机 | 低 | 最高 | 对数据丢失不敏感 |

**AOF 重写**（防止文件无限膨胀）：

```
BGREWRITEAOF：fork 子进程，读当前内存数据生成等效的精简命令集
（不是重放旧 AOF，是重新生成最终状态）
重写期间新命令写入旧 AOF + AOF 重写缓冲区，子进程完成后合并追加
```

### 混合持久化（Redis 4.0+，生产推荐）

```
AOF 重写时：
┌──────────────────────────────────────────────────┐
│  前半段：RDB 格式（某时刻全量快照，二进制压缩，恢复快）│
│  后半段：AOF 增量命令（快照后的写操作，保证最新数据） │
└──────────────────────────────────────────────────┘
恢复：先加载 RDB（快）→ 再回放少量 AOF（准）
```

**生产推荐配置**：

```
appendonly yes               # 开启 AOF
appendfsync everysec         # 每秒 fsync，平衡安全与性能
aof-use-rdb-preamble yes     # 开启混合持久化（4.0+）
```

### 三种持久化方案对比

| 维度 | RDB | AOF (Everysec) | 混合 |
|------|-----|----------------|------|
| 数据丢失 | 分钟级 | ≤1 秒 | ≤1 秒 |
| 恢复速度 | 快 | 慢（重放所有命令）| 快（RDB + 少量 AOF）|
| 文件大小 | 小（二进制压缩）| 大（所有命令文本）| 中 |
| 性能影响 | fork 瞬间卡顿 | 每秒 fsync 有抖动 | 同 AOF |
| 适用场景 | 可接受分钟级丢失 | 对丢失敏感 | **生产推荐** |

## 四、分布式锁

### 基础实现：SET NX EX + Lua 解锁

**加锁**：

```lua
-- 原子命令：key 不存在时设置 value，并设 TTL
SET lock_key {uuid} EX 10 NX
-- EX 10：10 秒自动过期（防止死锁）
-- NX：只有 key 不存在才设置
-- value 用 UUID：标识锁的持有者，防止误删
```

**解锁（必须 Lua 脚本保证原子性）**：

```lua
-- GET + DEL 两步必须原子，否则可能误删别人的锁
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**为什么解锁必须用 Lua**：`GET` 之后 `DEL` 之前，可能其他线程已获取锁并修改了 value，直接 `DEL` 会误删别人的锁。

**基础实现的三个问题**：

| 问题 | 描述 |
|------|------|
| TTL 难估计 | 设短了业务未完成锁就过期；设长了宕机后死锁等待时间长 |
| 不可重入 | 同一线程再次加锁会阻塞（JVM 的 ReentrantLock 是可重入的）|
| 单点故障 | Redis 挂了所有锁失效 |

### Redisson：工程级分布式锁

**Watchdog 自动续期机制**：

```
线程获取锁 → 默认 TTL = 30s
               ↓
          每 10s（TTL/3）后台线程检查
               ↓
          线程仍持有锁 → 重置 TTL 为 30s（续期）
               ↓
          线程调用 unlock() → 停止续期
               ↓
          JVM 宕机 → 后台线程消亡 → TTL 倒计时 → 30s 内自动释放
```

关键点：**线程活着锁就不过期；线程死了锁最多 30s 内自动释放**。

**可重入实现（Hash 结构）**：

Redisson 用 Hash 而非 String 存锁，支持可重入：

```
key:   lock_key
field: {uuid}:{threadId}    # 精确到线程
value: 重入次数（整数）
```

```lua
-- 加锁 Lua（简化）
if (redis.call('EXISTS', KEYS[1]) == 0) then
    redis.call('HSET', KEYS[1], ARGV[2], 1)      -- 新建，重入次数=1
    redis.call('PEXPIRE', KEYS[1], ARGV[1])
    return nil
end
if (redis.call('HEXISTS', KEYS[1], ARGV[2]) == 1) then
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1)   -- 重入，次数+1
    redis.call('PEXPIRE', KEYS[1], ARGV[1])
    return nil
end
return redis.call('PTTL', KEYS[1])               -- 被其他线程持有，返回剩余 TTL

-- 解锁 Lua（简化）
if (redis.call('HEXISTS', KEYS[1], ARGV[3]) == 0) then
    return nil                                    -- 不是自己的锁
end
local counter = redis.call('HINCRBY', KEYS[1], ARGV[3], -1)  -- 重入次数-1
if (counter > 0) then
    redis.call('PEXPIRE', KEYS[1], ARGV[2])      -- 仍有重入，续期
    return 0
else
    redis.call('DEL', KEYS[1])                   -- 完全释放
    return 1
end
```

**Redisson API**：

```java
RLock lock = redisson.getLock("order:lock:123");
try {
    lock.lock();          // 阻塞等待（watchdog 自动续期）
    // lock.lock(10, TimeUnit.SECONDS); // 手动指定 TTL，watchdog 不续期
    // business logic
} finally {
    lock.unlock();
}
```

### RedLock（多节点锁，了解即可）

向 N 个独立 Redis 节点同时申请锁，超过 N/2+1 个成功才算获取锁。工程争议大（Martin Kleppmann 指出时钟漂移下仍不安全），**简单场景用单节点 Redisson 即可**。

## 五、集群与高可用

### 主从复制（Replication）

```
主节点（Master）: 读写
   │  复制
   ▼
从节点（Slave）: 只读
从节点（Slave）: 只读
```

**全量同步**（新从节点首次接入）：

```
从节点发 PSYNC → 主节点 BGSAVE → 生成 RDB → 传输给从节点
从节点加载 RDB → 同步完成
BGSAVE 期间主节点新写命令 → 存入复制缓冲区 → RDB 加载后追加发送
```

**增量同步**（断线重连）：

- 主节点维护 `replication buffer`（环形缓冲区）
- 从节点重连时发送上次复制的 offset
- offset 在缓冲区内 → 增量同步；超出缓冲区 → 重新全量同步

**局限**：主节点故障需**手动**执行 `SLAVEOF NO ONE` 提升从节点。

### 哨兵模式（Sentinel）

在主从基础上增加哨兵进程实现**自动故障转移**：

```
Sentinel-1、Sentinel-2、Sentinel-3 （奇数个，形成仲裁集群）
监控 Master + Slave-1、Slave-2
```

**故障切换流程**：

```
① 哨兵每秒向 Master 发 PING
② 超时无响应 → 该哨兵标记 Master 为"主观下线"（SDOWN）
③ 超过 quorum 数量的哨兵都认为下线 → "客观下线"（ODOWN）
④ 哨兵之间 Raft 选举出 Leader
⑤ Leader 选择最新 offset 的从节点提升为新 Master
⑥ 通知其他从节点改为复制新 Master
⑦ 通知客户端（发布订阅），客户端重新获取 Master 地址
```

**局限**：仍是单主，写能力受限；容量受限于单机内存；切换过程有秒级不可用。

### Redis Cluster（分片集群）

**哈希槽分片**：

```
总槽位：16384 个 slot（固定）
key → CRC16(key) % 16384 → 确定 slot → 由对应主节点负责

例：3 主节点平均分配
节点 A: slot 0    - 5460
节点 B: slot 5461 - 10922
节点 C: slot 10923 - 16383
```

**为什么是 16384 个槽**：节点数不超过 1000 时足够分配；心跳包中 16384 bit 的 bitmap 仅 2KB，不会太大。

**每个主节点配置若干从节点**（提供高可用）：

```
Master-A  ← Slave-A1, Slave-A2
Master-B  ← Slave-B1
Master-C  ← Slave-C1
```

**MOVED 重定向**：

```
客户端 → 向节点 A 请求 key → 节点 A 判断 key 属于节点 B 的 slot
→ 返回 MOVED {slot} {nodeB-ip:port}
→ 客户端重定向到节点 B
```

智能客户端（Lettuce / Jedis Cluster）会缓存槽位映射，避免每次重定向。

### 脑裂问题与解法

**场景**：网络分区时哨兵选出新 Master，但旧 Master 仍活着继续接受写请求 → 两个 Master 同时写，数据分裂。

**Redis 配置解法**：

```
min-replicas-to-write 1      # 至少 1 个从节点连接才允许写
min-replicas-max-lag 10      # 从节点延迟超过 10s 则拒绝写
```

旧 Master 发现从节点全断 → 拒绝写请求 → 网络恢复后降为从节点，数据丢失最小化。

## 六、综合对比

### RDB vs AOF vs 混合

| 维度 | RDB | AOF | 混合 |
|------|-----|-----|------|
| 数据安全 | 低（分钟级丢失）| 高（秒级）| 高（秒级）|
| 恢复速度 | 快 | 慢 | 快 |
| 文件大小 | 小 | 大 | 中 |
| 运维复杂度 | 低 | 中 | 中 |
| 生产推荐 | 纯缓存场景 | 数据主存储 | **通用推荐** |

### 哨兵 vs Cluster

| 维度 | 哨兵模式 | Redis Cluster |
|------|---------|---------------|
| 故障转移 | 自动 | 自动 |
| 写扩展性 | 单主 | 多主分片 |
| 数据容量 | 单机上限 | 横向扩展 |
| 运维复杂度 | 中 | 高 |
| 事务/多key | 支持 | 受限（必须同 slot）|
| 适用场景 | 数据量 < 100GB | 数据量大或需写扩展 |

### Redis 锁 vs ZooKeeper 锁

| 维度 | Redis 锁 | ZooKeeper 锁 |
|------|---------|-------------|
| 性能 | 高（内存 <1ms）| 低（ZAB 协议 10ms+）|
| 一致性 | AP（主从同步异步，可能短暂不一致）| CP（强一致）|
| 可重入 | Redisson Hash 实现 | 临时顺序节点实现 |
| 适用场景 | 性能敏感、可接受极低概率的锁丢失 | 强一致性要求 |

### Redis vs MySQL

| 维度 | Redis | MySQL |
|------|-------|-------|
| 存储介质 | 内存（可持久化到磁盘）| 磁盘（缓冲到 Buffer Pool）|
| 访问速度 | ns ~ μs | ms 级 |
| 数据结构 | String/List/Hash/Set/ZSet | 表格（行列）|
| 查询能力 | Key 直接访问，无复杂查询 | SQL，支持复杂查询 |
| 事务 | 弱（MULTI/EXEC 不支持回滚）| 完整 ACID |
| 适用场景 | 缓存、计数、排行榜、分布式锁 | 持久化业务数据 |

### 分布式锁三段答法（面试标准）

1. **基础**：`SET key uuid EX 10 NX` + Lua 原子解锁
2. **进阶**：Redisson watchdog（每 10s 续期 30s）+ Hash 可重入
3. **高可用**：RedLock（多节点，了解即可）

## 七、生产风险

| 风险 | 表现 | 应对 |
|------|------|------|
| **大 Key** | 单个 key 的 value 过大（>1MB），序列化/传输/删除慢，阻塞主线程 | `redis-cli --bigkeys` 扫描；用 `UNLINK` 异步删除；业务层拆分 |
| **热 Key** | 单个 key QPS 过高，打爆单节点 | 本地缓存（Caffeine）+ Redis 多副本分读；Cluster 中用 hashtag 分散 |
| **fork 卡顿** | 大内存实例 BGSAVE/BGREWRITEAOF fork 慢（页表复制），主线程瞬间卡几秒 | 控制单实例 maxmemory（建议 ≤ 10GB）；规避业务高峰触发 BGSAVE |
| **持久化配置过松** | `appendfsync no` + 不开 RDB，崩溃丢大量数据 | 生产开混合持久化 + Everysec；主节点禁用 `SAVE` |
| **不要在主节点 `SAVE`** | `SAVE` 阻塞主线程，期间所有请求超时 | 只用 `BGSAVE` 或配置触发 |
| **主从切换锁丢失** | Redis 主从异步复制，主节点宕机时未同步的锁被丢失，可能两节点同时持锁 | 业务层加幂等兜底；高一致性场景改用 ZooKeeper |
| **脑裂数据丢失** | 网络分区后旧 Master 继续写，分区恢复后数据被覆盖 | 配置 `min-replicas-to-write` + `min-replicas-max-lag` |
| **Cluster 跨 slot 操作** | `MGET`/事务/Lua 涉及多 key 跨 slot，报错 | 用 hashtag `{user:1}:name` 保证同 slot；或应用层拆分 |
| **AOF 文件膨胀** | 未配置 AOF 重写，文件无限增长 | 确认 `auto-aof-rewrite-percentage` 和 `auto-aof-rewrite-min-size` 配置 |

## 八、与其他概念的关系

- 类比 [[概念-MySQL]] 日志：AOF ≈ binlog（追加写操作日志），RDB ≈ 数据库全量备份；混合持久化的思路与 MySQL 的 redo log（快照 + 增量）异曲同工
- 依赖 [[机制-B+树]]：理解为什么 ZSet 不用 B+ 树——跳表已足够，且 Redis 是内存操作，B+ 树的磁盘 IO 友好特性无优势
- 类比 [[机制-CAS]]：`SET NX` 相当于 CAS（只有 key 不存在才写入），失败则自旋重试；Redisson 可重入锁与 [[机制-AQS]] 的 ReentrantLock 思路相同，都是用计数实现重入
- 支撑 [[概念-Redis]]：String 是主要缓存载体；Cluster 多主节点让单点故障不影响整个缓存层；分布式锁可用于解决缓存击穿（只让一个请求穿透到 DB）
- 支撑 [[概念-幂等设计]]：`SET NX` 天然幂等（重复执行只有第一次生效）；SETNX + Lua 是防重复提交的经典方案

**层间依赖**：

- 依赖 **L4 数据结构**：ZSet 底层跳表是 L4 的工程实现；Dict 是哈希表；布隆过滤器底层是 Bitmap
- 依赖 **L3 并发**：分布式锁解决的是分布式场景下的 L3 互斥问题；watchdog 是后台线程续期
- 与 **L5 MySQL** 互补：Redis 是 MySQL 的缓存层，两者协同处理读多写少场景；一致性问题跨越两个系统
- 被 **L6 框架**使用：Spring Cache 可用 Redis 作为缓存后端；Spring Session 用 Redis 实现分布式 Session
- 被 **L7 分布式**使用：分布式限流（令牌桶/滑动窗口用 Lua + Redis）、分布式 Session、消息幂等去重

## 九、应用边界

**数据类型选型**：

| 场景 | 推荐类型 | 原因 |
|------|---------|------|
| 计数器 / 简单 KV | String + INCR | 原子自增，O(1) |
| 排行榜 / 带权重排序 | ZSet | ZADD/ZRANGE 自动排序 |
| 用户标签 / 共同好友 | Set | SINTERCARD 集合交集 |
| 购物车 / 对象属性 | Hash | HGETALL 一次取所有字段 |
| 简单消息队列 | List | LPUSH/BRPOP 阻塞弹出 |

**持久化选型**：
- 纯缓存（数据可从 DB 重建）→ 只用 RDB，或不开持久化
- Redis 作主存储 → 混合持久化 + Everysec
- 不容许任何数据丢失 → AOF Always（性能下降明显，接近磁盘速度）

**集群选型**：
- 数据量 < 10GB，高可用一般 → 主从 + 哨兵
- 数据量 > 100GB 或需要写扩展 → Redis Cluster
- 开发测试 → 单机或简单主从

**分布式锁边界**：
- 适合：防重复提交（幂等）、分布式定时任务（单节点执行）、库存扣减（防超卖）
- 不适合：锁安全性极高的金融强一致场景（改用 DB 行锁或 ZK）；锁持有时间极短（< 1ms，直接用 DB 乐观锁）

**不要做的事**：
- 不要在主节点执行 `SAVE`（阻塞主线程）
- 不要关闭 AOF 重写（文件无限增长）
- 不要把大对象（> 1MB）直接存 Redis String（序列化开销 + 网络传输 + 阻塞主线程）
- 不要用 Redis 替代关系型数据库（没有索引查询、事务语义弱、Cluster 下事务受限）

## 十、实战场景设计

### 排行榜（ZSet）

```redis
ZADD leaderboard <score> <userId>      -- 添加/更新分数
ZREVRANK leaderboard <userId>           -- 查询排名（从高到低）
ZREVRANGE leaderboard 0 9 WITHSCORES   -- 查询前10名
```

ZSet 内部跳表 + 哈希表，插入/删除/范围查询均为 O(log N)。

**百万级排行榜的四个工程挑战**：

| 问题 | 解法 |
|------|------|
| BigKey（成员 > 10w）| 按维度分片：`leaderboard:province:guangdong`，查全国前10则各省取前10做归并排序 |
| 高并发写（分数实时变化）| 行为写 MQ，消费端异步 `ZINCRBY` 更新 ZSet |
| 分数相同按时间排名 | 编码合并：`score = (业务分数 × 1e13) + (Long.MAX_VALUE - 时间戳)`，分数高排前，同分则时间早排前 |
| 容灾与持久化 | 开启 AOF + RDB；重要周期榜到期前备份到 MySQL，重启从 DB 重建 |

### 附近的人（GEO）

Redis 3.2+ 内置 Geospatial，底层是 ZSet（GeoHash 编码为 score）：

```redis
GEOADD users:location <longitude> <latitude> <userId>  -- 写入位置
GEOSEARCH users:location FROMLONLAT 116.40 39.90 BYRADIUS 1 km ASC  -- 附近1km，6.2+推荐
```

**工程要点**：GeoHash 精度 100m 量级内够用，超精度用 PostGIS；大城市用户多时按区域分 key；位置频繁更新时用 MQ 异步写，避免写热点。

### 朋友圈点赞（ZSet / Set）

```redis
ZADD likes:post123 <timestamp> user456    -- 点赞（score=时间戳，天然按时间排序）
ZREM likes:post123 user456                -- 取消赞
ZCARD likes:post123                       -- 点赞总数
ZREVRANGEBYSCORE likes:post123 +inf -inf  -- 按时间倒序展示谁赞了
```

选 ZSet 而非 Set 的原因：**天然支持时间排序**，"张三、李四刚刚赞过"不需要额外存时间字段。仅需计数时用 String INCR 更省内存。

### 抢红包（List 预分配 + 二倍均值法）

**核心算法：二倍均值法**（保证公平性）：

```
每次随机上限 = (剩余金额 / 剩余个数) × 2
随机金额 ∈ [0.01, 上限]
最后一个人金额 = 剩余所有金额
```

**高并发方案**：

```
1. 红包发出时：预先计算好所有金额，存入 Redis List
   RPUSH hongbao:123 500 300 200 ...（单位：分）

2. 用户抢：原子弹出
   LPOP hongbao:123   -- 返回 null 表示抢完

3. 到账记录：写 MQ，异步落库（userId, amount, timestamp）
```

每次抢红包 O(1) 且原子，无超卖无并发冲突。

### UV 统计（HyperLogLog）

```redis
PFADD uv:2026-05-07 user123         -- 记录访问
PFCOUNT uv:2026-05-07               -- 查今日 UV（约数，误差 0.81%）
PFMERGE uv:week uv:05-01 ... uv:05-07  -- 合并周 UV
```

| | HyperLogLog | Set |
|--|--|--|
| 内存 | 固定 12KB | O(n)，千万 UV = 数百 MB |
| 精度 | 约 0.81% 误差 | 精确 |
| 支持删除 | 不支持 | 支持 |

精度要求严格（账单统计）→ Set；可接受误差（展示 PV/UV）→ HyperLogLog。

### 购物车（Hash）

```redis
HSET cart:user123 sku:456 2       -- 加入/更新购物车
HINCRBY cart:user123 sku:456 1    -- 数量 +1
HDEL cart:user123 sku:456         -- 删除商品
HGETALL cart:user123              -- 查看全部
```

选 Hash 而非 String：可精确操作单个 sku，不需要序列化/反序列化整个购物车。

**持久化策略**：已登录用户 Redis Hash 作缓存，定时同步 DB；未登录用户存 localStorage，登录后合并；Redis 重启从 DB 重建热点用户购物车。

### 实战场景选型速查

| 场景 | 核心数据结构 | 主要挑战 |
|------|------------|---------|
| 排行榜 | ZSet | BigKey 分片、并发写异步、分数相同排序编码 |
| 附近的人 | GEO（底层 ZSet）| 精度边界、大城市 BigKey、位置更新频率 |
| 点赞 | ZSet（需时间排序）/ Set（仅计数）| 按时间顺序展示选 ZSet |
| 抢红包 | List（预分配）| 公平性（二倍均值法）、原子 LPOP 防超卖 |
| UV 统计 | HyperLogLog | 可接受 0.81% 误差则用；精确统计用 Set |
| 购物车 | Hash | 未登录合并策略、持久化策略 |

## 十一、缓存三大问题

> 三大问题本质都是"缓存挡不住请求，流量打到数据库"，成因不同，解法不同。

### 缓存穿透

**场景**：请求一个数据库中根本不存在的 key（如恶意攻击用不存在的用户 ID）→ 缓存永远 miss → 全量打 DB

**解法 1：布隆过滤器**

```
请求 → 先查布隆过滤器
  → 不存在 → 直接返回（不查 DB）
  → 可能存在 → 查缓存 → 查 DB
```

布隆过滤器用多个哈希函数将 key 映射到 bit 数组（参见 [[概念-BitMap]]）：判断"不存在"永远准确；判断"存在"有误判率；不支持删除。

**解法 2：缓存空值**

DB miss 时缓存 `NULL` 或特殊标记，设置短 TTL（5 分钟）。简单但恶意攻击用不同不存在 key 时浪费内存。

### 缓存击穿

**场景**：某热点 key 过期，大量并发同时 miss，全部打 DB

| 解法 | 机制 | 代价 |
|------|------|------|
| 互斥锁 | 缓存 miss → 抢分布式锁 → 重建 → 释放 | 等锁期间响应变慢 |
| 逻辑过期 | 热点 key 永不设 TTL，后台异步更新 | 短暂可能读到旧数据 |
| 提前预热 + 主动续期 | 监控 TTL，过期前主动重建 | 额外定时任务 |

### 缓存雪崩

**场景 1**：大量 key 相同 TTL 同时过期 → 解法：随机化 TTL（`base_ttl + random(0, 300)`）

**场景 2**：Redis 服务宕机 → 解法：Redis Cluster（单节点故障自动切换）；熔断降级（超阈值返回兜底数据）；多级缓存（Caffeine 本地缓存兜底）

### 缓存与数据库一致性

| 策略 | 操作顺序 | 分析 |
|------|---------|------|
| 先更新 DB，再删缓存 | UPDATE DB → DELETE cache | **推荐**。短暂不一致（DB 新，缓存旧）；删除失败可重试 |
| 先删缓存，再更新 DB | DELETE cache → UPDATE DB | 脏读风险：删后线程查缓存 miss → 读旧 DB → 回填旧值 |
| 延迟双删 | DELETE → UPDATE DB → sleep → DELETE | 防脏读，但 sleep 时长难定 |
| Binlog 异步删缓存 | 监听 binlog（Canal）→ 异步删 cache | 最终一致，解耦，推荐大规模场景 |

**最终一致性方案**：`应用 → UPDATE MySQL → Canal 监听 binlog → 发消息 → 消费者删除 Redis cache`

### 内存淘汰策略（8 种）

Redis 内存用满（达到 `maxmemory`）时触发淘汰：

| 策略 | 范围 | 淘汰规则 |
|------|------|---------|
| `noeviction` | 全部 | 不淘汰，写操作报错 |
| `allkeys-lru` | 全部 key | LRU（最近最少使用）|
| `allkeys-lfu` | 全部 key | LFU（最近最少频率，4.0+）|
| `allkeys-random` | 全部 key | 随机淘汰 |
| `volatile-lru` | 有 TTL 的 key | LRU |
| `volatile-lfu` | 有 TTL 的 key | LFU（4.0+）|
| `volatile-random` | 有 TTL 的 key | 随机淘汰 |
| `volatile-ttl` | 有 TTL 的 key | TTL 越短越先淘汰 |

**生产选择**：缓存场景 → `allkeys-lru` 或 `allkeys-lfu`（LFU 对热点保护更好）；有不能丢的数据 → `volatile-lru`（不淘汰无 TTL 的 key）。

### 三大问题速查

| 问题 | 根因 | 推荐解法 |
|------|------|---------|
| 穿透 | key 根本不存在 | 布隆过滤器（大规模）/ 缓存空值（简单场景）|
| 击穿 | 热点 key 在关键时刻过期 | 互斥锁（强一致）/ 逻辑过期（高性能）|
| 雪崩（批量失效）| 大量 key 同 TTL | 随机化 TTL |
| 雪崩（宕机）| Redis 实例故障 | Cluster + 熔断 + 多级缓存 |

## 十二、面试追问

**问**：Redis 为什么快？
**答**：①纯内存，ns 级读写；②单线程命令执行，无上下文切换和锁竞争；③epoll IO 多路复用，单线程处理大量连接；④高效数据结构（SDS/ziplist/skiplist/listpack）；⑤Redis 6.0+ 多线程处理网络 I/O，命令执行仍单线程。瓶颈不在 CPU 而在网络 I/O，单线程足够。

**问**：Redis 和 MySQL 数据一致性怎么保证？
**答**：**先更新 DB，再删缓存**（Cache Aside Pattern）。不用"先删后更新"，并发场景产生脏数据（线程A删缓存→线程B读DB写旧缓存→线程A写新DB，缓存永远是旧值）。两层保障：①写路径删缓存；②Canal 订阅 binlog 异步删缓存（防写路径删除失败残留）。强一致场景（如余额）不走缓存，直接读 DB。

**问**：Redis bigkey 怎么处理？
**答**：发现：`redis-cli --bigkeys` 或 `MEMORY USAGE key`。  
删除：用 `UNLINK`（Redis 4.0+ 异步删除），**不要** `DEL`（主线程阻塞）。  
预防：写入时按 key 分桶（如 `rank:0~9`，每个子 key 控制在合理大小）。  
实战：排行榜 ZSet 200 万成员，`DEL` 导致主线程阻塞 2 秒，改 `UNLINK` + 按前缀分 10 个 key 解决。

**问**：Redis Cluster 迁移期间 MOVED 和 ASK 的区别？
**答**：`MOVED`：key 已迁移完成，客户端更新本地 slot→节点路由表，之后直接请求新节点。`ASK`：key 正在迁移中，客户端临时向新节点发 `ASKING` 再发请求（一次性，不更新路由表，下次同 slot 还走旧节点）。Redisson/Lettuce 已内置自动处理，业务层无感知。
