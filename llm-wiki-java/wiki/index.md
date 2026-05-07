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

### 基础知识（CS 基础）
- [[概念-计算机网络]](concepts/概念-计算机网络.md) — TCP三次握手/拥塞控制、HTTP版本演进(HTTP/3 QUIC)、HTTPS握手、Cookie/Session/Token、跨域CORS `#network`
- [[概念-操作系统基础]](concepts/概念-操作系统基础.md) — 进程/线程/协程、用户态/内核态、零拷贝(sendfile)、epoll/IO多路复用、MESI缓存一致性、Page Cache `#os`

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
- [[概念-布隆过滤器]](concepts/概念-布隆过滤器.md) — BitMap+多哈希概率过滤、缓存穿透防护、无法删除缺陷、布谷鸟过滤器指纹+双桶支持删除 `#data-structure`
- [[概念-线性数据结构]](concepts/概念-线性数据结构.md) — 数组/链表/栈/队列，ArrayList/LinkedList/Vector对比，fail-fast陷阱 `#data-structure`
- [[机制-红黑树]](concepts/机制-红黑树.md) — 5条规则近似平衡，O(log n) 增删查，HashMap/TreeMap 底层 `#data-structure`
- [[机制-B树与B加树]](concepts/机制-B树与B加树.md) — 多路平衡树，低树高减少磁盘IO，MySQL InnoDB 索引底层 `#data-structure`
- [[机制-堆与优先队列]](concepts/机制-堆与优先队列.md) — 完全二叉树 + 数组，Top K 用小顶堆，PriorityQueue `#data-structure`
- [[概念-前缀树]](concepts/概念-前缀树.md) — 共享公共前缀，O(m) 字符串检索，搜索补全/AC自动机 `#data-structure`
- [[概念-BitMap]](concepts/概念-BitMap.md) — 1 bit 标记整数存在性，极致空间压缩，布隆过滤器基础 `#data-structure`
- [[概念-图论基础]](concepts/概念-图论基础.md) — 多对多关系，DFS/BFS 两种遍历，Dijkstra 依赖小顶堆 `#data-structure`
- [[机制-HashMap底层实现]](concepts/机制-HashMap底层实现.md) — 数组+链表+红黑树，扰动hash，0.75负载因子，JDK8高低位拆分扩容 `#data-structure`
- [[机制-ConcurrentHashMap并发设计]](concepts/机制-ConcurrentHashMap并发设计.md) — 分段锁(JDK7)→CAS+节点锁(JDK8)，fail-safe，不允许null的原因 `#data-structure`
- [[概念-Java集合框架]](concepts/概念-Java集合框架.md) — Collection/Map体系、List/Set/Queue接口语义、排序与线程安全选型 `#data-structure`

### L5 存储层
- [[概念-读写分离]](concepts/概念-读写分离.md) — 主从复制路由写主读从、ShardingSphere中间件分流、主从延迟处理策略(强制读主库/读请求分类) `#storage`
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
- [[机制-设计模式]](concepts/机制-设计模式.md) — SOLID七原则、单例(枚举最佳)、三种工厂、代理/享元、观察者/模板方法/策略/责任链及Spring应用 `#framework`
- [[机制-IoC容器]](concepts/机制-IoC容器.md) — 控制反转、Bean生命周期11步、三级缓存解决循环依赖 `#framework`
- [[机制-AOP织入]](concepts/机制-AOP织入.md) — JDK/CGLIB代理、5种Advice、失效场景（this调用/private/static/final）`#framework`
- [[机制-Spring事务]](concepts/机制-Spring事务.md) — @Transactional=AOP切面、7种传播机制、失效场景（代理失效/异常被吞/多线程）`#framework`
- [[机制-SpringBoot自动装配]](concepts/机制-SpringBoot自动装配.md) — @EnableAutoConfiguration、spring.factories→.imports、@Conditional条件装配、自定义starter `#framework`

### L7 分布式体系
- [[机制-ZAB协议与Zookeeper]](concepts/机制-ZAB协议与Zookeeper.md) — ZAB原子广播协议、ZNode/Watch机制、Leader选举、临时顺序节点分布式锁、CP强一致 `#distributed`
- [[机制-消息队列可靠性]](concepts/机制-消息队列可靠性.md) — Publisher Confirm+持久化+消费者手动ACK三段保证、死信队列、延迟消息、幂等消费 `#distributed`
- [[机制-倒排索引与ElasticSearch]](concepts/机制-倒排索引与ElasticSearch.md) — 倒排索引原理、集群角色、深度分页(scroll/search_after)、ES与DB一致性同步方案 `#distributed`
- [[机制-RPC与Dubbo]](concepts/机制-RPC与Dubbo.md) — RPC透明代理、Dubbo三阶段(注册/发现/调用)、5种负载均衡、Dubbo SPI增强、服务治理体系 `#distributed`
- [[机制-微服务与SpringCloud]](concepts/机制-微服务与SpringCloud.md) — Eureka(AP) vs ZK(CP)、熔断器三态(Hystrix→Sentinel)、OpenFeign声明式调用、Gateway响应式网关 `#distributed`
- [[机制-容器化与Docker]](concepts/机制-容器化与Docker.md) — 容器vs虚拟机(namespace+cgroup共享内核)、Dockerfile、Docker Compose、K8s编排核心对象 `#devops`
- [[概念-限流与熔断]](concepts/概念-限流与熔断.md) — 漏桶/令牌桶/滑动窗口对比、Guava RateLimiter、熔断器三态、Sentinel自适应限流 `#distributed`
- [[概念-高可用设计]](concepts/概念-高可用设计.md) — SLA四个九(99.99%)、冷热暖备、异地多活、全链路压测(流量染色+影子表) `#distributed`
- [[概念-分布式系统理论]](concepts/概念-分布式系统理论.md) — CAP三选二(CP/AP)、BASE基本可用+软状态+最终一致性、一致性哈希环+虚拟节点减少迁移 `#distributed`
- [[概念-幂等设计]](concepts/概念-幂等设计.md) — 业务幂等vs请求幂等、一锁二判三更新(分布式锁+流水表+DB约束兜底)、MQ消费幂等 `#distributed`
- [[机制-Canal数据同步]](concepts/机制-Canal数据同步.md) — 模拟MySQL slave拉取binlog、CDC变更捕获、MySQL→ES/缓存/异构库同步、分库分表买卖家表维护 `#distributed`

### L8 工程实践
<!-- 面经实战高频考点已提取到 Synthesis 分区 -->

---

## Entities（实体页）

> 具体框架、工具、组件、规范。有明确身份边界的具体对象。

### L5 存储层
- [[实体-MyBatis]](entities/实体-MyBatis.md) — 半自动ORM；#{}预编译防注入；一/二级缓存；PageHelper物理分页；插件责任链 `#storage`

---

## Summaries（主题总结页）

> 每页对应一个知识主题，聚合多个来源。不按源文件逐篇生成。

- [[主题-Spring体系]](summaries/主题-Spring体系.md) — L6 Spring/SpringBoot 知识地图，含IoC/AOP/事务/自动装配四大机制 `#framework`
- [[主题-Java集合框架]](summaries/主题-Java集合框架.md) — L4 集合框架知识地图 + 高频考点，含List/Set/Map/并发容器/Stream选型 `#data-structure`
- [[主题-Redis体系]](summaries/主题-Redis体系.md) — L5 Redis 知识地图 + 高频考点，含数据类型/持久化/缓存三大问题/分布式锁/集群 `#storage`
- [[主题-MySQL体系]](summaries/主题-MySQL体系.md) — L5 MySQL 知识地图 + 高频考点，含索引/MVCC/三种日志/锁机制/SQL优化 `#storage`
- [[主题-Java并发体系]](summaries/主题-Java并发体系.md) — L3 并发知识地图 + 高频考点，含 JMM/锁/CAS/AQS/线程池/ThreadLocal 依赖关系 `#concurrency`
- [[主题-Java语言基础]](summaries/主题-Java语言基础.md) — L1 概念地图 + 高频考点汇总，含 11 个 concept 页的依赖关系 `#java-lang`
- [[主题-数据结构体系]](summaries/主题-数据结构体系.md) — L4 数据结构知识地图 + 高频考点，含与 L5 MySQL/Redis 的联系 `#data-structure`
- [[主题-JVM体系]](summaries/主题-JVM体系.md) — L2 JVM 知识地图 + 高频考点，含 GC/类加载/JIT 的依赖关系 `#jvm`

---

## Synthesis（综合分析页）

> 围绕一个问题、比较或判断展开。涉及跨层或跨模块知识时使用。

### L8 工程实践
- [[设计-秒杀系统]](synthesis/设计-秒杀系统.md) — 多层漏斗过滤模型、Redis Lua预扣减、MQ削峰、热点行库存拆分、全链路幂等 `#practice`
- [[设计-分布式事务]](synthesis/设计-分布式事务.md) — 2PC/TCC/本地消息表/事务消息对比、TCC空回滚与悬挂、Seata AT vs 2PC区别、选型决策树 `#practice`
- [[设计-分库分表]](synthesis/设计-分库分表.md) — 分片键选择原则、雪花算法时钟回拨、基因法跨维度查询、双写扩容迁移方案 `#practice`
