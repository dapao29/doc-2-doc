# d2d — Doc-to-Doc 跨模型对抗式文档审查

> 让两个 AI 互相找茬，比自己审自己靠谱 10 倍。

<p align="center">
  <img src="https://img.shields.io/badge/Claude_Code-Skill-blueviolet?style=for-the-badge" alt="Claude Code Skill"/>
  <img src="https://img.shields.io/badge/Cross--Model-Adversarial_Review-orange?style=for-the-badge" alt="Cross-Model"/>
  <img src="https://img.shields.io/badge/version-1.0.0-green?style=for-the-badge" alt="Version"/>
</p>

---

## 这是什么？

**d2d** 是一个 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill，实现**跨模型对抗式文档审查**。

灵感来自 [a2a (Agent-to-Agent)](https://github.com/CtriXin/agent-2-agent) 的代码审查机制——**谁写的就让对手来审**。d2d 将这一理念扩展到所有文本内容：

- 📊 **研究报告** — 数据来源可靠吗？结论站得住脚吗？
- 📈 **数据分析** — 统计方法对吗？有没有 cherry-picking？
- 💼 **商业计划** — 市场假设合理吗？财务预测经得起推敲吗？
- 🏗️ **技术方案** — 架构选型有道理吗？有没有过度设计？
- 📝 **学术论文** — 实验设计严谨吗？论证链完整吗？

## 核心理念

**Self-review is self-deception.**

不同 AI 模型有不同的 blind spot：
- **Claude** 倾向堆砌论据、过度修饰、"正确的废话"
- **Codex** 倾向跳过战略分析、忽略受众适配、结论过于直白

让它们互审，漏洞无处藏身。

## 三大审查视角

| 视角 | 角色 | 问的核心问题 |
|------|------|-------------|
| **The Skeptic 质疑者** | 数据和逻辑警察 | "你的结论听起来很漂亮，但数据真的支持吗？" |
| **The Strategist 战略家** | 结构和战略顾问 | "你回答的是正确的问题吗？还是你以为的问题？" |
| **The Editor 精简者** | 冗余内容杀手 | "如果这一段消失了，读者会注意到吗？" |

## 运行模式

| 模式 | 条件 | 质量 |
|------|------|------|
| **adversarial** | Claude + Codex 双向可用 | ⭐⭐⭐ 最高 |
| **single-model-multi-lens** | 仅一侧可用 | ⭐⭐ 多视角覆盖 |

## 快速开始

### 1. 安装

```bash
git clone https://github.com/CtriXin/doc-2-doc.git ~/.claude/skills/d2d
```

### 2. 依赖（可选，用于跨模型审查）

```bash
# Claude CLI（如果你用 VS Code 扩展，可能已经有了）
npm install -g @anthropic-ai/claude-code

# Codex CLI（OpenAI）
npm install -g @anthropic-ai/claude-code  # Claude
npm install -g codex                       # Codex
```

### 3. 使用

在 Claude Code 对话中：

```
/d2d path/to/report.md          # 审查指定文件
/d2d                             # 审查粘贴的文本
帮我写一份XX报告然后用 d2d 审查    # 生成 + 审查模式
```

## 审查报告示例

```
## d2d Review — RPT-20240315

文档: 2024年AI行业研究报告
类型: 研究报告
规模: Heavy
Reviewers: Skeptic + Strategist + Editor
Mode: adversarial

### Verdict: CONTESTED

### Findings

| # | Sev | Lens | 问题 | 裁定 |
|---|-----|------|------|------|
| 1 | 🔴 | Sk | "市场规模达500亿" — 引用2019年数据，已严重过时 | Accept |
| 2 | 🔴 | Sk | "用户增长率200%" — 分母为试运营期数据，有误导性 | Accept |
| 3 | 🟡 | St | 竞品分析仅覆盖3家，遗漏市占率第二的Y公司 | Accept |
| 4 | 🟡 | St | 结论"建议加大投入"过于笼统，缺乏具体路径 | Flag |
| 5 | 🟢 | Ed | 行业背景占全文38%，核心分析被稀释 | Accept |
| 6 | 🟢 | Ed | "综上所述"段落重复前文结论，无新增信息 | Accept |
```

## Verdict 标准

| Verdict | 条件 |
|---------|------|
| **PASS** | 无 🔴 Critical finding |
| **CONTESTED** | 有 🔴 但 reviewer 存在分歧 |
| **REJECT** | 多个 reviewer 共识性 🔴 finding |

## 文件结构

```
d2d/
├── SKILL.md                  # 核心：skill 定义 + 完整执行流程
├── CLAUDE.md                 # AI agent 快速上下文
├── README.md                 # 本文件
└── references/
    └── review-lenses.md      # 三大审查视角详细定义
```

## 与 a2a 的关系

d2d 是 [a2a](https://github.com/CtriXin/agent-2-agent) 的姊妹项目：

| | a2a | d2d |
|---|---|---|
| 审查对象 | 代码（git diff） | 任意文档 |
| 审查视角 | Challenger / Architect / Subtractor | Skeptic / Strategist / Editor |
| 环境检测 | 自带 preflight.sh | 复用 a2a 的 preflight |
| 核心机制 | 跨模型对抗 | 跨模型对抗 |

## License

MIT
