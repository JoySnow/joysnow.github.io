---
layout: post
title: "Harness Engineering（二）：SDD/TDD 方法论映射与产品格局"
categories:
 - LLM
tags:
 - Harness
 - Agent
 - AI
 - Spec_Coding
 - SDD
---

> **前置阅读**：[Harness Engineering（一）：六层确定性框架](harness-engineering.md) — 定义、六层栈、递归结构、控制力分布。
>
> **摘要**：本文承接第一页的框架，展开两个主题：
>  -（A）传统软件开发方法论（TDD、SDD 等）如何在 Harness 各层中被重新诠释和编译为可执行约束；
>  -（B）2026 年 Harness Engineering 的产品格局——三大 SDD 流派、桥接工具、AI 代码审查生态。
> 以 **Superpowers** 和 **spec-kit** 的细节实现与对比为具体入口，理解 Harness Engineering 如何在实践中落地。

<!--more-->

> 基于 2026-06-29 的在Claude Code Agent中的对话讨论, 由Deepseek-v4-pro整理。

# Harness Engineering（二）：SDD/TDD 方法论映射与产品格局

## 一、软件开发方法论谱系

### 1.1 三代方法论

#### 第一代：重量级（计划驱动）

| 方法论 | 核心思想 | 确定性来源 |
|--------|---------|-----------|
| Waterfall | 需求→设计→实现→测试→部署，线性不可逆 | 阶段门控，上游冻结 |
| V-Model | 每个开发阶段对应一个测试阶段 | 验证与实现的对称性 |
| SSADM | 结构化系统分析与设计 | 数据流图、实体关系图的形式化 |

#### 第二代：轻量级（反馈驱动）

| 方法论 | 核心思想 | 确定性来源 |
|--------|---------|-----------|
| **TDD** (Test-Driven Development) | 先写失败测试 → 写代码使其通过 → 重构 | 测试即规格，红色→绿色循环 |
| **BDD** (Behavior-Driven Development) | 用自然语言描述行为，可执行规格 | Given-When-Then 场景 |
| **SDD** (Spec-Driven Development) | 先写完整规格再写代码 | 规格文档是真理的单一来源 |
| **DDD** (Domain-Driven Design) | 用领域模型驱动设计 | 统一语言、限界上下文、聚合根 |
| **CI/CD** | 持续集成 + 持续交付 | 自动化流水线即契约 |
| **Agile / Scrum / Kanban** | 迭代增量，小批量反馈 | 时间盒 + 回顾 + 可视化 |

#### 第三代：设计约束驱动

| 方法论 | 核心思想 | 确定性来源 |
|--------|---------|-----------|
| Type-Driven Development | 先定义类型再实现 | 类型系统在编译期证明 |
| Design by Contract | 前置条件/后置条件/不变量 | 合约断言 |
| Property-Based Testing | 生成随机输入验证不变量 | 数学性质的形式化 |
| Model Checking (TLA+/Alloy) | 形式化验证状态空间 | 穷举模型检查 |

---

### 1.2 方法论在 Harness 六层中的递归重演

> 引用自 [harness-engineering.md](harness-engineering.md) 的六层框架。

```
传统软件工程           →    Harness Engineering 对应层
────────────────────────────────────────────────────
SDD (先规格后实现)     →    Layer 0: Structured Output / Tool Use schema
                            Layer 1: Workflow DSL 的 Schema 校验
                            Layer 2: CLAUDE.md 约束 + Plan Mode 先出方案

TDD (测试先行)        →    Layer 2: "你写测试定义对，Agent 写实现探索怎么做"
                            Layer 0: System Prompt 中的行为约束

Type-Driven Dev       →    Layer 2: TypeScript/Rust 编译期兜底 + Pre-commit hooks
                            Layer 0: JSON Schema for StructuredOutput

BDD (行为驱动)        →    Layer 2: 测试合约 = Given-When-Then 场景
                            Layer 4: 团队级 Agent 行为规范

Design by Contract    →    Layer 0: Tool Use schema 定义输入输出合约
                            Layer 1: Pipeline/Parallel 控制流 DSL 的确定性语义

CI/CD                 →    Layer 2: lint + format + type-check + test → commit
                            Layer 4: CI 流水线统一验证

DDD (统一语言)        →    Layer 1: Memory 系统的 wikilink + frontmatter 分类

Agile (小批量迭代)    →    Layer 2: "多轮小批量，每轮可验证可回滚"
                            Layer 1: Pipeline 优先，避免 Barrier

Model Checking        →    Layer 1: Adversarial Verify (多 skeptic 证伪)
```

---

## 二、从「方法论文档」到「可执行约束」

传统方法论是**给人看的文档和规程**——人需要主动遵守。Harness Engineering 把它变成**可执行、可自动化的约束**：

| 传统形态 | Harness 形态 (以 Superpowers 为例) |
|---------|------------|
| TDD：开发者自己记得先写测试 | `test-driven-development` skill **硬拦截**，不写测试不能写实现 [^1] |
| SDD：开会对齐后各自写 | `writing-plans` + `brainstorming` **强制**先出计划，用户确认后才动手 [^1] |
| Code Review：PR 时审查 | `requesting-code-review` + `receiving-code-review` **系统化**触发，多维度证伪 [^1] |
| CI：提交时跑流水线 | `verification-before-completion` **阻断**声称，必须贴验证输出 [^1] |
| Agile 回顾：sprint 末复盘 | Layer 4 的**事后复盘 + Meta 持续改进**，制度化 |

**核心转变：从 trusting humans to follow process → building the process into the harness.**

---

### 2.1 Superpowers 技能：方法论 → Agent 操作系统的编译

Superpowers 插件（obra/Jesse Vincent, ~167k stars）不是新方法论，而是**已有方法论在 AI Agent 时代的「编译产物」**[^2]：

```
方法论层 (WHAT)          Skill 层 (HOW — 可执行)
─────────────────────    ──────────────────────────────────
TDD + SDD                brainstorming → writing-plans → executing-plans
                          + test-driven-development

Agile 小批量              subagent-driven-development
                          + dispatching-parallel-agents

CI + Code Review         requesting-code-review
                          + receiving-code-review
                          + verification-before-completion

Design by Contract       using-git-worktrees (隔离即合约)

Agile 收尾               finishing-a-development-branch
```

技能是「硬编码的纪律」而非「建议性的最佳实践」——当 `using-superpowers` 说「即使 1% 概率匹配也必须调用技能」，这本质上是把 SDD/TDD 的哲学从「人做决策」推到了「系统做约束」[^1]。

**跨 Agent 支持**（用户重要纠正）：Superpowers 有**平台适配层**，支持 Claude Code、Codex、Copilot CLI、Gemini CLI、Pi、Antigravity 等多个 Agent 产品 [^1]。每个技能用通用动作语言编写（"dispatch a subagent"、"create a todo"），运行时由对应平台的适配层翻译成具体工具调用。这与 spec-kit 的 slash command 注入策略不同——spec-kit 是「寄生」在 Agent 已有的命令系统上，Superpowers 既有适配也有增强（如 Claude Code 的 Workflow DSL）。

---

## 三、TDD 方法论深度解析

### 3.1 经典 TDD（Kent Beck, 2002）：Red-Green-Refactor

```
RED ──→ GREEN ──→ REFACTOR ──→ RED ──→ ...
```

| 阶段 | 做什么 | 铁律 |
|------|--------|------|
| **RED** | 写一个最小测试，描述一段尚未实现的行为 | 测试必须失败——如果直接通过，你不知道它测的是正确的东西还是什么都不测 |
| **GREEN** | 写最少代码让测试通过 | 不添加测试没要求的功能、不重构、不「顺手改」 |
| **REFACTOR** | 消除重复、改善命名、提取抽象 | 测试必须保持全绿；不添加新行为 |

核心反直觉要点：
1. **测试先行不是「测试重要」，而是「定义什么是正确」** — 测试是可执行规格，不是在代码之后写的验证
2. **看到测试失败 = 获得信息** — 测试直接通过说明要么测了已有行为，要么测试本身没测任何有意义的东西
3. **最小代码 = 最小猜测** — 多写的代码是基于假设的「以后可能需要」，而这些假设往往是错的
4. **Tests-after 回答 "What does this do?"，Tests-first 回答 "What should this do?"** — 本质不同

### 3.2 TDD Skill：方法论 → 可执行约束的编译

Superpowers 的 `test-driven-development` skill [^1] 做了四件事，把 Kent Beck 的方法论编译成了 Agent 无法绕过的约束：

**1. Iron Law — 从建议到铁律**
```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
Write code before the test? Delete it. Start over.
```

**2. Red Flags — 对抗认知偏差**

| 你脑子里想的 | 技能的红牌 |
|-------------|-----------|
| 「这个函数太简单了不需要测试」 | Simple code breaks. Test takes 30 seconds. |
| 「我写完再补测试」 | Tests passing immediately prove nothing. |
| 「我都手动测过了」 | Ad-hoc ≠ systematic. No record, can't re-run. |
| 「删了这几个小时的代码太浪费」 | 沉没成本谬误。Keeping unverified code is technical debt. |
| 「留作参考，我先写测试」 | 你会忍不住 adapt 它，那就不是 test-first。Delete means delete. |
| 「Tests after achieve the same purpose」 | Tests-after = "what does this do?" Tests-first = "what should this do?" |

**3. 测试质量的内置规范** — 区分好测试（测真实行为）和坏测试（测 mock 行为），给出具体代码示例

**4. 自我执行的验证检查表**
```
Can't check all boxes? You skipped TDD. Start over.
```

| 维度 | 经典 TDD 方法论 | Superpowers TDD Skill |
|------|---------------|-----------------|
| **形态** | 书籍、博客、培训课程 | System prompt 注入，每个会话都强制执行 |
| **约束力** | 建议性（人可以跳过） | 硬编码（技能要求删代码重来） |
| **失败处理** | 没有机制 | Red flags 列表 + "delete and start over" |
| **质量门控** | 看覆盖率数字 | 看测试是否测了**真实行为**而非 mock |
| **认知偏差对抗** | 没有 | 逐条列了常见合理化借口和反驳 |
| **验证要求** | 可选 | 强制：必须亲眼看到测试失败，且失败的**原因是对**的 |

---

## 四、SDD 的三种实现形态

经过对比 spec-kit、Superpowers、OpenSpec 三个主要 SDD 产品后，可以识别出 SDD 在 Harness Engineering 中的三种实现哲学 [^3][^4][^5]：

```
制品型 SDD（spec-kit, GitHub, ~116k stars）
    「写出对的规格」
    精确的结构化中间制品 → LLM 看到精确的上下文 → 减少幻觉空间
    工作流：constitution → specify → clarify → plan → tasks → implement → analyze

过程型 SDD（Superpowers, obra, ~167k stars）
    「走对的流程」
    硬拦截的过程节点 → 人类在关键点确认 → 减少错误累积
    工作流：brainstorming → writing-plans → TDD → executing → verification → review

变更型 SDD（OpenSpec, Fission-AI, ~48k stars）
    「管好规格的生命周期」
    propose → review → apply → archive → 旧变更不污染新上下文
    4 个标准工件：proposal.md / specs/ / design.md / tasks.md
```

| | 制品型（spec-kit） | 过程型（Superpowers） | 变更型（OpenSpec） |
|---|---|---|---|
| **确定性来源** | 结构化的中间制品 | 硬拦截的过程节点 | 规格的全生命周期可追溯 |
| **核心约束** | `.specify/` 模板 + constitution.md | Iron Law + 技能硬门控 | 4 个标准工件 + archive 机制 |
| **跨 Agent** | ✅ 30+ (slash cmd 注入) | ✅ 6+ (平台适配器) | ✅ 25+ (slash cmd 注入) |
| **强项** | 绿场 0→1、企业治理 | 质量执念、长时自主会话 | 棕场迭代、跨工具团队 |
| **弱项** | LLM 一次性生成仍可能偏离 | 不规定规格格式 | 规格粒度偏粗 |
| **人类 gate 数** | 2（constitution + spec 审阅） | 5+（每 section + plan + task 间 + 验证 + 审查） | 2（propose + review） |
| **归档机制** | ❌ 无 | ❌ 无 | ✅ archive — 旧变更不污染新上下文 |
| **特点** | contracts/api-spec.json 机器可读合约 | 每个 plan task 含完整代码 | 最轻量，适合迭代式项目 |

---

## 五、spec-kit vs Superpowers：Spec 和 Plan 的深度对比

### 5.1 Spec（规格定义）[^1][^3]

| 维度 | spec-kit | Superpowers |
|------|----------|-------------|
| **产物位置** | `specs/<feature>/spec.md` | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` |
| **格式** | 模板驱动：user stories + functional requirements + review checklist | 自由格式：architecture → components → data flow → error handling → testing |
| **构建方式** | 一次性生成，LLM 填写模板 | 交互式，逐 section 用户确认后才能继续 |
| **粒度** | 按用户故事组织 | 按架构关注面组织 |
| **验收标准** | 检查表（checkbox） | 用户逐段口头确认 |
| **人类 gate** | spec 写完后审阅（1 个 gate） | 每个 section + 整个文件（多个 gate） |

### 5.2 Plan（实施计划）[^1][^3]

| 维度 | spec-kit | Superpowers |
|------|----------|-------------|
| **产物数量** | **6+ 个文件**：plan.md, data-model.md, research.md, quickstart.md, contracts/api-spec.json, contracts/signalr-spec.md | **1 个文件**（但包含完整可执行代码） |
| **代码在 plan 里？** | ❌ 没有 — 只说「创建 `src/models/user.ts`」但不给代码 | ✅ 有 — 每个 step 写完整实现代码 |
| **接口合约** | 独立文件 `contracts/api-spec.json`（形式化 JSON，**机器可读**） | Consumes/Produces 在每个 task 头部（prose，**给人读**） |
| **任务粒度** | User Story Phase（粗）— 如 `T001: 创建 User 模型` | TDD 循环步骤（细）— Step 1: 写测试 → 确认失败 → 写代码 → 确认通过 → Commit |
| **每步耗时** | 未规定 | **2-5 分钟** |
| **自审机制** | `/speckit.analyze`（跨制品一致性检查） | Plan self-review checklist（spec 覆盖率 + 占位符扫描 + 类型一致性） |
| **禁止占位符** | 未明确禁止 | **严格禁止** — 不允许 TBD、TODO、"add error handling"、"similar to Task N" |
| **TDD 内置** | 未强制（tasks.md 有 test→impl 顺序但不强制 red-green-refactor） | **强制内置** — 每个 task 就是 red-green-refactor 循环 |

### 5.3 信任模型的根本分歧

```
spec-kit：
  信任结构 → 只要模板够精确、制品够结构化，Agent 就能自己走完
  人类参与：开始定义 constitution + spec 审阅 → 后续 LLM 自动执行

Superpowers：
  信任人类判断 → 关键的确认点必须有人的介入，结构是辅助不是替代
  人类参与：全程在场 — 每个 section 确认 → plan 审阅 → task 间 review → 验证 → 审查
```

### 5.4 spec-kit tasks.md 的可度量性分析 [^3]

spec-kit 的任务列表有五个可度量机制，但粒度止于「文件是否存在 + 测试是否通过」：

```
可度量的部分：
  ✅ 文件是否被创建（tasks.md 指定具体路径）
  ✅ 测试是否全绿（TDD 顺序：测试在实现之前）
  ✅ 编译是否通过（依赖排序：model → service → endpoint）
  ✅ 并行度是否合理（[P] 标记）
  ✅ User Story checkpoint 是否通过

不可度量（仍需人工判断）的部分：
  ❌ 实现是否正确理解了需求意图
  ❌ 代码质量是否过关
  ❌ 是否过度工程
  ❌ API 设计是否真的好用
```

这也是为什么 spec-kit 有 `/speckit.analyze`（跨制品一致性检查）和 `/speckit.converge`（对照代码库评估规格）——它知道 LLM 生成的计划到实现之间有 gap，需要额外的验证层。

---

## 六、Harness Engineering 产品格局

### 6.1 按 Harness 层分布 [^1][^3][^4][^5][^6]

```
Layer 4: 组织治理
├── CodeRabbit（团队审查策略、学习规则）
├── Graphite Agent（stacked-PR 质量门控）
└── CI/CD 平台的 AI 策略配置

Layer 3: AI 产物适配
├── CodeRabbit（AI 特征 bug 检测，600万+ 仓库，~44% bug 捕获率）
├── Greptile（全仓库上下文交叉检查，~82% 捕获率）
├── Qodo Merge（测试生成 + 审查一体，~85% 捕获率）
└── Semgrep / Snyk（传统安全扫描，AI 代码上下文不足）

Layer 2: 开发者 Harness — SDD / Workflow 工具
├── ★ Superpowers     — 过程型 SDD（技能硬拦截）
├── ★ spec-kit        — 制品型 SDD（模板 + 结构化）
├── ★ OpenSpec        — 变更型 SDD（propose → apply → archive）
├── Forge             — 跨 20+ Agent 便携 SDD harness（任务运行器抽象层）
├── Trellis           — 16 平台，渐进式 spec + task + workspace memory
├── OSpec             — SDD + Loop Engineering（plan → act → verify，安全等级 L1/L2/L3）
├── Kiro              — 轻量 SDD，强调速度和自动化
├── BMAD              — 模拟企业团队角色（architect + developer + tester）
├── GSD               — Goal-Structured，上下文隔离的并行波
├── SpecForge         — 8 阶段 DAG + Iron Law 门控 + 三巨头融合
└── cc-sdd / ai-sdd   — 多 Agent × 多语言 SDD

Layer 1: Agent 产品（SDD 工具不修改这一层，而是寄生/适配于它）
├── Claude Code, Cursor, Copilot, Codex, Gemini CLI
├── Windsurf, Aider, Cline, Roo Code
└── Amazon Q Developer

Layer 0: Model API
├── Anthropic API, OpenAI API, Google AI
└── Together, Groq, Fireworks
```

### 6.2 Layer 3 细节：AI 代码审查产品 [^6]

| 工具 | 定位 | 特点 | Bug 捕获率 |
|------|------|------|-----------|
| **CodeRabbit** | 市场龙头 | 600万+ 仓库，行级评论、PR 内对话、学习用户偏好 | ~44% |
| **Graphite Agent** | stacked-PR 专用 | 低噪音（每 run 仅 2 误报）、82% 修复率 | ~6%（追求高信号） |
| **Greptile** | 大仓/全仓库上下文 | 跨文件引用影响分析 | ~82% |
| **Qodo Merge** | 测试生成 + 审查一体 | 自动补测试 | ~85% |
| **GitHub Copilot Reviews** | 零配置原生集成 | 深度最浅，随 Copilot 订阅赠送 | 最弱 |

**2026 年核心问题：「AI 审 AI 代码」的循环困境** [^6]

- **87% 的 AI 生成 PR 引入安全漏洞**（DryRun Security / CSA 2026 年 4 月）
- AI reviewer 和 AI code generator 共享训练数据 → 学的是同一套反模式
- Claude 写的代码被 Claude-powered reviewer 批准，因为模式看起来「正常」
- **Layer 3 不能用同一个模型审同一个模型的产出**

推荐的三层审查模式：
1. **AI first pass** — 机械问题（风格、明显 bug、缺失测试）
2. **人类 senior review（不可跳过）** — 架构、业务逻辑、并发
3. **CI gates** — linter + type checker + test suite + security scanner

---

## 七、桥接工具与生态趋势

### 7.1 桥接/组合工具 [^4]

一个正在出现的趋势：**把多个 SDD 工具组合使用**，各取所长：

| 工具 | 做什么 |
|------|--------|
| **speckit-superpowers-bridge** | spec-kit 管 WHAT（规格→计划→任务）→ Superpowers 管 HOW（TDD/审查/验证），5 个硬编码 guard rules + 6 个 state scripts + 1 个 handoff JSON |
| **spec-driven-tdd** | OpenSpec 管规划 + Superpowers 管 TDD + simplify + review |
| **SpecForge** | 融合 OpenSpec + Superpowers + spec-kit 到一个 CLI，Iron Law 门控 + 8 阶段 DAG（foundation → requirements → design → planning → implementation → quality → release → evolution） |
| **spec-kit-clean-architecture** | spec-kit + YAML 确定化执行 + RLHF 评分（每步 -2 ~ +2）+ 可恢复状态（步骤 N 失败，N-1 步已 git-committed） |
| **Comet** | OpenSpec 变更模型 + Superpowers 执行引擎 |

核心洞察：**spec-kit 管 WHAT，Superpowers 管 HOW，OpenSpec 管 LIFECYCLE，桥接层让它们互操作。**

### 7.2 五个生态趋势 [^4][^6]

1. **三巨头不互斥，趋向融合** — SpecForge、speckit-superpowers-bridge 等桥接工具证明「spec-kit 管 WHAT + Superpowers 管 HOW + OpenSpec 管 LIFECYCLE」的互补架构可行

2. **Layer 2 的「插件化」** — 新一代工具（Forge、Trellis、OSpec）不构建新 Agent，而是生成 Agent 可消费的约束文件（AGENTS.md、`.specify/`、slash commands），依赖现有 Layer 1 Agent 执行

3. **Harness 本身的元层（Meta）** — SpecForge 的 8 阶段 DAG、spec-kit-clean-architecture 的 YAML 状态持久化、OpenSpec 的 archive 机制 — 这些都是在做「harness 的 harness」，即第一页框架中 Meta 层的工作

4. **Layer 3 的「AI 审 AI」困境** — 87% 的 AI 生成 PR 引入安全漏洞。跨模型审查 + 人类 gate 不可跳过

5. **收敛方向明确** — 整个生态正在向 **SDD 工作流 + TDD 铁律 + 跨 Agent 可移植 + 变更生命周期管理** 收敛

---

## 八、最终抽象

```
         方法论文档（TDD/SDD/BDD/DDD/...）
         (人类阅读的书籍、博客、培训)
              │
              ▼
     ┌──────────────────┐
     │  Harness 六层      │  ← 把方法论翻译成该层的确定性机制
     │  L0 → L1 → L2 → L3 → L4 → Meta
     └──────────────────┘
              │
              ▼
     ┌──────────────────┐
     │  可执行约束        │  ← 代码/配置/Skill/Slash Command
     │  · CLAUDE.md     │
     │  · .specify/      │
     │  · Workflow DSL  │
     │  · TDD Skill     │
     │  · Pre-commit    │
     │  · AI Code Review│
     └──────────────────┘
              │
              ▼
     ┌──────────────────────┐
     │  SDD 产品层            │
     │  spec-kit（WHAT）      │
     │  Superpowers（HOW）    │
     │  OpenSpec（LIFECYCLE） │
     └──────────────────────┘
              │
              ▼
     ┌──────────────────────┐
     │  桥接/融合层           │
     │  speckit-superpowers- │
     │  bridge, SpecForge,   │
     │  spec-kit-clean-arch  │
     └──────────────────────┘
```

---

## 关键结论

- **SDD/TDD/BDD/DDD 的本质**：在错误的代价最低的阶段，用结构化的方式消除歧义
- **Harness Engineering 没有发明新方法论**：它将原有方法论**编译成了可执行的约束框架**——从 trusting humans to building process into the harness
- **TDD Skill 是最佳编译案例**：Iron Law + Red Flags + 自判检查表 + 惩罚路径 = 方法论从建议变成铁律
- **SDD 有三种实现形态**：制品型（spec-kit，信任结构化输入）、过程型（Superpowers，信任人类判断）、变更型（OpenSpec，信任生命周期管理）
- **spec-kit vs Superpowers 的根本差异**：前者信任结构化模板 → Agent 自主执行；后者信任人类 gate → 全程确认。两者在 spec 写法、plan 粒度、人类参与深度上完全不同
- **Superpowers 支持多个 Agent**（用户纠正）：通过平台适配层，而非寄生在 slash command 系统上
- **Layer 3（AI 代码审查）有独立生态**（CodeRabbit、Greptile、Qodo），但面临「AI 审 AI 代码」的根本性挑战——87% 的 AI 生成 PR 引入安全漏洞
- **三巨头正在收敛**：speckit-superpowers-bridge、SpecForge 等证明 WHAT + HOW + LIFECYCLE 可以融合
- **所有产品的共同模式**：用确定性的约束包裹非确定性的 LLM — 递归的 Harness Engineering 模式

---

## 引用来源

[^1]: Superpowers 插件源码（skills: brainstorming, writing-plans, test-driven-development, using-superpowers），v6.0.3，通过 `Skill` 工具在 2026-06-29 会话中加载。包含跨 Agent 平台适配层（Claude Code / Codex / Copilot CLI / Gemini CLI / Pi / Antigravity）。

[^2]: Superpowers GitHub: `obra/superpowers`，~167k stars（截至 2026-06）。

[^3]: GitHub spec-kit: `github/spec-kit`，~116k stars，MIT 许可证。README 及 `.specify/templates/` 目录结构通过 WebFetch 在 2026-06-29 获取。包含 7 步 SDD 工作流（constitution → specify → clarify → plan → tasks → implement → analyze），支持 30+ Agent。

[^4]: Web 搜索 "AI coding agent harness tools spec-driven development workflow 2025 2026"（2026-06-29）。覆盖 Forge、OSpec、Trellis、Kiro、BMAD、GSD、SpecForge 等工具信息。

[^5]: GitHub OpenSpec: `Fission-AI/OpenSpec`，~48k stars，npm 安装。README 通过 WebFetch 在 2026-06-29 获取。4 阶段工作流（propose → review → apply → archive），25+ Agent 支持。

[^6]: Web 搜索 "AI code review tools automated PR review agent-generated code quality 2026"（2026-06-29）。CodeRabbit、Graphite Agent、Greptile、Qodo Merge 数据和 "87% AI-generated PR vulnerabilities" 来自 DryRun Security / CSA 2026 年 4 月报告。
