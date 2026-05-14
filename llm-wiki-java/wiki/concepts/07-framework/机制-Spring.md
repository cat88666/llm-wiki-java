---
type: concept
status: active
name: "Spring"
layer: L6
aliases: ["Spring体系", "IoC", "DI", "IoC容器", "依赖注入", "控制反转", "BeanFactory", "ApplicationContext", "三级缓存", "循环依赖", "AOP", "面向切面编程", "切面", "代理", "Aspect", "织入", "切点", "通知", "@Transactional", "Spring事务", "事务传播", "声明式事务", "TransactionInterceptor"]
related:
  - "[[机制-动态代理]]"
  - "[[机制-SpringBoot]]"
  - "[[机制-SpringMVC]]"
  - "[[机制-MVCC]]"
  - "[[机制-InnoDB锁]]"
  - "[[概念-ThreadLocal]]"
  - "[[机制-SPI]]"
sources:
  - "../../../raw/note/Hollis/Spring/"
  - "../../../raw/note/Hollis/Spring/✅介绍一下Spring的IOC.md"
  - "../../../raw/note/Hollis/Spring/✅Spring Bean的生命周期是怎么样的？.md"
  - "../../../raw/note/Hollis/Spring/✅什么是Spring的循环依赖问题？.md"
  - "../../../raw/note/Hollis/Spring/✅什么是Spring的三级缓存.md"
  - "../../../raw/note/Hollis/Spring/✅三级缓存是如何解决循环依赖的问题的？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring 中的 Bean 作用域有哪些？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring默认支持循环依赖吗？如果发生如何解决？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring解决循环依赖一定需要三级缓存吗？.md"
  - "../../../raw/note/Hollis/Spring/✅@Lazy注解能解决循环依赖吗？.md"
  - "../../../raw/note/Hollis/Spring/✅介绍一下Spring的AOP.md"
  - "../../../raw/note/Hollis/Spring/✅Spring的AOP在什么场景下会失效？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring中用到了哪些设计模式.md"
  - "../../../raw/note/Hollis/Spring/✅Spring中@Transactional事务的实现原理.md"
  - "../../../raw/note/Hollis/Spring/✅Spring的事务传播机制有哪些？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring事务失效可能是哪些原因？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring的事务在多线程下生效吗？为什么？.md"
created: 2026-05-06
updated: 2026-05-14
lint_notes: ""
---

# Spring

> Spring 的核心是 IoC 容器和 AOP 织入：容器统一管理 Bean 的创建、依赖和生命周期，AOP 在 Bean 代理上插入事务、日志、权限等横切逻辑；`@Transactional` 是这套机制最典型的落地。

## 第一性原理

没有 Spring 时，对象通常自己 `new` 依赖、自己控制生命周期、自己重复编写事务和日志样板代码。随着系统变大，调用方会被依赖构造细节、对象复用、横切逻辑和事务边界绑住。

Spring 做了两层反转：

1. **IoC 反转对象控制权**：对象不再主动创建依赖，而是声明依赖，由容器创建、装配和管理。
2. **AOP 反转横切逻辑位置**：业务方法不再手写事务、日志、权限等重复逻辑，而是由代理在方法调用前后织入。

事务、缓存、权限、监控等能力之所以能用注解声明，本质上都依赖“Bean 被容器管理”和“调用经过代理对象”这两个前提。

## 体系全景

```
Spring Framework
├── IoC 容器
│   ├── BeanDefinition → Bean 生命周期
│   ├── singleton/prototype/request/session/application 作用域
│   └── 三级缓存 → 解决单例 setter/field 循环依赖
│
├── AOP 织入
│   ├── 运行时代理：JDK 动态代理 / CGLIB
│   ├── 5 种 Advice：Before / AfterReturning / AfterThrowing / After / Around
│   └── 典型失效：this 自调用、private/static/final、Bean 未托管
│
├── Spring 事务
│   ├── @Transactional = AOP 切面 + TransactionInterceptor
│   ├── PlatformTransactionManager 开启/提交/回滚事务
│   └── 事务上下文通过 ThreadLocal 绑定到当前线程连接
│
└── SpringMVC
    └── DispatcherServlet → HandlerMapping → Controller → ViewResolver

SpringBoot
└── 自动装配
    ├── @EnableAutoConfiguration
    ├── AutoConfigurationImportSelector
    ├── spring.factories / AutoConfiguration.imports
    └── @Conditional 条件过滤后注册 Bean
```

## IoC 容器

### Bean 的注册与获取

```
配置元数据（XML / 注解 / @Configuration）
  ↓
BeanDefinition（描述如何创建 Bean）
  ↓
BeanFactory（基础容器）
  ↓
ApplicationContext（高级容器，整合事件、资源、国际化等能力）
  ↓
调用方通过 @Autowired / @Resource / getBean() 获取依赖
```

**常见作用域**：

| 作用域 | 含义 |
|--------|------|
| `singleton` | 默认，容器内唯一实例，通常随容器启动创建 |
| `prototype` | 每次获取创建新实例，容器不负责完整销毁流程 |
| `request` | Web 场景中每个请求一个实例 |
| `session` | 每个 HTTP Session 一个实例 |
| `application` | 每个 ServletContext 一个实例 |

### Bean 生命周期

```
1. 实例化 createBeanInstance
2. 属性注入 populateBean
3. Aware 接口回调
4. BeanPostProcessor.before
5. @PostConstruct
6. InitializingBean.afterPropertiesSet
7. 自定义 init-method
8. BeanPostProcessor.after（AOP 代理通常在这里创建）
9. Bean 正常使用
10. DisposableBean.destroy
11. 自定义 destroy-method
```

初始化顺序通常记为：`@PostConstruct` > `afterPropertiesSet` > `init-method`。AOP 代理由 `AbstractAutoProxyCreator.postProcessAfterInitialization()` 在后置处理阶段创建。

### 三级缓存与循环依赖

Spring 在 `DefaultSingletonBeanRegistry` 中维护三级缓存：

| 缓存 | Map 名称 | 内容 |
|------|----------|------|
| 一级 | `singletonObjects` | 完整 Bean，已完成实例化、属性注入和初始化 |
| 二级 | `earlySingletonObjects` | 早期引用，已实例化但未完全初始化 |
| 三级 | `singletonFactories` | `ObjectFactory`，可按需生成早期引用或代理对象 |

A 依赖 B、B 又依赖 A 时，Spring 会先实例化 A，把 A 的工厂放入三级缓存；创建 B 时发现需要 A，就从三级缓存拿到 A 的早期引用，再让 B 完成初始化，最后回到 A 完成注入和初始化。

三级缓存的关键价值不是“多一层 Map”，而是**在 AOP 场景中延迟决定暴露原始对象还是代理对象**。如果只用二级缓存直接暴露原始对象，其他 Bean 可能持有非代理引用，导致事务、日志等 AOP 能力失效。

循环依赖只支持单例 Bean 的 setter/field 注入；构造器循环依赖在实例化时就需要对方，无法提前暴露早期引用。`@Lazy` 可以通过延迟初始化打破构造器依赖链，但循环依赖通常说明设计需要调整。SpringBoot 2.6 起默认不鼓励循环依赖，可通过 `spring.main.allow-circular-references=true` 显式开启。

## AOP 织入

### 核心术语

| 术语 | 含义 |
|------|------|
| Aspect | 切面，横切逻辑模块，如事务切面、日志切面 |
| JoinPoint | 可被拦截的程序执行点；Spring AOP 主要支持方法执行 |
| Pointcut | 切点，筛选哪些 JoinPoint 需要被拦截 |
| Advice | 通知，在连接点执行的逻辑 |
| Weaving | 织入，把切面应用到目标对象并创建代理的过程 |

**5 种 Advice**：

| 类型 | 时机 |
|------|------|
| `@Before` | 方法执行前 |
| `@AfterReturning` | 方法正常返回后 |
| `@AfterThrowing` | 方法抛异常后 |
| `@After` | 无论正常或异常，方法结束后 |
| `@Around` | 完全包裹方法调用，可控制是否执行 `proceed()` |

### 代理实现

Spring AOP 运行时通过代理实现织入：

```
目标类实现了接口？
  ├── 是 → JDK 动态代理，基于接口生成实现类
  └── 否 → CGLIB 代理，基于继承生成子类
```

SpringBoot 2.x 后通常默认使用 CGLIB。无论哪种方式，AOP 的核心约束都是：**调用必须经过代理对象**。

### AOP 失效场景

| 失效场景 | 原因 |
|----------|------|
| 类内部 `this` 自调用 | `this` 指向原始对象，不经过代理 |
| `private` 方法 | 代理无法拦截私有方法 |
| `static` 方法 | 静态方法属于类本身，不属于代理对象 |
| `final` 方法 | CGLIB 依赖子类重写，`final` 无法重写 |
| Bean 未被 Spring 管理 | 没有容器创建的代理对象 |
| 内部类调用外部类方法 | 通常仍是原始对象内部调用，不经过代理 |

自调用可通过拆分到另一个 Bean、从 `ApplicationContext` 获取代理 Bean、或使用 `AopContext.currentProxy()` 处理。`@Around` 中必须调用 `proceed()`，否则目标方法不会执行；捕获异常后也要按业务语义重新抛出或转换，否则上层切面看不到异常。

## 声明式事务

`@Transactional` 是 Spring AOP 的典型应用。手动事务需要 `setAutoCommit(false)`、执行业务、`commit`、异常时 `rollback`，样板代码重复且容易遗漏；Spring 用 `TransactionInterceptor` 把这套流程封装为 Around Advice。

```
1. 启动时：AnnotationTransactionAttributeSource 解析 @Transactional 属性
2. Bean 初始化后：事务切面为目标 Bean 创建 AOP 代理
3. 方法调用时：TransactionInterceptor 拦截
   ├── PlatformTransactionManager 开启事务
   ├── 执行业务方法
   ├── 正常返回 → commit
   └── 抛出需回滚异常 → rollback
```

核心代码路径通常落在 `TransactionAspectSupport.invokeWithinTransaction()`。

### 事务传播机制

| 传播行为 | 行为描述 | 场景 |
|----------|----------|------|
| `REQUIRED` | 默认；有事务则加入，没有则新建 | 绝大多数业务方法 |
| `REQUIRES_NEW` | 总是新建事务，挂起当前事务 | 独立日志、独立流水，不受主事务回滚影响 |
| `SUPPORTS` | 有事务则加入，没有则普通执行 | 不强制事务的只读查询 |
| `NOT_SUPPORTED` | 挂起当前事务，以非事务方式执行 | 避免长事务包住不必要逻辑 |
| `MANDATORY` | 必须在已有事务中执行，否则抛异常 | 强制调用方提供事务边界 |
| `NEVER` | 不能在事务中执行，否则抛异常 | 明确禁止事务的操作 |
| `NESTED` | 已有事务时创建 Savepoint | 部分回滚场景 |

### 事务失效场景

| 类型 | 典型原因 |
|------|----------|
| 代理未生效 | `private/static/final`、`this` 自调用、Bean 未被容器管理 |
| 配置不符合预期 | `NOT_SUPPORTED` / `NEVER`、checked exception 未配置 `rollbackFor` |
| 异常被吞 | 业务代码或其他切面 catch 后未重新抛出 |
| 跨线程 | `@Async` 或事务内新建线程，事务上下文不传播 |
| 注解用错 | 使用了非 Spring 的 `javax.transaction.Transactional` 时语义可能不一致 |

事务上下文与数据库连接通常通过 `ThreadLocal` 绑定到当前线程。跨线程意味着使用新的执行上下文和连接，原事务不会自动传播。

## 关键权衡

1. **单例 Bean 的线程安全**：默认 `singleton` 只代表实例唯一，不代表内部状态线程安全；Service/Repository 通常应保持无状态。
2. **循环依赖是兜底机制**：三级缓存能处理一部分单例循环依赖，但构造器循环依赖和 prototype 循环依赖无法解决，应优先重构依赖方向。
3. **AOP 简化横切逻辑，也隐藏调用路径**：调试时要识别 `$Proxy`、`$$EnhancerByCGLIB` 和代理边界。
4. **声明式事务简洁但有边界陷阱**：需要避开自调用、异常吞噬、跨线程和长事务。
5. **事务隔离和数据库实现相关**：隔离级别、[[机制-MVCC]]、[[机制-InnoDB锁]] 共同决定可见性、锁等待和并发性能。

## 与其他概念的关系

- 底层依赖 [[机制-动态代理]]：AOP 和 `@Transactional` 都靠代理对象拦截方法调用。
- 被 [[机制-SpringBoot]] 扩展：自动装配本质是向 Spring 容器按条件注册 BeanDefinition。
- 与 [[机制-SpringMVC]] 协作：Controller、Service、HandlerMapping 等都运行在 Spring 容器中。
- 与 [[概念-ThreadLocal]] 相关：事务连接绑定到线程，解释了多线程事务失效。
- 与 [[机制-SPI]] 相关：SpringBoot 自动配置文件体现了按约定发现扩展点的思想。
- 与 [[机制-MVCC]]、[[机制-InnoDB锁]] 相关：数据库事务的隔离和锁行为由存储引擎真正执行。

## 应用边界

**适合 Spring 容器管理**：Service、Repository、Controller、配置组件，以及需要事务、日志、权限等横切能力的对象。

**不适合强行交给容器**：频繁创建销毁的领域实体、简单值对象、必须携带运行时参数构造的对象；这些通常用普通构造、工厂方法或原型作用域处理。

**适合 `@Transactional`**：单数据源内需要原子性的增删改组合。

**不适合只靠 `@Transactional`**：分布式事务、跨线程事务、包含外部接口调用的长事务；这些需要拆分事务边界、使用 Outbox/TCC/Seata 等方案或改成最终一致性设计。
