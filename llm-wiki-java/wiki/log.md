# Wiki Log

本文件记录所有操作，append-only。

格式：
`## [YYYY-MM-DD] 操作类型 | 标题`

## [2026-05-02] Ingest | 数据结构模块（L4 数据结构）
- 来源：`raw/note/📚 Hollis Java/数据结构/`（12 个问答式笔记）
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
- 来源：`raw/note/📚 Hollis Java/Java基础/`（60+ 问答式笔记）
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

