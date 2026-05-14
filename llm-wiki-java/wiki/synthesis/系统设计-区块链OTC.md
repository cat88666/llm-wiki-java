# 系统设计-区块链OTC

> 基于 bitcoinj + web3j 的场外加密货币交易平台设计，核心挑战是私钥安全、链上确认异步对账、资金原子操作三大问题。
>
> 来源：`../../raw/note/Interview/Eson.md`（CoinsOTC 项目）

---

## 一、比特币 vs 以太坊账户模型

理解两条链的根本差异是设计冷热钱包和对账的前提。

### BTC：UTXO 模型

```
UTXO = Unspent Transaction Output（未花费的交易输出）

比特币不存在"账户余额"，只有 UTXO 集合。
每笔交易 = inputs（引用已有 UTXO）+ outputs（创建新 UTXO）

用户"余额" = 其地址控制的所有 UTXO 之和
Change（找零）= 超出支付金额的部分打回自己的找零地址
```

**bitcoinj 核心类**：
```java
WalletAppKit kit = new WalletAppKit(MAINNET, new File("."), "myapp");
kit.startAsync().awaitRunning();
Wallet wallet = kit.wallet();

// 发送交易
SendRequest req = SendRequest.to(toAddress, Coin.parseCoin("0.01"));
req.feePerKb = Transaction.DEFAULT_TX_FEE;
wallet.sendCoins(kit.peerGroup(), req);

// HD 钱包派生新地址（BIP44）：每次充值给用户唯一地址
DeterministicKey key = wallet.freshReceiveKey();
Address depositAddr = key.toAddress(MAINNET);
```

### ETH：账户模型

```
账户类型：
  外部账户（EOA）：由私钥控制，发起交易
  合约账户：由代码控制，只能被调用触发

账户状态：nonce（已发送TX数）+ balance + storageRoot + codeHash
Nonce：每发一笔TX递增，防止重放攻击
```

**web3j 核心类**：
```java
Web3j web3j = Web3j.build(new HttpService("https://mainnet.infura.io/v3/KEY"));
Credentials credentials = Credentials.create(privateKey);

// ERC20 Transfer
Erc20Contract token = Erc20Contract.load(tokenAddress, web3j, credentials, gasPrice, gasLimit);
TransactionReceipt receipt = token.transfer(toAddress, amount).send();

// 查余额
EthGetBalance balance = web3j.ethGetBalance(address, DefaultBlockParameterName.LATEST).send();
```

---

## 二、冷热钱包架构

```
┌─────────────────────────────────────────┐
│              热钱包（在线）               │
│  小额日常充提 < 热钱包阈值（如 1 BTC）   │
│  私钥 = AES-256-GCM 加密存 DB            │
│  使用时解密，用完立即 Arrays.fill(0)     │
└─────────────────────────────────────────┘
           ↑ 超阈值触发自动归集 ↓
┌─────────────────────────────────────────┐
│              冷钱包（离线）              │
│  大额储备资产                            │
│  私钥从不上网（air-gapped 硬件设备）      │
│  签名流程：                              │
│    热钱包生成未签名 PSBT（BTC）           │
│    导出到冷设备 → 离线签名 → 导回广播    │
└─────────────────────────────────────────┘
```

**私钥安全实现**：
```java
// 加载时解密，使用完立即清零
byte[] keyBytes = aesDecrypt(encryptedKey);
try {
    ECKey key = ECKey.fromPrivate(keyBytes);
    // ... 签名操作
} finally {
    Arrays.fill(keyBytes, (byte) 0);  // 防止 Heap Dump 泄露
    key = null;
}
```

**热→冷归集触发逻辑**：
```java
@Scheduled(cron = "0 0 2 * * *")  // 每天凌晨2点
public void consolidateToCold() {
    BigDecimal hotBalance = getHotBalance();
    if (hotBalance.compareTo(HOT_THRESHOLD) > 0) {
        BigDecimal transferAmount = hotBalance.subtract(HOT_THRESHOLD);
        // 提交人工审批流 → 审批通过后才真正广播 TX
        submitColdTransferApproval(coldWalletAddress, transferAmount);
    }
}
```

---

## 三、P2P OTC 撮合引擎

```
角色：
  买家（Buyer）：有法币，想买加密货币
  卖家（Seller）：有加密货币，想换法币

流程：
  1. 卖家挂单：数量 + 单价 + 支持的支付方式
  2. 买家下单：选择卖家订单，锁定资金
  3. 平台托管：卖家的加密货币转入平台托管地址（escrow）
  4. 买家付款：通过银行/支付宝/微信向卖家转账法币（链下）
  5. 卖家确认：确认收到法币 → 平台从 escrow 释放加密货币给买家
  6. 超时仲裁：买家 N 分钟不付款则订单取消；付款后 M 分钟卖家不确认则进人工仲裁
```

**Escrow 逻辑（以 ETH 为例）**：
```java
// 1. 撮合成功：将卖家 ETH 转入平台托管地址
String escrowTxHash = transferToEscrow(sellerAddress, escrowAddress, amount);
order.setEscrowTxHash(escrowTxHash);
order.setStatus(OrderStatus.ESCROW_LOCKED);

// 2. 买家付款 + 卖家确认 → 释放
String releaseTxHash = releaseFromEscrow(escrowAddress, buyerAddress, amount);
order.setReleaseTxHash(releaseTxHash);
order.setStatus(OrderStatus.COMPLETED);
```

**Disruptor 单线程无锁撮合**（高并发下避免订单簿并发修改）：
```java
// 订单簿操作全部经过 Disruptor，单线程处理，天然串行
Disruptor<OrderEvent> disruptor = new Disruptor<>(
    OrderEvent::new, 1024, DaemonThreadFactory.INSTANCE);
disruptor.handleEventsWith(new MatchingEngine(orderBook));
disruptor.start();
```

---

## 四、充值（Deposit）异步对账流程

```
链上事件 ──→ 监听器 ──→ 待确认记录 ──→ 定时扫描 ──→ 达到阈值 ──→ 入账
                                            ↑
                                       每 5min 轮询

BTC 确认阈值：6 confirmations（约 60min）
ETH 确认阈值：12-20 confirmations（约 3-5min）
```

**bitcoinj 监听**：
```java
wallet.addCoinsReceivedEventListener((wallet, tx, prevBalance, newBalance) -> {
    // 仅记录"待确认"，不立即入账
    String txHash = tx.getTxId().toString();
    depositRepo.save(new PendingDeposit(userId, txHash, amount, 0));
});
```

**定时对账扫描**：
```java
@Scheduled(fixedDelay = 300_000)  // 每5分钟
public void reconcileDeposits() {
    List<PendingDeposit> pending = depositRepo.findByStatus(PENDING);
    for (PendingDeposit d : pending) {
        int confirms = bitcoinj.getTransactionConfirms(d.getTxHash());
        if (confirms >= REQUIRED_CONFIRMS) {
            // 通过 MQ 发送入账事件，消费端做幂等
            mq.send("deposit.confirmed", new DepositEvent(d.getUserId(), d.getAmount(), d.getTxHash()));
            d.setStatus(CONFIRMING);
        }
    }
}

// 消费端幂等：txHash 唯一索引防重复入账
@Transactional
public void onDepositConfirmed(DepositEvent event) {
    if (depositRepo.existsByTxHash(event.getTxHash())) return; // 幂等检查
    userBalanceService.credit(event.getUserId(), event.getAmount());
    depositRepo.updateStatus(event.getTxHash(), CREDITED);
}
```

---

## 五、提现（Withdrawal）流程

```
用户申请提现
    → 冻结用户余额（available -= amount; frozen += amount）
    → 风控审核（金额阈值/频率/黑名单）
    → 自动/人工审批
    → 构建交易 → 热钱包签名 → 广播
    → 记录 txHash，状态 = BROADCASTING
    → 定时轮询确认数
    → 达阈值：状态 = COMPLETED，清零 frozen
    → 超时未确认：Fee Bump（BTC RBF / ETH gas bump）
```

**幂等保障**：
```java
// DB 建唯一索引 (withdrawal_id, direction)
// 广播前查重，防止网络重试导致二次广播
CREATE UNIQUE INDEX uk_withdrawal ON withdrawal_tx(withdrawal_id);
```

---

## 六、资金安全三道防线

| 防线 | 措施 |
|------|------|
| 代码层 | Redisson 分布式锁（`userId + operation`），Lua 原子操作，私钥内存清零 |
| DB 层 | 唯一索引（txHash/withdrawal_id），余额 CHECK 约束，流水表 append-only |
| 监控层 | 热钱包余额监控（低于阈值告警），异常大额提现告警，链上对账差异告警 |

---

## 七、面试 STAR 模板

**Q：CoinsOTC 最难的点是什么？**

> **S**：平台资金安全是区块链 OTC 的命脉，一旦资金出现差错（超提、重复入账、私钥泄露）损失无法追回。
>
> **T**：需要在高并发下保证余额操作原子性，同时处理链上异步确认的数据一致性。
>
> **A**：①Redisson 分布式锁保障同一用户的余额操作串行；②txHash 唯一索引在 DB 层做最终幂等防护；③充值采用"监听→待确认→定时对账→MQ 入账"四步异步流程，与链上确认解耦；④私钥 AES 加密存 DB，使用时解密，用完 Arrays.fill(0) 清零。
>
> **R**：上线后资金零损失，充值确认平均延迟 < 30s（ETH），对账差异率 0%。
