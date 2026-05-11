# Wiki Log

## [2026-05-11] Ingest | 模拟面试 — Eson 简历 → synthesis/mock-interview-eson.md
- 触发：用户基于 `raw/note/Interview/Eson.md` 生成约 1 小时结构化模拟面试
- **新建** `synthesis/mock-interview-eson.md`（#practice）
  - 7 阶段 60 分钟面试脚本：暖场/项目深挖 A+B+C/基础问答/系统设计/行为
  - 项目深挖 A（Seata TCC）：TCC vs AT vs Saga 选型理由、try 阶段 SQL 原子更新、幂等唯一索引、悬挂/空回滚防护
  - 项目深挖 B（JVM 0 FGC）：MAT Heap Dump 分析 → 对象池 → G1 调参（IHOP 35%）→ P99 结算 800ms→100ms RocksDB 兜底
  - 项目深挖 C（CoinsOTC）：冷热钱包 PSBT 架构、RedLock + DB 幂等双兜底、充提款异步对账
  - 基础问答：AQS CLH 队列 + 公/非公平锁差异、volatile 复合操作不原子、MVCC 快照读/当前读幻读边界、分片键选择 + 跨分片聚合、Nacos AP/CP 分场景选型、RocketMQ vs Kafka 对比表
  - 系统设计（Gacha）：Redis Lua 原子扣减 + 本地消息表 + 幂等兜底 + Redis 宕机处理
  - 附：面试官评分参考表（6 维度权重）
- **更新** `wiki/index.md`：Synthesis 新增模拟面试条目

## [2026-05-09] Ingest | 彩票系统岗位专项：彩票核心设计/账户钱包/Kafka大流量调优
- 触发：用户提供高级Java/架构师（彩票系统）岗位 JD，分析 wiki 缺口后摄入
- **新建** `synthesis/设计-彩票系统核心设计.md`（#practice #distributed）
  - 投注系统：幂等下单5步、乐观锁余额冻结（vs分布式锁选型分析）、限号玩法Redis Lua预扣减
  - 开奖系统：ACCEPTING→CLOSED→DRAWING→DRAWN→SETTLED状态机、承诺-揭示VRF公平性、第三方开奖源、开奖结果防篡改（链上存证）
  - 结算系统：固定赔率 vs 彩池模型对比表、Kafka分区驱动批量结算（100万注单~1-2分钟）、结算幂等唯一索引
  - 风控：5类异常投注检测规则、设备指纹、套利行为识别
- **新建** `synthesis/设计-账户钱包系统.md`（#practice #storage）
  - 余额三态模型（available/frozen/total）+ 彩票场景完整流转（投注/结算/充值/提现）
  - 并发扣减三方案对比（行锁/乐观锁/分布式锁）+ 彩票场景推荐乐观锁理由
  - append-only流水表设计（SQL含balance_after快照+txn_id唯一索引）+ 余额一致性校验SQL
  - 充值链路：异步回调二次确认 + 幂等防重复入账
  - 提现链路：先冻结后审核后出款 + 出款幂等（withdraw_id）
  - 日终T+1对账 + 实时监控 + 超时扫单兜底
- **更新** `concepts/08-distributed/机制-Kafka.md`：新增「大流量实战补充」章节
  - 生产者调优参数表（linger.ms/batch.size/compression.type/buffer.memory）+ 吞吐公式
  - 消费者调优参数表（fetch.min.bytes/max.poll.records/max.poll.interval.ms）
  - 消息积压处理策略（扩Consumer/扩Partition/并行消费+按key分桶保序）
  - 开奖广播Fanout模式（1 Topic → N Consumer Group各自独立消费）
  - 重平衡期不丢消息：onPartitionsRevoked回调中commitSync
- **更新** `wiki/index.md`：Synthesis新增2个条目

## [2026-05-09] Ingest | P2+P3批次：DDD落地/P8边界问题/云原生选型
- 来源：raw/note/tuling/DDD架构.md + raw/note/Hollis/面经实战/（55文件，抽样10个高经验面经）+ raw/note/Hollis/云计算/（6文件全量）
- **新建** `synthesis/设计-DDD落地实战.md`（#practice #framework）
  - 限界上下文→微服务拆分：5种上下文映射关系（防腐层是关键）
  - 聚合根粒度权衡表：太小（分布式事务）vs 太大（热点并发）
  - 四层架构职责清单：仓储接口在领域层，实现在基础设施层（依赖倒置）
  - 领域事件两种传播：Spring Event（同进程）→ MQ（跨服务）
  - CQRS 适用场景：读写比极不对称、跨聚合查询
  - 常见落地坑：贫血模型惯性/领域层框架污染/聚合过重
- **新建** `synthesis/设计-P8高频边界问题.md`（#practice）
  - 9大追问模式：系统边界职责/造轮子五问/库存三态旁路验证/雪花算法边界/MQ可靠性深追/设计追问/方案质疑/技术选型四维/GC追问
  - 雪花算法边界：Hutool workerid上限31，超32台机器解法，无时钟回拨仍可重复
  - 库存三态：可售/冻结/已扣减；TCC各阶段映射；Confirm后Cancel→反向补偿
  - MQ核心：异步刷盘无法100%不丢；设计MQ需覆盖存储/可靠性/语义/推拉/顺序5维度
- **新建** `synthesis/设计-云原生选型.md`（#practice #distributed）
  - IaaS/PaaS/SaaS/Serverless交付模型对比表（厨房类比）
  - 公有云/私有云/混合云适用场景、云爆发机制
  - Java冷启动问题：JVM 3-10s vs GraalVM AOT < 100ms
  - 无状态设计原则：本地缓存→Redis/本地文件→OSS/Session→JWT
  - 云原生改造6要点（配置/日志/健康检查/链路追踪/优雅停机/服务发现）
- **更新** `wiki/index.md`：Synthesis新增3个条目

## [2026-05-08] Ingest | P0+P1批次：项目难点/大厂实践/三高体系/分布式深水区
- 来源：raw/note/Hollis/项目难点&亮点/（19文件）/ 大厂实践/（3文件）/ 高并发&高可用&高性能目录（25+文件）/ 分布式/（3PC/拜占庭/NewSQL/Leaf 4文件）
- 策略：P0优先，synthesis聚合项目亮点和大厂实践；P1聚合三高体系；更新已有分布式理论页面
- **新建** `synthesis/设计-项目难点表达.md`（#practice）
  - STAR框架核心用法 + 亮点挖掘维度表（并发/缓存/IO/架构/数据库）
  - 10个量化STAR模板：CompletableFuture(10s→1s)/状态机+乐观锁(0超卖)/BitSet预约(400x压缩)/ZSet排行榜(float分数编码)/本地消息表(最终一致)/XXL-JOB分片+userId反转/TTL上下文传播/Spring Event(200k→1k扫描)/Seata TCC/CompletableFuture+Semaphore
- **新建** `synthesis/设计-大厂秒杀实践.md`（#practice #storage #distributed）
  - 阿里Inventory Hint：Leader/Follower分组、Row Cache、组提交三大优化
  - 小红书合并秒杀：全局缓存+Leader-Follower状态机，TPS ~200→4276→23543（5.5x）
  - 决策树：并发量梯度方案（普通MySQL/Inventory Hint/分桶/近端缓存）
- **新建** `summaries/主题-三高体系.md`（#practice #distributed #concurrency）
  - 三高制约矩阵、4种限流算法对比（漏桶/令牌桶/固定窗口/滑动窗口/自适应）
  - 熔断器三态、SLA表（2~5个九）、全链路压测方法论、读写分离主从延迟处理
  - 布隆vs布谷鸟过滤器、线程池调优公式、三高联动场景分析
- **更新** `concepts/08-distributed/概念-分布式系统理论.md`（4个新章节）
  - 3PC：三阶段+超时机制，致命缺陷（网络分区超时自提交导致不一致）
  - 拜占庭将军问题：n≥3f+1，PoW/PBFT/Raft对比，区块链应用
  - NewSQL选型：MySQL+分库 vs TiDB(LSM/Raft/HTAP) vs OceanBase(Paxos/金融级)
  - Leaf ID生成器：号段模式(双Buffer预取) + Snowflake模式(ZK解决时钟漂移)
- **更新** `wiki/index.md`：Summaries新增 主题-三高体系；Synthesis新增 设计-项目难点表达/设计-大厂秒杀实践

## [2026-05-08] Ingest | 德州扑克核心算法：深度代码分析 → 1个synthesis页
- 来源：dx-game-texas 项目（card/、CardInsureSmartOutsService.java、ProvisionalSettlementService.java）
- 策略：深度读取核心算法代码（~3000行），按四层结构编译
- **新建** `synthesis/设计-德州扑克核心算法.md`（#practice）
  - 层1 牌型评估：工厂模式三变体评估器、10级牌型层级、多项式系数编码O(1)比较
  - 层2 胜率/Outs：分阶段策略（Pre-Flop查表/Flop枚举C(47,2)=1081/River精确）、反超Outs vs 平分Outs
  - 层3 智能保险：SmartInsureOuts数据结构、四种投保方案、方程组建立与迭代求解（最多20次）、背回逻辑
  - 层4 底池结算：边池生成公式、比牌赢家算法、平分余数处理、按底池/盈利两种抽水模式、三种保险模式结算差异
  - 游戏变体差异对比表（标准德州/短牌6+/奥马哈）
  - 面试4大问答模板
- **更新** `wiki/index.md`：Synthesis区新增 设计-德州扑克核心算法 条目

## [2026-05-08] Ingest | 实际项目代码：5个微服务 → 1个synthesis页
- 来源：dx-net-bolt / dx-net-gateway / dx-game-hall / dx-game-texas / dx-game-race-hall（5个项目代码库）
- 策略：深度探索代码结构和核心类，提取架构亮点，聚合为一个综合synthesis页
- **新建** `synthesis/设计-在线游戏平台架构.md`（#practice #distributed）
  - 覆盖：整体架构图（5个服务依赖关系、通信路径）
  - 亮点1：用户级线程隔离（userId → SingleThreadExecutor，心跳独立线程池）
  - 亮点2：工厂模式服务路由（InitializingBean扫描+ServiceTypeEnum O(1)路由，开闭原则）
  - 亮点3：协议智能转换（同一接口按参数动态路由不同后端+RaceForwardCache记录转换）
  - 亮点4：游戏状态机+分布式任务调度（任务链+Redis eventKey，多桌并发+超时自动推进）
  - 亮点5：保险赔率实时计算（组合数学枚举C(47,2)=1081种排列，手牌级缓存，<10ms）
  - 亮点6：三级缓存（Guava L1 + Redis L2 + MySQL L3）
  - 亮点7：Netty高性能配置（内存池、TCP_NODELAY、主从Reactor、IdleStateHandler）
  - 技术栈全景表 + 关键权衡 + 面试STAR法则回答模板
- **更新** `wiki/index.md`：Synthesis区新增 设计-在线游戏平台架构 条目

## [2026-05-08] Ingest | 架构设计目录：13个文件 → 1个synthesis页
- 来源：raw/note/Hollis/架构设计/（13个文件，全部读取）
- 策略：13个问答聚合为一个P7架构思维综合页
- **新建** `synthesis/设计-架构设计思维.md`（#practice #distributed）
  - 覆盖：架构权衡本质、没有银弹哲学、好架构标准、SOLID+架构原则
  - 覆盖：技术债管理、微服务拆分7原则（康威定律）、技术选型框架
  - 覆盖：单元化架构（优势/代价）、MVC vs 三层、介绍项目架构4维度、亿级商品存储
- **更新** `wiki/index.md`：Synthesis区新增 设计-架构设计思维 条目

## [2026-05-08] Ingest | P7 场景题批次：6个synthesis综合页
- 触发：用户要求直接处理 raw/note/Hollis/场景题/ 目录（134个文件）
- 策略：按主题聚合，不单独为每题建页，覆盖 ~70 个场景题文件
- **新建** `synthesis/设计-MySQL大表与查询优化.md`（#storage #practice）
  - 来源：场景题/ 大表DDL、深分页、热点行、长事务、逻辑删除唯一约束等
  - B+树视角2000w分表阈值（树高3层，填充率2/3）
  - 深分页3种优化（游标/子查询覆盖索引/倒序）
  - OnlineDDL vs pt-osc vs gh-ost 三方案对比
  - 热点行更新六大危害 + 长事务锁包事务编程模式
- **新建** `synthesis/设计-海量数据处理.md`（#data-structure #practice）
  - 来源：场景题/ 40亿QQ号、布隆过滤器、1TB排序、TopK关键词、黑名单URL、骚扰电话等
  - BitMap选型速查表 + 40亿QQ内存计算（476M vs 1192M两种场景）
  - Bloom filter公式推导 + 5亿数据误判率1%=571MB估算
  - 外部归并排序：分块→块内排序→多路归并（小顶堆）全流程
  - AC自动机原理 + 黑名单URL多方案对比
- **新建** `synthesis/设计-接口设计与防护.md`（#distributed #practice）
  - 来源：场景题/ 频率限制、防重复点击、防刷流量、三方超时、支付超时、Token窃取、拉黑等
  - Redis ZSet滑动窗口 + Lua原子锁定（登录5分钟3次限制）
  - Token机制 vs 幂等Key两种防重方案
  - 三方接口3种隔离（异步收单/超时降级/熔断隔离）
  - 支付超时处理：查单优先不直接认为失败
- **新建** `synthesis/设计-消息队列场景综合.md`（#distributed #practice）
  - 来源：场景题/ 乱序、Kafka吞吐、SpringEvent、推拉模式、订单超时、BlockingQueue、本地消息表等
  - Spring Event vs MQ对比表（作用范围/事务/可靠性）
  - 消息乱序4种解法：顺序消息/前置状态/序列号/事件表+定时重试
  - 本地消息表：上游不回滚，下游指数退避重试至最终一致
  - 消息表完整SQL schema（含唯一索引+state+retry_count）
- **新建** `synthesis/设计-容量规划与性能基准.md`（#practice #jvm）
  - 来源：场景题/ 4C8G指标、机器选型、QPS预估、压测vs线上、启动飙高、内存增长、频繁FGC等
  - 4C8G正常指标基准表（CPU/Load/内存/YoungGC/FullGC 9指标×3档）
  - 机器数公式：单机QPS=线程数×1000/RT，例题3000QPS/200ms=6~9台
  - 压测600线上300扛不住：8大原因排查清单（数据倾斜/定时任务/缓存未预热/JIT未预热等）
  - 堆外内存泄漏：NMT追踪工具链（DirectBuffer/JNI/Metaspace）
  - 银行系统GC：ZGC(JDK15+) STW<1ms
- **新建** `synthesis/设计-分布式场景综合.md`（#distributed #practice）
  - 来源：场景题/ 三种锁对比、库存扣减、分布式锁并发、Session共享、SSO、订单号生成、跨库JOIN、数据迁移、定时任务、Excel导入等
  - 三种锁选型对比表（乐观/悲观/分布式）+ 选型决策流程
  - 高并发库存3方案：DB乐观锁/Redis预扣异步落盘/库存拆分
  - Session演进：粘性→Redis集中→JWT无状态
  - 两网站SSO：CAS/OAuth2+OIDC全流程（Ticket一次性令牌）
  - 平滑数据迁移：双写→追平→灰度切流→停老库 + 回滚策略
- **更新** `wiki/index.md`：Synthesis区新增上述6个页面条目

## [2026-05-08] Ingest | P7 优先级第二轮：Seata/SpringMVC/SpringBoot启动流程/OAuth2
- 触发：用户请求继续处理第二优先级模块
- **新建** `concepts/08-distributed/机制-Seata框架机制.md`（L7 #distributed）
  - 来源：raw/note/Hollis/分布式/ Seata 相关5个文件
  - TC/TM/RM 三组件架构及全局事务5步流程
  - AT 模式：代理数据源 + UNDO_LOG(before/after image)一阶段本地提交 + 二阶段清理/回滚
  - TCC 空回滚/悬挂问题与分布式事务记录表统一解法
  - 四种模式对比表（AT/TCC/Saga/XA）及 AT vs XA 核心区别
- **新建** `concepts/07-framework/机制-SpringMVC请求处理链.md`（L6 #framework）
  - 来源：raw/note/Hollis/Spring/ SpringMVC 路由文件
  - 路由注册：@RequestMapping → RequestMappingInfo → MappingRegistry
  - 请求处理5步：HandlerMapping → preHandle → HandlerAdapter(参数解析+反射+返回值) → postHandle → ViewResolver
  - ExceptionResolver 责任链 + 设计模式全景（适配器/组合/策略/责任链/门面）
- **新建** `concepts/08-distributed/概念-OAuth2授权协议.md`（L7 #distributed）
  - 来源：raw/note/Hollis/分布式/ OAuth2 文件
  - Client/ResourceServer/AuthorizationServer 三角色、Access Token 授权流程
  - 四种授权类型（授权码/隐式/密码/客户端凭证）及安全边界
  - OAuth2 vs OIDC 区别（授权 vs 认证）
- **更新** `concepts/07-framework/机制-SpringBoot自动装配.md`
  - 启动流程从"精简版"扩展为"完整版"：new SpringApplication 5步 + run 完整阶段（含 Tomcat 启动位置）
  - 新增"优雅停机"：server.shutdown=graceful + timeout-per-shutdown-phase 配置
  - 新增 source：SpringBoot如何做优雅停机
- **更新** `wiki/index.md`：新增 Seata、OAuth2、SpringMVC 条目；更新 SpringBoot 自动装配描述

## [2026-05-08] Ingest | P7 优先级第一轮：消息队列/配置中心/任务调度/异步编程/JVM调优
- 触发：用户请求按 P7 缺口分析优先级摄入 raw/ 中高优先级目录
- **新建** `concepts/08-distributed/机制-RabbitMQ.md`（L7 #distributed）
  - 来源：raw/note/Hollis/RabbitMQ/ 全部10个文件
  - AMQP 三层路由、6种工作模式、三道防线（Confirm+持久化+ACK）
  - 死信队列两种延迟消息方案（TTL vs 插件）、镜像集群高可用
- **新建** `concepts/08-distributed/机制-配置中心与Nacos.md`（L7 #distributed）
  - 来源：raw/note/Hollis/配置中心/ 全部8个文件
  - 配置中心作用与选型表（Nacos/Apollo/Consul/ZK）
  - Nacos 双协议（JRaft CP + Distro AP）、长轮询→gRPC演进、推拉结合服务发现
- **新建** `concepts/08-distributed/机制-分布式任务调度.md`（L7 #distributed）
  - 来源：raw/note/Hollis/定时任务/ 全部11个文件
  - 小顶堆 vs 时间轮数据结构对比、XXL-Job DB悲观锁唯一触发
  - 分片任务原理（ShardIndex/ShardTotal）、定时扫表三缺陷与解法、退避策略
- **新建** `concepts/04-concurrency/机制-CompletableFuture与异步编程.md`（L3 #concurrency）
  - 来源：raw/note/Hollis/Java并发/ CompletableFuture 文件
  - Completion 链事件驱动底层、thenApply/thenCompose/allOf/anyOf API 分类
  - I/O 密集型必须自定义线程池（ForkJoinPool 陷阱）
- **新建** `synthesis/设计-JVM调优实战.md`（#jvm #practice）
  - 来源：raw/note/Hollis/JVM/ GC调优相关文件
  - GC 正常基线指标（经验值）、频繁FullGC/OOM分类排查流程
  - Arthas/jmap/jstack 工具链、G1/ZGC 参数速查、OOM 不导致 JVM 退出机制
- **更新** `concepts/07-framework/机制-IoC容器.md`
  - 补充：SpringBoot 2.6 默认禁止循环依赖、@Lazy 解决构造器注入循环依赖、三级缓存 vs 二级缓存的根本原因
  - 新增3个 sources 引用
- **新建** `summaries/主题-消息队列体系.md`
  - Kafka/RabbitMQ/RocketMQ 全景对比表、三道防线通用模式、消息幂等方案、选型决策树
- **更新** `wiki/index.md`：新增3个 concepts、1个 synthesis、1个 summary 的索引条目

## [2026-05-07] Ingest | Interview/Eson.md → 4个缺失知识点补充
- 触发：检查简历核心知识点与 wiki 的差距，发现 4 处完全缺失
- 新建 `concepts/03-jvm/机制-对象池技术.md`（L2 #jvm）
  - HikariCP ConcurrentBag 无锁设计、Netty PooledByteBufAllocator jemalloc 分层内存
  - 对象池与 G1 GC 协同实现 0 FGC 的完整路径
  - 关键权衡：状态重置、池满策略、minIdle 保活代价
- 新建 `concepts/06-storage/机制-LSM树与RocksDB.md`（L5 #storage）
  - WAL + Memtable + SSTable + Compaction 四层结构
  - 写放大/读放大/空间放大三大权衡因子对比表（LSM vs B+树）
  - Leveled/Size-Tiered/FIFO 三种 Compaction 策略选型
  - RocksDB 在 TiKV/Flink State Backend/游戏兜底存储的应用场景
- 新建 `synthesis/设计-多级缓存架构.md`（L8 #practice）
  - Caffeine W-TinyLFU vs expireAfterWrite vs refreshAfterWrite 对比
  - TTL/MQ广播/Canal+binlog 三种一致性策略的适用边界
  - 双重检测防雪崩：JVM本地锁→Redis分布式锁→三次重检查完整代码
- 新建 `synthesis/设计-支付系统设计.md`（L8 #practice）
  - 聚合支付架构：统一下单幂等、渠道路由（金额/类型/成功率/成本）
  - TCC 在支付的完整落地：空回滚/悬挂的防护实现
  - 双记账流水表：不可变流水、balance_after 快照、余额可重算验证
  - T+1全量对账 + 实时MQ对账双兜底，差异处理（长款/短款）
  - OTC撮合引擎：Disruptor单线程无锁撮合、冷热钱包分离架构
  - 资损防控三道防线：代码层幂等→DB层约束→监控对账层告警

## [2026-05-07] Lint | 全库健康检查 + 修复
- **CHECK 1 (index.md路径)**：OK — 86条引用全部有效
- **CHECK 2 (孤立页面)**：OK — 无孤立concepts页面
- **CHECK 3 (内部链接)**：修复5处断链
  - `synthesis/设计-线上问题排查.md`：`[[概念-JVM内存与GC]]`→`[[机制-JVM内存模型]]/[[机制-GC算法与垃圾收集器]]`，`[[概念-MySQL索引]]`→`[[机制-InnoDB索引模型]]`，`[[机制-锁与并发控制]]`→`[[机制-InnoDB锁机制]]`
  - `summaries/主题-Java并发体系.md`：ASCII art箱图内链接跨行→修复为单行
- **CHECK 4 (sources路径)**：修复4个concept文件的sources路径（文件名与实际不符）
  - `机制-容器化与Docker.md`：补全✅前缀及正确文件名（6条）
  - `概念-网络安全.md`：修正9条paths（文件名变化：CSRF+XSS合并为一文件等）
  - `概念-限流与熔断.md`：补全✅前缀（7条）
  - `概念-高可用设计.md`：补全✅前缀（8条）
  - `synthesis/设计-算法高频题型.md`：层级错误 `../../../raw/` → `../../raw/`
- 当前状态：concepts×69 / summaries×8 / synthesis×8 / entities×1

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

## [2026-05-07] Ingest | 架构体系/ → 3新建 + 3更新
- 新建 `concepts/02-java-lang/概念-JDK新特性.md`（L1 #java-lang）：覆盖 7个架构体系文件
  - JDK8：Lambda(invokedynamic)/Stream(惰性求值)/Optional/LocalDateTime
  - JDK16 Record：不可变数据类，自动生成构造器/getter/equals/hashCode
  - JDK17 Sealed Classes：限制合法子类，配合 pattern matching switch 穷举类型
  - JDK21 虚拟线程：JVM管理的轻量级线程，IO阻塞时自动卸载载体线程；synchronized 会 pin 载体线程，应改 ReentrantLock
- 新建 `concepts/08-distributed/概念-可观测性.md`（L7 #distributed）
  - 三大支柱：Metrics(是否有问题) / Tracing(问题在哪) / Logging(为什么)
  - Prometheus：Counter/Gauge/Histogram/Summary 四种类型；P99/CPM/错误率
  - SkyWalking：Agent无侵入字节码插桩 → OAP Server → ES存储；TraceId/Span/Segment
  - 三支柱协作排障流程（Grafana发现 → SkyWalking定位 → Kibana根因）
- 新建 `synthesis/设计-算法高频题型.md`（L8 #practice）
  - LRU: HashMap+双向链表，O(1) get/put
  - TopK: 小顶堆O(n log k) / 快速选择O(n)
  - 海量数据: 哈希分片+局部处理+合并结果模式
  - 排序: TimSort/双轴快排/快速排序优化(随机pivot/三路划分)
  - 二分: 标准模板+变体(lower_bound/旋转有序)
  - 双指针/滑动窗口: 对撞/快慢/窗口三种模式
  - DP: 最大子数组/LIS/LCS/0-1背包四经典转移方程
  - 图: 并查集路径压缩代码 + 拓扑排序Kahn算法
- 更新 `concepts/08-distributed/概念-分布式系统理论.md`：补充 Paxos/Raft
  - Raft三子问题：Leader选举(多数票+任期Term防脑裂) / 日志复制(多数确认后commit) / 安全性(新Leader必须有最新committed日志)
  - Raft应用：etcd/TiKV/CockroachDB/Consul
- 更新 `concepts/08-distributed/机制-RPC与Dubbo.md`：新增 gRPC + 序列化对比表
  - gRPC: HTTP/2(多路复用/头部压缩) + Protobuf；4种通信模式(Unary/双向Streaming等)
  - 序列化对比: JSON/Hessian2/Protobuf/Kryo/Fury/Avro 性能/跨语言/场景
- 更新 `concepts/09-practice/概念-DDD.md`：新增 CQRS + Event Sourcing
  - CQRS: 写路径走领域模型，读路径走专用查询模型(反范式)
  - Event Sourcing: 存事件序列而非当前状态；审计/时间旅行优点；Snapshot缓解重放成本
- 知识库现状：concepts×70 / entities×1 / summaries×8 / synthesis×8

## [2026-05-07] Ingest | 项目经验/ → synthesis/设计-线上问题排查
- 新建 `synthesis/设计-线上问题排查.md`（L8 #practice）：覆盖 25 个实战案例文件
  - 诊断工具速查表：Arthas(thread/trace/watch/sc)/top/jstack/jmap/MAT/df/du/netstat/tcpdump
  - CPU 飙高：BeanValidator重复初始化/Sequence单例丢失/logback AsyncAppender配置不当；死循环=CPU高，死锁=CPU不升高（WAITING让出CPU）
  - Load 飙高：JIT预热期 → 解释执行导致，解法：小流量切流/JwarmUp/缓存预热
  - FullGC 三类案例：AviatorEvaluator未缓存(MetaSpace耗尽)；数据倾斜(22开头用户量远超均值)；接口未做非空校验全量查询60万条
  - OOM 两类案例：分治回溯算法空间复杂度爆炸；POI XSSFWorkbook全量加载→改SXSSFWorkbook
  - 慢SQL四类：最左前缀未命中(type=index)；索引区分度低导致回表；缺覆盖索引；filesort用索引排序
  - 数据库死锁根因：前缀索引截断相同 + 同一事务内两条UPDATE走不同索引→加锁顺序不一致→循环等待
  - 数据库连接池满：热点行高并发UPDATE→InnoDB行锁排队耗尽连接→合并批量更新
  - 数据库CPU打满：分布式任务并发修改同一审核单→预合案+批量更新(任务耗时2h→10min)
  - RocketMQ消费堆积：Spring Cloud Stream + Native混用同一ConsumerId→订阅关系被覆盖→修改ConsumerId
  - Java进程挂了：假死(死锁/活锁/IO阻塞/GC暂停) vs 真挂(OOM/资源满/kill -9)
  - 服务器挖矿木马：kswapd0 CPU高 + netstat发现境外IP → 删文件/kill进程/清crontab
- 更新 `index.md`：Synthesis新增 设计-线上问题排查
- 知识库现状：concepts×67 / entities×1 / summaries×8 / synthesis×7

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
