---
type: summary
status: active
name: "JVM体系"
layer: L2
topics:
  - 运行时内存区域
  - 堆分代与GC
  - 垃圾收集器选型
  - 类加载与双亲委派
  - JIT编译与优化
  - 引用类型
sources:
  - "../../raw/note/Hollis/JVM/"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# 主题：JVM体系

> L2 运行时层。JVM 是 Java 平台无关性的核心，也是理解性能调优、内存问题、GC 调参的基础。依赖 L1 语言基础，支撑 L3 并发（JMM）、L4 集合框架（HashMap 分析）、L5 存储（连接池管理）。

---

## 知识地图

```
JVM
│
├── 内存结构
│   └── [[机制-JVM内存模型]]
│       ├── 运行时 5 大区域：堆/方法区/虚拟机栈/本地方法栈/程序计数器
│       └── 堆分代：Eden + S0 + S1（新生代） + Old（老年代）
│
├── 垃圾回收
│   ├── [[机制-GC算法与垃圾收集器]]
│   │   ├── 存活判断：可达性分析（GC Roots），引用计数法的缺陷
│   │   ├── 三大算法：标记-清除 / 标记-复制 / 标记-整理
│   │   └── 收集器：CMS → G1（JDK9 默认）→ ZGC（JDK15，亚毫秒）
│   └── [[概念-引用类型]]
│       └── 强 > 软 > 弱 > 虚；ThreadLocal 弱引用陷阱
│
├── 类加载
│   └── [[机制-类加载机制]]
│       ├── 生命周期：加载→验证→准备→解析→初始化→使用→卸载
│       ├── 双亲委派：Bootstrap→Extension→Application 三层委派
│       └── 破坏场景：SPI（JDBC）、Tomcat 隔离、OSGi 热部署
│
└── 执行引擎
    └── [[机制-JIT编译]]
        ├── 混合执行模式：解释器 + JIT 热点编译
        ├── 热点探测：方法调用计数器 + 回边计数器
        └── 优化：方法内联、逃逸分析、栈上分配、锁消除
```

---

## 核心高频考点

### 1. JVM 内存区域（高频 ⭐⭐⭐）

- **5 大区域**：堆（共享，GC 主战场）、方法区（共享，元数据）、虚拟机栈（私有，方法帧）、本地方法栈（私有，native）、程序计数器（私有，唯一不会 OOM）
- **堆分代**：Eden:S0:S1 = 8:1:1；新生代:老年代 = 1:2（默认）
- **方法区演变**：JDK7 永久代（堆内）→ JDK8 元空间（本地内存，无固定上限）
- **OOM 定位**：heap space → 堆溢出；Metaspace → 类加载器泄漏；StackOverflow → 递归过深

### 2. GC 算法（高频 ⭐⭐⭐）

- **三种算法**：标记-清除（快但碎片）→ 复制（无碎片但浪费一半）→ 标记-整理（无碎片无浪费但慢）
- **分代对应**：新生代用复制（Survivor 来回）；老年代用标记-整理（G1/ZGC）或标记-清除（CMS）
- **可达性分析**：从 GC Roots 遍历引用链，不可达 = 垃圾；引用计数法无法解决循环引用

### 3. 垃圾收集器选型（高频 ⭐⭐⭐）

- **CMS（已淘汰）**：老年代+标记-清除，有碎片，STW 不可预测，JDK14 删除
- **G1（主流）**：整堆+Region 化，`-XX:MaxGCPauseMillis` 控制 STW 上限，JDK9+ 默认
- **ZGC（低延迟）**：亚毫秒级停顿，染色指针+读屏障，适合 TB 级堆，JDK15+
- **选型口诀**：通用 → G1；超低延迟（< 1ms）→ ZGC；老项目维持 → ParallelGC

### 4. 双亲委派（高频 ⭐⭐⭐）

- **流程**：收到加载请求 → 查缓存 → 委派父加载器 → 父找不到才自己加载
- **作用**：保证 `Object` 等核心类唯一性，防止用户伪造 `java.lang.Object`
- **破坏方式**：重写 `loadClass()`（破坏）vs 重写 `findClass()`（不破坏）
- **三大破坏场景**：SPI（线程上下文 ClassLoader）、Tomcat（WebApp 隔离）、OSGi（模块化）

### 5. JIT 与预热（中频 ⭐⭐）

- **混合执行**：字节码先解释执行，热点代码 JIT 编译为机器码缓存
- **热点阈值**：Server 模式默认 10000 次方法调用触发
- **逃逸分析**：无逃逸对象 → 栈上分配（不触发 GC）+ 锁消除 + 标量替换
- **预热问题**：应用刚启动 JIT 未完成，解释执行慢 → 发布时用流量预热

### 6. 引用类型（中频 ⭐⭐）

- 软引用：OOM 前回收，适合缓存（Guava Cache softValues）
- 弱引用：每次 GC 回收，`WeakHashMap`、`ThreadLocal` Key
- **ThreadLocal 陷阱**：Key 弱引用 Value 强引用 → 线程池复用时 Value 泄漏 → 必须 `finally` 中 `remove()`
- 虚引用：不影响生命周期，配合 ReferenceQueue 做回收后资源清理

---

## 概念依赖关系

```
JVM内存模型（堆分代结构）
    ↓ 决定
GC算法选择（新生代复制，老年代整理/清除）
    ↓ 实现为
垃圾收集器（CMS/G1/ZGC）
         ↑ 受影响
引用类型（软/弱/虚引用改变可达性分析结果）

类加载机制（双亲委派加载类到方法区）
    ↓ Class 对象
反射（运行期读取类结构）
    ↓
动态代理 / SPI（L1）

JIT编译（热点代码编译机器码）
    ↓ 逃逸分析
栈上分配（减少 GC 对象数量）
```

---

## 与上下层的关系

| 层 | 依赖 JVM 体系的方式 |
|----|-------------------|
| L1 语言基础 | Lambda 的 `invokedynamic` 依赖 JVM 指令集；反射依赖 Class 对象（由类加载产生）|
| L3 并发 | JMM（Java Memory Model）是 JVM 内存模型的并发视角；`volatile`/`synchronized` 语义由 JVM 保证 |
| L4 集合框架 | `HashMap` 扩容创建大量对象，触发 Young GC；理解 GC 才能分析集合框架性能 |
| L5 存储 | 数据库连接池（JDBC）依赖 SPI 机制（破坏双亲委派）加载驱动 |

---

## 待摄入深化

- JMM（Java Memory Model）：happens-before 规则，`volatile` 的内存语义 → 与 L3 并发合并
- 字符串常量池：String.intern() 的行为变化（JDK6 vs JDK7+）
- 对象内存布局：对象头（Mark Word + Klass Pointer）+ 实例数据 + 对齐填充
- TLAB（Thread Local Allocation Buffer）：对象分配的线程安全保证
- Safe Point：GC STW 的触发点，所有线程需要到达 Safe Point 才能开始 GC

---

## 来源

- `raw/note/Hollis/JVM/`（57 个问答式笔记，2026-05-02 摄入）
