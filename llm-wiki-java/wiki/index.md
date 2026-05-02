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
<!-- 待摄入：JVM/ -->

### L3 并发编程
<!-- 待摄入：Java并发/ -->

### L4 数据结构
- [[概念-线性数据结构]](concepts/概念-线性数据结构.md) — 数组/链表/栈/队列，访问模式决定选择 `#data-structure`
- [[机制-红黑树]](concepts/机制-红黑树.md) — 5条规则近似平衡，O(log n) 增删查，HashMap/TreeMap 底层 `#data-structure`
- [[机制-B树与B加树]](concepts/机制-B树与B加树.md) — 多路平衡树，低树高减少磁盘IO，MySQL InnoDB 索引底层 `#data-structure`
- [[机制-堆与优先队列]](concepts/机制-堆与优先队列.md) — 完全二叉树 + 数组，Top K 用小顶堆，PriorityQueue `#data-structure`
- [[概念-前缀树]](concepts/概念-前缀树.md) — 共享公共前缀，O(m) 字符串检索，搜索补全/AC自动机 `#data-structure`
- [[概念-BitMap]](concepts/概念-BitMap.md) — 1 bit 标记整数存在性，极致空间压缩，布隆过滤器基础 `#data-structure`
- [[概念-图论基础]](concepts/概念-图论基础.md) — 多对多关系，DFS/BFS 两种遍历，Dijkstra 依赖小顶堆 `#data-structure`

### L5 存储层
<!-- 待摄入：MySQL/、Redis/、MyBatis/ -->

### L6 应用框架
<!-- 待摄入：Spring/、SpringBoot -->

### L7 分布式体系
<!-- 待摄入：SpringCloud/、Dubbo/、Zookeeper/、RabbitMQ/、ElasticSearch/ -->

### L8 工程实践
<!-- 待摄入：面经实战/ -->

---

## Entities（实体页）

> 具体框架、工具、组件、规范。有明确身份边界的具体对象。

<!-- 待摄入后填充 -->

---

## Summaries（主题总结页）

> 每页对应一个知识主题，聚合多个来源。不按源文件逐篇生成。

- [[主题-Java语言基础]](summaries/主题-Java语言基础.md) — L1 概念地图 + 高频考点汇总，含 11 个 concept 页的依赖关系 `#java-lang`
- [[主题-数据结构体系]](summaries/主题-数据结构体系.md) — L4 数据结构知识地图 + 高频考点，含与 L5 MySQL/Redis 的联系 `#data-structure`

---

## Synthesis（综合分析页）

> 围绕一个问题、比较或判断展开。涉及跨层或跨模块知识时使用。

<!-- 待摄入后填充 -->
