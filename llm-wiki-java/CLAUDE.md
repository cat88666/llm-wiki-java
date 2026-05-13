# llm-wiki-java — 实例执行规范

> 本文件是 `llm-wiki-java/` 实例的操作规范，Claude Code 每次执行操作前必须读取。  
> 角色定位：不是问答机器人，而是这个 wiki 的主动维护者和编译者。

---

## 一、目录结构

```text
llm-wiki-java/
├── raw/                        ← 只读原始资料（永不修改）
│   ├── note/Hollis/              ← Hollis 课程笔记（问答式）
│   │   ├── Java基础/
│   │   ├── Java并发/
│   │   ├── JVM/
│   │   ├── MySQL/
│   │   ├── Redis/
│   │   ├── Spring/
│   │   ├── SpringCloud/
│   │   ├── Dubbo/
│   │   ├── Zookeeper/
│   │   ├── RabbitMQ/
│   │   ├── ElasticSearch/
│   │   ├── MyBatis/
│   │   ├── 数据结构/
│   │   ├── 容器/
│   │   ├── Tomcat/
│   │   ├── IDEA/
│   │   ├── Maven&Git/
│   │   ├── 面经实战/
│   │   └── 非技术问题/
│   └── articles/               ← 技术文章原文
└── wiki/                       ← AI 编译的结构化知识库
    ├── index.md                ← 自动生成的总索引，不直接手改
    ├── index.meta.toml         ← 手动维护的索引配置
    ├── log.md                  ← 操作日志，append-only
    ├── concepts/               ← 抽象概念（机制 / 原理 / 模型 / 算法）
    ├── entities/               ← 具体实体（框架 / 工具 / 组件 / 规范）
    ├── summaries/              ← 每个知识主题一页聚合总结
    ├── synthesis/              ← 问题驱动的比较、策略、判断
    └── templates/              ← 标准模板文件
```

---

## 二、知识分层与 concepts/ 归属

| 层级 | 标签 | 典型 concept 页 | 来源模块 |
|------|------|----------------|---------|
| L1 语言基础 | `#java-lang` | 动态代理、泛型擦除、异常体系 | Java基础 |
| L2 运行时 | `#jvm` | 类加载机制、GC算法、JMM | JVM |
| L3 并发 | `#concurrency` | AQS、volatile、线程池模型 | Java并发 |
| L4 数据结构 | `#data-structure` | 红黑树、B+树、跳表 | 数据结构、容器 |
| L5 存储 | `#storage` | MVCC、索引模型、缓存雪崩 | MySQL、Redis、MyBatis |
| L6 框架 | `#framework` | IoC容器模型、AOP织入 | Spring、SpringBoot |
| L7 分布式 | `#distributed` | Raft一致性、服务注册发现 | SpringCloud、Dubbo、Zookeeper、RabbitMQ、ES |
| L8 实践 | `#practice` | 限流算法、秒杀设计 | 面经实战 |

---

## 三、concepts/ 页面结构（6部分，缺一不可）

```markdown
# [概念名]

> 一句话定义

## 第一性原理
[为什么这个概念存在？解决了什么根本问题？]

## 核心机制
[它是如何运作的？关键数据结构或步骤]

## 关键权衡
[这个设计的代价是什么？什么情况下它会失效？]

## 与其他概念的关系
- 区别于 [[X]]：
- 依赖 [[Y]]：
- 支撑了 [[Z]]：

## 应用边界
[什么时候用它？什么时候不该用？]
```

---

## 四、三种核心操作

### 4.1 Ingest（摄入）

1. 读取指定 `raw/` 目录下所有文件，完整理解内容
2. 提取：核心概念、底层机制、关键权衡、典型误区
3. 判断归属：
   - 抽象机制/原理 → `concepts/`
   - 具体框架/工具 → `entities/`
   - 模块知识聚合 → `summaries/`
   - 跨层比较/设计 → `synthesis/`
4. 优先更新已有页面，再新建
5. 必要时更新 `wiki/index.meta.toml`，运行 `python3 scripts/build_index.py` 生成 `wiki/index.md`，追加 `wiki/log.md`

**Java 特定规则：**
- 原始笔记是"✅问题" → 提取其背后的概念，不要照搬问答形式
- 同一概念在多个模块出现（如红黑树在数据结构和集合框架均有）→ 合并到同一 concept 页
- 面经实战笔记 → 提取高频考点组合写入 `synthesis/`，不单独为每道题建页

### 4.2 Query（查询）

1. 读取 `wiki/index.md` 定位相关页面
2. 读取相关 concept / entity / summary / synthesis 页
3. 综合回答，标注来源
4. 判断是否有回写价值 → 回写到 `summaries/` 或 `synthesis/`
5. 追加 `wiki/log.md`

### 4.3 Lint（健康检查）

扫描所有 wiki 页面，检查：
- 矛盾：同一概念不同页面描述不一致
- 孤立：没有被任何页面引用
- 重复：两个 concept 页描述同一概念
- 失效链接：`[[X]]` 引用但文件不存在
- 缺少来源：有结论但无 `raw/` 路径引用
- 分层错误：L5 concept 页没有链接 L3/L4 依赖

---

## 五、页面格式规范

- 内部链接：Obsidian 风格 `[[页面名]]`
- 来源路径：相对路径 `../../raw/note/Hollis/模块名/文件名.md`
- 文件命名：遵循 `.claude/skills/naming/SKILL.md` 规则
- 层级标签：每个 concept 页 frontmatter 中必须包含 `layer` 字段（L1-L8）
- 重构 `wiki/concepts/` 页面时，默认使用 `wiki/templates/rewrite-concept-prompt.md` 中的重写模板：保留 frontmatter/sources，增加快速导航，二级标题使用中文编号，内容按第一性原理、核心机制、Java 核心使用、使用原则、案例、综合对比、生产风险、概念关系、应用边界组织。

---

## 六、index.md 与 log.md 更新规范

**index.md** 是生成文件，不直接手改。

手动维护入口：
- `wiki/index.meta.toml`：维护 L0-L8 层级、主题入口、综合分析入口、关键词。
- `wiki/concepts/`、`wiki/summaries/`、`wiki/synthesis/`：新增或调整知识页面。

生成与校验：
```bash
python3 scripts/build_index.py
python3 scripts/build_index.py --check
```

**log.md** 格式：
```markdown
## [YYYY-MM-DD] 操作类型 | 标题
- 关键动作1
- 关键动作2
```

操作类型：`Init` / `Ingest` / `Query` / `Lint` / `Update`

---

## 七、行为准则

1. **不臆造**：只呈现来源中实际存在的内容
2. **建立链接**：每个页面至少引用 1 个来源，与至少 1 个其他 wiki 页面链接
3. **增量更新**：优先更新已有页面，而非总是新建
4. **标注矛盾**：发现矛盾在 `lint_notes` 字段标注，不默默覆盖
5. **简洁优先**：concepts/ 核心机制不超过 5 个要点
