# Wiki Log

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

- 重写 `概念-Java基础类型.md`：增加快速导航、中文编号标题、生产风险表格、应用边界表格
- 重写 `概念-Java异常体系.md`：增加 try-with-resources、自定义异常示例、综合对比表格、生产风险表格
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

- `系统设计-在线游戏平台架构` → `系统设计-游戏架构系统`
- `系统设计-IM即时通信系统` → `系统设计-IM通信系统`
- `系统设计-彩票系统核心设计` → `系统设计-彩票系统`
- 一批 `设计-*` 系统类页面统一改为 `系统设计-*`（秒杀/短链/区块链OTC/游戏/彩票/IM）
- 同步更新全部交叉引用及 `index.md`

## [2026-05-13] Merge | 6 个碎片页收口到已有页面

- `设计-线上问题排查` → 并入 `概念-可观测性`（删除）
- `设计-大厂秒杀实践` → 并入 `系统设计-秒杀系统`（删除）
- `设计-消息队列场景综合` → 并入 `机制-消息队列可靠性`（删除）
- `设计-JVM调优实战` → 并入 `机制-GC算法与垃圾收集器`（删除）
- `设计-云原生选型` → 并入 `机制-容器化与Docker`（删除）
- `设计-MySQL大表与查询优化` → 并入 `概念-SQL查询优化`（删除）

## [2026-05-13] Merge | DDD/支付专题收口

- `设计-DDD落地实战` → 并入 `概念-DDD`（删除）
- `设计-支付系统设计` + `设计-账户钱包系统` → 并入 `mock-interview-eson`（删除 2 页）

## [2026-05-12] Update | Wiki 合并整理 — 删除 8 个碎片页，新建 3 个合并页

- 反射 + 动态代理 → `机制-反射与动态代理`
- String不可变性 + 包装类 → `概念-Java基础值类型`
- BitMap + 布隆过滤器 → `概念-BitMap`
- 引用类型 → 并入 `机制-GC算法与垃圾收集器`
- Java集合框架 → 并入 `概念-数据结构`

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
- 新建 synthesis × 3：`设计-区块链OTC平台` / `设计-IM即时通信系统` / `设计-Gacha与游戏高并发`
- 更新 `机制-Netty`：新增 TCP 参数调优章节

## [2026-05-11] Ingest | 模拟面试 — Eson 简历

- 新建 `synthesis/mock-interview-eson.md`：7 阶段 60 分钟面试脚本

## [2026-05-09] Ingest | 彩票系统岗位专项

- 新建 `synthesis/设计-彩票系统核心设计` / `设计-账户钱包系统`
- 更新 `机制-Kafka`：新增大流量实战补充

## [2026-05-09] Ingest | P2+P3 批次：DDD/P8 边界/云原生

- 新建 `synthesis/设计-DDD落地实战` / `设计-P8高频边界问题` / `设计-云原生选型`

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

- 新建 `机制-Seata框架机制` / `机制-SpringMVC请求处理链` / `概念-OAuth2授权协议`
- 更新 `机制-SpringBoot自动装配`：完整启动流程 + 优雅停机

## [2026-05-08] Ingest | P7 第一轮：消息队列/配置中心/任务调度/异步编程/JVM 调优

- 新建 `机制-RabbitMQ` / `机制-配置中心与Nacos` / `机制-分布式任务调度` / `机制-CompletableFuture与异步编程` / `设计-JVM调优实战`
- 新建 `summaries/主题-消息队列体系`
- 更新 `机制-IoC容器`：循环依赖补充

## [2026-05-07] Ingest | Interview/Eson → 4 缺失知识点

- 新建 `机制-对象池技术` / `机制-RocksDB` / `设计-多级缓存架构` / `设计-支付系统设计`

## [2026-05-07] Lint | 全库健康检查

- 修复 5 处断链、4 个 sources 路径
- 知识库：concepts×69 / summaries×8 / synthesis×8

## [2026-05-07] Ingest | tuling/ → Kafka + Netty + DDD + Nacos

- 新建 `机制-Kafka` / `机制-Netty` / `概念-DDD`
- 更新 `机制-微服务与SpringCloud`：Nacos 注册中心章节

## [2026-05-07] Ingest | 架构体系/ → JDK 新特性 + 可观测性 + 算法高频

- 新建 `概念-JDK新特性` / `概念-可观测性` / `设计-算法高频题型`
- 更新 `概念-分布式系统理论`：Paxos/Raft
- 更新 `机制-RPC与Dubbo`：gRPC + 序列化对比
- 更新 `概念-DDD`：CQRS + Event Sourcing

## [2026-05-07] Ingest | 多模块批量摄入

- 项目经验/ → `synthesis/设计-线上问题排查`（25 实战案例）
- 网络安全/ → `concepts/概念-网络安全`（13 文件）
- 微服务/ → 更新 `机制-微服务与SpringCloud`（5 新章节）
- 分布式/ → `概念-分布式系统理论` / `概念-幂等设计` / `机制-Canal数据同步`
- 设计模式/ → `机制-设计模式`（21 文件）
- 操作系统/ → `概念-操作系统`（30 文件）
- 计算机网络/ → `概念-计算机网络`（30 文件）
- 容器/ + 高并发/ + 高可用/ → `机制-容器化与Docker` / `概念-限流与熔断` / `概念-高可用设计`
- 高性能/ + 架构设计/ → `概念-布隆过滤器` / `概念-读写分离` + 微服务拆分原则
- 场景题/ + 面经实战/ → `设计-Redis实战场景` / `设计-短链服务` / `设计-订单超时关闭`

## [2026-05-06] Ingest | 初始批量摄入

- 面经实战/ → synthesis × 3：秒杀系统/分布式事务/分库分表
- SpringCloud/ → `机制-微服务与SpringCloud`
- Dubbo/ → `机制-RPC与Dubbo`
- ElasticSearch/ → `机制-倒排索引与ElasticSearch`
- RabbitMQ/ → `机制-消息队列可靠性`
- Zookeeper/ → `机制-ZAB协议与Zookeeper`
- Spring/ → concept × 4 + `主题-Spring体系`
- MyBatis/ → `机制-MyBatis`
- 集合类/ → `机制-HashMap` / `机制-ConcurrentHashMap` + `主题-Java集合框架`

## [2026-05-06] Lint | 健康检查

- 删除 4 重复页：机制-HashMap / 机制-ConcurrentHashMap / 机制-CopyOnWrite / 概念-集合迭代一致性
- 修复 `主题-Redis体系` 畸形 wikilink

## [2026-05-02] Ingest | 基础模块批量摄入

- Redis/ → concept × 5 + `主题-Redis体系`
- MySQL/ → concept × 5 + `主题-MySQL体系`
- Java并发/ → concept × 7 + `主题-Java并发体系`
- JVM/ → concept × 5 + `主题-JVM体系`
- 数据结构/ → concept × 7 + `主题-数据结构体系`
- Java基础/ → concept × 11 + `主题-Java语言基础`

## [2026-05-02] Init | Java 知识库初始化

- 重命名 `llm-wiki-template/` → `llm-wiki-java/`
- 重写根目录/实例 `CLAUDE.md`，建立 L1-L8 知识结构
- Hollis 来源目录更名，清理 Notion 导出文件名 ID
