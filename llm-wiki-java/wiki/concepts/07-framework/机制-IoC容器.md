---
type: concept
status: active
name: "IoC容器"
layer: L6
aliases: ["IoC", "DI", "依赖注入", "控制反转", "BeanFactory", "ApplicationContext", "三级缓存", "循环依赖"]
related:
  - "[[机制-AOP织入]]"
  - "[[机制-动态代理]]"
  - "[[机制-SpringBoot自动装配]]"
sources:
  - "../../../raw/note/Hollis/Spring/✅介绍一下Spring的IOC.md"
  - "../../../raw/note/Hollis/Spring/✅Spring Bean的生命周期是怎么样的？.md"
  - "../../../raw/note/Hollis/Spring/✅什么是Spring的循环依赖问题？.md"
  - "../../../raw/note/Hollis/Spring/✅什么是Spring的三级缓存.md"
  - "../../../raw/note/Hollis/Spring/✅三级缓存是如何解决循环依赖的问题的？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring 中的 Bean 作用域有哪些？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring默认支持循环依赖吗？如果发生如何解决？.md"
  - "../../../raw/note/Hollis/Spring/✅Spring解决循环依赖一定需要三级缓存吗？.md"
  - "../../../raw/note/Hollis/Spring/✅@Lazy注解能解决循环依赖吗？.md"
created: 2026-05-06
updated: 2026-05-08
lint_notes: ""
---

# IoC容器

> 控制反转（IoC）：把对象的创建和依赖管理的控制权从业务代码转移给框架容器，调用方只需声明依赖，由容器注入——这是 Spring 的核心设计理念。

## 第一性原理

没有 IoC 时，一个对象要使用另一个对象，必须自己 `new`。这带来三个问题：调用方要了解依赖的构造细节；同一个对象可能被 `new` 多次（浪费）；被依赖对象变化时，所有调用方都要改。

IoC 的核心反转是：**对象不再主动获取依赖，而是被动接受容器注入**。容器统一管理对象的创建、生命周期和依赖关系，调用方只需声明"我需要什么"（通过注解/XML）。

## 核心机制

### Bean 的注册与获取

```
配置元数据（XML / 注解 / @Configuration）
  ↓
BeanDefinition（描述如何创建 Bean）注入 BeanFactory
  ↓
ApplicationContext（BeanFactory 的高级封装）统一管理
  ↓
调用方通过 @Autowired / getBean() 获取
```

**Bean 作用域**：
- `singleton`（默认）：容器内唯一实例，容器启动时创建
- `prototype`：每次获取创建新实例，容器不管理销毁
- `request` / `session` / `application`：Web 场景下与请求/会话绑定

### Bean 生命周期（核心11步）

```
1. 实例化（createBeanInstance）           ← new 出来，只分配内存
2. 属性注入（populateBean）               ← 填充 @Autowired 依赖
3. Aware 接口回调                         ← BeanNameAware / ApplicationContextAware
4. BeanPostProcessor.before              ← 前置处理
5. InitializingBean.afterPropertiesSet   ← 属性设置完成后钩子
6. 自定义 init-method                    ← XML/注解中指定的初始化方法
7. BeanPostProcessor.after              ← 后置处理（AOP 代理在此创建）
8. 注册 Destruction 回调                 ← registerDisposableBean
9. Bean 正常使用
10. DisposableBean.destroy               ← 容器关闭时
11. 自定义 destroy-method
```

**关键**：AOP 代理在第 7 步（`postProcessAfterInitialization`）中由 `AbstractAutoProxyCreator` 创建。

**初始化方法优先级**：`@PostConstruct` > `InitializingBean.afterPropertiesSet` > `init-method`

### 三级缓存与循环依赖

Spring 在 `DefaultSingletonBeanRegistry` 中维护三级缓存：

| 缓存 | Map 名称 | 内容 |
|------|---------|------|
| 一级 | `singletonObjects` | 完整 Bean（实例化 + 初始化完成）|
| 二级 | `earlySingletonObjects` | 半成品 Bean（已实例化，未完成属性注入）|
| 三级 | `singletonFactories` | Bean 的 ObjectFactory（可生成代理对象）|

**循环依赖解决流程**（A 依赖 B，B 依赖 A）：

```
1. 创建 A：实例化 → 将 A 的工厂放入三级缓存
2. 属性注入 A → 发现需要 B → 去创建 B
3. 创建 B：实例化 → 将 B 的工厂放入三级缓存
4. 属性注入 B → 发现需要 A → 从三级缓存获取 A 的工厂 → 生成 A 的早期引用 → 存入二级缓存
5. B 完成初始化 → 存入一级缓存
6. A 获得 B 的引用 → A 完成初始化 → 清理二级/三级缓存 → 存入一级缓存
```

**为什么需要三级缓存而非二级**：如果 A 需要被 AOP 代理，二级缓存中暴露的是原始对象，第三方 B 持有的是原始引用而非代理引用，导致代理失效。三级缓存的 Factory 在被调用时才决定生成代理还是原始对象，保证一致性。

**循环依赖的限制**：
- 只支持**单例** Bean（prototype 无法缓存半成品）
- **构造器注入**无法解决（实例化时就需要依赖，根本无法先放入三级缓存）
- 解法：改为 setter 注入 / `@Lazy` 注解 / 重新设计消除循环

**@Lazy 解决构造器注入循环依赖**：`@Lazy` 使 Bean 在首次被使用时才初始化，打破构造器注入的实例化时依赖要求。但 `@Lazy` 只是推迟问题，循环依赖本身是设计缺陷，应从根源消除。

**SpringBoot 2.6 默认关闭循环依赖支持**：2.6 之前默认允许循环依赖；2.6 开始启动时会直接报错（`spring.main.allow-circular-references=true` 可手动开启）。Spring 的设计哲学：三级缓存是"兜底"机制，不鼓励循环依赖存在。

**为什么需要三级缓存而非二级**：二级缓存能解决普通对象的循环依赖，但若 Bean 需要 AOP 代理，二级缓存暴露的是原始对象，导致其他 Bean 持有的是原始引用而非代理引用。三级缓存存 `ObjectFactory`，调用时才决定返回代理对象还是原始对象，保证同一 Bean 引用的一致性。

## 关键权衡

1. **singleton 线程安全**：Spring Bean 默认单例，若 Bean 有可变成员变量（状态），在多线程下不安全；无状态 Bean（Service/Repository）本身是线程安全的
2. **三级缓存的代价**：增加了 BeanFactory 的复杂度；循环依赖本质上是设计问题，Spring 的解法是"兜底"而非"鼓励"
3. **@Autowired vs @Resource**：`@Autowired` 按类型注入（Spring 原生）；`@Resource` 按名称注入（JDK 标准，JSR-250）；多个同类型 Bean 时用 `@Qualifier` 或 `@Resource(name=...)` 消歧

## 与其他概念的关系

- 支撑了 [[机制-AOP织入]]：AOP 代理在 Bean 生命周期第 7 步（BeanPostProcessor.after）中创建，依赖 IoC 容器管理 Bean
- 底层用 [[机制-动态代理]]：AOP 代理和 @Transactional 代理均由 IoC 容器在 Bean 初始化后注入
- 被 [[机制-SpringBoot自动装配]] 扩展：自动装配本质是自动向 IoC 容器注册 BeanDefinition

## 应用边界

**适合 IoC 管理**：无状态的 Service / Repository / Controller；需要横切关注点（事务、日志）的 Bean。

**不适合 IoC 管理**：频繁创建销毁的领域对象（Order、User 实体）；需要传参构造的值对象——用工厂方法或 `prototype` 作用域。
