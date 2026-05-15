---
type: concept
status: active
name: "ZAB协议与Zookeeper"
layer: L7
aliases: ["Zookeeper", "ZAB", "ZNode", "Watch", "选主", "分布式协调", "脑裂", "EPHEMERAL", "临时顺序节点", "ZK分布式锁"]
related:
  - "[[概念-Redis]]"
  - "[[机制-AQS]]"
  - "[[概念-分布式理论]]"
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
updated: 2026-05-15
lint_notes: ""
---

# ZAB协议与Zookeeper

> Zookeeper 是基于 ZAB 协议的 CP 分布式协调服务：所有写操作经 Leader 处理并广播，过半节点确认后提交，保证顺序一致性（而非线性一致性）；核心应用场景是分布式锁、配置管理、Master 选举和服务注册发现。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 共享状态协调的根本需求 |
| [二、ZNode 数据模型](#二znode-数据模型) | 四种节点类型与路径结构 |
| [三、Watch 机制](#三watch-机制) | 一次性事件通知原理 |
| [四、ZAB 协议](#四zab-协议) | Leader 广播 + 半数确认，CP vs AP |
| [五、集群角色与 Leader 选举](#五集群角色与-leader-选举) | Leader/Follower/Observer，遵强投强规则 |
| [六、分布式锁实现](#六分布式锁实现) | 临时顺序节点 + 监听前驱节点 |
| [七、脑裂与防护](#七脑裂与防护) | 过半写原则防止脑裂 |
| [八、关键权衡](#八关键权衡) | 写性能低、全内存、ZK vs Nacos |
| [九、与其他概念的关系](#九与其他概念的关系) | Redis 分布式锁对比、AQS |
| [十、应用边界](#十应用边界) | 适合 vs 不适合的场景 |

## 一、第一性原理

分布式系统中多节点需要就某个状态达成共识（谁是 Master、某配置当前值是多少）。Zookeeper 的存在理由：**提供一个强一致的共享存储，让分布式节点可以基于它做协调决策**，而不需要各自实现复杂的共识协议。

简单说：ZK 是分布式系统的"公告牌"——任何节点都可以读，但写操作由 Leader 串行化，保证所有节点看到的数据顺序一致。

## 二、ZNode 数据模型

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

| 类型 | 说明 | 典型用途 |
|------|------|---------|
| PERSISTENT | 持久节点，手动删除才消失 | 服务目录、配置根节点 |
| PERSISTENT_SEQUENTIAL | 持久顺序节点，自动追加编号 | 有序数据 |
| EPHEMERAL | 临时节点，客户端 Session 断开自动删除 | 服务实例注册 |
| EPHEMERAL_SEQUENTIAL | 临时顺序节点（分布式锁的核心）| 分布式锁排队 |

**EPHEMERAL 的关键价值**：客户端宕机时 Session 超时，临时节点自动删除，无需手动清理——这是 ZK 分布式锁天然防死锁的基础。

## 三、Watch 机制

```
客户端 → 注册 watcher 到 ZkWatcherManager（客户端侧）
          ↓ watcher 信息发送到服务端
服务端 → WatchManager 注册 watcher 到对应 ZNode
          ↓ ZNode 发生变化（创建/删除/数据变更/子节点变化）
服务端 → 通知客户端（一次性通知，消费后即失效）
客户端 → ZkWatcherManager 触发事件处理
```

**Watch 是一次性的**：触发一次后即失效，需要重新注册（通常在回调中重新调用 `getData` 或 `getChildren` + 新 watcher）。

**Watch 事件类型**：
- NodeCreated / NodeDeleted / NodeDataChanged / NodeChildrenChanged
- 分布式锁中：监听前驱节点的 NodeDeleted 事件

## 四、ZAB 协议

ZAB（Zookeeper Atomic Broadcast）= 原子广播协议，保证所有节点按相同顺序应用写操作。

```
写请求（任意节点）→ 转发给 Leader
Leader → 分配全局唯一 zxid（事务 ID）→ 广播 Proposal 给所有 Follower
过半 Follower → ACK → Leader commit → 通知所有节点提交
```

**CP 还是 AP**：
- **CP**（一致性优先）：存活节点 < N/2 时拒绝写请求；Leader 选举期间整个集群停写
- 保证的是**顺序一致性**（非线性一致性）：所有节点看到的写操作顺序相同，但 Follower 可能短暂读到旧数据（同步延迟）
- 想要线性一致性：调用 `sync()` 强制 Follower 与 Leader 同步后再读（有性能代价）

**ZAB vs Raft**：两者思想相同（Leader + 多数确认），ZAB 早于 Raft，ZK 使用 ZAB；etcd/Consul/TiKV 使用 Raft。

## 五、集群角色与 Leader 选举

### 集群角色

| 角色 | 职责 |
|------|------|
| Leader | 处理所有写请求，发起投票，更新系统状态；提供读写服务 |
| Follower | 接受客户端请求，转发写请求到 Leader；参与投票；提供读服务 |
| Observer | 不参与投票，只同步 Leader 状态；扩展读能力，不影响写性能 |

### Leader 选举（遵强投强）

1. 节点启动 → 进入 LOOKING 状态，广播投票（包含自己的 `zxid` 和 `sid`）
2. 收到他人投票 → 比较 zxid，**谁的 zxid 大（数据更新）就投谁**；zxid 相同则比 sid（节点 ID 大者优先）
3. 超过半数节点投同一候选人 → 该节点当选 Leader
4. 其他节点同步 Leader 数据后对外服务

**zxid 优先的原因**：持有最新 zxid 的节点数据最完整，保证已提交的事务不丢失（等同于 Raft 的"日志最新者赢"）。

## 六、分布式锁实现

```
1. 客户端在 /locks/ 下创建临时顺序节点（如 lock-0000000003）
2. 获取 /locks/ 下所有子节点，排序
3. 自己的节点是最小的 → 获取到锁，开始执行业务
4. 不是最小 → Watch 前一个节点（避免惊群效应）
5. 前一个节点删除（持锁者释放/宕机）→ Watch 触发 → 重新判断
```

**关键设计**：
- 监听**前驱节点**而非父节点，避免惊群效应（一个节点删除通知所有等待者）
- 临时顺序节点保证客户端宕机时自动释放锁（节点自动删除）

**优点**：客户端宕机后临时节点自动删除，不会死锁；公平锁（按 zxid 顺序排队）

**缺点**：每次加锁需创建节点并同步到过半节点，性能不如 Redis 分布式锁（`SETNX`）；网络抖动可能导致 Session 超时误删节点

## 七、脑裂与防护

**脑裂**：网络分区导致两组 Follower 各自认为自己可以选出新 Leader，出现两个 Leader 同时对外服务。

**ZK 防护**：过半写原则——分区后少于 N/2 节点的分区无法选出 Leader（无法获得多数票），自然停止服务，保证只有一个活跃 Leader。

| 集群规模 | 允许宕机节点数 |
|---------|-------------|
| 3 节点 | 1 个 |
| 5 节点 | 2 个 |
| 7 节点 | 3 个 |

**推荐奇数节点**：奇数和偶数相比，容错能力相同但节点数更少（4节点和3节点都只能容忍1个故障）。

## 八、关键权衡

| 维度 | 结论 |
|------|------|
| 写性能 | 所有写必须经 Leader + 过半确认，写 QPS 通常只有数千，不适合高频写场景 |
| 存储容量 | ZNode 存储在内存中（+事务日志），适合小型元数据/配置，不适合大数据量 |
| ZK vs Nacos | ZK 是通用分布式协调，不是专为服务发现设计，配置复杂；Nacos 专为服务注册发现优化，同时支持 AP+CP |
| ZK vs Redis 分布式锁 | Redis SETNX 性能更高（毫秒级）；ZK 临时节点天然防死锁（Session 断开自动删除）|

## 九、与其他概念的关系

- 与 [[概念-Redis]] 对比：Redis 分布式锁（SETNX）性能更高（毫秒级 vs 十毫秒级）；ZK 临时节点自动删除，天然防死锁；强一致性要求高选 ZK，性能要求高选 Redis
- 选举机制与 [[机制-AQS]] 的 CLH 队列有相通之处：都是"排队"机制——ZK 用临时顺序节点排队，AQS 用 CLH 双向队列排队
- 依赖 [[概念-分布式理论]]：ZAB 协议是 Paxos 的简化变体，保证顺序一致性（CP）；Dubbo/Elastic-Job 的注册中心使用 ZK

## 十、应用边界

**适合 ZK**：
- 分布式配置管理（Watch 机制动态感知变更）
- Master 选举（EPHEMERAL 节点 + 临时有序，只有一个节点能创建特定临时节点）
- 分布式锁（对性能要求不极端，优先考虑 Redisson）
- 服务注册发现（Dubbo 默认注册中心）

**不适合 ZK**：
- 高频写场景（用 Redis）
- 大数据量存储（用数据库）
- 对高可用要求极高（ZK 选举期间停写，通常 10~30 秒）
- 新项目服务发现（优先考虑 Nacos，功能更丰富，运维更简单）
