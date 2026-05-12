---
type: concept
status: active
name: "Java基础值类型"
layer: L1
aliases: ["String不可变性", "String immutability", "String常量池", "包装类", "Autoboxing", "Unboxing", "Integer Cache", "自动装箱", "BigDecimal"]
related:
  - "[[机制-泛型类型擦除]]"
  - "[[机制-Lambda表达式]]"
  - "[[概念-JMM]]"
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
updated: 2026-05-12
lint_notes: ""
---

# Java 基础值类型

> 涵盖 Java 中三类"值语义"对象的设计：**String**（不可变 + 常量池）、**包装类**（泛型桥梁 + 整数缓存 + 拆装箱陷阱）、**BigDecimal**（精确金额计算）。三者共享"不可变类 + 缓存"的设计思路。

## 第一性原理

**String 不可变**：字符串是程序中最频繁使用的对象。可变则无法安全共享（多线程）、无法用作 HashMap key（hashCode 变化）、无法放入常量池。以"修改创建新对象"换取线程安全、可哈希、可缓存。

**包装类存在**：Java 泛型和集合框架要求 `Object` 类型，`int/long` 等基本类型不继承 `Object`，无法放入 `List<Integer>` 或用作 Map key。包装类让基本类型具备对象身份，同时保留值语义。

**BigDecimal 存在**：浮点数（IEEE 754 二进制分数）无法精确表示 0.1，金融计算误差不可接受。BigDecimal 用十进制任意精度算术解决此问题。

## 核心机制

### 一、String 不可变性

**三重保证（JDK 源码）**：

```java
public final class String {              // final class：禁止继承，防止子类破坏不可变性
    private final byte[] value;          // final 引用：不可重指向
    // 无任何 public 方法修改 value 内容：substring/concat 等均返回新对象
}
```

> JDK 8：`char[]`；JDK 9+：`byte[]` + `coder`（Latin-1 单字节 / UTF-16 双字节），纯 ASCII 字符串节省 50% 内存。

**常量池**：

```java
String a = "hello";        // 存入 StringTable（堆，JDK 7+）
String b = "hello";        // 复用常量池对象
a == b;                    // true

String c = new String("hello");  // 堆中新建
a == c;                           // false
c.intern();                       // 返回常量池中已有对象（或将 c 放入）
```

**字符串拼接演化**：

| 场景 | 行为 | 性能 |
|------|------|------|
| `"a" + "b"`（字面量）| 编译期合并为 `"ab"` | 最优 |
| 代码中 `s1 + s2` | JDK 8：生成 `StringBuilder`；JDK 9+：`invokedynamic → StringConcatFactory` | 好 |
| 循环内 `+=` | 每次新建 StringBuilder + 新 String | 差，手动用 `StringBuilder` |

**String / StringBuilder / StringBuffer 对比**：

| | String | StringBuilder | StringBuffer |
|--|--------|--------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 是（不可变）| 否 | 是（synchronized）|
| 场景 | 常量、key | 单线程拼接 | 多线程拼接（罕见）|

### 二、包装类与自动拆装箱

**8 种基本类型与包装类**（关键差异：默认值）：

| 基本类型 | 包装类 | 基本默认值 | 包装默认值 |
|---------|--------|---------|---------|
| `int` | `Integer` | 0 | null |
| `long` | `Long` | 0L | null |
| `double` | `Double` | 0.0 | null |
| `boolean` | `Boolean` | false | null |

**自动拆装箱（编译器语法糖）**：

```java
Integer i = 42;    // → Integer.valueOf(42)
int n = i;         // → i.intValue()
```

**Integer 缓存陷阱（面试必考）**：

```java
Integer a = 127, b = 127;  a == b;   // true（-128~127 缓存同一对象）
Integer c = 128, d = 128;  c == d;   // false（超出缓存范围，新建对象）
// 结论：Integer 比较值必须用 equals()，禁用 ==
```

缓存范围：`-128 ~ 127`，上界可通过 `-XX:AutoBoxCacheMax=<n>` 调整。

**NPE 陷阱**：

```java
Integer i = null;
int n = i;               // NullPointerException（null.intValue()）

Boolean flag = null;
int k = flag ? 1 : 0;   // 三目运算符触发拆箱 → NPE
```

**RPC 接口选型**：返回值用包装类（区分"值为 0"和"无返回 null"）；必填参数用基本类型（防 null 传入）。

### 三、BigDecimal 精确计算

**为什么不能用 double 表示金额**：

```java
0.1 + 0.2 == 0.30000000000000004  // IEEE 754 二进制分数无法精确表示 0.1
```

**正确使用 BigDecimal**：

```java
// ✅ 用 String 构造
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");

// ❌ 用 double 构造仍有精度问题
BigDecimal wrong = new BigDecimal(0.1);

// ✅ 比较值相等用 compareTo（equals 还比较精度：1.0 ≠ 1.00）
a.add(b).compareTo(new BigDecimal("0.3")) == 0;  // true
```

**金额存储两种方案**：

| 方案 | 说明 | 优点 |
|------|------|------|
| BigDecimal | 字符串形式入库 | 直观，精度无损 |
| Long（分/厘）| 金额 × 100 存整型 | 计算更快，无精度问题 |

## 关键权衡

1. **String `intern()` 滥用**：把大量动态字符串放入常量池（Metaspace）可能内存泄漏
2. **装箱开销**：频繁拆装箱产生大量临时对象，高频循环内用基本类型避免
3. **包装类 null 语义**：POJO 字段应用包装类（区分 null），方法内计算用基本类型
4. **BigDecimal 除法必须指定精度**：`a.divide(b)` 无限循环小数时抛 `ArithmeticException`，必须指定 `RoundingMode`

## 与其他概念的关系

- 被 [[机制-泛型类型擦除]] 约束：泛型集合不能存基本类型 → 必须用包装类
- String 常量池在 L2 JVM 堆中（JDK 7+），可被 GC 回收（JDK 6 在 PermGen，OOM 风险）
- `Integer.hashCode()` 稳定（不可变）→ 适合作 [[机制-HashMap底层实现]] 的 key

## 应用边界

**String**：常量/配置值；HashMap/HashSet 的 key；跨方法传递的字符串；**禁止在循环中 `+=`**。

**StringBuilder**：单线程字符串构建（日志、SQL 拼接、报文组装）。

**包装类**：POJO 字段；泛型集合元素；RPC 接口返回值；可选参数。

**基本类型**：局部变量和方法内计算；不需要 null 语义的必填参数。

**BigDecimal**：金额/财务计算；用 `String` 构造；`compareTo()` 比较；`divide()` 指定 `RoundingMode`。
