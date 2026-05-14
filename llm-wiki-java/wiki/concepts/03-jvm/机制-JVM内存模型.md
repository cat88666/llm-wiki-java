---
type: concept
status: active
name: "JVM内存模型"
layer: L2
aliases: ["JVM运行时区域", "堆分代", "方法区", "元空间", "程序计数器", "虚拟机栈", "TLAB", "堆外内存"]
related:
  - "[[机制-垃圾收集器]]"
  - "[[机制-JIT编译]]"
  - "[[概念-IO模型]]"
sources:
  - "../../../raw/note/Hollis/JVM/✅JVM的运行时内存区域是怎样的？.md"
  - "../../../raw/note/Hollis/JVM/✅Java的堆是如何分代的？为什么分代？.md"
  - "../../../raw/note/Hollis/JVM/✅什么是方法区？是如何实现的？.md"
created: 2026-05-02
updated: 2026-05-14
lint_notes: ""
---

# JVM内存模型

> JVM 将运行时内存切分为 5 个区域：线程共享的堆（对象实例）+ 方法区（类元数据）、线程私有的虚拟机栈（方法帧）+ 本地方法栈 + 程序计数器；堆内部再按对象生命周期分代，以支持分代 GC。

**快速导航**：[一、第一性原理](#一第一性原理) | [二、核心机制](#二核心机制) | [三、Java 核心使用](#三java-核心使用) | [四、核心使用原则](#四核心使用原则) | [五、使用案例](#五使用案例) | [六、综合对比](#六综合对比) | [七、生产风险](#七生产风险) | [八、与其他概念的关系](#八与其他概念的关系) | [九、应用边界](#九应用边界)

---

## 一、第一性原理

不同数据的生命周期和访问模式差异巨大：对象在堆上存活从毫秒到永久；方法局部变量随调用结束即消失；类元数据加载后基本不变。JVM 分区的目的：**让每个区域只做一件事，从而为各区域选择最适合的分配/回收策略**。

分代假说进一步细化了堆的管理：大多数对象"朝生夕死"，少数对象长期存活——据此将堆分为新生代和老年代，各自采用最匹配的 GC 算法。

---

## 二、核心机制

### 2.1 五大运行时区域

| 区域 | 线程归属 | 存什么 | 异常 |
|------|---------|--------|------|
| **Java 堆** | 共享 | 几乎所有对象实例、数组 | `OutOfMemoryError` |
| **方法区** | 共享 | 类信息、常量、静态变量、JIT 编译后的代码（CodeCache） | `OutOfMemoryError` |
| **运行时常量池** | 共享（方法区的一部分） | 字面量 + 符号引用（类名、字段名、方法名） | `OutOfMemoryError` |
| **虚拟机栈** | 私有 | 每次方法调用创建一个栈帧（局部变量表、操作数栈、动态链接、返回地址） | `StackOverflowError` / `OutOfMemoryError` |
| **程序计数器** | 私有 | 当前线程正在执行的字节码指令地址 | 无（唯一不会 OOM 的区域） |

**本地方法栈**：功能与虚拟机栈相同，专服务 native 方法（C/C++ 实现），在 HotSpot 中与虚拟机栈合二为一。

### 2.2 堆分代设计

```
Java 堆
├── 新生代（Young Generation）  默认占堆 1/3
│   ├── Eden 区                 新对象首先在此分配
│   ├── Survivor From（S0）     GC 存活对象暂存区
│   └── Survivor To（S1）       复制算法的另一侧
│       Eden : S0 : S1 = 8 : 1 : 1（默认）
└── 老年代（Old Generation）    默认占堆 2/3
    存放长期存活对象
```

**Survivor 两区设计原因**：Young GC 用复制算法，需要两块区域交替——若只有一块，复制无处落脚，且会产生内存碎片。

### 2.3 对象晋升老年代规则

满足任一条即晋升：

| 规则 | 说明 | 相关参数 |
|------|------|---------|
| **年龄 >= 15** | 每次 Young GC 存活 +1 岁，达到阈值晋升 | `-XX:MaxTenuringThreshold`（默认 15） |
| **动态年龄判断** | Survivor 中年龄 <= N 的对象总大小 > Survivor 空间 50%，则年龄 >= N 的全部晋升 | — |
| **大对象直接进老年代** | 超过阈值的大对象跳过新生代 | `-XX:PretenureSizeThreshold`（默认 0，即不预晋升） |

### 2.4 对象创建流程

| 步骤 | 说明 |
|------|------|
| 1. 类加载检查 | 检查 new 指令的参数能否在常量池中定位到类的符号引用，且该类已被加载 |
| 2. 分配内存 | 根据堆是否规整选择分配方式（见下表） |
| 3. 初始化零值 | 将分配到的内存空间初始化为零值（保证字段不赋初值即可使用） |
| 4. 设置对象头 | 填入哈希码、GC 分代年龄、锁状态、类元数据指针等 |
| 5. 执行 `<init>` | 执行构造方法，完成程序员定义的初始化 |

**内存分配方式对比**：

| 方式 | 适用条件 | 原理 |
|------|---------|------|
| **指针碰撞（Bump the Pointer）** | 堆内存规整（如 Serial、ParNew 等带压缩的 GC） | 维护一个分界指针，分配时向空闲方向移动指针 |
| **空闲列表（Free List）** | 堆内存不规整（如 CMS 等不带压缩的 GC） | 维护空闲块链表，找到足够大的块进行分配 |

### 2.5 TLAB（Thread Local Allocation Buffer）

多线程并发分配对象时，指针碰撞存在竞争问题。TLAB 机制：

- JVM 为每个线程在 Eden 区预分配一小块私有缓冲区（TLAB）
- 线程分配对象时优先在自己的 TLAB 中分配，无需加锁
- TLAB 用完后再申请新的 TLAB（此时才需要同步）
- 默认开启：`-XX:+UseTLAB`
- TLAB 大小可调：`-XX:TLABSize=N`

效果：大幅减少堆内存分配时的锁竞争，提升多线程分配性能。

### 2.6 方法区的演变

| JDK 版本 | 方法区实现 | 位置 | 限制参数 |
|---------|-----------|------|---------|
| JDK 7 及以前 | 永久代（PermGen） | JVM 堆内 | `-XX:MaxPermSize` |
| JDK 8+ | **元空间（Metaspace）** | 使用**本地内存**（堆外） | `-XX:MaxMetaspaceSize` |

改用元空间原因：永久代固定上限易导致 `PermGen OOM`；本地内存由 OS 管理，上限更灵活；同时为 JRockit 与 HotSpot 合并铺路。

### 2.7 堆外内存（直接内存）

`DirectByteBuffer`（NIO）分配的内存不在 JVM 堆内，不受 GC 直接管理。

| 维度 | 说明 |
|------|------|
| 优势 | 减少 Java 堆与 OS 内存的复制开销（零拷贝）；适合 I/O 密集场景 |
| 缺点 | 不受 GC 直接管理，需依赖 `Cleaner` 机制或手动释放；泄漏排查困难 |
| 限制参数 | `-XX:MaxDirectMemorySize`（默认等于 `-Xmx`） |

---

## 三、Java 核心使用

### 3.1 JVM 常用内存参数

| 参数 | 作用 | 推荐/说明 |
|------|------|----------|
| `-Xms` | 初始堆大小 | 建议与 `-Xmx` 设为相同，避免堆动态扩缩带来的性能抖动 |
| `-Xmx` | 最大堆大小 | 根据应用实际需要设置，一般为物理内存的 60%-75% |
| `-Xss` | 每个线程的栈大小 | 默认 512K-1M，递归深的场景可适当调大 |
| `-Xmn` | 新生代大小 | 可替代 `-XX:NewRatio` 直接指定 |
| `-XX:NewRatio=N` | 老年代/新生代比值 | 默认 2（即 Old:Young = 2:1） |
| `-XX:SurvivorRatio=N` | Eden/Survivor 比值 | 默认 8（即 Eden:S0:S1 = 8:1:1） |
| `-XX:MaxMetaspaceSize` | 元空间上限 | 建议显式设置，避免无限增长耗尽系统内存 |
| `-XX:MaxDirectMemorySize` | 直接内存上限 | NIO 重度使用时需关注 |
| `-XX:+UseTLAB` | 开启 TLAB | 默认开启，通常无需修改 |

### 3.2 内存查看命令

```bash
# 查看堆内存分布
jmap -heap <pid>

# 生成堆转储
jmap -dump:format=b,file=heap.hprof <pid>

# 查看 GC 和内存统计
jstat -gc <pid> 1000

# 查看本地内存（NMT，需启动时加 -XX:NativeMemoryTracking=summary）
jcmd <pid> VM.native_memory summary
```

---

## 四、核心使用原则

1. **-Xms 等于 -Xmx**：避免堆动态扩缩带来 Full GC 和性能抖动
2. **显式设置 MaxMetaspaceSize**：防止类加载器泄漏导致元空间无限增长
3. **合理设置新生代比例**：对象存活率低的应用加大 Eden，减少 Young GC 复制开销
4. **注意堆外内存**：NIO 大量使用 DirectByteBuffer 时，`-XX:MaxDirectMemorySize` 必须配合监控
5. **优先栈上分配**：短生命周期对象控制在方法内，配合 [[机制-JIT编译]] 的逃逸分析实现栈上分配

---

## 五、使用案例

### 5.1 典型 Spring Boot 应用内存配置

```bash
java -Xms4g -Xmx4g \
     -Xmn1536m \
     -Xss512k \
     -XX:MaxMetaspaceSize=512m \
     -XX:MaxDirectMemorySize=256m \
     -jar app.jar
```

### 5.2 OOM 排查定位

1. `java.lang.OutOfMemoryError: Java heap space`
   - 堆内对象过多 → `jmap -dump` 导出堆转储 → MAT / VisualVM 分析大对象和 GC Root 引用链
2. `java.lang.OutOfMemoryError: Metaspace`
   - 元空间满 → 排查类加载器泄漏（动态代理、JSP 热加载、CGLIB 大量生成类）
3. `java.lang.StackOverflowError`
   - 递归过深 → 检查递归终止条件，必要时改为迭代或调大 `-Xss`

---

## 六、综合对比

### 6.1 堆 vs 栈

| 维度 | 堆 | 栈 |
|------|----|----|
| 线程归属 | 共享 | 私有 |
| 存储内容 | 对象实例 | 局部变量、方法帧 |
| 大小 | 大（GB 级） | 小（默认 512K-1M/线程） |
| 分配速度 | 较慢（需 GC 管理） | 极快（指针移动） |
| 回收方式 | GC 回收 | 方法返回时栈帧自动弹出 |
| 并发安全 | 需同步 | 天然线程安全 |

### 6.2 永久代 vs 元空间

| 维度 | 永久代（JDK 7-） | 元空间（JDK 8+） |
|------|-----------------|-----------------|
| 存储位置 | JVM 堆内 | 本地内存（堆外） |
| 大小限制 | `-XX:MaxPermSize` 固定上限 | `-XX:MaxMetaspaceSize`（默认无上限） |
| GC 行为 | 受 Full GC 管理 | 有独立的元空间 GC |
| 典型问题 | `PermGen space OOM` | 不设上限可能耗尽系统内存 |

### 6.3 对象分配方式对比

| 维度 | 指针碰撞 | 空闲列表 |
|------|---------|---------|
| 前提 | 堆内存规整 | 堆内存不规整 |
| 适配 GC | Serial、ParNew（带压缩） | CMS（不带压缩） |
| 性能 | 更快（仅移动指针） | 略慢（需遍历链表） |

---

## 七、生产风险

| 风险 | 表现 | 原因 | 应对 |
|------|------|------|------|
| **堆 OOM** | `OutOfMemoryError: Java heap space` | 内存泄漏 / 大对象 / 堆设置过小 | `jmap -dump` 分析 → 修复泄漏或调大 `-Xmx` |
| **元空间 OOM** | `OutOfMemoryError: Metaspace` | 类加载器泄漏、动态代理大量生成类 | 设置 `-XX:MaxMetaspaceSize` + 排查 ClassLoader 泄漏 |
| **栈溢出** | `StackOverflowError` | 递归过深 / 方法调用链过长 | 检查递归逻辑，调大 `-Xss` |
| **直接内存 OOM** | `OutOfMemoryError: Direct buffer memory` | NIO DirectByteBuffer 未释放 | 设置 `-XX:MaxDirectMemorySize`，确保 ByteBuffer 及时释放 |
| **堆内存抖动** | Full GC 频繁、RT 毛刺 | `-Xms` != `-Xmx`，堆不断扩缩 | 设置 `-Xms` = `-Xmx` |
| **TLAB 浪费** | Eden 空间利用率低 | TLAB 过大导致碎片 | 一般无需调整，极端场景可调 `-XX:TLABSize` |
| **NMT 内存不可见** | top 显示 RSS 远大于 -Xmx | 堆外内存（Direct/MMap/JNI）不在 JVM 堆统计中 | 开启 NMT（`-XX:NativeMemoryTracking=summary`）排查 |

---

## 八、与其他概念的关系

- 依赖 [[机制-垃圾收集器]]：分代设计决定了 Young GC 用复制算法、Old GC 用标记整理/标记清除
- 依赖 [[机制-类加载机制]]：类加载后的元数据存入方法区/元空间
- 与 [[机制-JIT编译]] 关联：JIT 编译的机器码缓存在方法区的 CodeCache；逃逸分析可实现栈上分配，减少堆压力
- 与 [[概念-IO模型]] 关联：NIO 零拷贝依赖堆外直接内存（DirectByteBuffer）
- TLAB 与并发分配：TLAB 是 Eden 区的线程私有缓冲，减少多线程分配时的锁竞争

---

## 九、应用边界

**必须深入理解内存模型的场景**：
- 高并发服务的 JVM 调优——堆大小、新生代比例、GC 选型都依赖对内存区域的理解
- OOM 排查——不同区域的 OOM 有不同的排查路径和工具
- 性能优化——理解 TLAB、栈上分配、对象晋升规则才能做出有效优化

**容易误解的边界**：
- "JVM 内存模型"不等于"Java 内存模型（JMM）"——前者是运行时区域划分，后者是并发编程的内存可见性规范
- 堆外内存不在 `-Xmx` 控制范围内——生产中 RSS 远大于 Xmx 往往是堆外内存导致
- 元空间默认无上限——不显式设置 `-XX:MaxMetaspaceSize` 可能导致系统内存被耗尽
