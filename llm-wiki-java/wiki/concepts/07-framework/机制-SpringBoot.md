---
type: concept
status: active
name: "SpringBoot"
layer: L6
aliases: ["自动配置", "AutoConfiguration", "AutoConfigurationImportSelector", "DeferredImportSelector", "spring.factories", "@EnableAutoConfiguration", "条件装配", "starter", "MANIFEST.MF", "Start-Class", "配置文件加载顺序", "jar启动", "@SpringBootApplication", "@ComponentScan"]
related:
  - "[[机制-Spring]]"
  - "[[机制-SPI]]"
---

# SpringBoot

> SpringBoot 的核心价值：根据 classpath 中已有的 jar 包和类，自动向 IoC 容器注册合适的 Bean，消除 XML 配置的样板代码——"约定优于配置"的工程实践。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 约定优于配置、消除样板配置 |
| [二、核心机制](#二核心机制) | 启动流程、自动配置链路、@Conditional、spring.factories 演变、配置文件加载顺序、jar 启动原理 |
| [三、Java 核心使用](#三java-核心使用) | 自定义 Starter、优雅停机、@Bean/@SpringBootApplication 底层 |
| [四、关键权衡](#四关键权衡) | 自动配置优先级、启动速度、调试困难 |
| [五、与其他概念的关系](#五与其他概念的关系) | Spring IoC、SPI、SpringCloud |
| [六、应用边界](#六应用边界) | 适用与不适用场景、面试速答 |

## 一、第一性原理

Spring 本身已经能管理 Bean，但需要开发者显式写大量 XML 或 @Configuration 配置。SpringBoot 解决的问题是：**当某个功能的 jar 包已经在 classpath 上，就应该自动配置好它，无需开发者手工声明**。

实现手段：`@Conditional` 条件化注解 + `SPI` 机制（`spring.factories` / `.imports` 文件）。

## 二、核心机制

### 2.1 启动流程

**阶段一：new SpringApplication()**

```
1. 添加源（主配置类）
2. 推断 Web 环境类型（SERVLET / REACTIVE / NONE）
3. 从 spring.factories 加载 ApplicationContextInitializer 列表
4. 从 spring.factories 加载 ApplicationListener 列表
5. 确定主应用类（通过调用栈推断 main() 所在类）
```

**阶段二：SpringApplication.run()**

```
启动计时器（StopWatch）
  ↓
获取 SpringApplicationRunListeners（从 spring.factories 加载）
  → listeners.starting()
  ↓
prepareEnvironment（装配环境参数）
  → 加载 application.properties / application.yml / 环境变量 / 系统属性
  → listeners.environmentPrepared()
  ↓
printBanner（打印启动横幅）
  ↓
createApplicationContext（创建 Spring 上下文）
  → Web 场景用 AnnotationConfigServletWebServerApplicationContext
  ↓
prepareContext（准备上下文）
  → 执行 ApplicationContextInitializer.initialize()
  → 加载主配置类的 BeanDefinition
  → listeners.contextPrepared() / contextLoaded()
  ↓
refreshContext（刷新上下文，核心步骤）
  → 创建 BeanFactory + 注册所有 BeanDefinition（含自动配置）
  → 实例化所有单例 Bean（含依赖注入、AOP 代理）
  → onRefresh() → createWebServer() → 启动内嵌 Tomcat ★
  ↓
afterRefresh（空扩展点）
  ↓
停止计时器，打印启动耗时
  → listeners.started()
  ↓
调用 CommandLineRunner / ApplicationRunner
  → listeners.running()
```

**内嵌 Tomcat 启动位置**：`refreshContext → onRefresh → ServletWebServerApplicationContext.createWebServer() → TomcatServletWebServerFactory.getWebServer()` → 创建并启动 Tomcat 实例。

### 2.2 自动配置核心链路

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

关键点在 `@Import(AutoConfigurationImportSelector.class)`：`AutoConfigurationImportSelector` 实现了 `DeferredImportSelector`，Spring 解析 `@Import` 时会延迟调用它的 `getAutoConfigurationEntry()`，读取候选自动配置类，再经过去重、排除、条件过滤，最后把满足条件的自动配置类导入容器。

### 2.3 条件化配置（@Conditional）

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

### 2.4 spring.factories vs .imports 文件（Boot 2→3 演变）

| 版本 | 配置文件位置 | 格式 |
|------|------------|------|
| Boot 2.x | `META-INF/spring.factories` | key=value，支持多种扩展点 |
| Boot 2.7+ | 新增 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | 每行一个类名 |
| Boot 3.x | **移除** `spring.factories` 对 AutoConfiguration 的支持，只用 `.imports` | 职责分离，启动性能更好 |

**移除原因**：`spring.factories` 承载了太多功能（Listeners、Initializers、AutoConfiguration 混在一起），启动时全量加载性能差；新的 `.imports` 专门用于自动配置，可懒加载。

这与 L1 [[机制-SPI]] 思想一脉相承，但是 Spring 自己的扩展实现。

### 2.5 配置文件加载顺序（高优先级覆盖低优先级）

高优先级覆盖低优先级，多个来源共同生效时形成互补配置：

| 优先级 | 来源 |
|--------|------|
| 1（最高）| 命令行参数（`--server.port=8080`）|
| 2 | Java 系统属性（`System.getProperties()`）|
| 3 | 操作系统环境变量 |
| 4 | jar **包外** `application-{profile}.yml / .properties` |
| 5 | jar **包内** `application-{profile}.yml / .properties` |
| 6 | jar **包外** `application.yml / .properties` |
| 7 | jar **包内** `application.yml / .properties` |
| 8（最低）| `@Configuration` 类上的 `@PropertySource` |

**记忆规则**：外 > 内；profile 精准 > 通用；命令行 > 系统属性 > 环境变量 > 配置文件。

### 2.6 jar 启动原理（`java -jar`）

```bash
java -jar xxx.jar
```

1. `java -jar` 读取 jar 包中的 `META-INF/MANIFEST.MF` 文件
2. `MANIFEST.MF` 中指定：
   ```
   Main-Class: org.springframework.boot.loader.JarLauncher   ← Spring Boot Loader
   Start-Class: com.example.MyApplication                     ← 真正的启动类
   ```
3. JVM 执行 `JarLauncher.main()` → 加载 BOOT-INF/classes 和 BOOT-INF/lib 下的 jar → 反射调用 `Start-Class.main()` 启动应用

**为什么需要 JarLauncher**：标准 `java -jar` 不支持 jar 中嵌套 jar（nested jars）。Spring Boot 自己实现了 `JarLauncher` 类加载器来展开 BOOT-INF/lib 中的依赖 jar，实现了"fat jar"格式。

## 三、Java 核心使用

### 3.0 核心注解底层

**`@SpringBootApplication` 组合拆解**：

| 元注解 | 等价关系 | 作用 |
|--------|---------|------|
| `@SpringBootConfiguration` | = `@Configuration` | 启动类本身是配置类 |
| `@EnableAutoConfiguration` | `@Import(AutoConfigurationImportSelector)` | 加载自动配置类 |
| `@ComponentScan` | 默认扫描启动类所在包及其子包 | 注册 @Component 系列 Bean |

**`@Bean` 底层机制**：
- 启动时 Spring 解析 `@Bean` 方法，**方法名即 beanName**（可用 `name` 属性覆盖）
- 调用方法返回值注册为 Bean 实例
- 在 `@Configuration` 类中，Spring 通过 CGLIB 代理使 `@Bean` 方法具有单例语义（多次调用同一 `@Bean` 方法返回同一实例）

### 3.1 自定义 Starter

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

### 3.2 优雅停机

```yaml
# application.yml
server:
  shutdown: graceful                            # 开启优雅停机
spring:
  lifecycle:
    timeout-per-shutdown-phase: 2m             # 等待最长时间
```

收到 SIGTERM 信号后，SpringBoot 停止接收新请求，等待已有请求处理完毕（最长 2 分钟），再关闭容器、销毁 Bean（`DisposableBean.destroy()` / `@PreDestroy`）。

## 四、关键权衡

| 权衡点 | 说明 |
|--------|------|
| 自动配置优先级 | 用户显式定义的 Bean 通过 `@ConditionalOnMissingBean` 覆盖自动配置，自定义总是优先 |
| 启动速度 | 大量 `@Conditional` 判断在启动时执行；Boot 3.x 的 `.imports` 懒加载改善了这个问题 |
| 调试困难 | 不知道哪些 Bean 被自动配置、为什么某个配置没生效 → 用 `spring.autoconfigure.exclude` 排除或查看 `/actuator/conditions` endpoint |

## 五、与其他概念的关系

- 向 [[机制-Spring]] 注册 BeanDefinition：自动装配的结果就是自动填充 IoC 容器
- 思想来自 [[机制-SPI]]：`spring.factories` 是 Spring 版的 SPI 扩展点，`ServiceLoader` 的演变
- L7 框架（SpringCloud、Dubbo）均通过 starter 机制与 Spring 集成

## 六、应用边界

**理解自动装配的场景**：排查"为什么这个 Bean 没被注入"；编写 starter 供团队复用；配置 `spring.autoconfigure.exclude` 禁用不需要的自动配置。

**不依赖自动装配的场景**：需要精细控制 Bean 初始化顺序时，用 `@DependsOn` + 显式 `@Configuration`。

### 面试速答

| 问题 | 速答 |
|------|------|
| `java -jar` 如何找到启动类 | JVM 读取 `META-INF/MANIFEST.MF`；SpringBoot fat jar 的 `Main-Class` 通常是 `JarLauncher`，`Start-Class` 才是真正业务启动类，如 `com.xxx.Application` |
| SpringBoot 启动原理 | `SpringApplication.run()` 准备环境、创建 ApplicationContext、加载主配置类和自动配置、刷新容器、启动内嵌 WebServer，最后执行 Runner |
| `@SpringBootApplication` 做了什么 | 组合了 `@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan` |
| 自动装配核心 | `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入自动配置类，Boot 2.x 从 `spring.factories` 读取，Boot 3.x 从 `.imports` 读取 |
| 为什么用户配置优先 | 自动配置通常使用 `@ConditionalOnMissingBean`，用户自己声明 Bean 后自动配置不会覆盖 |
