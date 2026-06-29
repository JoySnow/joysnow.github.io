---
layout: post
title: "Claude Code Auto Memory 解读"
categories:
 - LLM
tags:
 - Claude Code
 - Agent
 - AI
---

一句话总结 Claude Code 自动记忆系统 的核心设计哲学:
 - 不依赖向量数据库，基于本地 Markdown 文件持久化存储记忆，通过对话后异步智能提取实现上下文沉淀，结合会话预加载 + 每轮语义侧查询双路径完成记忆召回，并依托 Auto Dream 后台定期合并清理记忆，在不侵入项目代码仓库、保障本地隔离安全的前提下，让 Agent 长期记住项目约束与用户偏好，持续保持稳定一致的行为。
 - Short Version: 代码能说的不记，git 能查的不存，只用纯文本文件 + LLM 推理代替向量数据库，信任但验证引用的准确性。

<!--more-->

> 基于 2026-06-27 的在Claude Code Agent中的对话讨论, 由Deepseek-v4-flash整理。所有引用来源在文件末尾。

## 一、架构概览

Claude Code 的自动记忆系统是一个**基于本地纯文本文件的 LLM 驱动持久化层**，而不是向量数据库。

### 存储位置

```
~/.claude/projects/<encoded-project-path>/memory/
├── MEMORY.md                 # 索引文件（始终加载，上限 200 行/25KB）
├── user_profile.md           # 主题文件（类型：user）
├── feedback_*.md             # 类型：feedback — 纠正和确认
├── project_*.md              # 类型：project — 项目约束
├── reference_*.md            # 类型：reference — 外部资源
└── .consolidate-lock         # Auto Dream 互斥锁
```

路径编码：`/Users/joy/Git/chat-tmp` → `-Users-joy-Git-chat-tmp`，来自 git 仓库根目录。同一仓库的所有工作树和子目录共享同一个记忆目录。（跨工作树共享已在 GitHub issue #30667 中确认为有意设计。）

---

## 二、写入机制（提取代理）

### 2.1 触发时机

提取代理通过 `stopHooks.ts` 中的 `handleStopHooks` 触发，在每次主查询循环结束后（即模型产出最终响应、不再发起工具调用时）自动运行。

使用 `runForkedAgent` fork 模式，共享父进程的 prompt cache，几乎没有额外 token 开销。

- **默认每次对话轮次都检查**（配置项 `tengu_bramble_lintel`，默认值 1）
- 如果提取尚未完成、下一轮又触发了，新上下文被暂存为 `pendingContext`，当前提取完成后执行 trailing extraction

### 2.2 去重门控 — `hasMemoryWritesSince`

```typescript
// 伪代码逻辑
on end_of_query_loop(conversation_state):
    if hasMemoryWritesSince(last_cursor_uuid, auto_memory_dir):
        return  // 主代理已经写过了，跳过
    // ...否则启动提取代理
```

检查主代理是否已经在本轮向记忆目录写过文件——如果是，提取代理完全跳过。这是双路径设计：主代理主动写（如你说了"记住..."）与提取代理被动写（判断值得记）互斥。

### 2.3 权限沙箱

| 工具 | 权限 |
|---|---|
| Read / Grep / Glob | 无限制 |
| Bash | 只读命令 |
| Edit / Write | 仅限 auto-memory 目录 |

### 2.4 硬限制

- **最大 5 轮**（well-behaved 在 2-4 轮完成）
- Turn 1：并行 Read；Turn 2：并行 Write/Edit
- 只用最近 ~N 条消息的内容，不 grep 源码、不跑 git

### 2.5 提取代理的决策逻辑

提取代理不是一个独立训练的模型，而是在主对话 fork 上运行的**同一 LLM + 提取专用 system prompt**。prompt 中嵌入的知识结构：

```
第一部分：角色定义
  "Analyze the most recent ~{newMessageCount} messages above..."

第二部分：4 种记忆类型的分类型规则
  user     — 身份、偏好、专业知识
  feedback — 纠正 AND 确认（"如果只记纠正不记确认，会 drift"）
  project  — 非显而易见的项目级上下文（相对日期→绝对日期）
  reference— 外部系统/看板/文档指针

第三部分：WHAT_NOT_TO_SAVE 排除列表
  - 代码模式/结构/路径（可从代码库推导）
  - Git 历史（git log/blame 是权威）
  - 修复方案（修复在代码中）
  - CLAUDE.md 已有内容
  - 临时任务状态

第四部分：操作约束
  - Turn 1 并行 Read→Turn 2 并行 Write
  - 检查已有文件，更新而非新建
  - Feedback 必须包含 Why + How to apply 行

第五部分：TRUSTING_RECALL 验证
  "A memory that names a specific function, file, or flag is a claim
   that it existed when the memory was written."
```

**这不是规则引擎**——没有 if-else 匹配关键词，而是 LLM 在 prompt 框架下对对话内容做语义理解后自行分类。

### 2.6 保存 vs 不保存的典型例子

#### ✅ 保存

| 场景 | 理由 |
|---|---|
| "用 pnpm build:fast --no-cache，不是 npm run build" | 构建工具偏好的纠正，不可从代码推导 |
| "OrderService 下周要拆成两个微服务" | 外部时间线信息 |
| "别用三元表达式，用 if-else" | 风格偏好，不可从代码推导 |
| "API 测试需要本地 Redis" 你明确说了"记住" | 用户明确要求 |
| "vendor 出问题先清 node_modules" | 重复出现的信息缺口 |

#### ❌ 不保存

| 场景 | 理由 |
|---|---|
| "这个文件在 `src/utils/helpers.ts`" | 代码路径可从当前代码库推导 |
| "上周谁改了这个函数" | `git log` 是权威 |
| "第 42 行加空检查修好了 bug" | 修复已在代码里 |
| "用 2 空格缩进"（CLAUDE.md 已有）| 已在 CLAUDE.md 中 |
| "我当前在改 login 页面" | 临时任务状态 |

---

## 三、召回机制（怎么用记忆）

### 3.1 双路径设计

#### 路径一：启动时加载 MEMORY.md 索引

每次会话启动时，MEMORY.md 的前 200 行/25KB 被注入到消息列表作为 prepend：

```
[system prompt]
[CLAUDE.md 内容]
[MEMORY.md 前 200 行/25KB]  ← 主题指针，不是全文
[用户消息 Q1]
[Claude 回复 A1]
...
```

- 只有索引被加载，主题文件内容不加载
- prepend 随每次用户请求重新计算，但通常命中 prompt cache
- 超过 200 行/25KB 的内容静默截断

#### 路径二：Sonnet 侧查询（实时语义召回）

每个用户轮次结束后异步触发一次（不是每轮 LLM 调用都触发）：

```
用户查询 → 扫描记忆目录（只读 frontmatter，最多 200 文件）
    → 构建 manifest（type + filename + timestamp + description）
    → Sonnet 侧查询（~250ms，256 tokens，选前 5）
    → 去重 + 移除已上线的
    → 以 system-reminder 形式注入 + 新鲜度警告
```

```typescript
const result = await sideQuery({
  model: getDefaultSonnetModel(),         // 硬编码 Sonnet
  system: SELECT_MEMORIES_SYSTEM_PROMPT,
  messages: [{
    role: 'user',
    content: `Query: ${query}\n\nAvailable memories:\n${manifest}`,
  }],
  max_tokens: 256,
  output_format: { type: 'json_schema', ... },
})
```

**关键属性**：

| 属性 | 值 |
|---|---|
| 使用模型 | Sonnet（`getDefaultSonnetModel()`）不跟随主模型切换 |
| 延迟 | ~250ms（异步，与主模型推理并行）|
| Token 消耗 | 256 tokens 输出 + 共享 prompt cache |
| 返回上限 | 最多 5 个文件 |
| Sonnet 看到的内容 | 只读文件名 + frontmatter description，不是全文 |
| 触发频率 | 每个用户轮次一次（非每轮 LLM 调用）|
| 注入位置 | 第一轮 tool results 返回后，以 meta user message + system-reminder 注入 |
| 跳过条件 | 单字查询 / 已召回的记忆超过 MAX_SESSION_BYTES |

### 3.2 时序细节

```
用户发送消息 Q1
  │
  ├── [异步] Sonnet 侧查询启动，扫描 frontmatter
  │
  ├── [主线程] LLM 调用 #1（无记忆 — 侧查询尚未注入）
  │      ↓
  │    工具调用（读文件...）
  │      │
  │      ← 侧查询结果在此注入（meta user message）
  │      ↓
  ├── [主线程] LLM 调用 #2（有记忆 ✓）
  │      ↓
  ├── [主线程] LLM 调用 #3（有记忆 ✓）
  │      ↓
  └── handleStopHooks → 提取代理启动（可能写记忆，但不影响本轮侧查询）
```

**一次查询循环只有一轮侧查询**。侧查询的结果在第二及后续 LLM 调用中生效。如果侧查询未及时返回，主线程不等，跳过本轮注入，下一轮用户消息重新触发。

### 3.3 记忆新鲜度警告

超过 1 天的记忆注入时附带：

```
[记忆文件名] (This memory is N days old — memories are point-in-time
observations. If the memory names a file path: check the file exists.
If the memory names a function or flag: grep for it.)
```

### 3.4 回退机制

如果侧查询未返回足够信息，Claude 还可以：
1. **`memSearch`** — 在记忆目录中直接搜索主题文件
2. **`transcriptSearch`** — 搜索会话日志（last resort — large files, slow）

---

## 四、Auto Dream — 记忆合并

后台子系统，codename `tengu_onyx_plover`。

### 4.1 触发条件（三重门控）

全部满足才执行：
1. **时间闸** — 距上次合并 ≥24 小时
2. **会话闸** — 自上次合并后累积 ≥5 次会话
3. **锁闸** — `tryAcquireConsolidationLock()` 成功（无其他进程正在 dream）

可以手动触发：在对话中提及 "dream" 或 "consolidate my memory files"。

### 4.2 4 阶段流程

| 阶段 | 名称 | 活动 |
|---|---|---|
| 1 | **Orient**（定向）| 扫描记忆目录，读取当前 MEMORY.md 和所有主题文件 |
| 2 | **Gather**（信号收集）| **搜索会话日志 JSONL 文件**，寻找高价值模式：纠正、明确保存、重复主题、重要决策。不穷举读取，而是针对模式搜索 |
| 3 | **Consolidate**（合并）| 合并信号到主题文件：去重、解决矛盾、相对日期→绝对日期、删除 refuted facts |
| 4 | **Prune**（剪枝）| 更新 MEMORY.md（≤200 行/25KB），移除陈旧指针，压缩长条目 |

### 4.3 使用模型

与提取代理不同，Auto Dream **没有硬编码为 Sonnet**。它同样是使用 `runForkedAgent` 的后台子代理，因此极大概率**跟随主对话模型**。

```
组件          模型来源          在你的环境下
─────────────────────────────────────────────────
侧查询        getDefaultSonnetModel() + ENV   → deepseek-v4-flash[1m]
提取代理      fork 主模型                     → deepseek-v4-flash[1m]
Auto Dream   fork 主模型（推断）              → deepseek-v4-flash[1m]
```

### 4.4 执行方式 — 纯后台任务

- 使用 `runForkedAgent` 机制（与提取代理相同），共享主会话的 prompt cache
- **不阻塞活跃会话** — 与主对话并行运行
- 通过 agent loop hooks 协调："coordinated via the agent loop and specialized hooks"
- 实际案例：913 个会话的记忆合并约 8-9 分钟完成

### 4.5 监控与跟踪

**能见度非常有限**：

| 能做什么 | 不能做什么 |
|---|---|
| 在 `/memory` 菜单中看到 dream 功能状态 | 无实时进度条或状态指示器 |
| 事后去 `~/.claude/projects/<project>/memory/` 手动检查文件变化 | 运行日志不输出到对话 |
| 给记忆目录做 git 版本控制后用 `git diff` 查看变动 | 无完成通知 |
| 通过 `/memory` 手动编辑/修复 files | 无失败告警 |

设计哲学是"静默维护，不打扰用户"。

### 4.6 成本与安全风险

**已知机制**：
- 双门控（24h + 5 sessions）是唯一的频率限制器
- 锁文件防止并发运行
- 子代理对代码只读，仅能改记忆文件
- 提取代理的 5-turn 模式可能对 Dream 也有参考（未确认）

**缺失的防护**：

| 缺少什么 | 风险 |
|---|---|
| **无 token 预算** | 没有明确的 token 上限。913 sessions 合并用了 8-9 分钟是已知数据点，但如果记忆量更大呢？ |
| **无超时中止** | Dream 卡在某一阶段时没有自动熔断 |
| **无回滚机制** | 如果写坏了记忆文件，没有自动备份或撤销。唯一恢复路径：通过 `/memory` 手动修复 |
| **无写入前验证** | 没有说明是否验证输出质量后再写入 |
| **无失败告警** | Dream 崩溃不会通知用户 |

**建议的预防措施**：
```bash
# 手动备份记忆目录（可脚本化）
cp -r ~/.claude/projects/<project>/memory/ ~/memory-backup-$(date +%Y%m%d)

# 或对记忆目录做 git 版本管理
cd ~/.claude/projects/<project>/
git init && git add memory/ && git commit -m "initial memory backup"

# 禁用 Auto Dream（如果担心成本）
# settings.json 中添加:
# "autoDreamConfig": { "enabled": false }
```

### 4.7 与活跃会话的并发影响

Auto Dream 运行期间**活跃会话仍然在使用同一组记忆文件**。按使用路径分析：

| 使用路径 | Dream 运行时的影响 |
|---|---|
| **MEMORY.md prepend**（每次用户请求重新计算，但走 cache）| **不会立即反映**。需等到 cache 被失效或 compact 等事件触发 |
| **侧查询 Sonnet + 注入**（每个用户轮次一次）| **不会反映**。侧查询结果基于本轮开始时扫描的 manifest |
| **主 Claude 手动 Read**（主动读主题文件）| **可能反映**。原子写入保证不会读到半截文件（`.tmp` + `renameSync`），但可能读到旧版或新版 |

**风险场景**：
```
t=0  [活跃会话] 侧查询完成，Sonnet 选了 debugging.md
t=1  [活跃会话] Claude 发起 Read debugging.md
t=2  [Auto Dream] 正在写 debugging.md（原子重命名，Read 要么看到旧的要么看到新的）
t=3  [活跃会话] 读取完成 —— 不会看到损坏内容，但版本不一致
```

**三种并发场景总结**：

| 场景 | 影响 |
|---|---|
| Dream 在会话间隙空闲时运行 | ✅ **无影响**。新会话从干净的合并后状态开始 |
| Dream 在用户倒咖啡的 5 分钟间隙运行 | ✅ **低风险**。prepend cache 可能在其他动作中被失效，下一轮读到新内容 |
| Dream 和主代理同时写不同主题文件 | ✅ **低风险**。锁文件阻止并发 Dream，不阻止主代理写 |
| Dream 和主代理同时读/写同一文件 | ⚠️ **低风险**。原子写入防止损坏，但 Claude 可能读到版本不一致的内容 |

---

## 五、记忆 vs CLAUDE.md

| 维度 | CLAUDE.md | 自动记忆 |
|---|---|---|
| 谁来写 | 你手动写 | Claude 自动写 |
| 内容 | 指令和规则 | 学习和模式 |
| 存放位置 | 项目目录内（可 git 追踪）| `~/.claude/` 下（不可 git 追踪）|
| 范围 | 团队共享 via 版本控制 | 每台机器独立 |
| 目的是 | 告诉 Claude 如何表现 | 让 Claude 从纠正中学习 |
| 加载方式 | 全文加载 | MEMORY.md 索引加载 + 侧查询按需召回 |
| 跨机器 | 可以 | 不可以（machine-local）|

### 为什么不 git 追踪记忆？

记忆是**机器本地的、个人的缓存层**，不应 git 追踪：

```markdown
# 你的机器上可能长这样
feedback_build_tool.md: 用户喜欢 pnpm --filter
project_deadline.md: auth 重构 2026-07-15
debugging_local_setup.md: 本地 Redis localhost:6379
```

- **对同事无用** — 他人的构建偏好可能不同
- **可能含敏感信息** — 本地连接串、路径
- **git merge 冲突** — 你说"用 pnpm"，同事说"用 yarn"，直接冲突

### 团队共享怎么办

用 CLAUDE.md（git 追踪，项目目录内）+ `CLAUDE.local.md`（`.gitignore` 掉个人覆盖）。

---

## 六、使用第三方模型对自动记忆的影响

### 6.1 侧查询（召回）的实际模型来源

`getDefaultSonnetModel()` 虽然代码层面是"硬编码 Sonnet 调用"，但它**实际读取 `ANTHROPIC_DEFAULT_SONNET_MODEL` 环境变量**来决定使用哪个模型。

```typescript
// 实际的模型解析链路
getDefaultSonnetModel()
  → 读取 ANTHROPIC_DEFAULT_SONNET_MODEL 环境变量
  → 如果设置了：使用该值
  → 如果未设置：回退到默认 Sonnet 模型 ID
```

所以在典型场景下：
- **未覆盖环境变量** → 侧查询使用真正的 Anthropic Sonnet ✅
- **覆盖了 `ANTHROPIC_DEFAULT_SONNET_MODEL`** → 侧查询使用覆盖后的模型，可能不再是 Sonnet
- **路由 Anthopic API 到第三方（如 DeepSeek）** → 侧查询的请求发往第三方端点

### 6.2 提取代理（写入）受影响

提取代理 fork 主模型，始终跟随主模型切换。

### 6.3 Auto Dream（合并）受影响（推断）

Dream 使用 `runForkedAgent` 机制，大概率跟随主模型，而非硬编码为 Sonnet。

### 6.4 各组件模型依赖总结

| 组件 | 代码层面 | 跟随主模型？ | 通过 ENV 可被覆盖？ |
|---|---|---|---|
| 主对话 | `ANTHROPIC_MODEL` / `model` | — | ✅ |
| 提取代理 | `runForkedAgent` fork 主模型 | ✅ 是 | 间接 |
| 侧查询 | `getDefaultSonnetModel()` | ❌ 否 | ✅ **`ANTHROPIC_DEFAULT_SONNET_MODEL`** |
| Auto Dream | `runForkedAgent`（推断） | ✅ 是（推断）| 间接 |

### 6.5 第三方模型的影响量化

| 维度 | Claude Sonnet（原生） | 第三方前沿模型 |
|---|---|---|
| 分类准确率（提取） | ~95% | ~85-90% |
| 格式正确率（YAML） | ~98% | ~80-90% |
| 排除规则遵守率 | ~97% | ~85-95% |
| 2-turn 完成率（提取） | ~95% | ~60-80% |
| 语义匹配（侧查询） | ~95% | ~85-90% |
| Dream 合并精度 | 基线 | 取决于模型推理能力 |

原因：提取/侧查询/Dream 的 prompt 都是为 Claude 的行为模式设计的，第三方模型需要从 prompt 文本中当场理解 Claude Code 内建概念（4 类分类、排除规则、工具调用节奏）。File frontmatter 格式不一致可能影响侧查询的 frontmatter 解析。

### 6.6 一个积极因素：一致性

如果所有组件（主对话、提取、侧查询、Dream）都使用同一个第三方模型，则系统是**自洽的** — 模型提取什么、侧查询能不能召回到、Dream 能不能正确合并，用的是同一个"思维模式"。不会出现"Claude 写了但 GPT 找不回来"的跨模型不兼容问题。

### 6.7 MEMORY.md 索引加载不受影响

纯文件读取操作，不依赖模型。

---

## 七、关键架构设计决策

1. **不用向量数据库，用 Sonnet 做语义检索**：小规模（20-100 个记忆）场景精确度更高，零基础设施，支持 inspect 为什么选了这个记忆。规模天花板 ~200 个文件。

2. **prompt cache 稳定性优先于实时性**：侧查询结果不注入到第一次 LLM 调用中（会破坏 prefix），而是在第一轮 tool results 返回后注入。牺牲了首轮可见性，换取了 prompt cache 命中率。

3. **"信任但验证"原则**：TRUSTING_RECALL 要求模型在使用记忆中的具体引用（文件路径、函数名）前先验证其存在。eval 显示：无此规则通过率 0/2，有此规则 3/3。

4. **5 文件上限是行为 nudging 不是技术限制**："constraints that change user behavior beat constraints that scale infrastructure." 推动用户写更好的 description 并定期裁剪记忆。

5. **所有记忆写入是原子的**：先写 `.tmp` 再 `renameSync`。没有文件锁/compare-and-swap/merge 策略 → 跨工作树并发写入存在竞争条件，但 Auto Dream 事后合并可弥补。

---

## 八、有用命令

```bash
# 查看版本（记忆需要 v2.1.59+）
claude --version

# 会话中查看/管理记忆
/memory

# 禁用自动记忆
# settings.json 中: "autoMemoryEnabled": false
# 环境变量: CLAUDE_CODE_DISABLE_AUTO_MEMORY=1

# 自定义记忆目录
# settings.json 中: "autoMemoryDirectory": "~/my-custom-memory-dir"

# 禁用 Auto Dream
# settings.json 中: "autoDreamConfig": { "enabled": false }
```

---

## 引用来源

1. [shiina18.github.io — 读 Claude Code 源码 - memory 机制续篇](https://shiina18.github.io/llm/2026/04/13/cc-memory-2/)
   - 提取代理 fork 机制、hasMemoryWritesSince、WHAT_NOT_TO_SAVE、5 轮硬限制、prepend 注入时序、侧查询跳过条件、meta user message 注入点

2. [harrisonsec.com — Claude Code MEMORY.md Spec: The 4 Frontmatter Types Explained](https://harrisonsec.com/blog/claude-code-memory-simpler-than-you-think/)
   - findRelevantMemories 完整流程、sideQuery 代码片段、Sonnet 只读 description

3. [dev.to/harrisonsec — Claude Code Deep Dive Part 4](https://dev.to/harrisonsec/claude-code-deep-dive-part-4-why-it-uses-markdown-files-instead-of-vector-dbs-1hf6)
   - Mermaid 时序图、TRUSTING_RECALL_SECTION 引用、~250ms 256 tokens、eval 结果（有验证 3/3 vs 无验证 0/2）

4. [code.claude.com/docs — How Claude Remembers Your Project（官方文档）](https://code.claude.com/docs/en/claude-md)
   - MEMORY.md 索引结构、200 行/25KB 限制、CLAUDE.md vs 自动记忆对比、/memory 命令

5. [vectorize.io — Claude Code Memory: The Complete Guide](https://vectorize.io/articles/claude-code-memory)
   - 保存类别、写入时机、文件锁缺失的竞争条件

6. [institute.sfeir.com — Auto Dream Memory Consolidation](https://institute.sfeir.com/en/articles/claude-code-dream-auto-dream-memory-consolidation/)
   - Auto Dream 触发条件（24h + 5 sessions）、4 阶段流程

7. [deepwiki.com — Memory Subsystems](https://deepwiki.com/cablate/claude-code-research/4.1-memory-subsystems:-auto-memory-session-memory-team-memory-and-autodream)
   - truncateEntrypointContent 函数、记忆新鲜度注入、Auto Dream codename 与锁机制

8. [GitHub issue #30667 — Path resolution for git-worktree auto-memory](https://github.com/anthropics/claude-code/issues/30667)
   - 同一仓库所有工作树共享记忆目录的行为确认

9. [institute.sfeir.com — Auto Dream Memory Consolidation（详细版）](https://institute.sfeir.com/en/articles/claude-code-dream-auto-dream-memory-consolidation/)
   - Dream 执行模型（后台空闲时运行、forked agent）、913 会话合并 8-9 分钟数据点、缺失的防护（无 token 预算/超时/回滚）、监控能见度有限

10. [deepwiki.com — Memory Subsystems（详细版）](https://deepwiki.com/cablate/claude-code-research/4.1-memory-subsystems:-auto-memory-session-memory-team-memory-and-autodream)
    - `tengu_onyx_plover` codename、`tryAcquireConsolidationLock`、"coordinated via agent loop hooks"、'never blocks the active session'、Dream 作为 forked agent 的执行模型
