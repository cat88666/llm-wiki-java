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
  - "../../../raw/note/tuling/06-spring.md"
created: 2026-05-06
updated: 2026-05-16
lint_notes: ""
---

# Spring

> Spring 的核心是 IoC 容器和 AOP 织入：容器统一管理 Bean 的创建、依赖和生命周期，AOP 在 Bean 代理上插入事务、日志、权限等横切逻辑；`@Transactional` 是这套机制最典型的落地。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | IoC 反转对象控制权、AOP 反转横切逻辑 |
| [二、核心机制——IoC 容器](#二核心机制ioc-容器) | 启动流程、BeanDefinition、生命周期、作用域、三级缓存、循环依赖 |
| [三、核心机制——AOP 织入](#三核心机制aop-织入) | 术语、代理实现、五种 Advice、失效场景 |
| [四、核心机制——声明式事务](#四核心机制声明式事务) | @Transactional 原理、事务传播、事务失效 |
| [五、SpringMVC 调用链](#五springmvc-调用链) | DispatcherServlet、HandlerMapping、HandlerAdapter、ViewResolver |
| [六、关键权衡](#六关键权衡) | 线程安全、循环依赖、调试、事务边界 |
| [七、与其他概念的关系](#七与其他概念的关系) | 动态代理、SpringBoot、SpringMVC、ThreadLocal、MVCC |
| [八、应用边界](#八应用边界) | 适合 / 不适合容器管理、适合 / 不适合 @Transactional |

## 一、第一性原理

没有 Spring 时，对象通常自己 `new` 依赖、自己控制生命周期、自己重复编写事务和日志样板代码。随着系统变大，调用方会被依赖构造细节、对象复用、横切逻辑和事务边界绑住。

Spring 做了两层反转：

1. **IoC 反转对象控制权**：对象不再主动创建依赖，而是声明依赖，由容器创建、装配和管理。
2. **AOP 反转横切逻辑位置**：业务方法不再手写事务、日志、权限等重复逻辑，而是由代理在方法调用前后织入。

事务、缓存、权限、监控等能力之所以能用注解声明，本质上都依赖"Bean 被容器管理"和"调用经过代理对象"这两个前提。

### 体系全景

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

## 二、核心机制——IoC 容器

### 2.0 容器启动整体流程

Spring 是一个快速开发框架，核心能力是通过 IoC 管理对象，通过 AOP/事务/事件等扩展机制承载通用能力。它的源码实现大量使用设计模式、面向接口编程和并发安全容器。

容器启动可以简化为五段：

```
扫描配置与组件
  ↓
生成 BeanDefinition，并放入 BeanDefinitionMap
  ↓
执行 BeanFactoryPostProcessor，允许修改 BeanDefinition
  ↓
创建非懒加载 singleton Bean（prototype 获取时才创建）
  ↓
发布容器启动事件，ApplicationContext 启动完成
```

更细一点看：

1. **扫描组件**：扫描包路径，解析 `@Component`、`@Configuration`、`@Bean`、`@Import` 等元数据，生成 `BeanDefinition`。
2. **注册定义**：将 `BeanDefinition` 放入 `BeanDefinitionMap`，此时还没有真正创建业务对象。
3. **扩展定义**：执行 `BeanFactoryPostProcessor`，允许在 Bean 实例化前修改 BeanDefinition；`@ComponentScan`、`@Import` 等能力也会在这一阶段参与注册。
4. **创建 Bean**：创建非懒加载的单例 Bean；多例 Bean 不会在启动阶段统一创建，而是在每次 `getBean()` 时创建。
5. **生命周期增强**：执行构造推断、实例化、依赖注入、Aware 回调、初始化前后处理；AOP 代理通常发生在初始化后。
6. **事件发布**：发布容器刷新完成事件，ApplicationContext 对外可用。

### 2.1 Bean 的注册与获取

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

### 2.1.1 BeanFactory vs ApplicationContext

| 维度 | BeanFactory | ApplicationContext |
| --- | --- | --- |
| 定位 | Spring 最底层 IoC 容器 | 企业开发常用高级容器 |
| 核心职责 | Bean 创建、维护和获取 | 继承 BeanFactory，并整合更多上下文能力 |
| 环境能力 | 较弱 | `EnvironmentCapable`，支持环境变量、Profile、配置属性 |
| 国际化 | 无内置完整上下文 | `MessageSource` |
| 事件机制 | 需额外组合 | `ApplicationEventPublisher` |
| 资源加载 | 基础能力 | 更完整的 Resource 体系 |

一句话：**BeanFactory 是容器底座，ApplicationContext 是面向应用开发的完整上下文**。企业开发基本都使用 ApplicationContext。

### 2.2 Bean 作用域

| 作用域 | 含义 |
|--------|------|
| `singleton` | 默认，容器内唯一实例，通常随容器启动创建 |
| `prototype` | 每次获取创建新实例，容器不负责完整销毁流程 |
| `request` | Web 场景中每个请求一个实例 |
| `session` | 每个 HTTP Session 一个实例 |
| `application` | 每个 ServletContext 一个实例 |

### 2.3 Bean 生命周期

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

面试中也可以压缩为 7 步：

1. 推断构造方法。
2. 实例化。
3. 属性填充，也就是依赖注入。
4. Aware 接口回调。
5. 初始化前：`BeanPostProcessor.before`、`@PostConstruct`。
6. 初始化：`InitializingBean.afterPropertiesSet()`、自定义 `init-method`。
7. 初始化后：`BeanPostProcessor.after`，AOP 代理通常在这里生成。

### 2.3.1 两类核心扩展点

| 扩展点 | 执行时机 | 能做什么 | 典型能力 |
| --- | --- | --- | --- |
| `BeanFactoryPostProcessor` | Bean 实例化前，BeanDefinition 已注册后 | 修改 BeanDefinition、追加 BeanDefinition | `@ComponentScan`、`@Import`、配置类解析 |
| `BeanPostProcessor` | Bean 创建过程中，初始化前后 | 参与依赖注入、初始化增强、返回代理对象 | `@Autowired` 注入、AOP 代理、`@PostConstruct` |

这两个扩展点的区别是：**BFPP 处理 Bean 的定义，BPP 处理 Bean 的实例**。很多 Spring 能力看起来是“注解魔法”，本质上都是在这两个阶段挂入扩展逻辑。

### 2.4 三级缓存与循环依赖

Spring 在 `DefaultSingletonBeanRegistry` 中维护三级缓存：

| 缓存 | Map 名称 | 内容 |
|------|----------|------|
| 一级 | `singletonObjects` | 完整 Bean，已完成实例化、属性注入和初始化 |
| 二级 | `earlySingletonObjects` | 早期引用，已实例化但未完全初始化 |
| 三级 | `singletonFactories` | `ObjectFactory`，可按需生成早期引用或代理对象 |

**解决流程**：A 依赖 B、B 又依赖 A 时，Spring 会先实例化 A，把 A 的工厂放入三级缓存；创建 B 时发现需要 A，就从三级缓存拿到 A 的早期引用，再让 B 完成初始化，最后回到 A 完成注入和初始化。

**三级缓存的关键价值**不是"多一层 Map"，而是**在 AOP 场景中延迟决定暴露原始对象还是代理对象**。如果只用二级缓存直接暴露原始对象，其他 Bean 可能持有非代理引用，导致事务、日志等 AOP 能力失效。

**循环依赖的边界**：

| 场景 | 是否支持 | 原因 |
|------|---------|------|
| 单例 setter/field 注入 | 支持 | 可提前暴露早期引用 |
| 构造器循环依赖 | 不支持 | 实例化时就需要对方，无法提前暴露 |
| prototype 循环依赖 | 不支持 | prototype 不缓存 |

`@Lazy` 可以通过延迟初始化打破构造器依赖链，但循环依赖通常说明设计需要调整。SpringBoot 2.6 起默认不鼓励循环依赖，可通过 `spring.main.allow-circular-references=true` 显式开启。

## 三、核心机制——AOP 织入

### 3.1 核心术语

| 术语 | 含义 |
|------|------|
| Aspect | 切面，横切逻辑模块，如事务切面、日志切面 |
| JoinPoint | 可被拦截的程序执行点；Spring AOP 主要支持方法执行 |
| Pointcut | 切点，筛选哪些 JoinPoint 需要被拦截 |
| Advice | 通知，在连接点执行的逻辑 |
| Weaving | 织入，把切面应用到目标对象并创建代理的过程 |

### 3.2 五种 Advice

| 类型 | 时机 |
|------|------|
| `@Before` | 方法执行前 |
| `@AfterReturning` | 方法正常返回后 |
| `@AfterThrowing` | 方法抛异常后 |
| `@After` | 无论正常或异常，方法结束后 |
| `@Around` | 完全包裹方法调用，可控制是否执行 `proceed()` |

### 3.3 代理实现

Spring AOP 运行时通过代理实现织入：

```
目标类实现了接口？
  ├── 是 → JDK 动态代理，基于接口生成实现类
  └── 否 → CGLIB 代理，基于继承生成子类
```

SpringBoot 2.x 后通常默认使用 CGLIB。无论哪种方式，AOP 的核心约束都是：**调用必须经过代理对象**。

### 3.4 AOP 失效场景（高频考点）

| 失效场景 | 原因 |
|----------|------|
| 类内部 `this` 自调用 | `this` 指向原始对象，不经过代理 |
| `private` 方法 | 代理无法拦截私有方法 |
| `static` 方法 | 静态方法属于类本身，不属于代理对象 |
| `final` 方法 | CGLIB 依赖子类重写，`final` 无法重写 |
| Bean 未被 Spring 管理 | 没有容器创建的代理对象 |
| 内部类调用外部类方法 | 通常仍是原始对象内部调用，不经过代理 |

**自调用解法**：拆分到另一个 Bean、从 `ApplicationContext` 获取代理 Bean、或使用 `AopContext.currentProxy()`。

**`@Around` 注意**：必须调用 `proceed()`，否则目标方法不会执行；捕获异常后也要按业务语义重新抛出或转换，否则上层切面看不到异常。

## 四、核心机制——声明式事务

`@Transactional` 是 Spring AOP 的典型应用。手动事务需要 `setAutoCommit(false)`、执行业务、`commit`、异常时 `rollback`，样板代码重复且容易遗漏；Spring 用 `TransactionInterceptor` 把这套流程封装为 Around Advice。

### 4.1 执行链路

```
1. 启动时：AnnotationTransactionAttributeSource 解析 @Transactional 属性
2. Bean 初始化后：事务切面为目标 Bean 创建 AOP 代理
3. 方法调用时：TransactionInterceptor 拦截
   ├── PlatformTransactionManager 开启事务
   ├── 执行业务方法
   ├── 正常返回 → commit
   └── 抛出需回滚异常 → rollback
```

核心代码路径：`TransactionAspectSupport.invokeWithinTransaction()`。

### 4.2 事务传播机制

| 传播行为 | 行为描述 | 场景 |
|----------|----------|------|
| `REQUIRED` | 默认；有事务则加入，没有则新建 | 绝大多数业务方法 |
| `REQUIRES_NEW` | 总是新建事务，挂起当前事务 | 独立日志、独立流水，不受主事务回滚影响 |
| `SUPPORTS` | 有事务则加入，没有则普通执行 | 不强制事务的只读查询 |
| `NOT_SUPPORTED` | 挂起当前事务，以非事务方式执行 | 避免长事务包住不必要逻辑 |
| `MANDATORY` | 必须在已有事务中执行，否则抛异常 | 强制调用方提供事务边界 |
| `NEVER` | 不能在事务中执行，否则抛异常 | 明确禁止事务的操作 |
| `NESTED` | 已有事务时创建 Savepoint | 部分回滚场景 |

### 4.3 事务失效场景（高频考点）

| 类型 | 典型原因 |
|------|----------|
| 代理未生效 | `private/static/final`、`this` 自调用、Bean 未被容器管理 |
| 配置不符合预期 | `NOT_SUPPORTED` / `NEVER`、checked exception 未配置 `rollbackFor` |
| 异常被吞 | 业务代码或其他切面 catch 后未重新抛出 |
| 跨线程 | `@Async` 或事务内新建线程，事务上下文不传播 |
| 注解用错 | 使用了非 Spring 的 `javax.transaction.Transactional` 时语义可能不一致 |

事务上下文与数据库连接通常通过 `ThreadLocal` 绑定到当前线程。跨线程意味着使用新的执行上下文和连接，原事务不会自动传播。

## 五、SpringMVC 调用链

SpringMVC 运行在 Spring 容器之上，Web 请求入口是 `DispatcherServlet`。它把请求分发、Controller 调用和视图渲染串成一条标准流程：

```
Client
  -> DispatcherServlet
  -> HandlerMapping
  -> HandlerAdapter
  -> Controller
  -> ModelAndView
  -> ViewResolver
  -> View
  -> Response
```

完整步骤：

1. 用户请求到达前端控制器 `DispatcherServlet`。
2. `DispatcherServlet` 调用 `HandlerMapping` 查找处理器。
3. `HandlerMapping` 根据 XML 或注解等映射信息找到具体 Handler，并连同拦截器链返回。
4. `DispatcherServlet` 调用 `HandlerAdapter`。
5. `HandlerAdapter` 适配并调用具体 Controller。
6. Controller 执行完成后返回 `ModelAndView`。
7. `HandlerAdapter` 将执行结果返回给 `DispatcherServlet`。
8. `DispatcherServlet` 将 `ModelAndView` 交给 `ViewResolver`。
9. `ViewResolver` 解析出具体 View。
10. `DispatcherServlet` 根据 View 渲染模型数据。
11. 响应返回客户端。

现代前后端分离项目中，很多接口返回 JSON 而不是 `ModelAndView`，但核心分发链路仍然是 `DispatcherServlet -> HandlerMapping -> HandlerAdapter -> Controller`。

## 六、关键权衡

| 权衡点 | 说明 |
|--------|------|
| 单例 Bean 线程安全 | 默认 `singleton` 只代表实例唯一，不代表内部状态线程安全；Service/Repository 通常应保持无状态 |
| 循环依赖是兜底机制 | 三级缓存能处理一部分单例循环依赖，但构造器和 prototype 循环依赖无法解决，应优先重构依赖方向 |
| AOP 隐藏调用路径 | 简化横切逻辑的同时使调试更复杂——需要识别 `$Proxy`、`$$EnhancerByCGLIB` 和代理边界 |
| 声明式事务边界陷阱 | 需要避开自调用、异常吞噬、跨线程和长事务 |
| 事务隔离与数据库实现 | 隔离级别、[[机制-MVCC]]、[[机制-InnoDB锁]] 共同决定可见性、锁等待和并发性能 |

## 七、与其他概念的关系

- 底层依赖 [[机制-动态代理]]：AOP 和 `@Transactional` 都靠代理对象拦截方法调用
- 被 [[机制-SpringBoot]] 扩展：自动装配本质是向 Spring 容器按条件注册 BeanDefinition
- 与 [[机制-SpringMVC]] 协作：Controller、Service、HandlerMapping 等都运行在 Spring 容器中
- 与 [[概念-ThreadLocal]] 相关：事务连接绑定到线程，解释了多线程事务失效
- 与 [[机制-SPI]] 相关：SpringBoot 自动配置文件体现了按约定发现扩展点的思想
- 与 [[机制-MVCC]]、[[机制-InnoDB锁]] 相关：数据库事务的隔离和锁行为由存储引擎真正执行

## 八、应用边界

**适合 Spring 容器管理**：Service、Repository、Controller、配置组件，以及需要事务、日志、权限等横切能力的对象。

**不适合强行交给容器**：频繁创建销毁的领域实体、简单值对象、必须携带运行时参数构造的对象；这些通常用普通构造、工厂方法或原型作用域处理。

**适合 `@Transactional`**：单数据源内需要原子性的增删改组合。

**不适合只靠 `@Transactional`**：分布式事务、跨线程事务、包含外部接口调用的长事务；这些需要拆分事务边界、使用 Outbox/TCC/Seata 等方案或改成最终一致性设计。

## 九、面试追问

**问**：`@Transactional` 的失效场景有哪些？
**答**：①方法非 public（CGLIB/JDK 代理不拦截）；②自调用（`this.method()`，不走代理对象）；③异常被 catch 吞掉，未向上抛出；④`rollbackFor` 不匹配（默认只回滚 `RuntimeException`，checked exception 需显式配置）；⑤多线程（事务绑定 ThreadLocal，新线程拿不到）。  
踩坑案例：`this.sendNotification()` 声明了 `REQUIRES_NEW`，但自调用没走代理，通知失败把外层事务一起回滚，订单状态未更新但钱已扣。修法：`@Lazy` 注入自身，通过 `self.sendNotification()` 调用。

**问**：Spring 三级缓存为什么要三级？二级不够？
**答**：三级缓存各有职责：①一级（`singletonObjects`）：完整 Bean；②二级（`earlySingletonObjects`）：已实例化未注入的半成品 Bean；③三级（`singletonFactories`）：`ObjectFactory` 工厂，决定返回原始对象还是代理对象。  
只有二级不够的根因：AOP 场景下 B 依赖 A，B 拿到的 A 必须是代理后的 A（否则 AOP 失效），而代理对象需要在"第一次被循环依赖获取时"才能确定是否创建。三级的 `ObjectFactory` 支持延迟决策。构造器注入的循环依赖用 `@Lazy` 解决；SpringBoot 2.6+ 默认禁止循环依赖（推荐重构而非绕过）。
