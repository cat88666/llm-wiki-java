# Wiki Log

## [2026-05-18] Update | concepts/02-java 快速导航锚点修复（11 页）
- 扫描并修复 `concepts/02-java/*.md` 的“快速导航”跳转问题：统一改为显式锚点 `#sec-*`，避免不同 Markdown 渲染器对中文标题 slug 规则差异导致跳转失效
- 为每个二级标题补充 `<a id=\"sec-N\"></a>`，并按章节顺序重写导航链接目标（如 `#sec-2` 对应正文第一章）
- 覆盖文件：`机制-HashMap`、`机制-Java序列化`、`机制-Lambda`、`机制-SPI`、`机制-动态代理`、`机制-泛型`、`概念-JDK新特性`、`概念-Java异常`、`概念-Java类型`、`概念-Java集合`、`概念-OOP特征`
- 逐文件校验导航目标均存在对应锚点，无缺失项

## [2026-05-18] Update | 8 个 02-java 概念页按 rewrite-concept-prompt 重写优化
- 重写并统一结构：`机制-Lambda.md`、`机制-SPI.md`、`机制-动态代理.md`、`机制-泛型.md`、`概念-Java异常.md`、`概念-Java类型.md`、`概念-JDK新特性.md`、`概念-OOP特征.md`
- 全部页面补充 `sources` 与 `updated: 2026-05-18`，并保留既有 frontmatter 主字段
- 章节统一为“本质问题/机制链路/设计原则/对比结论/生产风险/边界/面试口径”，强化可直接用于评审和面试的表达
- 补强关键实战内容：SPI 类加载器边界、JDK/CGLIB 代理决策、泛型 PECS 与擦除陷阱、异常治理策略、JDK 版本升级路径、类型系统高频事故排查

## [2026-05-18] Update | 机制-Java序列化.md — 按 rewrite-concept-prompt 重写优化
- 保留并增强 frontmatter：补充 `related`（SPI/Redis/Dubbo）、新增 `sources`、更新 `updated: 2026-05-18`
- 章节重构为“本质问题 → 原生机制链路 → UID 兼容 → 字段与钩子 → 协议选型 → 风险防御 → 排障迁移 → 关系边界 → 面试速答”
- 强化可执行内容：新增 UID 变更策略表、协议选型决策树、`InvalidClassException` 排障流程、JDK→Protobuf 迁移步骤
- 补齐高频风险口径：反序列化 RCE、敏感字段泄漏、单例破坏、fastjson AutoType 风险与防御

## [2026-05-18] Update | 概念-JVM.md — 按 rewrite-concept-prompt 重写优化
- 保留 frontmatter（aliases/related/sources），并将 `updated` 更新为 `2026-05-18`
- 章节重组为“职责与代价 → 内存流转 → GC链路 → 类加载隔离 → JIT → 调优闭环 → 选型结论 → 故障模式 → 面试边界”，减少泛化标题
- 强化因果链：对象创建与晋升、三色标记与 CMS/G1 处理差异、ThreadLocal 弱引用陷阱、G1/ZGC 选型条件
- 调优部分改为闭环打法（基线/证据/修复/回归），保留命令与参数速查，突出“先证据后调参”

## [2026-05-16] Update | 主题-模拟面试.md — 新增零章（一面复盘 & 二面补救）
- 一面反馈（Eson）：7 个被打脸的点（语言栈/QPS数据/网络链路/三高/DDD/事务机制/场景题）
- 新增 §0 一面复盘：0.1失分清单 / 0.2语言栈说明 / 0.3QPS数据自检 / 0.4完整网络链路 / 0.5三高项目速查 / 0.6DDD基础口径 / 0.7事务机制InnoDB层 / 0.8场景题百万数据导出完整方案
- 更新薄弱预警：标注"一面暴露"7条，保留既有6条
- 更新备考清单：增加"二面专项"分区，7个针对性 checkbox

## [2026-05-16] Update | 主题-模拟面试.md — 重组结构（一基本信息/二项目介绍/三架构能力/四软技能）
- 重组前结构混乱（行为&管理放末尾、评分维度嵌套在项目追问下、架构实战与面试追问分散两处）
- 一、基本信息介绍：自我介绍90秒口径 + 核心技能表 + 面试官评分维度 + 薄弱预警&备考清单
- 二、项目介绍：6个项目各自完整（游戏/支付/IM/撮合/区块链OTC/CloudStack），每节含一句话+核心贡献+高频追问+详细设计链接
- 三、架构能力介绍：技术职责+架构师职责+微服务拆分+实战速查+难点清单+案例
- 四、软技能：技术决策+培养开发+线上事故五步法+反问
- 所有内容全量保留，无删减

## [2026-05-16] Update | 主题-模拟面试.md — 合并 up.md（简历）
- 6.4 区块链OTC 新增"技术亮点"表：低代码 Redisson @DistributedLock 注解方案 + Server层队列合并DB操作（100ms换30%吞吐）+ 多链集成；附 Redisson 注解方案追问口径
- 新增 6.5 CloudStack 云计算平台网络改造：多运营商网络适配（电信/移动/联通多线路）+ 深信服硬件插件替代虚拟路由 VR，含面试口径和高频追问
- 已有内容（游戏0FGC/支付链路/撮合引擎/区块链安全/MongoDB/架构速查表/自我介绍）不重复

## [2026-05-16] Update | 主题-三高架构.md — 合并 Java 高并发架构师 100 个必会知识点.md
- 二、高并发：新增"架构师高并发思维"5原则（CPU Cache命中率/顺序IO/无锁/少线程/事件驱动）
- 二、高并发：新增 CPU 硬件并发基础表（Cache层级/Cache Line/False Sharing/MESI/NUMA/内存屏障）
- 四、高性能：新增"高性能网络"节（Reactor三模型/IO多路复用select→epoll/零拷贝对比/Netty三件套/Kafka高性能原理）
- 五、联动场景：新增"高并发架构模式"表（Actor Model/Event-Driven/Backpressure/CQRS/Event Sourcing/Pipeline）
- Backpressure 核心约束说明（Kafka poll主动拉取=天然背压；Reactor Flowable/Flux=编程式背压）
- 已有内容（限流算法/熔断/分布式锁/线程池/分库分表/MQ可靠性/CAP/BASE）不重复

## [2026-05-16] Update | 概念-JVM.md — 三文件合并重写（机制-JVM内存模型 + 机制-垃圾收集器 + 机制-类加载机制）
- 大幅扩充：对象晋升老年代规则表、对象创建5步、TLAB机制（来自 机制-JVM内存模型）
- 新增三色标记法（白/灰/黑）+ CMS增量更新 vs G1 SATB（来自 机制-垃圾收集器）
- 新增调优方法论5步 + 正常基线经验值表、YoungGC/STW过长排查（来自 机制-垃圾收集器）
- 新增三种类加载器 ASCII图 + loadClass核心代码 + 线程上下文类加载器（来自 机制-类加载机制）
- 新增 JDK8 vs JDK9+ 类加载器架构对比 + 自定义ClassLoader使用场景表（来自 机制-类加载机制）
- 新增七、综合对比（堆vs栈/永久代vs元空间/对象内存分配方式三表）
- 生产风险新增类加载5种风险表；应用边界扩充何时需要自定义ClassLoader
- 删除已合并的三个源文件；index.meta.toml 关键词全部重定向至 概念-JVM.md

## [2026-05-16] Update | 概念-分布式理论.md — 合并 Hollis/架构设计 CAP/BASE/缓存方案
- CAP/BASE 理论已有深度覆盖，跳过重复内容
- 三、BASE 理论末尾新增"分布式缓存层次"子节：6层架构（客户端→CDN→Nginx→服务端→DB→OS）

## [2026-05-16] Update | 主题-三高架构.md — 合并 架构设计.md 六、多线程 + 并发三要素
- 二、高并发：新增并发编程三要素表（原子性/有序性/可见性，修正源文"可用性"笔误）
- 四、线程池调优：新增 5 步执行优先级状态机（core→queue→max→reject→keepAlive回收）

## [2026-05-16] Update | 主题-三高架构.md — 合并 架构设计.md 五、高并发设计
- 一：新增 CAP（CP/AP 对比表）+ BASE 理论
- 二：补充三板斧 + 垂直/水平扩展；锁策略扩展分布式锁三方案 + 锁续期/防误删
- 三：新增负载均衡 4 种策略
- 四：新增分库分表三类型、MQ 削峰核心纪律（四大可靠性问题）、性能瓶颈根因分析（6步顺序）
- 五：新增高并发典型架构图（CDN→Nginx→网关→Service→Cache/MQ/DB/Search）

## [2026-05-16] Update | 主题-设计模式.md — 合并 架构设计.md 一~四节
- 创建型补充：建造者（链式/Lombok @Builder）、原型（clone/Prototype Scope）
- 结构型补充：适配器（HandlerAdapter/AdvisorAdapter）、装饰器（HttpServletRequestWrapper）、桥接/组合/外观速查表
- 行为型补充：命令/迭代器/中介者/备忘录/状态/访问者/解释器速查表
- 新增 七、Spring 设计模式全景（10种 pattern → Spring class 完整映射，含中介者/DispatcherServlet）
- 原七→八（与其他概念的关系），导航表同步更新
- 已有深度内容（代理/工厂/单例/观察者/模板方法/策略/责任链）未重复

## [2026-05-16] Ingest | 主题-架构体系.md — 新建（raw/note/架构体系/ 7个文件）
- 新建 `summaries/主题-架构体系.md`，覆盖 7 大模块知识全景
- 聚焦 wiki 其他页未覆盖内容：DDD（战略/战术/分层/CQRS）、Spring Cloud Netflix vs Alibaba 对比、高可用 SLA 指标/异地多活策略、秒杀系统分层拦截、SOLID 原则、CI/CD + K8s 发布策略
- 已有详细 wiki 页的模块（JVM/并发/MySQL/Redis/分布式/Kafka/Dubbo/设计模式/算法）仅给出链接索引，不重复内容
- 更新 `index.meta.toml` summaries 列表，重建 index.md（92 pages）

## [2026-05-16] Update | 概念-Java集合.md — 合并  04-数据结构.md
- Queue体系 新增 SynchronousQueue（无容量直接移交，newCachedThreadPool 场景）和 DelayQueue（堆+Delayed接口）
- LinkedBlockingQueue 补充"两把锁，吞吐高于 ArrayBlockingQueue"细节
- 九、L4数据结构全景 结构图新增跳表（SkipList）节点；高频考点速查新增跳表一节（原理/O(log n)/ConcurrentSkipListMap/vs红黑树）
- 其余内容（List/Set/Map/树对比表）已有更完整版本，不重复合并

## [2026-05-16] Ingest | 机制-线程.md — 新建（ 03-线程.md）
- 新建 `concepts/04-JUC/机制-线程.md`（L3 层）
- 覆盖内容：线程生命周期6状态图、创建方式5种对比、关键方法辨析（start/run/sleep/join/yield/wait/notify）、线程间通信6种机制、中断机制三方法、死锁四条件与活锁/饥饿区分、jstack/Arthas 排查
- 不重复已有页面内容：synchronized/AQS/CAS/volatile/线程池/CompletableFuture 均已有独立页面
- 更新 `index.meta.toml`：L3 层新增5条关键词（线程生命周期/线程状态/BLOCKED·WAITING/死锁/线程中断）
- 重新生成 `index.md`（91 pages）

## [2026-05-15] Update | 支付中台内容从主题-模拟面试迁移到系统设计-支付系统
- 删除 主题-模拟面试.md 中"二、支付中台 & Seata TCC"章节（约 140 行）
- 将其中新增知识合并进 系统设计-支付系统.md：分户账/总账/在途户、分账解耦、清分/清算/结算/对账术语、对账差异闭环（长款/短款/重单/漏单）、支付超时中间态、信封加密、三条新面试追问
- 主题-模拟面试.md 添加 [[系统设计-支付系统]] 导航提示，章节重新编号 一→八

## [2026-05-15] Update | 重写 summaries/主题-模拟面试.md
- 去掉时间列和所有章节标题中的时间标注
- 去掉面试官/候选人对话格式（🧑‍💼/👨‍💻），改为直接"### 问题标题 + 答案"结构
- 去掉所有 📝考察点、💡加分技巧、⚠️陷阱预警 旁注，精炼内容
- 快速导航改为两列（标题索引 + 概述），去掉时间列

## [2026-05-15] Rename | synthesis/mock-interview-eson.md → summaries/主题-模拟面试.md
- 移动文件：`synthesis/mock-interview-eson.md` → `summaries/主题-模拟面试.md`
- 更新 frontmatter：`type: synthesis` → `type: summary`，`name` 改为 `模拟面试`
- 更新 `index.meta.toml`：从 synthesis 删除，加入 summaries 列表
- 移除断链 `[[概念-分布式事务]]`（已由 synthesis 页面覆盖）
- 重新生成 index.md（90 pages）

## [2026-05-15] Update | 重写 wiki/synthesis/ 全部 12 个页面
- 使用 rewrite-synthesis-prompt.md 按问题驱动格式重写所有 synthesis 页
- 统一结构：场景概述→方案对比矩阵（含结论列）→决策树→落地实践→面试追问（**问**/**答**格式）→概念关系
- 修复所有断链：`[[主题-三高架构]]`→`[[主题-三高架构]]`、`[[概念-Redis]]`→`[[概念-Redis]]`、`[[主题-消息队列]]`→`[[机制-RabbitMQ]]`、`[[概念-MySQL]]`→`[[概念-MySQL]]`、`[[概念-分布式理论]]`→`[[概念-分布式理论]]`、`[[机制-Spring]]`→`[[机制-Spring]]`、`[[概念-分布式事务]]`→`[[机制-Seata]]`、`[[概念-分布式任务]]`→`[[概念-分布式任务]]`、`[[机制-Zookeeper]]`→`[[机制-Zookeeper]]`、`[[机制-SpringCloud]]`→`[[机制-SpringCloud]]`
- 删除所有 `[[主题-模拟面试]]` 引用（文件不存在）
- 为缺少 frontmatter 的文件补全：IM系统/电商系统/区块链OTC 新增完整 YAML frontmatter
- 为 frontmatter 不完整的文件补全字段：P8架构思维/项目难点表达 补充 type/status/name/related/sources
- 从 `index.meta.toml` 移除已不存在的 `synthesis/mock-interview-eson.md` 引用
- 重新生成 index.md（89 pages）

## [2026-05-15] Rename | 主题-Java-Python-Go对比 → 主题-编程语言
- 重命名 `summaries/主题-Java-Python-Go对比.md` → `summaries/主题-编程语言.md`
- 同步更新 `index.meta.toml` 和 `index.md` 的 summaries 引用
- 更新页面元信息：`name` 改为 `编程语言`，标题改为 `编程语言（Java / Python / Go 对比）`

## [2026-05-15] Update | 新增主题：Java / Python / Go 对比
- 新建 `summaries/主题-Java-Python-Go对比.md`，覆盖语言设计哲学、类型系统、运行时/内存、并发、生态、部署成本与选型框架
- 更新 `index.meta.toml` 的 `summaries` 列表，新增 `summaries/主题-Java-Python-Go对比.md`
- 重建 `index.md`（`/opt/miniconda3/bin/python3 scripts/build_index.py`）

## [2026-05-15] Update | 文件改名
- `概念-OAuth2授权协议.md` → `概念-OAuth2.md`，更新 概念-网络安全.md、index.meta.toml 引用
- `机制-配置中心与Nacos.md` → `机制-Nacos.md`
- `机制-容器化与Docker.md` → `机制-Docker.md`
- 更新 index.meta.toml、synthesis/系统设计-游戏架构.md 中的引用，重建 index.md

## [2026-05-15] Update | 合并 概念-读写分离 到 概念-MySQL
- `概念-读写分离.md` 内容合并为 `概念-MySQL.md` 第十三章（读写分离）
- 合并 aliases（读写分离/主从分离/主从延迟/强制读主库）、sources（2篇Hollis笔记）、related 到 MySQL frontmatter
- 更新内部引用 `[[概念-MySQL]]` → 内部锚点链接
- 删除 `概念-读写分离.md`，重建 index.md

## [2026-05-15] Update | 重写 concepts/08-distributed 全部 19 个概念页

- **链接修复**：全部文件中 `[[机制-Zookeeper]]` → `[[机制-Zookeeper]]`、`[[机制-SpringCloud]]` → `[[机制-SpringCloud]]`、`[[主题-三高架构]]` → `[[主题-三高架构]]`、`[[概念-分布式理论]]` → `[[概念-分布式理论]]`、`[[概念-MySQL]]`/`[[概念-MySQL]]` → `[[概念-MySQL]]`、`[[主题-消息队列]]` → `[[机制-RabbitMQ]]`、`[[概念-Redis]]` → `[[概念-Redis]]`；移除 `[[主题-模拟面试]]` 孤立链接
- **补全残缺 frontmatter**：`机制-Protobuf.md`（补 type/status/name/layer）、`机制-数据加密与脱敏.md`（同）
- **概念-分布式理论.md**：已在上轮完成，快速导航 + 10个编号章节，ZAB vs Raft 对比
- **概念-分布式事务.md**：type改为concept，layer=L8，空回滚/悬挂统一解法（分布式事务记录表），本地消息表 vs 事务消息对比
- **概念-幂等设计.md**：8个编号章节；幂等号设计原则；MQ消费幂等 @Transactional 代码；唯一性约束三大缺陷
- **机制-Zookeeper.md**：ZAB vs Raft 差异；集群容错表（3/5/7节点）；惊群效应 → 监听前驱节点解法
- **机制-Kafka.md**：Producer TPS 参数表；Consumer 并行消费 bucketing 代码；ISR/HW/LEO 详解
- **机制-RabbitMQ.md**：本地消息表完整 SQL Schema；Spring Event vs MQ 对比；消息乱序4种解法
- **机制-RocketMQ.md**：CommitLog vs Kafka Segment 差异表；4.x vs 5.x 延迟消息对比；三路 MQ 对比表
- **机制-Netty.md**：主从 Reactor 完整链路；ByteBuf 四种实现（Pooled/UnPooled × Heap/Direct）；WriteBufferWaterMark 背压实践
- **机制-Dubbo.md**：9层分层架构；5种负载均衡；Dubbo SPI vs JDK SPI 对比；gRPC 四种通信模式
- **机制-Protobuf.md**：完整 frontmatter；Varint/Field Tag 编码；Deadline 传播；17种 Status Code；gRPC vs REST 对比
- **概念-OAuth2授权协议.md**：PKCE 扩展详解；Token 验证方式（JWT vs Introspection）；微服务 Client Credentials 模式
- **概念-分布式场景.md**：type=synthesis，layer=L8；修复所有链接；库存三方案/锁选型/SSO/跨库JOIN/平滑迁移/Excel导入
- **概念-分布式任务.md**：Timer缺陷；分层时间轮；XXL-Job DB悲观锁防重复；退避策略三类型；PowerJob MapReduce动态分片
- **概念-可观测性.md**：修复链接；Prometheus 4种 Metric 类型；SkyWalking 无侵入原理；三支柱协作流程；12类故障案例
- **机制-ElasticSearch.md**：修复链接；Hot-Warm-Cold + ILM；乐观锁 if_seq_no；search_after vs scroll；DB一致性4方案
- **机制-Seata.md**：name改为"Seata"；修复链接；AT vs XA 对比表；空回滚/悬挂统一解法；四模式对比
- **机制-配置中心与Nacos.md**：修复链接；Nacos 1.x 长轮询 vs 2.x gRPC；Distro vs JRaft；四注册中心选型对比
- **机制-容器化与Docker.md**：修复链接；namespace+cgroup原理；K8s核心对象；冷启动4方案对比；云原生改造要点表
- **机制-数据加密与脱敏.md**：完整 frontmatter；修复链接；AES-GCM IV铁律；RSA-OAEP混合加密；防重放三层防御；Arrays.fill清零

## [2026-05-14] Update | 重写 concepts/04-JUC 全部 9 个概念页

- **概念-Java并发.md**：从 ASCII 知识地图页重写为标准结构；增加快速导航、并发三大问题表、L3知识地图（带模块定位）、高频考点速查（7个考点详解）、工具选型决策树、常见误区表
- **概念-JMM.md**：增加快速导航、中文编号标题；新增"硬件层真相"章节（Store Buffer/Invalidation Queue/MESI不够用的原因）；完整 8 条 happens-before 规则表；as-if-serial vs happens-before 对比；StoreLoad 屏障开销说明
- **机制-Volatile.md**：增加快速导航、中文编号标题；展开 LOCK 前缀指令 + MESI 缓存失效流程；DCL 详解（new 三步字节码+重排危害+屏障位置）；volatile vs synchronized vs AtomicXxx 三维对比；生产风险表（long/double 字撕裂等）
- **机制-Synchronized.md**：增加快速导航、中文编号标题；新增 Mark Word 结构（64-bit JVM 5种状态）；ObjectMonitor 详细字段；锁升级各阶段性能表；JIT 锁消除（逃逸分析示例）/锁粗化；生产风险表（死锁/String锁/wait虚假唤醒）
- **机制-CAS.md**：增加快速导航、中文编号标题；展开 cmpxchg + 缓存行锁 vs 总线锁；新增"LongAdder 高竞争优化"章节（base + Cell 数组分段机制）；AtomicLong vs LongAdder 对比表；生产风险表
- **机制-AQS.md**：增加快速导航、中文编号标题；Node.waitStatus 5种状态表；LockSupport vs Object.wait 对比；新增"AQS 实现的工具类"章节（ReentrantLock/CountDownLatch/Semaphore/CyclicBarrier 代码级原理）；AQS 工具类横向对比表
- **机制-线程池.md**：增加快速导航、中文编号标题；ctl 变量设计说明；新增"生产监控与动态线程池"章节（关键指标/动态调整/Hippo4j）；ThreadPoolExecutor vs ForkJoinPool 对比；容量规划公式
- **概念-ThreadLocal.md**：增加快速导航、中文编号标题；移除不存在的 [[机制-垃圾收集器]] 链接；新增"父子线程传递"章节（InheritableThreadLocal/TTL capture-replay-restore 机制）；ThreadLocal vs synchronized vs ScopedValue 三维对比；弱引用误解澄清
- **机制-CompletableFuture.md**：增加快速导航、中文编号标题；result volatile 字段 happens-before 说明；新增链式串行+异常回退场景示例；thenApply vs thenApplyAsync 详细对比；CompletableFuture vs CountDownLatch 对比；join() vs get() 对比；allOf 结果收集注意事项
- 重建索引：91 页

## [2026-05-14] Update | 5次合并：数据结构体系/缓存三大问题/Canal/分库分表→目标页

- 删除 `concepts/05-structure/概念-数据结构体系.md`，内容已在上一轮合并入 `concepts/02-java/概念-数据结构.md`（六、L4知识地图）
- 删除 `concepts/06-storage/概念-缓存三大问题.md`，内容合并入 `concepts/06-storage/概念-Redis.md`（新增十一章：缓存穿透/击穿/雪崩解法、缓存一致性4种策略、8种内存淘汰策略）
- 删除 `concepts/08-distributed/机制-Canal数据同步.md`，内容合并入 `concepts/06-storage/概念-MySQL.md`（新增十一章：Canal工作原理、binlog订阅、CDC、幂等消费）
- 删除 `concepts/06-storage/概念-分库分表.md`，内容合并入 `concepts/06-storage/概念-MySQL.md`（新增十二章：分片键选择、Hash取模、基因法、雪花算法、时钟回拨、扩容双写迁移）
- 更新 `index.meta.toml`：L4 summaries 移除 概念-数据结构体系.md；L5 缓存问题关键词路径改为 Redis.md；L7 synthesis 移除 概念-分库分表.md
- 重建索引：91 页

## [2026-05-14] Update | 重写 concepts/03-jvm 下 2 个概念页（JIT编译、JVM内存模型）

- 重写 `机制-JIT编译.md`：增加快速导航、C1/C2 分层编译 5 层级表、JVM 参数表（CompileThreshold/CodeCacheSize/MaxInlineSize 等）、C1 vs C2 对比、生产风险表格（预热超时/CodeCache 满/反优化）
- 重写 `机制-JVM内存模型.md`：增加快速导航、对象创建流程（指针碰撞 vs 空闲列表）、TLAB 机制、堆 vs 栈/永久代 vs 元空间/分配方式三组对比表、生产风险表格（7 类 OOM/内存抖动场景）、NMT 排查说明
- 两个页面均按九段式重写模板重构，二级标题使用中文编号

## [2026-05-14] Update | 重写 concepts/03-jvm/机制-垃圾收集器.md

- 按九段式重写模板重构全文（280行 → ~310行），增加快速导航表、中文编号二级标题
- 去重：GC 收集器选型表合并为一张（含 Shenandoah），Java 8→11 差异合并为一处
- JVM 调优实战从核心机制区域拆出，分散到四/五/七章（调优原则、排查案例、生产风险）
- 关键权衡改为表格化综合对比（第六章）
- 去掉所有 `---` 水平分隔线

## [2026-05-14] Update | 重写 3 个 concepts/02-java 概念页（动态代理、泛型、JDK新特性）

- 重写 `机制-动态代理.md`：增加快速导航、Spring AOP 代理选择策略、JDK 代理 vs CGLIB 全维度对比表、自调用失效风险、JDK 17+ 强封装影响
- 重写 `机制-泛型.md`：增加快速导航、桥方法机制、PECS 原则详解、Java vs C++ vs C# 泛型三方对比、堆污染/重载冲突/序列化丢失泛型等生产风险
- 重写 `概念-JDK新特性.md`：增加快速导航、LTS 版本选型表、虚拟线程 vs 响应式对比、pin 问题与 ThreadLocal 膨胀风险、各特性应用边界表
- 三个页面均按重写模板调整为中文编号结构

## [2026-05-14] Update | 重写 3 个 L1 Java 基础概念页

- 重写 `概念-Java类型.md`：增加快速导航、中文编号标题、生产风险表格、应用边界表格
- 重写 `概念-Java异常.md`：增加 try-with-resources、自定义异常示例、综合对比表格、生产风险表格
- 重写 `概念-OOP特征.md`：增加重载vs重写对比、继承vs组合对比、JDK 8 default 方法说明
- 三个文件均按重写模板统一为九段式结构

## [2026-05-14] Update | 重写 3 个 concepts/02-java 概念页

- 重写 `机制-Java序列化.md`：增加快速导航、Externalizable、readResolve、Kryo 对比、fastjson 漏洞追问、生产风险表格
- 重写 `机制-Lambda.md`：增加快速导航、invokedynamic 执行流程、变量捕获、方法引用分类、并行流 commonPool 风险、序列化 Lambda 陷阱
- 重写 `机制-SPI.md`：增加快速导航、类加载器选择、JDBC/SLF4J/Boot 3.x 案例、Java SPI vs Dubbo SPI vs Spring SPI 三方对比
- 三个页面均按重写模板调整为中文编号九章结构

## [2026-05-14] Rename | concepts/02-java-lang → 02-java

- 目录 `concepts/02-java-lang/` → `concepts/02-java/`
- 更新 `index.meta.toml` 全部引用，重新生成 `index.md`

## [2026-05-13] Rename | 批量重命名 synthesis 页

- `系统设计-在线游戏平台架构` → `系统设计-游戏架构`
- `系统设计-IM即时通信系统` → `系统设计-IM系统`
- `系统设计-彩票系统核心设计` → `系统设计-彩票系统`
- 一批 `设计-*` 系统类页面统一改为 `系统设计-*`（秒杀/短链/区块链OTC/游戏/彩票/IM）
- 同步更新全部交叉引用及 `index.md`

## [2026-05-13] Merge | 6 个碎片页收口到已有页面

- `设计-线上问题排查` → 并入 `概念-可观测性`（删除）
- `设计-大厂秒杀实践` → 并入 `系统设计-秒杀系统`（删除）
- `设计-消息队列场景综合` → 并入 `机制-消息队列可靠性`（删除）
- `概念-JVM` → 并入 `机制-GC算法与垃圾收集器`（删除）
- `设计-云原生选型` → 并入 `机制-容器化与Docker`（删除）
- `设计-MySQL大表与查询优化` → 并入 `概念-SQL查询优化`（删除）

## [2026-05-13] Merge | DDD/支付专题收口

- `设计-DDD落地实战` → 并入 `概念-DDD`（删除）
- `设计-账户钱包系统` / `设计-订单超时关闭` → 并入 `系统设计-支付系统`

## [2026-05-12] Update | Wiki 合并整理 — 删除 8 个碎片页，新建 3 个合并页

- 反射 + 动态代理 → `机制-反射与动态代理`
- String不可变性 + 包装类 → `概念-Java基础值类型`
- BitMap + 布隆过滤器 → `概念-BitMap`
- 引用类型 → 并入 `机制-GC算法与垃圾收集器`
- Java集合框架 → `概念-Java集合`

## [2026-05-12] Ingest | 源码体系 + 锁体系主题页

- 新建 `summaries/主题-源码体系.md`：Java核心类库/并发包/Spring/MyBatis/Tomcat 源码地图
- 更新 `主题-源码体系.md`：补充 Redis/Redisson 分布式锁源码章节
- 新建 `summaries/主题-锁体系.md`：Java锁/MySQL锁/分布式锁三大体系

## [2026-05-12] Query | 高并发

- 输出高并发方法论，无新增知识点，不回写

## [2026-05-12] Update | 模拟面试加强版

- 更新 `synthesis/mock-interview-eson.md`：60min → 90min，7 → 9 阶段，新增算法/行为/追问/备考清单

## [2026-05-11] Ingest | 简历深度分析 — 补全 7 个缺失知识点

- 新建 concept × 4：`机制-gRPC与Protobuf` / `机制-WebSocket协议` / `机制-数据加密与脱敏` / `机制-RocketMQ`
- 新建 synthesis × 3：`系统设计-区块链OTC` / `系统设计-IM系统` / `系统设计-电商系统`
- 更新 `机制-Netty`：新增 TCP 参数调优章节

## [2026-05-11] Ingest | 模拟面试 — Eson 简历

- 新建 `synthesis/mock-interview-eson.md`：7 阶段 60 分钟面试脚本

## [2026-05-09] Ingest | 彩票系统岗位专项

- 新建 `synthesis/设计-彩票系统核心设计`
- 更新 `机制-Kafka`：新增大流量实战补充

## [2026-05-09] Ingest | P2+P3 批次：DDD/P8 边界/云原生

- 新建 `synthesis/设计-DDD落地实战` / `设计-P8架构思维` / `设计-云原生选型`

## [2026-05-08] Ingest | P0+P1 批次：项目难点/大厂实践/三高体系/分布式深水区

- 新建 `synthesis/设计-项目难点表达` / `设计-大厂秒杀实践`
- 新建 `summaries/主题-三高体系`
- 更新 `概念-分布式系统理论`：3PC/拜占庭/NewSQL/Leaf

## [2026-05-08] Ingest | 德州扑克核心算法 + 在线游戏平台架构

- 新建 `synthesis/设计-德州扑克核心算法`：牌型评估/胜率/保险/结算四层
- 新建 `synthesis/设计-在线游戏平台架构`：5 微服务架构 7 大亮点

## [2026-05-08] Ingest | 架构设计 + 场景题

- 新建 `synthesis/设计-P7架构思维`
- 新建 synthesis × 6：MySQL大表/海量数据/接口防护/消息队列场景/容量规划/分布式场景综合

## [2026-05-08] Ingest | P7 第二轮：Seata/SpringMVC/OAuth2

- 新建 `机制-Seata框架机制` / `机制-SpringMVC` / `概念-OAuth2授权协议`
- 更新 `机制-SpringBoot`：完整启动流程 + 优雅停机

## [2026-05-08] Ingest | P7 第一轮：消息队列/配置中心/任务调度/异步编程/JVM 调优

- 新建 `机制-RabbitMQ` / `机制-配置中心与Nacos` / `机制-分布式任务调度` / `机制-CompletableFuture` / `概念-JVM`
- 新建 `summaries/主题-消息队列体系`
- 更新 `机制-Spring`：IoC 循环依赖补充

## [2026-05-07] Ingest | Interview/Eson → 4 缺失知识点

- 新建 `机制-对象池技术` / `机制-RocksDB` / `系统设计-支付系统`

## [2026-05-07] Lint | 全库健康检查

- 修复 5 处断链、4 个 sources 路径
- 知识库：concepts×69 / summaries×8 / synthesis×8

## [2026-05-07] Ingest |  → Kafka + Netty + DDD + Nacos

- 新建 `机制-Kafka` / `机制-Netty` / `概念-DDD`
- 更新 `机制-微服务与SpringCloud`：Nacos 注册中心章节

## [2026-05-07] Ingest | 架构体系/ → JDK 新特性 + 可观测性 + 算法高频

- 新建 `概念-JDK新特性` / `概念-可观测性` / `设计-算法高频题型`
- 更新 `概念-分布式系统理论`：Paxos/Raft
- 更新 `机制-Dubbo`：gRPC + 序列化对比
- 更新 `概念-DDD`：CQRS + Event Sourcing

## [2026-05-07] Ingest | 多模块批量摄入

- 项目经验/ → `synthesis/设计-线上问题排查`（25 实战案例）
- 网络安全/ → `concepts/概念-网络安全`（13 文件）
- 微服务/ → 更新 `机制-微服务与SpringCloud`（5 新章节）
- 分布式/ → `概念-分布式系统理论` / `概念-幂等设计` / `机制-Canal数据同步`
- 设计模式/ → `主题-设计模式`（21 文件）
- 操作系统/ → `概念-操作系统`（30 文件）
- 计算机网络/ → `概念-计算机网络`（30 文件）
- 容器/ + 高并发/ + 高可用/ → `机制-容器化与Docker` / `主题-三高体系` / `主题-三高体系`
- 高性能/ + 架构设计/ → `概念-布隆过滤器` / `概念-读写分离` + 微服务拆分原则
- 场景题/ + 面经实战/ → `概念-Redis实战` / `设计-短链服务`

## [2026-05-06] Ingest | 初始批量摄入

- 面经实战/ → synthesis × 3：秒杀系统/分布式事务/分库分表
- SpringCloud/ → `机制-微服务与SpringCloud`
- Dubbo/ → `机制-Dubbo`
- ElasticSearch/ → `机制-倒排索引与ElasticSearch`
- RabbitMQ/ → `机制-消息队列可靠性`
- Zookeeper/ → `机制-ZAB协议与Zookeeper`
- Spring/ → Spring 机制体系 + SpringBoot/SpringMVC/MyBatis
- MyBatis/ → `机制-MyBatis`
- 集合类/ → `机制-HashMap` / `机制-ConcurrentHashMap` + `概念-Java集合`

## [2026-05-06] Lint | 健康检查

- 删除 4 重复页：机制-HashMap / 机制-ConcurrentHashMap / 机制-CopyOnWrite / 概念-集合迭代一致性
- 修复 `概念-Redis体系` 畸形 wikilink

## [2026-05-02] Ingest | 基础模块批量摄入

- Redis/ → concept × 5 + `概念-Redis体系`
- MySQL/ → concept × 5 + `概念-MySQL体系`
- Java并发/ → concept × 7 + `概念-Java并发`
- JVM/ → concept × 5 + `概念-JVM`
- 数据结构/ → concept × 7 + `概念-数据结构体系`
- Java基础/ → concept × 11 + `主题-Java语言基础`

## [2026-05-02] Init | Java 知识库初始化

- 重命名 `llm-wiki-template/` → `llm-wiki-java/`
- 重写根目录/实例 `CLAUDE.md`，建立 L1-L8 知识结构
- Hollis 来源目录更名，清理 Notion 导出文件名 ID

## [2026-05-14] Update | 重命名并移动分布式事务页面
- synthesis/设计-分布式事务.md → concepts/08-distributed/概念-分布式事务.md
- 更新 index.meta.toml 中的路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名并移动分布式场景综合页面
- synthesis/设计-分布式场景综合.md → concepts/08-distributed/设计-分布式场景.md
- 更新 index.meta.toml 中的路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名分布式任务调度页面
- concepts/08-distributed/机制-分布式任务调度.md → 概念-分布式任务.md
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名分布式系统理论页面
- concepts/08-distributed/概念-分布式系统理论.md → 概念-分布式理论.md
- 更新 index.meta.toml 路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名分布式场景页面
- concepts/08-distributed/设计-分布式场景.md → 概念-分布式场景.md
- 更新 index.meta.toml 路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名并移动容量规划页面
- synthesis/设计-容量规划与性能基准.md → summaries/主题-机器容量.md
- 更新 index.meta.toml 两处路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名三高体系页面
- summaries/主题-三高体系.md → 主题-三高架构.md
- 更新 index.meta.toml 两处路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名消息队列体系页面
- summaries/主题-消息队列体系.md → 主题-消息队列.md
- 更新 index.meta.toml 路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名并移动海量数据处理页面
- synthesis/设计-海量数据处理.md → summaries/主题-海量数据.md
- 更新 index.meta.toml 路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名并移动算法高频题型页面
- synthesis/设计-算法高频题型.md → summaries/主题-高频算法.md
- 更新 index.meta.toml 路径引用
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名 Seata 页面
- concepts/08-distributed/机制-Seata框架机制.md → 机制-Seata.md
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名 Zookeeper 页面
- concepts/08-distributed/机制-ZAB协议与Zookeeper.md → 机制-Zookeeper.md
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名 ElasticSearch 页面
- concepts/08-distributed/机制-倒排索引与ElasticSearch.md → 机制-ElasticSearch.md
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 重命名并移动 WebSocket 页面
- concepts/08-distributed/机制-WebSocket协议.md → concepts/01-CS/机制-WebSocket.md
- 重新生成 wiki/index.md（107 pages）

## [2026-05-14] Update | 合并消息队列可靠性页面至 RabbitMQ
- 将 concepts/08-distributed/机制-消息队列可靠性.md 内容合并入 机制-RabbitMQ.md
- 新增：场景综合（Spring Event/BlockingQueue/拉推模式/乱序/Kafka吞吐/订单超时）、本地消息表两章节
- 补充 8 条场景题 sources，更新 aliases 和 related
- 删除 机制-消息队列可靠性.md
- 重新生成 wiki/index.md

## [2026-05-14] Update | 重命名 gRPC 页面
- concepts/08-distributed/机制-gRPC与Protobuf.md → 机制-Protobuf.md
- 重新生成 wiki/index.md（106 pages）

## [2026-05-14] Update | 重命名 SpringCloud 页面
- concepts/08-distributed/机制-微服务与SpringCloud.md → 机制-SpringCloud.md
- 重新生成 wiki/index.md（106 pages）

## [2026-05-14] Update | 重写 机制-WebSocket.md
- 按 rewrite-concept-prompt.md 模板重写
- 新增快速导航、中文编号标题（九章）
- 补充 Java核心使用（Netty Pipeline / Spring WebSocket）、生产风险、综合对比章节
- 修正内部链接：机制-gRPC与Protobuf → 机制-Protobuf
- 重新生成 wiki/index.md（106 pages）

## [2026-05-14] Update | 合并四个 MySQL 机制页至 概念-MySQL体系.md
- 合并来源：机制-InnoDB索引.md、机制-InnoDB锁.md、机制-MVCC.md、机制-MySQL三种日志.md
- 按 rewrite-concept-prompt.md 重写，九章结构：第一性原理、索引、锁、MVCC、三种日志、综合对比、生产风险、关系、边界
- 新增快速导航、知识依赖图、综合对比章节、生产风险表格
- 合并所有 sources（15 条）和 aliases
- 删除四个源文件，更新 index.meta.toml keywords
- 重新生成 wiki/index.md

## [2026-05-14] Update | 重命名 MySQL 体系页面
- concepts/06-storage/概念-MySQL体系.md → 概念-MySQL.md
- 更新 index.meta.toml 两处路径引用
- 重新生成 wiki/index.md（102 pages）

## [2026-05-14] Update | 合并三个 Redis 机制页至 概念-Redis.md
- 合并来源：机制-Redis分布式锁.md、机制-Redis持久化.md、机制-Redis集群与高可用.md
- 按 rewrite-concept-prompt.md 重写，九章结构：第一性原理、数据类型与编码、持久化、分布式锁、集群高可用、综合对比、生产风险、关系、边界
- 新增快速导航、综合对比（RDB/AOF/混合、哨兵/Cluster、Redis锁/ZK锁）、生产风险表格（9 条）
- 合并所有 sources（8 条）和 aliases
- 删除三个源文件，更新 index.meta.toml keywords
- 重新生成 wiki/index.md

## [2026-05-14] Update | 移动 SpringCloud 页面至 07-framework
- concepts/08-distributed/机制-SpringCloud.md → concepts/07-framework/机制-SpringCloud.md
- 重新生成 wiki/index.md（99 pages）

## [2026-05-14] Update | 合并 Redis体系 和 Redis实战 至 概念-Redis.md
- 合并来源：概念-Redis体系.md、概念-Redis实战.md
- 新增内容：Redis vs MySQL 对比表（六、综合对比）、层间依赖（八、关系）
- 新增十、实战场景设计：排行榜（ZSet+分片+编码合并）、附近的人（GEO）、点赞（ZSet/Set）、抢红包（List+二倍均值法）、UV统计（HyperLogLog）、购物车（Hash）
- 补充 6 条场景题 sources，补充 aliases（HyperLogLog、GEO等）
- 删除 概念-Redis体系.md、概念-Redis实战.md
- 更新 index.meta.toml，重新生成 wiki/index.md

## [2026-05-14] Update | 合并 概念-SQL查询优化.md 至 概念-MySQL.md
- 新增十、SQL 查询优化章节：慢SQL发现路径、EXPLAIN关键字段、索引失效场景表、深分页优化、JOIN优化、大表查询/DDL/数据清理、热点行解法、长事务危害与写法、分布式锁与事务位置、逻辑删除唯一约束、慢SQL原因清单
- 补充 17 条 sources，补充 aliases（慢SQL/EXPLAIN/大表优化等）
- 删除 概念-SQL查询优化.md
- 更新 index.meta.toml keywords，重新生成 wiki/index.md

## [2026-05-14] Update | 合并 机制-ConcurrentHashMap.md 至 机制-HashMap.md
- 新增三、ConcurrentHashMap 章节：分段锁 vs 节点锁、CAS+synchronized、fail-fast/fail-safe、COW
- 新增五、综合对比：HashMap vs CHM vs Hashtable、CHM vs COW 横向表格
- 原四-五改为四-七，补充 CHM 使用原则（null 禁止、弱一致性、size 近似）
- 合并 16 条 sources、aliases、related
- 删除 机制-ConcurrentHashMap.md，更新 index.meta.toml
- 重新生成 wiki/index.md

## [2026-05-15] Update | 主题-模拟面试 快速导航修正
- 更新 快速导航 表：移除已迁移的三节（JVM/CoinsOTC/Netty），修正章节编号（五→二、六→三、七→四、八→五）
- 添加迁移去向注释（指向 概念-JVM、系统设计-区块链OTC、系统设计-游戏架构）
- 重建 wiki/index.md（90 pages）

## [2026-05-15] Update | 技术内容知识聚合（模拟面试拆解）
- 主题-模拟面试.md：删除二（基础技术问答）、三（Gacha系统设计）、四（算法LRU）、附C（陷阱题集），仅保留一（暖场）、二（行为，原五）、附A、附B
- 迁移目标文件（新增面试追问/实战内容）：
  - 主题-锁体系.md → AQS、volatile vs synchronized、线程池CPU/IO参数
  - 概念-Redis.md → Redis为什么快、缓存一致性Cache Aside、bigkey、Cluster MOVED/ASK
  - 机制-Spring.md → @Transactional失效5场景+踩坑、Spring三级缓存为什么要三级
  - 概念-分布式事务.md → TCC vs本地消息表选型、Seata TC挂了、为什么不替换TCC
  - 主题-消息队列.md → RocketMQ vs Kafka场景对比、消息堆积止血+根因
  - 概念-MySQL.md → 跨分片聚合（汇总表+ClickHouse OLAP）
  - 系统设计-彩票系统.md → Gacha高并发设计、Redis挂了、热点分桶、概率公平、全球部署
  - 主题-高频算法.md → LRU完整Java实现+线程安全方案
  - 概念-JVM.md → G1 vs ZGC时机选择、对象池reset测试方法
  - 设计-P7架构思维.md → 性能瓶颈分层评估、从0到1搭系统五步法
- 重建 wiki/index.md（90 pages）

## [2026-05-15] Update | 系统设计-支付系统 一、支付架构 重写
- 用 6 张 ASCII 架构图完整覆盖 5 大核心工程问题
  - 1.1 聚合支付全链路（API网关→支付中台→渠道适配→MQ→账务→对账）
  - 1.2 多渠道路由决策（商户白名单/金额/成功率/熔断恢复，Nacos热更新）
  - 1.3 不丢单+不重单幂等防护链（Redis SETNX+DB唯一索引+双重防丢单机制）
  - 1.4 TCC空回滚+悬挂（dist_tx表作为锚点，完整正常/异常流程）
  - 1.5 双记账vs单记账（IN/OUT流水+在途户+余额三态+CHECK约束）
  - 1.6 清结算对账全流程（清分→清算→结算→对账差异闭环四级）
- 更新快速导航锚链接

## [2026-05-16] Update | 概念-JVM.md — 合并 06-JVM
- 新增「生命周期与类加载」：JVM 实例生命周期、对象生命周期、类生命周期、类初始化触发/被动引用、类加载器与双亲委派
- 扩展 JVM 内存区域：PC、虚拟机栈/栈帧、本地方法栈、堆分代、方法区/元空间
- 扩展 GC Roots、Minor GC 对象移动流程
- 新增「JVM 调优与排查」：jstat/jmap/jstack/jinfo、OOM dump、jstat 估算对象增长/GC 频率/GC 耗时、常见参数和调优结论
- 新增面试速答；L2 索引新增 JVM生命周期、类生命周期、jstat/jmap/jstack

## [2026-05-16] Update | 概念-DDD.md — 合并 DDD架构
- 补充 DDD vs 表驱动开发、DDD 收益与成本
- 战术建模新增实体 vs 值对象细化对比与订单/订单项示例
- 新增「DDD 落地流程」章节：确定业务领域、设计领域模型、建立统一语言、实现领域模型、应用架构设计、持续演进
- Java 落地补充洋葱架构、六边形架构与依赖方向
- 领域事件扩展：订单创建/支付/取消事件示例，明确领域事件与 MQ 消息的区别
- 新增面试速答；L8 索引新增实体/值对象、领域事件关键词

## [2026-05-16] Update | 机制-RocketMQ.md — 合并 08-mq
- 新增「可靠性与高性能设计」：生产/存储/消费三阶段可靠性、CommitLog + ConsumeQueue + PageCache + 零拷贝、Tag/SQL92 过滤、重复消费幂等
- 新增「顺序消息机制」：全局顺序 vs 局部顺序、MessageQueueSelector、MessageListenerOrderly、Broker MessageQueue 锁/本地 MessageQueue 锁/ProcessQueue 锁
- 扩展消息积压治理：mqadmin 定位、扩容/批量消费/临时 Topic 分流/位点跳过、监控预防
- 新增集群部署与调优参数：单 Master、多 Master、多 Master 多 Slave、Dledger、Broker/Consumer 参数
- 扩展 RocketMQ vs Kafka、Pull vs Push、面试速答与记忆口诀
- L7 索引新增顺序消息、消息堆积关键词

## [2026-05-16] Update | 机制-Spring.md — 合并 06-spring
- 二、IoC 容器新增「容器启动整体流程」：扫描组件、生成 BeanDefinition、BeanFactoryPostProcessor、创建非懒加载 singleton、发布启动事件
- 新增 BeanFactory vs ApplicationContext 对比，补充 Environment/MessageSource/ApplicationEventPublisher 等能力差异
- Bean 生命周期补充 7 步面试速记；新增 BeanFactoryPostProcessor vs BeanPostProcessor 扩展点对比
- 新增「SpringMVC 调用链」章节，保留 DispatcherServlet → HandlerMapping → HandlerAdapter → Controller → ViewResolver 流程
- L6 索引新增 Bean生命周期、BeanFactory/ApplicationContext 关键词

## [2026-05-16] Merge | SpringCloudGateway/OpenFeign → 机制-SpringCloud.md
- 合并 `机制-SpringCloudGateway.md` 与 `机制-SpringCloudOpenfeign.md` 到 `机制-SpringCloud.md`
- `机制-SpringCloud.md` 新增「OpenFeign 机制详解」与「Spring Cloud Gateway 机制详解」两章，保留动态代理执行流程、超时配置、重试/容错、Route/Predicate/Filter、Gateway 执行链、动态路由、过滤器机制和应用边界
- 综合对比扩展 OpenFeign vs RestTemplate vs Dubbo、Gateway vs Zuul vs Nginx
- L6 索引中的 OpenFeign/Gateway 关键词回指 `机制-SpringCloud.md`
- 删除合并后的独立文件 `机制-SpringCloudGateway.md`、`机制-SpringCloudOpenfeign.md`

## [2026-05-16] Add | 机制-SpringCloudOpenfeign.md — 摄入 09-微服务/13-openFeign
- 整理 OpenFeign 第一性原理、`@FeignClient` 用法、动态代理执行流程、服务发现与负载均衡、超时/重试/容错、与 RestTemplate/Dubbo 对比和应用边界
- L6 索引新增 `OpenFeign/Feign`、`Feign超时` 关键词
- `机制-SpringCloud.md` related 增加 `机制-SpringCloudOpenfeign`（后续已合并回 SpringCloud 总页）

## [2026-05-16] Add | 机制-SpringCloudGateway.md — 摄入 09-微服务/16-gateway
- 整理 Gateway 第一性原理、Route/Predicate/Filter、执行流程、动态路由、鉴权限流、过滤器链、Zuul/Nginx 对比和应用边界
- L6 索引将 `网关/Gateway` 指向独立 Gateway 页，并新增 `Route/Predicate/Filter` 关键词
- `机制-SpringCloud.md` related 增加 `机制-SpringCloudGateway`（后续已合并回 SpringCloud 总页）

## [2026-05-16] Update | 机制-Zookeeper.md — 补充 ZAB 三阶段与数据同步细节
- 四、ZAB 协议：补充三阶段划分（领导者选举→数据同步→请求广播）
- 四、ZAB 协议：补充两阶段提交完整流程（Follower持久化→ACK→Leader commit→Follower更新内存→Observer直接更新）
- 四、ZAB 协议：补充数据同步阶段（快照 vs Diff日志）；明确 ZK 是最终一致性而非强一致
- 五、Leader 选举：补充 PK 细节（初始投自己、改票机制、投票箱计数超半数确认）
- 八、关键权衡：补充 ZK 作为注册中心的优劣分析（内存+NIO性能、CP代价、推荐用Nacos/Eureka）

## [2026-05-16] Add | 概念-k8s.md — 摄入 10-k8s
- 整理 K8s 架构、核心对象、Pod 生命周期、调度/HPA、Service/Ingress/CNI、PV/PVC/CSI、安全、发布运维与 Helm
- L7 索引新增 K8s/Kubernetes、Pod/Deployment/Service、Ingress/HPA 关键词
- `机制-Docker.md` related 增加 `[[概念-k8s]]`

## [2026-05-16] Update | 概念-Java集合.md — 补充 CopyOnWriteArrayList 底层原理
- 新增 3.3 CopyOnWriteArrayList 核心问题：写时复制流程（加锁→复制新数组→写新数组→切换volatile引用→解锁）、读无锁
- 补充关键特性表：volatile array、ReentrantLock写、无锁读、迭代器快照
- 补充适用与不适用场景：读多写极少适合；数据量大+写频繁、实时性要求高不适合

## [2026-05-16] Update | 合并 概念-数据结构.md → 概念-Java集合.md
- 概念-Java集合.md 新增 2.6（线性结构基础：数组·链表·栈·队列 + 四者选型）
- 概念-Java集合.md 新增 2.7（Set去重机制 + Comparable vs Comparator）
- 概念-Java集合.md 新增 九、L4数据结构全景（结构全景图/高频考点速查/结构间依赖/上下层关系）
- 概念-Java集合.md 生产风险强化：ArrayList.subList()陷阱 + 序列化说明
- 概念-Java集合.md frontmatter：合并 related/sources/aliases
- 概念-Java集合.md 七、与其他概念的关系 更新为完整链接
- 7个引用文件中 [[概念-Java集合]] 替换为 [[概念-Java集合]]
- 删除 wiki/concepts/02-java/概念-数据结构.md
- 重建索引：89页（-1）

## [2026-05-16] Update | 概念-分布式理论.md — 服务雪崩/熔断/降级
- 新增 aliases：服务雪崩、熔断、降级、服务限流
- 新增 3 个 source 文件（微服务/高并发/SpringCloud）
- 三、BASE 理论 快速导航补充服务雪崩/熔断/降级说明
- 新增「服务雪崩、限流、熔断与降级」子节：雪崩定义、三层防护表、熔断渐进式恢复流程、降级 FailOver 机制、熔断 vs 降级对比表
- 八、关键权衡新增「熔断 vs 降级」行

## [2026-05-16] Update | 概念-分布式事务.md — 本地消息表/事务消息/Seata
- 新增 6 个 source 文件（本地消息表/事务消息/Seata AT/XA/4种模式）
- 五、本地消息表：补充订单+库存具体示例流程，完整状态机，强调下游幂等
- 五、事务消息（RocketMQ）：细化 7 步流程，明确 half 消息对 Consumer 不可见，加入死信队列兜底说明
- 六、Seata：开篇说明"底层基于两阶段提交理论"，TC/TM/RM 对应协调者/参与者

## [2026-05-16] Update | 机制-Nacos.md — 注册中心 + 配置中心补充
- 新增 aliases：雪崩保护/保护阈值/Namespace/Group/DataId/RefreshScope/配置优先级
- 新增 source：09-微服务/11-nacos.md
- 三、注册中心：新增「雪崩保护（保护阈值）」：健康实例/总实例 < 阈值时不健康实例也加入列表；「临时实例 vs 持久实例」对比表（ephemeral=false）
- 二、配置中心：新增 Namespace/Group/DataId 三元组表；vs Spring Cloud Config 对比表（Git依赖/动态感知/UI/复杂度）；配置优先级（C>B>A，精准>扩展>共享）；@RefreshScope 代码示例及注意事项

## [2026-05-16] Update | 机制-Kafka.md —  Kafka 笔记合并
- 新增 aliases：消费者组状态/批量消费/Controller选举/Partition Leader选举/Topic vs Partition
- 二、核心架构：补充 Topic vs Partition 存在理由（吞吐/负载均衡/扩展性）
- 四、ISR：新增版本演进表（<0.9.x lag.max.messages → ≥0.9.x lag.max.ms）；「不能100%不丢」改为3端分析表（生产者/Broker/消费者原因+最佳实践）
- 五、重平衡：新增消费者组5种状态机（Empty/Dead/PreparingRebalance/CompletingRebalance/Stable）及状态流转说明
- 新增 六、选举机制：Partition Leader选举（ZK临时节点+最小序列号机制）；Controller选举（/controller竞争写入）；两者均说明KRaft 4.0优化
- 七→十一各节编号顺移
- 十、大流量实战：新增批量消费最佳实践（@KafkaListener + setBatchListener + CompletionService + finally陷阱说明）

## [2026-05-16] Update | 机制-SpringBoot.md —  SpringBoot 笔记合并
- 新增 source：07-springBoot.md
- 新增 aliases：MANIFEST.MF/配置文件加载顺序/jar启动/@SpringBootApplication/@ComponentScan
- 三、Java 核心使用：新增 3.0 核心注解底层——@SpringBootApplication 三元注解拆解表；@Bean 底层（方法名=beanName + CGLIB 单例代理）
- 二、核心机制：新增 2.5 配置文件加载顺序（8级优先级表，记忆规则：外>内/profile>通用/命令行>配置文件）；新增 2.6 jar 启动原理（MANIFEST.MF → Main-Class JarLauncher + Start-Class → fat jar 类加载器说明）

## [2026-05-16] Update | 机制-Dubbo.md —  05-doubble.md 合并
- 新增 source：05-doubble.md
- 新增 aliases：服务导出/服务引入/平滑加权轮询/Directory/Invoker/Exporter
- 三、Dubbo 整体架构：分层架构表新增"关键组件"列（Protocol三件套/Cluster四组件/Exchange四组件/Transport四组件/Serialize核心接口）；新增"RPC最小三件套"总结（Protocol+Invoker+Exporter）
- 新增 四、服务导出与引入流程：Provider端5步（注解解析→export→注册中心→配置监听→Netty/Tomcat启动）；Consumer端4步（注解解析→Directory→监听器→代理注入Spring）
- 原四→五（透明代理），五→六（负载均衡），六→七（通信协议），七→八（SPI），八→九（服务治理），九→十（优雅停机），十→十一（gRPC），十一→十二（权衡），十二→十三（关系），十三→十四（边界）
- 六、LB策略：RoundRobin补充平滑加权轮询算法；ConsistentHash补充游戏分区等具体场景
