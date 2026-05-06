---
type: concept
status: active
name: "SpringBoot自动装配"
layer: L6
aliases: ["自动配置", "AutoConfiguration", "spring.factories", "@EnableAutoConfiguration", "条件装配", "starter"]
related:
  - "[[机制-IoC容器]]"
  - "[[机制-SPI]]"
sources:
  - "../../raw/note/Hollis/Spring/✅Springboot是如何实现自动配置的？.md"
  - "../../raw/note/Hollis/Spring/✅SpringBoot的启动流程是怎么样的？.md"
  - "../../raw/note/Hollis/Spring/✅SpringBoot和Spring的区别是什么？.md"
  - "../../raw/note/Hollis/Spring/✅如何自定义一个starter？.md"
  - "../../raw/note/Hollis/Spring/✅为什么SpringBoot 3中移除了spring factories.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# SpringBoot自动装配

> SpringBoot 的核心价值：根据 classpath 中已有的 jar 包和类，自动向 IoC 容器注册合适的 Bean，消除 XML 配置的样板代码——"约定优于配置"的工程实践。

## 第一性原理

Spring 本身已经能管理 Bean，但需要开发者显式写大量 XML 或 @Configuration 配置。SpringBoot 解决的问题是：**当某个功能的 jar 包已经在 classpath 上，就应该自动配置好它，无需开发者手工声明**。

实现手段：`@Conditional` 条件化注解 + `SPI` 机制（`spring.factories` / `.imports` 文件）。

## 核心机制

### 启动流程（精简版）

```
SpringApplication.run(App.class, args)
  ↓
new SpringApplication()
  ├── 加载 ApplicationContextInitializer（spring.factories）
  └── 加载 ApplicationListener（spring.factories）
  ↓
SpringApplication.run()
  ├── 创建并准备 Environment（加载 application.properties）
  ├── 创建 ApplicationContext
  ├── 刷新 Context（核心：Bean 注册 + 自动配置）
  └── 调用 CommandLineRunner / ApplicationRunner
```

### 自动配置核心链路

```
@SpringBootApplication
  └── @EnableAutoConfiguration
        └── @Import(AutoConfigurationImportSelector.class)
              └── 读取 META-INF/spring.factories（Boot 2.x）
                  或 META-INF/spring/xxx.imports（Boot 3.x）
                  → 获取所有 AutoConfiguration 类列表
                        └── 条件过滤（@Conditional）
                              → 满足条件的配置类 → 注册 BeanDefinition → IoC 容器
```

### 条件化配置（@Conditional）

| 注解 | 条件 |
|------|------|
| `@ConditionalOnClass` | classpath 上存在指定类 |
| `@ConditionalOnMissingBean` | 容器中不存在某 Bean 时 |
| `@ConditionalOnProperty` | 配置文件中某属性存在/等于某值 |
| `@ConditionalOnWebApplication` | 是 Web 应用 |

```java
@Configuration
@ConditionalOnClass(DataSource.class)        // 有数据源驱动
@ConditionalOnMissingBean(DataSource.class)  // 用户未自定义
public class DataSourceAutoConfiguration { ... }
```

这是"约定优于配置"的关键：有 class 就配，用户自定义了则不覆盖。

### spring.factories vs .imports 文件（Boot 2→3 演变）

| 版本 | 配置文件位置 | 格式 |
|------|------------|------|
| Boot 2.x | `META-INF/spring.factories` | key=value，支持多种扩展点 |
| Boot 2.7+ | 新增 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | 每行一个类名 |
| Boot 3.x | **移除** `spring.factories` 对 AutoConfiguration 的支持，只用 `.imports` | 职责分离，启动性能更好 |

**移除原因**：`spring.factories` 承载了太多功能（Listeners、Initializers、AutoConfiguration 混在一起），启动时全量加载性能差；新的 `.imports` 专门用于自动配置，可懒加载。

这与 L1 [[机制-SPI]] 思想一脉相承，但是 Spring 自己的扩展实现。

### 自定义 Starter

```
my-spring-boot-starter/
├── my-autoconfigure/
│   ├── src/.../MyAutoConfiguration.java     ← @Configuration + @Conditional
│   └── META-INF/spring/
│       └── ...AutoConfiguration.imports     ← 注册 MyAutoConfiguration
└── my-starter/
    └── pom.xml                              ← 只依赖 my-autoconfigure + 相关 jar
```

starter 本身不写代码，只是一个"依赖聚合"；autoconfigure 模块包含真正的自动配置逻辑。

## 关键权衡

1. **自动配置的优先级**：用户显式定义的 Bean 通过 `@ConditionalOnMissingBean` 覆盖自动配置，自定义总是优先
2. **启动速度**：大量 `@Conditional` 判断在启动时执行；Boot 3.x 的 `.imports` 懒加载改善了这个问题
3. **调试困难**：不知道哪些 Bean 被自动配置、为什么某个配置没生效 → 用 `spring.autoconfigure.exclude` 排除或查看 `/actuator/conditions` endpoint

## 与其他概念的关系

- 向 [[机制-IoC容器]] 注册 BeanDefinition：自动装配的结果就是自动填充 IoC 容器
- 思想来自 [[机制-SPI]]：`spring.factories` 是 Spring 版的 SPI 扩展点，`ServiceLoader` 的演变
- L7 框架（SpringCloud、Dubbo）均通过 starter 机制与 Spring 集成

## 应用边界

**理解自动装配的场景**：排查"为什么这个 Bean 没被注入"；编写 starter 供团队复用；配置 `spring.autoconfigure.exclude` 禁用不需要的自动配置。

**不依赖自动装配的场景**：需要精细控制 Bean 初始化顺序时，用 `@DependsOn` + 显式 `@Configuration`。
