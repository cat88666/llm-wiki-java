---
type: concept
status: active
name: "Redis数据类型与底层结构"
layer: L5
aliases: ["Redis数据结构", "SDS", "ZipList", "ListPack", "SkipList", "跳表", "Redis快的原因"]
related:
  - "[[机制-Redis持久化]]"
  - "[[概念-缓存三大问题]]"
  - "[[机制-Redis集群与高可用]]"
  - "[[机制-B树与B加树]]"
sources:
  - "../../raw/note/Hollis/Redis/✅Redis 支持哪几种数据类型？ 30f3673e113881a3bfb2c64c52e11fa7.md"
  - "../../raw/note/Hollis/Redis/✅Redis中的Zset是怎么实现的？ 30f3673e113881319e8ddeb6b71b3b4c.md"
  - "../../raw/note/Hollis/Redis/✅Redis为什么这么快？ 30f3673e11388198b8b8d7e0cdd9b62f.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# Redis 数据类型与底层结构

> Redis 对外暴露 5 种基本数据类型（String/List/Hash/Set/ZSet），每种类型根据数据规模动态切换底层编码（小数据用紧凑结构，大数据用高性能结构）；所有操作基于内存 + 单线程事件循环，是其高性能的根本原因。

## 第一性原理

Redis 的核心矛盾：**内存宝贵，但访问必须快**。解决方式是"二态编码"——小数据使用紧凑的连续内存结构（ZipList/ListPack）减少内存碎片和指针开销；数据增大后切换到高性能结构（SkipList/Dict/LinkedList）保证操作复杂度。Redis 为什么快？根本是内存 + 单线程消除锁竞争 + IO多路复用 + 为缓存场景设计的数据结构。

## 核心机制

### 5 种基本数据类型

| 类型 | 典型用途 | 底层编码（小→大） |
|------|---------|----------------|
| **String** | 计数器、缓存对象、分布式锁 | int（整数）/ embstr（≤44字节）/ raw（SDS）|
| **List** | 消息队列、最新N条 | ListPack（小）→ QuickList（大）|
| **Hash** | 对象字段存储 | ListPack（小）→ Dict哈希表（大）|
| **Set** | 去重、共同好友、标签 | ListPack（小）→ Dict（大）|
| **ZSet** | 排行榜、带权重队列 | ListPack（小）→ SkipList + Dict（大）|

> 阈值（默认）：元素数 ≤ 128 且单个元素 ≤ 64 字节 → 用紧凑结构；超出阈值 → 切换高性能结构

### SDS（Simple Dynamic String）

Redis 的字符串底层不是 C 字符串，而是 SDS：

```c
struct sdshdr {
    int len;    // 已用长度，O(1) 获取长度
    int free;   // 剩余空间，支持预分配，减少扩容次数
    char buf[]; // 实际数据，以 '\0' 结尾，兼容 C 函数
};
```

SDS 优势：O(1) 获取长度 / 预分配减少 realloc / 二进制安全（可存 `\0` 字节）

### ZSet 双结构（最典型的二态编码）

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

**为什么 ZSet 用跳表而非 B+ 树？**
- 跳表实现更简单，代码易读易维护
- 跳表范围查询（`ZRANGEBYSCORE`）性能与 B+ 树相当
- 跳表内存局部性差但 Redis 本身是内存操作，磁盘 IO 不是瓶颈

### Redis 快的 5 个原因

1. **内存存储**：所有数据在内存，ns 级访问 vs 磁盘 ms 级
2. **单线程处理命令**：消除锁竞争，避免线程切换开销（6.0+ 网络 IO 多线程，命令执行仍单线程）
3. **IO 多路复用**：epoll 事件驱动，单线程处理大量并发连接，无阻塞等待
4. **高效数据结构**：为缓存场景设计（SDS/Dict/SkipList），时间复杂度低
5. **二态编码**：小数据用紧凑结构减少内存分配和 GC 压力

## 关键权衡

1. **单线程 vs 多线程**：单线程命令处理天然无锁，但 CPU 利用率低；Redis 6.0 在网络 IO 引入多线程，命令处理保持单线程，是折中方案
2. **ZipList/ListPack 的阈值选择**：阈值太低 → 过早切换到大结构，浪费内存；阈值太高 → ListPack 中查找太慢（O(n)）
3. **跳表 vs 平衡树**：跳表实现简单，随机层数让插入天然 O(log n) 均摊；平衡树实现复杂但最坏情况更稳定

## 与其他概念的关系

- 依赖 [[机制-B树与B加树]]：理解为什么 ZSet 不用 B+ 树（跳表已经足够，且实现更简单）
- 被 [[概念-缓存三大问题]] 使用：String 是主要缓存载体，ZSet 常用作排行榜
- 被 [[机制-Redis分布式锁]] 使用：String 类型的 SET NX EX 是分布式锁的基础命令
- 与 [[机制-Redis持久化]] 关联：持久化针对所有数据类型，理解底层结构有助于理解序列化

## 应用边界

**类型选型速查**：
- 计数器 / 简单 KV → String（INCR 原子，SET/GET O(1)）
- 排行榜 / 带权重排序 → ZSet（ZADD/ZRANGE 自动排序）
- 用户标签 / 共同好友 → Set（SINTERCARD 集合交集）
- 购物车 / 对象属性 → Hash（HGETALL 一次获取所有字段）
- 消息队列（简单） → List（LPUSH/BRPOP 阻塞弹出）

**注意边界**：
- Redis String 最大 512MB，但实践上单个 value 超过 1MB 就要考虑是否合理
- ZSet 的 Dict + SkipList 双结构是内存换速度：同一份数据存了两次
- 不要用 Redis 存需要复杂查询的数据（Redis 不是数据库，没有索引查询能力）
