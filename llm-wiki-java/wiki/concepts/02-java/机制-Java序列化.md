---
type: concept
status: active
name: "Java序列化"
layer: L1
aliases: ["Serialization", "序列化", "反序列化", "Serializable", "Externalizable"]
related:
  - "[[机制-动态代理]]"
  - "[[概念-Java异常体系]]"
sources:
  - "../../../raw/note/Hollis/Java基础/✅什么是序列化与反序列化.md"
  - "../../../raw/note/Hollis/Java基础/✅Java序列化的原理是啥.md"
  - "../../../raw/note/Hollis/Java基础/✅serialVersionUID 有何用途 如果没定义会有什么问题？.md"
  - "../../../raw/note/Hollis/Java基础/✅你知道fastjson的反序列化漏洞吗.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# Java序列化

> 将 JVM 堆中对象的状态转换为与平台无关的字节序列，使对象能跨时间（持久化）、跨进程（网络传输）存在；反序列化是其逆过程。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 对象生命周期受限于进程、序列化导出状态 |
| [二、核心机制](#二核心机制) | Serializable、ObjectOutputStream、二进制协议、serialVersionUID |
| [三、Java 核心使用](#三java-核心使用) | 原生序列化 API、transient、Externalizable、自定义钩子 |
| [四、核心使用原则](#四核心使用原则) | UID 必须显式定义、敏感字段排除、父类可序列化 |
| [五、使用案例](#五使用案例) | 深拷贝、Redis 缓存、RPC 参数传递 |
| [六、综合对比](#六综合对比) | Java 原生 vs JSON vs Protobuf vs Hessian |
| [七、生产风险](#七生产风险) | 反序列化漏洞、版本不兼容、性能瓶颈 |
| [八、与其他概念的关系](#八与其他概念的关系) | 反射、异常体系、Redis、RPC |
| [九、应用边界](#九应用边界) | 适用与不适用场景 |

## 一、第一性原理

JVM 进程终止后堆内对象全部消亡。序列化解决的根本问题：**把对象状态从内存导出为可存储/可传输的字节流，需要时再还原**。这是持久化、网络通信、进程间通信的基础能力。

## 二、核心机制

### 2.1 原生序列化协议

1. 类实现 `java.io.Serializable`（标记接口，无方法）
2. `ObjectOutputStream.writeObject(obj)` 写入字节流
3. `ObjectInputStream.readObject()` 还原对象
4. 底层通过 [[机制-动态代理]] 遍历所有非 `static`、非 `transient` 字段，按固定二进制格式写入

### 2.2 serialVersionUID

```java
private static final long serialVersionUID = 1L;
```

| 场景 | 行为 |
|------|------|
| 显式定义 UID | 新增字段有默认值，旧数据可正常反序列化 |
| 未定义 UID | JVM 根据类结构（字段名、方法签名等）自动计算，类变更后 UID 变化，旧数据抛 `InvalidClassException` |

**面试高频误区**：serialVersionUID 不是"可选的"，生产环境必须显式定义，否则任何类结构调整都会破坏已持久化的数据。

### 2.3 序列化字段规则

| 规则 | 说明 |
|------|------|
| `static` 字段 | 不序列化，属于类而非实例 |
| `transient` 字段 | 不序列化，用于标记密码、Token 等敏感数据 |
| 父类未实现 Serializable | 父类字段不被序列化，反序列化时父类用无参构造器初始化 |
| 引用字段 | 引用的对象也必须实现 Serializable，否则抛 `NotSerializableException` |

## 三、Java 核心使用

### 3.1 自定义序列化钩子

```java
// 在可序列化类中声明（注意：是 private 方法，非接口方法）
private void writeObject(ObjectOutputStream out) throws IOException {
    out.defaultWriteObject();       // 写默认字段
    out.writeObject(encrypt(pwd));  // 自定义写入加密后的敏感字段
}

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
    in.defaultReadObject();
    this.pwd = decrypt((String) in.readObject());
}
```

### 3.2 Externalizable

实现 `Externalizable` 接口，完全自控序列化过程（需实现 `writeExternal` / `readExternal`），性能优于默认反射方式，但开发成本高。

### 3.3 readResolve 防止单例破坏

```java
private Object readResolve() throws ObjectStreamException {
    return INSTANCE; // 反序列化时返回已有单例
}
```

反序列化默认绕过构造器创建新对象，会破坏单例。`readResolve` 是标准防御手段。

## 四、核心使用原则

1. **必须显式定义 serialVersionUID**——避免类演进导致数据不兼容
2. **敏感字段加 `transient`**——密码、密钥、Session 不应进入字节流
3. **优先选择非 Java 原生序列化**——跨语言用 JSON/Protobuf，RPC 用 Hessian/Protobuf
4. **反序列化输入不可信时必须做白名单校验**——`ObjectInputFilter`（JDK 9+）或框架级白名单
5. **单例类必须实现 `readResolve`**——否则反序列化产生新实例

## 五、使用案例

| 场景 | 方案 | 说明 |
|------|------|------|
| 对象深拷贝 | 序列化 + 反序列化 | 简单但性能差，适合测试或低频场景 |
| Redis 缓存 | Jackson/Protobuf | Spring Data Redis 默认 JDK 序列化，生产建议替换为 JSON 或 Protobuf |
| RPC 参数传递 | Hessian/Protobuf | Dubbo 默认 Hessian2，gRPC 用 Protobuf |
| 分布式 Session | JDK 序列化 | Tomcat Session 持久化使用原生序列化 |

## 六、综合对比

| 框架 | 需要 Serializable | 跨语言 | 可读性 | 性能 | 安全性 |
|------|-------------------|--------|--------|------|--------|
| Java 原生 | 是 | 否 | 二进制不可读 | 差 | 反序列化漏洞风险高 |
| JSON（Jackson/Gson）| 否 | 是 | 好 | 中 | 需防范多态反序列化 |
| Protobuf | 否（需 .proto） | 是 | 二进制不可读 | 最优 | 强类型、风险低 |
| Hessian | 否 | 是（有限） | 二进制不可读 | 良 | 中等 |
| Kryo | 否 | 否 | 二进制不可读 | 优 | 需注册类白名单 |

**选型结论**：对外接口用 JSON；高性能 RPC 用 Protobuf；Dubbo 体系用 Hessian2；JVM 内部短期缓存可用 Kryo。Java 原生序列化仅在遗留系统中使用。

## 七、生产风险

| 风险 | 原因 | 防御 |
|------|------|------|
| 反序列化远程代码执行（RCE） | 反序列化时触发构造链（gadget chain），如 fastjson AutoType、Commons Collections | 白名单限制可反序列化类、禁用 AutoType、升级依赖 |
| 版本不兼容 | 未定义 UID 或删除/重命名字段 | 显式定义 UID、字段只增不删 |
| 性能瓶颈 | 原生序列化依赖反射，体积大 | 换用 Protobuf/Kryo |
| 单例被破坏 | 反序列化创建新实例 | 实现 `readResolve` |
| 循环引用 | 对象图中存在循环 | Java 原生支持（通过引用句柄），JSON 框架需注意配置 |

**面试高频追问**：fastjson 反序列化漏洞的核心原因是 AutoType 机制——JSON 中携带 `@type` 指定任意类，反序列化时通过 setter 触发恶意逻辑。防御措施：关闭 AutoType、升级到安全版本、使用 SafeMode。

## 八、与其他概念的关系

- 依赖 [[机制-动态代理]]：原生序列化通过反射读写对象字段，Externalizable 除外
- 关联 [[概念-Java异常体系]]：`writeObject` / `readObject` 抛 `IOException`（Checked），`InvalidClassException` 是版本不兼容的标志
- 支撑了 L5 Redis：Java 对象存入 Redis 需选择序列化方案，影响存储体积和可读性
- 支撑了 L7 RPC：Dubbo、gRPC 的参数传递性能直接取决于序列化协议选择

## 九、应用边界

| 适用 | 不适用 |
|------|--------|
| JVM 内部缓存持久化 | 跨语言通信（用 Protobuf/JSON） |
| 对象深拷贝（低频） | 对外暴露的 API 接口（安全风险） |
| 遗留系统兼容 | 高性能 RPC（原生序列化太慢） |
| Tomcat Session 持久化 | 长期数据存储（类演进会破坏兼容性） |
