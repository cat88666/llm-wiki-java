# LLM-Wiki / Java 后端工程师知识体系

> 基于 Karpathy LLM Wiki 架构的 Java 知识编译工程。  
> 原始笔记只读，由 AI 持续编译为结构稳定的思维模型。

---

## 一、知识结构图

```
Java 后端工程师知识体系
│
├── L1 语言基础
│   ├── 类型系统（基本类型 / 包装类 / String / BigDecimal）
│   ├── OOP（封装 / 继承 / 多态 / 接口）
│   ├── 反射 · 注解 · 动态代理（JDK / CGLIB）
│   ├── 异常体系（Checked / Unchecked）
│   ├── 泛型与类型擦除
│   └── 序列化与 I/O / 文件处理
│
├── L2 运行时（JVM）
│   ├── 内存结构（堆 / 栈 / 方法区 / 直接内存）
│   ├── 类加载机制（双亲委派 / 热部署）
│   ├── GC 算法与收集器（CMS / G1 / ZGC）
│   └── JVM 调优（参数 / 工具 / 排查 OOM）
│
├── L3 并发编程
│   ├── Java 内存模型（JMM / happens-before / volatile）
│   ├── 线程与线程池（ThreadPoolExecutor / ForkJoin / 定时任务）
│   ├── 锁机制（synchronized / AQS / ReentrantLock）
│   ├── 并发容器（ConcurrentHashMap / CopyOnWriteArrayList）
│   └── 并发工具（CountDownLatch / Semaphore / CompletableFuture）
│
├── L4 数据结构与算法
│   ├── 集合框架（ArrayList / LinkedList / HashMap / TreeMap）
│   ├── 树结构（二叉树 / 红黑树 / B+树 / 前缀树）
│   ├── 堆与优先队列
│   ├── 图（有向 / 无向 / 最短路径）
│   └── 编程题（排序 / 搜索 / 动态规划 / 智商题）
│
├── L5 存储层
│   ├── MySQL（索引 / MVCC / 锁 / 分库分表）
│   ├── Redis（数据结构 / 持久化 / 集群 / 缓存设计）
│   ├── 本地缓存（Guava / Caffeine）
│   └── MyBatis（ORM / 动态 SQL / 缓存）
│
├── L6 应用框架
│   ├── Spring（IoC / AOP / 事务）
│   ├── SpringBoot（自动装配 / SPI / Starter）
│   ├── SpringMVC（请求链路 / 拦截器）
│   ├── 容器（Docker / K8s 基础）
│   ├── 日志（Log4j / SLF4J / ELK）
│   └── 单元测试（JUnit / Mockito）
│
├── L7 分布式体系
│   ├── 理论基础（CAP / BASE / Raft / Paxos）
│   ├── 消息队列（RabbitMQ — 可靠性 / 顺序 / 幂等）
│   ├── 服务治理（Zookeeper / Dubbo）
│   ├── 微服务（SpringCloud — 网关 / 熔断 / 配置中心）
│   ├── 全文搜索（ElasticSearch）
│   ├── 分布式锁 / 分布式事务 / 分布式 ID
│   └── 云计算基础（网络安全 / 计算机网络）
│
├── L8 系统设计
│   ├── 高并发（限流 / 削峰 / 秒杀 / 幂等）
│   ├── 高可用（降级 / 熔断 / 容灾）
│   ├── 高性能（缓存分层 / 异步 / 批处理）
│   ├── 架构设计（DDD / 微服务拆分 / 设计模式）
│   └── 操作系统（进程 / 线程 / IO 模型）
│
└── L9 工程实践
    ├── 场景题（系统设计实战）
    ├── 大厂实践（真实案例与踩坑）
    ├── 项目难点 & 亮点（STAR 结构化表达）
    └── 面经实战（高频考点组合 / 非技术问题）
```

---

## 二、仓库结构

```text
llm-wiki-java/
├── llm-wiki-java/           ← Java 知识库实例（唯一活跃实例）
│   ├── raw/
│   │   ├── note/            ← Hollis Java 原始笔记（只读）
│   │   └── articles/        ← 技术文章（只读）
│   └── wiki/
│       ├── index.md         ← 全局索引（每次操作后更新）
│       ├── log.md           ← 操作日志（append-only）
│       ├── concepts/        ← 抽象概念页（机制 / 原理 / 模型）
│       ├── entities/        ← 具体实体页（框架 / 工具 / 组件）
│       ├── summaries/       ← 每个知识主题一页聚合总结
│       ├── synthesis/       ← 问题驱动的比较、策略、判断
│       └── templates/       ← 标准模板
└── assets/
```

---

## 三、核心指令

| 指令 | 说明 |
|------|------|
| `/ingest [路径]` | 把 raw/ 原始笔记编译进 wiki 知识结构 |
| `/query <问题>` | 基于本地 wiki 检索并回答问题 |
| `/lint` | 检查知识库结构健康状态 |

---

## 四、摄入顺序规划

按知识依赖层级从底层往上摄入，确保概念页建立时有足够上下文：

| 阶段 | 模块（raw/note/ 目录） | 目标层 |
|------|----------------------|--------|
| 第一轮 | Java基础、集合类、数据结构、编程题 | L1 + L4 地基 |
| 第二轮 | JVM、Java并发、定时任务 | L2 + L3 运行时 |
| 第三轮 | MySQL、Redis、本地缓存、MyBatis | L5 存储层 |
| 第四轮 | Spring、容器、日志、单元测试 | L6 框架层 |
| 第五轮 | SpringCloud、Dubbo、Zookeeper、RabbitMQ、ElasticSearch、分布式、配置中心 | L7 分布式 |
| 第六轮 | 高并发、高可用、高性能、架构设计、设计模式、操作系统、计算机网络 | L8 系统设计 |
| 第七轮 | 场景题、大厂实践、项目难点&亮点、面经实战、非技术问题 | L9 实践综合 |
