---
type: concept
status: active
name: "SPI机制"
layer: L1
aliases: ["SPI", "Service Provider Interface", "ServiceLoader"]
related:
  - "[[机制-反射]]"
  - "[[机制-动态代理]]"
sources:
  - "../../raw/note/Hollis/Java基础/✅什么是SPI，和API有啥区别 30f3673e113881cf8607cc7f1d251e73.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# SPI机制

> 框架定义接口，第三方提供实现，运行时自动发现——把"用谁的实现"的决定权从框架转移到使用者。

## 第一性原理

API（Application Programming Interface）：框架提供能力，你调用。  
SPI（Service Provider Interface）：框架定义扩展点，你来实现，框架调用你的实现。

存在理由：**让框架在不修改源码的情况下支持可插拔扩展**——数据库驱动、日志实现、序列化协议均可替换，框架代码不变。

## 核心机制

### Java 原生 SPI（`java.util.ServiceLoader`）

**步骤一**：定义接口

```java
public interface IDriver { void connect(); }
```

**步骤二**：提供实现类

```java
public class MySQLDriver implements IDriver { ... }
```

**步骤三**：在 `META-INF/services/` 下创建配置文件

```
文件名：com.example.IDriver
内容：com.mysql.jdbc.MySQLDriver
```

**步骤四**：框架加载

```java
ServiceLoader<IDriver> loader = ServiceLoader.load(IDriver.class);
for (IDriver driver : loader) {
    driver.connect();
}
```

**加载流程**（ServiceLoader 内部）：
1. 读取 `META-INF/services/{接口全限定名}` 文件
2. `Class.forName()` 加载实现类
3. `newInstance()` 实例化
4. 缓存到 `LinkedHashMap`，懒加载

### SPI vs API

| | API | SPI |
|--|-----|-----|
| 面向 | 应用开发者 | 框架扩展者 |
| 谁定义接口 | 框架 | 框架 |
| 谁实现接口 | 框架 | 第三方/使用者 |
| 谁调用实现 | 使用者 | 框架 |
| 典型例子 | `java.util.List` | JDBC Driver |

## 关键权衡

**原生 SPI 的缺陷**：
1. **全量加载**：一次加载所有实现，不能按需加载
2. **不支持 Alias / 条件激活**：不能给实现命名，无法按名字选择
3. **线程安全**：ServiceLoader 非线程安全

**Dubbo SPI 的改进**：支持 `@SPI` 指定默认实现、`@Adaptive` 动态选择实现、按名字查找（`ExtensionLoader.getExtension("redis")`）。

**SpringBoot 的变体**：`spring.factories`（`META-INF/spring.factories`）是 Spring 对 SPI 的扩展，用于自动装配（`@EnableAutoConfiguration`）。

## 与其他概念的关系

- 依赖 [[机制-反射]]：ServiceLoader 通过 `Class.forName()` + `newInstance()` 实例化实现类
- 与 [[机制-动态代理]] 协作：Dubbo 的 `@Adaptive` 实现通过动态代理生成自适应扩展类
- 支撑了 L6 SpringBoot 自动装配：`AutoConfigurationImportSelector` 读取 `spring.factories` 是 SPI 思想的直接应用
- 支撑了 L7 Dubbo 扩展体系：整个 Dubbo 架构（Protocol/Cluster/LoadBalance 等）均基于增强 SPI 实现

## 应用边界

**适合用 SPI**：需要插件化、可替换实现的框架扩展点（日志、序列化、数据库驱动、加密算法）。

**不适合用**：业务代码中的简单策略选择（用 Spring Bean + 策略模式更直观）；需要按名字动态获取实现（用 Dubbo SPI 或 Spring ApplicationContext）。
