# d2d — AI Agent 快速上下文

> 本文件供 AI agent 首次进入项目时快速理解全局。

## 这是什么

d2d (Doc-to-Doc) 是一个 **Claude Code skill**，实现跨模型对抗式**文档审查**。核心逻辑在 `SKILL.md`，复用 a2a 的环境检测脚本。

与 a2a 的区别：a2a 审查代码（基于 git diff），d2d 审查任意文本内容（研究报告、数据分析、商业计划等）。

## 文件职责

| 文件 | 作用 | 什么时候改 |
|------|------|-----------|
| `SKILL.md` | skill 定义 + 完整执行流程 | 改流程、改 prompt、改审查逻辑 |
| `references/review-lenses.md` | 三大审查视角定义（Skeptic/Strategist/Editor） | 调整审查标准 |
| `CLAUDE.md` | 本文件，AI agent 快速上下文 | 架构变了才改 |

## 关键设计决策

1. **复用 a2a 的 preflight**：环境检测逻辑相同（检查 Claude CLI + Codex CLI），不重复造轮子。
2. **三个文档审查视角**：Skeptic（质疑数据/逻辑）、Strategist（审结构/战略）、Editor（删冗余）。
3. **不依赖 git diff**：审查对象是文件内容或粘贴文本，不需要 git 仓库。
4. **生成+审查模式**：可以先让 Claude 生成内容，再交给 Codex 审查。
5. **Reviewer 隔离**：同 a2a，每个 reviewer 独立执行，互不可见。
6. **只审不改**：skill 只产出 verdict report，修改由用户决定。

## 依赖

- a2a skill 的 `scripts/preflight.sh`（环境检测）
- 如果 a2a 不存在，自动降级为 single-model-multi-lens 模式
