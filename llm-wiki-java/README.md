# llm-wiki-java

Java 后端工程师知识库。把 `raw/` 中的问答式笔记编译为 `wiki/` 下按层级组织的概念图谱。

**入口：[知识库总索引](wiki/index.md)** | [更新日志](wiki/log.md) | [维护规范](CLAUDE.md)

## 知识分层

| 层级 | 知识域 | 典型内容 |
| --- | --- | --- |
| L0 | 计算机基础 | 进程/线程/协程、网络协议、IO 模型 |
| L1 | Java 基础 | 值类型、OOP、异常、泛型、反射、SPI |
| L2 | JVM | 内存模型、类加载、GC、JIT |
| L3 | 并发 | JMM、synchronized、volatile、CAS、AQS、线程池 |
| L4 | 集合/数据结构 | HashMap、红黑树、B+ 树、堆、布隆过滤器 |
| L5 | 存储 | InnoDB、MVCC、Redis、缓存 |
| L6 | 框架 | IoC、AOP、事务、SpringMVC、MyBatis |
| L7 | 分布式 | MQ、Kafka、Dubbo、Netty、Nacos |
| L8 | 架构实践 | DDD、秒杀、支付、容量规划、项目表达 |

## 当前规模

| 类型 | 目录 | 数量 | 说明 |
| --- | --- | --- | --- |
| 概念与机制 | `wiki/concepts/` | 78 | 第一性原理、核心机制、权衡 |
| 主题聚合 | `wiki/summaries/` | 12 | 体系化总结 |
| 综合分析 | `wiki/synthesis/` | 25 | 系统设计、场景题、面试表达 |

## 目录结构

```text
raw/          # 只读原始资料
wiki/
├── index.md          # 自动生成的总索引
├── index.meta.toml   # 索引配置（手动维护）
├── log.md            # 更新日志
├── concepts/         # 概念与机制
├── summaries/        # 主题聚合
└── synthesis/        # 综合分析
scripts/
└── build_index.py    # 生成 index.md
```

## 索引维护

```bash
python3 scripts/build_index.py          # 重新生成
python3 scripts/build_index.py --check  # 检查覆盖率和坏链
```

不要直接改 `wiki/index.md`，只需维护 `wiki/index.meta.toml` 和各 `wiki/` 子目录。
