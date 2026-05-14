---
type: concept
status: active
name: "Spring体系"
layer: L6
sources:
  - "../../../raw/note/Hollis/Spring/"
created: 2026-05-06
updated: 2026-05-06
---

# Spring体系

> L6 Spring/SpringBoot 知识地图，涵盖 IoC、AOP、事务、自动装配四大核心机制。

## 知识地图

```
Spring Framework
├── IoC 容器（[[机制-IoC容器]]）
│   ├── BeanDefinition → Bean 生命周期（11 步）
│   ├── 三级缓存 → 解决单例循环依赖
│   └── Bean 作用域（singleton/prototype/request/session）
│
├── AOP 织入（[[机制-AOP织入]]）
│   ├── 运行时代理（JDK / CGLIB）
│   ├── 5 种 Advice 类型
│   └── 失效场景（this 自调用、private、static、final）
│
├── Spring 事务（[[机制-Spring]]）
│   ├── @Transactional = AOP 切面（TransactionInterceptor）
│   ├── 7 种传播机制（REQUIRED 默认）
│   └── 失效场景（代理失效 + 配置错误 + 异常被吞 + 多线程）
│
└── SpringMVC
    └── DispatcherServlet → HandlerMapping → Controller → ViewResolver

SpringBoot
├── 自动装配（[[机制-SpringBoot]]）
│   ├── @EnableAutoConfiguration → spring.factories / .imports
│   ├── @Conditional 条件过滤
│   └── 自定义 Starter 三步：autoconfigure + imports 文件 + starter pom
│
└── 启动流程
    └── new SpringApplication → 加载 Initializer/Listener → run → 刷新 Context
```

## 高频考点

### IoC / Bean

| 考点 | 关键答案 |
|------|---------|
| Bean 生命周期 | 实例化→属性注入→Aware→BeanPostProcessor.before→afterPropertiesSet→init-method→BeanPostProcessor.after→使用→destroy |
| 初始化方法顺序 | `@PostConstruct` > `afterPropertiesSet` > `init-method` |
| 三级缓存作用 | 一级：完整Bean；二级：半成品早期引用；三级：ObjectFactory（生成代理前的工厂）|
| 为何需要三级 | 有 AOP 代理时，二级缓存暴露原始对象，其他 Bean 持有的是非代理引用导致错误；三级 Factory 在调用时决定返回原始还是代理 |
| 循环依赖限制 | 只支持单例的 setter/field 注入；构造器注入无法解决（实例化时就需要依赖） |
| @Autowired vs @Resource | 前者按类型（Spring）；后者按名称（JSR-250）|

### AOP

| 考点 | 关键答案 |
|------|---------|
| JDK vs CGLIB | 有接口→JDK（基于接口）；无接口→CGLIB（继承子类）；SpringBoot 2.x+ 默认 CGLIB |
| AOP 失效 | this 自调用、private/static/final 方法、Bean 未被 Spring 管理 |
| @Around 注意 | 必须调用 `proceed()` 否则目标方法不执行 |

### Spring 事务

| 考点 | 关键答案 |
|------|---------|
| 实现原理 | AOP 代理 + TransactionInterceptor（Around Advice）+ PlatformTransactionManager |
| 默认回滚 | RuntimeException + Error；checked exception 不回滚（需 `rollbackFor=Exception.class`）|
| 事务失效 | this 调用（无代理）/ 异常被吞 / @Async 跨线程 / 错误的传播机制 |
| REQUIRED vs REQUIRES_NEW | REQUIRED 加入已有事务；REQUIRES_NEW 挂起当前，新开独立事务 |
| 多线程下事务 | 事务绑定 ThreadLocal，新线程无法继承当前事务 |

### SpringBoot 自动装配

| 考点 | 关键答案 |
|------|---------|
| 入口注解 | `@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| 自动配置原理 | `@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → 读 `.imports` 文件 → `@Conditional` 过滤 → 注册 Bean |
| Boot 2 vs Boot 3 | Boot 2 用 `spring.factories`；Boot 3 改用 `.imports` 文件（职责分离，性能更好）|
| 自定义 starter | autoconfigure 模块 + `@Conditional` 注解 + `.imports` 注册 + starter pom 聚合依赖 |

## 与其他层级的联系

- **依赖 L1**：[[机制-动态代理]] 是 AOP 的底层实现；[[机制-SPI]] 是 spring.factories 的思想来源
- **依赖 L3**：Spring 事务在多线程下失效，根因是 [[概念-ThreadLocal]] 绑定 Connection
- **支撑 L7**：SpringCloud 的 Feign/Gateway/Ribbon 都是通过 starter + AOP 集成到 Spring 容器

## 常见误区

1. **三级缓存解决所有循环依赖**：只解决单例 setter 注入；构造器循环依赖仍然报错
2. **@Transactional 就是线程安全的**：事务是单线程单连接的，多线程下事务不传播
3. **Spring 默认使用 JDK 代理**：SpringBoot 2.0 起默认使用 CGLIB（即使有接口）
4. **@Async 和 @Transactional 可以同时用在一个方法上**：@Async 开新线程，新线程无事务上下文，事务不生效
