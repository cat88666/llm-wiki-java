# Wiki Index — Java 后端工程师知识体系

> 本文件是当前知识库的总索引。每次 Ingest、Query 回写、Lint 后必须更新。  
> 格式：`- [[页面名]](相对路径) — 一句话描述 #标签`

---

## Template Files

> 模板文件，不参与 Query 和知识索引，仅供新建页面时参考。

- `wiki/templates/concept.md` — 概念页标准模板
- `wiki/templates/entity.md` — 实体页标准模板
- `wiki/templates/summary.md` — 主题总结页标准模板
- `wiki/templates/synthesis.md` — 综合分析页标准模板

---

## Concepts（概念页）

> 抽象机制、原理、模型、算法。脱离具体工具后仍成立的知识对象。

### L1 语言基础
- [[概念-OOP三大特征]](concepts/概念-OOP三大特征.md) — 封装/继承/多态，接口 vs 抽象类，组合优于继承 `#java-lang`
- [[机制-反射]](concepts/机制-反射.md) — 运行期读取类结构并调用，慢的根本原因 `#java-lang`
- [[机制-动态代理]](concepts/机制-动态代理.md) — JDK（接口）vs CGLIB（继承），Spring AOP 选择策略 `#java-lang`
- [[机制-泛型类型擦除]](concepts/机制-泛型类型擦除.md) — 编译期类型安全 + 运行期擦除，PECS 原则 `#java-lang`
- [[概念-Java异常体系]](concepts/概念-Java异常体系.md) — Checked vs Unchecked，勿用异常控流 `#java-lang`
- [[机制-Java序列化]](concepts/机制-Java序列化.md) — Serializable，serialVersionUID，反序列化安全风险 `#java-lang`
- [[机制-String不可变性]](concepts/机制-String不可变性.md) — final class + 常量池，StringBuilder 选择 `#java-lang`
- [[概念-包装类与自动拆装箱]](concepts/概念-包装类与自动拆装箱.md) — Integer 缓存陷阱，NPE 风险，金额用 BigDecimal `#java-lang`
- [[概念-IO模型]](concepts/概念-IO模型.md) — BIO/NIO/AIO，同步异步阻塞非阻塞 `#java-lang`
- [[机制-SPI]](concepts/机制-SPI.md) — 框架扩展点，ServiceLoader，Dubbo/SpringBoot 的变体 `#java-lang`
- [[机制-Lambda表达式]](concepts/机制-Lambda表达式.md) — invokedynamic 实现，Stream API，并行流陷阱 `#java-lang`

### L2 运行时（JVM）
- [[机制-JVM内存模型]](concepts/机制-JVM内存模型.md) — 5大运行时区域、堆分代（Eden/Survivor/Old）、元空间演变 `#jvm`
- [[机制-GC算法与垃圾收集器]](concepts/机制-GC算法与垃圾收集器.md) — 三大GC算法、可达性分析、CMS/G1/ZGC对比选型 `#jvm`
- [[机制-类加载机制]](concepts/机制-类加载机制.md) — 类生命周期、双亲委派模型、破坏场景（SPI/Tomcat/OSGi）`#jvm`
- [[机制-JIT编译]](concepts/机制-JIT编译.md) — 混合执行模式、热点检测、逃逸分析、预热问题 `#jvm`
- [[概念-引用类型]](concepts/概念-引用类型.md) — 强/软/弱/虚四种引用、ThreadLocal弱引用陷阱 `#jvm`

### L3 并发编程
- [[概念-JMM]](concepts/概念-JMM.md) — 主内存/工作内存模型，三大并发问题，happens-before 8条规则，内存屏障 `#concurrency`
- [[机制-synchronized]](concepts/机制-synchronized.md) — ObjectMonitor，偏向→轻量级→重量级锁升级链，JDK15废弃偏向锁 `#concurrency`
- [[机制-volatile]](concepts/机制-volatile.md) — LOCK前缀刷主内存，4种内存屏障，不保证原子性，DCL双重检验锁 `#concurrency`
- [[机制-AQS]](concepts/机制-AQS.md) — volatile state + CLH双向队列，同步队列/条件队列，独占/共享模式 `#concurrency`
- [[机制-CAS]](concepts/机制-CAS.md) — cmpxchg硬件原子指令，ABA问题+AtomicStampedReference，乐观锁vs悲观锁 `#concurrency`
- [[机制-线程池]](concepts/机制-线程池.md) — 7参数ThreadPoolExecutor，核心→队列→最大→拒绝执行流，4种拒绝策略 `#concurrency`
- [[概念-ThreadLocal]](concepts/概念-ThreadLocal.md) — Thread→ThreadLocalMap→弱引用key，线程池内存泄漏，必须remove `#concurrency`

### L4 数据结构
- [[概念-线性数据结构]](concepts/概念-线性数据结构.md) — 数组/链表/栈/队列，访问模式决定选择 `#data-structure`
- [[机制-红黑树]](concepts/机制-红黑树.md) — 5条规则近似平衡，O(log n) 增删查，HashMap/TreeMap 底层 `#data-structure`
- [[机制-B树与B加树]](concepts/机制-B树与B加树.md) — 多路平衡树，低树高减少磁盘IO，MySQL InnoDB 索引底层 `#data-structure`
- [[机制-堆与优先队列]](concepts/机制-堆与优先队列.md) — 完全二叉树 + 数组，Top K 用小顶堆，PriorityQueue `#data-structure`
- [[概念-前缀树]](concepts/概念-前缀树.md) — 共享公共前缀，O(m) 字符串检索，搜索补全/AC自动机 `#data-structure`
- [[概念-BitMap]](concepts/概念-BitMap.md) — 1 bit 标记整数存在性，极致空间压缩，布隆过滤器基础 `#data-structure`
- [[概念-图论基础]](concepts/概念-图论基础.md) — 多对多关系，DFS/BFS 两种遍历，Dijkstra 依赖小顶堆 `#data-structure`

### L5 存储层
- [[机制-InnoDB索引模型]](concepts/机制-InnoDB索引模型.md) — B+树索引、聚簇/二级索引、回表、覆盖索引、索引下推、最左前缀 `#storage`
- [[机制-MVCC]](concepts/机制-MVCC.md) — 快照读/当前读、undo log版本链、ReadView可见性、RC vs RR差异 `#storage`
- [[机制-MySQL三种日志]](concepts/机制-MySQL三种日志.md) — undo/redo/binlog的角色、两阶段提交保证主备一致、WAL原理 `#storage`
- [[机制-InnoDB锁机制]](concepts/机制-InnoDB锁机制.md) — S/X/IS/IX锁、Record/Gap/Next-Key Lock、RR防幻读、死锁 `#storage`
- [[概念-SQL查询优化]](concepts/概念-SQL查询优化.md) — EXPLAIN执行计划、索引失效场景、深分页、慢SQL排查流程 `#storage`
- [[概念-Redis数据类型与底层结构]](concepts/概念-Redis数据类型与底层结构.md) — 5种类型、SDS、ZipList/ListPack/SkipList二态编码、Redis快的5个原因 `#storage`
- [[机制-Redis持久化]](concepts/机制-Redis持久化.md) — RDB全量快照(BGSAVE+COW)、AOF三种写回策略、混合持久化(4.0+) `#storage`
- [[概念-缓存三大问题]](concepts/概念-缓存三大问题.md) — 穿透/击穿/雪崩定义与解法、8种内存淘汰策略、缓存与DB一致性 `#storage`
- [[机制-Redis分布式锁]](concepts/机制-Redis分布式锁.md) — SETNX+Lua防误删、Redisson watchdog续期、Hash可重入结构 `#storage`
- [[机制-Redis集群与高可用]](concepts/机制-Redis集群与高可用.md) — 主从/哨兵/Cluster三种模式、16384槽分片、脑裂防护 `#storage`

### L6 应用框架
<!-- 待摄入：Spring/、SpringBoot -->

### L7 分布式体系
<!-- 待摄入：SpringCloud/、Dubbo/、Zookeeper/、RabbitMQ/、ElasticSearch/ -->

### L8 工程实践
<!-- 待摄入：面经实战/ -->

---

## Entities（实体页）

> 具体框架、工具、组件、规范。有明确身份边界的具体对象。

<!-- 待摄入后填充 -->

---

## Summaries（主题总结页）

> 每页对应一个知识主题，聚合多个来源。不按源文件逐篇生成。

- [[主题-Redis体系]](summaries/主题-Redis体系.md) — L5 Redis 知识地图 + 高频考点，含数据类型/持久化/缓存三大问题/分布式锁/集群 `#storage`
- [[主题-MySQL体系]](summaries/主题-MySQL体系.md) — L5 MySQL 知识地图 + 高频考点，含索引/MVCC/三种日志/锁机制/SQL优化 `#storage`
- [[主题-Java并发体系]](summaries/主题-Java并发体系.md) — L3 并发知识地图 + 高频考点，含 JMM/锁/CAS/AQS/线程池/ThreadLocal 依赖关系 `#concurrency`
- [[主题-Java语言基础]](summaries/主题-Java语言基础.md) — L1 概念地图 + 高频考点汇总，含 11 个 concept 页的依赖关系 `#java-lang`
- [[主题-数据结构体系]](summaries/主题-数据结构体系.md) — L4 数据结构知识地图 + 高频考点，含与 L5 MySQL/Redis 的联系 `#data-structure`
- [[主题-JVM体系]](summaries/主题-JVM体系.md) — L2 JVM 知识地图 + 高频考点，含 GC/类加载/JIT 的依赖关系 `#jvm`

---

## Synthesis（综合分析页）

> 围绕一个问题、比较或判断展开。涉及跨层或跨模块知识时使用。

<!-- 待摄入后填充 -->
