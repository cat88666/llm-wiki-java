# 设计-Gacha与游戏高并发

> 游戏后端高并发的核心场景：Gacha（抽卡）限量道具零超卖、实时对战状态一致性、跨地域低延迟路由。
>
> 来源：`../../raw/note/Interview/Eson.md`（东京游戏公司：Redis Lua原子扣款、对象池+G1 GC、跨地域延迟优化）

---

## 一、Gacha 系统：零超卖设计

### 问题拆分

Gacha 并发挑战分两类：
1. **概率道具**（普通抽奖）：无库存限制，并发写日志即可
2. **限量道具**（UP 角色 / 限定皮肤）：全局限量 N 个，高峰 10 万 QPS，必须零超卖

### Redis Lua 原子扣减（核心方案）

```lua
-- deduct_stock.lua：原子检查并扣减库存
-- KEYS[1] = 库存 key，ARGV[1] = 购买数量
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock == nil then
    return -2  -- 道具不存在
end
if stock < tonumber(ARGV[1]) then
    return -1  -- 库存不足
end
return redis.call('DECRBY', KEYS[1], ARGV[1])  -- 返回扣减后剩余库存
```

```java
// Java 调用
Long remaining = redisTemplate.execute(
    new DefaultRedisScript<>(luaScript, Long.class),
    Collections.singletonList("item:rare:" + itemId),
    String.valueOf(quantity)
);
if (remaining < 0) {
    throw new SoldOutException("道具已售罄");
}
// 走后续落盘流程（本地消息表 → MQ → 异步写 DB）
```

**Lua 为什么能保证原子性**：Redis 单线程执行脚本，整个脚本是一个不可分割的操作单元，中间不会有其他命令插入。

### 落盘流程（保证不丢）

```
Redis Lua 扣减成功
    │
    ├── 写本地消息表（同一 DB 事务）
    │   INSERT INTO pending_grant(userId, itemId, orderId) ...
    │
    └── 触发 MQ 消息（orderId 为 key）
            │
            ▼
    MQ Consumer 异步处理
            │
            ├── 写用户背包（idempotent by orderId）
            └── 更新本地消息表 status = DONE

// 定时任务扫描 pending_grant，补发 MQ（防止 MQ 投递失败）
@Scheduled(fixedDelay = 30_000)
public void retryPendingGrants() {
    List<PendingGrant> pending = pendingGrantRepo.findByStatus(PENDING, 100);
    pending.forEach(g -> mqTemplate.send("gacha.grant", g));
}
```

### 概率算法（内存级，无 DB IO）

```java
// 服务启动时加载道具权重表到内存（或 Redis）
// 查询时无任何网络 IO，微秒级完成
public Item drawItem(String poolId) {
    WeightedPool pool = pools.get(poolId);  // 内存中
    int roll = ThreadLocalRandom.current().nextInt(pool.totalWeight());
    return pool.hitItem(roll);  // 权重区间命中
}

// WeightedPool 数据结构：有序列表 + 区间查找
// [0, 1000) → 普通道具 A（权重1000）
// [1000, 1050) → 稀有道具 B（权重50）
// [1050, 1051) → 限定道具 C（权重1）
```

### 保底（Pity System）

```java
// 用户级别的保底计数存 Redis（玩家连续未得稀有时自动触发）
Long pullCount = redisTemplate.opsForValue().increment("pity:" + userId + ":" + poolId);
if (pullCount >= PITY_THRESHOLD) {  // 如 90 连保底
    result = forcedRareItem();
    redisTemplate.delete("pity:" + userId + ":" + poolId);  // 重置计数
} else {
    result = drawItem(poolId);
    if (result.isRare()) {
        redisTemplate.delete("pity:" + userId + ":" + poolId);
    }
}
```

---

## 二、对象池 + G1 GC 消除 Gacha 高峰 GC 抖动

**问题**：Gacha 高峰期（活动开放瞬间）QPS 激增 10000 倍，大量临时对象（抽奖请求、随机数上下文、结果对象）涌入 Young Gen，频繁 Minor GC，P99 出现抖动。

```java
// 使用 Apache Commons Pool2 池化 DrawContext 对象
GenericObjectPool<DrawContext> contextPool = new GenericObjectPoolFactory<>(
    new DrawContextFactory(),
    new GenericObjectPoolConfig<DrawContext>() {{
        setMaxTotal(2000);
        setMinIdle(100);
        setMaxWaitMillis(100);  // 等待 100ms，超时抛异常（拒绝服务，保护系统）
    }}
).createPool();

// 每次 Gacha 借用 → 使用 → 归还，对象不进 GC
DrawContext ctx = contextPool.borrowObject();
try {
    return gacha(ctx, userId, poolId);
} finally {
    ctx.reset();           // 清除上次状态
    contextPool.returnObject(ctx);
}
```

**G1 参数配合**：
```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=5            # 目标停顿 5ms（Gacha 场景对延迟敏感）
-XX:G1HeapRegionSize=16m          # 大对象（DrawPool）不进 Humongous Region
-XX:InitiatingHeapOccupancyPercent=35  # 提前触发 Mixed GC，防 Old Gen 堆满
-XX:G1ReservePercent=20           # 预留 20% 防 Evacuation Failure
-Xms4g -Xmx4g                     # 固定堆大小，防动态扩展触发 FGC
```

**效果**：对象池消除了大量短命对象的 Young GC 压力；G1 参数确保即使发生 GC，停顿 < 5ms，用户无感知。

---

## 三、多级缓存防雪崩

```java
// 道具配置：不频繁变化，适合多级缓存
// 本地 Caffeine L1（毫秒级访问）→ Redis L2 → DB（托底）
@Cacheable(value = "itemConfig", key = "#itemId")
public ItemConfig getItemConfig(String itemId) {
    // Spring Cache 抽象：L1 Caffeine 命中则返回
    // L1 未命中 → 查 Redis → 未命中 → 查 DB → 回写 L1+L2
    return itemConfigRepo.findById(itemId);
}

// 双重检测防雪崩（多节点并发查 DB 时只允许一个查询）
public ItemConfig getItemConfigWithLock(String itemId) {
    ItemConfig config = localCache.getIfPresent(itemId);
    if (config != null) return config;
    // L1 未命中，加分布式锁
    RLock lock = redisson.getLock("lock:itemConfig:" + itemId);
    if (lock.tryLock(100, 500, TimeUnit.MILLISECONDS)) {
        try {
            config = redisCache.get(itemId);  // 再查 L2，其他节点可能已写入
            if (config == null) {
                config = db.findById(itemId);
                redisCache.set(itemId, config, 300, TimeUnit.SECONDS);
            }
            localCache.put(itemId, config);
        } finally {
            lock.unlock();
        }
    }
    return config;
}
```

---

## 四、实时对战：房间状态机

```
房间生命周期：
WAITING → READY（所有玩家就绪）→ PLAYING → SETTLING → ENDED
                          ↑
                  超时自动推进（XXL-Job 分片任务扫描超时房间）

// 状态转换加乐观锁，防止并发重复推进
UPDATE game_room
SET status = 'READY', version = version + 1
WHERE room_id = ? AND status = 'WAITING' AND version = ?
```

**用户动作路由**（保证同一房间的操作串行）：
```java
// 按 roomId 路由到固定 EventLoop 线程（Netty）
// userId → roomId → roomId.hashCode() % eventLoopCount → 同一 EventLoop 处理
// 避免多线程并发修改同一房间状态
EventLoop el = eventLoopGroup.next(Math.abs(roomId.hashCode() % GROUP_SIZE));
el.execute(() -> processAction(roomId, userId, action));
```

---

## 五、跨地域延迟优化

**Eson 简历实践**：跨 Asia/NA/EU 地域，服务间通信延迟降低 15%（通过 gRPC/Protobuf 替换 REST/JSON）。

**跨地域延迟优化策略（层层递进）**：

| 层级 | 策略 | 降低延迟 |
|------|------|---------|
| 协议层 | REST → gRPC（HTTP/2 多路复用 + Protobuf 二进制） | 20-30% |
| 连接层 | 长连接池 + TCP `TCP_NODELAY=true`（关闭 Nagle 算法） | 5-10% |
| 路由层 | GeoDNS / Anycast → 用户就近接入最近 Region | 30-50% |
| 数据层 | 非强一致数据在 Region 内缓存（如道具配置、公告） | 消除跨 Region 读 |
| 传输层 | TCP 缓冲区调优：`SO_RCVBUF/SO_SNDBUF = 4MB`（高吞吐跨地域链路） | 10-20% |

**Netty TCP 参数调优**（高延迟跨地域链路）：
```java
ServerBootstrap bootstrap = new ServerBootstrap()
    .group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .childOption(ChannelOption.TCP_NODELAY, true)        // 禁用 Nagle，减少小包延迟
    .childOption(ChannelOption.SO_KEEPALIVE, true)       // TCP keepalive 检测死连接
    .childOption(ChannelOption.SO_RCVBUF, 4 * 1024 * 1024)  // 4MB 接收缓冲（跨地域大带宽延迟积）
    .childOption(ChannelOption.SO_SNDBUF, 4 * 1024 * 1024)  // 4MB 发送缓冲
    .childOption(ChannelOption.WRITE_BUFFER_WATER_MARK,
        new WriteBufferWaterMark(512 * 1024, 1024 * 1024));  // 写缓冲水位（背压）
```

**`TCP_NODELAY` 关键原理**：Nagle 算法会将小包缓积到 MSS 大小才发送，造成额外 ~200ms 延迟（等待窗口满）；游戏实时对战包小而时延敏感，必须关闭 Nagle。

---

## 六、RocksDB 作为结算事务兜底

（适用于：游戏结算写入高峰期，MySQL 行锁竞争导致 P99 高）

```
结算请求 → 先写本地 RocksDB（μs 级，顺序写 LSM） → 立即返回
                    ↓
          异步线程批量读取 RocksDB → 合并写 MySQL（减少行锁争用）
                    ↓
          MySQL 确认写入 → 清理 RocksDB 已处理条目

优势：
- 用户感知延迟 = 写 RocksDB（μs）而非等 MySQL 行锁（ms）
- 极端情况（进程 crash）：重启后 replay RocksDB 未处理条目，数据不丢
```

---

## 七、防作弊机制

| 作弊类型 | 检测 | 防护 |
|---------|------|------|
| 抽奖脚本刷量 | 单用户单分钟请求数异常 | Sentinel 用户级令牌桶限流 |
| 修改客户端概率 | 客户端发送任意结果 | 服务端权威（概率算法只在服务端运行，客户端仅展示） |
| 重放 Gacha 请求 | 重复 orderId | 唯一索引幂等，重复请求返回已有结果 |
| 账号共享抽奖 | 同 IP 多账号 | 设备指纹 + IP 频率限制 |

---

## 八、选型决策树

```
高峰 QPS > 1万，有限量道具？
  → Yes → Redis Lua 原子扣减（必选）
  → 还需要落盘可靠性？→ 本地消息表 + MQ（必选）
  → 有大量短命对象导致 GC？→ 对象池 + G1 调参
  → 有实时对战房间状态？→ 单线程 EventLoop 路由 + 状态机 + 乐观锁
  → 跨地域服务调用？→ gRPC + TCP_NODELAY + 连接池
```
