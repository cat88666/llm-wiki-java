# llm-wiki-java

Java 后端工程师知识库。

本仓库把 `raw/` 中分散的问答式笔记、面经和专题资料，整理为 `wiki/` 下按知识层级组织的结构化知识体系。核心目标不是保存原文，而是沉淀可检索、可串联、可用于复习和面试表达的概念图谱。

## 快速入口

- [知识库总索引](wiki/index.md)
- [更新日志](wiki/log.md)
- [维护规范](CLAUDE.md)

`wiki/index.md` 是主要入口，由脚本自动生成。索引用一张表按 L0-L8 分层组织，并在最后一列提供可点击关键词，例如进程、线程、JVM、AQS、InnoDB、Kafka、DDD、秒杀等。

## 当前规模

| 类型 | 目录 | 数量 | 说明 |
| --- | --- | ---: | --- |
| 概念与机制 | `wiki/concepts/` | 78 | 第一性原理、核心机制、权衡和应用边界 |
| 主题聚合 | `wiki/summaries/` | 12 | Java 基础、JVM、并发、MySQL、Redis、Spring、三高等体系化总结 |
| 综合分析 | `wiki/synthesis/` | 25 | 系统设计、架构权衡、场景题和面试表达 |
| 维护日志 | `wiki/log.md` | 1 | 记录知识库更新动作 |

除 `wiki/index.md` 外，当前 `wiki/` 共 116 个 Markdown 页面，并已全部被总索引覆盖。

## 知识分层

| 层级 | 知识域 | 典型内容 |
| --- | --- | --- |
| L0 | 计算机基础 | 进程、线程、协程、网络协议、IO 模型 |
| L1 | Java 基础 | 值类型、OOP、异常、泛型、反射、SPI、序列化 |
| L2 | JVM | 内存模型、类加载、GC、JIT、调优 |
| L3 | Java 并发 | JMM、ThreadLocal、synchronized、volatile、CAS、AQS、线程池 |
| L4 | 集合、算法、数据结构 | HashMap、ConcurrentHashMap、红黑树、B+ 树、堆、布隆过滤器 |
| L5 | Redis、MySQL、存储 | SQL 优化、InnoDB、MVCC、MySQL 日志、Redis 锁、缓存问题 |
| L6 | Spring 与框架 | IoC、AOP、事务、自动装配、SpringMVC、MyBatis、设计模式 |
| L7 | 分布式与中间件 | 分布式理论、MQ、SpringCloud、Nacos、Dubbo、Kafka、Netty、OAuth2 |
| L8 | 架构与实践 | DDD、秒杀、短链、支付、容量规划、接口防护、项目难点表达 |

## 目录说明

```text
.
├── raw/                 # 原始资料，只作为来源，不直接当作知识库入口
├── scripts/
│   └── build_index.py   # 根据 index.meta.toml 自动生成 wiki/index.md
├── wiki/
│   ├── index.md         # 自动生成的总索引，覆盖所有 wiki 页面
│   ├── index.meta.toml  # 手动维护的索引配置：层级、主题入口、关键词
│   ├── log.md           # 更新日志
│   ├── concepts/        # 概念、机制、模型、算法
│   ├── summaries/       # 主题级聚合总结
│   └── synthesis/       # 问题驱动的设计、比较、权衡和面试表达
├── CLAUDE.md            # 维护规范
└── README.md            # 项目入口说明
```

## 使用方式

1. 从 [知识库总索引](wiki/index.md) 进入。
2. 如果要按体系复习，优先看 `主题` 列。
3. 如果要查一个具体概念，优先看 `关键词` 列。
4. 如果要准备系统设计或高阶面试，优先看 `综合分析` 列。
5. 如果要追溯维护记录，看表格里的 `M / 维护` 行，或直接看 [wiki/log.md](wiki/log.md)。

## 索引维护

不要直接手改 `wiki/index.md`。它是生成文件。

日常只需要维护：

- `wiki/index.meta.toml`：层级名称、主题入口、综合分析入口、关键词。
- `wiki/concepts/`、`wiki/summaries/`、`wiki/synthesis/`：新增或调整知识页面。

重新生成索引：

```bash
python3 scripts/build_index.py
```

检查索引是否最新、是否覆盖所有 wiki 页面、是否存在坏链：

```bash
python3 scripts/build_index.py --check
```

## 维护原则

- `raw/` 保持只读，用作来源资料。
- 新增或调整 `wiki/` 页面后，必要时更新 `wiki/index.meta.toml`，再运行 `python3 scripts/build_index.py`。
- 总索引必须覆盖所有 wiki Markdown 页面，且不能出现失效链接。
- 概念页优先写清第一性原理、核心机制、关键权衡和应用边界。
- 综合分析页优先服务真实问题：系统设计、选型权衡、面试追问和项目表达。
