---
type: concept
status: active
name: "ZAB协议与Zookeeper"
layer: L7
aliases: ["Zookeeper", "ZAB", "ZNode", "Watch", "选主", "分布式协调", "脑裂"]
related:
  - "[[机制-Redis分布式锁]]"
  - "[[机制-AQS]]"
sources:
  - "../../../raw/note/Hollis/Zookeeper/✅Zookeeper如何保证数据的一致性？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅Zookeeper是CP的还是AP的？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅Zookeeper是选举机制是怎样的？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅Zookeeper的数据结构是怎么样的？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅Zookeeper的watch机制是如何工作的？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅如何用Zookeeper实现分布式锁？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅Zookeeper的典型应用场景有哪些？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅Zookeeper集群中的角色有哪些？有什么区别？.md"
  - "../../../raw/note/Hollis/Zookeeper/✅什么是脑裂？如何解决？.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# ZAB协议与Zookeeper

> Zookeeper 是基于 ZAB 协议的 CP 分布式协调服务：所有写操作经 Leader 处理并广播，过半节点确认后提交，保证顺序一致性（而非线性一致性）。

## 第一性原理

分布式系统中多节点需要就某个状态达成共识（谁是 Master、某配置当前值是多少）。Zookeeper 的存在理由：**提供一个强一致的共享存储，让分布式节点可以基于它做协调决策**，而不需要各自实现复杂的共识协议。

## 核心机制

### ZNode 数据模型

```
/（根节点）
├── /services/                    ← 持久节点（服务注册目录）
│   └── /services/user-service/  ← 持久节点
│       └── 192.168.1.1:8080     ← 临时节点（服务实例，session 断开自动删除）
└── /locks/order-lock/            ← 持久节点（分布式锁目录）
    ├── lock-0000000001           ← 临时顺序节点（持锁者）
    └── lock-0000000002           ← 临时顺序节点（等待者）
```

**4 种节点类型**：

| 类型 | 说明 |
|------|------|
| PERSISTENT | 持久节点，手动删除才消失 |
| PERSISTENT_SEQUENTIAL | 持久顺序节点，自动追加编号 |
| EPHEMERAL | 临时节点，客户端 Session 断开自动删除 |
| EPHEMERAL_SEQUENTIAL | 临时顺序节点（分布式锁的核心）|

### Watch 机制

```
客户端 → 注册 watcher 到 ZkWatcherManager（客户端侧）
          ↓ watcher 信息发送到服务端
服务端 → WatchManager 注册 watcher 到对应 ZNode
          ↓ ZNode 发生变化
服务端 → 通知客户端（一次性通知，消费后即失效）
客户端 → ZkWatcherManager 触发事件处理（processDataChanged/processChildChanged）
```

Watch 是**一次性**的：触发一次后即失效，需要重新注册。用于服务注册发现中的节点变更通知。

### ZAB 协议（类 Raft）

ZAB（Zookeeper Atomic Broadcast）= 原子广播协议，保证所有节点按相同顺序应用写操作。

```
写请求（任意节点）→ 转发给 Leader
Leader → 分配全局唯一 zxid（事务 ID）→ 广播 Proposal 给所有 Follower
过半 Follower → ACK → Leader commit → 通知所有节点提交
```

**CP 还是 AP**：
- **CP**（一致性优先）：存活节点 < N/2 时拒绝写请求；Leader 选举期间整个集群停写
- 保证的是**顺序一致性**（非线性一致性）：所有节点看到的写操作顺序相同，但 Follower 可能短暂读到旧数据（同步延迟）
- 想要线性一致性：调用 `sync()` 强制 Follower 与 Leader 同步后再读

### 集群角色

| 角色 | 职责 |
|------|------|
| Leader | 处理所有写请求，发起投票，更新系统状态；提供读写服务 |
| Follower | 接受客户端请求，转发写请求到 Leader；参与投票；提供读服务 |
| Observer | 不参与投票，只同步 Leader 状态；扩展读能力，不影响写性能 |

### Leader 选举（遵强投强）

1. 节点启动 → 进入 LOOKING 状态，广播投票（包含自己的 `zxid` 和 `sid`）
2. 收到他人投票 → 比较 zxid，**谁的 zxid 大（数据更新）就投谁**；zxid 相同则比 sid
3. 超过半数节点投同一候选人 → 该节点当选 Leader
4. 其他节点同步 Leader 数据后对外服务

### 分布式锁（临时顺序节点）

```
1. 客户端在 /locks/ 下创建临时顺序节点（如 lock-0000000003）
2. 获取 /locks/ 下所有子节点，排序
3. 自己的节点是最小的 → 获取到锁
4. 不是最小 → Watch 前一个节点（避免惊群效应）
5. 前一个节点删除（持锁者释放/宕机）→ Watch 触发 → 重新判断
```

**优点**：客户端宕机后临时节点自动删除，不会死锁  
**缺点**：每次加锁需创建节点并同步到过半节点，性能不如 Redis 分布式锁（`SETNX`）；网络抖动可能误删节点（Session 断开删临时节点）

### 脑裂与防护

**脑裂**：网络分区导致多个 Follower 各自认为自己是 Master。  
**ZK 防护**：过半写原则——分区后少于 N/2 节点的分区无法选出 Leader，自然停止服务，保证只有一个活跃 Leader。

## 关键权衡

1. **写性能低**：所有写必须经 Leader + 过半确认，写 QPS 通常只有数千，不适合高频写场景
2. **数据全内存**：ZNode 存储在内存中（+事务日志），适合小型元数据/配置，不适合大数据量
3. **ZK vs Nacos/Consul**：ZK 是通用分布式协调，不是专为服务发现设计，配置复杂；Nacos/Consul 专为服务注册发现优化，功能更丰富（健康检查、负载均衡）

## 与其他概念的关系

- 与 [[机制-Redis分布式锁]] 对比：Redis 分布式锁（SETNX）性能更高；ZK 临时节点自动删除，天然防死锁；各有适用场景
- Dubbo 使用 ZK 作为服务注册中心（见 L7 Dubbo）
- 选举机制与 [[机制-AQS]] 的 CLH 队列有相通之处：都是"排队"机制

## 应用边界

**适合 ZK**：分布式配置管理（动态读取）；Master 选举；分布式锁（对性能要求不极端）；服务注册发现（Dubbo 默认）。

**不适合 ZK**：高频写场景（用 Redis）；大数据量存储（用数据库）；对高可用要求极高（ZK 选举期间停写）。
