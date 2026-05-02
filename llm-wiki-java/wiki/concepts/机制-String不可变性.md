---
type: concept
status: active
name: "String不可变性"
layer: L1
aliases: ["String immutability", "String常量池", "String intern"]
related:
  - "[[概念-包装类与自动拆装箱]]"
sources:
  - "../../raw/note/Hollis/Java基础/✅String是如何实现不可变的？ 30f3673e1138812e8805f6e2d5cdba43.md"
  - "../../raw/note/Hollis/Java基础/✅String为什么设计成不可变的？ 30f3673e113881709953c4e6b3b1870b.md"
  - "../../raw/note/Hollis/Java基础/✅String、StringBuilder和StringBuffer的区别？ 30f3673e113881c8b9f0c4968a9a27e6.md"
  - "../../raw/note/Hollis/Java基础/✅String中intern的原理是什么？ 30f3673e11388153aabdd7e79baec27a.md"
  - "../../raw/note/Hollis/Java基础/✅JDK 9中对字符串的拼接做了什么优化？ 30f3673e113881a897e4d7ab3959a8ee.md"
  - "../../raw/note/Hollis/Java基础/✅为什么JDK 9中把String的char[]改成了byte[]？ 30f3673e1138811fab40e9175519b311.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# String不可变性

> String 一旦创建其内容就不可修改——所有"修改"操作都返回新对象，使 String 天然线程安全且可共享。

## 第一性原理

字符串是程序中使用最频繁的对象。如果可变，则：
1. 不能安全共享（多线程修改）
2. 不能用作 HashMap key（修改后 hashCode 变化，找不回）
3. 不能放入常量池（别人可以改掉你的引用）

不可变性解决：**以"每次修改创建新对象"为代价，换取线程安全、可哈希、可缓存**。

## 核心机制

### 不可变的三重保证（JDK 源码层面）

```java
public final class String {              // 1. final class，禁止继承和方法覆盖
    private final byte[] value;          // 2. final byte[]，数组引用不可重指向
    // 3. 无任何公开方法修改 value 内容
    //    substring/concat 等都是 new String(...)
}
```

> JDK 8 之前是 `char[]`，JDK 9 改为 `byte[]` + `coder` 字段（Latin-1 单字节 / UTF-16 双字节），节省内存。

### 常量池（String Pool）

```java
String a = "hello";  // 存入常量池
String b = "hello";  // 直接复用常量池对象
a == b;              // true（同一对象）

String c = new String("hello");  // 堆中新建
a == c;                           // false
c.intern();                       // 将 c 放入常量池（或返回池中已有对象）
```

### 字符串拼接演化

| 场景 | 编译器行为 | 性能 |
|------|-----------|------|
| `"a" + "b"`（字面量）| 编译期合并为 `"ab"` | 最优 |
| 循环外 `s1 + s2` | JDK 8：生成 `StringBuilder.append()`；JDK 9+：`invokedynamic` → `StringConcatFactory` | 好 |
| 循环内反复 `+=` | 每次创建新 StringBuilder + 新 String | 差，应手动用 `StringBuilder` |

### String vs StringBuilder vs StringBuffer

| | String | StringBuilder | StringBuffer |
|--|--------|--------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 是（不可变）| 否 | 是（synchronized）|
| 性能 | 拼接慢 | 快 | 慢于 SB |
| 使用场景 | 常量、key | 单线程拼接 | 多线程拼接（罕见）|

## 关键权衡

1. **内存**：大量字符串拼接（循环中 `+=`）产生大量临时对象，GC 压力大 → 用 StringBuilder
2. **`intern()` 滥用**：把大量动态字符串放入常量池（PermGen/Metaspace）可能导致内存泄漏
3. **char vs byte**：JDK 9 的 byte[] 优化对全 ASCII 字符串节省 50% 内存，对中文（UTF-16）无变化

## 与其他概念的关系

- 共享了 [[概念-包装类与自动拆装箱]] 的设计思路：两者都是不可变类，都有缓存机制（String Pool vs Integer Cache）
- 在 L2 JVM 层：String 常量池位于堆（JDK 7+），GC 可回收；JDK 6 在 PermGen，OOM 风险
- 在 L5 Redis：作为 key 时 String 的 `hashCode` 稳定性至关重要

## 应用边界

**用 String**：常量、配置值、HashMap/HashSet 的 key、跨方法传递的字符串。

**用 StringBuilder**：单线程字符串构建（日志拼接、SQL 拼接、报文组装）。

**避免**：在高频路径中用 `String +=` 拼接；滥用 `intern()` 把动态字符串强塞常量池。
