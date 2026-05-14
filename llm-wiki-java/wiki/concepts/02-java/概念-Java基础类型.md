---
type: concept
status: active
name: "Java基础类型"
layer: L1
aliases: ["String不可变性", "String immutability", "String常量池", "包装类", "Autoboxing", "Unboxing", "Integer Cache", "自动装箱", "BigDecimal"]
related:
  - "[[机制-泛型]]"
  - "[[机制-Lambda]]"
  - "[[概念-JMM]]"
  - "[[机制-HashMap]]"
sources:
  - "../../../raw/note/Hollis/Java基础/✅String是如何实现不可变的？.md"
  - "../../../raw/note/Hollis/Java基础/✅String为什么设计成不可变的？.md"
  - "../../../raw/note/Hollis/Java基础/✅String、StringBuilder和StringBuffer的区别？.md"
  - "../../../raw/note/Hollis/Java基础/✅String中intern的原理是什么？.md"
  - "../../../raw/note/Hollis/Java基础/✅JDK 9中对字符串的拼接做了什么优化？.md"
  - "../../../raw/note/Hollis/Java基础/✅为什么JDK 9中把String的char[]改成了byte[]？.md"
  - "../../../raw/note/Hollis/Java基础/✅Java中有了基本类型为什么还需要包装类？.md"
  - "../../../raw/note/Hollis/Java基础/✅RPC接口返回中，使用基本类型还是包装类？.md"
  - "../../../raw/note/Hollis/Java基础/✅BigDecimal和Long表示金额哪个更合适，怎么选择？.md"
  - "../../../raw/note/Hollis/Java基础/✅为什么不能用BigDecimal的equals方法做等值比较？.md"
  - "../../../raw/note/Hollis/Java基础/✅为什么不能用浮点数表示金额？.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# Java 基础值类型

> Java 中三类值语义对象 -- String、包装类、BigDecimal -- 共享"不可变 + 缓存"设计，分别解决字符串安全共享、基本类型对象化、精确十进制运算的根本需求。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | String 不可变动机、包装类存在理由、BigDecimal 精度问题 |
| [二、核心机制](#二核心机制) | 不可变三重保证、常量池、拆装箱、IEEE 754 缺陷 |
| [三、Java 核心使用](#三java-核心使用) | String 拼接选型、Integer 缓存陷阱、BigDecimal 正确用法 |
| [四、核心使用原则](#四核心使用原则) | 基本类型 vs 包装类选型、RPC 接口规约 |
| [五、综合对比](#五综合对比) | String/StringBuilder/StringBuffer、金额存储方案 |
| [六、生产风险](#六生产风险) | NPE 拆箱、intern 泄漏、BigDecimal 除法异常 |
| [七、与其他概念的关系](#七与其他概念的关系) | 泛型、JMM、HashMap |
| [八、应用边界](#八应用边界) | 各类型适用场景与禁区 |

## 一、第一性原理

**String 不可变**：字符串是最频繁使用的对象。可变则无法安全共享（多线程竞争）、无法用作 HashMap key（hashCode 变化）、无法放入常量池复用。以"修改即创建新对象"换取线程安全、可哈希、可缓存。

**包装类存在**：泛型擦除到 `Object`，`int/long` 等基本类型不继承 `Object`，无法放入 `List<Integer>` 或用作 Map key。包装类让基本类型具备对象身份，同时保留值语义。

**BigDecimal 存在**：IEEE 754 二进制浮点无法精确表示 `0.1`，金融计算误差不可接受。BigDecimal 用十进制任意精度算术解决精度问题。

## 二、核心机制

### 2.1 String 不可变性 -- 三重保证

```java
public final class String {              // final class：禁止继承，防止子类破坏不可变性
    private final byte[] value;          // final 引用：不可重指向
    // 无任何 public 方法修改 value 内容：substring/concat 等均返回新对象
}
```

- JDK 8：底层 `char[]`
- JDK 9+：`byte[]` + `coder` 字段（Latin-1 单字节 / UTF-16 双字节），纯 ASCII 字符串节省 50% 内存

### 2.2 String 常量池（StringTable）

```java
String a = "hello";                     // 存入 StringTable（堆中，JDK 7+）
String b = "hello";                     // 复用常量池对象
a == b;                                 // true

String c = new String("hello");         // 堆中新建对象
a == c;                                 // false
c.intern();                             // 返回常量池中已有对象（或将 c 放入）
```

**面试要点**：JDK 6 常量池在 PermGen，容量有限易 OOM；JDK 7+ 移至堆中，可被 GC 回收。

### 2.3 自动拆装箱

编译器语法糖，编译期展开：

```java
Integer i = 42;    // → Integer.valueOf(42)     装箱
int n = i;         // → i.intValue()            拆箱
```

### 2.4 Integer 缓存机制

`Integer.valueOf()` 对 `-128 ~ 127` 返回缓存对象，超出范围新建对象。

```java
Integer a = 127, b = 127;  a == b;   // true（缓存命中）
Integer c = 128, d = 128;  c == d;   // false（新建对象）
```

上界可通过 `-XX:AutoBoxCacheMax=<n>` 调整。**结论：Integer 比较值必须用 `equals()`，禁用 `==`。**

### 2.5 IEEE 754 精度缺陷

```java
0.1 + 0.2 == 0.30000000000000004   // 二进制分数无法精确表示十进制 0.1
```

## 三、Java 核心使用

### 3.1 字符串拼接选型

| 场景 | 行为 | 性能 |
|------|------|------|
| `"a" + "b"`（字面量）| 编译期常量折叠为 `"ab"` | 最优 |
| `s1 + s2`（变量）| JDK 8：`StringBuilder`；JDK 9+：`invokedynamic → StringConcatFactory` | 好 |
| 循环内 `+=` | 每次新建 StringBuilder + 新 String | 差，必须手动用 `StringBuilder` |

### 3.2 BigDecimal 正确用法

```java
// 构造：必须用 String，不能用 double
BigDecimal a = new BigDecimal("0.1");     // 正确
BigDecimal wrong = new BigDecimal(0.1);   // 错误：仍有精度问题

// 比较：用 compareTo，不用 equals（equals 还比较 scale：1.0 ≠ 1.00）
a.compareTo(new BigDecimal("0.1")) == 0;  // true

// 除法：必须指定精度和舍入模式
a.divide(b, 2, RoundingMode.HALF_UP);     // 不指定则无限循环小数时抛 ArithmeticException
```

## 四、核心使用原则

| 场景 | 选择 | 原因 |
|------|------|------|
| POJO 字段 | 包装类 | 区分 null（未赋值）和 0（有值） |
| 局部变量、方法内计算 | 基本类型 | 无 null 需求，避免装箱开销 |
| RPC 接口返回值 | 包装类 | 区分"值为 0"和"无返回 null" |
| RPC 必填参数 | 基本类型 | 防 null 传入导致下游 NPE |
| 泛型集合元素 | 包装类 | 泛型擦除要求 Object |
| 金额计算 | BigDecimal / Long（分） | 禁止 float/double |

## 五、综合对比

### 5.1 String / StringBuilder / StringBuffer

| 维度 | String | StringBuilder | StringBuffer |
|------|--------|--------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 是（不可变） | 否 | 是（synchronized） |
| 性能 | 拼接差 | 最快 | 略慢于 StringBuilder |
| 场景 | 常量、key | 单线程拼接 | 多线程拼接（实际罕见） |

### 5.2 金额存储方案

| 方案 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| BigDecimal | 字符串形式入库（DECIMAL 类型） | 直观，精度无损 | 运算性能略低 |
| Long（分/厘） | 金额 x 100 存整型 | 计算快，无精度问题 | 需统一换算约定 |

## 六、生产风险

| 风险 | 触发场景 | 后果 | 防御 |
|------|---------|------|------|
| 拆箱 NPE | `Integer i = null; int n = i;` | NullPointerException | 包装类使用前判空 |
| 三目运算符 NPE | `Boolean flag = null; flag ? 1 : 0` | 自动拆箱触发 NPE | 避免三目中使用可能为 null 的包装类 |
| `intern()` 内存泄漏 | 大量动态字符串放入常量池 | Metaspace 膨胀 | 仅对高频复用的有限集合使用 intern |
| `BigDecimal.divide()` 异常 | 不指定 RoundingMode | ArithmeticException | 除法必须指定 scale 和 RoundingMode |
| `BigDecimal.equals()` 误判 | `new BigDecimal("1.0").equals(new BigDecimal("1.00"))` | 返回 false | 用 `compareTo()` 做等值比较 |
| `==` 比较包装类 | `Integer(128) == Integer(128)` | 超出缓存范围返回 false | 一律用 `equals()` |

## 七、与其他概念的关系

- 被 [[机制-泛型]] 约束：泛型集合不能存基本类型，必须用包装类
- String 常量池位于 JVM 堆（JDK 7+），与 [[概念-JMM]] 中的内存区域划分相关
- `Integer.hashCode()` 稳定（不可变），适合作 [[机制-HashMap]] 的 key
- 拆装箱在 [[机制-Lambda]] 中有性能影响，IntStream 等原始类型流可避免

## 八、应用边界

| 类型 | 适用 | 禁区 |
|------|------|------|
| String | 常量、配置值、HashMap key、跨方法传递 | 循环内 `+=` 拼接 |
| StringBuilder | 单线程字符串构建（日志、SQL、报文） | 多线程共享（非线程安全） |
| 包装类 | POJO 字段、泛型集合、RPC 返回值 | 高频循环计算（频繁拆装箱） |
| 基本类型 | 局部变量、方法内计算、必填参数 | 需要 null 语义的场景 |
| BigDecimal | 金额/财务计算 | 用 double 构造；不指定 RoundingMode 的除法 |
