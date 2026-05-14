---
type: concept
status: active
name: "SPI机制"
layer: L1
aliases: ["SPI", "Service Provider Interface", "ServiceLoader", "ExtensionLoader"]
related:
  - "[[机制-反射]]"
  - "[[机制-动态代理]]"
sources:
  - "../../../raw/note/Hollis/Java基础/✅什么是SPI，和API有啥区别.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# SPI机制

> SPI（Service Provider Interface）是一种服务发现机制——框架定义接口，第三方提供实现，运行时通过约定配置文件自动发现并加载，将"用谁的实现"的决定权从框架侧转移到使用者侧。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | API vs SPI、控制反转、可插拔扩展 |
| [二、核心机制](#二核心机制) | ServiceLoader 加载流程、META-INF/services 约定 |
| [三、Java 核心使用](#三java-核心使用) | 原生 SPI 三步接入、JDBC 驱动自动发现 |
| [四、核心使用原则](#四核心使用原则) | 何时用 SPI、何时用 Spring Bean |
| [五、使用案例](#五使用案例) | JDBC、SLF4J、Dubbo Protocol、SpringBoot AutoConfiguration |
| [六、综合对比](#六综合对比) | Java SPI vs Dubbo SPI vs Spring SPI |
| [七、生产风险](#七生产风险) | 全量加载、线程安全、类加载器问题 |
| [八、与其他概念的关系](#八与其他概念的关系) | 反射、动态代理、SpringBoot 自动装配 |
| [九、应用边界](#九应用边界) | 适用与不适用场景 |

## 一、第一性原理

| | API | SPI |
|--|-----|-----|
| 面向角色 | 应用开发者 | 框架扩展者 |
| 谁定义接口 | 框架 | 框架 |
| 谁实现接口 | 框架 | 第三方/使用者 |
| 谁调用实现 | 使用者 | 框架 |
| 控制方向 | 使用者调用框架 | 框架调用使用者的实现 |

SPI 的存在理由：**让框架在不修改自身源码的前提下支持可插拔扩展**。数据库驱动、日志门面、序列化协议、负载均衡策略——这些都需要在运行时动态替换实现，SPI 提供了标准化的发现与加载机制。

## 二、核心机制

### 2.1 Java 原生 SPI 加载流程

```
META-INF/services/{接口全限定名}
    ↓ ServiceLoader.load(接口.class)
    ↓ 读取配置文件，逐行获取实现类全限定名
    ↓ Class.forName() 加载类
    ↓ newInstance() 实例化（要求无参构造器）
    ↓ 缓存到内部 LinkedHashMap，懒加载（迭代时才实例化）
```

### 2.2 标准接入步骤

**步骤一**：定义接口

```java
public interface IDriver { void connect(); }
```

**步骤二**：提供实现

```java
public class MySQLDriver implements IDriver {
    @Override
    public void connect() { /* ... */ }
}
```

**步骤三**：声明配置文件

```
文件路径: META-INF/services/com.example.IDriver
文件内容: com.mysql.jdbc.MySQLDriver
```

**步骤四**：框架侧加载

```java
ServiceLoader<IDriver> loader = ServiceLoader.load(IDriver.class);
for (IDriver driver : loader) {
    driver.connect();
}
```

### 2.3 类加载器选择

`ServiceLoader.load(Class)` 默认使用线程上下文类加载器（`Thread.currentThread().getContextClassLoader()`）。这是为了解决 SPI 场景下的双亲委派局限：接口在 Bootstrap ClassLoader（如 `java.sql.Driver`），但实现在应用 ClassLoader（如 MySQL 驱动 jar），父加载器无法加载子路径的类。

## 三、Java 核心使用

### 3.1 JDBC 驱动自动发现

JDBC 4.0+ 不再需要 `Class.forName("com.mysql.jdbc.Driver")`。`DriverManager` 在静态块中通过 `ServiceLoader<Driver>` 自动扫描 classpath 下所有 `META-INF/services/java.sql.Driver`，实现驱动自动注册。

### 3.2 SLF4J 日志绑定

SLF4J 通过 SPI 思想发现具体日志实现（Logback、Log4j2）。`StaticLoggerBinder` 类在各实现 jar 中提供，SLF4J 在初始化时自动定位。

### 3.3 JDK 9+ 模块化下的 SPI

JDK 9 引入模块系统后，SPI 声明方式扩展为 `module-info.java` 中的 `provides ... with ...` 和 `uses` 语句，但底层机制不变。

## 四、核心使用原则

| 场景 | 推荐方案 |
|------|---------|
| 框架级扩展点（跨 jar、跨团队） | SPI |
| 业务层策略选择（同项目内） | Spring Bean + 策略模式 |
| 需要按名字选择实现 | Dubbo SPI / Spring ApplicationContext |
| 需要条件激活 | SpringBoot @Conditional |
| 需要 AOP 增强扩展点 | Dubbo SPI @Activate |

## 五、使用案例

| 案例 | SPI 变体 | 配置位置 | 说明 |
|------|---------|---------|------|
| JDBC 驱动 | Java 原生 | `META-INF/services/java.sql.Driver` | DriverManager 自动发现 |
| SLF4J 日志 | 类似 SPI | `StaticLoggerBinder` 类 | 编译时绑定 |
| Dubbo 协议/负载均衡 | Dubbo SPI | `META-INF/dubbo/接口名` | 支持按名获取、AOP 包装 |
| SpringBoot 自动装配 | Spring SPI | `META-INF/spring.factories`（Boot 2.x）/ `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Boot 3.x） | AutoConfigurationImportSelector 读取 |
| Java 安全 Provider | Java 原生 | `java.security` 配置 | JCE 加密算法扩展 |

## 六、综合对比

| 维度 | Java 原生 SPI | Dubbo SPI | Spring SPI（spring.factories） |
|------|--------------|-----------|-------------------------------|
| 配置位置 | `META-INF/services/` | `META-INF/dubbo/` | `META-INF/spring.factories` |
| 按名获取 | 不支持（全量遍历） | 支持（`key=value` 格式） | 不支持（按接口全量加载） |
| 按需加载 | 不支持（全量实例化） | 支持（按名字精确加载） | 不支持（全量加载后由 @Conditional 过滤） |
| 默认实现 | 无 | `@SPI("default")` 指定 | 无 |
| 自适应选择 | 无 | `@Adaptive` 运行时动态选择 | 无 |
| AOP 包装 | 无 | `@Activate` 自动激活包装类 | 无 |
| 线程安全 | 非线程安全 | 线程安全（缓存 + 同步） | 线程安全 |
| IoC/DI | 无 | 支持 setter 注入 | 完整 Spring IoC |

**选型结论**：Java 原生 SPI 适合简单场景（JDBC、日志）；Dubbo SPI 是微服务框架扩展的标杆设计；Spring SPI 适合 Boot 生态的自动装配。

## 七、生产风险

| 风险 | 原因 | 防御 |
|------|------|------|
| 全量加载浪费 | Java 原生 SPI 迭代时加载所有实现，无法按需选择 | 使用 Dubbo SPI 按名获取 |
| ServiceLoader 非线程安全 | 多线程并发迭代同一 ServiceLoader 实例会抛 `ConcurrentModificationException` | 每次调用 `ServiceLoader.load()` 创建新实例，或加锁 |
| 类加载器错配 | SPI 接口与实现不在同一 ClassLoader 路径下 | 显式传入正确的 ClassLoader：`ServiceLoader.load(SPI.class, classLoader)` |
| 实现类初始化失败 | 某个实现类构造器抛异常导致整个 ServiceLoader 迭代中断 | 遍历时 try-catch，跳过异常实现 |
| spring.factories 膨胀 | 大量 starter 导致启动扫描变慢 | SpringBoot 3.x 改用 imports 文件，按需加载 |

## 八、与其他概念的关系

- 依赖 [[机制-反射]]：ServiceLoader 通过 `Class.forName()` + `newInstance()` 实例化实现类
- 与 [[机制-动态代理]] 协作：Dubbo `@Adaptive` 通过动态代理生成自适应扩展类，运行时根据 URL 参数选择实现
- 支撑了 L6 SpringBoot 自动装配：`AutoConfigurationImportSelector` 读取 `spring.factories` 是 SPI 思想的直接应用
- 支撑了 L7 Dubbo 扩展体系：Protocol、Cluster、LoadBalance、Serialization 等核心抽象均基于 Dubbo SPI 实现可插拔

## 九、应用边界

| 适用 | 不适用 |
|------|--------|
| 框架级可插拔扩展点（驱动、协议、算法） | 业务层简单策略选择（用 Spring Bean + Map 更直观） |
| 跨 jar / 跨团队的接口约定 | 需要按名字动态获取（原生 SPI 不支持，用 Dubbo SPI） |
| 标准化的服务发现（JDBC、日志、安全） | 需要依赖注入的复杂扩展（用 Spring IoC） |
| 中间件 SDK 的扩展点设计 | 需要条件激活 / 环境感知（用 @Conditional） |
