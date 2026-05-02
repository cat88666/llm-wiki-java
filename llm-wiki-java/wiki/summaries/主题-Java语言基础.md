---
type: summary
status: active
name: "Java语言基础"
layer: L1
topics:
  - 类型系统
  - OOP
  - 反射与动态代理
  - 泛型
  - 异常
  - 序列化
  - IO模型
  - 语言特性
sources:
  - "../../raw/note/📚 Hollis Java/Java基础/"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# 主题：Java语言基础

> L1 地基层。Java 语言机制的核心概念聚合，是 JVM（L2）、并发（L3）、集合框架（L4）所有上层知识的基础。

---

## 知识地图

```
Java 语言基础
│
├── 类型系统
│   ├── [[概念-包装类与自动拆装箱]]     基本类型 ↔ 对象，缓存陷阱，NPE 风险
│   ├── [[机制-String不可变性]]         不可变 + 常量池 + StringBuilder 选择
│   └── [[机制-泛型类型擦除]]           编译期安全 + 运行期擦除，PECS 原则
│
├── OOP
│   ├── [[概念-OOP三大特征]]            封装/继承/多态，接口 vs 抽象类
│   └── [[机制-Lambda表达式]]           函数式扩展，invokedynamic，Stream API
│
├── 运行时能力
│   ├── [[机制-反射]]                   运行期读类结构，慢但灵活
│   ├── [[机制-动态代理]]               JDK（接口）vs CGLIB（继承），AOP 基础
│   └── [[机制-SPI]]                   框架扩展点机制，ServiceLoader
│
├── 错误处理
│   └── [[概念-Java异常体系]]           Checked vs Unchecked，勿用异常控流
│
├── 持久化
│   └── [[机制-Java序列化]]             Serializable，serialVersionUID，安全风险
│
└── IO
    └── [[概念-IO模型]]                 BIO/NIO/AIO，同步/异步/阻塞/非阻塞
```

---

## 核心高频考点

### 1. 反射与动态代理（高频 ⭐⭐⭐）

- 反射慢的根本原因：装拆箱 + 方法遍历 + 跳过 JIT
- JDK 代理必须有接口；CGLIB 通过继承，不能代理 final 类
- Spring AOP 默认选择策略：有接口→JDK，无接口→CGLIB

### 2. 泛型（高频 ⭐⭐⭐）

- 类型擦除：编译期检查，运行期变 Object
- `List<String>` 不是 `List<Object>` 的子类（非协变）
- PECS：Producer `extends`，Consumer `super`
- `List<?>`：只读，不能写

### 3. String（高频 ⭐⭐⭐）

- final class + final byte[] + 无修改方法 = 不可变
- `==` 比较引用，`.equals()` 比较值
- 常量池：`"a" == "a"` 为 true；`new String("a") == "a"` 为 false
- JDK 9：`char[]` 改 `byte[]`，节省 Latin-1 字符串 50% 内存

### 4. 包装类（高频 ⭐⭐）

- Integer 缓存 -128 ~ 127，超出范围 `==` 为 false
- 包装类可为 null，拆箱时空指针
- RPC 返回值用包装类（区分 0 和 null）
- 金额用 `BigDecimal(String)`，比较用 `compareTo`

### 5. 异常（高频 ⭐⭐）

- Error（系统）vs Exception（程序）
- Checked（编译强制）vs Unchecked/RuntimeException（运行期）
- finally 必执行（System.exit 除外）；finally return 覆盖 try return
- 不用异常控制业务流程（性能 + 可读性双差）

### 6. 序列化（中频 ⭐⭐）

- 必须实现 `Serializable`；`transient` 字段不序列化；`static` 字段不序列化
- serialVersionUID：不定义→类改动→反序列化失败
- 原生序列化安全风险；生产推荐 JSON/Protobuf

### 7. IO 模型（中频 ⭐⭐）

- BIO：一连接一线程，高并发不适用
- NIO：Selector 多路复用，Netty 基础，聊天/网关
- AIO：OS 回调，Linux 实现不成熟，Netty 放弃
- 同步/异步描述通知方式，阻塞/非阻塞描述等待行为

---

## 概念依赖关系

```
OOP三大特征
    ↓ 支撑
动态代理（多态 + 接口）
    ↓ 依赖
反射（invoke 底层）
    ↓ 支撑
SPI（forName + newInstance）

泛型类型擦除
    ↓ 约束
包装类（集合不存基本类型）

序列化
    ↓ 依赖
反射（字段读写）
```

---

## 待摄入深化

以下话题在原始笔记中有涉及，后续摄入时可合并进已有 concept 页或新建：

- `static` 关键字（修饰类/方法/字段/代码块，加载时机）→ 合并进 OOP 页
- 枚举实现原理（实际是 final class，线程安全单例）
- 注解原理（`@Retention`、APT 注解处理器）
- `deep copy` vs `shallow copy` → 与序列化关联
- 字符编码（char 存 Unicode，UTF-8/UTF-16 区别）

---

## 来源

- `raw/note/📚 Hollis Java/Java基础/`（60+ 问答式笔记，2026-05-02 摄入）
