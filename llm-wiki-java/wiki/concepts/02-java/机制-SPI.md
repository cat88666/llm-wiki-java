---
type: concept
status: active
name: "SPI机制"
layer: L1
aliases: ["SPI", "Service Provider Interface", "ServiceLoader", "ExtensionLoader"]
related:
  - "[[机制-动态代理]]"
  - "[[机制-Dubbo]]"
  - "[[机制-Spring]]"
sources:
  - "../../../raw/note/Hollis/Java基础/"
  - "../../../raw/note/Hollis/Dubbo.md"
updated: 2026-05-18
---

# SPI机制

> SPI 把“实现选择权”从框架代码转移到部署配置，核心价值是可插拔扩展，核心代价是加载时机、类加载器与治理复杂度上升。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、SPI 的本质与价值](#sec-2) | API 与 SPI 的控制反转 |
| [二、Java SPI 执行链路](#sec-3) | `META-INF/services` + ServiceLoader |
| [三、类加载器与委派破局](#sec-4) | TCCL 解决父加载器看不到子实现 |
| [四、主流框架 SPI 变体](#sec-5) | Java SPI / Dubbo SPI / Spring SPI |
| [五、设计与实现原则](#sec-6) | 扩展点命名、兼容策略、默认实现 |
| [六、选型对比结论](#sec-7) | 不同 SPI 机制适用场景 |
| [七、生产风险与排查](#sec-8) | 全量扫描、类冲突、实例线程安全 |
| [八、关系与边界](#sec-9) | 与代理、反射、配置中心关系 |
| [九、面试速答口径](#sec-10) | 高频问题答案 |

<a id="sec-2"></a>
## 一、SPI 的本质与价值

| 维度 | API | SPI |
| --- | --- | --- |
| 调用方向 | 业务调用框架 | 框架回调业务实现 |
| 实现归属 | 框架提供 | 使用方或第三方提供 |
| 扩展方式 | 改代码 | 加 jar + 配置 |

SPI 的核心目标：不改框架核心代码，仍可扩展协议、驱动、策略。

<a id="sec-3"></a>
## 二、Java SPI 执行链路

```text
定义接口
  -> 实现类打包到 jar
  -> jar 中声明 META-INF/services/<接口全限定名>
  -> ServiceLoader.load(接口)
  -> 懒迭代加载实现
```

关键代码：

```java
ServiceLoader<MyService> loader = ServiceLoader.load(MyService.class);
for (MyService svc : loader) {
    svc.execute();
}
```

反直觉点：`ServiceLoader` 是懒加载，遍历时才实例化实现。

<a id="sec-4"></a>
## 三、类加载器与委派破局

JDBC 场景中，接口 `java.sql.Driver` 由引导类加载器可见，但具体驱动类在应用 Classpath。Java SPI 通过线程上下文类加载器（TCCL）让父层代码“反向”看到子层实现。

<a id="sec-5"></a>
## 四、主流框架 SPI 变体

| 机制 | 扩展声明 | 按名查找 | IOC/AOP | 典型场景 |
| --- | --- | --- | --- | --- |
| Java SPI | `META-INF/services` | 弱 | 无 | JDBC 驱动、JDK 扩展 |
| Dubbo SPI | `META-INF/dubbo` | 强（name->impl） | 支持自动包装 | 协议、序列化、负载均衡 |
| Spring SPI | `spring.factories` / `AutoConfiguration.imports` | 条件化 | 强 | 自动装配 |

<a id="sec-6"></a>
## 五、设计与实现原则

1. 扩展点必须稳定：接口尽量小、版本演进可兼容。
2. 默认实现必须可用：避免“无实现即启动失败”。
3. 实现类必须无副作用初始化：否则扫描阶段放大风险。
4. 扩展命名可治理：建议统一 key 命名和文档。

<a id="sec-7"></a>
## 六、选型对比结论

```text
只需标准发现机制           -> Java SPI
需要按名称路由和扩展包装     -> Dubbo SPI
需要条件装配与容器生命周期   -> Spring SPI
```

<a id="sec-8"></a>
## 七、生产风险与排查

| 风险 | 触发条件 | 现象 | 处理 |
| --- | --- | --- | --- |
| 启动变慢 | 扩展实现过多、初始化重 | 冷启动慢 | 懒初始化 + 采样启动日志 |
| 类冲突 | 多版本实现同名 | 找到错误实现 | 锁定依赖版本、排查 classpath |
| 线程安全问题 | 扩展实例持有可变状态 | 并发异常 | 无状态实现或加并发保护 |
| 扩展失效 | 配置文件路径错误 | 无实现被发现 | 校验打包产物与路径 |

<a id="sec-9"></a>
## 八、关系与边界

- 与 [[机制-动态代理]]：SPI 负责“找实现”，动态代理负责“增强调用”。
- 与 [[机制-Dubbo]]：Dubbo SPI 是 Java SPI 的工程化增强。
- 与 [[机制-Spring]]：Spring SPI 结合条件注解和容器生命周期。

边界：SPI 适合框架扩展，不适合业务层频繁条件分支。

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| SPI 和 API 差异 | API 是你调用框架，SPI 是框架回调你的实现 |
| Java SPI 如何加载实现 | `META-INF/services` + `ServiceLoader` 懒加载 |
| JDBC 为什么不再手动 `Class.forName` | JDBC 4 通过 SPI 自动发现驱动 |
| SPI 最大风险 | 可观测性弱，类加载和初始化时机问题难排查 |
