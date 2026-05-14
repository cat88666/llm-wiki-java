---
type: concept
status: active
name: "Redis分布式锁"
layer: L5
aliases: ["SETNX", "Redisson", "watchdog", "续期", "可重入锁", "分布式锁"]
related:
  - "[[概念-Redis]]"
  - "[[机制-Redis集群与高可用]]"
  - "[[机制-CAS]]"
  - "[[机制-AQS]]"
sources:
  - "../../../raw/note/Hollis/Redis/✅如何用SETNX实现分布式锁？.md"
  - "../../../raw/note/Hollis/Redis/✅Redisson的watchdog机制是怎么样的？.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# Redis 分布式锁

> 分布式锁解决多个进程/节点并发访问共享资源的互斥问题；Redis 分布式锁基于 `SET NX EX` 原子命令实现互斥，Redisson 在此基础上增加了 watchdog 自动续期、可重入、公平锁等工程能力。

## 第一性原理

单机锁（`synchronized` / `ReentrantLock`）只在 JVM 内有效，多节点部署时失效。分布式锁需要一个**所有节点都能访问的共享状态**来实现互斥。Redis 作为分布式锁的选型理由：
- 单线程模型保证命令的原子性（SET NX EX 是一条命令）
- 内存速度，加解锁延迟低（<1ms）
- 天然支持 TTL，进程崩溃后锁自动释放

## 核心机制

### 基础实现：SET NX EX + Lua 解锁

**加锁**：
```lua
-- 一条原子命令：key 不存在时设置 value，并设 TTL
SET lock_key {uuid} EX 10 NX
-- EX 10：10秒自动过期（防止死锁）
-- NX：只有 key 不存在时才设置
-- value 用 UUID：标识锁的持有者，防止误删
```

**解锁（必须用 Lua 脚本保证原子性）**：
```lua
-- 检查是否是自己的锁，是才删除（判断 + 删除 两步必须原子）
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**为什么解锁要用 Lua**：`GET` 之后 `DEL` 之前，可能其他线程已经获取了锁并修改了 value，直接 `DEL` 会误删别人的锁。

**基础实现的问题**：
1. **TTL 设多少合适**？设短了业务未完成锁就过期；设长了宕机后死锁时间长
2. **不可重入**：同一线程再次加锁会阻塞（JVM 锁是可重入的）
3. **单点 Redis 故障**：Redis 挂了所有锁都失效

### Redisson：工程级 Redis 分布式锁

Redisson 是 Redis 的 Java 客户端，提供完整的分布式锁实现。

**Watchdog 自动续期机制**：
```
线程获取锁 → Redisson 默认 TTL = 30s
                 ↓
            每 10s（TTL/3）后台线程检查
                 ↓
            线程仍持有锁 → 重置 TTL 为 30s（续期）
                 ↓
            线程完成 unlock() → 停止续期
                 ↓
            JVM 宕机 → 后台线程消亡 → TTL 倒计时 → 自然过期
```

关键点：**只要线程还活着，锁就永不过期；线程死了，锁最多 30 秒内自动释放**。

**可重入实现（Hash 结构）**：

Redisson 不用 String 存锁，而是用 **Hash**：
```
key: lock_key
field: {uuid}:{threadId}    # 标识到具体线程
value: 重入次数（整数）
```

```lua
-- 加锁 Lua（简化）
if (redis.call('EXISTS', KEYS[1]) == 0) then
    redis.call('HSET', KEYS[1], ARGV[2], 1)  -- 新建，重入次数=1
    redis.call('PEXPIRE', KEYS[1], ARGV[1])
    return nil
end
if (redis.call('HEXISTS', KEYS[1], ARGV[2]) == 1) then
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1)  -- 重入，次数+1
    redis.call('PEXPIRE', KEYS[1], ARGV[1])
    return nil
end
return redis.call('PTTL', KEYS[1])  -- 锁被其他线程持有，返回剩余 TTL
```

```lua
-- 解锁 Lua（简化）
if (redis.call('HEXISTS', KEYS[1], ARGV[3]) == 0) then
    return nil  -- 不是自己的锁
end
local counter = redis.call('HINCRBY', KEYS[1], ARGV[3], -1)  -- 重入次数-1
if (counter > 0) then
    redis.call('PEXPIRE', KEYS[1], ARGV[2])  -- 仍有重入，续期
    return 0
else
    redis.call('DEL', KEYS[1])  -- 完全释放
    return 1
end
```

**Redisson API**：
```java
RLock lock = redisson.getLock("order:lock:123");
try {
    lock.lock();          // 阻塞等待（watchdog 自动续期）
    // lock.lock(10, TimeUnit.SECONDS);  // 手动指定 TTL，watchdog 不续期
    // business logic
} finally {
    lock.unlock();
}
```

### RedLock（多节点锁，了解即可）

Redisson 实现了 Antirez 提出的 RedLock 算法：向 N 个独立 Redis 节点同时申请锁，超过 N/2+1 个成功才算获取锁，防止单节点故障。但 Martin Kleppmann 指出 RedLock 在时钟漂移下仍不安全，工程上争议大，**简单场景用单节点 Redisson 即可**。

## 关键权衡

1. **手动设 TTL vs watchdog 续期**：手动 TTL 需要精确估计业务耗时，估错就有问题；watchdog 自动续期但增加了网络开销（每 10s 一次 Redis 命令），且强依赖 Redis 连通性
2. **Redis 分布式锁 vs ZooKeeper 分布式锁**：Redis 锁高性能（内存）但 AP（可能有短暂不一致）；ZK 锁强一致（CP）但性能低（ZAB 协议），对锁可靠性要求极高时选 ZK
3. **String vs Hash 存锁**：String（基础实现）简单但不可重入；Hash（Redisson）可重入但实现复杂

## 与其他概念的关系

- 依赖 [[概念-Redis]]：基础实现用 String（SET NX EX），Redisson 可重入用 Hash（HINCRBY）
- 类比 [[机制-AQS]]：Redisson 可重入锁与 JVM 的 ReentrantLock（基于 AQS）思路相同，都是用计数实现重入；区别是 AQS 用 volatile int state，Redisson 用 Redis Hash
- 类比 [[机制-CAS]]：SET NX 相当于 CAS 操作（只有 key 不存在才 SET），失败则重试（自旋）
- 被 [[机制-Redis集群与高可用]] 影响：Redis 主从切换时，未同步的锁信息可能导致两个客户端同时持锁

## 应用边界

**适用场景**：
- 防止重复提交（订单支付幂等）
- 分布式定时任务（只让一个节点执行）
- 库存扣减（防止超卖）

**不适用场景**：
- 对锁安全性要求极高（金融强一致场景）→ 考虑数据库行锁或 ZooKeeper
- 锁持有时间极短（<1ms）→ 直接用数据库乐观锁（CAS UPDATE）

**面试标准答法**：
1. 基础：`SET key uuid EX 10 NX` + Lua 原子解锁
2. 进阶：Redisson watchdog（每10s续期30s）+ Hash 可重入
3. 高可用：RedLock（多节点，了解即可）
