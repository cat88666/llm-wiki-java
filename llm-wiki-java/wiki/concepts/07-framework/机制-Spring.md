---
type: concept
status: active
name: "Spring"
layer: L6
aliases: ["Spring体系", "IoC", "DI", "IoC容器", "依赖注入", "控制反转", "BeanFactory", "ApplicationContext", "BeanDefinition", "BeanDefinitionMap", "BeanFactoryPostProcessor", "BeanPostProcessor", "三级缓存", "循环依赖", "AOP", "面向切面编程", "切面", "代理", "Aspect", "织入", "切点", "通知", "@Transactional", "Spring事务", "事务传播", "声明式事务", "TransactionInterceptor"]
related:
  - "[[机制-动态代理]]"
  - "[[机制-SpringBoot]]"
  - "[[机制-SpringMVC]]"
  - "[[概念-MySQL]]"
  - "[[概念-ThreadLocal]]"
  - "[[机制-SPI]]"
sources:
  - "../../../raw/note/Hollis/Spring/"
  - "../../../raw/note/Hollis/其他专属内容/为啥我不建议使用@Transactional事务.md"
updated: 2026-05-18
---

# Spring

> Spring 的核心是把对象创建、依赖装配、生命周期、横切增强和事务边界交给容器管理；收益是解耦和统一治理，代价是调用必须符合容器与代理规则。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、Spring 的核心抽象](#sec-2) | IoC 反转对象控制权，AOP 反转横切逻辑位置 |
| [二、IoC 容器启动链路](#sec-3) | BeanDefinition、BeanFactory、ApplicationContext |
| [三、Bean 生命周期与扩展点](#sec-4) | BFPP、BPP、Aware、初始化与销毁 |
| [四、循环依赖与三级缓存](#sec-5) | 单例属性注入如何提前暴露引用 |
| [五、AOP 代理增强链路](#sec-6) | 代理、Advice、切点匹配、调用入口 |
| [六、Spring 事务核心原理和流程](#sec-7) | TransactionInterceptor、事务管理器、两条 insert 落库/回滚、传播行为 |
| [七、设计权衡与使用原则](#sec-8) | 容器管理边界、事务边界、线程边界 |
| [八、生产风险与排查](#sec-9) | Bean 冲突、代理失效、循环依赖、事务失效 |
| [九、关系与边界](#sec-10) | 与 Boot、MVC、MyBatis、ThreadLocal 的关系 |
| [十、面试速答口径](#sec-11) | 高频问题答案 |

<a id="sec-2"></a>
## 一、Spring 的核心抽象

Spring 解决两类重复问题：

1. 对象如何创建、装配、复用、销毁。
2. 事务、日志、权限、缓存这类横切逻辑如何统一织入。

```text
IoC 容器
  -> BeanDefinition
  -> Bean 创建与依赖注入
  -> 生命周期回调

AOP
  -> 为 Bean 创建代理
  -> 在方法调用前后执行横切逻辑
```

关键前提：对象必须由 Spring 容器管理，方法调用必须经过代理对象。

<a id="sec-3"></a>
## 二、IoC 容器启动链路

```text
扫描配置与注解
  -> 生成 BeanDefinition
  -> 注册到 BeanDefinitionMap
  -> 执行 BeanFactoryPostProcessor
  -> 创建非懒加载 singleton
  -> 执行 BeanPostProcessor
  -> 发布容器刷新完成事件
```

BeanFactory vs ApplicationContext：

| 维度 | BeanFactory | ApplicationContext |
| --- | --- | --- |
| 定位 | IoC 底座 | 应用级上下文 |
| 能力 | Bean 创建与获取 | 事件、资源、国际化、环境、Web 集成 |
| 使用场景 | 框架底层 | 业务应用默认选择 |

<a id="sec-4"></a>
## 三、Bean 生命周期与扩展点

### 3.1 生命周期主线

```text
实例化
  -> 属性填充
  -> Aware 回调
  -> BeanPostProcessor.before
  -> @PostConstruct
  -> InitializingBean.afterPropertiesSet
  -> init-method
  -> BeanPostProcessor.after（AOP 代理常在这里）
  -> 使用
  -> destroy
```

### 3.2 两类扩展点

| 扩展点 | 时机 | 处理对象 | 典型能力 |
| --- | --- | --- | --- |
| `BeanFactoryPostProcessor` | Bean 实例化前 | BeanDefinition | 修改定义、注册新定义 |
| `BeanPostProcessor` | Bean 创建过程中 | Bean 实例 | 注入、初始化、AOP 代理 |

一句话：BFPP 改“图纸”，BPP 改“成品”。

<a id="sec-5"></a>
## 四、循环依赖与三级缓存

Spring 只解决单例 Bean 的 setter/field 循环依赖，不解决构造器循环依赖和 prototype 循环依赖。

| 缓存 | 内容 | 作用 |
| --- | --- | --- |
| 一级 `singletonObjects` | 完整 Bean | 正常可用对象 |
| 二级 `earlySingletonObjects` | 早期引用 | 解决注入时引用 |
| 三级 `singletonFactories` | 对象工厂 | 延迟决定原始对象还是代理对象 |

三级缓存真正价值：在出现循环依赖且 Bean 需要 AOP 时，通过工厂提前暴露代理对象，避免其他 Bean 注入原始对象。

SpringBoot 2.6 起默认不鼓励循环依赖；遇到循环依赖优先重构，`@Lazy` 只是过渡手段。

<a id="sec-6"></a>
## 五、AOP 代理增强链路

### 5.1 AOP 基础链路

```text
目标 Bean 初始化后
  -> AutoProxyCreator 判断是否需要增强
  -> JDK 动态代理或 CGLIB 创建代理
  -> 调用代理方法
  -> Advice 链执行
  -> 目标方法执行
```

声明式事务本质上是 AOP 的一种应用：Spring 通过代理拦截 `@Transactional` 方法，在业务方法前后插入开启事务、提交事务、回滚事务等横切逻辑。

<a id="sec-7"></a>
## 六、Spring 事务核心原理和流程

Spring 事务不是自己实现数据库事务，而是用统一抽象屏蔽 JDBC、JPA、MyBatis 等资源差异，再把事务边界织入到代理方法调用中。

### 6.1 核心组件

| 组件 | 职责 | 关键点 |
| --- | --- | --- |
| `@Transactional` | 声明事务属性 | 传播行为、隔离级别、超时、只读、回滚规则 |
| `TransactionAttributeSource` | 解析事务元数据 | 从方法、类、接口上读取事务配置 |
| `TransactionInterceptor` | 事务 AOP 拦截器 | 围绕目标方法执行开启、提交、回滚 |
| `PlatformTransactionManager` | 事务管理器抽象 | JDBC 用 `DataSourceTransactionManager`，JPA 用 `JpaTransactionManager` |
| `TransactionStatus` | 当前事务状态 | 是否新事务、是否回滚、保存点、挂起资源 |
| `TransactionSynchronizationManager` | 线程上下文管理 | 用 [[概念-ThreadLocal]] 绑定连接、同步回调和事务属性 |

一句话：`@Transactional` 只负责声明，真正执行链路是 `TransactionInterceptor -> PlatformTransactionManager -> 数据库连接/事务资源`。

### 6.2 执行流程

```text
外部调用 Spring 代理对象的方法
  -> 进入 TransactionInterceptor
  -> 读取 @Transactional 事务属性
  -> 根据传播行为判断加入事务 / 新建事务 / 挂起旧事务
  -> PlatformTransactionManager 获取连接并关闭 autoCommit
  -> TransactionSynchronizationManager 将连接绑定到当前线程
  -> 执行业务方法
       ├── 正常返回
       │    -> beforeCommit / beforeCompletion
       │    -> commit
       │    -> afterCommit / afterCompletion
       └── 抛出异常
            -> 判断是否匹配 rollbackFor / 默认 RuntimeException 或 Error
            -> rollback 或 commit
  -> 清理 ThreadLocal 资源，恢复被挂起事务
```

核心点：业务代码里通过 MyBatis/JdbcTemplate 获取连接时，拿到的是当前线程已绑定的同一个连接，所以同一事务内多次数据库操作才能落在同一个数据库事务中。

### 6.3 两条 insert 的落库与回滚案例

业务代码：

```java
@Service
public class OrderService {
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(CreateOrderCommand cmd) {
        orderMapper.insertOrder(cmd.order());        // SQL 1
        orderMapper.insertOrderItem(cmd.orderItem()); // SQL 2
    }
}
```

成功提交流程：

```text
Controller 调用 orderService 代理对象
  -> TransactionInterceptor 拦截 createOrder()
  -> DataSourceTransactionManager 从 DataSource 获取 Connection
  -> connection.setAutoCommit(false)                 ← 关闭自动提交，事务开始
  -> TransactionSynchronizationManager 绑定 Connection 到当前线程
  -> orderMapper.insertOrder()
       -> MyBatis 通过 DataSourceUtils 获取当前线程绑定的 Connection
       -> 执行 SQL 1：INSERT INTO orders ...
       -> 数据库写入当前事务的 undo/redo/binlog 相关记录，但未对外提交
  -> orderMapper.insertOrderItem()
       -> 仍然拿到同一个 Connection
       -> 执行 SQL 2：INSERT INTO order_item ...
  -> 业务方法正常返回
  -> TransactionInterceptor 调用 commit
  -> connection.commit()                              ← 两条 insert 一起提交
  -> 释放连接，清理 ThreadLocal
```

失败回滚流程：

```text
Controller 调用 orderService 代理对象
  -> 开启事务并绑定同一个 Connection
  -> SQL 1 执行成功：订单主表 insert 进入当前数据库事务
  -> SQL 2 执行失败：唯一键冲突 / 外键失败 / SQL 异常
  -> 异常向外抛到 TransactionInterceptor
  -> 匹配 rollbackFor = Exception.class
  -> TransactionInterceptor 调用 rollback
  -> connection.rollback()                            ← 数据库撤销本事务内 SQL 1 和 SQL 2 的影响
  -> 释放连接，清理 ThreadLocal
  -> 异常继续向上抛给调用方
```

关键原理：

| 关键点 | 说明 |
| --- | --- |
| 两条 SQL 为什么在同一事务 | Spring 把同一个 `Connection` 绑定到当前线程，MyBatis/JdbcTemplate 后续取连接时复用它 |
| 为什么 SQL 1 成功后还能回滚 | `autoCommit=false` 后 SQL 1 只是进入当前数据库事务，未执行 `commit` 前对外不算最终落库 |
| 真正提交发生在哪里 | 业务方法正常返回后，`TransactionInterceptor` 调用 `PlatformTransactionManager.commit()`，最终落到 `Connection.commit()` |
| 真正回滚发生在哪里 | 方法抛出匹配回滚规则的异常后，拦截器调用 `Connection.rollback()`，由数据库引擎撤销本事务改动 |
| Spring 负责什么 | 代理拦截、连接绑定、提交/回滚时机、资源清理 |
| 数据库负责什么 | ACID、锁、undo/redo、隔离级别、最终 commit/rollback |

反直觉点：`insert` 方法返回成功不代表已经提交，只代表 SQL 在当前事务内执行成功；只要外层事务还没 `commit`，后续异常仍可回滚。

### 6.4 传播行为的底层语义

| 传播行为 | 当前有事务 | 当前无事务 | 底层动作 |
| --- | --- | --- | --- |
| `REQUIRED` | 加入当前事务 | 新建事务 | 默认选择，最常用 |
| `REQUIRES_NEW` | 挂起当前事务，新建事务 | 新建事务 | 两个事务独立提交/回滚 |
| `NESTED` | 创建保存点 | 新建事务 | 依赖 JDBC Savepoint，外层回滚会影响内层 |
| `SUPPORTS` | 加入当前事务 | 非事务执行 | 查询类方法可用 |
| `MANDATORY` | 加入当前事务 | 抛异常 | 强制要求上游已开事务 |
| `NOT_SUPPORTED` | 挂起当前事务，非事务执行 | 非事务执行 | 不希望参与事务的慢操作 |
| `NEVER` | 抛异常 | 非事务执行 | 明确禁止事务 |

`REQUIRES_NEW` 是“挂起旧连接、重新拿连接开新事务”；`NESTED` 是“同一个连接内创建保存点”。两者不是一回事。

### 6.5 提交、回滚与失效根因

Spring 默认只对 `RuntimeException` 和 `Error` 回滚，受检异常默认提交；需要回滚受检异常时必须显式配置 `rollbackFor = Exception.class`。

| 场景 | 结果 | 原因 |
| --- | --- | --- |
| 同类方法 `this.inner()` 调用 | 事务不生效 | 调用未经过代理对象 |
| `private/final/static` 方法加事务 | 通常不生效 | 代理无法覆盖或拦截 |
| 异常被 catch 后未抛出 | 事务提交 | 拦截器看不到异常 |
| 子线程内写库 | 不参与父事务 | 事务资源绑定在线程 ThreadLocal |
| 数据库引擎不支持事务 | Spring 配置无效 | 例如 MySQL MyISAM 不支持事务 |

### 6.6 即用面试口径

> Spring 声明式事务是基于 AOP 代理实现的。外部调用代理方法时，`TransactionInterceptor` 先解析 `@Transactional`，再交给 `PlatformTransactionManager` 获取连接、关闭自动提交，并把连接绑定到当前线程的 `ThreadLocal` 中。同一个事务内的两条 insert 会复用同一个连接；方法正常返回时统一 `commit`，第二条 insert 抛异常且匹配回滚规则时统一 `rollback`，第一条已经执行成功的 insert 也会被数据库撤销。因此事务失效通常不是数据库问题，而是调用没经过代理、异常被吞、跨线程或传播行为配置不符合预期。

<a id="sec-8"></a>
## 七、设计权衡与使用原则

1. 能被切面增强的逻辑必须放在 Spring Bean 的 public 方法调用链上。
2. 事务边界放在 Service 层，避免 Controller 或 DAO 层事务语义混乱。
3. 不把所有对象都交给容器；值对象、DTO、临时策略对象可普通创建。
4. 循环依赖不是能力点，是设计异味。

<a id="sec-9"></a>
## 八、生产风险与排查

| 风险 | 现象 | 排查点 |
| --- | --- | --- |
| Bean 冲突 | 启动失败或注入错误 | beanName、`@Primary`、`@Qualifier` |
| AOP 失效 | 事务/缓存不生效 | 是否自调用、是否 public、是否 Spring Bean |
| 事务不回滚 | 异常被吞或类型不匹配 | rollbackFor、异常链、代理入口 |
| 多线程事务失效 | 子线程写库不回滚 | ThreadLocal 线程隔离 |
| 循环依赖 | 启动报依赖环 | 构造器注入、服务职责拆分 |

<a id="sec-10"></a>
## 九、关系与边界

- [[机制-SpringBoot]] 是 Spring 的自动配置与运行时封装。
- [[机制-SpringMVC]] 依赖 Spring 容器管理 Controller 与 Web 组件。
- [[机制-动态代理]] 是 AOP 和事务的底层能力。
- [[概念-MySQL]] 事务能力由 Spring 事务管理器统一协调。

边界：Spring 管理对象和横切逻辑，不保证业务建模正确，也不替代分布式事务和跨线程上下文治理。

<a id="sec-11"></a>
## 十、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| IoC 是什么 | 对象创建和依赖装配控制权交给容器 |
| BeanFactory 和 ApplicationContext 区别 | 前者是底座，后者是应用级上下文，集成事件/资源/环境 |
| 三级缓存为什么需要第三级 | 延迟生成早期代理引用，兼容 AOP 和循环依赖 |
| Spring 事务为什么会失效 | 调用没经过代理、方法不可代理、异常被吞、多线程切换 |
| `@Transactional` 多线程生效吗 | 不自动生效，事务上下文在 ThreadLocal 中按线程隔离 |
