---
type: concept
status: active
name: "DDD领域驱动设计"
layer: L8
aliases: ["DDD", "领域驱动设计", "领域建模", "充血模型", "贫血模型", "聚合根", "实体", "值对象", "限界上下文", "领域事件", "六边形架构", "洋葱架构", "整洁架构", "统一语言", "CQRS", "Event Sourcing", "事件溯源", "仓储", "Repository", "DDD落地实战"]
tags: ["#practice", "#framework"]
related:
  - "[[机制-SpringCloud]]"
  - "[[主题-设计模式]]"
  - "[[主题-三高架构]]"
  - "[[概念-分布式事务]]"
  - "[[机制-Spring]]"
---

# DDD 领域驱动设计

> 以业务领域为核心的软件设计方法论：先对业务建模，再考虑持久化；让领域对象自包含业务行为，避免逻辑散落在 Service 层。战略建模解决"系统怎么拆"，战术建模解决"类怎么写"。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、DDD 本质](#一ddd-本质) | 表驱动的困境、DDD 的颠覆、适用判断 |
| [二、战略建模：系统怎么拆](#二战略建模系统怎么拆) | 统一语言、限界上下文、上下文映射、微服务拆分 |
| [三、战术建模：类怎么写](#三战术建模类怎么写) | 实体、值对象、聚合根、领域服务、仓储、充血 vs 贫血 |
| [四、分层架构与依赖倒置](#四分层架构与依赖倒置) | 四层架构、六边形/洋葱架构、应用层 vs 领域层 |
| [五、领域事件](#五领域事件) | 事件定义、vs MQ 消息、实现方式、原子性保证 |
| [六、CQRS 与事件溯源](#六cqrs-与事件溯源) | 读写分离架构、事件流还原状态、引入时机 |
| [七、DDD 落地路径](#七ddd-落地路径) | 从业务问题出发的六步落地、持续演进 |
| [八、生产风险与关键权衡](#八生产风险与关键权衡) | 贫血惯性、技术泄漏、聚合过重、复杂度 vs 长期价值 |
| [九、面试速答](#九面试速答) | 高频问题 + 即用口径 |

## 一、DDD 本质

### 1.1 表驱动的困境

传统开发路径：需求 → 设计表 → 生成 POJO → Service 写所有逻辑 → POJO 退化为纯数据容器（贫血模型）。

结果：Service 臃肿、逻辑分散、改一处牵一片、业务规则散落在多个 Service 且互相打架。

### 1.2 DDD 的颠覆

**代码的核心是业务知识，不是数据库表**。开发人员与业务人员共建统一语言（Ubiquitous Language），将业务概念映射到领域对象，由领域对象自己完成行为（充血模型）。

| 方式 | 起点 | 典型结果 |
| --- | --- | --- |
| 表驱动 | 先设计表，再写 Service | 数据结构先行，业务逻辑堆到 Service |
| DDD | 先理解领域和业务语言，再建模，最后考虑持久化 | 模型围绕业务能力组织，代码贴近业务语义 |

### 1.3 DDD 的收益与成本

| 维度 | 收益 | 成本 |
| --- | --- | --- |
| 业务理解 | 统一语言让研发和业务沟通更准确 | 需要业务人员深度参与 |
| 模块化 | 按领域概念组织模块，边界更清晰 | 前期建模和评审成本高 |
| 可维护性 | 业务规则内聚在领域对象，便于重构 | 团队要理解聚合、值对象、领域事件等概念 |
| 可扩展性 | 业务变化更容易落到模型演进 | 简单项目会显得"架构过重" |

### 1.4 什么时候用 / 不用 DDD

```
业务逻辑复杂（金融、电商核心链路）+ 长线迭代     → DDD
系统规模大、需要明确模块边界和一致性保障           → DDD
团队对业务领域有深入理解，能建立统一语言           → DDD

纯 CRUD 后台管理系统                             → 表驱动更高效
技术驱动的底层组件（数据管道、消息中间件）          → 不需要 DDD
初创 MVP 阶段（业务还在频繁试错）                 → 建模过早成为负担
```

> DDD 最大价值不是"多几个包名"，而是通过建模把业务知识沉淀成代码结构。表驱动不是错误，CRUD 系统直接表驱动效率更高。

## 二、战略建模：系统怎么拆

### 2.1 统一语言（Ubiquitous Language）

业务人员和开发人员共用术语：不谈数据库字段，谈业务概念。代码中的类名/方法名应与业务术语一致。

```
业务说"取消订单" → 代码就是 Order.cancel()
业务说"商品价格" → 代码统一用 productPrice，不要一会儿 salePrice 一会儿 productAmount
```

统一语言不是文档工程，需要在需求、代码、测试和接口中持续保持一致。命名不一致会在长期协作中变成沟通成本。

### 2.2 限界上下文（Bounded Context）

每个领域概念在特定上下文中含义明确且统一。**同一个词在不同上下文含义不同**：

```
"商品"
  ├── 商品中心：详情（名称、描述、图片、价格）
  ├── 库存中心：SKU（规格、库存数量、仓库）
  └── 订单中心：OrderItem（下单时的快照：名称、单价、数量）
```

**上下文边界 = 微服务边界**。识别流程：

```
1. 业务分析：识别核心域（支付）/ 支撑域（风控）/ 通用域（用户中心）
2. 识别限界上下文：同一概念在哪些场景下含义不同
3. 划定 BC 边界 = 微服务边界
4. 定义上下文映射：BC 之间如何交互
```

### 2.3 上下文映射（Context Map）

| 关系 | 含义 | 实现方式 |
|------|------|---------|
| 共享内核（Shared Kernel）| 两个 BC 共享部分领域模型 | 独立 common 模块，变更需协商 |
| 客户-供应商（Customer-Supplier）| 下游依赖上游 | API 契约，上游保障 SLA |
| **防腐层（ACL）** | 隔离外部不良模型，防止外部变化污染核心领域 | 基础设施层做翻译/适配 |
| 开放主机服务（OHS）| 提供通用协议供外部集成 | RESTful API + 契约文档 |

**防腐层是最常用的模式**：对接第三方支付、外部 ERP、遗留系统时，在基础设施层用 Adapter 将外部模型翻译为内部领域模型，核心领域不直接依赖外部数据结构。

### 2.4 微服务拆分决策

| 维度 | 判断依据 |
|------|---------|
| 粒度原则 | 需要强一致事务的操作放同一上下文 |
| 康威定律 | 一个上下文对应一个能完整负责的团队 |
| 太细的信号 | 大量跨服务事务、网络开销激增、一个需求改 5 个服务 |
| 太粗的信号 | 订单/库存/支付全塞一处，逐步演化成大泥球 |
| 演进策略 | 先粗后细，需要同事务处理的能力优先放在同一上下文 |

## 三、战术建模：类怎么写

### 3.1 核心构件

| 构件 | 定义 | 关键特征 | 示例 |
|------|------|---------|------|
| **实体（Entity）** | 有唯一 ID，状态可变 | 关注"是不是同一个对象"，有创建/修改/删除生命周期 | Order、User、Product |
| **值对象（Value Object）** | 无唯一 ID，不可变 | 关注"值是否相等"，修改 = 创建新对象 | Money（amount+currency）、Address |
| **聚合根（Aggregate Root）** | 一致性边界的唯一入口 | 外部只能通过聚合根操作内部对象 | Order 是聚合根，OrderItem 在聚合内 |
| **领域服务（Domain Service）** | 跨聚合的业务计算 | 不属于任何单一实体，无状态 | PricingDomainService.calculateTotal() |
| **仓储（Repository）** | 聚合根的持久化接口 | 接口定义在领域层，实现在基础设施层 | OrderRepository.findById() / .save() |

**实体 vs 值对象的实战判断**：

订单是实体（订单号唯一，状态从待支付变为已支付）；地址通常是值对象（由省/市/街道描述，修改地址更适合创建新 Address）。

OrderItem 在电商中常建成**值对象快照**，保存下单时的商品名称、数量、单价，避免商品后续改名影响历史订单语义。

### 3.2 聚合根粒度

聚合根设计的核心问题：**哪些对象需要一起保持一致？**

```java
// 正确：通过聚合根操作
Order order = orderRepository.findById(orderId);
order.addItem(product, quantity);      // ← 聚合根保证内部一致性
order.pay(amount);
orderRepository.save(order);

// 错误：绕过聚合根直接操作内部对象
orderItemRepository.save(item);        // ← 破坏聚合边界
```

| 粒度 | 优点 | 缺点 |
|------|------|------|
| 聚合太大 | 天然强一致，规则内聚 | 热点聚合根并发冲突多，一次加载过多关联对象 |
| 聚合太小 | 锁粒度小，按需加载 | 跨聚合操作需分布式事务或最终一致 |

**经验判断**：
- 订单 + 订单项 → 通常一个聚合（订单项不能脱离订单独立存在）
- 订单 + 物流 → 通常拆开，走最终一致（物流有独立生命周期）
- 账户 + 流水 → 通常拆开，账户是聚合根，流水是不可变凭证

### 3.3 充血模型 vs 贫血模型

| 维度 | 充血模型（DDD 推荐） | 贫血模型（传统 MVC） |
|------|---------------------|---------------------|
| 行为位置 | 领域对象内（`Order.pay()`） | Service 层 |
| 对象角色 | 自包含行为和状态 | 纯数据容器（getter/setter）|
| 内聚性 | 高，业务规则与数据紧耦合 | 低，逻辑分散在多个 Service |
| 可测试性 | 高，纯 POJO 单测 | 低，需 Mock 大量依赖 |
| 上手成本 | 高，需理解业务模型 | 低，接近传统 CRUD |

```java
// 贫血模型（反模式）— 所有逻辑在 Service，Order 只是数据容器
@Service
public class OrderService {
    public void pay(Order order, BigDecimal amount) {
        if (order.getAmount().compareTo(amount) != 0) throw ...;
        order.setStatus(OrderStatus.PAID);        // ← 外部直接 set 状态
    }
}

// 充血模型（DDD）— Order 自己封装业务规则
public class Order {
    public void pay(BigDecimal amount) {
        if (!this.amount.equals(amount))
            throw new DomainException("金额不匹配");
        this.status = OrderStatus.PAID;            // ← 状态变更在对象内部
        this.domainEvents.add(new OrderPaidEvent(this.id, amount));
    }
}
```

**判断标准**：如果业务规则已经散落到多个 Service 且经常互相打架，就应该考虑把规则收回到领域模型。小型后台或简单流程用贫血模型更高效。

## 四、分层架构与依赖倒置

### 4.1 DDD 四层架构

```
用户接口层（interfaces）
├── Controller（接收 HTTP/RPC 请求）
├── DTO（与前端交互的数据结构）
└── Assembler（DTO ↔ Command/Query 转换）
         ↓
应用层（application）
├── 用例编排（ApplicationService）          ← 协调多个领域对象完成一个业务用例
├── 事务边界（@Transactional）
├── 权限校验
└── 领域事件收集与发布
         ↓
领域层（domain）                            ← 核心：不依赖任何技术框架
├── 实体 / 值对象 / 聚合根
├── 领域服务
├── 领域事件
└── Repository 接口                         ← 只定义接口，不写实现
         ↓
基础设施层（infrastructure）
├── Repository 实现（MyBatis/JPA）
├── MQ Publisher
├── ACL 适配器（翻译外部模型）
└── Redis / ES / HTTP Client
```

**依赖倒置（DIP）是核心**：Repository 接口定义在领域层，实现在基础设施层。领域层不依赖 Spring/MyBatis/JPA，保持纯粹。

### 4.2 应用层 vs 领域层

| 逻辑类型 | 归属 | 判断标准 | 示例 |
|----------|------|---------|------|
| 业务规则 | 领域层 | "这个规则是业务固有的吗？" | `Order.pay()` 内校验金额 |
| 用例编排 | 应用层 | "这是一个完整的业务用例吗？" | `placeOrder()` 协调订单+库存+支付 |
| 跨聚合计算 | 领域服务 | "这个计算不属于任何单一聚合" | `PricingDomainService.calculateTotal()` |
| 持久化/外部调用 | 基础设施层 | "这是技术实现细节" | `OrderRepositoryImpl.save()` |

### 4.3 六边形架构与洋葱架构

三种架构表现形式不同，核心相同：**领域模型在中心，技术细节在外圈**。

| 架构 | 核心思想 | 与四层架构的关系 |
| --- | --- | --- |
| DDD 四层 | 层次分工，上层调用下层 | 基础形式 |
| 洋葱架构 | 从外到内包裹领域模型 | 强调依赖方向：外层依赖内层 |
| 六边形架构 | 端口（接口）+ 适配器（实现）隔离输入输出 | 强调可替换性：换数据库只改适配器 |

```
Java 项目包结构：
domain/           实体、值对象、聚合根、领域服务、Repository 接口
application/      用例编排、事务、命令/查询处理
interfaces/       Controller、DTO、Assembler
infrastructure/   Repository 实现、MQ、Redis、外部服务适配
```

## 五、领域事件

### 5.1 领域事件 vs MQ 消息

| 维度 | 领域事件 | MQ 消息 |
|------|---------|---------|
| 本质 | 领域内状态变化的建模概念 | 跨进程传递事件的技术手段 |
| 作用域 | 可以只在单个微服务内部流转 | 跨服务 |
| 定义位置 | 领域层 | 基础设施层 |
| 关系 | 先有领域事件，再决定是否通过 MQ 传递 | MQ 是领域事件跨服务的载体之一 |

> 不要把所有领域事件都直接设计成 MQ 消息。先在领域层定义事件，跨服务时才由基础设施层转为 MQ 消息。

### 5.2 事件驱动解耦

| 事件 | 可能的订阅者 |
| --- | --- |
| `OrderCreatedEvent` | 库存预占、支付单创建、营销记录 |
| `OrderPaidEvent` | 积分发放、发货单生成、通知用户 |
| `OrderCanceledEvent` | 库存释放、优惠券退回、通知用户 |

核心价值：订单聚合不需要知道库存、积分、通知的存在，发事件即可，后续动作通过订阅触发。

### 5.3 实现方式

| 场景 | 实现 | 一致性保证 |
|------|------|-----------|
| 同进程内 | Spring `@EventListener` | 与事务绑定（`@TransactionalEventListener`）|
| 跨服务 | 领域事件 → MQ Publisher → MQ 消息 | 本地消息表或事务消息 |

```java
// 领域对象收集事件
public class Order {
    private List<DomainEvent> domainEvents = new ArrayList<>();

    public void pay(BigDecimal amount) {
        this.status = OrderStatus.PAID;
        domainEvents.add(new OrderPaidEvent(this.id, amount));  // ← 收集，不立即发布
    }
}

// 应用层：事务提交后发布事件
@Transactional
public void pay(String orderId, BigDecimal amount) {
    Order order = orderRepository.findById(orderId);
    order.pay(amount);
    orderRepository.save(order);
    order.getDomainEvents().forEach(applicationEventPublisher::publishEvent);
}

// 事件订阅者
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)  // ← 事务提交后才触发
public void onOrderPaid(OrderPaidEvent event) {
    inventoryService.confirmDeduct(event.getOrderId());
    pointService.addPoints(event.getUserId(), event.getAmount());
}
```

**跨服务原子性**：本地事务提交和事件发出必须一致。两种主流方案：
- **本地消息表**：事件先写本地 DB（与业务同事务），后台任务轮询发送到 MQ
- **事务消息**（RocketMQ）：半消息 → 本地事务 → 提交/回滚消息

详细机制 → [[概念-分布式事务]]

## 六、CQRS 与事件溯源

### 6.1 CQRS（命令查询职责分离）

写端和读端使用不同模型，独立优化。

```
写端（Command Side）：
  POST /orders → OrderApplicationService.placeOrder()
             → Order 聚合根（充血模型，保证业务规则）
             → MySQL 写库（范式化）
             → 发布 OrderCreatedEvent

读端（Query Side）：
  OrderCreatedEvent → 异步更新 order_view 宽表 / ES / Redis
  GET /orders       → 直接查宽表（反范式化，查询快）
```

| 判断 | 说明 |
|------|------|
| **适合** | 读写比例极不对称；查询要跨多个聚合做宽表；读写模型扩容需求不同 |
| **不适合** | 简单 CRUD；小团队小系统；读写模型差异不大 |
| **代价** | 读写最终一致（有延迟）；维护两套模型和同步逻辑 |

### 6.2 事件溯源（Event Sourcing）

不存当前状态，存所有变更事件流。通过**重放事件**还原任意时刻的状态。

```
传统方式：直接存 Order { status: PAID, amount: 100 }
事件溯源：存事件流
  ├── OrderCreatedEvent { amount: 100 }
  ├── OrderItemAddedEvent { productId: "A", qty: 2 }
  ├── OrderPaidEvent { paidAmount: 100 }
  └── 重放以上事件 → 还原 Order 当前状态
```

| 维度 | 优势 | 代价 |
|------|------|------|
| 审计追溯 | 完整变更历史，天然审计日志 | 事件流增长快，需要快照优化 |
| 时间旅行 | 可还原任意历史时刻状态 | 事件 schema 演进困难 |
| 适用场景 | 金融合规、高审计要求、需要回溯的业务 | 简单 CRUD 得不偿失 |

事件溯源通常与 CQRS 配合：写端存事件流，读端通过事件投影构建查询视图。

## 七、DDD 落地路径

DDD 落地不是从"建几个包"开始，而是从业务问题开始。

```
① 识别业务领域
   → 这个系统要解决什么业务问题？核心域/支撑域/通用域分别是什么？
   → 电商示例：商品、订单、用户、支付、库存、营销

② 划定限界上下文
   → 同一概念在哪些场景下含义不同？
   → "商品"在商品中心是详情，在订单中心是快照

③ 建立统一语言
   → 与业务对齐术语，落实到代码命名
   → Order.cancel() 而非 OrderService.updateStatus(CANCELLED)

④ 战术建模
   → 识别实体、值对象、聚合根、领域服务
   → 判断聚合粒度：哪些对象需要一起保持一致？

⑤ 分层架构
   → domain / application / interfaces / infrastructure
   → 领域层不依赖任何技术框架（DIP）

⑥ 持续演进
   → 业务变化后实时更新领域模型
   → 新需求先审视是否需要新增领域概念，而非在 Service 里堆 if/else
```

> DDD 不是一次性建模。如果只在项目初期画了领域模型图，后续需求全在 Service 里加逻辑，DDD 就名存实亡了。

## 八、生产风险与关键权衡

### 8.1 常见踩坑

| 风险 | 表现 | 应对 |
|------|------|------|
| 贫血模型惯性 | 习惯性把所有逻辑写在 Service，领域对象只有 getter/setter | 每次写逻辑先问"这个动作能由对象自己做吗？" |
| 领域层技术泄漏 | 领域实体上加 `@Column`、`@Table` 等 ORM 注解 | 领域对象保持纯粹，ORM 映射放基础设施层的 PO 中 |
| 聚合根过重 | 加载一个订单把用户、商品、物流全加载了 | 跨聚合用 **ID 引用**，按需加载 |
| 分布式事务滥用 | 跨聚合操作全用 XA/TCC | 优先用领域事件 + 最终一致性 |
| 过度建模 | 简单 CRUD 也强行 DDD，包结构复杂但没有真正的领域逻辑 | DDD 解决复杂业务问题，简单项目表驱动更高效 |

```java
// 错误：领域层直接依赖 ORM 注解
public class Order {
    @Column(name = "status")             // ← 技术细节泄漏到领域层
    private OrderStatus status;
}

// 正确：领域对象纯粹，PO 在基础设施层
// domain/Order.java — 纯 POJO
// infrastructure/OrderPO.java — 带 ORM 注解，与 Order 互转
```

### 8.2 关键权衡

| 权衡点 | 说明 |
|--------|------|
| 复杂度 vs 长期价值 | 初创 MVP 不建议 DDD；业务稳定且复杂后值得投入 |
| 纯粹性 vs 实用性 | 领域层理论上不应有 Spring 注解，但实战中往往有务实妥协（如 `@Value` 注入配置）|
| 聚合根粒度 | 太大 → 并发冲突多、加载慢；太小 → 分布式事务激增 |
| CQRS 引入时机 | 读写比例悬殊且查询跨聚合时有价值，否则增加维护成本 |
| 领域事件 vs 直接调用 | 事件解耦但引入最终一致性；简单场景直接调用更直观 |

## 九、面试速答

| 问题 | 速答 |
| --- | --- |
| DDD 和传统开发最大区别 | 传统从表出发，DDD 从业务领域和统一语言出发，先建模再考虑持久化 |
| 聚合根是什么 | 聚合的一致性边界和唯一入口，外部通过聚合根操作内部对象，保证事务一致性 |
| 实体和值对象区别 | 实体有唯一 ID 和生命周期（Order）；值对象无 ID、不可变、按属性值判等（Money、Address）|
| 领域事件和 MQ 消息区别 | 领域事件是领域内状态变化的建模概念，可以只在进程内流转；MQ 是跨进程传递事件的技术手段 |
| 什么时候不适合 DDD | 简单 CRUD、MVP 快速试错、业务规则很薄、团队缺少建模共识 |
| DDD 四层怎么分 | 用户接口层接请求 → 应用层编排用例 → 领域层放业务模型 → 基础设施层接 DB/MQ/外部服务 |
| 充血和贫血怎么选 | 业务规则散落多个 Service 且互相打架时收回到充血模型；简单 CRUD 用贫血更高效 |
| 防腐层（ACL）干什么 | 隔离外部不良模型，在基础设施层用 Adapter 翻译外部数据结构，防止外部变化污染核心领域 |

> **面试口径**：我对 DDD 有战术设计的实践，在支付和游戏项目中用聚合根划分事务边界——PaymentOrder 是聚合根，内含 PaymentRecord 和 RefundRecord，外部不能直接操作这两张表，只能通过 PaymentOrder 的领域方法修改状态，保证订单状态机一致性。金额用值对象封装，防止直接传裸数字导致单位错误。战略设计（上下文映射/事件风暴）实践较少，不会声称精通。
