---
type: concept
status: active
name: "Seata"
layer: L7
aliases: ["Seata", "AT模式", "TCC模式", "分布式事务框架", "全局事务", "TC", "TM", "RM", "UNDO_LOG", "空回滚", "悬挂", "XA模式", "Saga模式"]
tags: ["#distributed"]
related:
  - "[[概念-分布式理论]]"
  - "[[机制-Spring]]"
  - "[[概念-幂等设计]]"
---

# Seata框架机制

> Seata（Simple Extensible Autonomous Transaction Architecture）是阿里开源的分布式事务解决方案，通过 TC/TM/RM 三组件协调本地事务，支持 AT/TCC/Saga/XA 四种模式，解决微服务架构中跨服务的数据一致性问题。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、Seata 要解决的跨库事务问题](#一seata-要解决的跨库事务问题) | 分布式事务的根本难题，Seata 的核心思想 |
| [二、三组件架构](#二三组件架构) | TC/TM/RM 职责与执行流程 |
| [三、AT 模式](#三at-模式) | before/after image + UNDO_LOG，一阶段提交 |
| [四、TCC 模式](#四tcc-模式) | 空回滚、悬挂问题与统一解法 |
| [五、四种模式对比](#五四种模式对比) | XA/AT/TCC/Saga 一致性、性能、侵入性 |
| [六、AT vs XA 核心区别](#六at-vs-xa-核心区别) | 锁时机、一致性级别、性能差异 |
| [七、Seata 模式取舍](#七seata-模式取舍) | 脏读风险、TCC侵入性、TC单点 |
| [八、Seata 的系统位置](#八seata-的系统位置) | 分布式理论、Spring AOP、幂等设计 |
| [九、Seata 适用边界](#九seata-适用边界) | 各模式适用场景，何时不用 Seata |

## 一、Seata 要解决的跨库事务问题

分布式事务的根本难题：**多个独立本地事务无法形成原子提交**——任意一个失败都可能导致其他已提交事务无法回滚。

Seata 的核心思想：**把分布式事务看作若干本地事务的组合**，引入独立的事务协调器（TC）维护全局状态，强制所有参与者（RM）执行两阶段提交，从而在本地事务的 ACID 基础上构建跨服务的一致性保证。

## 二、三组件架构

| 组件 | 角色 | 职责 |
|------|------|------|
| **TC** (Transaction Coordinator) | 独立 JVM 进程，无业务代码 | 维护全局事务状态；协调所有 RM 执行提交或回滚 |
| **TM** (Transaction Manager) | 聚合服务（如 TradeCenter） | 开启/提交/回滚全局事务；持有 XID |
| **RM** (Resource Manager) | 各个微服务（Order/Stock/Account） | 注册事务分支；执行本地事务并上报结果给 TC |

**全局事务执行流程**：
```
1. TM 接收用户请求 → 调用 TC 创建全局事务，获取 XID
2. TM 通过 RPC/REST 调用各 RM，将 XID 一并传递
3. 每个 RM 收到 XID → 将本次操作注册为 Branch Transaction
4. TM 调用链全部结束 → 评估结果：
     所有成功 → 决议 Commit
     有失败/超时 → 决议 Rollback
5. TM 通知 TC → TC 协调所有 RM 执行二阶段动作
```

## 三、AT 模式

**无侵入**：业务只需将 `@Transactional` 替换为 `@GlobalTransactional`。

核心手段：**代理数据源**——用 `SeataDataSourceProxy` 包装原始 DataSource，在 JDBC 层拦截 SQL 执行，实现透明的两阶段控制。

**一阶段**：
```
1. 解析 SQL → 获取类型、表名、WHERE 条件
2. 执行前查询 → 生成 before image（变更前数据快照）
3. 执行业务 SQL → 数据库变更
4. 执行后查询 → 生成 after image（变更后数据快照）
5. before/after image + 业务 SQL → 写入 UNDO_LOG 表
6. 向 TC 注册分支，申请行排他锁
7. 将业务数据更新 + UNDO_LOG 在同一本地事务中提交（ACID 保证两者同时落盘）
8. 上报结果给 TC
```

**二阶段**：
- **提交**：TC 通知 RM 释放全局锁 → 异步删除 UNDO_LOG（轻量）
- **回滚**：TC 通知 RM → 读取 UNDO_LOG → 比对当前数据与 after image：
  - 一致 → 用 before image 生成反向 SQL 执行回滚
  - 不一致（脏数据）→ 抛出异常，需人工处理

## 四、TCC 模式

**有侵入**：业务需显式实现 Try / Confirm / Cancel 三个接口。

适合多数据源场景（MySQL + Redis + ES 混合事务），性能极高（无全局锁）。

**两大经典问题与解法**：

| 问题 | 根因 | 表现 |
|------|------|------|
| **空回滚** | Try 超时/失败，Cancel 被调用，但该参与者从未执行 Try | Cancel 找不到 Try 记录，异常报错或无限失败 |
| **悬挂** | 网络延迟导致 Cancel 先于 Try 到达，空回滚完成后 Try 才执行 | Try 占用的资源无人释放，事务永久悬挂 |

**统一解法：分布式事务记录表**

```sql
-- distribute_transaction(tx_id, state, ...)
-- state: trying → confirmed/cancelled
```

- **空回滚处理**：Cancel 时查 `tx_id` 无记录 → 写入 `state=cancelled` + 空回滚（不抛异常）
- **悬挂处理**：Try 时查 `tx_id` 已有 `state=cancelled` 记录 → 直接拒绝 Try（或返回成功）
- **幂等保证**：每次 Try/Confirm/Cancel 前先查表，已处理则直接返回

## 五、四种模式对比

| 维度 | XA | AT | TCC | Saga |
|------|----|----|-----|------|
| 一致性 | 强一致 | 最终一致 | 最终一致 | 最终一致 |
| 隔离性 | 完全隔离 | 基于全局锁隔离 | 基于资源预留隔离 | 无隔离 |
| 代码侵入 | 无 | 无 | 有（实现3个接口） | 有（状态机+补偿代码） |
| 性能 | 差（一阶段锁资源） | 较高（一阶段提交） | 非常高 | 非常高 |
| 适用场景 | 强一致性要求场景 | 关系型DB大多数场景 | 高性能、多数据源 | 长事务、外部接口调用 |

## 六、AT vs XA 核心区别

| 维度 | AT | XA |
|------|----|----|
| 一阶段 | 直接提交本地事务（释放行锁）| 持有资源锁等到二阶段决议 |
| 回滚方式 | 通过 UNDO_LOG 反向 SQL 补偿 | 数据库原生 Rollback |
| 一致性 | 最终一致（一阶段提交后可能被读到）| 强一致（二阶段前不可见）|
| 性能 | 好（行锁时间短）| 差（行锁从一阶段持续到二阶段）|
| 适用 | 大多数微服务跨 DB 场景 | 必须强一致的金融核心场景 |

## 七、Seata 模式取舍

1. **AT 模式的脏读风险**：一阶段已提交，二阶段未完成期间，其他事务可读到"提交但可能被回滚"的数据；需通过全局锁（`SELECT ... FOR UPDATE` 携带 XID）实现读隔离
2. **TCC 侵入性**：业务必须保证 Confirm/Cancel 幂等；分布式事务记录表引入了额外的写操作，是性能和可靠性的折中
3. **TC 单点风险**：TC 是独立服务，需做高可用部署（Seata Server 集群 + Raft/DB 模式存储全局事务状态）
4. **Saga 无隔离**：并发执行的事务可能相互干扰，仅适合对隔离性要求低的长流程场景

## 八、Seata 的系统位置

- 基于 [[机制-Spring]] 扩展：`@GlobalTransactional` 是 `@Transactional` 的分布式版，底层仍走 Spring AOP 代理
- 幂等控制参考 [[概念-幂等设计]]：TCC 的分布式事务记录表是"一锁、二判、三更新"的工程实践
- 理论基础来自 [[概念-分布式理论]]：Seata 本质是 2PC 的工程化实现，AT 为最终一致（BASE），XA 为强一致（ACID）

## 九、Seata 适用边界

**适合 Seata AT**：同构关系型数据库、改造成本低（只加注解）、对性能要求中等的场景。

**适合 TCC**：异构数据源（MySQL+Redis混合）、高并发（无全局锁）、团队能接受侵入性开发。

**适合 Saga**：调用外部第三方接口（如微信支付）的长流程事务，无法做 TCC 的遗留系统集成。

**不适合 Seata**：超高并发核心链路（分布式事务本身有协调开销，极致性能场景考虑本地消息表 + 最终一致）。
