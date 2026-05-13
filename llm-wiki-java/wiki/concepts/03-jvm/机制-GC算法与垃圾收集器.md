---
type: concept
status: active
name: "GC算法与垃圾收集器"
layer: L2
aliases: ["垃圾回收", "GC", "CMS", "G1", "ZGC", "标记清除", "标记整理", "复制算法", "可达性分析", "三色标记", "STW", "强引用", "软引用", "弱引用", "虚引用", "SoftReference", "WeakReference", "PhantomReference", "JVM调优实战", "FullGC", "OOM"]
related:
  - "[[机制-JVM内存模型]]"
  - "[[概念-ThreadLocal]]"
  - "[[机制-对象池技术]]"
sources:
  - "../../../raw/note/Hollis/JVM/✅什么是强引用、软引用、弱引用和虚引用？.md"
  - "../../../raw/note/Hollis/JVM/✅JVM有哪些垃圾回收算法？.md"
  - "../../../raw/note/Hollis/JVM/✅JVM如何判断对象是否存活？.md"
  - "../../../raw/note/Hollis/JVM/✅G1和CMS有什么区别？.md"
  - "../../../raw/note/Hollis/JVM/✅ZGC和CMS和G1的区别对比.md"
  - "../../../raw/note/Hollis/JVM/✅Java的堆是如何分代的？为什么分代？.md"
  - "../../../raw/note/Hollis/JVM/✅常见的JVM调优工具有哪些.md"
  - "../../../raw/note/Hollis/JVM/✅FullGC多久一次算正常？.md"
  - "../../../raw/note/Hollis/JVM/✅Java发生了OOM一定会导致JVM 退出吗？.md"
  - "../../../raw/note/Hollis/JVM/✅JVM 中一次完整的 GC 流程是怎样的？.md"
  - "../../../raw/note/Hollis/JVM/✅YoungGC和FullGC的触发条件是什么？.md"
  - "../../../raw/note/Hollis/JVM/✅Java 8 和 Java 11 的GC有什么区别？.md"
created: 2026-05-02
updated: 2026-05-13
lint_notes: ""
---

# GC算法与垃圾收集器

> GC（垃圾回收）= 判断对象存活（可达性分析）+ 回收死亡对象（三种基础算法）+ 具体回收器实现（CMS / G1 / ZGC）；核心矛盾是**吞吐量与低延迟（STW 时长）的权衡**。本文同时覆盖 GC 原理和 JVM 调优实战。

## 第一性原理

Java 堆由 JVM 自动管理，不自动回收则内存持续增长终至 OOM，全部暂停应用回收则 STW 过长影响响应时间。GC 设计的根本问题：**如何以尽可能低的停顿时间，回收尽可能多的垃圾，同时维持尽可能高的吞吐量**。

---

## 核心机制

### 第一步：判断对象是否存活

**引用计数法**：
- 每个对象有引用计数器，归零即可回收
- 缺陷：无法解决循环引用
- HotSpot 不用此法

**可达性分析（主流）**：
- 从 GC Roots 出发遍历引用链
- 不可达对象视为死亡

GC Roots 包括：系统类加载器加载的类、活跃线程的栈变量、JNI 引用、`synchronized` 持有的对象、Remembered Set。

### 第二步：三种基础 GC 算法

| 算法 | 步骤 | 优点 | 缺点 | 适用代 |
|------|------|------|------|--------|
| **标记-清除** | 标记存活对象 → 清除死亡对象 | 速度快 | 内存碎片 | 老年代（CMS）|
| **标记-复制** | 存活对象复制到另一半内存 | 无碎片 | 浪费空间 | 新生代 |
| **标记-整理** | 标记 → 移动存活对象到一端 | 无碎片 | 移动对象耗时 | 老年代（G1/Serial Old）|

### 第三步：垃圾收集器对比

| 特性 | CMS | G1 | ZGC |
|------|-----|----|----|
| **JDK 版本** | ≤1.8（JDK14 删除）| 1.7+（1.9+ 默认）| JDK15+（JDK21 支持分代）|
| **设计目标** | 低延迟 | 可预测停顿 + 兼顾吞吐 | 亚毫秒级停顿 |
| **回收范围** | 仅老年代 | 整堆（Region 化）| 整堆 |
| **GC 算法** | 标记-清除 | 年轻代复制 + 老年代整理 | 并发标记-整理 |
| **内存碎片** | 有 | 无 | 无 |
| **STW 可预测** | 否 | 是 | 极低 |
| **适用** | JDK8 遗留系统 | 通用首选 | 超低延迟大堆 |

### 三色标记法（CMS/G1 基础）

- **白色**：未被访问
- **灰色**：已访问但子引用未全部扫描
- **黑色**：已访问且子引用已扫描

并发标记时引用关系会变化：
- CMS 用**增量更新**
- G1 用**原始快照（SATB）**

### 四种引用类型与 GC 行为

| 引用类型 | 创建方式 | GC 回收时机 | 获取对象 | 典型用途 |
|---------|---------|------------|---------|---------|
| **强引用** | `Object o = new Object()` | 可达时不回收 | 直接访问 | 普通对象 |
| **软引用** | `new SoftReference<>(obj)` | OOM 前 | `.get()` | 内存敏感缓存 |
| **弱引用** | `new WeakReference<>(obj)` | 每次 GC 必回收 | `.get()` | `WeakHashMap`、ThreadLocal key |
| **虚引用** | `new PhantomReference<>(obj, queue)` | GC 时 | 始终 null | 资源清理通知 |

**ThreadLocal 弱引用陷阱**：key 是弱引用，value 是强引用；线程池复用时必须 `remove()`。

### GC 触发条件

- **Young GC（Minor GC）**：Eden 满
- **Full GC**：老年代空间不足、元空间满、`System.gc()`、晋升失败等

---

## JVM 调优实战

### 调优方法论

```
1. 确定基线
2. 发现异常
3. 工具定位
4. 参数调整或代码修复
5. 验证效果
```

### 正常基线（经验值）

以下以 4C8G 机器、4G 堆、G1 收集器为参考：

| 指标 | 正常值 | 告警阈值 |
|------|--------|---------|
| YoungGC 频率 | 100次/分钟 | > 500次/分钟 |
| YoungGC 耗时 | ~20ms | > 100ms |
| FullGC 频率 | 日常 ≤ 1次/周；大促 ≤ 1次/2小时 | > 1次/小时 |
| FullGC 耗时 | 400~700ms | > 1s |
| 堆使用率 | < 50% | > 80% |

### 常用调优工具

| 工具 | 用途 | 典型命令 |
|------|------|---------|
| `jps` | 查看 Java 进程 | `jps -v` |
| `jstat` | 监控 GC 频率/堆使用 | `jstat -gcutil <pid> 1000` |
| `jmap` | 对象分布 / Heap Dump | `jmap -histo:live <pid>` / `jmap -dump:format=b,file=heap.bin <pid>` |
| `jstack` | 线程快照 | `jstack <pid>` |
| `jhat` | 分析 heap dump（浏览器访问）| `jhat heap.bin` |
| `Arthas` | 线上诊断 | `dashboard` / `trace` / `watch` |
| `VisualVM` | 图形化监控 | GUI |
| `JProfiler/YourKit` | 商业分析工具 | 付费 |

**生产首选 Arthas**：无侵入、不停服，支持 `dashboard`（实时监控）、`trace`（方法调用链路耗时）、`watch`（方法入参/返回值）、`jad`（反编译）。

### 典型问题排查

#### 频繁 FullGC

```bash
# 1. 确认 GC 频率
jstat -gcutil <pid> 1000 10
# 看 FGC 列计数和 FGCT 耗时

# 2. 查堆内存分配
jmap -histo:live <pid> | head -30
# 找占用内存最多的对象类型

# 3. 生成 Heap Dump（内存快照）
jmap -dump:format=b,file=/tmp/heap.bin <pid>
# 用 VisualVM 或 MAT 分析 GC Roots 引用链
```

常见根因：
- 老年代大量长生命周期对象，如缓存或大对象滞留
- 静态集合累积，典型是 `static List/Map`
- 对象分配速率超过 GC 回收速度，需要减少对象创建或做对象复用
- 堆设置过小，需要结合业务负载评估 `-Xmx`

#### OOM（OutOfMemoryError）

| OOM 类型 | 原因 | 处理 |
|---------|------|------|
| `Java heap space` | 堆内存耗尽 | Heap Dump 分析 + 修复内存泄漏 |
| `GC overhead limit exceeded` | GC 时间 > 98% 但回收 < 2% | 堆内存不足或内存泄漏 |
| `Metaspace` | 类加载过多（动态代理/CGLIB）| 增大 `-XX:MaxMetaspaceSize` 或减少动态代理 |
| `Direct buffer memory` | Netty/NIO 堆外内存泄漏 | 检查 `ByteBuf` 是否调用 `release()` |
| `unable to create new native thread` | 线程数超系统限制 | 减少线程数，检查线程泄漏 |

**OOM 不一定导致 JVM 退出**：JVM 会把问题转成 `Error` 抛出；如果被 catch 住，进程可能继续运行。生产常配合 `-XX:OnOutOfMemoryError` 做强制退出或拉起。

#### YoungGC 过于频繁

原因：新生代（Eden）太小，对象分配速率高，Eden 很快填满触发 YoungGC。

处理：
- 增大新生代比例：`-XX:NewRatio=2`（老年代:新生代 = 2:1）
- 直接设置新生代大小：`-Xmn2g`
- 减少短生命周期对象创建，关键代码路径避免频繁 `new`

#### STW 暂停过长

**G1 常用参数**：
```
-XX:MaxGCPauseMillis=200    # 目标最大停顿时间（G1 会自动调整 Region 数量）
-XX:G1HeapRegionSize=16m    # Region 大小（1~32MB，取 2 的幂）
-XX:G1NewSizePercent=20     # 新生代最小比例
-XX:G1MaxNewSizePercent=60  # 新生代最大比例
```

**ZGC** 适合：
- 堆 > 8G
- 极低延迟场景
- 能接受更高 CPU 开销

补充：
- 停顿时间通常可做到 < 1ms，且与堆大小弱相关
- 代价是更高的并发 GC 线程 CPU 开销

### Java 8 → Java 11 GC 差异

- Java 8 默认 Parallel GC
- Java 11 默认 G1
- G1 的 Mixed GC 避免了 CMS 的严重碎片化问题

## GC 收集器选型

| 收集器 | JDK | 停顿 | 吞吐量 | 适用场景 |
|--------|-----|------|--------|---------|
| CMS | ≤ JDK 8 | 短（并发）| 一般 | 对延迟敏感，JDK 8 遗留系统 |
| G1 | JDK 9+ 默认 | 可控（目标停顿）| 高 | 大堆（4G+）通用场景 |
| ZGC | JDK 15+（生产）| 极短（< 1ms）| 略低 | 超大堆、极低延迟 |
| Shenandoah | JDK 12+ | 极短 | 略低 | 与 ZGC 类似，Red Hat 维护 |

**Java 8 → Java 11 GC 差异**：Java 8 默认 Parallel GC，Java 11 默认 G1；G1 的 Mixed GC 解决了 CMS 的碎片化问题（不再需要 Full GC 整理碎片）。

---

### 常用 JVM 参数速查

```bash
# 堆内存
-Xms4g -Xmx4g          # 初始/最大堆（设置相同避免动态扩缩开销）
-Xmn2g                 # 新生代大小
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m  # 元空间

# GC 选择
-XX:+UseG1GC           # G1（JDK 9+ 默认）
-XX:+UseZGC            # ZGC（JDK 15+ 推荐）
-XX:MaxGCPauseMillis=200  # G1 目标停顿时间

# OOM 时生成 Heap Dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.bin

# GC 日志（JDK 9+）
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=20m

# OOM 时执行命令（如重启进程）
-XX:OnOutOfMemoryError="kill -9 %p"
```

---

## 关键权衡

1. **STW vs 并发**：并发 GC 复杂，但停顿更短
2. **吞吐量 vs 延迟**：Parallel 追吞吐，CMS/G1/ZGC 追延迟
3. **碎片 vs 停顿**：CMS 快但有碎片；G1/ZGC 整理开销更大但无碎片
4. **G1 vs ZGC**：G1 通用稳定；ZGC 适合超低延迟
5. **参数调优 vs 代码优化**：参数只能缓解，内存泄漏和对象 churn 往往要靠代码修复

---

## 与其他概念的关系

- 依赖 [[机制-JVM内存模型]]：各类 OOM 的根因落在不同内存区域
- 与 [[概念-ThreadLocal]] 关联：弱引用 key / 强引用 value 导致泄漏
- 与 [[机制-对象池技术]] 关联：对象池常用于降低对象分配速率、减少 FullGC

---

## 应用边界

**选 CMS**：JDK8 遗留系统、短期不升级。

**选 G1**：通用首选，JDK 9+ 默认；堆 4G+；需要可预测停顿。

**选 ZGC**：超低延迟、超大堆、JDK15+。

**调优口诀**：
1. 先看 GC 日志和 `jstat`
2. 再看堆分布和 Dump
3. 先修代码问题，再调参数
