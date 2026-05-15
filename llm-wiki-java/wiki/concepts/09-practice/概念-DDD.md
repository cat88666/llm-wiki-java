---
type: concept
status: active
name: "DDD领域驱动设计"
layer: L8
aliases: ["DDD", "领域驱动设计", "充血模型", "贫血模型", "聚合根", "值对象", "限界上下文", "领域事件", "六边形架构", "洋葱架构", "统一语言", "CQRS", "Event Sourcing", "事件溯源", "仓储", "Repository", "DDD落地实战"]
tags: ["#practice", "#framework"]
related:
  - "[[机制-微服务与SpringCloud]]"
  - "[[主题-设计模式]]"
  - "[[主题-三高体系]]"
  - "[[设计-分布式事务]]"
  - "[[机制-Spring]]"
sources:
  - "../../../raw/note/tuling/DDD架构.md"
  - "../../../raw/note/架构体系/五、系统设计与架构.md"
created: 2026-05-07
updated: 2026-05-15
---

# DDD 领域驱动设计

> 以**业务领域**为核心的软件设计方法论：先对业务进行领域建模，再考虑持久化存储；让领域对象自包含业务行为，避免业务逻辑散落在 Service 层。本文同时覆盖 DDD 的概念层和落地实战层。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 表驱动开发的困境、DDD 颠覆：代码核心是业务知识 |
| [二、核心机制——战略建模](#二核心机制战略建模) | 统一语言、限界上下文、上下文映射、微服务拆分 |
| [三、核心机制——战术建模](#三核心机制战术建模) | 实体、值对象、聚合根、充血 vs 贫血 |
| [四、Java 核心使用](#四java-核心使用) | 四层架构、领域事件、代码示例 |
| [五、使用案例](#五使用案例) | CQRS、事件溯源 |
| [六、生产风险](#六生产风险) | 贫血惯性、技术依赖泄漏、聚合根过重、分布式事务滥用 |
| [七、关键权衡](#七关键权衡) | 复杂度 vs 长期价值、纯粹性 vs 实用性 |
| [八、与其他概念的关系](#八与其他概念的关系) | 微服务、设计模式、分布式事务、Spring |
| [九、应用边界](#九应用边界) | 适用与不适用场景 |

## 一、第一性原理

传统"表驱动开发"路径：来了需求 → 先设计数据库表 → 生成 POJO → 在 Service 层写所有业务逻辑 → POJO 退化为纯数据容器（贫血模型）。这条路导致 Service 臃肿、逻辑分散、难以维护。

DDD 的颠覆：**代码的核心是业务知识，而不是数据库表**。软件开发人员与业务人员共同建立统一语言（Ubiquitous Language），将业务概念映射到代码中的领域对象，由领域对象自己完成行为（充血模型）。

## 二、核心机制——战略建模

战略建模解决"微服务怎么拆"的问题。

### 2.1 统一语言 (Ubiquitous Language)

业务人员和开发人员共用术语：不谈数据库字段，谈业务概念（如"下单"而非"Insert Order"）。代码中的类名/方法名应与业务术语一致（`Order.cancel()`）。

### 2.2 限界上下文 (Bounded Context)

每个领域概念在特定上下文中含义明确且统一。

**拆分逻辑**：商品在"商品中心"是详情，在"库存中心"是 SKU，在"订单中心"是 OrderItem。上下文边界即微服务边界。

**落地流程**：

```
1. 业务分析：识别核心域（支付）/ 支撑域（风控）/ 通用域（用户中心）
2. 识别限界上下文（BC）：同一概念在不同上下文含义不同
3. 划定 BC 边界 = 微服务边界
4. 定义上下文映射（Context Map）：BC 之间如何交互
```

### 2.3 上下文映射 (Context Map)

| 关系 | 含义 | 实现方式 |
|------|------|----------|
| 共享内核（Shared Kernel）| 两个 BC 共享部分领域模型 | 独立 common 模块，变更需协商 |
| 客户-供应商（Customer-Supplier）| 下游 BC 依赖上游 BC | API 契约，上游保障 SLA |
| **防腐层（ACL）** | 隔离外部不良模型 | 在基础设施层做翻译/适配 |
| 开放主机服务（OHS）| 提供通用协议供他人集成 | RESTful API + Swagger |

### 2.4 微服务拆分决策

| 判断 | 说明 |
|------|------|
| **粒度原则** | 以"是否需要强一致事务"为参考，同一事务内的操作放同一上下文 |
| **康威定律逆用** | 一个上下文对应一个能完整负责的团队 |
| **太细** | 大量微服务导致跨服务事务和网络开销激增 |
| **太粗** | 订单、库存、支付全塞一处，逐步演化成大泥球 |
| **推荐** | 需要同事务处理的能力尽量放在同一上下文 |

## 三、核心机制——战术建模

战术建模解决"类怎么写"的问题。

### 3.1 核心概念

| 概念 | 说明 |
|------|------|
| **实体 (Entity)** | 有唯一 ID，状态可变（如 Order） |
| **值对象 (Value Object)** | 无 ID，不可变。修改=创建新对象（如 Address, Money） |
| **聚合根 (Aggregate Root)** | 一致性边界的入口。外部只能通过聚合根操作内部对象 |
| **领域服务 (Domain Service)** | 跨聚合的业务计算，不属于任何单一实体 |
| **仓储 (Repository)** | 聚合根的持久化接口，定义在领域层，实现在基础设施层 |

### 3.2 聚合根粒度的实战判断

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

### 3.3 充血模型 vs 贫血模型

| 维度 | 充血模型（DDD 推荐） | 贫血模型（传统 MVC） |
|------|---------------------|---------------------|
| 行为位置 | 在领域对象内（`Order.pay()`） | 在 Service 层 |
| 对象角色 | 自包含行为和状态 | 仅是数据容器 |
| 内聚性 | 高，业务规则与数据紧耦合 | 低，逻辑分散在多个 Service |
| 可测试性 | 高，纯 POJO 单测即可 | 低，需 Mock 大量依赖 |

```java
// 贫血模型（反模式）
@Service
public class OrderService {
    public void pay(Order order, BigDecimal amount) {
        if (order.getAmount().compareTo(amount) != 0) throw ...;
        order.setStatus(OrderStatus.PAID);
    }
}

// 充血模型（DDD 推荐）
public class Order {
    public void pay(BigDecimal amount) {
        if (!this.amount.equals(amount)) throw new DomainException("金额不匹配");
        this.status = OrderStatus.PAID;
    }
}
```

## 四、Java 核心使用

### 4.1 四层架构

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

**依赖倒置 (DIP)**：仓储接口定义在**领域层**，实现在**基础设施层**。领域层不依赖具体技术（Spring/MyBatis）。

### 4.2 应用层 vs 领域层

| 逻辑类型 | 归属 | 示例 |
|----------|------|------|
| 业务规则 | 领域层 | `Order.pay()` 内校验 |
| 用例流程编排 | 应用层 | `placeOrder()` 协调多个对象 |
| 跨聚合业务计算 | 领域服务 | `PricingDomainService.calculateTotal()` |
| 持久化/外部调用 | 基础设施层 | `OrderRepositoryImpl.save()` |

### 4.3 领域事件 (Domain Event)

| 场景 | 实现方式 |
|------|---------|
| 同进程内 | Spring `@EventListener` |
| 跨服务 | 领域事件 → MQ Publisher → MQ 消息 |
| 原子性保证 | 本地消息表或事务消息确保"事件发出"与"事务提交"一致 |

```java
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

跨服务时，领域事件会在基础设施层转成 MQ 消息；要结合本地消息表或事务消息，保证"本地事务成功"和"事件发出"一致。

## 五、使用案例

### 5.1 CQRS (命令查询职责分离)

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

| 判断 | 说明 |
|------|------|
| **适合引入** | 读写比例极不对称；查询要跨多个聚合；读写模型扩容诉求不同 |
| **不适合** | 简单 CRUD、小团队、小系统，为了架构而架构得不偿失 |

### 5.2 事件溯源 (Event Sourcing)

不存状态，存变更记录流。通过重放事件还原状态。适用于金融合规、高审计要求的场景。

## 六、生产风险

| 风险 | 表现 | 应对 |
|------|------|------|
| **贫血模型惯性** | 习惯性把逻辑写在 Service | 思考"这个动作能由对象自己做吗？" |
| **领域层引入技术依赖** | 在领域实体上加 JPA/MyBatis 注解 | 通过 PO 转换隔离，领域对象保持纯粹 |
| **聚合根过重** | 加载一个订单把用户、商品、物流全加载了 | 采用 **ID 引用**，按需加载 |
| **分布式事务滥用** | 跨聚合操作用 XA 事务 | 优先使用 **领域事件 + 最终一致** |

```java
// 错误：领域层直接依赖 ORM 注解
public class Order {
    @Column(name = "status")
    private OrderStatus status;
}
// 应让领域对象保持纯粹，ORM 映射放到基础设施层的 PO / Repository 实现中
```

## 七、关键权衡

| 权衡点 | 说明 |
|--------|------|
| 复杂度 vs 长期价值 | 初创项目 MVP 期不建议 DDD，业务稳定且复杂后值得投入 |
| 纯粹性 vs 实用性 | 领域层理论上不应有 Spring 注解，但实战中往往会有务实妥协 |
| 聚合根粒度 | 太大→并发冲突多、加载慢；太小→分布式事务激增 |
| CQRS 引入时机 | 读写比例悬殊且查询跨聚合时有价值，否则增加维护成本 |

## 八、与其他概念的关系

- **依赖 [[机制-微服务与SpringCloud]]**：限界上下文边界 = 微服务边界；微服务拆分的方法论来自 DDD 战略建模
- **依赖 [[主题-设计模式]]**：仓储模式（Repository）、工厂模式（Factory）、策略模式（Strategy）在 DDD 中频繁使用
- **依赖 [[设计-分布式事务]]**：跨聚合的一致性问题通过领域事件 + 最终一致性解决，与 Saga/TCC/本地消息表等方案关联
- **依赖 [[机制-Spring]]**：应用层事务用 `@Transactional`；领域事件用 Spring `@EventListener`；依赖倒置通过 Spring IoC 实现

## 九、应用边界

**适合 DDD**：
- 业务逻辑复杂（金融、核心电商）、长线迭代项目
- 团队对业务领域有深入理解，能建立统一语言
- 系统规模大到需要明确的模块边界和一致性保障

**不适合 DDD**：
- 纯 CRUD 后台管理系统
- 技术驱动的底层组件（数据管道、消息中间件）
- 初创 MVP 阶段（业务还在频繁试错，建模过早会成为负担）
