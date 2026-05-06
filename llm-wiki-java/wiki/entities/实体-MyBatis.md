---
type: entity
status: active
name: "MyBatis"
layer: L5
aliases: ["MyBatis", "MyBatis-Plus", "ORM", "Mapper", "PageHelper"]
related:
  - "[[机制-动态代理]]"
  - "[[机制-InnoDB索引模型]]"
  - "[[概念-SQL查询优化]]"
  - "[[概念-缓存三大问题]]"
sources:
  - "../../raw/note/Hollis/MyBatis/✅Mybatis的工作原理？.md"
  - "../../raw/note/Hollis/MyBatis/✅Mybatis的缓存机制.md"
  - "../../raw/note/Hollis/MyBatis/✅#和$的区别是什么？什么情况必须用$.md"
  - "../../raw/note/Hollis/MyBatis/✅Mybatis插件的运行原理？.md"
  - "../../raw/note/Hollis/MyBatis/✅使用MyBatis如何实现分页？.md"
  - "../../raw/note/Hollis/MyBatis/✅PageHelper分页的原理是什么？.md"
  - "../../raw/note/Hollis/MyBatis/✅Mybatis 是否支持延迟加载？实现原理是什么？.md"
  - "../../raw/note/Hollis/MyBatis/✅MyBatis与Hibernate有何不同？.md"
created: 2026-05-06
updated: 2026-05-06
lint_notes: ""
---

# MyBatis

> 半自动化 ORM 框架：开发者写 SQL，MyBatis 负责参数绑定、结果集映射和缓存管理；与 Hibernate 的核心区别是"SQL 控制权在开发者手中"。

## 定位与选型

**ORM（Object Relational Mapping）**：将 Java 对象与数据库表的 CRUD 操作关联，省去手写 JDBC 样板代码。

| 框架 | 模式 | SQL 控制 | 适用场景 |
|------|------|---------|---------|
| MyBatis | 半自动 | 开发者写 SQL | 高性能要求、复杂 SQL、定制优化 |
| Hibernate | 全自动 | 框架自动生成 | 快速开发、标准 CRUD、DB 可移植 |

## 工作原理

### 启动阶段
1. 解析 `mybatis-config.xml` 和 Mapper XML（或注解），载入内存构建 `Configuration` 对象
2. 通过 `SqlSessionFactory` 管理配置

### 运行阶段（执行一条 SELECT）

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

## 核心机制

### # vs $ 占位符（高频考点）

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

### 一级缓存 vs 二级缓存

| 维度 | 一级缓存 | 二级缓存 |
|------|---------|---------|
| 作用域 | SqlSession 级（默认开启）| namespace 级（手动开启）|
| 底层结构 | PerpetualCache（HashMap，无容量限制）| 可配置（LruCache、BlockingCache 等，装饰者模式）|
| 清除时机 | update/delete 执行、事务提交/回滚 | 事务提交后刷新 |
| 多表查询脏读 | 存在（不同 SqlSession 间）| 更严重（namespace 隔离导致跨表更新不感知）|
| 实践建议 | 保持默认，必要时设 `flushCache=true` | **不建议用**，改用 Redis 等外部缓存 |

**为何不推荐二级缓存**：student namespace 缓存了 student+class 联表结果，class namespace 更新 class 表后，student namespace 的缓存不会失效，产生脏数据，且难以排查。

### 插件机制（责任链模式）

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

### 分页方案对比

| 方案 | 类型 | 原理 | 适用 |
|------|------|------|------|
| 手写 `LIMIT` | 物理 | SQL 直接带分页 | 简单场景 |
| PageHelper | 物理 | ThreadLocal 存分页参数 + 插件拦截 SQL 拼接 `LIMIT` | 最常用 |
| MyBatis-Plus `PaginationInnerInterceptor` | 物理 | 同 PageHelper 思路，通过 `beforeQuery` 改写 SQL | MyBatis-Plus 项目 |
| RowBounds | **逻辑** | 全量查询后在内存中截取 | 数据量小，不推荐 |

**大数据量必须用物理分页**（RowBounds 全量查询会撑爆内存并拖慢数据库）。

### 延迟加载

开启后，查询主对象时 MyBatis 返回**代理对象**，只有真正访问关联属性时才触发额外 SQL 查询。

```xml
<settings>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

适合"查主表但不一定用关联表"的场景（如查订单列表，不总是需要每条订单的商品明细）。

### 字段映射

优先级：`resultMap` 显式映射 > `AS` 别名 > 列名自动驼峰转换（需开启 `mapUnderscoreToCamelCase`）> 列名直接匹配属性名。自定义类型转换实现 `TypeHandler` 接口（如枚举映射、日期格式化）。

## 与其他概念的关系

- 底层用 [[机制-动态代理]]：`MapperProxy` 通过 JDK 动态代理让接口方法"有实现"
- 执行的 SQL 依赖 [[机制-InnoDB索引模型]] 的索引设计，慢 SQL 通过 [[概念-SQL查询优化]] 排查
- MyBatis 一/二级缓存是本地缓存，生产更多用 [[概念-缓存三大问题]] 中讨论的 Redis 缓存方案
- 插件责任链是 [[机制-动态代理]] 的具体应用场景之一

## 应用边界

**推荐 MyBatis**：需要精细控制 SQL（大厂高并发场景、复杂联表、特殊优化）；数据库不会频繁切换。

**推荐 Hibernate**：快速开发、ERP/管理系统等标准 CRUD；对 DB 可移植性有要求。

**实践要点**：
- `${}` 必须使用枚举或白名单校验，防止 SQL 注入
- 二级缓存生产环境几乎不用，改用 Redis
- 分页默认用 PageHelper（物理分页），RowBounds 仅限小数据量
