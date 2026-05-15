---
type: concept
status: active
name: "MyBatis"
layer: L5
aliases: ["MyBatis", "MyBatis-Plus", "ORM", "Mapper", "PageHelper"]
related:
  - "[[机制-动态代理]]"
  - "[[机制-InnoDB索引]]"
  - "[[概念-SQL查询优化]]"
  - "[[概念-缓存三大问题]]"
sources:
  - "../../../raw/note/Hollis/MyBatis/✅Mybatis的工作原理？.md"
  - "../../../raw/note/Hollis/MyBatis/✅Mybatis的缓存机制.md"
  - "../../../raw/note/Hollis/MyBatis/✅#和$的区别是什么？什么情况必须用$.md"
  - "../../../raw/note/Hollis/MyBatis/✅Mybatis插件的运行原理？.md"
  - "../../../raw/note/Hollis/MyBatis/✅使用MyBatis如何实现分页？.md"
  - "../../../raw/note/Hollis/MyBatis/✅PageHelper分页的原理是什么？.md"
  - "../../../raw/note/Hollis/MyBatis/✅Mybatis 是否支持延迟加载？实现原理是什么？.md"
  - "../../../raw/note/Hollis/MyBatis/✅MyBatis与Hibernate有何不同？.md"
created: 2026-05-06
updated: 2026-05-15
lint_notes: ""
---

# MyBatis

> 半自动化 ORM 框架：开发者写 SQL，MyBatis 负责参数绑定、结果集映射和缓存管理；与 Hibernate 的核心区别是"SQL 控制权在开发者手中"。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | ORM 存在意义、半自动 vs 全自动 |
| [二、核心机制](#二核心机制) | 启动加载、运行时执行链路、动态代理 |
| [三、Java 核心使用](#三java-核心使用) | `#{}` vs `${}`、缓存、插件、分页、延迟加载、字段映射 |
| [四、综合对比](#四综合对比) | MyBatis vs Hibernate 选型 |
| [五、生产风险](#五生产风险) | SQL 注入、缓存脏读、分页内存溢出 |
| [六、与其他概念的关系](#六与其他概念的关系) | 动态代理、索引、缓存 |
| [七、应用边界](#七应用边界) | 适用场景与不适用场景 |

## 一、第一性原理

**ORM（Object Relational Mapping）** 解决的根本问题：Java 对象与关系型数据库之间的阻抗不匹配。手写 JDBC 需要反复处理 `Connection`→`PreparedStatement`→`ResultSet`→`close` 的样板代码，且参数绑定和结果集映射容易出错。

ORM 框架分两条路线：

| 路线 | 代表 | SQL 控制 | 代价 |
|------|------|---------|------|
| 全自动 | Hibernate | 框架生成 SQL | 复杂查询难优化，生成 SQL 不可控 |
| 半自动 | MyBatis | 开发者写 SQL | 需手写 SQL 和映射，开发量略大 |

MyBatis 选择半自动路线的根本原因：**在高并发、复杂查询场景下，SQL 的写法直接决定性能，开发者必须保留 SQL 控制权**。

## 二、核心机制

### 2.1 启动阶段

1. 解析 `mybatis-config.xml` 和 Mapper XML（或注解），载入内存构建 `Configuration` 对象
2. 通过 `SqlSessionFactory` 管理配置

### 2.2 运行阶段（执行一条 SELECT）

```
调用 Mapper 接口方法
  ↓
JDK 动态代理（MapperProxy）生成代理对象       → 见 [[机制-动态代理]]
  ↓
MapperMethod：解析入参、确定 SQL 类型
  ↓
CachingExecutor：检查二级缓存（namespace 级）
  ↓（miss）
BaseExecutor：检查一级缓存（SqlSession 级）
  ↓（miss）
SimpleExecutor → StatementHandler → JDBC PreparedStatement
  ↓
ResultSetHandler：结果集映射（列名 → 属性名）
```

Mapper 接口没有实现类，核心靠 JDK 动态代理：`MapperProxyFactory` 为每个 Mapper 接口创建 `MapperProxy`，拦截方法调用后委托给 `MapperMethod` 执行对应 SQL。

## 三、Java 核心使用

### 3.1 `#{}` vs `${}` 占位符（高频考点）

| 占位符 | 处理方式 | SQL 注入风险 | 适用场景 |
|--------|---------|------------|---------|
| `#{}` | JDBC PreparedStatement `?` 预编译 | **无**（参数安全转义）| 所有值参数（绝大多数情况）|
| `${}` | 字符串直接拼接到 SQL | **有** | 动态表名、列名、ORDER BY 字段（无法预编译的 SQL 结构）|

```xml
<!-- 安全 -->
SELECT * FROM user WHERE id = #{userId}
<!-- 危险（必须使用时需枚举/白名单校验输入值） -->
SELECT * FROM user ORDER BY ${sortField}
```

**面试高频误区**：不是"所有情况都用 `#{}`"——动态表名、列名、`ORDER BY` 字段在 JDBC 层面无法用 `?` 预编译，此时只能用 `${}`，但必须做输入校验。

### 3.2 一级缓存 vs 二级缓存

| 维度 | 一级缓存 | 二级缓存 |
|------|---------|---------|
| 作用域 | SqlSession 级（默认开启）| namespace 级（手动开启）|
| 底层结构 | PerpetualCache（HashMap，无容量限制）| 可配置（LruCache、BlockingCache 等，装饰者模式）|
| 清除时机 | update/delete 执行、事务提交/回滚 | 事务提交后刷新 |
| 多表查询脏读 | 存在（不同 SqlSession 间）| 更严重（namespace 隔离导致跨表更新不感知）|
| 实践建议 | 保持默认，必要时设 `flushCache=true` | **不建议用**，改用 Redis 等外部缓存 |

**为何不推荐二级缓存**：student namespace 缓存了 student+class 联表结果，class namespace 更新 class 表后，student namespace 的缓存不会失效，产生脏数据，且难以排查。

### 3.3 插件机制（责任链模式）

MyBatis 允许拦截四个核心组件：`Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`。

```java
@Intercepts({@Signature(type = Executor.class, method = "query", args = {...})})
public class MyPlugin implements Interceptor {
    public Object intercept(Invocation invocation) throws Throwable {
        // 前置处理
        Object result = invocation.proceed();
        // 后置处理
        return result;
    }
}
```

所有插件通过 `InterceptorChain` 按注册顺序形成责任链，PageHelper 就是利用此机制在 SQL 执行前拼接 `LIMIT`。

### 3.4 分页方案对比

| 方案 | 类型 | 原理 | 适用 |
|------|------|------|------|
| 手写 `LIMIT` | 物理 | SQL 直接带分页 | 简单场景 |
| PageHelper | 物理 | ThreadLocal 存分页参数 + 插件拦截 SQL 拼接 `LIMIT` | 最常用 |
| MyBatis-Plus `PaginationInnerInterceptor` | 物理 | 同 PageHelper 思路，通过 `beforeQuery` 改写 SQL | MyBatis-Plus 项目 |
| RowBounds | **逻辑** | 全量查询后在内存中截取 | 数据量小，不推荐 |

**大数据量必须用物理分页**（RowBounds 全量查询会撑爆内存并拖慢数据库）。

### 3.5 延迟加载

开启后，查询主对象时 MyBatis 返回**代理对象**，只有真正访问关联属性时才触发额外 SQL 查询。

```xml
<settings>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

适合"查主表但不一定用关联表"的场景（如查订单列表，不总是需要每条订单的商品明细）。

### 3.6 字段映射

优先级：`resultMap` 显式映射 > `AS` 别名 > 列名自动驼峰转换（需开启 `mapUnderscoreToCamelCase`）> 列名直接匹配属性名。自定义类型转换实现 `TypeHandler` 接口（如枚举映射、日期格式化）。

## 四、综合对比

| 维度 | MyBatis | Hibernate |
|------|---------|-----------|
| SQL 控制 | 开发者手写，精细可控 | 框架自动生成，HQL/JPQL 可干预 |
| 学习成本 | 低，会 SQL 就能用 | 高，需理解 Session、缓存、级联策略 |
| 性能优化 | 直接优化 SQL | 需理解框架生成的 SQL 再优化 |
| DB 移植性 | 低（SQL 方言绑定）| 高（方言抽象层）|
| 适用场景 | 高并发、复杂查询、DBA 强参与 | 快速开发、标准 CRUD、跨 DB |

## 五、生产风险

| 风险 | 原因 | 应对 |
|------|------|------|
| SQL 注入 | `${}` 拼接用户输入 | 枚举/白名单校验，优先 `#{}` |
| 二级缓存脏读 | namespace 隔离导致跨表更新不感知 | 生产不用二级缓存，改用 Redis |
| RowBounds 内存溢出 | 逻辑分页全量加载数据 | 用物理分页（PageHelper）|
| 一级缓存误命中 | 同一 SqlSession 内 update 后未刷新缓存 | 设 `flushCache=true` 或手动清缓存 |
| 延迟加载 N+1 | 循环访问关联属性，触发 N 次额外 SQL | 预判场景，必要时用 JOIN 一次查出 |

## 六、与其他概念的关系

- 底层用 [[机制-动态代理]]：`MapperProxy` 通过 JDK 动态代理让接口方法"有实现"
- 执行的 SQL 依赖 [[机制-InnoDB索引]] 的索引设计，慢 SQL 通过 [[概念-SQL查询优化]] 排查
- MyBatis 一/二级缓存是本地缓存，生产更多用 [[概念-缓存三大问题]] 中讨论的 Redis 缓存方案
- 插件责任链是 [[机制-动态代理]] 的具体应用场景之一

## 七、应用边界

**推荐 MyBatis**：需要精细控制 SQL（大厂高并发场景、复杂联表、特殊优化）；数据库不会频繁切换。

**推荐 Hibernate**：快速开发、ERP/管理系统等标准 CRUD；对 DB 可移植性有要求。

**实践要点**：
- `${}` 必须使用枚举或白名单校验，防止 SQL 注入
- 二级缓存生产环境几乎不用，改用 Redis
- 分页默认用 PageHelper（物理分页），RowBounds 仅限小数据量
