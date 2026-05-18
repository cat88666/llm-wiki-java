---
type: concept
status: active
name: "SpringBoot"
layer: L6
aliases: ["自动配置", "AutoConfiguration", "AutoConfigurationImportSelector", "DeferredImportSelector", "spring.factories", "@EnableAutoConfiguration", "条件装配", "starter", "MANIFEST.MF", "Start-Class", "配置文件加载顺序", "jar启动", "@SpringBootApplication", "@ComponentScan"]
related:
  - "[[机制-Spring]]"
  - "[[机制-SPI]]"
  - "[[机制-SpringMVC]]"
  - "[[机制-SpringCloud]]"
sources:
  - "../../../raw/note/Hollis/Spring/✅SpringBoot和Spring的区别是什么？.md"
  - "../../../raw/note/Hollis/Spring/✅SpringBoot的启动流程是怎么样的？.md"
  - "../../../raw/note/Hollis/Spring/✅为什么SpringBoot 3中移除了spring factories.md"
  - "../../../raw/note/Hollis/Spring/✅SpringBoot是如何实现main方法启动Web项目的？.md"
updated: 2026-05-18
---

# SpringBoot

> SpringBoot 是 Spring 的工程化封装：通过自动配置、Starter、内嵌容器和可执行 fat jar，把“能跑起来的 Spring 应用”从手工配置变成约定驱动。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、SpringBoot 的本质](#sec-2) | 约定优于配置，降低 Spring 应用装配成本 |
| [二、启动流程](#sec-3) | `SpringApplication` 初始化与 `run` 主链路 |
| [三、自动配置机制](#sec-4) | `@EnableAutoConfiguration`、条件注解、`.imports` |
| [四、配置加载与 Starter](#sec-5) | 配置优先级、自定义 Starter |
| [五、内嵌容器与 fat jar](#sec-6) | Tomcat 启动位置、`JarLauncher` |
| [六、版本演进与选型](#sec-7) | Boot 2.x 到 3.x 的自动配置变化 |
| [七、生产风险与排查](#sec-8) | 自动配置误命中、启动慢、配置覆盖 |
| [八、关系与边界](#sec-9) | 与 Spring、SpringMVC、SpringCloud、SPI 的关系 |
| [九、面试速答口径](#sec-10) | 高频问题答案 |

<a id="sec-2"></a>
## 一、SpringBoot 的本质

SpringBoot 没有替代 Spring，它解决的是 Spring 应用落地成本：

| 能力 | Spring | SpringBoot |
| --- | --- | --- |
| Bean 管理 | IoC/AOP 底座 | 复用 Spring |
| 配置方式 | 手工配置多 | 自动配置优先 |
| Web 容器 | 外部部署为主 | 内嵌容器 |
| 依赖管理 | 开发者组合 | Starter 聚合 |

核心思想：classpath 上出现某类能力时，自动注册合理默认 Bean；用户显式配置时，用户优先。

<a id="sec-3"></a>
## 二、启动流程

```text
new SpringApplication(primarySources)
  -> 推断应用类型（SERVLET / REACTIVE / NONE）
  -> 加载 Initializer / Listener
  -> 推断 main class

run(args)
  -> 准备 Environment
  -> 创建 ApplicationContext
  -> 加载主配置类
  -> refreshContext
  -> 创建非懒加载单例 Bean
  -> 启动内嵌 WebServer
  -> 执行 Runner
```

`refreshContext` 是核心分水岭：Spring 容器刷新、Bean 创建、自动配置落地、内嵌 Tomcat 启动都在这里串起来。

<a id="sec-4"></a>
## 三、自动配置机制

### 3.1 注解链路

```text
@SpringBootApplication
  -> @SpringBootConfiguration
  -> @ComponentScan
  -> @EnableAutoConfiguration
     -> @Import(AutoConfigurationImportSelector.class)
        -> 读取自动配置候选
        -> @Conditional 过滤
        -> 注册 BeanDefinition
```

### 3.2 条件装配

| 注解 | 含义 |
| --- | --- |
| `@ConditionalOnClass` | classpath 存在指定类 |
| `@ConditionalOnMissingBean` | 用户未提供同类 Bean |
| `@ConditionalOnProperty` | 配置开关满足条件 |
| `@ConditionalOnWebApplication` | 当前是 Web 应用 |

自动配置可覆盖性来自 `@ConditionalOnMissingBean`：框架提供默认值，用户定义即优先。

<a id="sec-5"></a>
## 四、配置加载与 Starter

### 4.1 配置优先级

```text
命令行参数
  -> Java 系统属性
  -> OS 环境变量
  -> jar 外 profile 配置
  -> jar 内 profile 配置
  -> jar 外通用配置
  -> jar 内通用配置
```

记忆：外部覆盖内部，profile 覆盖通用，命令行最高。

### 4.2 自定义 Starter

```text
xxx-spring-boot-starter       -> 依赖聚合
xxx-spring-boot-autoconfigure -> 自动配置代码
  -> AutoConfiguration 类
  -> META-INF/spring/...AutoConfiguration.imports
```

Starter 本身通常不写业务逻辑，真正逻辑在 autoconfigure 模块。

<a id="sec-6"></a>
## 五、内嵌容器与 fat jar

### 5.1 内嵌 Tomcat 启动位置

Servlet Web 应用在 `refreshContext -> onRefresh -> createWebServer` 中创建并启动内嵌 Tomcat。

### 5.2 `java -jar` 启动原理

```text
java -jar app.jar
  -> 读取 MANIFEST.MF
  -> Main-Class: JarLauncher
  -> Start-Class: 业务启动类
  -> 加载 BOOT-INF/classes 与 BOOT-INF/lib
  -> 反射调用 main
```

标准 jar 不支持嵌套 jar，SpringBoot Loader 通过自定义加载器支持 fat jar。

<a id="sec-7"></a>
## 六、版本演进与选型

| 版本 | 自动配置文件 | 变化 |
| --- | --- | --- |
| Boot 2.x | `META-INF/spring.factories` | 多类扩展点混在一个文件 |
| Boot 2.7+ | 新增 `.imports` | 自动配置开始拆分 |
| Boot 3.x | 自动配置只用 `.imports` | 更利于 AOT 与云原生 |

Boot 3 同时要求 Java 17+，并迁移到 Jakarta EE 包名，升级前必须检查依赖兼容。

<a id="sec-8"></a>
## 七、生产风险与排查

| 风险 | 现象 | 排查方式 |
| --- | --- | --- |
| 自动配置未生效 | Bean 缺失 | 看 `/actuator/conditions` 或 condition report |
| 自动配置误命中 | 多出不需要的 Bean | `spring.autoconfigure.exclude` 排除 |
| 配置覆盖错误 | 本地/线上行为不一致 | 打印 Environment 来源与 profile |
| 启动慢 | 条件扫描、Bean 初始化重 | startup actuator / 日志分段 |
| Boot 3 升级失败 | ClassNotFound / Jakarta 包冲突 | 依赖矩阵与迁移清单 |

<a id="sec-9"></a>
## 八、关系与边界

- 建立在 [[机制-Spring]] 上：自动配置的结果仍是注册 BeanDefinition。
- 装配 [[机制-SpringMVC]]：自动注册 `DispatcherServlet` 和 MVC 基础组件。
- 支撑 [[机制-SpringCloud]]：微服务组件通过 Starter 接入 Boot 应用。
- 借鉴 [[机制-SPI]]：通过约定文件发现自动配置。

边界：SpringBoot 解决默认装配和部署形态，不替代架构设计、容量治理和业务模块划分。

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| SpringBoot 和 Spring 区别 | Boot 是 Spring 工程化封装，核心是自动配置、Starter、内嵌容器 |
| 自动配置核心 | `AutoConfigurationImportSelector` 导入候选配置，`@Conditional` 过滤 |
| 为什么用户 Bean 优先 | 自动配置常用 `@ConditionalOnMissingBean` |
| Boot 3 为什么不用 spring.factories 做自动配置 | 职责拆分，提升启动与 AOT 友好性 |
| `java -jar` 如何启动 | `JarLauncher` 加载嵌套 jar，再调用 `Start-Class.main` |
