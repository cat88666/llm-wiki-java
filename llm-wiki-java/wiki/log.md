# Wiki Log

## [2026-05-07] Ingest | tuling/ → 机制-Kafka + 机制-Netty + 概念-DDD + Nacos补充
- 新建 `concepts/机制-Kafka.md`（L7 #distributed）：覆盖 tuling/08-mq/Kafka.md + Kafka子目录6个文件
  - 核心架构：Topic→Partition→Segment(.log/.index/.timeindex)，Producer/Broker/ConsumerGroup
  - 为什么快：顺序写+零拷贝(sendfile)+批处理+页缓存+稀疏索引
  - 可靠性：acks(0/1/-1)、ISR机制、HW高水位/LEO/Leader Epoch防数据回滚
  - 重平衡：3个触发条件、5种状态、CooperativeStickyAssignor渐进式重平衡(2.4+/4.0升级)
  - 顺序消费：单分区有序、3种指定分区方式
  - 消费语义：At-least-once(默认)/Exactly-once(事务)
  - vs RocketMQ vs RabbitMQ功能对比表
- 新建 `concepts/机制-Netty.md`（L7 #distributed）：覆盖 tuling/07-Netty.md
  - IO模型对比表(BIO/NIO/AIO/select/poll/epoll)
  - 主从Reactor多线程模型（BossGroup+WorkerGroup）
  - 核心组件：EventLoop单线程无锁、ChannelPipeline责任链双向链表
  - ByteBuf：读写双指针/池化4种类型/零拷贝(DirectBuf/CompositeBuf/FileRegion)
  - 粘包拆包3种解码器（FixedLength/Delimiter/LengthField）
  - IdleStateHandler心跳机制、完整链路图
  - 高性能5点：无锁串行/内存池/零拷贝/MPSC队列/epoll边缘触发
- 新建 `concepts/概念-DDD.md`（L8 #practice）：覆盖 tuling/DDD架构.md
  - 实体/值对象/聚合/聚合根/领域服务/领域事件核心概念及Java代码示例
  - 充血模型vs贫血模型对比表（行为归属/代码维护/测试/适用场景）
  - 四层架构（用户接口→应用→领域→基础设施，严格单向依赖）
  - 洋葱架构/六边形架构变体
  - 统一语言（Ubiquitous Language）
  - 限界上下文→微服务边界理论依据
- 更新 `concepts/机制-微服务与SpringCloud.md`：新增「Nacos注册中心与配置中心机制」章节
  - 注册生命周期(5s心跳/15s unhealthy/30s剔除)
  - 雪崩保护阈值
  - Namespace/Group/DataId三元组及优先级
  - @RefreshScope长轮询动态配置
- 更新 `index.md`：L7新增 机制-Kafka/机制-Netty/概念-DDD
- 源文件覆盖：tuling/08-mq/Kafka.md(+6子文件) / tuling/07-Netty.md / tuling/DDD架构.md / tuling/09-微服务/11-nacos.md
- 知识库现状：concepts×67 / entities×1 / summaries×8 / synthesis×6

## [2026-05-07] Ingest | 网络安全/ + 微服务/ → 概念-网络安全 + 机制-微服务与SpringCloud 补充
- 新建 `concepts/概念-网络安全.md`（L7 #security）：覆盖 13 个原始文件
  - 密码学基础（MD5 非加密/SHA-256/加密 vs 加签流程）
  - SQL 注入与预编译防御（PreparedStatement/MyBatis #{}/最小权限）
  - XSS（输出编码/CSP/HttpOnly）vs CSRF（CSRF Token/SameSite Cookie）对比表
  - 垂直越权（RBAC+接口鉴权）vs 水平越权（Session取userId/Sqids混淆ID）
  - 中间人攻击（HTTPS/HSTS/证书固定/DNSSEC）
  - 撞库/拖库/洗库三阶段防御
  - DDoS 多层防御（网络/传输/应用/调度）
  - 国密算法对比表（SM4对称/SM2非对称ECC/SM3哈希/SM9 IBE）
- 更新 `concepts/机制-微服务与SpringCloud.md`：新增 5 个章节
  - 限流/降级/熔断本质区别（触发时机 + 作用对象 + 目的对比表）
  - 部署策略（蓝绿 vs 灰度/金丝雀）对比及快速回滚
  - Service Mesh（Sidecar 模式/Istio 控制平面/异构系统统一治理）
  - 循环依赖危害与解法（重设计/MQ解耦/共享服务）
  - CI/CD & DevOps 简要说明
  - 更新 frontmatter：新增 aliases 9 个 + sources 11 个
- 更新 `index.md`：L7新增 概念-网络安全，更新 机制-微服务与SpringCloud 描述
- 知识库现状：concepts×64 / entities×1 / summaries×8 / synthesis×6

## [2026-05-07] Ingest | 场景题/ + 面经实战/ → L8 synthesis ×3
- 新建 `synthesis/设计-Redis实战场景.md`（L8 #practice）：ZSet排行榜(分片/异步/分数编码三大挑战)、GEO查找附近的人、ZSet点赞(按时间顺序)、二倍均值法抢红包、HyperLogLog UV统计、Hash购物车
- 新建 `synthesis/设计-短链服务.md`（L8 #practice）：MurmurHash+Base62生成、302 vs 301取舍(可统计选302)、Redis+MySQL双存储查找流程、防滥用(黑名单/限流/鉴权)
- 新建 `synthesis/设计-订单超时关闭.md`（L8 #practice）：11种方案对比表、定时扫表(小业务)/Redisson延迟队列(分布式首选)/MQ延迟消息对比、关闭逻辑幂等设计
- 更新 `index.md`：Synthesis区新增 Redis实战场景/短链服务/订单超时关闭
- 知识库现状：concepts×63 / entities×1 / summaries×8 / synthesis×6

## [2026-05-07] Ingest | 高性能/ + 架构设计/ → 概念-布隆过滤器 + 概念-读写分离 + 微服务拆分原则
- 新建 `concepts/概念-布隆过滤器.md`（L4 #data-structure）：布隆过滤器原理(bit数组+多哈希)、误判率控制、三种删除方案(定期重建/计数布隆/布谷鸟过滤器)、布谷鸟过滤器指纹+双候选桶+XOR索引+支持删除
- 新建 `concepts/概念-读写分离.md`（L5 #storage）：主从复制基础、三种路由分流(代码/中间件/代理)、主从延迟处理(读请求分类/强制读主库/GTID等主库位点)
- 更新 `concepts/机制-微服务与SpringCloud.md`：新增「微服务拆分原则」章节（职责单一/业务边界/中台化/系统保障/康威定律/单向依赖）
- 源文件覆盖：`高性能/✅什么是布隆过滤器` / `布隆过滤器缺点` / `无法删除如何解决` / `布谷鸟过滤器` / `什么是读写分离` / `读写分离遇到主从延迟` / `架构设计/✅微服务的拆分有哪些原则`
- 更新 `index.md`：L4新增 概念-布隆过滤器，L5新增 概念-读写分离
- 知识库现状：concepts×63 / entities×1 / summaries×8 / synthesis×3

## [2026-05-07] Ingest | 分布式/ → 概念-分布式系统理论 + 概念-幂等设计 + 机制-Canal数据同步
- 新建 `concepts/概念-分布式系统理论.md`（L7 #distributed）：CAP三选二(P不可放弃故为CP vs AP)、一致性模型层次(线性/顺序/最终)、BASE基本可用+软状态+最终一致性、一致性哈希环+虚拟节点解决hash倾斜
- 新建 `concepts/概念-幂等设计.md`（L7 #distributed）：业务幂等vs请求幂等、一锁(Redis tryLock)二判(流水表/状态机)三更新(持久化)、唯一性约束作最后兜底、MQ消费幂等
- 新建 `concepts/机制-Canal数据同步.md`（L7 #distributed）：模拟MySQL slave拉取binlog、ROW格式要求、MySQL→ES/缓存/异构库场景、分库分表买卖家表维护
- 源文件覆盖：`分布式/✅什么是CAP理论` / `BASE理论` / `一致性哈希` / `分布式系统的一致性` / `如何解决接口幂等` / `为什么不建议唯一约束幂等控制` / `什么是Canal`
- 更新 `index.md`：L7新增 概念-分布式系统理论/概念-幂等设计/机制-Canal数据同步
- 知识库现状：concepts×61 / entities×1 / summaries×8 / synthesis×3

## [2026-05-07] Ingest | 设计模式/ → 机制-设计模式.md
- 新建 `concepts/机制-设计模式.md`（L6 #framework）：覆盖 21 个原始文件
- 重点：SOLID+迪米特+合成复用7大原则、单例5种实现(枚举最佳:线程安全+防反序列化)、三种工厂(简单→方法→抽象工厂OCP支持递增)、代理/享元结构型、观察者/模板方法/策略/责任链行为型及Spring具体应用(AOP/事件/Template/FilterChain)
- 更新 `index.md`：L6 新增设计模式条目
- 知识库现状：concepts×58 / entities×1 / summaries×8 / synthesis×3

## [2026-05-07] Ingest | 操作系统/ → 概念-操作系统基础.md
- 新建 `concepts/概念-操作系统基础.md`（#os）：覆盖 30 个原始文件核心知识
- 重点：进程/线程/协程三级对比、用户态/内核态切换机制、零拷贝4种实现(普通→mmap→sendfile→DMA Scatter)、五种IO模型、select/poll/epoll对比(O(n)→O(1))、MESI缓存一致性协议与JMM关系、Page Cache延迟写与fsync、进程调度6种算法(CFS是Linux默认)
- 更新 `index.md`：CS基础区新增操作系统基础
- 知识库现状：concepts×57 / entities×1 / summaries×8 / synthesis×3

## [2026-05-07] Ingest | 计算机网络/ → 概念-计算机网络.md
- 新建 `concepts/概念-计算机网络.md`（#network）：覆盖 30 个原始文件核心知识
- 重点：TCP三次握手/四次挥手原理、可靠传输5机制、拥塞控制4阶段、HTTP版本演进(1.0→1.1→2→3/QUIC)、HTTPS TLS握手(1.2 vs 1.3)、DNS解析链路、Cookie/Session/Token选型、正反向代理与CDN、跨域CORS解法
- 更新 `index.md`：新增「基础知识（CS基础）」分区
- 知识库现状：concepts×56 / entities×1 / summaries×8 / synthesis×3

## [2026-05-07] Ingest | 容器/ + 高并发/ + 高可用/ → L7 概念页 ×3
- 新建 `concepts/机制-容器化与Docker.md`（L7 #devops）：容器vs虚拟机原理(namespace+cgroup)、Dockerfile指令、Docker Compose、K8s核心对象（Pod/Deployment/Service/Ingress）
- 新建 `concepts/概念-限流与熔断.md`（L7 #distributed）：漏桶/令牌桶/滑动窗口算法对比、熔断器三态(Closed→Open→Half-Open)、Sentinel自适应限流、预热策略
- 新建 `concepts/概念-高可用设计.md`（L7 #distributed）：SLA四个九(99.99%=52.56min/year)、冷/热/暖备、异地多活架构要素、全链路压测(流量染色+影子表)
- 修复 `concepts/概念-Java集合框架.md`（远程拉取）两处失效链接：`[[机制-HashMap]]`→`[[机制-HashMap底层实现]]`、`[[概念-集合迭代一致性]]`→`[[机制-ConcurrentHashMap并发设计]]`
- 更新 `index.md`：L4新增 概念-Java集合框架，L7新增 容器化与Docker/限流与熔断/高可用设计
- 知识库现状：concepts×55 / entities×1 / summaries×8 / synthesis×3

## [2026-05-06] Lint | 健康检查
- **失效链接**：修复 `主题-Redis体系.md` 第37行畸形 wikilink（管道符截断了 `[[机制-Redis持久化]]`）
- **重复/孤立页面删除** ×4：`机制-HashMap.md`、`机制-ConcurrentHashMap.md`（均与 `底层实现`/`并发设计` canonical 页重复）、`机制-CopyOnWrite.md`、`概念-集合迭代一致性.md`（内容已覆盖于 `ConcurrentHashMap并发设计.md`）
- **sources 路径**：全量验证 51+4 个页面，0 个缺失
- **概念页六段结构**：全部 51 个 concept 页结构完整
- **index.md 链接**：全部文件存在，0 失效
- **遗留 Demo 文件**（未修改，待人工决定是否删除）：`entities/demo-*.md` ×4 + `synthesis/demo-AI时代...md` ×1
- 知识库现状：concepts×51 / entities×1 / summaries×8 / synthesis×3

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
