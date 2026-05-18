---
type: concept
status: active
name: "Java类型"
layer: L1
aliases: ["String不可变性", "String immutability", "String常量池", "包装类", "Autoboxing", "Unboxing", "Integer Cache", "自动装箱", "BigDecimal"]
related:
  - "[[机制-泛型]]"
  - "[[机制-Lambda]]"
  - "[[概念-JMM]]"
  - "[[机制-HashMap]]"
sources:
  - "../../../raw/note/Hollis/Java基础/"
updated: 2026-05-18
---

# Java 类型

> String、包装类、BigDecimal 是 Java 值语义设计的三大核心：共享安全、对象化兼容、精确计算。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、类型设计的核心矛盾](#一类型设计的核心矛盾) | 性能、可用性与语义安全平衡 |
| [二、String 不可变机制](#二string-不可变机制) | 常量池、hash 稳定性、线程安全 |
| [三、包装类与装箱拆箱](#三包装类与装箱拆箱) | 对象化需求与缓存机制 |
| [四、BigDecimal 精度模型](#四bigdecimal-精度模型) | 十进制精确运算规则 |
| [五、编码选型原则](#五编码选型原则) | String/Builder、基本类型/包装类选择 |
| [六、对比与结论](#六对比与结论) | 常见类型方案对比 |
| [七、生产风险与排查](#七生产风险与排查) | 拆箱 NPE、`==` 陷阱、精度事故 |
| [八、关系与边界](#八关系与边界) | 与泛型、JMM、集合关系 |
| [九、面试速答口径](#九面试速答口径) | 高频问答 |

## 一、类型设计的核心矛盾

Java 类型系统在“性能、正确性、可用性”之间做了多处工程权衡：

- String 用不可变换共享安全
- 包装类让基本类型进入对象世界
- BigDecimal 用十进制换取精度确定性

## 二、String 不可变机制

| 机制 | 作用 |
| --- | --- |
| `final class` | 禁止继承篡改语义 |
| 内部值不可变引用 | 保证状态只读 |
| 修改返回新对象 | 保证共享安全 |

衍生收益：

- 适合作为 HashMap key
- 可安全进入 String 常量池复用
- 多线程读共享无需额外同步

## 三、包装类与装箱拆箱

### 3.1 自动装箱拆箱

```java
Integer i = 128; // valueOf
int n = i;       // intValue
```

### 3.2 Integer 缓存陷阱

`Integer.valueOf` 默认缓存 `-128~127`，超出范围创建新对象。

结论：包装类比较值统一使用 `equals`。

### 3.3 拆箱 NPE 风险

`Integer x = null; int y = x;` 会触发空指针。

## 四、BigDecimal 精度模型

```java
BigDecimal a = new BigDecimal("0.1"); // 正确
BigDecimal b = new BigDecimal(0.1);   // 不推荐
```

关键规则：

- 构造优先字符串
- 比较优先 `compareTo`
- `divide` 指定精度和舍入模式

## 五、编码选型原则

1. 循环拼接字符串用 `StringBuilder`。
2. 集合元素、可空字段用包装类；纯计算链路优先基本类型。
3. 金额统一 `BigDecimal`，并在领域层收敛 scale/rounding 规则。
4. 对外接口避免暴露模糊数值语义（如字符串金额无单位）。

## 六、对比与结论

| 主题 | 方案 A | 方案 B | 结论 |
| --- | --- | --- | --- |
| 字符串拼接 | `+`（循环内） | `StringBuilder` | 热路径选 `StringBuilder` |
| 数值比较 | `BigDecimal.equals` | `compareTo` | 语义比较用 `compareTo` |
| 对象比较 | `==` | `equals` | 值比较用 `equals` |

## 七、生产风险与排查

| 风险 | 现象 | 处理 |
| --- | --- | --- |
| 包装类 `==` 误判 | 线上条件分支异常 | 全量替换为 `equals` |
| 自动拆箱 NPE | 看似无空判断却 NPE | 可空字段禁用直接拆箱 |
| BigDecimal 精度漂移 | 金额对账不一致 | 统一构造、统一舍入策略 |
| `intern` 滥用 | 内存压力升高 | 限制动态字符串驻留 |

## 八、关系与边界

- 与 [[机制-泛型]]：泛型容器依赖包装类对象化。
- 与 [[概念-JMM]]：String 不可变对象天然线程安全。
- 与 [[机制-HashMap]]：String 稳定 hash 支撑 key 语义。

边界：这些类型保证语义安全，但不能替代业务校验规则。

## 九、面试速答口径

| 问题 | 关键答案 |
| --- | --- |
| String 为什么不可变 | 为共享安全、常量池复用、稳定 hash |
| Integer 为什么别用 `==` | 缓存区间导致引用比较不稳定 |
| BigDecimal 为什么用字符串构造 | 避免二进制浮点转十进制误差 |
