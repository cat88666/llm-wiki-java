# Wiki Log

## [2026-05-06] Ingest | 面经实战模块（L8 工程实践）
- 来源：`raw/note/Hollis/面经实战/`（55 个面经实战笔记，按高频考点组合提取）
- 新建 synthesis 页 × 3：
  - `设计-秒杀系统.md`（多层漏斗架构、Redis Lua原子扣减防超卖、MQ削峰填谷、热点行库存拆分、全链路幂等Token+唯一键+消息ID）
  - `设计-分布式事务.md`（2PC同步阻塞对比TCC补偿最终一致、TCC空回滚/悬挂/幂等三大问题事务状态表解决、本地消息表vs事务消息RocketMQ半消息机制、Seata AT undo log原理、选型决策树）
  - `设计-分库分表.md`（分片键选择原则、Hash取模为何先hash再取模、雪花算法64位结构与时钟回拨4种解法、基因法订单号携带分片信息、双写迁移扩容方案）
- 更新 `wiki/index.md`：Synthesis 分区填充 L8 工程实践 3 个 synthesis 链接

## [2026-05-06] Ingest | SpringCloud模块（L7 分布式体系）
- 来源：`raw/note/Hollis/SpringCloud/`（27 个问答式笔记）
- 新建 concept 页 × 1：
  - `机制-微服务与SpringCloud.md`（现代组件选型全景、SpringCloud vs Dubbo对比、Eureka AP vs ZK CP注册中心选型、Nacos替代方案、OpenFeign声明式HTTP+动态代理原理、熔断器三态关闭/开启/半开、Hystrix vs Sentinel多维度限流、Gateway WebFlux非阻塞响应式 vs Zuul阻塞、Nginx vs Gateway定位对比与组合使用）
- 更新 `wiki/index.md`：L7 分布式体系新增 1 个 concept 链接

## [2026-05-06] Ingest | Dubbo模块（L7 分布式体系）
- 来源：`raw/note/Hollis/Dubbo/`（17 个问答式笔记）
- 新建 concept 页 × 1：
  - `机制-RPC与Dubbo.md`（RPC vs HTTP维度对比、Dubbo架构三阶段服务注册/发现/调用、动态代理透明调用9层架构、5种负载均衡策略、dubbo协议+Hessian2序列化、Dubbo SPI懒加载+注入+Wrapper vs JDK SPI、服务治理Failover/Failfast/金丝雀/降级、优雅停机JVM shutdown hook+Spring事件链）
- 更新 `wiki/index.md`：L7 分布式体系新增 1 个 concept 链接

## [2026-05-06] Ingest | ElasticSearch模块（L7 分布式体系）
- 来源：`raw/note/Hollis/ElasticSearch/`（13 个问答式笔记）
- 新建 concept 页 × 1：
  - `机制-倒排索引与ElasticSearch.md`（倒排索引建立过程、ES快的5维度、集群角色Master/Data/Coordinating、乐观锁seq_no+primary_term、不支持事务、深度分页from+size→scroll→search_after、ES与DB一致性binlog监听方案推荐、Hot-Warm-Cold架构）
- 更新 `wiki/index.md`：L7 分布式体系新增 1 个 concept 链接

## [2026-05-06] Ingest | RabbitMQ模块（L7 分布式体系）
- 来源：`raw/note/Hollis/RabbitMQ/`（8 个问答式笔记）
- 新建 concept 页 × 1：
  - `机制-消息队列可靠性.md`（AMQP架构、Exchange四种类型、消息可靠性三段保证、持久化+Publisher Confirm+消费者手动ACK、死信队列、延迟消息DLQ+TTL vs 插件、幂等一锁二判三更新、镜像集群/Quorum Queue高可用）
- 更新 `wiki/index.md`：L7 分布式体系新增 1 个 concept 链接

## [2026-05-06] Ingest | Zookeeper模块（L7 分布式体系）
- 来源：`raw/note/Hollis/Zookeeper/`（9 个问答式笔记）
- 新建 concept 页 × 1：
  - `机制-ZAB协议与Zookeeper.md`（ZNode四种类型、Watch一次性通知机制、ZAB原子广播协议、CP顺序一致性、集群角色Leader/Follower/Observer、选举遵强投强、临时顺序节点分布式锁避免惊群、脑裂与过半写防护）
- 更新 `wiki/index.md`：L7 分布式体系新增 1 个 concept 链接

## [2026-05-06] Ingest | Spring模块（L6 应用框架）
- 来源：`raw/note/Hollis/Spring/`（55 个问答式笔记）
- 新建 concept 页 × 4：
  - `机制-IoC容器.md`（控制反转/DI原理、Bean生命周期11步、三级缓存解决循环依赖机制、为何需要三级缓存、Bean作用域）
  - `机制-AOP织入.md`（AOP 5大术语、JDK/CGLIB代理选择策略、5种Advice类型、失效场景完整列表、解决自调用方案）
  - `机制-Spring事务.md`（@Transactional=TransactionInterceptor Around Advice、7种传播机制及场景、失效10种场景、多线程下事务不传播的根因）
  - `机制-SpringBoot自动装配.md`（@EnableAutoConfiguration链路、spring.factories→Boot3 .imports演变原因、@Conditional条件化装配、自定义starter三步法）
- 新建 summary 页 × 1：`主题-Spring体系.md`（L6 Spring全景知识图 + 高频考点 + 常见误区）
- 更新 `wiki/index.md`：L6 应用框架分区填充 4 个 concept 链接；Summaries 新增 1 条

## [2026-05-06] Ingest | MyBatis模块（L5 存储层）
- 来源：`raw/note/Hollis/MyBatis/`（16 个问答式笔记）
- 新建 entity 页 × 1：
  - `实体-MyBatis.md`（工作原理、# vs $占位符、一/二级缓存对比与不推荐二级缓存的原因、PageHelper/RowBounds/MyBatis-Plus分页原理、插件责任链、延迟加载代理机制）
- 更新 `wiki/index.md`：Entities 分区 L5 新增 1 条

## [2026-05-06] Ingest | 集合类模块（L4 数据结构补充）
- 来源：`raw/note/Hollis/集合类/`（32 个问答式笔记）
- 新建 concept 页 × 2：
  - `机制-HashMap底层实现.md`（数组+链表+红黑树、扰动hash、0.75负载因子、JDK7头插/死循环、JDK8高低位拆分扩容）
  - `机制-ConcurrentHashMap并发设计.md`（分段锁JDK7→CAS+节点锁JDK8、为何用synchronized非ReentrantLock、fail-safe弱一致性、不允许null的二义性、COW机制）
- 更新 concept 页 × 1：
  - `概念-线性数据结构.md`：补充 ArrayList/LinkedList/Vector 对比表、subList陷阱、遍历删除安全写法
- 新建 summary 页 × 1：`主题-Java集合框架.md`（集合体系全图、高频考点、选型决策树、常见误区）
- 更新 `wiki/index.md`：L4 新增 2 个 concept 链接；Summaries 新增 1 条

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

## [2026-05-02] Update | 清理 Notion 导出文件名 ID
- 清理 `raw/note/` 下 Notion 导出产生的 32 位十六进制文件名/目录名后缀
- 同步更新 wiki 来源路径、raw 笔记目录页中的普通链接和 URL 编码链接
- 修正 tuling 目录页中受本次路径变更影响的相对链接
- 验证 raw 文件名 ID 残留为 0，wiki 来源路径缺失为 0，raw 本地 markdown 链接缺失为 0
