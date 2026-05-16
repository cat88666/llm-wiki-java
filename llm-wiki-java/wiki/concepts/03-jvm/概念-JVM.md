---
type: concept
status: active
name: "JVM"
layer: L2
aliases: ["JVM", "Java虚拟机", "HotSpot", "JVM生命周期", "对象生命周期", "类生命周期", "类初始化", "程序计数器", "虚拟机栈", "本地方法栈", "堆", "方法区", "元空间", "GC Roots", "jstat", "jmap", "jstack", "JVM调优", "JVM内存模型", "堆分代", "TLAB", "堆外内存", "垃圾收集器", "GC", "CMS", "G1", "ZGC", "标记清除", "标记整理", "复制算法", "可达性分析", "三色标记", "STW", "强引用", "软引用", "弱引用", "虚引用", "FullGC", "OOM", "双亲委派", "类加载器", "ClassLoader", "Parents Delegation"]
related:
  - "[[概念-JMM]]"
  - "[[机制-JIT编译]]"
  - "[[机制-动态代理]]"
  - "[[机制-SPI]]"
  - "[[概念-IO模型]]"
  - "[[概念-ThreadLocal]]"
sources:
  - "../../../raw/note/Hollis/JVM/"
updated: 2026-05-16
---

# JVM

> JVM 是 Java 平台无关性的核心运行时：负责字节码加载与执行（类加载）、内存自动管理（GC）、运行期编译优化（JIT）；核心矛盾是吞吐量、低延迟、内存开销三者的权衡。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 平台无关、自动内存管理、运行期优化 |
| [二、运行时内存区域](#二运行时内存区域) | 五大区域、堆分代、对象晋升、对象创建、TLAB、堆外内存 |
| [三、GC 算法与收集器](#三gc-算法与收集器) | 可达性分析、三种算法、三色标记、引用类型、收集器选型 |
| [四、类加载机制](#四类加载机制) | 类生命周期7阶段、三种加载器、双亲委派、破坏场景 |
| [五、执行引擎（JIT）](#五执行引擎jit) | 热点编译、逃逸分析、预热 |
| [六、JVM 调优与排查](#六jvm-调优与排查) | 方法论、基线、FullGC/YoungGC/STW/OOM 排查、jstat、参数速查 |
| [七、综合对比](#七综合对比) | 堆vs栈、永久代vs元空间、对象分配方式、概念依赖关系 |
| [八、生产风险](#八生产风险) | 内存风险、类加载风险、OOM类型、调优工具 |
| [九、应用边界](#九应用边界) | 收集器决策、自定义ClassLoader、面试速答 |

---

## 一、第一性原理

JVM 解决三个根本问题：

1. **平台无关**：将 `.java` 编译为字节码 `.class`，由 JVM 在不同 OS 上统一解释/编译执行——"Write Once, Run Anywhere"
2. **自动内存管理**：开发者不需要手动 malloc/free，GC 自动识别和回收不可达对象
3. **运行期优化**：JIT 在程序运行后，基于实际热点数据做激进优化，峰值性能可超越静态编译

这三件事的代价是引入了 GC 停顿、预热延迟和内存开销，JVM 调优的本质就是在这些代价之间做取舍。

---

## 二、运行时内存区域

### 五大运行时区域

| 区域 | 线程归属 | 存什么 | 异常 |
|------|---------|--------|------|
| **Java 堆** | 共享 | 几乎所有对象实例、数组 | `OutOfMemoryError: Java heap space` |
| **方法区/元空间** | 共享 | 类信息、常量池、静态变量、JIT 编译后的代码（CodeCache） | `OutOfMemoryError: Metaspace` |
| **运行时常量池** | 共享（方法区的一部分）| 字面量 + 符号引用（类名、字段名、方法名）| `OutOfMemoryError` |
| **虚拟机栈** | 私有 | 每次方法调用创建一个栈帧（局部变量表、操作数栈、动态链接、返回地址）| `StackOverflowError` / `OutOfMemoryError` |
| **程序计数器** | 私有 | 当前线程正在执行的字节码指令地址；执行 Native 方法时为空 | 无（唯一不会 OOM 的区域）|

**本地方法栈**：功能与虚拟机栈相同，专服务 native 方法；HotSpot 中与虚拟机栈合二为一。

**虚拟机栈帧包含**：局部变量表（基本类型/对象引用/returnAddress，long/double 占 2 slot）、操作数栈、动态链接、方法返回地址。

### 堆分代设计

```
Java 堆
├── 新生代（Young Generation）  默认占堆 1/3
│   ├── Eden 区                新对象首先在此分配
│   ├── Survivor From（S0）    GC 存活对象暂存区
│   └── Survivor To（S1）      复制算法的另一侧
│       Eden : S0 : S1 = 8 : 1 : 1（默认）
└── 老年代（Old Generation）   默认占堆 2/3
    存放长期存活对象、大对象、晋升对象
```

**Survivor 两区设计原因**：Young GC 用复制算法，需要两块区域交替；若只有一块，复制无处落脚且产生碎片。

### 对象晋升老年代规则

满足任一条即晋升：

| 规则 | 说明 | 相关参数 |
|------|------|---------|
| **年龄 ≥ 15** | 每次 Young GC 存活 +1 岁，达阈值晋升 | `-XX:MaxTenuringThreshold`（默认 15）|
| **动态年龄判断** | Survivor 中年龄 ≤ N 的对象总大小 > Survivor 空间 50%，则年龄 ≥ N 的全部晋升 | — |
| **大对象直接进老年代** | 超过阈值的大对象跳过新生代 | `-XX:PretenureSizeThreshold`（默认 0，即不预晋升）|

### 对象创建流程

| 步骤 | 说明 |
|------|------|
| 1. 类加载检查 | 检查 new 指令参数能否在常量池定位到类符号引用，且该类已被加载 |
| 2. 分配内存 | 堆规整用指针碰撞（Serial/ParNew），堆不规整用空闲列表（CMS）|
| 3. 初始化零值 | 将分配到的内存空间初始化为零值（保证字段不赋初值即可使用）|
| 4. 设置对象头 | 填入哈希码、GC 分代年龄、锁状态标志、类元数据指针等 |
| 5. 执行 `<init>` | 执行构造方法，完成程序员定义的初始化 |

### TLAB（Thread Local Allocation Buffer）

多线程并发分配对象时，指针碰撞存在竞争问题：

- JVM 为每个线程在 Eden 区预分配一小块**私有缓冲区**（TLAB）
- 线程分配对象时优先在自己的 TLAB 中分配，**无需加锁**
- TLAB 用完后再申请新的 TLAB（此时才需要同步）
- 默认开启 `-XX:+UseTLAB`；大幅减少堆内存分配时的锁竞争

### 方法区的演变

| JDK 版本 | 实现 | 存储位置 | 限制参数 |
|---------|------|---------|---------|
| JDK 7 及以前 | 永久代（PermGen） | JVM 堆内 | `-XX:MaxPermSize` |
| **JDK 8+** | **元空间（Metaspace）** | **本地内存（堆外）** | `-XX:MaxMetaspaceSize` |

改用元空间原因：永久代固定上限易 `PermGen OOM`；本地内存由 OS 管理，上限更灵活；同时为 JRockit 与 HotSpot 合并铺路。

### 堆外内存（直接内存）

`DirectByteBuffer`（NIO）分配的内存不在 JVM 堆内，不受 GC 直接管理：

| 维度 | 说明 |
|------|------|
| 优势 | 减少 Java 堆与 OS 内存的复制开销（零拷贝）；适合 I/O 密集场景 |
| 缺点 | 不受 GC 直接管理，需依赖 `Cleaner` 机制或手动释放；泄漏排查困难 |
| 限制参数 | `-XX:MaxDirectMemorySize`（默认等于 `-Xmx`）|

---

## 三、GC 算法与收集器

### 可达性分析与 GC Roots

**引用计数法**：每个对象有引用计数器，归零即可回收。缺陷：无法解决循环引用，HotSpot 不用此法。

**可达性分析（主流）**：从 GC Roots 出发遍历引用链，不可达对象视为死亡。

GC Roots 包括：虚拟机栈中局部变量引用的对象、方法区中类静态变量引用的对象、常量池引用的对象、本地方法栈 JNI 引用的对象、活跃线程对象、`synchronized` 持有的对象、Remembered Set。

### 三种基础 GC 算法

| 算法 | 步骤 | 优点 | 缺点 | 适用代 |
|------|------|------|------|--------|
| **标记-清除** | 标记存活对象 → 清除死亡对象 | 速度快 | 内存碎片 | 老年代（CMS）|
| **标记-复制** | 存活对象复制到另一半内存 | 无碎片 | 浪费空间（一半）| 新生代（Survivor 来回）|
| **标记-整理** | 标记 → 移动存活对象到一端 | 无碎片无浪费 | 移动对象耗时 | 老年代（G1/ZGC）|

### 三色标记法（CMS/G1 基础）

并发标记期间应用线程持续修改引用关系，三色标记维护扫描进度：

- **白色**：未被访问（初始状态，扫描结束后白色 = 垃圾）
- **灰色**：已访问但子引用未全部扫描（扫描中间状态）
- **黑色**：已访问且子引用已扫描完（存活对象）

并发标记时引用关系会变化，导致漏标问题的解决方案：
- **CMS**：**增量更新**——重新标记阶段扫描并发标记期间发生变化的引用
- **G1**：**原始快照（SATB）**——并发标记开始时对堆做快照，按快照中的引用关系扫描

### 四种引用类型

| 引用类型 | 创建方式 | GC 回收时机 | 获取对象 | 典型用途 |
|---------|---------|------------|---------|---------|
| **强引用** | `Object o = new Object()` | 可达时不回收 | 直接访问 | 普通变量引用 |
| **软引用** | `new SoftReference<>(obj)` | OOM 前回收 | `.get()` | 内存敏感缓存（Guava Cache softValues）|
| **弱引用** | `new WeakReference<>(obj)` | 每次 GC 必回收 | `.get()` | `WeakHashMap`、ThreadLocal key |
| **虚引用** | `new PhantomReference<>(obj, queue)` | GC 时 | 始终 null | 配合 ReferenceQueue 做回收后资源清理 |

**ThreadLocal 弱引用陷阱**：key 是弱引用，value 是强引用；线程池复用时 key 被 GC 回收但 value 泄漏，必须在 `finally` 中 `remove()`。

### GC 触发条件

- **Young GC（Minor GC）**：Eden 满
- **Full GC**：老年代空间不足、元空间满、`System.gc()`（不保证执行）、晋升失败、堆外内存满等

### 收集器选型

| 特性 | CMS | G1 | ZGC | Shenandoah |
|------|-----|----|----|------------|
| **JDK 版本** | ≤ 1.8（JDK14 删除）| 1.7+（1.9+ 默认）| JDK15+（JDK21 支持分代）| JDK12+ |
| **设计目标** | 低延迟 | 可预测停顿 + 兼顾吞吐 | 亚毫秒级停顿 | 亚毫秒级停顿 |
| **回收范围** | 仅老年代 | 整堆（Region 化）| 整堆 | 整堆 |
| **GC 算法** | 标记-清除 | 年轻代复制 + 老年代整理 | 并发标记-整理 | 并发标记-整理 |
| **内存碎片** | 有 | 无 | 无 | 无 |
| **STW 可预测** | 否 | 是 | 极低 | 极低 |
| **停顿量级** | 短（并发）| 可控（目标停顿）| 极短（< 1ms）| 极短 |
| **吞吐量** | 一般 | 高 | 略低 | 略低 |
| **适用场景** | JDK8 遗留系统 | 大堆（4G+）通用首选 | 超大堆、极低延迟 | 与 ZGC 类似，Red Hat 维护 |

**选型口诀**：通用 → G1；超低延迟（< 1ms）且堆 8G+ → ZGC；老项目维持 → ParallelGC。

---

## 四、类加载机制

### 类的生命周期（7 个阶段）

```
加载（Loading）
  → 验证（Verification）
  → 准备（Preparation）    ← 链接
  → 解析（Resolution）     ←
  → 初始化（Initialization）
  → 使用（Using）
  → 卸载（Unloading）
```

| 阶段 | 做什么 | 关键细节 |
|------|--------|---------|
| 加载 | 通过类名找到 .class 文件，读入字节数组，创建 `Class` 对象 | 触发时机：首次主动使用（new/静态方法调用等）|
| 验证 | 字节码合法性检查（文件格式、元数据、字节码语义）| 防止恶意字节码破坏 JVM |
| 准备 | 为类的**静态变量**分配内存，赋**默认值**（int = 0，引用 = null）| `static int x = 10` 此时 x = 0，不是 10 |
| 解析 | 将符号引用（类名字符串）替换为直接引用（内存地址）| 可延迟到运行时（lazy resolution）|
| 初始化 | 执行 `<clinit>`（静态代码块 + 静态变量赋值），**真正赋业务初值** | `static int x = 10` 此时 x = 10；父类先于子类 |
| 使用 | 创建实例、调用方法、访问字段 | 正常业务阶段 |
| 卸载 | 该类所有实例被 GC + ClassLoader 被 GC → FullGC 时从方法区回收 | JDK 自带类永不卸载 |

**主动触发初始化**：new 创建实例、读写静态变量、调用静态方法、`Class.forName()` 反射、初始化子类前先初始化父类、JVM 启动时加载 main 类。

**不触发初始化的被动引用**：通过子类访问父类静态字段（只初始化父类）、定义某类的数组、访问编译期常量。

### 三种类加载器（JDK 8）

```
Bootstrap ClassLoader（启动类加载器）   ← C++ 实现，Java 中表示为 null
    加载：JRE/lib/rt.jar（Object, String 等核心类）
    ↓ 子
Extension ClassLoader（扩展类加载器）
    加载：JRE/lib/ext/ 下的类
    ↓ 子
Application ClassLoader（应用类加载器）
    加载：Classpath 下的类（业务代码）
```

> JDK 9 后模块化，Extension ClassLoader 改名为 Platform ClassLoader，Bootstrap 从 rt.jar 整体加载改为按模块按需加载。

### 双亲委派模型

加载请求自下而上委派，加载动作自上而下尝试：

```java
// ClassLoader.loadClass 核心逻辑
protected Class<?> loadClass(String name, boolean resolve) {
    Class<?> c = findLoadedClass(name);        // 1. 先查缓存
    if (c == null) {
        if (parent != null)
            c = parent.loadClass(name, false); // 2. 委派父加载器
        else
            c = bootstrapClassLoader.loadClass(name);
        if (c == null)
            c = findClass(name);               // 3. 父找不到，自己加载
    }
    return c;
}
```

**价值**：保证 `Object` 等核心类唯一性，防止用户伪造 `java.lang.Object`；避免重复加载。

**扩展点选择**：

| 扩展方式 | 行为 | 是否破坏双亲委派 |
|---------|------|-----------------|
| 重写 `findClass()` | 自定义加载逻辑（如从网络/加密文件读取字节码）| 不破坏（推荐）|
| 重写 `loadClass()` | 改变委派顺序或跳过委派 | 破坏 |

### 线程上下文类加载器

`Thread.currentThread().getContextClassLoader()` 提供"反向委派"通道：父加载器加载的代码可以通过当前线程获取子加载器，从而加载子加载器路径上的类——SPI 机制的基础。

### 破坏双亲委派的三种典型场景

| 场景 | 原因 | 实现方式 |
|------|------|---------|
| **SPI（如 JDBC）** | Bootstrap 加载 `java.sql.Driver` 接口，但实现类在 Classpath 由 App ClassLoader 加载，父加载器无法"向下"找子类 | 线程上下文类加载器 |
| **Tomcat** | 多个 webapp 需要隔离，各自有独立的 WebApp ClassLoader | 每个 webapp 自己的 ClassLoader 优先加载 |
| **热部署（OSGi）** | 模块化，需要不同模块加载不同版本类 | 网状加载，而非树形委派 |

### JDK 8 vs JDK 9+ 类加载器变化

| 对比维度 | JDK 8 | JDK 9+ |
|---------|-------|--------|
| 第二层加载器 | Extension ClassLoader | Platform ClassLoader |
| 加载路径 | JRE/lib/ext/ 目录 | 模块系统中的平台模块 |
| Bootstrap 加载范围 | rt.jar 整体加载 | 按模块按需加载（java.base 等）|
| 类路径可见性 | Classpath 全局可见 | 模块间需 exports/opens 声明 |

---

## 五、执行引擎（JIT）

- **混合执行**：字节码先解释执行，热点代码 JIT 编译为机器码缓存
- **热点探测**：方法调用计数器 + 回边计数器；Server 模式默认 10000 次方法调用触发 JIT 编译
- **逃逸分析**：无逃逸对象 → 栈上分配（不触发 GC）+ 锁消除 + 标量替换
- **预热问题**：应用刚启动 JIT 未完成，解释执行慢，RT 偏高 → 发布时用流量预热

---

## 六、JVM 调优与排查

### 调优方法论（五步）

```
1. 确定基线（GC 频率、耗时、堆使用率）
2. 发现异常（与基线对比，确认是否越过告警阈值）
3. 工具定位（jstat/jmap/jstack/Arthas）
4. 参数调整或代码修复
5. 验证效果（通过压测/监控验证）
```

**核心原则：先修代码问题，再调参数**。参数只能缓解，内存泄漏和对象 churn 要靠代码修复。

### 正常基线（4C8G 机器、4G 堆、G1 收集器）

| 指标 | 正常值 | 告警阈值 |
|------|--------|---------|
| YoungGC 频率 | 100 次/分钟 | > 500 次/分钟 |
| YoungGC 耗时 | ~20ms | > 100ms |
| FullGC 频率 | 日常 ≤ 1次/周；大促 ≤ 1次/2小时 | > 1次/小时 |
| FullGC 耗时 | 400~700ms | > 1s |
| 堆使用率 | < 50% | > 80% |

### 排查流程

生产排查先采集证据，不要一上来就调参数：

1. `jstat -gc <pid> 1000 10` — 看 GC 次数/耗时/Eden/Survivor/Old 使用变化
2. `jmap -histo:live <pid>` — 看堆内对象分布，定位最大占用类型
3. `jstack <pid>` — 看线程状态，定位 BLOCKED/WAITING/死锁/高 CPU 线程
4. `jinfo -flags <pid>` — 查看 JVM 参数是否符合预期
5. Arthas `dashboard`/`trace`/`watch` — 线上诊断，无需停服

### 频繁 FullGC 排查

```bash
jstat -gcutil <pid> 1000 10           # 看 FGC 列计数和 FGCT 耗时
jmap -histo:live <pid> | head -30     # 找占用内存最多的对象类型
jmap -dump:format=b,file=/tmp/heap.bin <pid>  # MAT/VisualVM 分析 GC Root 引用链
```

常见根因：老年代大量长生命周期对象（缓存/大对象滞留）、静态集合无限增长（`static List/Map`）、对象分配速率超过 GC 回收速度、堆设置过小。

### YoungGC 过于频繁

**原因**：Eden 太小，对象分配速率高，Eden 很快填满触发 YoungGC。

**处理**：增大新生代（`-XX:NewRatio=2` 或 `-Xmn2g`）；减少短生命周期对象创建，关键路径避免频繁 `new`。

### STW 暂停过长

**G1 调优**：`-XX:MaxGCPauseMillis=200`（设定目标停顿时间，G1 自动调整 Region 数量）

**ZGC 适用**：堆 > 8G、极低延迟场景；停顿通常 < 1ms，与堆大小弱相关；代价是更高并发 GC 线程 CPU 开销。

### OOM 排查

| OOM 类型 | 原因 | 处理 |
|---------|------|------|
| `Java heap space` | 堆内存耗尽 / 内存泄漏 | `jmap -dump` 分析，找 retained size 最大对象和 GC Root 引用链 |
| `GC overhead limit exceeded` | GC 时间 > 98% 但回收 < 2% | 堆内存不足或内存泄漏，增大堆或修复泄漏 |
| `Metaspace` | 类加载过多（动态代理/CGLIB/JSP 热加载）| 增大 `-XX:MaxMetaspaceSize`，排查 ClassLoader 是否被 GC Root 引用无法回收 |
| `Direct buffer memory` | Netty/NIO 堆外内存泄漏 | 检查 `ByteBuf` 是否调用 `release()`；设置 `-XX:MaxDirectMemorySize` |
| `unable to create new native thread` | 线程数超系统限制 | 减少线程数，检查线程泄漏 |

生产建议开启：`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump`

### jstat 估算调优

```bash
jstat -gc <pid> 1000 10
```

| 指标 | 方法 |
|------|------|
| 年轻代对象增长速率 | 观察 EU（Eden used）在间隔内的增长 |
| Young GC 触发频率 | 根据 Eden 大小和增长速率估算多久打满 |
| Young GC 平均耗时 | `YGCT / YGC` |
| 每次 Young GC 后存活量 | 观察 Survivor 和 Old 在 GC 后的增长 |
| 老年代增长速率 | 多次 Young GC 后 Old 使用量增量 |
| Full GC 平均耗时 | `FGCT / FGC` |

核心目标：让短命对象在 Young GC 中被回收，减少晋升老年代，降低 Full GC 频率。

### 常用 JVM 参数速查

```bash
# 堆内存
-Xms4g -Xmx4g                # 初始/最大堆（设置相同避免动态扩缩开销）
-Xmn2g                        # 新生代大小
-Xss512k                      # 每线程栈大小
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m   # 元空间
-XX:MaxDirectMemorySize=256m  # 堆外内存（NIO 重度使用时需关注）

# GC 选择
-XX:+UseG1GC                  # G1（JDK 9+ 默认）
-XX:+UseZGC                   # ZGC（JDK 15+ 推荐）
-XX:MaxGCPauseMillis=200      # G1 目标停顿时间

# G1 专用参数
-XX:G1HeapRegionSize=16m      # Region 大小（1~32MB，取 2 的幂）
-XX:G1NewSizePercent=20       # 新生代最小比例
-XX:G1MaxNewSizePercent=60    # 新生代最大比例

# 故障诊断
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-XX:OnOutOfMemoryError="kill -9 %p"   # OOM 时强制退出
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=20m  # GC 日志（JDK 9+）
```

---

## 七、综合对比

### 堆 vs 栈

| 维度 | 堆 | 栈 |
|------|----|----|
| 线程归属 | 共享 | 私有 |
| 存储内容 | 对象实例 | 局部变量、方法帧 |
| 大小 | 大（GB 级）| 小（默认 512K-1M/线程）|
| 分配速度 | 较慢（需 GC 管理）| 极快（指针移动）|
| 回收方式 | GC 回收 | 方法返回时栈帧自动弹出 |
| 并发安全 | 需同步 | 天然线程安全 |

### 永久代 vs 元空间

| 维度 | 永久代（JDK 7-）| 元空间（JDK 8+）|
|------|-----------------|-----------------|
| 存储位置 | JVM 堆内 | 本地内存（堆外）|
| 大小限制 | `-XX:MaxPermSize` 固定上限 | `-XX:MaxMetaspaceSize`（默认无上限）|
| GC 行为 | 受 Full GC 管理 | 有独立的元空间 GC |
| 典型问题 | `PermGen space OOM` | 不设上限可能耗尽系统内存 |

### 对象内存分配方式

| 方式 | 适用条件 | 原理 |
|------|---------|------|
| **指针碰撞（Bump the Pointer）** | 堆内存规整（Serial、ParNew 等带压缩的 GC）| 维护一个分界指针，分配时向空闲方向移动指针 |
| **空闲列表（Free List）** | 堆内存不规整（CMS 等不带压缩的 GC）| 维护空闲块链表，找到足够大的块进行分配 |

### 概念依赖关系

```
JVM 内存区域（堆分代结构）
    ↓ 决定
GC 算法选择（新生代复制，老年代整理/清除）
    ↓ 实现为
垃圾收集器（CMS/G1/ZGC）
         ↑ 受影响
引用类型（软/弱/虚引用改变可达性分析结果）

类加载机制（双亲委派加载类到方法区）
    ↓ Class 对象
反射（运行期读取类结构）
    ↓
动态代理 / SPI（L1）

JIT 编译（热点代码编译机器码）
    ↓ 逃逸分析
栈上分配（减少 GC 对象数量）
```

### JVM 与上下层关系

| 层 | 依赖 JVM 体系的方式 |
|----|-------------------|
| L1 语言基础 | Lambda 的 `invokedynamic` 依赖 JVM 指令集；反射依赖 Class 对象（由类加载产生）|
| L3 并发 | JMM 是 JVM 内存模型的并发视角；`volatile`/`synchronized` 语义由 JVM 保证 |
| L4 集合框架 | `HashMap` 扩容创建大量对象，触发 Young GC；理解 GC 才能分析集合框架性能 |
| L5 存储 | JDBC 驱动加载依赖 SPI 机制（破坏双亲委派） |

---

## 八、生产风险

### 内存与 GC 风险

| 风险 | 表现 | 原因 | 应对 |
|------|------|------|------|
| **堆 OOM** | `OutOfMemoryError: Java heap space` | 内存泄漏 / 大对象 / 堆设置过小 | `jmap -dump` 分析，修复泄漏或调大 `-Xmx` |
| **元空间 OOM** | `OutOfMemoryError: Metaspace` | 类加载器泄漏、动态代理大量生成类 | 设置 `-XX:MaxMetaspaceSize` + 排查 ClassLoader 泄漏 |
| **栈溢出** | `StackOverflowError` | 递归过深 / 方法调用链过长 | 检查递归逻辑，调大 `-Xss` |
| **直接内存 OOM** | `OutOfMemoryError: Direct buffer memory` | NIO DirectByteBuffer 未释放 | 设置 `-XX:MaxDirectMemorySize`，确保 ByteBuffer 及时释放 |
| **堆内存抖动** | Full GC 频繁、RT 毛刺 | `-Xms` != `-Xmx`，堆不断扩缩 | 设置 `-Xms` = `-Xmx` |
| **NMT 内存不可见** | top 显示 RSS 远大于 -Xmx | 堆外内存（Direct/MMap/JNI）不在 JVM 堆统计中 | 开启 NMT（`-XX:NativeMemoryTracking=summary`）排查 |

### 类加载风险

| 风险 | 现象 | 根因 | 应对 |
|------|------|------|------|
| ClassCastException | 同名类之间转换报错 | 不同 ClassLoader 加载同一 .class → JVM 视为不同类型 | 统一类加载器；避免跨 ClassLoader 传递对象 |
| 元空间 OOM（类加载型）| `Metaspace OOM` 且持续增长 | 动态代理/JSP 编译/Groovy 脚本频繁生成类，ClassLoader 未被 GC | 排查 ClassLoader 是否被 GC Root 引用；限制 `-XX:MaxMetaspaceSize` |
| 类加载死锁 | 应用启动卡死 | 两个类加载器互相触发对方的 loadClass，形成锁循环 | JDK 7+ 已改为并行加载；自定义加载器注册 `registerAsParallelCapable` |
| 热部署类泄漏 | 每次热部署后 Metaspace 持续增长 | 旧 ClassLoader 未被回收（被 ThreadLocal/静态字段等 GC Root 引用）| 热部署前清理 ThreadLocal；避免 static 字段持有跨 ClassLoader 对象 |
| SPI 加载失败 | `ServiceConfigurationError` | 线程上下文类加载器未正确设置 | 确认 `Thread.setContextClassLoader` 指向正确加载器 |

### 调优工具速查

| 工具 | 用途 | 典型命令 |
|------|------|---------|
| `jps` | 查看 Java 进程 | `jps -v` |
| `jstat` | 监控 GC 频率/堆使用 | `jstat -gcutil <pid> 1000` |
| `jmap` | 对象分布 / Heap Dump | `jmap -histo:live <pid>` / `jmap -dump:format=b,file=heap.bin <pid>` |
| `jstack` | 线程快照 | `jstack <pid>` 查死锁/阻塞 |
| `jinfo` | 查看 JVM 参数 | `jinfo -flags <pid>` |
| **Arthas** | 线上诊断（首选）| `dashboard`/`trace`/`watch`/`jad`；无侵入，不停服 |
| `VisualVM/MAT` | 图形化分析 Heap Dump | 分析 retained size 和 GC Root 引用链 |

---

## 九、应用边界

### 收集器选型决策

- JDK 8 遗留系统且短期不升级 → CMS
- 通用首选，JDK 9+，大堆（4G+），需要可预测停顿 → G1
- 超低延迟（< 1ms），超大堆（8G+），JDK 15+ → ZGC

**G1 vs ZGC**：G1 通用稳定（JDK9+ 默认），ZGC 亚毫秒停顿但内存开销大约 15%；游戏/金融对延迟敏感且有富余内存选 ZGC，资源受限或堆 < 8G 仍选 G1。

### 何时需要自定义 ClassLoader

| 场景 | 是否需要 | 说明 |
|------|---------|------|
| 热部署 | 需要 | 重新加载修改后的类，需新建 ClassLoader 替换旧实例 |
| 加密 class 文件 | 需要 | 在 `findClass` 中加入解密逻辑 |
| 多版本类库隔离 | 需要 | 如 Tomcat 多 webapp 各自加载不同版本依赖 |
| 普通业务开发 | 不需要 | 默认三层加载器已满足需求 |
| 单体 Spring Boot 应用 | 不需要 | DevTools 热重启已内置处理 |

### JVM 体系不解决的问题

- 分布式状态共享——JVM 是单进程视角，跨进程需 Redis/数据库
- 硬实时（Hard Real-Time）保证——GC STW 无法完全消除（即使 ZGC 也有亚毫秒级停顿）
- 内存布局精确控制——Java 无法控制对象在内存中的精确位置

### 面试速答

| 问题 | 速答 |
|------|------|
| JVM 内存区域有哪些 | 堆、方法区/元空间、虚拟机栈、本地方法栈、程序计数器 |
| 唯一不会 OOM 的区域 | 程序计数器 |
| 类生命周期 | 加载、链接（验证/准备/解析）、初始化、使用、卸载 |
| 双亲委派作用 | 防止核心类被篡改，避免重复加载，保证类唯一性 |
| GC Roots 有哪些 | 栈中引用、静态变量、常量池引用、JNI 引用、活动线程等 |
| 三色标记法 | 白=未访问/灰=访问中/黑=已完成；CMS 增量更新，G1 原始快照（SATB） |
| 对象晋升老年代条件 | 年龄≥15、动态年龄判断（Survivor 50%）、大对象直接晋升 |
| Full GC 频繁怎么排查 | `jstat` 看频率和耗时，`jmap`/dump 看对象，判断晋升过快或泄漏 |
| JVM 调优核心目标 | 让短命对象在年轻代回收，减少老年代增长和 Full GC |
| 元空间为什么要设置上限 | 默认无上限，动态扩容会触发 Full GC，生产建议显式设置合理值 |
| G1 vs ZGC | G1 通用稳定（JDK9+默认），ZGC 亚毫秒停顿但内存开销更大（推荐堆 8G+）|
