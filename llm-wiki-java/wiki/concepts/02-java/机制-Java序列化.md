---
type: concept
status: active
name: "Java序列化"
layer: L1
aliases: ["Serialization", "序列化", "反序列化", "Serializable", "Externalizable"]
related:
  - "[[机制-动态代理]]"
  - "[[概念-Java异常]]"
  - "[[机制-SPI]]"
  - "[[概念-Redis]]"
  - "[[机制-Dubbo]]"
sources:
  - "../../../raw/note/Hollis/Java基础/✅serialVersionUID 有何用途 如果没定义会有什么问题？.md"
  - "../../../raw/note/Hollis/Java基础/✅你知道fastjson的反序列化漏洞吗.md"
updated: 2026-05-18
---

# Java序列化

> Java 序列化把“JVM 内存态对象”转换成“可落盘、可传输的字节序列”，核心矛盾是可演进性、性能、安全性三者不能同时最优。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、序列化要解决的本质问题](#一序列化要解决的本质问题) | 跨进程与跨时间保存对象状态 |
| [二、Java 原生序列化机制链路](#二java-原生序列化机制链路) | Serializable、ObjectStream、对象图编码 |
| [三、serialVersionUID 与类演进兼容](#三serialversionuid-与类演进兼容) | UID 校验机制与变更策略 |
| [四、字段规则与自定义钩子](#四字段规则与自定义钩子) | transient、writeObject/readObject、readResolve |
| [五、协议选型与场景匹配](#五协议选型与场景匹配) | JDK/JSON/Protobuf/Hessian/Kryo 对比 |
| [六、生产风险与防御策略](#六生产风险与防御策略) | 反序列化漏洞、兼容事故、性能问题 |
| [七、实战排障与迁移思路](#七实战排障与迁移思路) | InvalidClassException、协议切换、灰度 |
| [八、关系图与边界](#八关系图与边界) | 与异常、RPC、缓存、类加载的关系 |
| [九、面试速答口径](#九面试速答口径) | 高频问题的可直接复述答案 |

## 一、序列化要解决的本质问题

对象只在当前 JVM 进程内存活。进程退出后，堆内对象状态消失。序列化的核心价值是把对象状态从“瞬时内存态”导出为“可复用字节态”。

```
JVM 对象
  -> 序列化（编码）
  -> 字节流（磁盘/网络/缓存）
  -> 反序列化（解码）
  -> JVM 对象
```

典型需求：

- 持久化（文件、数据库 BLOB）
- 远程调用（RPC 请求/响应体）
- 分布式缓存（Redis value）
- 深拷贝（低频工具场景）

## 二、Java 原生序列化机制链路

### 2.1 标准流程

| 步骤 | 关键类 | 说明 |
| --- | --- | --- |
| 编码 | `ObjectOutputStream#writeObject` | 写入类描述 + 字段值 + 引用关系 |
| 解码 | `ObjectInputStream#readObject` | 根据类描述恢复对象并填充字段 |
| 标记 | `Serializable` | 标记接口，无方法，实现即声明“可序列化” |

### 2.2 原生协议不是“按字段简单拼接”

Java 原生序列化保存的不只是字段值，还包括对象图结构（共享引用、循环引用）和类描述信息。

- 优点：对象图还原能力强
- 代价：字节冗余、协议绑定 Java 生态、反射开销高

### 2.3 与 `Externalizable` 的分工

| 机制 | 可控性 | 性能 | 代价 |
| --- | --- | --- | --- |
| `Serializable` | 中（默认 + 可定制钩子） | 中 | 最省开发成本 |
| `Externalizable` | 高（完全自定义） | 可更高 | 代码复杂，兼容责任全自担 |

## 三、serialVersionUID 与类演进兼容

### 3.1 UID 校验规则

反序列化时，流中的 UID 必须与本地类 UID 一致，否则抛 `InvalidClassException`。

```java
private static final long serialVersionUID = 1L;
```

### 3.2 不显式定义 UID 的风险

未声明 UID 时，JVM 会根据类结构计算 UID。字段增删改、方法签名变化都可能导致 UID 变化，历史数据无法反序列化。

### 3.3 类演进策略（生产可执行）

| 变更类型 | 是否兼容历史数据 | 建议 |
| --- | --- | --- |
| 新增字段 | 通常兼容 | 显式 UID 不变，给新字段兜底默认值 |
| 删除字段 | 风险高 | 先废弃后下线，分阶段迁移 |
| 字段改名/改类型 | 高风险 | 通过自定义 `readObject` 做兼容映射 |
| UID 变更 | 不兼容 | 非必要不改 UID |

反直觉点：UID 不是版本号工具，而是“反序列化兼容闸门”。

## 四、字段规则与自定义钩子

### 4.1 字段参与规则

| 字段类型 | 是否参与序列化 | 说明 |
| --- | --- | --- |
| 普通实例字段 | 是 | 默认参与 |
| `static` 字段 | 否 | 属于类而不是对象状态 |
| `transient` 字段 | 否 | 用于排除敏感或不可持久化状态 |

### 4.2 自定义钩子：把“默认行为”变成“可控行为”

```java
private void writeObject(ObjectOutputStream out) throws IOException {
    out.defaultWriteObject();                // 默认字段
    out.writeObject(encrypt(password));      // 敏感字段单独处理
}

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
    in.defaultReadObject();
    this.password = decrypt((String) in.readObject());
}
```

### 4.3 `readResolve`：防止单例失效

反序列化默认会创建新实例，可能破坏单例语义。通过 `readResolve` 回收为既有实例。

```java
private Object readResolve() {
    return INSTANCE;
}
```

## 五、协议选型与场景匹配

### 5.1 主流协议横向对比

| 协议 | 跨语言 | 性能 | 可读性 | 安全面 | 典型场景 |
| --- | --- | --- | --- | --- | --- |
| JDK 原生 | 差 | 较低 | 差 | 风险高（反序列化攻击面） | 遗留系统、短期兼容 |
| JSON（Jackson/Gson） | 好 | 中 | 好 | 中（多态配置要收敛） | 对外 API、日志、配置 |
| Protobuf | 好 | 高 | 差（二进制） | 较好（强 schema） | gRPC、高吞吐 RPC |
| Hessian2 | 中 | 高 | 差 | 中 | Dubbo 生态常见默认 |
| Kryo | 差（偏 JVM） | 很高 | 差 | 中（需类注册策略） | 内网 JVM 高性能传输 |

### 5.2 选型决策树

```text
需要跨语言 + 可观测调试   -> JSON
需要高性能 + 强 schema     -> Protobuf
已有 Dubbo 体系            -> Hessian2（或按网关策略升级）
遗留系统短期兼容           -> JDK 原生（但应制定迁移计划）
```

> 面试口径：我们在支付链路把内部 RPC 从 JDK 序列化迁到 Protobuf，主要是为了降低报文体积和 CPU 反序列化开销；压测下 P99 延迟下降、网络带宽占用下降，且协议可演进性更强。

## 六、生产风险与防御策略

| 风险 | 触发条件 | 典型现象 | 防御策略 |
| --- | --- | --- | --- |
| 反序列化 RCE | 不可信输入 + 可利用 gadget | 远程命令执行、服务被控 | 禁止反序列化外部不可信字节流；白名单过滤；升级依赖 |
| 版本不兼容 | UID 漂移或字段破坏性变更 | `InvalidClassException` | 显式 UID；兼容读取；灰度双读 |
| 性能瓶颈 | 大对象频繁编解码 | RT 抖动、CPU 飙升 | 协议升级（Protobuf/Kryo）、对象瘦身、批量化 |
| 数据泄漏 | 敏感字段未排除 | 密码/Token 落盘 | `transient` + 自定义加密写入 |
| 单例破坏 | 反序列化重建对象 | 单例语义失效 | `readResolve` |

### 6.1 fastjson 漏洞本质（高频追问）

- 风险根因：AutoType 允许输入指定目标类型，反序列化时可触发恶意链。
- 防御原则：关闭危险能力、启用安全模式/白名单、持续升级到修复版本。

## 七、实战排障与迁移思路

### 7.1 `InvalidClassException` 处理流程

```text
确认异常类名/UID
  -> 对比线上字节流版本与当前类定义
  -> 判断是 UID 漂移还是字段破坏性变更
  -> 制定兼容读取/数据回填/重建缓存方案
```

### 7.2 从 JDK 原生迁移到 Protobuf 的常见路径

1. 定义 `.proto` 与版本策略（字段编号不可复用）
2. 新老协议并存（双写或双编解码）
3. 灰度流量验证兼容性与性能
4. 清理旧协议入口，收敛攻击面

## 八、关系图与边界

### 8.1 概念关系

```
Java 序列化
├── 依赖 [[机制-动态代理]] / 反射能力（默认实现路径）
├── 关联 [[概念-Java异常]]（IO/类不兼容异常链路）
├── 支撑 [[概念-Redis]]（对象入缓存的编码方案）
├── 支撑 [[机制-Dubbo]]（RPC 参数编解码）
└── 与 [[机制-SPI]] 协同（序列化扩展实现发现）
```

### 8.2 应用边界

| 能解决 | 不能解决 |
| --- | --- |
| 对象状态持久化与传输 | 跨语言强兼容（需选跨语言协议） |
| JVM 内部对象图恢复 | 不可信输入的天然安全（必须额外防御） |
| 低频深拷贝便利性 | 高性能与强安全同时最优（需要权衡） |

## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| 为什么要显式定义 `serialVersionUID` | 防止类结构变更导致 UID 漂移，历史数据反序列化失败 |
| `transient` 的作用 | 排除不应持久化字段，常用于敏感信息或临时态 |
| JDK 序列化为什么不推荐对外 | 协议 Java 绑定、性能一般、反序列化安全风险高 |
| 如何防反序列化漏洞 | 不反序列化不可信数据 + 类型白名单 + 依赖升级 + 关闭高危特性 |
| 什么时候用 Protobuf 而不是 JSON | 内部高吞吐 RPC、带宽敏感、需要稳定 schema 演进 |

> 可直接复述：序列化选型不是“谁更快”单指标问题，而是“兼容性、性能、安全”三维决策；外部接口优先 JSON，可演进高性能链路优先 Protobuf，JDK 原生只做遗留兼容并尽快迁移。
