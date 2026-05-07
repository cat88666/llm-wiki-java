---
type: concept
status: active
name: "Spring事务"
layer: L6
aliases: ["@Transactional", "Spring事务", "事务传播", "声明式事务", "TransactionInterceptor"]
related:
  - "[[机制-AOP织入]]"
  - "[[机制-IoC容器]]"
  - "[[机制-MVCC]]"
  - "[[机制-InnoDB锁机制]]"
sources:
  - "../../../raw/note/Hollis/Spring/✅Spring中@Transactional事务的实现原理.md"
  - "../../../raw/note/Hollis/Spring/✅Spring的事务传播机制有哪些？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring事务失效可能是哪些原因？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring的事务在多线程下生效吗？为什么？.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# Spring事务

> `@Transactional` 是 AOP 切面的声明式封装：Spring 在方法调用前后插入事务开始/提交/回滚逻辑，开发者只需标注注解，无需手写 JDBC 事务代码。

## 第一性原理

手动管理事务（`conn.setAutoCommit(false)` → 业务逻辑 → `conn.commit()` / `conn.rollback()`）的样板代码重复且容易遗漏。Spring 用 AOP 将这个模式抽取为 `TransactionInterceptor`，通过 `@Transactional` 注解声明"这个方法需要事务"，其余由框架处理。

## 核心机制

### @Transactional 实现原理

```
1. 启动时：AnnotationTransactionAttributeSource 解析 @Transactional 属性
2. Bean 初始化后：TransactionProxyFactoryBean 为目标方法创建 AOP 代理
3. 方法调用时：TransactionInterceptor（Around Advice）拦截
   ├── 调用 PlatformTransactionManager 开启事务
   ├── 执行目标方法
   ├── 正常返回 → commit
   └── 抛出指定异常 → rollback
```

核心代码位于 `TransactionAspectSupport.invokeWithinTransaction()`。

### 7 种事务传播机制（高频考点）

| 传播行为 | 行为描述 | 场景 |
|----------|---------|------|
| **REQUIRED**（默认）| 有事务则加入，没有则新建 | 绝大多数业务方法 |
| **REQUIRES_NEW** | 总是新建事务，挂起当前事务 | 独立操作（如记录日志，不受主事务回滚影响）|
| **SUPPORTS** | 有事务则加入，没有则普通执行 | 只读查询（不强制要求事务）|
| **NOT_SUPPORTED** | 挂起当前事务，以非事务方式执行 | 不参与事务的操作（避免无谓的锁竞争）|
| **MANDATORY** | 必须在事务中执行，否则抛异常 | 强制调用方开启事务 |
| **NEVER** | 不能在事务中执行，否则抛异常 | 禁止事务的操作 |
| **NESTED** | 如有事务则创建嵌套事务（Savepoint），子回滚不影响父 | 部分回滚场景 |

**典型场景**：长事务 A（读+写）中调用读方法 B：
- B 是中间步骤 → `REQUIRED`（B 失败需要 A 回滚）
- B 是最后一步 → `NOT_SUPPORTED`（B 读失败不应回滚写操作）

### 事务失效场景（高频考点）

**因代理失效（AOP 不起作用）**：
1. `private` / `static` / `final` 方法（代理无法拦截）
2. 类内部 `this` 自调用（不经过代理）
3. Bean 未被 Spring 管理（没有代理对象）

**因配置错误**：
4. `propagation` 设置为 `NOT_SUPPORTED` / `NEVER`（显式不使用事务）
5. `rollbackFor` 未包含实际抛出的异常（默认只回滚 `RuntimeException` 和 `Error`，checked exception 不回滚）
6. 用了 `javax.transaction.Transactional` 而非 `org.springframework.transaction.annotation.Transactional`

**因异常被吞**：
7. 异常被 catch 后未重新抛出（事务看不到异常，无法回滚）
8. 其他 AOP 切面捕获了异常（如日志切面 catch 了异常）

**多线程下失效**：
9. `@Transactional` 与 `@Async` 同时使用（异步线程开了新连接，不在同一事务中）
10. 事务内启新线程执行 DB 操作（新线程无事务上下文）

**根本原因**：事务绑定到 `Connection`，而 `Connection` 通过 `ThreadLocal` 与当前线程绑定。跨线程即意味着跨连接，事务无法传播。

## 关键权衡

1. **隔离级别与性能**：`SERIALIZABLE` 最安全但并发度最低；`READ_COMMITTED` 是生产常见选择（防脏读，允许不可重复读）；与 [[机制-MVCC]] 共同决定 MySQL 的可见性行为
2. **长事务的危害**：事务持续时间越长，持有的数据库锁越久，并发性能越差（见 [[机制-InnoDB锁机制]]）；避免在事务中调用外部接口、执行慢查询
3. **声明式 vs 编程式**：`@Transactional` 声明式简洁但有失效陷阱；`TransactionTemplate` 编程式灵活可控，适合需要精细控制事务边界的场景

## 与其他概念的关系

- 实现依赖 [[机制-AOP织入]]：`TransactionInterceptor` 是 Around Advice，代理失效则事务失效
- 运行在 [[机制-IoC容器]] 管理的 Bean 上：Bean 不被 Spring 管理则无代理
- 事务隔离级别对应 [[机制-MVCC]] 的 RC/RR 快照读行为
- 事务中的锁由 [[机制-InnoDB锁机制]] 负责，长事务导致锁时间延长

## 应用边界

**适合 @Transactional**：单数据源的增删改操作组合；需要原子性保证的业务流程。

**不适合或需谨慎**：分布式事务（需要 Seata/TCC 等，`@Transactional` 只管单数据源）；超长耗时操作（拆短事务）；与 `@Async` 共存（不在同一线程，事务不传播）。
