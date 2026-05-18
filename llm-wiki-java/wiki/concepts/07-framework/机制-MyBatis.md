---
type: concept
status: active
name: "MyBatis"
layer: L5
aliases: ["MyBatis", "MyBatis-Plus", "ORM", "Mapper", "PageHelper", "一级缓存", "二级缓存", "TypeHandler", "ResultMap"]
related:
  - "[[机制-动态代理]]"
  - "[[概念-MySQL]]"
  - "[[概念-Redis]]"
  - "[[机制-Spring]]"
sources:
  - "../../../raw/note/Hollis/网络安全/✅什么是SQL注入攻击？如何防止.md"
  - "../../../raw/note/Hollis/MyBatis/"
updated: 2026-05-18
---

# MyBatis

> MyBatis 是半自动 ORM：开发者保留 SQL 控制权，框架负责 Mapper 代理、参数绑定、结果映射、缓存与插件扩展；核心权衡是“SQL 可控”换来“映射与缓存治理成本”。

<a id="sec-1"></a>
## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、MyBatis 的定位](#sec-2) | 半自动 ORM 为什么适合复杂 SQL |
| [二、Mapper 调用链路](#sec-3) | 动态代理、Executor、StatementHandler、ResultSetHandler |
| [三、SQL 参数与结果映射](#sec-4) | `#{}`/`${}`、resultMap、TypeHandler |
| [四、缓存与分页机制](#sec-5) | 一级缓存、二级缓存、PageHelper、RowBounds |
| [五、插件扩展与 MyBatis-Plus](#sec-6) | Interceptor 责任链与工程增强 |
| [六、选型对比](#sec-7) | MyBatis vs Hibernate vs JPA |
| [七、生产风险与排查](#sec-8) | SQL 注入、缓存脏读、N+1、分页内存 |
| [八、关系与边界](#sec-9) | 与 MySQL、Redis、动态代理、Spring 的关系 |
| [九、面试速答口径](#sec-10) | 高频问题可直接回答 |

<a id="sec-2"></a>
## 一、MyBatis 的定位

JDBC 的痛点是样板代码多：连接管理、参数绑定、结果集映射和资源关闭都要手写。全自动 ORM 又会把 SQL 生成权交给框架，复杂查询和性能优化难以精确控制。

MyBatis 选择中间路线：

| 路线 | SQL 控制权 | 优势 | 代价 |
| --- | --- | --- | --- |
| JDBC | 开发者 | 最可控 | 样板代码多 |
| MyBatis | 开发者 | SQL 可控，映射自动化 | XML/注解映射要治理 |
| Hibernate/JPA | 框架 | CRUD 快 | 复杂 SQL 与性能不透明 |

结论：复杂查询、高并发、DBA 强参与的系统更适合 MyBatis。

<a id="sec-3"></a>
## 二、Mapper 调用链路

Mapper 接口没有实现类，真正的实现来自 JDK 动态代理。

```text
调用 Mapper 接口方法
  -> MapperProxy 拦截
  -> MapperMethod 解析方法与 SQL 映射
  -> Executor 执行查询/更新
  -> StatementHandler 创建 PreparedStatement
  -> ParameterHandler 绑定参数
  -> ResultSetHandler 映射结果集
```

核心组件职责：

| 组件 | 职责 |
| --- | --- |
| `SqlSessionFactory` | 持有全局配置并创建 `SqlSession` |
| `SqlSession` | 一次数据库会话，承载一级缓存 |
| `Executor` | 执行 SQL、管理缓存 |
| `MappedStatement` | 一条 SQL 的完整元数据 |
| `ResultSetHandler` | 列到对象属性的映射 |

<a id="sec-4"></a>
## 三、SQL 参数与结果映射

### 3.1 `#{}` vs `${}`

| 写法 | 底层行为 | 风险 | 使用场景 |
| --- | --- | --- | --- |
| `#{}` | 预编译参数，占位符 `?` | 防 SQL 注入 | 值参数 |
| `${}` | 字符串拼接 | 高风险 | 表名、列名、排序字段 |

`ORDER BY ${sort}` 这类场景无法用 `?` 代替 SQL 结构，只能白名单校验后拼接。

### 3.2 结果映射优先级

```text
resultMap 显式映射
  -> SQL AS 别名
  -> mapUnderscoreToCamelCase
  -> 字段名直接匹配
```

复杂联表、嵌套对象、枚举转换优先使用 `resultMap` 和 `TypeHandler`，避免把映射规则散落在 SQL 里。

<a id="sec-5"></a>
## 四、缓存与分页机制

### 4.1 一级缓存与二级缓存

| 维度 | 一级缓存 | 二级缓存 |
| --- | --- | --- |
| 作用域 | `SqlSession` | Mapper namespace |
| 默认状态 | 开启 | 需显式开启 |
| 失效时机 | update/commit/rollback | namespace 维度刷新 |
| 生产建议 | 保持默认，必要时清理 | 谨慎使用，通常改 Redis |

二级缓存最大问题是 namespace 隔离：多表联查缓存和跨 namespace 更新无法天然联动，容易脏读。

### 4.2 分页方案

| 方案 | 类型 | 结论 |
| --- | --- | --- |
| 手写 `LIMIT` | 物理分页 | 简单稳定 |
| PageHelper | 物理分页 | ThreadLocal + 插件改写 SQL |
| MyBatis-Plus 分页插件 | 物理分页 | MP 项目首选 |
| RowBounds | 逻辑分页 | 小数据可用，生产大数据禁用 |

<a id="sec-6"></a>
## 五、插件扩展与 MyBatis-Plus

MyBatis 插件拦截四类核心对象：`Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`。

```text
Invocation
  -> Interceptor 1
  -> Interceptor 2
  -> target.proceed()
```

常见用途：

| 用途 | 拦截点 |
| --- | --- |
| 分页 | `StatementHandler` 或 `Executor` |
| SQL 审计 | `Executor` |
| 多租户条件 | SQL 改写阶段 |
| 慢 SQL 记录 | 执行前后计时 |

MyBatis-Plus 的价值是减少标准 CRUD 样板代码，但复杂 SQL 仍应回到手写 Mapper。

<a id="sec-7"></a>
## 六、选型对比

| 维度 | MyBatis | Hibernate/JPA |
| --- | --- | --- |
| SQL 可控性 | 强 | 中 |
| 上手成本 | 低 | 高 |
| 复杂查询 | 直接写 SQL | 需理解生成 SQL |
| 数据库迁移 | SQL 方言绑定 | 方言抽象更好 |
| 适用 | 交易系统、复杂报表、高并发服务 | 标准 CRUD、后台管理系统 |

<a id="sec-8"></a>
## 七、生产风险与排查

| 风险 | 触发条件 | 现象 | 应对 |
| --- | --- | --- | --- |
| SQL 注入 | `${}` 拼用户输入 | 数据泄露/篡改 | 枚举白名单，值参数用 `#{}` |
| 二级缓存脏读 | 联表查询 + 跨 namespace 更新 | 读到旧数据 | 关闭二级缓存，外置 Redis |
| N+1 查询 | 循环触发延迟加载 | SQL 数暴增 | JOIN/批量查询/显式预取 |
| 逻辑分页 OOM | RowBounds 大结果集 | 内存飙升 | 物理分页 |
| Mapper 结果错映射 | 列名冲突或别名缺失 | 字段错值/null | resultMap 显式声明 |

<a id="sec-9"></a>
## 八、关系与边界

- 依赖 [[机制-动态代理]]：Mapper 接口通过 `MapperProxy` 获得运行期实现。
- 依赖 [[概念-MySQL]]：SQL 质量、索引和事务隔离决定真实性能。
- 关联 [[概念-Redis]]：MyBatis 二级缓存不适合跨表一致性场景，生产缓存通常外置。
- 集成 [[机制-Spring]]：Mapper 扫描后作为 Bean 注入业务层。

边界：MyBatis 管的是 SQL 执行与映射，不负责数据库建模、索引设计和缓存一致性闭环。

<a id="sec-10"></a>
## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| MyBatis 为什么是半自动 ORM | SQL 开发者写，参数绑定和结果映射框架做 |
| Mapper 接口为什么能调用 | JDK 动态代理生成 `MapperProxy`，转为 `MappedStatement` 执行 |
| `#{}` 和 `${}` 区别 | `#{}` 预编译防注入，`${}` 字符串拼接需白名单 |
| 为什么不推荐二级缓存 | namespace 隔离导致跨表更新无法可靠失效 |
| RowBounds 为什么危险 | 逻辑分页先查全量再内存截取 |
