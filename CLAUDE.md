# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库用途

受约束的 Karpathy LLM Wiki 工程，面向 **Java 后端工程师知识体系**。  
把 `raw/` 中的 Hollis 问答式笔记**编译**为 `wiki/` 中以第一性原理组织的概念图谱。

活跃实例：`llm-wiki-java/`（Java 后端工程师知识体系）  
规则母版：`llm-wiki-java/CLAUDE.md`（实例级执行规范）

## 技能指令

| 指令 | 触发条件 |
|------|---------|
| `/ingest [路径]` | 用户要摄入/导入/纳入资料到知识库 |
| `/query <问题>` | 用户要基于 wiki 提问 |
| `/lint` | 用户要检查知识库健康状态 |

详细执行流水线见 `.claude/skills/` 对应 SKILL.md。  
命名规则的唯一权威：`.claude/skills/naming/SKILL.md`。

## 铁律（任何操作均不可违反）

1. **`raw/` 只读**：绝对不写入、移动或修改 `raw/` 下任何文件
2. **操作后必须收尾**：每次 Ingest / Query 回写 / Lint 后，必须更新 `llm-wiki-java/wiki/index.meta.toml`，运行 `python3 scripts/build_index.py` 重新生成 `llm-wiki-java/wiki/index.md`，并追加 `llm-wiki-java/wiki/log.md`
3. **新建页面前必须命名**：任何新文件落盘前，先按 `naming/SKILL.md` 规则确定文件名，禁止自由命名
4. **优先更新，不轻易新建**：新资料优先合并进已有页面，确实需要新主题时才新建
5. **`concepts/` 页面结构**：必须包含以下 6 个部分，缺一不可：
   > 一句话定义 → 第一性原理 → 核心机制 → 关键权衡 → 与其他概念的关系 → 应用边界

## Java 知识编译原则

- **问答 → 概念**：原始笔记是"✅什么是红黑树？"，编译后是 `concepts/红黑树.md`（含第一性原理）
- **碎片 → 主题**：同模块多个问答合并为一个 `summaries/` 主题页
- **跨层 → 综合**：涉及多层（如"HashMap 在并发下如何选型"）写入 `synthesis/`
- **依赖方向**：L1→L2→L3→L4→L5→L6→L7 依次构建，后层概念页需链接前层
