---
type: concept
status: active
name: "DDD领域驱动设计"
layer: L8
aliases: ["DDD", "领域驱动设计", "充血模型", "贫血模型", "聚合根", "值对象", "限界上下文", "领域事件", "六边形架构", "洋葱架构", "统一语言", "CQRS", "Event Sourcing", "事件溯源", "仓储", "Repository", "DDD落地实战"]
tags: ["#practice", "#framework"]
related:
  - "[[机制-微服务与SpringCloud]]"
  - "[[机制-设计模式]]"
  - "[[概念-高可用设计]]"
  - "[[设计-分布式事务]]"
  - "[[机制-Spring]]"
sources:
  - "../../../raw/note/tuling/DDD架构.md"
  - "../../../raw/note/架构体系/五、系统设计与架构.md"
created: 2026-05-07
updated: 2026-05-13
---

# DDD 领域驱动设计

> 以**业务领域**为核心的软件设计方法论：先对业务进行领域建模，再考虑持久化存储；让领域对象自包含业务行为，避免业务逻辑散落在 Service 层。本文同时覆盖 DDD 的概念层和落地实战层，作为该主题的单一归宿页。

## 第一性原理

传统"表驱动开发"路径：来了需求 → 先设计数据库表 → 生成 POJO → 在 Service 层写所有业务逻辑 → POJO 退化为纯数据容器（贫血模型）。这条路导致 Service 臃肿、逻辑分散、难以维护。

DDD 的颠覆：**代码的核心是业务知识，而不是数据库表**。软件开发人员与业务人员共同建立统一语言（Ubiquitous Language），将业务概念映射到代码中的领域对象，由领域对象自己完成行为（充血模型）。

---

## 战略建模 (Strategic Modeling)

战略建模解决"微服务怎么拆"的问题。

### 统一语言 (Ubiquitous Language)

业务人员和开发人员共用术语：不谈数据库字段，谈业务概念（如"下单"而非"Insert Order"）。代码中的类名/方法名应与业务术语一致（`Order.cancel()`）。

### 限界上下文 (Bounded Context)

每个领域概念在特定上下文中含义明确且统一。
- **拆分逻辑**：商品在"商品中心"是详情，在"库存中心"是 SKU。上下文边界即微服务边界。
- **上下文映射 (Context Map)**：
  - **共享内核 (Shared Kernel)**：共享模型。
  - **防腐层 (ACL)**：**最重要**。隔离外部不良模型，在基础设施层做翻译适配。
  - **客户-供应商**：上游提供服务，下游消费。

### 微服务拆分决策

- **粒度原则**：以"是否需要强一致事务"为参考，同一事务内的操作放同一上下文。
- **康威定律逆用**：一个上下文对应一个能完整负责的团队。

### 限界上下文到微服务的落地流程

```
1. 业务分析：识别核心域（支付）/ 支撑域（风控）/ 通用域（用户中心）
2. 识别限界上下文（BC）：同一概念在不同上下文含义不同
   - "商品"在商品中心 = 商品详情
   - "商品"在库存中心 = 库存 SKU
   - "商品"在订单中心 = 订单行 OrderItem
3. 划定 BC 边界 = 微服务边界
4. 定义上下文映射（Context Map）：BC 之间如何交互
```

### 上下文映射关系补充

| 关系 | 含义 | 实现方式 |
|------|------|----------|
| 共享内核（Shared Kernel）| 两个 BC 共享部分领域模型 | 独立 common 模块，变更需协商 |
| 客户-供应商（Customer-Supplier）| 下游 BC 依赖上游 BC | API 契约，上游保障 SLA |
| 防腐层（Anti-Corruption Layer, ACL）| 隔离外部不良模型 | 在基础设施层做翻译/适配 |
| 开放主机服务（OHS）| 提供通用协议供他人集成 | RESTful API + Swagger |

**实战判断**：
- **太细**：大量微服务导致跨服务事务和网络开销激增
- **太粗**：订单、库存、支付全塞一处，逐步演化成大泥球
- **推荐**：以强一致边界为参考，需要同事务处理的能力尽量放在同一上下文

---

## 战术建模 (Tactical Modeling)

战术建模解决"类怎么写"的问题。

### 核心概念

- **实体 (Entity)**：有唯一 ID，状态可变（如 Order）。
- **值对象 (Value Object)**：无 ID，不可变。修改=创建新对象（如 Address, Money）。
- **聚合与聚合根 (Aggregate Root)**：
  - 聚合是一致性边界。**外部只能通过聚合根操作内部对象**。
  - **聚合根粒度权衡**：
    - **太大**：并发冲突多、加载慢（如包含所有关联对象）。
    - **太小**：跨聚合分布式事务激增（如订单与订单项拆分）。
    - **推荐**：订单+订单项（一聚合）；订单+物流（分开，最终一致）。

### 聚合根粒度的实战判断

聚合根设计的核心问题是：**哪些对象需要一起保持一致？**

```java
// 推荐：Order 作为聚合根，OrderItem 在聚合内
Order order = orderRepository.findById(orderId);
order.addItem(product, quantity);
order.pay(amount);
orderRepository.save(order);

// 禁止：外部直接操作 OrderItem
orderItemRepository.save(item);
```

| 问题 | 聚合太小 | 聚合太大 |
|------|---------|---------|
| 跨聚合操作 | 需要分布式事务或最终一致 | 天然强一致 |
| 并发冲突 | 锁粒度小，冲突少 | 热点聚合根更新频繁 |
| 对象加载 | 按需加载，内存占用少 | 一次加载过多关联对象 |
| 业务规则 | 跨聚合规则难表达 | 规则内聚，语义清晰 |

**经验判断**：
- **订单 + 订单项**：通常一个聚合
- **订单 + 物流**：通常拆开，走最终一致
- **账户 + 流水**：通常拆开，账户是聚合根，流水是不可变凭证

### 充血模型 vs 贫血模型

- **充血 (DDD 推荐)**：行为在对象内（`Order.pay()`）。高内聚，易于测试。
- **贫血 (传统 MVC)**：行为在 Service。POJO 仅是数据容器，逻辑分散。

---

## 架构落地 (Implementation)

### 四层架构职责

| 层级 | 职责 | 关键组件 |
|------|------|----------|
| **用户接口层** | 接收请求，DTO 转换 | Controller, Assembler, DTO |
| **应用层** | **编排**业务流程，控制**事务边界** | Application Service, 事务注解 |
| **领域层** | 核心业务规则、状态变更 | 实体、值对象、聚合根、领域服务、Repository 接口 |
| **基础设施层** | 持久化实现、消息发布、ACL 适配 | Repository 实现, MQ Publisher, HTTP Client |

**依赖倒置 (DIP)**：仓储接口定义在**领域层**，实现在**基础设施层**。领域层不依赖具体技术（Spring/MyBatis）。

### 四层架构的落地视角

```
用户接口层
├── Controller（接收 HTTP 请求）
├── DTO（与前端交互）
└── Assembler（DTO ↔ Command/Query 转换）
         ↓
应用层
├── 用例编排
├── 事务边界
├── 权限校验
└── 领域事件收集与发布
         ↓
领域层
├── 实体 / 值对象 / 聚合根
├── 领域服务
├── 领域事件
└── Repository 接口
         ↓
基础设施层
├── Repository 实现
├── MQ Publisher
├── ACL 适配器
└── Redis / ES / HTTP Client 等外部依赖
```

### 应用层 vs 领域层如何区分

| 逻辑类型 | 归属 | 示例 |
|----------|------|------|
| 业务规则 | 领域层 | `Order.pay()` 内校验 |
| 用例流程编排 | 应用层 | `placeOrder()` 协调多个对象 |
| 跨聚合业务计算 | 领域服务 | `PricingDomainService.calculateTotal()` |
| 持久化/外部调用 | 基础设施层 | `OrderRepositoryImpl.save()` |

### 领域事件 (Domain Event)

- **同进程内**：Spring `@EventListener`。
- **跨服务**：领域事件 → MQ Publisher → MQ 消息。
- **原子性**：利用本地消息表或事务消息确保"事件发出"与"事务提交"一致。

### 领域事件实战

```java
public class OrderPaidEvent {
    private final String orderId;
    private final BigDecimal amount;
}

public class Order {
    private List<DomainEvent> domainEvents = new ArrayList<>();

    public void pay(BigDecimal amount) {
        this.status = OrderStatus.PAID;
        domainEvents.add(new OrderPaidEvent(this.id, amount));
    }
}

@Transactional
public void pay(String orderId, BigDecimal amount) {
    Order order = orderRepository.findById(orderId);
    order.pay(amount);
    orderRepository.save(order);
    order.getDomainEvents().forEach(applicationEventPublisher::publishEvent);
}

@EventListener
public void onOrderPaid(OrderPaidEvent event) {
    inventoryService.confirmDeduct(event.getOrderId());
    pointService.addPoints(event.getUserId(), event.getAmount());
}
```

跨服务时，领域事件会在基础设施层转成 MQ 消息；要结合本地消息表或事务消息，保证“本地事务成功”和“事件发出”一致。

---

## 高级模式

### CQRS (命令查询职责分离)

- **写端 (Command)**：走领域模型，保证业务规则强一致。
- **读端 (Query)**：直接查反范式的宽表（ES/Redis/DB），为查询性能优化。
- **适用**：读写比例悬殊（10:1+）；读需要跨聚合多表 JOIN。

### CQRS 的典型落地

```
写端（Command Side）：
  POST /orders -> OrderApplicationService.placeOrder()
               -> Order 聚合根
               -> MySQL 写库
               -> 发布 OrderCreatedEvent

读端（Query Side）：
  OrderCreatedEvent -> 更新 order_view 宽表
  GET /orders       -> 直接查宽表 / ES / Redis
```

**适合引入 CQRS**：
1. 读写比例极不对称
2. 查询要跨多个聚合
3. 读模型和写模型扩容诉求不同

**不适合**：简单 CRUD、小团队、小系统，为了架构而架构得不偿失。

### 事件溯源 (Event Sourcing)

不存状态，存变更记录流。通过重放事件还原状态。适用于金融合规、高审计要求的场景。

---

## 实战坑点

1. **贫血模型惯性**：习惯性把逻辑写在 Service。应思考："这个动作能由对象自己做吗？"
2. **领域层引入技术依赖**：如在领域实体上加 JPA/MyBatis 注解。应通过 PO 转换隔离。
3. **聚合根过重**：加载一个订单把用户、商品、物流全加载了。应采用 **ID 引用**。
4. **分布式事务滥用**：应优先使用 **领域事件 + 最终一致** 替代 XA 事务。

### 典型反模式示例

```java
// 贫血模型
@Service
public class OrderService {
    public void pay(Order order, BigDecimal amount) {
        if (order.getAmount().compareTo(amount) != 0) throw ...;
        order.setStatus(OrderStatus.PAID);
    }
}

// 充血模型
public class Order {
    public void pay(BigDecimal amount) {
        if (!this.amount.equals(amount)) throw new DomainException("金额不匹配");
        this.status = OrderStatus.PAID;
    }
}
```

```java
// 错误：领域层直接依赖 ORM 注解
public class Order {
    @Column(name = "status")
    private OrderStatus status;
}
```

应让领域对象保持纯粹，ORM 映射放到基础设施层的 PO / Repository 实现中。

---

## 关键权衡

1. **复杂度 vs 长期价值**：初创项目 MVP 期不建议 DDD，业务稳定且复杂后值得投入。
2. **纯粹性 vs 实用性**：领域层理论上不应有 Spring 注解，但实战中往往会有务实妥协。

---

## 面试话术

- **Q：DDD 和传统 MVC 区别？**
  > "MVC 是技术视角分层，逻辑易在 Service 堆积；DDD 是业务视角分层，核心是领域对象自包含行为（Order.pay()）。它让业务意图更清晰，通过聚合根保护一致性边界。"
- **Q：微服务怎么拆？**
  > "通过限界上下文。同一个词在不同语境意义不同（商品中心 vs 库存中心）就是天然边界。拆分标准是：强一致放一处，最终一致可跨服务。"
- **Q：聚合根粒度怎么定？**
  > "聚合根的本质是一致性边界。我会先问：哪些对象必须在同一个事务里一起变更？哪些对象是热点写？订单和订单项通常放一起，因为下单时要同时创建；订单和物流通常拆开，因为发货发生在后续流程，最终一致就够。避免聚合太大，也避免聚合太小导致频繁分布式事务。"

---

## 应用边界

- **适合**：业务逻辑复杂（金融、核心电商）、长线迭代项目。
- **不适合**：纯 CRUD 背景后台、技术驱动的底层组件（数据管道）。
