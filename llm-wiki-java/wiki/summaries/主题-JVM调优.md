---
type: summary
status: active
name: "JVM调优"
tags: ["#jvm", "#practice"]
related:
  - "[[机制-垃圾收集器]]"
  - "[[机制-JVM内存模型]]"
  - "[[概念-引用类型]]"
sources:
  - "../../raw/note/Hollis/JVM/✅常见的JVM调优工具有哪些.md"
  - "../../raw/note/Hollis/JVM/✅FullGC多久一次算正常？.md"
  - "../../raw/note/Hollis/JVM/✅Java发生了OOM一定会导致JVM 退出吗？.md"
  - "../../raw/note/Hollis/JVM/✅JVM 中一次完整的 GC 流程是怎样的？.md"
  - "../../raw/note/Hollis/JVM/✅YoungGC和FullGC的触发条件是什么？.md"
  - "../../raw/note/Hollis/JVM/✅ZGC和CMS和G1的区别对比.md"
  - "../../raw/note/Hollis/JVM/✅Java 8 和 Java 11 的GC有什么区别？.md"
created: 2026-05-08
updated: 2026-05-16
---

# JVM调优

> 线上 JVM 出现频繁 FullGC / OOM / 高延迟时，核心方法论是：**确定基线 → 发现异常 → 工具定位根因 → 制定方案 → 验证效果**。

| 章节 | 概述 |
| --- | --- |
| [一、调优方法论](#一调优方法论) | 五步闭环、确定基线 |
| [二、正常基线（经验值）](#二正常基线经验值) | 4C8G+G1 下各指标正常范围 |
| [三、常用调优工具](#三常用调优工具) | jstat/jmap/jstack/Arthas/VisualVM |
| [四、典型问题排查](#四典型问题排查) | 频繁FullGC、OOM、YoungGC频繁、STW过长 |
| [五、GC收集器选型](#五gc收集器选型) | CMS/G1/ZGC/Shenandoah 对比 |
| [六、常用JVM参数速查](#六常用jvm参数速查) | 堆内存/GC/OOM诊断/GC日志 |
| [七、常见问题](#七常见问题) | G1 vs ZGC时机、对象池reset测试 |
| [八、与其他概念的关系](#八与其他概念的关系) | 垃圾收集器、JVM内存模型、引用类型 |

---

## 一、调优方法论

```
1. 确定基线：正常指标是什么？（FullGC 频率、YoungGC 耗时、堆使用率）
2. 发现异常：监控/报警发现偏离基线
3. 工具定位：jstat/jmap/Arthas/VisualVM 确认根因
4. 制定方案：参数调整 or 代码修复
5. 验证效果：对比调优前后指标
```

---

## 二、正常基线（经验值）

以下以 4C8G 机器（JVM 堆 4G，G1 收集器）为参考：

| 指标 | 正常值 | 告警阈值 |
|------|--------|---------|
| YoungGC 频率 | 100次/分钟 | > 500次/分钟 |
| YoungGC 耗时 | ~20ms | > 100ms |
| FullGC 频率 | 日常 ≤ 1次/周；大促 ≤ 1次/2小时 | > 1次/小时 |
| FullGC 耗时 | 400~700ms | > 1s（STW 影响可用性）|
| 堆使用率 | < 50% | > 80% |

---

## 三、常用调优工具

| 工具 | 用途 | 典型命令 |
|------|------|---------|
| `jps` | 查看 Java 进程 PID | `jps -v` |
| `jstat` | 监控 GC 频率/堆使用 | `jstat -gcutil <pid> 1000` |
| `jmap` | 生成堆 Dump / 查看对象分布 | `jmap -histo:live <pid>` / `jmap -dump:format=b,file=heap.bin <pid>` |
| `jstack` | 生成线程快照（死锁/阻塞诊断）| `jstack <pid>` |
| `jhat` | 分析 heap dump（浏览器访问）| `jhat heap.bin` |
| `Arthas` | 阿里开源，线上诊断利器，支持热更新/方法追踪 | `watch`/`trace`/`dashboard` |
| `VisualVM` | 图形化监控内存/CPU/GC | GUI 工具 |
| `JProfiler/YourKit` | 商业级内存/CPU分析 | 付费工具 |

**生产首选 Arthas**：无侵入、不停服，支持 `dashboard`（实时监控）、`trace`（方法调用链路耗时）、`watch`（方法入参/返回值）、`jad`（反编译）。

---

## 四、典型问题排查

### 问题1：频繁 FullGC

**排查步骤**：
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

**常见根因**：
- 老年代存在大量生命周期很长的大对象 → 检查缓存是否有内存泄漏
- 代码中持有静态集合不断累积 → 检查 `static List/Map`
- 内存分配速率太高超过 GC 速度 → 减少对象创建（对象池/复用）
- 堆内存设置过小 → 适当扩大 `-Xmx`

### 问题2：OOM（OutOfMemoryError）

**分类处理**：

| OOM 类型 | 原因 | 处理 |
|---------|------|------|
| `Java heap space` | 堆内存耗尽 | Heap Dump 分析 + 修复内存泄漏 |
| `GC overhead limit exceeded` | GC 时间 > 98% 但回收 < 2% | 堆内存不足或内存泄漏 |
| `Metaspace` | 类加载过多（动态代理/CGLIB）| 增大 `-XX:MaxMetaspaceSize` 或减少动态代理 |
| `Direct buffer memory` | Netty/NIO 堆外内存泄漏 | 检查 `ByteBuf` 是否调用 `release()` |
| `unable to create new native thread` | 线程数超系统限制 | 减少线程数，检查线程泄漏 |

**OOM 不一定导致 JVM 退出**：JVM 注册了 SIGSEGV 信号处理函数，将 OOM 转换为 Error 抛出；若被 catch 住，程序可继续运行。但 `-XX:OnOutOfMemoryError="kill -9 %p"` 配置可强制在 OOM 时退出。

### 问题3：YoungGC 过于频繁

**原因**：新生代（Eden）太小，对象分配速率高，Eden 很快填满触发 YoungGC。

**处理**：
- 增大新生代比例：`-XX:NewRatio=2`（老年代:新生代 = 2:1）
- 直接设置新生代大小：`-Xmn2g`
- 减少短生命周期对象创建（关键代码路径避免频繁 new）

### 问题4：STW 暂停过长

**G1 调优参数**：
```
-XX:MaxGCPauseMillis=200    # 目标最大停顿时间（G1 会自动调整 Region 数量）
-XX:G1HeapRegionSize=16m    # Region 大小（1~32MB，取 2 的幂）
-XX:G1NewSizePercent=20     # 新生代最小比例
-XX:G1MaxNewSizePercent=60  # 新生代最大比例
```

**ZGC（JDK 15+ 推荐用于低延迟场景）**：
- 停顿时间 < 1ms（与堆大小无关）
- 代价：更高的 CPU 开销（并发 GC 线程）
- 适合：堆 > 8G、延迟敏感（金融交易、游戏）

### 实战案例：0 Full GC 调优（STAR）

**Situation**：游戏逻辑服高峰期 Full GC 3-5 次/天，每次 STW 1-3 秒，玩家掉线投诉。

**Action**：
1. **MAT 分析 Heap Dump**：RoomContext 对象峰值在 Old Gen 堆积 2GB+，CMS concurrent mode failure 退化成 Serial GC
2. **对象池改造**：Commons Pool2 池化 RoomContext（borrow/return+reset），Old Gen 使用率 85% → 25%
3. **切换 G1 调参**：
   ```
   -XX:+UseG1GC -Xms4g -Xmx4g
   -XX:MaxGCPauseMillis=50
   -XX:G1HeapRegionSize=16m
   -XX:G1ReservePercent=20
   -XX:InitiatingHeapOccupancyPercent=35  # 默认 45%，调低提前触发 Mixed GC
   ```
4. **Metaspace 控制**：`-XX:MaxMetaspaceSize=512m`，清理不必要的 String intern

**Result**：0 Full GC，Minor GC 每 2 分钟一次，< 30ms。

**IHOP 调低的代价**：Mixed GC 频率增加，CPU 稍高；收益是 Old Gen 不会堆满避免 Evacuation Failure 退化 Full GC。游戏场景 GC 停顿比多几个百分点 CPU 更重要。

**对象池大小如何确定**：统计高峰期 P99 在线房间数（如 200），池大小设为 250（25% 余量）。`minIdle=50, maxTotal=300, maxIdle=200`。超 maxTotal 配 `BlockWhenExhausted=true + MaxWaitMillis=3000`，超时返回"服务器繁忙"——拒绝单请求比全服 Full GC 代价小。监控：Prometheus 暴露 `numActive/numIdle`。

---

## 五、GC收集器选型

| 收集器 | JDK | 停顿 | 吞吐量 | 适用场景 |
|--------|-----|------|--------|---------|
| CMS | ≤ JDK 8 | 短（并发）| 一般 | 对延迟敏感，JDK 8 遗留系统 |
| G1 | JDK 9+ 默认 | 可控（目标停顿）| 高 | 大堆（4G+）通用场景 |
| ZGC | JDK 15+（生产）| 极短（< 1ms）| 略低 | 超大堆、极低延迟 |
| Shenandoah | JDK 12+ | 极短 | 略低 | 与 ZGC 类似，Red Hat 维护 |

**Java 8 → Java 11 GC 差异**：Java 8 默认 Parallel GC，Java 11 默认 G1；G1 的 Mixed GC 解决了 CMS 的碎片化问题（不再需要 Full GC 整理碎片）。

---

## 六、常用JVM参数速查

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

## 七、常见问题

### 1. 当时为什么选 G1 不选 ZGC？

时机问题。2019-2023 年 ZGC 在 JDK 11 是 Experimental 状态，生产不敢用。现在（JDK 17+）新项目会优先评估 ZGC——STW < 1ms，且无需手动调 IHOP 和 MaxGCPauseMillis。

代价：内存占用大（着色指针多用 ~15% 内存），4G 堆可能不够，需要 8G+；并发 GC 线程 CPU 开销比 G1 高。**游戏/金融对延迟敏感、有富余内存时选 ZGC；资源受限或堆 < 8G 仍选 G1。**

---

### 2. 对象池化后 reset 漏字段怎么发现？

`RoomContext` 实现 `Resettable` 接口，写单元测试通过反射遍历所有字段，检查 `reset()` 后每个字段是否等于其类型默认值（int=0, String=null, List=空集合）。新增字段时 CI 跑这个测试，漏了就 fail。

比 code review 更可靠，因为漏加 reset 是"遗忘型"错误，很难靠人眼发现。

---

## 八、与其他概念的关系

- 依赖 [[机制-垃圾收集器]]：调优的前提是理解 G1/ZGC 的 GC 流程
- 依赖 [[机制-JVM内存模型]]：堆/非堆结构决定了各类 OOM 的根因
- 关联 [[概念-引用类型]]：弱引用/软引用是防止内存泄漏的常用手段（缓存用 `WeakHashMap` / Guava Cache）
