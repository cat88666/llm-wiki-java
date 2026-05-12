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
- [[概念-计算机网络]](concepts/01-cs-base/概念-计算机网络.md) — TCP三次握手/拥塞控制、HTTP版本演进(HTTP/3 QUIC)、HTTPS握手、Cookie/Session/Token、跨域CORS `#network`
- [[概念-操作系统基础]](concepts/01-cs-base/概念-操作系统基础.md) — 进程/线程/协程、用户态/内核态、零拷贝(sendfile)、epoll/IO多路复用、MESI缓存一致性、Page Cache `#os`

### L1 语言基础
- [[概念-OOP三大特征]](concepts/02-java-lang/概念-OOP三大特征.md) — 封装/继承/多态，接口 vs 抽象类，组合优于继承 `#java-lang`
- [[机制-反射与动态代理]](concepts/02-java-lang/机制-反射与动态代理.md) — 运行期 Class 元数据读取/调用；JDK代理（接口）vs CGLIB（子类继承），Spring AOP 选择策略 `#java-lang`
- [[机制-泛型类型擦除]](concepts/02-java-lang/机制-泛型类型擦除.md) — 编译期类型安全 + 运行期擦除，PECS 原则 `#java-lang`
- [[概念-Java异常体系]](concepts/02-java-lang/概念-Java异常体系.md) — Checked vs Unchecked，勿用异常控流 `#java-lang`
- [[机制-Java序列化]](concepts/02-java-lang/机制-Java序列化.md) — Serializable，serialVersionUID，反序列化安全风险 `#java-lang`
- [[概念-Java基础值类型]](concepts/02-java-lang/概念-Java基础值类型.md) — String不可变/常量池/StringBuilder选择；Integer缓存[-128,127]陷阱/NPE/RPC选型；BigDecimal金额精确计算 `#java-lang`
- [[概念-IO模型]](concepts/01-cs-base/概念-IO模型.md) — BIO/NIO/AIO，同步异步阻塞非阻塞 `#java-lang`
- [[机制-SPI]](concepts/02-java-lang/机制-SPI.md) — 框架扩展点，ServiceLoader，Dubbo/SpringBoot 的变体 `#java-lang`
- [[机制-Lambda表达式]](concepts/02-java-lang/机制-Lambda表达式.md) — invokedynamic 实现，Stream API，并行流陷阱 `#java-lang`
- [[概念-JDK新特性]](concepts/02-java-lang/概念-JDK新特性.md) — JDK8(Lambda/Stream/Optional)、JDK16 Record、JDK17 Sealed Classes、JDK21 虚拟线程(Virtual Threads) `#java-lang`

### L2 运行时（JVM）
- [[机制-JVM内存模型]](concepts/03-jvm/机制-JVM内存模型.md) — 5大运行时区域、堆分代（Eden/Survivor/Old）、元空间演变 `#jvm`
- [[机制-GC算法与垃圾收集器]](concepts/03-jvm/机制-GC算法与垃圾收集器.md) — 三大GC算法、可达性分析、CMS/G1/ZGC对比选型；强/软/弱/虚四种引用类型与GC行为 `#jvm`
- [[机制-类加载机制]](concepts/03-jvm/机制-类加载机制.md) — 类生命周期、双亲委派模型、破坏场景（SPI/Tomcat/OSGi）`#jvm`
- [[机制-JIT编译]](concepts/03-jvm/机制-JIT编译.md) — 混合执行模式、热点检测、逃逸分析、预热问题 `#jvm`
- [[机制-对象池技术]](concepts/03-jvm/机制-对象池技术.md) — 预创建对象复用降低GC压力、HikariCP连接池ConcurrentBag无锁设计、Netty PooledByteBufAllocator堆外内存池化、0 FGC实现路径 `#jvm`

### L3 并发编程
- [[概念-JMM]](concepts/04-concurrency/概念-JMM.md) — 主内存/工作内存模型，三大并发问题，happens-before 8条规则，内存屏障 `#concurrency`
- [[机制-synchronized]](concepts/04-concurrency/机制-synchronized.md) — ObjectMonitor，偏向→轻量级→重量级锁升级链，JDK15废弃偏向锁 `#concurrency`
- [[机制-volatile]](concepts/04-concurrency/机制-volatile.md) — LOCK前缀刷主内存，4种内存屏障，不保证原子性，DCL双重检验锁 `#concurrency`
- [[机制-AQS]](concepts/04-concurrency/机制-AQS.md) — volatile state + CLH双向队列，同步队列/条件队列，独占/共享模式 `#concurrency`
- [[机制-CAS]](concepts/04-concurrency/机制-CAS.md) — cmpxchg硬件原子指令，ABA问题+AtomicStampedReference，乐观锁vs悲观锁 `#concurrency`
- [[机制-线程池]](concepts/04-concurrency/机制-线程池.md) — 7参数ThreadPoolExecutor，核心→队列→最大→拒绝执行流，4种拒绝策略 `#concurrency`
- [[概念-ThreadLocal]](concepts/04-concurrency/概念-ThreadLocal.md) — Thread→ThreadLocalMap→弱引用key，线程池内存泄漏，必须remove `#concurrency`
- [[机制-CompletableFuture与异步编程]](concepts/04-concurrency/机制-CompletableFuture与异步编程.md) — 链式Completion阶段事件驱动、ForkJoinPool默认线程池、thenApply/thenCompose/allOf任务编排、I/O密集型必须自定义线程池 `#concurrency`

### L4 数据结构
- [[概念-BitMap与布隆过滤器]](concepts/05-data-structure/概念-BitMap与布隆过滤器.md) — BitMap 32x压缩；布隆过滤器（BitMap+多哈希，缓存穿透防护，无法删除）；布谷鸟过滤器（指纹+双桶，支持删除）`#data-structure`
- [[概念-线性数据结构]](concepts/05-data-structure/概念-线性数据结构.md) — 数组/链表/栈/队列；Collection/Map体系概览；ArrayList/LinkedList/Vector对比；Set去重/Map选型/线程安全容器 `#data-structure`
- [[机制-红黑树]](concepts/05-data-structure/机制-红黑树.md) — 5条规则近似平衡，O(log n) 增删查，HashMap/TreeMap 底层 `#data-structure`
- [[机制-B树与B加树]](concepts/05-data-structure/机制-B树与B加树.md) — 多路平衡树，低树高减少磁盘IO，MySQL InnoDB 索引底层 `#data-structure`
- [[机制-堆与优先队列]](concepts/05-data-structure/机制-堆与优先队列.md) — 完全二叉树 + 数组，Top K 用小顶堆，PriorityQueue `#data-structure`
- [[概念-前缀树]](concepts/05-data-structure/概念-前缀树.md) — 共享公共前缀，O(m) 字符串检索，搜索补全/AC自动机 `#data-structure`
- [[概念-图论基础]](concepts/05-data-structure/概念-图论基础.md) — 多对多关系，DFS/BFS 两种遍历，Dijkstra 依赖小顶堆 `#data-structure`
- [[机制-HashMap底层实现]](concepts/05-data-structure/机制-HashMap底层实现.md) — 数组+链表+红黑树，扰动hash，0.75负载因子，JDK8高低位拆分扩容 `#data-structure`
- [[机制-ConcurrentHashMap并发设计]](concepts/05-data-structure/机制-ConcurrentHashMap并发设计.md) — 分段锁(JDK7)→CAS+节点锁(JDK8)，fail-safe，不允许null的原因 `#data-structure`

### L5 存储层
- [[概念-读写分离]](concepts/06-storage/概念-读写分离.md) — 主从复制路由写主读从、ShardingSphere中间件分流、主从延迟处理策略(强制读主库/读请求分类) `#storage`
- [[机制-InnoDB索引模型]](concepts/06-storage/机制-InnoDB索引模型.md) — B+树索引、聚簇/二级索引、回表、覆盖索引、索引下推、最左前缀 `#storage`
- [[机制-MVCC]](concepts/06-storage/机制-MVCC.md) — 快照读/当前读、undo log版本链、ReadView可见性、RC vs RR差异 `#storage`
- [[机制-MySQL三种日志]](concepts/06-storage/机制-MySQL三种日志.md) — undo/redo/binlog的角色、两阶段提交保证主备一致、WAL原理 `#storage`
- [[机制-InnoDB锁机制]](concepts/06-storage/机制-InnoDB锁机制.md) — S/X/IS/IX锁、Record/Gap/Next-Key Lock、RR防幻读、死锁 `#storage`
- [[概念-SQL查询优化]](concepts/06-storage/概念-SQL查询优化.md) — EXPLAIN执行计划、索引失效场景、深分页、慢SQL排查流程 `#storage`
- [[概念-Redis数据类型与底层结构]](concepts/06-storage/概念-Redis数据类型与底层结构.md) — 5种类型、SDS、ZipList/ListPack/SkipList二态编码、Redis快的5个原因 `#storage`
- [[机制-Redis持久化]](concepts/06-storage/机制-Redis持久化.md) — RDB全量快照(BGSAVE+COW)、AOF三种写回策略、混合持久化(4.0+) `#storage`
- [[概念-缓存三大问题]](concepts/06-storage/概念-缓存三大问题.md) — 穿透/击穿/雪崩定义与解法、8种内存淘汰策略、缓存与DB一致性 `#storage`
- [[机制-Redis分布式锁]](concepts/06-storage/机制-Redis分布式锁.md) — SETNX+Lua防误删、Redisson watchdog续期、Hash可重入结构 `#storage`
- [[机制-Redis集群与高可用]](concepts/06-storage/机制-Redis集群与高可用.md) — 主从/哨兵/Cluster三种模式、16384槽分片、脑裂防护 `#storage`
- [[机制-LSM树与RocksDB]](concepts/06-storage/机制-LSM树与RocksDB.md) — LSM-Tree顺序写原理、WAL+Memtable+SSTable三层结构、Compaction写放大权衡、RocksDB列族/事务特性、与B+树的对立选型 `#storage`

### L6 应用框架
- [[机制-设计模式]](concepts/07-framework/机制-设计模式.md) — SOLID七原则、单例(枚举最佳)、三种工厂、代理/享元、观察者/模板方法/策略/责任链及Spring应用 `#framework`
- [[机制-IoC容器]](concepts/07-framework/机制-IoC容器.md) — 控制反转、Bean生命周期11步、三级缓存解决循环依赖 `#framework`
- [[机制-AOP织入]](concepts/07-framework/机制-AOP织入.md) — JDK/CGLIB代理、5种Advice、失效场景（this调用/private/static/final）`#framework`
- [[机制-Spring事务]](concepts/07-framework/机制-Spring事务.md) — @Transactional=AOP切面、7种传播机制、失效场景（代理失效/异常被吞/多线程）`#framework`
- [[机制-SpringBoot自动装配]](concepts/07-framework/机制-SpringBoot自动装配.md) — @EnableAutoConfiguration、spring.factories→.imports、@Conditional条件装配、自定义starter、完整启动流程、优雅停机 `#framework`
- [[机制-SpringMVC请求处理链]](concepts/07-framework/机制-SpringMVC请求处理链.md) — DispatcherServlet前端控制器、MappingRegistry路由注册、HandlerAdapter适配、Interceptor责任链、ExceptionResolver异常处理 `#framework`

### L7 分布式体系
- [[机制-ZAB协议与Zookeeper]](concepts/08-distributed/机制-ZAB协议与Zookeeper.md) — ZAB原子广播协议、ZNode/Watch机制、Leader选举、临时顺序节点分布式锁、CP强一致 `#distributed`
- [[机制-消息队列可靠性]](concepts/08-distributed/机制-消息队列可靠性.md) — Publisher Confirm+持久化+消费者手动ACK三段保证、死信队列、延迟消息、幂等消费 `#distributed`
- [[机制-倒排索引与ElasticSearch]](concepts/08-distributed/机制-倒排索引与ElasticSearch.md) — 倒排索引原理、集群角色、深度分页(scroll/search_after)、ES与DB一致性同步方案 `#distributed`
- [[机制-RPC与Dubbo]](concepts/08-distributed/机制-RPC与Dubbo.md) — RPC透明代理、Dubbo三阶段(注册/发现/调用)、5种负载均衡、Dubbo SPI增强、服务治理体系 `#distributed`
- [[机制-微服务与SpringCloud]](concepts/08-distributed/机制-微服务与SpringCloud.md) — Eureka(AP) vs ZK(CP)、熔断器三态(Hystrix→Sentinel)、OpenFeign声明式调用、Gateway响应式网关、蓝绿/灰度部署、Service Mesh、限流/降级/熔断区别、CI/CD `#distributed`
- [[机制-容器化与Docker]](concepts/08-distributed/机制-容器化与Docker.md) — 容器vs虚拟机(namespace+cgroup共享内核)、Dockerfile、Docker Compose、K8s编排核心对象 `#devops`
- [[概念-限流与熔断]](concepts/08-distributed/概念-限流与熔断.md) — 漏桶/令牌桶/滑动窗口对比、Guava RateLimiter、熔断器三态、Sentinel自适应限流 `#distributed`
- [[概念-高可用设计]](concepts/08-distributed/概念-高可用设计.md) — SLA四个九(99.99%)、冷热暖备、异地多活、全链路压测(流量染色+影子表) `#distributed`
- [[概念-分布式系统理论]](concepts/08-distributed/概念-分布式系统理论.md) — CAP三选二(CP/AP)、BASE基本可用+软状态+最终一致性、一致性哈希环+虚拟节点减少迁移 `#distributed`
- [[概念-幂等设计]](concepts/08-distributed/概念-幂等设计.md) — 业务幂等vs请求幂等、一锁二判三更新(分布式锁+流水表+DB约束兜底)、MQ消费幂等 `#distributed`
- [[机制-Canal数据同步]](concepts/08-distributed/机制-Canal数据同步.md) — 模拟MySQL slave拉取binlog、CDC变更捕获、MySQL→ES/缓存/异构库同步、分库分表买卖家表维护 `#distributed`
- [[概念-网络安全]](concepts/08-distributed/概念-网络安全.md) — SQL注入(预编译)、XSS/CSRF防御、垂直/水平越权(RBAC/session)、中间人攻击(HTTPS/HSTS)、撞库/拖库/洗库、国密SM2/SM3/SM4/SM9 `#security`
- [[机制-Kafka]](concepts/08-distributed/机制-Kafka.md) — Topic/Partition/Segment存储结构、顺序写+零拷贝高吞吐原因、ISR/HW/LeaderEpoch可靠性、CooperativeStickyAssignor渐进式重平衡、At-least-once/Exactly-once语义 `#distributed`
- [[机制-Netty]](concepts/08-distributed/机制-Netty.md) — 主从Reactor多线程模型、epoll事件通知、EventLoop单线程无锁、ByteBuf读写双指针+池化、粘包拆包LengthField方案、IdleStateHandler心跳、TCP参数调优(TCP_NODELAY/SO_RCVBUF/写缓冲水位) `#distributed`
- [[机制-gRPC与Protobuf]](concepts/08-distributed/机制-gRPC与Protobuf.md) — HTTP/2多路复用+Protobuf二进制、4种通信模式(Unary/双向流)、Interceptor、Deadline传播、vs REST/Dubbo选型 `#distributed`
- [[机制-WebSocket协议]](concepts/08-distributed/机制-WebSocket协议.md) — HTTP Upgrade全双工握手、帧结构、心跳、WSS安全认证、跨节点路由(Redis注册表+MQ)、vs SSE/长轮询 `#distributed`
- [[机制-数据加密与脱敏]](concepts/08-distributed/机制-数据加密与脱敏.md) — AES-256-GCM认证加密、RSA混合加密模式、HMAC-SHA256请求签名防重放、数据脱敏策略、字段级加密+Hash索引 `#security`
- [[机制-RocketMQ]](concepts/08-distributed/机制-RocketMQ.md) — CommitLog顺序写、事务消息半消息+回查机制、延迟消息固定Level、集群vs广播消费、死信队列、vs Kafka选型 `#distributed`
- [[概念-可观测性]](concepts/08-distributed/概念-可观测性.md) — Metrics/Tracing/Logging三大支柱、SkyWalking字节码Agent架构、Prometheus四种Metric类型、P99指标、三支柱协作排障流程 `#distributed`
- [[机制-RabbitMQ]](concepts/08-distributed/机制-RabbitMQ.md) — AMQP三层路由(Exchange/Binding/Queue)、6种工作模式、Publisher Confirm+持久化+ACK三道防线、死信队列+延迟消息、镜像集群高可用 `#distributed`
- [[机制-配置中心与Nacos]](concepts/08-distributed/机制-配置中心与Nacos.md) — 配置中心解耦动态配置、Nacos CP(JRaft)+AP(Distro)双模式、长轮询→gRPC长连接感知变更、服务注册发现推拉结合、注册中心选型对比 `#distributed`
- [[机制-分布式任务调度]](concepts/08-distributed/机制-分布式任务调度.md) — XXL-Job DB悲观锁唯一触发、分片任务(ShardIndex+Total)、时间轮O(1)调度、定时扫表三缺陷与解法、退避策略、PowerJob动态分片 `#distributed`
- [[机制-Seata框架机制]](concepts/08-distributed/机制-Seata框架机制.md) — TC/TM/RM三组件全局事务协调、AT(代理数据源+UNDO_LOG)、TCC(空回滚/悬挂+分布式事务记录表)、四种模式选型 `#distributed`
- [[概念-OAuth2授权协议]](concepts/08-distributed/概念-OAuth2授权协议.md) — 开放授权、Client/ResourceServer/OAuthServer三角色、Access Token授权流程、四种授权类型、OAuth2 vs OIDC `#distributed`
- [[概念-DDD]](concepts/09-practice/概念-DDD.md) — 领域驱动设计、实体/值对象/聚合根、充血模型vs贫血模型、四层架构(用户接口/应用/领域/基础设施)、限界上下文作为微服务拆分依据、CQRS、Event Sourcing `#practice`

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
- [[主题-消息队列体系]](summaries/主题-消息队列体系.md) — Kafka/RabbitMQ/RocketMQ全景对比、三道防线可靠性、幂等消费、选型决策树 `#distributed`
- [[主题-三高体系]](summaries/主题-三高体系.md) — 高并发/高可用/高性能三者制约关系：限流算法/熔断/SLA/压测/读写分离/布隆过滤器/线程池调优 `#practice #distributed #concurrency`
- [[主题-锁体系]](summaries/主题-锁体系.md) — Java锁/MySQL锁/分布式锁全景：乐观悲观/读写/公平非公平/自旋阻塞/synchronized/AQS/InnoDB行锁/Redis+ZK分布式锁/选型决策树 `#concurrency #storage #distributed`
- [[主题-源码体系]](summaries/主题-源码体系.md) — Java核心类库/并发包/Spring全家桶/MyBatis/Tomcat五大域源码地图：关键入口→调用链→面试结论 `#java-lang #concurrency #data-structure #framework #jvm`

---

## Synthesis（综合分析页）

> 围绕一个问题、比较或判断展开。涉及跨层或跨模块知识时使用。

### L8 工程实践
- [[设计-秒杀系统]](synthesis/设计-秒杀系统.md) — 多层漏斗过滤模型、Redis Lua预扣减、MQ削峰、热点行库存拆分、全链路幂等 `#practice`
- [[设计-分布式事务]](synthesis/设计-分布式事务.md) — 2PC/TCC/本地消息表/事务消息对比、TCC空回滚与悬挂、Seata AT vs 2PC区别、选型决策树 `#practice`
- [[设计-分库分表]](synthesis/设计-分库分表.md) — 分片键选择原则、雪花算法时钟回拨、基因法跨维度查询、双写扩容迁移方案 `#practice`
- [[设计-Redis实战场景]](synthesis/设计-Redis实战场景.md) — ZSet排行榜(分片+异步+分数编码)、GEO附近的人、点赞系统、抢红包(二倍均值法)、HyperLogLog UV统计、购物车Hash `#practice`
- [[设计-短链服务]](synthesis/设计-短链服务.md) — MurmurHash+Base62生成、302 vs 301跳转选型、Redis+MySQL双存储、防滥用机制 `#practice`
- [[设计-订单超时关闭]](synthesis/设计-订单超时关闭.md) — 方案对比(定时扫表/DelayQueue/Redisson/MQ)、Redisson延迟队列分布式首选、关闭逻辑幂等设计 `#practice`
- [[设计-线上问题排查]](synthesis/设计-线上问题排查.md) — CPU高/FullGC/OOM/慢SQL/死锁/连接池/MQ堆积实战案例，Arthas/jstack/MAT工具链，死循环vs死锁对CPU影响，数据倾斜规律 `#practice`
- [[设计-算法高频题型]](synthesis/设计-算法高频题型.md) — LRU(HashMap+双向链表)、TopK(小顶堆/快速选择)、海量数据(哈希分片)、排序选型、二分模板、滑动窗口、DP转移方程、并查集、拓扑排序 `#practice`
- [[设计-多级缓存架构]](synthesis/设计-多级缓存架构.md) — 本地Caffeine(W-TinyLFU)+Redis两级缓存、TTL/MQ广播/Canal三种一致性策略、双重检测防雪崩代码模式、缓存预热方案 `#practice`
- [[设计-支付系统设计]](synthesis/设计-支付系统设计.md) — 聚合支付架构/渠道路由、TCC空回滚+悬挂处理、双记账流水表设计、T+1对账+实时对账双兜底、OTC撮合引擎+冷热钱包分离、资损防控三道防线 `#practice`
- [[设计-JVM调优实战]](synthesis/设计-JVM调优实战.md) — 正常GC基线指标、频繁FullGC/OOM排查步骤、Arthas/jmap工具链、G1/ZGC参数速查、OOM不一定导致JVM退出 `#jvm #practice`
- [[设计-MySQL大表与查询优化]](synthesis/设计-MySQL大表与查询优化.md) — B+树视角2000w分表阈值、深分页3种优化、大表DDL三方案(OnlineDDL/pt-osc/gh-ost)、热点行危害、长事务锁包事务 `#storage #practice`
- [[设计-海量数据处理]](synthesis/设计-海量数据处理.md) — BitMap去重、Bloom filter内存估算、外部归并排序(分块+多路归并)、TopK哈希分片+堆、AC自动机敏感词过滤 `#data-structure #practice`
- [[设计-接口设计与防护]](synthesis/设计-接口设计与防护.md) — 滑动窗口频率限制、Token防重复点击、6层纵深防刷、三方接口超时隔离、支付查单对账、Token踢下线 `#distributed #practice`
- [[设计-消息队列场景综合]](synthesis/设计-消息队列场景综合.md) — SpringEvent vs MQ、拉推模式对比、消息乱序4种解法、Kafka单分区提吞吐、本地消息表最终一致、消息表SQL设计 `#distributed #practice`
- [[设计-容量规划与性能基准]](synthesis/设计-容量规划与性能基准.md) — 4C8G正常指标基准表、机器数预估公式、压测线上差异8大原因、堆外内存泄漏诊断、银行系统ZGC选型 `#practice #jvm`
- [[设计-分布式场景综合]](synthesis/设计-分布式场景综合.md) — 三种锁选型表、Redis预扣库存、分布式Session演进、SSO全流程、订单号生成、跨库JOIN、平滑迁移双写策略 `#distributed #practice`
- [[设计-架构设计思维]](synthesis/设计-架构设计思维.md) — 架构本质是权衡、没有银弹、技术债管理、微服务拆分7原则、单元化架构、技术选型框架、亿级商品存储 `#practice #distributed`
- [[设计-在线游戏平台架构]](synthesis/设计-在线游戏平台架构.md) — SOFA-Bolt+Netty+Dubbo 5服务协作：用户级线程隔离、工厂路由、游戏状态机、保险赔率实时计算、多级缓存 `#practice #distributed`
- [[设计-德州扑克核心算法]](synthesis/设计-德州扑克核心算法.md) — 牌型多项式编码O(1)比较、三变体工厂评估器、智能保险方程组迭代求解、边池尾差处理、抽水两种模式 `#practice`
- [[设计-项目难点表达]](synthesis/设计-项目难点表达.md) — STAR框架亮点模板库：CompletableFuture(10s→1s)/状态机+乐观锁/BitSet预约(400x压缩)/ZSet排行榜/本地消息表/XXL-JOB分片/TTL/Spring Event等10个量化模板 `#practice`
- [[设计-大厂秒杀实践]](synthesis/设计-大厂秒杀实践.md) — 阿里Inventory Hint(行锁合并/Row Cache/组提交)+小红书合并秒杀(Leader-Follower 5.5x提升)对比教科书Redis预扣方案 `#practice #storage #distributed`
- [[设计-DDD落地实战]](synthesis/设计-DDD落地实战.md) — 限界上下文→微服务拆分决策、聚合根粒度权衡(太细→分布式事务/太粗→锁竞争)、四层架构各层职责、领域事件Spring Event实战、CQRS适用边界 `#practice #framework`
- [[设计-P8高频边界问题]](synthesis/设计-P8高频边界问题.md) — 系统边界职责/造轮子五问/库存三态与旁路验证/雪花算法边界(32台/时钟回拨)/MQ异步刷盘能不丢吗/OPS突增10倍框架/选型追问四维度 `#practice`
- [[设计-云原生选型]](synthesis/设计-云原生选型.md) — IaaS/PaaS/SaaS/Serverless交付模型、公有云/私有云/混合云选型、Java冷启动问题(GraalVM AOT)、无状态设计原则、云原生改造要点 `#practice #distributed`
- [[设计-彩票系统核心设计]](synthesis/设计-彩票系统核心设计.md) — 投注系统(乐观锁余额冻结/幂等下单)、开奖系统(VRF可验证随机/状态机/防篡改)、结算系统(固定赔率vs彩池/Kafka批量驱动百万注单)、风控防作弊 `#practice #distributed`
- [[设计-账户钱包系统]](synthesis/设计-账户钱包系统.md) — 余额三态模型(可用/冻结/总额)、乐观锁扣减并发安全、append-only流水表(balance_after快照)、充值/提现链路幂等、T+1对账+实时监控 `#practice #storage`
- [[模拟面试-Eson]](synthesis/mock-interview-eson.md) — 资深Java架构师60分钟结构化模拟面试：Seata TCC/JVM 0FGC/CoinsOTC区块链/Gacha系统设计 `#practice`
- [[设计-区块链OTC平台]](synthesis/设计-区块链OTC平台.md) — bitcoinj/web3j集成、UTXO vs账户模型、冷热钱包架构(PSBT离线签名)、P2P撮合引擎、充提款异步对账、资金安全三道防线 `#practice`
- [[设计-IM即时通信系统]](synthesis/设计-IM即时通信系统.md) — WebSocket接入层、消息可靠投递(seqId+ACK)、跨节点路由(Redis注册+MQ)、群消息扇出策略、离线推送、消息存储冷热分层 `#practice #distributed`
- [[设计-Gacha与游戏高并发]](synthesis/设计-Gacha与游戏高并发.md) — Redis Lua原子扣减+本地消息表零超卖、对象池+G1消除GC抖动、多级缓存防雪崩、实时对战状态机、跨地域延迟优化(TCP_NODELAY/gRPC) `#practice #distributed`