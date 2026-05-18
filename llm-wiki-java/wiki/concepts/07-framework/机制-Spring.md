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
| [五、AOP 与声明式事务](#sec-6) | 代理、Advice、事务传播、失效场景 |
| [六、设计权衡与使用原则](#sec-7) | 容器管理边界、事务边界、线程边界 |
| [七、生产风险与排查](#sec-8) | Bean 冲突、代理失效、循环依赖、事务失效 |
| [八、关系与边界](#sec-9) | 与 Boot、MVC、MyBatis、ThreadLocal 的关系 |
| [九、面试速答口径](#sec-10) | 高频问题答案 |

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
## 五、AOP 与声明式事务

### 5.1 AOP 基础链路

```text
目标 Bean 初始化后
  -> AutoProxyCreator 判断是否需要增强
  -> JDK 动态代理或 CGLIB 创建代理
  -> 调用代理方法
  -> Advice 链执行
  -> 目标方法执行
```

### 5.2 `@Transactional` 执行链路

```text
代理方法调用
  -> TransactionInterceptor
  -> PlatformTransactionManager 开启事务
  -> 业务方法执行
  -> 正常提交 / 异常回滚
```

事务上下文绑定在当前线程，底层依赖 [[概念-ThreadLocal]]。新线程、`@Async`、线程池中的代码不会自动共享原事务。

### 5.3 事务传播核心口径

| 传播行为 | 结论 |
| --- | --- |
| `REQUIRED` | 默认，有事务加入，无事务新建 |
| `REQUIRES_NEW` | 挂起当前事务，新建事务 |
| `NESTED` | 嵌套事务，依赖保存点 |
| `SUPPORTS` | 有事务加入，无事务非事务执行 |

<a id="sec-7"></a>
## 六、设计权衡与使用原则

1. 能被切面增强的逻辑必须放在 Spring Bean 的 public 方法调用链上。
2. 事务边界放在 Service 层，避免 Controller 或 DAO 层事务语义混乱。
3. 不把所有对象都交给容器；值对象、DTO、临时策略对象可普通创建。
4. 循环依赖不是能力点，是设计异味。

<a id="sec-8"></a>
## 七、生产风险与排查

| 风险 | 现象 | 排查点 |
| --- | --- | --- |
| Bean 冲突 | 启动失败或注入错误 | beanName、`@Primary`、`@Qualifier` |
| AOP 失效 | 事务/缓存不生效 | 是否自调用、是否 public、是否 Spring Bean |
| 事务不回滚 | 异常被吞或类型不匹配 | rollbackFor、异常链、代理入口 |
| 多线程事务失效 | 子线程写库不回滚 | ThreadLocal 线程隔离 |
| 循环依赖 | 启动报依赖环 | 构造器注入、服务职责拆分 |

<a id="sec-9"></a>
## 八、关系与边界

- [[机制-SpringBoot]] 是 Spring 的自动配置与运行时封装。
- [[机制-SpringMVC]] 依赖 Spring 容器管理 Controller 与 Web 组件。
- [[机制-动态代理]] 是 AOP 和事务的底层能力。
- [[概念-MySQL]] 事务能力由 Spring 事务管理器统一协调。

边界：Spring 管理对象和横切逻辑，不保证业务建模正确，也不替代分布式事务和跨线程上下文治理。

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| IoC 是什么 | 对象创建和依赖装配控制权交给容器 |
| BeanFactory 和 ApplicationContext 区别 | 前者是底座，后者是应用级上下文，集成事件/资源/环境 |
| 三级缓存为什么需要第三级 | 延迟生成早期代理引用，兼容 AOP 和循环依赖 |
| Spring 事务为什么会失效 | 调用没经过代理、方法不可代理、异常被吞、多线程切换 |
| `@Transactional` 多线程生效吗 | 不自动生效，事务上下文在 ThreadLocal 中按线程隔离 |
