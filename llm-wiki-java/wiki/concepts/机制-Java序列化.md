---
type: concept
status: active
name: "Java序列化"
layer: L1
aliases: ["Serialization", "序列化", "反序列化", "Serializable"]
related:
  - "[[机制-反射]]"
  - "[[概念-Java异常体系]]"
sources:
  - "../../raw/note/📚 Hollis Java/Java基础/✅什么是序列化与反序列化 30f3673e1138816b9400e50e4e845d2c.md"
  - "../../raw/note/📚 Hollis Java/Java基础/✅Java序列化的原理是啥 30f3673e113881ba98d5dc41e68fa0fc.md"
  - "../../raw/note/📚 Hollis Java/Java基础/✅serialVersionUID 有何用途 如果没定义会有什么问题？ 30f3673e1138815b9405c65617ba5e97.md"
  - "../../raw/note/📚 Hollis Java/Java基础/✅你知道fastjson的反序列化漏洞吗 30f3673e113881aea0bafa9f1be819ae.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# Java序列化

> 把 JVM 堆中的对象状态转换为字节序列（序列化），以及把字节序列还原为对象（反序列化）——使对象能跨时间、跨进程存在。

## 第一性原理

JVM 进程结束，堆中所有对象消失。序列化的存在理由：**把对象的状态从内存"导出"到可持久化/可传输的字节流，在需要时再还原**。这是持久化、网络传输、进程间通信的基础能力。

## 核心机制

### 原生 Java 序列化

1. 实现 `java.io.Serializable`（标记接口，无方法）
2. 用 `ObjectOutputStream.writeObject(obj)` 序列化
3. 用 `ObjectInputStream.readObject()` 反序列化
4. 底层：反射读取所有非 `static`、非 `transient` 字段，按固定格式写入二进制流

### serialVersionUID

```java
private static final long serialVersionUID = 1L;
```

- 序列化时写入 UID，反序列化时校验是否匹配
- **不定义**：JVM 根据类结构自动生成——增加字段后 UID 变化，旧数据无法反序列化（`InvalidClassException`）
- **定义**：手动控制版本兼容，新增字段有默认值，不破坏旧数据

### 关键规则

| 规则 | 说明 |
|------|------|
| `static` 字段 | 不序列化（属于类，不属于对象实例）|
| `transient` 字段 | 不序列化（用于标记敏感字段如密码）|
| 父类 | 父类也必须实现 `Serializable`，否则父类字段不被序列化 |
| 接口 | `Serializable` 是标记接口，实现它即可，无需实现方法 |

### 序列化框架对比

| 框架 | 是否需要 Serializable | 特点 |
|------|----------------------|------|
| Java 原生 | 是 | 二进制，性能一般，安全风险高 |
| JSON（Jackson/Gson）| 否 | 可读性好，跨语言 |
| Protobuf | 否（需定义 `.proto`）| 性能最好，强类型 |
| Hessian | 否 | RPC 常用，比原生快 |

## 关键权衡

1. **安全风险**：反序列化时执行类的构造逻辑，恶意字节流可触发任意代码（fastjson、log4shell 均是反序列化漏洞）。生产中应**白名单限制**可反序列化的类，或改用 JSON/Protobuf
2. **版本演进**：不定义 serialVersionUID 导致类变更后数据不兼容；字段改名会丢失旧数据
3. **性能**：原生序列化较慢；RPC 框架普遍换用 Protobuf/Hessian
4. **不序列化敏感字段**：密码、Token 用 `transient` 修饰

## 与其他概念的关系

- 依赖 [[机制-反射]]：序列化库通过反射读写对象字段
- 关联 [[概念-Java异常体系]]：`ObjectOutputStream.writeObject()` 抛 `IOException`（Checked），强制调用方处理 IO 失败
- 支撑了 L5 Redis：Java 对象存入 Redis 需序列化（默认 JDK 序列化或 JSON）
- 支撑了 L7 RPC：Dubbo、gRPC 等框架的参数传递依赖序列化协议选择

## 应用边界

**适合用原生序列化**：JVM 内部缓存持久化；对象深拷贝（序列化再反序列化）。

**不适合用原生序列化**：跨语言通信（用 Protobuf/JSON）；对外暴露的接口（有安全风险）；高性能 RPC（太慢）。
