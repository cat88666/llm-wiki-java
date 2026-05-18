---
type: synthesis
status: active
name: "区块链OTC系统"
layer: L8
aliases: ["区块链OTC", "P2P场外交易", "冷热钱包", "UTXO", "私钥安全", "链上对账", "Escrow", "Disruptor撮合"]
tags: ["#practice", "#distributed"]
related:
  - "[[机制-加密与脱敏]]"
  - "[[概念-幂等设计]]"
  - "[[概念-Redis]]"
  - "[[系统设计-支付系统]]"
---

# 系统设计-区块链OTC

> 区块链 OTC 系统的三大核心工程挑战：**私钥安全**（AES-256-GCM 加密存储 + Arrays.fill 清零）、**链上异步确认**（监听→待确认→定时对账→MQ 入账）、**资金原子操作**（分布式锁 + 唯一索引双重防护）；所有操作必须以"资金零损失"为最高约束。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、场景概述与问题拆解](#一场景概述与问题拆解) | BTC/ETH 模型差异、三大挑战 |
| [二、冷热钱包架构](#二冷热钱包架构) | 私钥加密、归集阈值、离线签名 |
| [三、P2P OTC 撮合引擎](#三p2p-otc-撮合引擎) | 订单簿、Escrow 流程、Disruptor 无锁 |
| [四、充值异步对账流程](#四充值异步对账流程) | 链上监听、确认阈值、幂等入账 |
| [五、提现流程与安全](#五提现流程与安全) | 冻结→审批→广播→确认、Gas 加速 |
| [六、资金安全三道防线](#六资金安全三道防线) | 代码层/DB层/监控层 |
| [七、面试高频追问](#七面试高频追问) | 私钥安全、链上一致性、资金原子 |
| [八、与其他概念的关系](#八与其他概念的关系) | 加密与脱敏、幂等设计、Redis、支付 |

## 一、场景概述与问题拆解

### BTC vs ETH 账户模型对比

| 维度 | BTC（UTXO 模型）| ETH（账户模型）| 对系统设计的影响 |
|------|--------------|-------------|--------------|
| 余额概念 | 不存在"账户余额"，只有 UTXO 集合 | 账户有明确余额字段 | BTC 需要合并 UTXO，ETH 直接查余额 |
| 地址使用 | 每次充值可用新地址（BIP44 HD 钱包）| 单一地址长期使用 | BTC 地址追踪更复杂 |
| 防重放 | UTXO 被消费后自动失效 | 依赖 Nonce 递增 | ETH 必须管理 Nonce，防止重放 |
| 确认阈值 | 6 个确认（~60 分钟）| 12-20 个确认（~3-5 分钟）| ETH 入账更快 |

**三大核心工程挑战**：

| 挑战 | 风险 | 设计要点 |
|------|------|---------|
| 私钥安全 | 私钥泄露 = 资产全失，不可追回 | AES-256-GCM 加密 + 使用后立即清零 |
| 链上异步确认 | 链上确认时间不确定（分钟到小时）| 异步监听 + 定时轮询 + 幂等入账 |
| 资金原子操作 | 并发导致重复入账/超额提现 | 分布式锁 + DB 唯一索引双重保障 |

## 二、冷热钱包架构

```
┌─────────────────────────────────────┐
│         热钱包（在线）               │
│  日常充提 < 热钱包阈值（如 1 BTC）   │
│  私钥 = AES-256-GCM 加密存 DB       │
│  使用时解密，用完立即 Arrays.fill(0) │
└─────────────────────────────────────┘
           ↑ 超阈值触发自动归集 ↓
┌─────────────────────────────────────┐
│         冷钱包（离线）              │
│  大额储备资产（> 90% 总资产）         │
│  私钥从不上网（air-gapped 硬件设备） │
│  签名流程：生成未签名 PSBT →        │
│   导出到冷设备 → 离线签名 → 导回广播│
└─────────────────────────────────────┘
```

**私钥内存安全实现**：

```java
byte[] keyBytes = aesDecrypt(encryptedKey);  // 解密得到私钥字节
try {
    ECKey key = ECKey.fromPrivate(keyBytes);
    // ... 签名操作（仅在此处使用）
} finally {
    Arrays.fill(keyBytes, (byte) 0);  // 使用后立即清零
    // 为什么必须清零：JVM GC 不保证立即回收，堆内存中的私钥可能在 heap dump 中暴露
}
```

**热→冷归集触发**：

```java
@Scheduled(cron = "0 0 2 * * *")  // 每天凌晨 2 点
public void consolidateToCold() {
    BigDecimal hotBalance = getHotBalance();
    if (hotBalance.compareTo(HOT_THRESHOLD) > 0) {
        BigDecimal amount = hotBalance.subtract(HOT_THRESHOLD);
        // 提交人工审批流，审批通过后才真正广播交易
        submitColdTransferApproval(coldWalletAddress, amount);
    }
}
```

## 三、P2P OTC 撮合引擎

**OTC 交易流程**：

```
1. 卖家挂单：数量 + 单价 + 支持支付方式
2. 买家下单：选择卖家订单，锁定资金
3. 平台托管：卖家加密货币转入 Escrow 地址
4. 买家付款：链下向卖家转账法币（银行/支付宝）
5. 卖家确认：确认收到法币 → 平台从 Escrow 释放加密货币给买家
6. 超时仲裁：买家超时不付款 → 取消；付款超时卖家不确认 → 人工仲裁
```

**Escrow 实现（ETH 为例）**：

```java
// 撮合成功：卖家 ETH 转入平台托管
String escrowTxHash = transferToEscrow(sellerAddress, escrowAddress, amount);
order.setEscrowTxHash(escrowTxHash);
order.setStatus(OrderStatus.ESCROW_LOCKED);

// 卖家确认收款 → 释放给买家
String releaseTxHash = releaseFromEscrow(escrowAddress, buyerAddress, amount);
order.setReleaseTxHash(releaseTxHash);
order.setStatus(OrderStatus.COMPLETED);
```

**Disruptor 单线程无锁撮合**：

```java
// 订单簿操作全部经过 Disruptor 单线程处理，天然串行，无需加锁
Disruptor<OrderEvent> disruptor = new Disruptor<>(
    OrderEvent::new, 1024, DaemonThreadFactory.INSTANCE);
disruptor.handleEventsWith(new MatchingEngine(orderBook));
disruptor.start();
// 所有下单操作发布到 Disruptor RingBuffer，由单线程 MatchingEngine 处理
```

## 四、充值异步对账流程

**链上确认阈值**：

| 链 | 确认阈值 | 原因 |
|----|---------|------|
| BTC | 6 个确认（~60 分钟）| 算力攻击成本高，需等更多确认 |
| ETH | 12-20 个确认（~3-5 分钟）| 区块时间短，确认更快 |

**充值四步异步流程**：

```java
// 步骤 1：链上事件监听（只记录"待确认"，不立即入账）
wallet.addCoinsReceivedEventListener((wallet, tx, prevBalance, newBalance) -> {
    depositRepo.save(new PendingDeposit(userId, tx.getTxId().toString(), amount, 0));
});

// 步骤 2：定时扫描确认数（每 5 分钟）
@Scheduled(fixedDelay = 300_000)
public void reconcileDeposits() {
    depositRepo.findByStatus(PENDING).forEach(d -> {
        int confirms = bitcoinj.getTransactionConfirms(d.getTxHash());
        if (confirms >= REQUIRED_CONFIRMS) {
            // 通过 MQ 发送入账事件（Consumer 做幂等）
            mq.send("deposit.confirmed", new DepositEvent(d.getUserId(), d.getAmount(), d.getTxHash()));
            d.setStatus(CONFIRMING);
        }
    });
}

// 步骤 3：MQ 消费端幂等入账（txHash 唯一索引防重复入账）
@Transactional
public void onDepositConfirmed(DepositEvent event) {
    if (depositRepo.existsByTxHash(event.getTxHash())) return;  // 幂等检查
    userBalanceService.credit(event.getUserId(), event.getAmount());
    depositRepo.updateStatus(event.getTxHash(), CREDITED);
}
```

**为什么不能立即入账**：链上存在"分叉"风险，区块可能被孤立（orphan block），确认数越多分叉可能性越低。少于 6 个确认的 BTC 交易存在双花风险。

## 五、提现流程与安全

```
用户申请提现
  → 冻结可用余额（available -= amount, frozen += amount）
  → 风控审核（金额/频率/黑名单）
  → 自动/人工审批
  → 构建交易 → 热钱包签名 → 广播上链
  → 记录 txHash，状态 = BROADCASTING
  → 定时轮询确认数
  → 达阈值：状态 = COMPLETED，清零 frozen
  → 超时未确认：Fee Bump（BTC RBF / ETH gas 加速）
```

**幂等保障**：

```sql
-- 广播前查重，防止网络重试导致二次广播
CREATE UNIQUE INDEX uk_withdrawal ON withdrawal_tx(withdrawal_id);
-- 广播时 INSERT IGNORE，已存在则说明已广播，查询状态返回即可
```

## 六、资金安全三道防线

| 防线 | 措施 | 结论 |
|------|------|------|
| **代码层** | Redisson 分布式锁（userId+operation）、Lua 原子操作、私钥 Arrays.fill 清零 | 防并发超扣和内存泄漏 |
| **DB 层** | 唯一索引（txHash/withdrawal_id）、余额 `CHECK available >= 0`、流水表 append-only | 数据库层最终兜底 |
| **监控层** | 热钱包余额告警（低于阈值）、异常大额提现告警、链上对账差异告警 | 问题发生后快速响应 |

**冷热钱包比例**：热钱包保留 < 10% 资产（满足日常充提需求），> 90% 资产归集到冷钱包（离线硬件签名）。

## 七、面试高频追问

**问**：CoinsOTC 最难的技术挑战是什么？
**答**：资金安全。私钥泄露、超提、重复入账的损失都无法追回。具体解法：①Redisson 分布式锁保障同用户余额操作串行；②txHash 唯一索引在 DB 层做最终幂等防护；③充值采用"监听→待确认→定时对账→MQ 入账"四步异步流程，与链上确认解耦；④私钥 AES-GCM 加密存 DB，使用时解密，用完 Arrays.fill(0) 清零防 heap dump。

**问**：链上交易超时未确认怎么处理？
**答**：Fee Bump 加速：BTC 使用 RBF（Replace By Fee）广播一笔相同输入但更高手续费的新交易，矿工优先打包高手续费交易；ETH 使用相同 Nonce 广播更高 gasPrice 的替换交易。系统层面：设置超时阈值（如 BTC 6 小时、ETH 30 分钟），超时自动触发 Fee Bump；需要幂等处理，防止 Fee Bump 也超时后再次触发。

**问**：Disruptor 单线程撮合和分布式撮合怎么选？
**答**：单线程撮合（Disruptor RingBuffer）：性能极高（百万级 QPS，无锁），全局有序，逻辑简单；缺点是单节点限制吞吐上限，单点故障影响整个撮合服务。分布式撮合：水平扩展能力强，但需要全局订单簿同步，一致性复杂。OTC 场景订单量相对有限，单节点 Disruptor 足够，水平扩展靠数据分片（按交易对分布在不同节点）。

**问**：UTXO 和账户模型对充值对账的影响是什么？
**答**：BTC UTXO 模型下，充值地址可以每次不同（HD 钱包派生），系统需要维护地址→userId 的映射表，对账时扫描所有派生地址的 UTXO。ETH 账户模型下，充值地址固定（或一个 userId 一个地址），对账通过订阅 Transfer 事件（ERC20）或监听账户余额变化。BTC 对账逻辑更复杂，ETH 更简单直接。

**问**：AES 密钥存哪里？DEK/KEK 分层是什么？
**答**：三层管理：①DEK（数据密钥）加密私钥，加密后存 DB；②KEK（密钥加密密钥）加密 DEK，存环境变量，部署时注入，不写代码或配置文件；③生产最终方案 KEK 由 KMS（AWS KMS/阿里云 KMS）管理，应用启动时通过 IAM 获取，KEK 从不落盘。拖库只能拿到加密后的 DEK + 加密私钥，没有 KEK 无法解密。

**问**：Redisson 主节点宕机，分布式锁失效怎么办？
**答**：两层保障：①RedLock——向 3 个独立 Redis 节点加锁，majority 成功才算成功，单节点宕机不影响；②DB 层幂等——链上转账有 txHash，DB 建唯一索引 `(txHash, direction)`，锁失效时二次提交被唯一约束拦截。

**问**：你了解 Kleppmann 批评 RedLock 的观点吗？
**答**：核心观点：RedLock 依赖系统时钟，时钟跳变（NTP 校准/VM 暂停）导致锁 TTL 提前过期，两个客户端同时持锁。解决方案是 Fencing Token——锁附带单调递增 token，存储层只接受比上次更大的 token 写入。我们的场景：RedLock + DB 唯一索引本质就是变相 Fencing，DB 唯一约束扮演最终裁决者。

## 八、与其他概念的关系

- 私钥内存安全（AES-GCM + Arrays.fill）参见 [[机制-加密与脱敏]]：对称加密存储、私钥清零是该页的工程实践
- 充值/提现幂等（txHash 唯一索引）属于 [[概念-幂等设计]]：链上交易幂等是支付系统幂等的区块链版
- 余额冻结和分布式锁依赖 [[概念-Redis]]：Redisson 锁防并发超扣，Redis 缓存热钱包余额
- 支付系统中的余额三态和对账设计参见 [[系统设计-支付系统]]：相同的冻结→扣减模型，区块链版需要额外处理链上异步确认
