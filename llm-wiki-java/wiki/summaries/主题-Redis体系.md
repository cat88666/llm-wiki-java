---
type: summary
status: active
name: "Redis体系"
layer: L5
tags: ["#storage"]
concepts:
  - "[[概念-Redis]]"
  - "[[机制-Redis持久化]]"
  - "[[概念-缓存三大问题]]"
  - "[[机制-Redis分布式锁]]"
  - "[[机制-Redis集群与高可用]]"
sources:
  - "../../raw/note/Hollis/Redis/"
created: 2026-05-02
updated: 2026-05-02
---

# Redis 体系（L5 存储层）

> 本页是 L5 Redis 知识地图，聚合 5 个核心概念页的依赖关系与高频考点。

---

## 知识地图

```
         [[概念-Redis]]
    ┌──────────────────────────────────────────────┐
    │  String / List / Hash / Set / ZSet           │
    │  小数据: ZipList/ListPack（紧凑内存）           │
    │  大数据: SkipList+Dict / QuickList            │
    │  快的根本: 内存 + 单线程 + epoll + 高效结构      │
    └──────┬────────────────┬─────────────────────┘
           │                │
    ┌──────▼──────┐  ┌──────▼──────────────────┐
    │[[机制-Redis持久化]]  │  │[[概念-缓存三大问题]]  │
    │              │  │穿透(布隆/空值)            │
    │RDB: 全量快照 │  │击穿(互斥锁/不过期)         │
    │AOF: 追加日志 │  │雪崩(随机TTL/集群/熔断)     │
    │混合: 4.0+   │  │8种内存淘汰策略             │
    │推荐everysec │  │缓存与DB一致性(先删/binlog) │
    └──────┬──────┘  └────────────────────────--┘
           │
    ┌──────▼──────────────────────────────────────┐
    │      [[机制-Redis集群与高可用]]               │
    │  主从: 异步复制，手动故障转移                   │
    │  哨兵: 自动故障转移（Raft Leader选举）           │
    │  Cluster: 16384槽分片，多主横向扩展             │
    │  脑裂: min-replicas-to-write 保护             │
    └──────────────────────────────────────────---┘

    [[机制-Redis分布式锁]]（横跨应用层与存储层）
    ┌──────────────────────────────────────────────┐
    │  SET key uuid EX 10 NX + Lua原子解锁          │
    │  Redisson: watchdog(10s续期30s) + Hash可重入  │
    │  RedLock: 多节点多数派（了解）                  │
    └──────────────────────────────────────────────┘
```

---

## Redis vs MySQL 核心对比

| 维度 | Redis | MySQL |
|------|-------|-------|
| 存储介质 | 内存（可持久化到磁盘）| 磁盘（缓冲到 Buffer Pool）|
| 访问速度 | ns~μs | ms 级 |
| 数据结构 | 多种（String/List/Hash/Set/ZSet）| 表格（行列）|
| 查询能力 | Key 直接访问，无复杂查询 | SQL，支持复杂查询 |
| 事务 | 弱（MULTI/EXEC 不支持回滚）| 完整 ACID |
| 适用场景 | 缓存、计数、排行榜、分布式锁 | 持久化业务数据 |

---

## 高频考点汇总

### 数据类型与底层结构

- **ZSet 底层**：小数据 ListPack（紧凑连续内存，O(n) 查找）；大数据 SkipList（O(log n) 有序操作）+ Dict（O(1) 按 key 查分数）
- **Redis 为什么快**：内存存储 + 单线程命令处理（无锁竞争）+ epoll IO多路复用 + 高效数据结构 + 6.0 网络IO多线程
- **SDS vs C字符串**：O(1) 长度获取 / 预分配减少 realloc / 二进制安全

### 持久化

- **RDB**：BGSAVE fork 子进程，COW 保证快照一致性；文件小恢复快；最多丢分钟级数据
- **AOF**：追加写命令；Always/Everysec/No 三种写回策略；AOF 重写压缩文件
- **混合持久化**（生产推荐）：RDB 前半段 + AOF 后半段；恢复快且数据损失小
- **推荐配置**：`appendonly yes` + `appendfsync everysec` + `aof-use-rdb-preamble yes`

### 缓存三大问题

- **穿透**：请求不存在的 key → 布隆过滤器（有误判率，内存友好）/ 缓存空值（简单但占内存）
- **击穿**：热点 key 过期瞬间并发 → 互斥锁（强一致，有等待）/ 逻辑过期（高性能，短暂不一致）
- **雪崩**：大量 key 同时过期 → 随机 TTL；Redis 宕机 → Cluster + 熔断降级 + 本地缓存兜底
- **一致性**（推荐）：先更新 DB 再删缓存；大规模用 Canal 监听 binlog 异步删缓存

### 分布式锁

- **基础**：`SET key uuid EX 10 NX`（原子加锁）+ Lua 脚本（原子解锁，防误删）
- **Redisson watchdog**：默认 TTL 30s，每 10s 自动续期；进程宕机后 30s 内自然过期
- **可重入**：Hash 结构存 `{uuid:threadId} → count`；加锁 HINCRBY +1，解锁 -1，归零才 DEL

### 集群

- **主从**：异步复制（全量 RDB + 增量缓冲区）；手动故障转移
- **哨兵**：SDOWN → ODOWN（quorum 多数确认）→ Raft 选 Leader → 提升新 Master → 通知客户端
- **Cluster**：16384 slots，CRC16(key) % 16384 确定节点；MOVED 重定向；每主节点配从节点
- **脑裂**：`min-replicas-to-write` + `min-replicas-max-lag` 限制孤立主节点写入

---

## 层间依赖

- L5 Redis 依赖 **L4 数据结构**：ZSet 底层跳表是 L4 的工程实现；Dict 是哈希表；BitMap 是 Redis Set 的基础，布隆过滤器的底层
- L5 Redis 依赖 **L3 并发**：分布式锁解决的是分布式场景下的 L3 互斥问题；watchdog 是后台线程续期
- L5 Redis 与 **L5 MySQL** 互补：Redis 是 MySQL 的缓存层，两者协同处理读多写少场景；一致性问题跨越两个系统
- L5 Redis 被 **L6 框架**使用：Spring Cache 可以用 Redis 作为缓存后端；Spring Session 用 Redis 实现分布式 Session
- L5 Redis 被 **L7 分布式**使用：分布式限流（令牌桶/滑动窗口用 Lua + Redis）、分布式 Session、消息幂等去重
