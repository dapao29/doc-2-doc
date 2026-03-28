---
name: d2d
version: "1.0.0"
description: "d2d (Doc-to-Doc) — 跨模型对抗式文档审查。研究报告、数据分析、商业计划、技术方案——让两个 AI 互相找茬，比自己审自己靠谱得多。"
argument-hint: '要审查的文档路径或主题描述'
allowed-tools: Read, Write, Bash, Grep, Glob, Agent, AskUserQuestion
---

# d2d — 对抗式文档审查

> 写文档的模型不审文档。Self-review is self-deception.

## 核心机制

**对抗原则**：review 优先在**对立模型**上执行。Claude 产出的文档由 Codex 审查，Codex 产出的由 Claude 审查。

**为什么有效**：不同模型有不同的 blind spot。Claude 倾向堆砌论据、过度修饰；Codex 倾向跳过战略分析、忽略受众适配。让它们互相找茬，漏洞无处藏身。

**适用内容类型**：
- 研究报告 / 行业分析
- 数据分析报告
- 技术方案 / 设计文档
- 商业计划 / 融资BP
- 竞品分析
- 市场调研
- 学术论文 / 综述
- 任何需要严谨审查的文本内容

### 运行模式

| 模式 | 条件 | 说明 |
|------|------|------|
| **adversarial** | Claude + Codex 双向均可用 | 跨模型对抗审查，最高质量 |
| **single-model-multi-lens** | 仅一侧可用 | 同模型多视角审查，质量降低但仍有多视角覆盖 |
| **blocked** | 无法执行 | 无法执行 review，中止 |

## 触发方式

- 用户说 "review 文档"、"审查报告"、"帮我审一下这个分析"
- 用户说 "d2d"、"文档审查"、"对抗审查"
- 手动指定：`/d2d docs/report.md`
- 传入文本内容：`/d2d` 后粘贴或描述要审查的内容

## 审查规模判定

按文档字数（中文按字符数，英文按单词数）判定，决定派几个 reviewer。

| 规模 | 条件 | Reviewer 配置 | 预估耗时 |
|------|------|--------------|---------|
| **Light** | < 1000 字 | Skeptic only | ~30s |
| **Medium** | 1000–5000 字 | Skeptic + Strategist | ~1min |
| **Heavy** | 5000–15000 字 | 三人全上 | ~2min |
| **Heavy+** | 15000+ 字 | 三人全上 + 分段审查 | ~3min |

## 三大审查视角

详见 `references/review-lenses.md`。

- **The Skeptic — 质疑者**：质疑数据来源、逻辑推理、因果关系、结论可靠性
- **The Strategist — 战略家**：审视整体结构、论证完整性、受众适配、建议可行性
- **The Editor — 精简者**：找冗余内容、正确的废话、不必要的堆砌、可压缩的段落

## 前置校验（Gate）

d2d 的审查对象是**文档内容**，不依赖 git diff。入口判定：

1. **文件模式**：用户指定了文件路径 → 读取文件内容进行审查
2. **文本模式**：用户在对话中直接提供/粘贴了文本 → 审查该文本
3. **生成+审查模式**：用户请求先生成内容再审查 → Claude 先产出内容，然后交给对立模型审查
4. **无审查对象**：未提供任何内容 → 提示：`"请提供要审查的文档（文件路径、粘贴文本、或告诉我要生成什么内容）"`

## 执行流程

### Step 1: 环境检测 + 确定审查对象

> 🔍 运行 preflight（复用 a2a 的环境检测），确认模式，确定审查内容。

```bash
~/.claude/skills/a2a/scripts/preflight.sh --json
```

如果 a2a 的 preflight 不存在，降级为 single-model-multi-lens 模式（仅用 Agent tool）。

确定审查对象：
- 用户指定了文件 → `Read` 读取内容
- 用户粘贴了文本 → 直接使用
- 用户要求生成 → 先产出内容，记录为 Claude 产出（后续交 Codex 审）

### Step 2: 加载审查基准

> 📋 读取文档审查视角定义

1. `references/review-lenses.md` — 三大视角定义（Skeptic / Strategist / Editor）
2. 识别文档类型（研究报告 / 数据分析 / 商业计划 / 技术方案 / 通用文档）
3. 根据文档类型调整视角权重：

| 文档类型 | Skeptic 重点 | Strategist 重点 | Editor 重点 |
|---------|-------------|----------------|-------------|
| 研究报告 | 数据可靠性、方法论 | 研究框架、结论完整性 | 文献堆砌、重复论述 |
| 数据分析 | 数据质量、统计方法 | 分析框架、洞察深度 | 图表冗余、废话 |
| 商业计划 | 市场数据、财务预测 | 商业模式、竞争策略 | 空话套话、过度包装 |
| 技术方案 | 可行性、性能假设 | 架构合理性、扩展性 | 过度设计、不必要的复杂度 |
| 学术论文 | 实验设计、数据分析 | 理论框架、贡献度 | 行文冗余、重复引用 |

### Step 3: 明确审查意图

**必须在 review 前声明**：

```
Document Type: {文档类型}
Purpose:       {这份文档要解决什么问题 / 给谁看}
Key Question:  {审查时最关注什么}
```

如果用户没有说明，Claude 根据内容自动推断并确认。

### Step 4: 派遣 Reviewer（并行）

> 🤖 跨模型对抗审查，机制与 a2a 相同

**Step 4a: 判定内容作者**

1. 用户明确告知（"这是我自己写的"、"刚让 Claude 生成的"、"Codex 输出的"）
2. 当前对话上下文推断（如果是本轮对话中 Claude 生成的 → Claude 产出）
3. 默认：假定为 Claude 产出

**Step 4b: 按作者 + 环境选择 dispatch 通道**

#### 路径 A: Claude 产出 → Codex 审查

```bash
codex exec --full-auto -- "你是一位专业的文档审查专家。按照以下审查视角严格审查这份文档..."
```

#### 路径 B: Codex/用户产出 → Claude 审查

通过 **Agent tool** spawn 隔离 sub-agent 执行 review：

```
使用 Agent tool，为每个 lens 分别 spawn 一个独立 agent：
- agent 1: Skeptic lens + 完整文档内容
- agent 2: Strategist lens + 文档结构摘要 + 核心论点
- agent 3: Editor lens + 完整文档内容
```

#### 路径 C: 降级模式（单模型多视角）

Codex 不可用时，所有 reviewer 通过 Agent tool spawn 独立 sub-agent，标注 `mode: single-model-multi-lens`。

**每个 reviewer 收到的 review packet**：

**通用字段**（所有 reviewer 共享）：
- **文档类型**：研究报告 / 数据分析 / 商业计划 / 技术方案 / 通用
- **审查意图**：1-2 句话概括本次审查关注点
- **审查视角定义**：只给自己那个 lens

**Lens 专属内容裁剪**：

| Lens | 收到的内容 |
|------|----------|
| **Skeptic** | 完整文档 + 数据/图表/引用清单（需要逐条验证） |
| **Strategist** | 文档大纲 + 核心论点摘要 + 结论/建议部分全文（审结构不需要逐段读） |
| **Editor** | 完整文档（需要逐段判断是否冗余） |

**Reviewer 输出限制**（附加到每个 reviewer 的 prompt 末尾）：
```
输出限制：
- 每条 finding ≤ 3 行（issue + impact + fix）
- 总 findings ≤ 10 条
- 无 finding 时只输出 "LGTM"，不要写分析过程
- 必须引用文档中的具体段落/句子，不接受泛泛而谈
```

所有 reviewer **并行执行**，互不可见对方结果。

### Step 5: 汇总 Findings

> 📊 收集所有 reviewer 发现，统一分级裁定

**汇总规则**：保留 reviewer provenance，按 severity 分级

| Severity | 定义 | 示例 |
|----------|------|------|
| **🔴 Critical** | 核心结论不可靠或严重误导 | 数据来源造假、逻辑链断裂、关键假设错误 |
| **🟡 Major** | 不影响核心结论但削弱说服力 | 样本偏差、论证跳跃、建议不可行、受众错配 |
| **🟢 Minor** | 可改进但不影响文档价值 | 措辞冗余、段落可压缩、图表可精简 |

主审裁定每条 finding：
- **Accept** — 确实是问题，需要修改
- **Dismiss** — reviewer 过度解读或不适用当前语境
- **Flag** — 有道理但 non-blocking，记录供作者参考

### Step 6: 输出 Verdict Report

**固定 5 列**：`#`, `Sev`, `Lens`, `问题`, `裁定`

| 字段 | 规则 |
|------|------|
| **Sev** | 只写 emoji：🔴🟡🟢，不加文字 |
| **Lens** | 缩写：`Sk` = Skeptic, `St` = Strategist, `Ed` = Editor；多人 `Sk+St` |
| **问题** | 引用原文关键句，后接问题描述，**不省略内容** |
| **裁定** | 自适应宽窄模式（同 a2a 规则） |

**宽模式**（≤ 5 条 findings）：
- 裁定列写完整：`Accept — 理由`、`Dismiss — 原因`、`Flag — 建议`

**窄模式**（≥ 6 条 findings）：
- 裁定列只写结果，原因移到表格下方 **裁定说明** 区

```markdown
## d2d Review — {DOC_ID}

**文档**: {标题或文件名}
**类型**: {研究报告 / 数据分析 / 商业计划 / 技术方案 / 通用}
**规模**: {Light / Medium / Heavy / Heavy+}
**Reviewers**: Skeptic + Strategist + Editor
**Mode**: {adversarial / single-model-multi-lens}

### Verdict: {PASS / CONTESTED / REJECT}

### Findings

| # | Sev | Lens | 问题 | 裁定 |
|---|-----|------|------|------|
| 1 | 🔴 | Sk | "市场规模达到500亿" — 引用来源为2019年数据，已过时3年 | Accept — 需更新数据源 |
| 2 | 🟡 | St | 竞品分析只覆盖3家，遗漏了市占率第二的Y公司 | Accept — 补充Y公司分析 |
| 3 | 🟢 | Ed | 第二章背景介绍占全文35%，核心论点被稀释 | Flag |

**裁定说明**
- #3 Flag: 背景可压缩至20%以内，但不影响核心结论

### 总结
{一段话总结 review 结论和改进建议}

### 修改优先级
1. {最重要的修改}
2. {次重要的修改}
3. ...
```

## Verdict 标准

| Verdict | 条件 |
|---------|------|
| **PASS** | 无 🔴 Critical finding |
| **CONTESTED** | 有 🔴 但 reviewer 之间存在分歧 |
| **REJECT** | 多个 reviewer 共识性 🔴 finding |

## 生成+审查模式

当用户请求 "写一份XXX报告然后审查"：

1. Claude 根据用户需求生成文档内容
2. 自动标记为 Claude 产出
3. 进入审查流程（优先派给 Codex）
4. 输出 verdict report
5. 如果 verdict 非 PASS，提供修改建议并**询问用户是否要根据审查意见修改**

**不自动修改**——审查和修改是两个独立动作，由用户决定。

## 自定义审查重点

用户可以指定额外的审查重点，作为补充 checklist 传给所有 reviewer：

```
/d2d report.md --focus "数据来源可靠性" "结论是否可行" "是否适合给投资人看"
```

或在对话中说明：
- "重点看看数据有没有问题"
- "主要审查逻辑是否自洽"
- "从投资人角度审一下这个BP"

自定义重点会附加到对应 lens 的 checklist 末尾，不替换默认 checklist。

## Dispatch 通道总结

| 发起环境 | 内容作者 | Reviewer | Dispatch 通道 | 模式 |
|---------|---------|----------|-------------|------|
| Claude Code | Claude | Codex | `codex exec` | adversarial |
| Claude Code | Codex/用户 | Claude | Agent tool (sub-agent) | adversarial |
| 任意 | 任意 | 同模型 | Agent tool | single-model-multi-lens |

## 注意事项

- 本 skill 只产出 **verdict report**，不自动修改文档
- Reviewer 不能看到彼此的输出（防止从众效应）
- 每条 finding 必须引用**文档中的具体段落或数据**，不接受泛泛而谈
- 审查是为了提升质量，不是否定作者——语气专业、建设性
- 对于超长文档（15000+字），按章节分段审查，最终合并 findings
