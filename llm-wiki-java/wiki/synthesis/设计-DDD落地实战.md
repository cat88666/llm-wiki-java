---
title: 设计-DDD落地实战
tags: [#practice #framework]
layer: L8
---

# DDD 落地实战

> `概念-DDD.md` 讲"是什么"；本页讲"怎么用"——限界上下文如何指导微服务拆分、聚合根粒度如何取舍、四层架构如何落地、常见踩坑点在哪里。

---

## 限界上下文 → 微服务拆分决策

### 拆分流程

```
1. 业务分析：识别核心域（支付）/ 支撑域（风控）/ 通用域（用户中心）
2. 识别限界上下文（BC）：同一概念在不同上下文含义不同
   - "商品"在商品中心 = 商品详情
   - "商品"在库存中心 = 库存 SKU
   - "商品"在订单中心 = 订单行 OrderItem
3. 划定 BC 边界 = 微服务边界
4. 定义上下文映射（Context Map）：BC 之间如何交互
```

### 上下文映射关系

| 关系 | 含义 | 实现方式 |
|------|------|----------|
| 共享内核（Shared Kernel）| 两个 BC 共享部分领域模型 | 独立 common 模块，变更需协商 |
| 客户-供应商（Customer-Supplier）| 下游 BC 依赖上游 BC | API 契约，上游保障 SLA |
| 防腐层（Anti-Corruption Layer, ACL）| 隔离外部不良模型 | 在基础设施层做翻译/适配 |
| 开放主机服务（OHS）| 提供通用协议供他人集成 | RESTful API + Swagger |

**防腐层是最重要的**：当接入遗留系统或第三方支付时，在基础设施层做 ACL 翻译，不让外部概念污染领域层。

### 拆分粒度原则

> "一个 Bounded Context 对应一个能完整负责的团队（康威定律逆用）"

- **太细**：拆出 30 个微服务 → 跨服务分布式事务激增、网络开销大
- **太粗**：把订单+库存+支付放一个服务 → 大泥球，职责不清
- **推荐边界**：以"是否需要强一致事务"为参考，同一事务内的操作放同一 BC

---

## 聚合根粒度权衡

聚合根是一致性边界，设计聚合的核心问题：**哪些对象需要一起保持一致？**

### 订单聚合示例

```java
// 推荐：Order 作为聚合根，OrderItem 在聚合内
Order order = orderRepository.findById(orderId);  // 加载整个聚合
order.addItem(product, quantity);                  // 通过聚合根操作
order.pay(amount);                                 // 业务规则在聚合内校验
orderRepository.save(order);                       // 整体持久化

// 禁止：外部直接操作 OrderItem
orderItemRepository.save(item);  // 绕过聚合根，破坏一致性边界
```

### 粒度取舍

| 问题 | 聚合太小 | 聚合太大 |
|------|---------|---------|
| 跨聚合操作 | 需要分布式事务或最终一致 | 天然强一致（在同一事务内）|
| 并发冲突 | 锁粒度小，冲突少 | 热点聚合根并发更新频繁 |
| 对象加载 | 按需加载，内存占用少 | 一次加载过多关联对象 |
| 业务规则 | 跨聚合规则难表达 | 业务规则全在聚合内，清晰 |

**权衡经验**：
- **订单+订单项**：一个聚合（下单时同时创建，强一致）
- **订单+物流信息**：分开（发货时间晚于下单，最终一致即可）
- **账户+流水记录**：分开（流水是不可变记录，账户是聚合根）

---

## 四层架构落地

### 各层职责划分

```
用户接口层
├── Controller（接收 HTTP 请求）
├── DTO（数据传输对象，与前端交互）
└── Assembler（DTO ↔ Command/Query 转换）
         ↓
应用层（Application Service）
├── 用例编排（协调多个领域对象完成业务流程）
├── 事务边界（@Transactional 在应用层）
├── 权限校验（非领域逻辑）
└── 领域事件发布（收集后统一发布）
         ↓
领域层（Domain）
├── 实体（Entity）：Order, OrderItem
├── 值对象（Value Object）：Money, Address
├── 聚合根（Aggregate Root）：Order.pay(), Order.cancel()
├── 领域服务（Domain Service）：跨聚合逻辑，如 TransferService
├── 领域事件（Domain Event）：OrderPaidEvent
└── 仓储接口（Repository Interface）：OrderRepository（接口定义在领域层！）
         ↓
基础设施层（Infrastructure）
├── 仓储实现（Repository Impl）：MyBatis/JPA 实现 OrderRepository 接口
├── MQ Publisher（领域事件 → MQ 消息）
├── 防腐层（ACL）：第三方支付接口适配
└── 外部调用（HTTP Client、Redis、Elasticsearch）
```

**关键原则**：仓储接口定义在**领域层**，实现在**基础设施层**（依赖倒置）。领域层不依赖 Spring、MyBatis 等框架。

### 应用层 vs 领域层区分

| 逻辑类型 | 归属 | 示例 |
|----------|------|------|
| 业务规则（"库存不足不能下单"）| 领域层 | `Order.pay()` 内校验 |
| 用例流程（"下单 = 创建订单 + 扣减库存 + 发 MQ"）| 应用层 | `OrderApplicationService.placeOrder()` |
| 跨聚合的业务计算 | 领域服务 | `PricingDomainService.calculateTotal()` |
| 持久化/外部调用 | 基础设施层 | `OrderRepositoryImpl.save()` |

---

## 领域事件实战

### 同进程内传播（Spring Event）

```java
// 领域层：定义事件
public class OrderPaidEvent {
    private final String orderId;
    private final BigDecimal amount;
    // 构造器、getter...
}

// 聚合根内发布事件（不直接发布，记录到集合）
public class Order {
    private List<DomainEvent> domainEvents = new ArrayList<>();
    
    public void pay(BigDecimal amount) {
        // 业务规则校验...
        this.status = OrderStatus.PAID;
        domainEvents.add(new OrderPaidEvent(this.id, amount));
    }
}

// 应用层：统一收集事件并发布
@Transactional
public void pay(String orderId, BigDecimal amount) {
    Order order = orderRepository.findById(orderId);
    order.pay(amount);
    orderRepository.save(order);
    // 事务提交后发布领域事件
    order.getDomainEvents().forEach(applicationEventPublisher::publishEvent);
}

// 其他模块监听（同进程，Spring @EventListener）
@EventListener
public void onOrderPaid(OrderPaidEvent event) {
    inventoryService.confirmDeduct(event.getOrderId());
    pointService.addPoints(event.getUserId(), event.getAmount());
}
```

### 跨服务传播（MQ 消息）

领域事件 ≠ MQ 消息，但领域事件可以转化为 MQ 消息跨服务传播：

```
订单服务                              库存服务
OrderPaidEvent（领域事件）
  → MQ Publisher（基础设施层）
    → RocketMQ topic: order.paid
                                    → 消费者监听
                                    → InventoryService.confirmDeduct()
```

**注意**：使用本地消息表或事务消息保证"事件发出"与"本地事务提交"的原子性。

---

## CQRS 落地决策

### 适合使用 CQRS 的场景

1. **读写比例极不对称**：订单列表查询 >> 下单操作（10:1 以上）
2. **读需要跨聚合**：订单列表需要展示商品名称、用户昵称（聚合内没有）
3. **独立扩容**：读库可以横向扩展，写库保持强一致

### 典型实现

```
写端（Command Side）：
  HTTP POST /orders → OrderApplicationService.placeOrder()
                    → Order 聚合根（业务规则校验）
                    → OrderRepository → 写库（MySQL 主库）
                    → 发布 OrderCreatedEvent

读端（Query Side）：
  OrderCreatedEvent → 读模型更新器（Infrastructure）
                    → 写入 order_view 宽表（反范式，包含商品名/用户昵称）
  HTTP GET /orders  → 直接查 order_view（可用 ES/Redis）
```

**不引入 CQRS 的场景**：简单 CRUD、写操作远多于读、团队规模小、维护成本不值得。

---

## 常见落地坑

### 坑1：贫血模型惯性

```java
// 错误：贫血 Service 反模式
@Service
public class OrderService {
    public void pay(Order order, BigDecimal amount) {
        if (order.getAmount().compareTo(amount) != 0) throw ...;
        order.setStatus(OrderStatus.PAID);  // setter 直接改状态
    }
}

// 正确：充血模型
public class Order {
    public void pay(BigDecimal amount) {
        if (!this.amount.equals(amount)) throw new DomainException("金额不匹配");
        this.status = OrderStatus.PAID;  // 自己的行为自己负责
    }
}
```

### 坑2：领域层引入框架依赖

```java
// 错误：领域层直接依赖 Spring Data
public class Order {
    @Column(name = "status")  // JPA 注解污染领域模型
    private OrderStatus status;
}

// 正确：领域层纯 POJO，ORM 映射放 Infrastructure
// 用 @Entity 放在 Infrastructure 的 PO（Persistent Object）上
// PO → 领域对象 的转换在 Repository Impl 中完成
```

### 坑3：聚合根过重

```java
// 错误：一个聚合根包含 20 个字段、10 个关联对象
// → 每次操作都加载整个聚合，性能差

// 正确：遵守最小化原则
// 只有需要强一致变更的对象才放入同一聚合
// 其他对象用 ID 引用（订单只存 userId，不存 User 对象）
```

### 坑4：分布式事务替代方案不足

当跨聚合操作不可避免时：
- 优先：**领域事件 + 最终一致**（发事件、各自处理）
- 次选：**TCC**（但增加代码复杂度）
- 避免：**分布式强事务**（XA/2PC 性能差）

---

## 面试话术

**Q：DDD 和传统分层有什么区别？**

> "传统分层（MVC）是技术视角的分层，Service 层往往变成业务逻辑的垃圾桶；DDD 是业务视角的建模，核心差别是领域对象（聚合根）自己封装行为——Order.pay() 而不是 OrderService.payOrder()。这样业务意图更清晰，单元测试粒度更细，也更符合开闭原则。"

**Q：怎么用限界上下文指导微服务拆分？**

> "一个限界上下文里，同一个概念有统一含义。比如'商品'在商品中心是商品详情，在库存中心是库存 SKU，这两个上下文不应该共用一个实体类——让它们独立，就是两个微服务的边界。拆分的判断标准是：需要强一致事务的操作放同一服务，最终一致就能接受的操作跨服务用事件驱动。"

**Q：聚合根粒度怎么定？**

> "聚合根的本质是一致性边界。我会问两个问题：1. 哪些对象需要在同一个事务内一起变更？2. 热点写操作在哪个对象上？如果发现订单和订单项需要同时创建，那是同一聚合；但订单和物流信息不需要同时更新，就分开。避免聚合太大（热点并发差）也避免太小（频繁分布式事务）。"

---

## 来源

- `../../raw/note/tuling/DDD架构.md`
- `../../raw/note/架构体系/五、系统设计与架构.md`

## 关联

- 基于 [[概念-DDD]]（充血模型/聚合根/限界上下文/CQRS 概念层）
- 依赖 [[机制-微服务与SpringCloud]]（限界上下文 → 微服务边界）
- 依赖 [[synthesis/设计-分布式事务]]（跨聚合一致性处理）
- 依赖 [[机制-Spring事务]]（应用层事务边界）
