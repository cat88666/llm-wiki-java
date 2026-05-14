---
type: concept
status: active
name: "AOP织入"
layer: L6
aliases: ["AOP", "面向切面编程", "切面", "代理", "Aspect", "织入", "切点", "通知"]
related:
  - "[[机制-IoC容器]]"
  - "[[机制-动态代理]]"
  - "[[机制-Spring]]"
sources:
  - "../../../raw/note/Hollis/Spring/✅介绍一下Spring的AOP.md"
  - "../../../raw/note/Hollis/Spring/✅Spring的AOP在什么场景下会失效？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring中用到了哪些设计模式.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# AOP织入

> AOP（面向切面编程）：将散布在多处的横切关注点（日志、权限、事务）抽取到独立的"切面"中，在不修改业务代码的情况下织入执行——OOP 的补充，而非替代。

## 第一性原理

OOP 以类/对象为单位组织代码，但有些逻辑（权限校验、事务管理、日志、性能监控）横跨多个类，若每个类都写一遍则严重违反 DRY。AOP 引入"切面"维度：**把"在哪里（切点）做什么（通知）"从业务逻辑中分离出来**，由框架负责在合适的时机织入。

## 核心机制

### AOP 核心术语

| 术语 | 含义 |
|------|------|
| **Aspect（切面）** | 横切逻辑的模块，包含切点 + 通知（如"事务切面"、"日志切面"）|
| **JoinPoint（连接点）** | 程序运行中可被拦截的点，Spring 只支持**方法执行**级别 |
| **Pointcut（切入点）** | 用表达式筛选哪些 JoinPoint 需要被拦截（where）|
| **Advice（通知）** | 在连接点处执行的逻辑（what），分 5 种类型 |
| **Weaving（织入）** | 将切面应用到目标对象、创建代理的过程 |

**5 种 Advice 类型**：

| 类型 | 时机 |
|------|------|
| `@Before` | 方法执行前 |
| `@AfterReturning` | 方法正常返回后 |
| `@AfterThrowing` | 方法抛异常后 |
| `@After`（Finally）| 无论正常/异常，方法结束后 |
| `@Around` | 完全控制方法调用（最强大，可修改入参/返回值）|

### Spring AOP 的实现方式

Spring AOP 在运行时通过**代理**实现织入（见 [[机制-动态代理]]）：

```
目标类实现了接口？
  ├── 是 → JDK 动态代理（基于接口，生成实现类）
  └── 否 → CGLIB 代理（基于继承，生成子类）

SpringBoot 2.x 后默认强制使用 CGLIB（即使有接口）
```

**织入时机**：`AbstractAutoProxyCreator.postProcessAfterInitialization()` ——  Bean 初始化后置处理阶段，即 [[机制-IoC容器]] Bean 生命周期第 7 步。

### AOP 失效的场景（高频考点）

AOP 本质是代理，凡是不经过代理对象的调用，AOP 都会失效：

| 失效场景 | 原因 |
|----------|------|
| **类内部 this 自调用** | `this` 指向原始对象，不经过代理 |
| **private 方法** | 代理无法重写 private 方法（JDK 代理/CGLIB 均无法拦截）|
| **static 方法** | 属于类本身，代理基于对象，无法拦截 |
| **final 方法** | CGLIB 通过子类重写，final 无法重写 |
| **Bean 未被 Spring 管理** | 没有代理对象，AOP 无从生效 |
| **内部类的方法调用** | 内部类对外部类方法的调用走 this，非代理 |

**解决自调用失效**：注入自身 `@Autowired ApplicationContext ctx; ctx.getBean(this.getClass()).method()`；或 `AopContext.currentProxy()`；或将方法移到另一个 Bean。

## 关键权衡

1. **运行时代理开销**：每次方法调用多一层代理调用，对高频方法（如百万级 QPS 的读操作）有轻微开销，通常可以忽略
2. **调试困难**：调用栈中出现 `$Proxy` 或 `$$EnhancerByCGLIB` 类名，需要理解代理机制才能定位问题
3. **Spring AOP 只支持方法级别**：若需要字段级拦截，需要 AspectJ 编译期/加载期织入（Spring 一般不需要）
4. **@Around 的陷阱**：`proceed()` 必须被调用（否则目标方法不执行）；异常要显式处理或 rethrow

## 与其他概念的关系

- 底层实现依赖 [[机制-动态代理]]：JDK 接口代理 / CGLIB 继承代理
- 在 [[机制-IoC容器]] 中：Bean 初始化后置阶段（BeanPostProcessor.after）自动创建 AOP 代理
- 支撑了 [[机制-Spring]]：`@Transactional` 就是一个 AOP 切面，TransactionInterceptor 是 Around Advice

## 应用边界

**适合 AOP**：日志、权限校验、事务管理、性能监控、接口幂等性检查、方法调用计数。

**不适合 AOP**：需要在构造期间织入的逻辑；需要访问字段级别的拦截；频繁创建的对象（每次创建代理有开销）。
