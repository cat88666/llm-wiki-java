# Wiki Log

## [2026-05-02] Ingest | Redis模块（L5 存储层）
- 来源：`raw/note/Hollis/Redis/`（11 个问答式笔记）
- 新建 concept 页 × 5：
  - `概念-Redis数据类型与底层结构.md`（5种类型、SDS、ZipList/ListPack/SkipList二态编码、Redis快的5个原因）
  - `机制-Redis持久化.md`（RDB/AOF/混合持久化、fork+COW、AOF三种写回策略、重写机制）
  - `概念-缓存三大问题.md`（穿透/击穿/雪崩定义与解法、布隆过滤器、8种淘汰策略、缓存与DB一致性）
  - `机制-Redis分布式锁.md`（SETNX+Lua防误删、Redisson watchdog续期、Hash可重入、RedLock）
  - `机制-Redis集群与高可用.md`（主从/哨兵/Cluster三种模式、16384槽、脑裂防护）
- 新建 summary 页 × 1：`主题-Redis体系.md`（含知识地图 + 高频考点 + 与 L4/L3/MySQL/L6/L7 联系）
- 更新 `wiki/index.md`：L5 存储层新增 5 个 Redis concept 链接；Summaries 分区新增 1 条

本文件记录所有操作，append-only。

格式：
`## [YYYY-MM-DD] 操作类型 | 标题`

## [2026-05-02] Ingest | MySQL模块（L5 存储层）
- 来源：`raw/note/Hollis/MySQL/`（120+ 个问答式笔记）
- 新建 concept 页 × 5：
  - `机制-InnoDB索引模型.md`（B+树选型、聚簇/非聚簇索引、回表、覆盖索引、索引下推、最左前缀）
  - `机制-MVCC.md`（undo log版本链、ReadView四字段、快照读/当前读、RC vs RR）
  - `机制-MySQL三种日志.md`（undo/redo/binlog角色对比、两阶段提交XID机制、WAL原理）
  - `机制-InnoDB锁机制.md`（S/X/IS/IX锁、Record/Gap/Next-Key Lock、RR加锁规则、死锁）
  - `概念-SQL查询优化.md`（EXPLAIN关键字段、索引失效场景、深分页优化、调优流程）
- 新建 summary 页 × 1：`主题-MySQL体系.md`（含知识地图 + 高频考点 + 与 L4/L6/L7 联系）
- 更新 `wiki/index.md`：L5 存储层分区填充 5 个 concept 链接；Summaries 分区新增 1 条

## [2026-05-02] Ingest | Java并发模块（L3 并发编程）
- 来源：`raw/note/Hollis/Java并发/`（68 个问答式笔记）
- 新建 concept 页 × 7：
  - `概念-JMM.md`（主内存/工作内存、三大并发问题、happens-before 8条规则、内存屏障）
  - `机制-synchronized.md`（ObjectMonitor、偏向→轻量级→重量级锁升级、JDK15废弃偏向锁）
  - `机制-volatile.md`（LOCK前缀、4种内存屏障、不保证原子性、DCL双重检验锁）
  - `机制-AQS.md`（volatile state + CLH双向队列、同步队列/条件队列、独占/共享模式）
  - `机制-CAS.md`（cmpxchg硬件指令、ABA问题+AtomicStampedReference、忙等待）
  - `机制-线程池.md`（7参数、核心→队列→最大→拒绝执行流、4种拒绝策略、Worker机制）
  - `概念-ThreadLocal.md`（ThreadLocalMap、弱引用key、value强引用泄漏、线程池必须remove）
- 新建 summary 页 × 1：`主题-Java并发体系.md`（含知识地图 + 高频考点 + 与 L2/L5/L6 联系）
- 更新 `wiki/index.md`：L3 并发编程分区填充 7 个 concept 链接；Summaries 分区新增 1 条

## [2026-05-02] Ingest | JVM模块（L2 运行时）
- 来源：`raw/note/Hollis/JVM/`（57 个问答式笔记）
- 注：`容器/` 目录是 Docker/k8s 内容（非 Java 集合框架），本次跳过
- 新建 concept 页 × 5：
  - `机制-JVM内存模型.md`（5大区域、堆分代、元空间演变）
  - `机制-GC算法与垃圾收集器.md`（三算法、可达性分析、CMS/G1/ZGC对比）
  - `机制-类加载机制.md`（类生命周期、双亲委派、三大破坏场景）
  - `机制-JIT编译.md`（热点探测、逃逸分析、AOT对比、预热问题）
  - `概念-引用类型.md`（强/软/弱/虚、ThreadLocal弱引用陷阱）
- 新建 summary 页 × 1：`主题-JVM体系.md`（含知识地图 + 高频考点 + 与 L1/L3/L5 的联系）
- 更新 `wiki/index.md`：L2 运行时分区填充 5 个 concept 链接；Summaries 分区新增 1 条

## [2026-05-02] Ingest | 数据结构模块（L4 数据结构）
- 来源：`raw/note/Hollis/数据结构/`（12 个问答式笔记）
- 新建 concept 页 × 7：
  - `概念-线性数据结构.md`（数组/链表/栈/队列）
  - `机制-红黑树.md`（5条规则、旋转着色、HashMap/TreeMap 应用）
  - `机制-B树与B加树.md`（多路平衡、B vs B+对比、InnoDB 选型）
  - `机制-堆与优先队列.md`（数组实现、Top K 小顶堆、TP99）
  - `概念-前缀树.md`（Trie、O(m)检索、搜索补全/AC自动机）
  - `概念-BitMap.md`（位数组、空间压缩、布隆过滤器基础）
  - `概念-图论基础.md`（有向/无向、邻接表、DFS/BFS、Dijkstra）
- 新建 summary 页 × 1：`主题-数据结构体系.md`（含知识地图 + 高频考点 + 与 L4/L5 联系）
- 更新 `wiki/index.md`：L4 数据结构分区填充 7 个 concept 链接；Summaries 分区新增 1 条

## [2026-05-02] Ingest | Java基础模块（L1 语言基础）
- 来源：`raw/note/Hollis/Java基础/`（60+ 问答式笔记）
- 新建 concept 页 × 11：
  - `概念-OOP三大特征.md`
  - `机制-反射.md`
  - `机制-动态代理.md`
  - `机制-泛型类型擦除.md`
  - `概念-Java异常体系.md`
  - `机制-Java序列化.md`
  - `机制-String不可变性.md`
  - `概念-包装类与自动拆装箱.md`
  - `概念-IO模型.md`
  - `机制-SPI.md`
  - `机制-Lambda表达式.md`
- 新建 summary 页 × 1：`主题-Java语言基础.md`（含知识地图 + 高频考点 + 依赖关系）
- 更新 `wiki/index.md`：L1 语言基础分区填充 11 个 concept 链接；Summaries 分区新增 1 条

## [2026-05-02] Init | Java 知识库初始化
- 重命名实例目录 `llm-wiki-template/` → `llm-wiki-java/`
- 重写根目录 `CLAUDE.md`：指向新实例路径，增加 Java 知识编译原则
- 重写根目录 `README.md`：建立 L1-L8 知识结构图，规划摄入顺序
- 重写实例 `CLAUDE.md`：定义知识分层（L1-L8）、concepts/ 6段结构、Java特定摄入规则
- 重置 `wiki/index.md`：清除 demo 内容，按 L1-L8 层级建立占位骨架

## [2026-05-02] Update | Hollis来源目录更名
- 重命名来源目录：emoji 前缀的旧 Hollis 目录 → `raw/note/Hollis/`
- 批量更新项目说明、实例规范、wiki concept/summary/log 中的来源路径引用
- 验证旧目录名称已无文本残留
