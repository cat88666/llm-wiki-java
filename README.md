# LLM-Wiki-Java 后端工程师知识体系

> 基于 Karpathy LLM Wiki 架构的 Java 知识编译工程。  
> 原始笔记只读，由 AI 持续编译为结构稳定的思维模型。

**入口：[知识库总索引](llm-wiki-java/wiki/index.md)** | [更新日志](llm-wiki-java/wiki/log.md) | [维护规范](llm-wiki-java/CLAUDE.md)

---

## 仓库结构

```text
llm-wiki-java/
├── raw/
│   ├── note/            ← Hollis 原始笔记（只读）
│   └── articles/        ← 技术文章（只读）
└── wiki/
    ├── index.md         ← 全局索引（自动生成）
    ├── index.meta.toml  ← 索引配置（手动维护）
    ├── log.md           ← 操作日志
    ├── concepts/        ← 概念与机制
    ├── summaries/       ← 主题聚合
    ├── synthesis/       ← 综合分析
    └── templates/       ← 标准模板
```

---

## 核心指令

| 指令 | 说明 |
|------|------|
| `/ingest [路径]` | 把 raw/ 原始笔记编译进 wiki 知识结构 |
| `/query <问题>` | 基于本地 wiki 检索并回答问题 |
| `/lint` | 检查知识库结构健康状态 |
