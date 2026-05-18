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
updated: 2026-05-18
---

# 系统设计-区块链OTC

> 区块链 OTC 的最高约束是资金零损失：私钥不能泄露、链上确认不能误判、同一 txHash 不能重复入账、提现不能超提；技术选型默认是 **冷热钱包隔离 + 链上监听/对账 + 余额冻结状态机 + DB 唯一索引兜底**。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、OTC 业务闭环：挂单、托管、付款、放币、对账](#一otc-业务闭环挂单托管付款放币对账) | 核心流程、BTC/ETH 模型差异 |
| [二、钱包安全：热钱包在线签名，冷钱包离线托管](#二钱包安全热钱包在线签名冷钱包离线托管) | 私钥加密、KMS、归集阈值 |
| [三、撮合与托管：订单簿选型和 Escrow 状态机](#三撮合与托管订单簿选型和-escrow-状态机) | Disruptor vs 分布式撮合、托管流转 |
| [四、链上充值：确认数、分叉风险与幂等入账](#四链上充值确认数分叉风险与幂等入账) | 监听、待确认、MQ、唯一索引 |
| [五、链上提现：冻结、审批、广播、Fee Bump](#五链上提现冻结审批广播fee-bump) | 提现状态机、Nonce/UTXO、重试 |
| [六、资金安全防线：锁只是加速，DB 才是裁决者](#六资金安全防线锁只是加速db-才是裁决者) | 代码层、DB 层、监控层 |
| [七、区块链 OTC 决策树](#七区块链-otc-决策树) | 钱包、撮合、确认数、锁选型 |
| [八、面试追问](#八面试追问) | 私钥、链上超时、RedLock、UTXO |
| [九、OTC 与加密、Redis、支付系统的关系](#九otc-与加密redis支付系统的关系) | 加密脱敏、幂等、余额模型 |

## 一、OTC 业务闭环：挂单、托管、付款、放币、对账

```
卖家挂广告/挂单
  → 买家下单，订单锁定价格和数量
  → 卖家币进入平台 Escrow/平台冻结卖家币
  → 买家线下支付法币
  → 卖家确认收款
  → 平台放币给买家
  → 链上确认/内部账务入账
  → 日终链上余额、平台账、用户账三方对账
```

| 不变量 | 破坏后果 | 防线 |
|--------|----------|------|
| 私钥不能明文落盘/长时间留在内存 | 资产不可追回 | AES-GCM/KMS + 使用后清零 |
| 充值少确认不能入账 | 分叉/双花导致资损 | 确认阈值 + 待确认状态 |
| 同一 txHash 只能入账一次 | 重复给用户加钱 | `uk_chain_tx(direction, tx_hash)` |
| 提现不能超出可用余额 | 平台垫资 | 余额冻结 + DB 条件更新 |
| 放币必须绑定订单状态 | 争议/重复放币 | Escrow 状态机 + 审计流水 |

BTC 与 ETH 对系统设计的影响：

| 维度 | BTC UTXO | ETH 账户模型 | 结论 |
|------|----------|--------------|------|
| 余额 | UTXO 集合 | 账户余额 | BTC 提现需选币和找零，ETH 直接扣账户 |
| 防重 | UTXO 消费即失效 | Nonce 单调递增 | ETH 必须集中管理 Nonce |
| 地址 | HD 钱包可一单一地址 | 常见一用户一地址 | BTC 地址映射和扫描更复杂 |
| 确认 | 常用 6 确认 | 常用 12-20 确认 | 阈值按链和资产风险配置 |

## 二、钱包安全：热钱包在线签名，冷钱包离线托管

```
用户充提小额资产
  → 热钱包（在线，保留 < 10% 资产）
       ├─ 私钥加密存储，签名时短暂解密
       └─ 自动归集/补充，满足日常流动性

大额储备资产
  → 冷钱包（离线，保留 > 90% 资产）
       ├─ 未签名交易导出
       ├─ 离线设备签名
       └─ 签名交易导回广播
```

| 密钥层级 | 存放位置 | 作用 | 结论 |
|----------|----------|------|------|
| 私钥明文 | 只在签名内存短暂存在 | 链上签名 | 用完必须清零 |
| DEK | DB 中加密保存 | 加密私钥 | 拖库拿不到明文私钥 |
| KEK | KMS/环境注入 | 加密 DEK | 生产推荐 KMS，不落盘 |

```java
byte[] keyBytes = aesGcmDecrypt(encryptedPrivateKey, dek);
try {
    ECKey key = ECKey.fromPrivate(keyBytes);
    return signer.sign(rawTx, key);
} finally {
    Arrays.fill(keyBytes, (byte) 0);
}
```

反直觉：`String` 不适合存私钥明文，因为不可变对象无法主动清零；私钥明文应使用 `byte[]`，签名后立即覆盖。

## 三、撮合与托管：订单簿选型和 Escrow 状态机

| 撮合方案 | 优点 | 代价 | 结论 |
|----------|------|------|------|
| 单线程 Disruptor | 全局有序、无锁、延迟低 | 单节点容量和可用性边界明显 | 中小 OTC 交易对首选 |
| 按交易对分片撮合 | 可水平扩展 | 分片路由和迁移复杂 | 多交易对规模化后使用 |
| 分布式共享订单簿 | 扩展能力最强 | 一致性和延迟成本高 | OTC 通常不推荐 |

Escrow 订单状态：

```
CREATED ──卖家托管币──▶ ESCROW_LOCKED ──买家标记已付款──▶ PAID
   │                         │                               │
   └─超时取消──▶ CANCELLED    └─托管失败──▶ FAILED             ▼
                         DISPUTING ◀──申诉──── SELLER_CONFIRMED
                              │                         │
                              └────仲裁────▶ RELEASED ◀─┘
```

```sql
UPDATE otc_order
SET status = 'RELEASED', release_tx_hash = #{txHash}, version = version + 1
WHERE order_id = #{orderId}
  AND status IN ('SELLER_CONFIRMED', 'DISPUTING')
  AND version = #{version};
```

## 四、链上充值：确认数、分叉风险与幂等入账

```
链上监听发现 tx
  → 写 pending_deposit(txHash, confirms=0, status=PENDING)
  → 定时任务轮询确认数
  → 达阈值后发送 deposit.confirmed MQ
  → 消费端写入账流水 + 增加用户余额
  → 更新 deposit=CREDITED
  → 日终按链上 tx 与平台流水对账
```

```sql
CREATE UNIQUE INDEX uk_deposit_tx
ON wallet_deposit(chain, tx_hash, asset);
```

```java
@Transactional
public void credit(DepositConfirmed event) {
    int inserted = depositFlowMapper.insertIgnore(event.chain(), event.txHash(), event.amount());
    if (inserted == 0) return;
    walletMapper.addAvailable(event.userId(), event.asset(), event.amount());
    depositMapper.updateStatus(event.txHash(), "CREDITED");
}
```

为什么不能监听到就入账：链上可能分叉，少确认交易可能从主链消失；确认数越高，被回滚概率越低。资金系统宁可慢入账，也不能误入账。

## 五、链上提现：冻结、审批、广播、Fee Bump

```
用户申请提现
  → DB 条件更新冻结余额（available -= amount, frozen += amount）
  → 风控/人工审批
  → 构建交易（BTC 选 UTXO；ETH 分配 Nonce）
  → 热钱包签名并广播
  → 记录 txHash，状态 BROADCASTED
  → 轮询确认数
  → 成功：frozen 扣减，状态 COMPLETED
  → 超时：BTC RBF / ETH 同 Nonce 高 Gas 替换
  → 失败：解冻余额，状态 FAILED
```

提现幂等关键约束：

| 环节 | 幂等键 | 防什么 |
|------|--------|--------|
| 申请提现 | `withdrawal_id` | 用户/网络重复提交 |
| 广播交易 | `withdrawal_id` 唯一 | 重试导致二次广播 |
| ETH Nonce | `address + nonce` 唯一 | 并发提现 Nonce 冲突 |
| Fee Bump | `withdrawal_id + bump_round` | 重复加速 |

```sql
UPDATE wallet
SET available = available - #{amount},
    frozen = frozen + #{amount},
    version = version + 1
WHERE user_id = #{userId}
  AND asset = #{asset}
  AND available >= #{amount}
  AND version = #{version};
```

## 六、资金安全防线：锁只是加速，DB 才是裁决者

| 防线 | 措施 | 结论 |
|------|------|------|
| 代码层 | Redisson 锁、签名内存清零、Nonce 串行分配 | 降低并发冲突，但不能作为唯一正确性来源 |
| DB 层 | 唯一索引、余额条件更新、流水 append-only | 资金正确性的最终裁决者 |
| 链上层 | 确认阈值、RBF/Gas 加速、链上扫描 | 解决链上异步和不确定性 |
| 监控层 | 热钱包阈值、异常提现、账实差异 | 快速发现资损风险 |

RedLock 结论：可以减少单 Redis 故障导致的并发问题，但它不能替代 DB 幂等。资金链路必须让唯一索引、状态机和流水表承担最终一致性裁决。

## 七、区块链 OTC 决策树

```
资产是否可在线自动签名？
  ├─ 小额高频 → 热钱包，KMS + 风控限额
  └─ 大额储备 → 冷钱包，离线签名 + 人工审批

充值是否可立即入账？
  ├─ 否 → 待确认 + 确认阈值 + MQ 幂等入账
  └─ 仅测试链/低风险资产 → 可降低确认数，但仍要唯一索引

撮合量是否超过单节点能力？
  ├─ 否 → Disruptor 单线程撮合
  └─ 是 → 按交易对/币种分片撮合

提现超时未确认？
  ├─ BTC → RBF 替换更高手续费交易
  └─ ETH → 同 Nonce 广播更高 Gas 交易
```

## 八、面试追问

### 1. OTC 最难的技术点是什么？

**资金安全和链上异步一致性**。私钥泄露、重复入账、超额提现都不可逆。我的设计会用冷热钱包隔离保护资产，用链上确认阈值避免分叉误入账，用 DB 唯一索引和余额条件更新做资金最终裁决。

---

### 2. 链上交易长时间未确认怎么办？

BTC 用 RBF 重新广播相同输入、更高手续费的交易；ETH 用相同 Nonce、更高 Gas 的交易替换。系统上要记录加速轮次并做幂等，避免多个任务同时 Fee Bump。

---

### 3. BTC UTXO 和 ETH 账户模型对系统有什么影响？

BTC 充值需要维护地址到用户的映射，提现要选 UTXO 和处理找零；ETH 余额查询更直接，但必须严格管理 Nonce，避免并发提现使用同一 Nonce。两者共同点是 txHash 入账必须唯一。

---

### 4. Redisson 主节点宕机会不会导致资金错账？

锁失效可能导致并发进入，但不应该导致错账。资金系统的最终防线是 DB：余额条件更新防超提，`txHash`/`withdrawal_id` 唯一索引防重复处理，流水表 append-only 便于对账。锁只提升并发体验，不承担最终正确性。

---

### 5. DEK/KEK 分层怎么讲？

DEK 加密私钥，密文和加密后的 DEK 可存 DB；KEK 加密 DEK，生产上由 KMS 管理，应用通过 IAM 临时获取解密能力。拖库只能拿到密文，没有 KMS 权限不能还原私钥。

## 九、OTC 与加密、Redis、支付系统的关系

- [[机制-加密与脱敏]]：私钥 AES-GCM 加密、KMS 分层、内存清零是加密脱敏在资金系统中的高风险实践。
- [[概念-幂等设计]]：充值 `txHash`、提现 `withdrawal_id`、Fee Bump 轮次都必须有幂等键。
- [[概念-Redis]]：Redisson 锁可降低同用户并发余额操作冲突，但不能替代 DB 唯一索引。
- [[系统设计-支付系统]]：余额冻结、支付状态机、对账模型可复用；区块链版额外处理链上确认和私钥安全。
