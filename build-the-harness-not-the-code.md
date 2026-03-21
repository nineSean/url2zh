# 构建 harness，而不是代码：面向 Staff/Principal Engineer 的 AI agent 系统指南

> 原文: https://vitthalmirji.com/2026/02/build-the-harness-not-the-code-a-staff/principal-engineers-guide-to-ai-agent-systems/
> 作者: Vitthal Mirji
> 日期: 2026-02-19（更新于 2026-03-07）

---

OpenAI 的一个团队交付了一款真正上线运行的产品，而且 **没有一行代码是人工手写的**。2025 年 8 月时，它还只是一个空的 git 仓库；五个月后，代码规模已经来到 100 万行。最初只有 3 名工程师参与，平均每人每天产出 3.5 个 PR；更惊人的是，团队扩展到 7 人后，吞吐量反而还在继续提升。

这不是一个研究演示。该产品每天拥有数百名内部用户和外部 alpha 测试人员。

这个约束是刻意设计出来的：**humans steer, agents execute**。应用逻辑、测试、CI 配置、文档、内部工具，这些代码全都是由 Codex 写出来的。他们估算，整个产品的构建速度大约是纯手写代码方式的 10 倍。

那么人类到底做了什么？

他们构建了 harness。

**而且，你也可以用 Claude Code 搭出同样的 harness。**

**Info**

本指南的内容

- **3 core harness lessons** 来自 OpenAI 的内部故事，具有并行的 Claude Code 实现模式
- **10 operational tips** 用于技能、shell 执行和上下文压缩（OpenAI + Claude 路径）
- **15 hard-won lessons** 来自构建 ChatGPT Apps，适用于 Claude 桌面和 MCP 服务器
- **Architecture enforcement patterns** 通过钩子进行机械 linting (OpenAI + Claude)
- **11-step autonomy ladder** 显示从 bug 重现到自主 PR 合并的路径
- **Quarterly implementation blueprint** 对于 Codex 和 Claude Code
- **Metrics, failure scenarios, and tool-agnostic adaptation** 已在整个生态系统中得到验证

**这份指南会把两条路径都讲清楚。** 无论您是使用 OpenAI Codex、Claude Code，还是计划在它们之间进行切换，您都将了解在任何地方都适用的底层模式。

这篇文章可以和 [Build a code review operating system](https://vitthalmirji.com/code-review-system/) 搭配阅读。本文关注的是如何搭建 harness；另一篇则聚焦于如何通过 code review 把反馈回路闭合起来。

---

## harness engineering 到底是什么意思

你知道事情是怎么回事——你是一名高级工程师，你花时间审查代码，修复架构偏差，确保初级开发人员遵循约定。现在想象一下，“初级开发人员”是一个 AI agent，其编写代码的速度比任何人快 10 倍，但没有机构记忆，没有品味，也不理解为什么选择 PostgreSQL 而不是 DynamoDB。

harness 是你围绕 agent 搭建的一整套东西，用来让它的输出变得可靠。说明、工具、检查机制、上下文、反馈回路，都属于 harness 的一部分。它本质上是在设计环境，让“agent 会写代码”升级成“agent 能交付生产级功能”。

OpenAI 是付出过代价之后才学到这一点的。早期进展比预想中慢，不是因为 Codex 做不到，而是因为环境定义得还不够完整。agent 缺少完成高层目标所需的工具、抽象和内部结构。不再由人类亲手写代码之后，工程工作并没有消失，而是变成了另一种形态：重点转向系统、脚手架，以及能产生复利的工程杠杆。

在实践里，这意味着要用深度优先的方式工作：先把更大的目标拆成更小的构建块（设计、编码、审查、测试），让 agent 先把这些能力拼起来，再用这些能力去解锁更复杂的任务。当事情失败时，修复办法几乎从来不是“再试一次”或“更努力一点”，而是回到这个问题：**到底缺了什么能力？又该怎样把它变成 agent 看得懂、而且能被强制执行的东西？**

这不是抽象的理念，而是一套可操作的方法。本文要讲的，正是怎么把这套方法真正落到工程实践里。

我一直在我自己的项目 [flowforge](https://github.com/com-vitthalmirji/flowforge) 中应用这些模式，这是一个编译时数据契约框架。在整篇文章中，我偶尔会展示我如何在实践中应用模式。但教学来自四个关键来源：

1. **harness engineering** - OpenAI 以零手动代码交付 100 万行的内部故事
2. **Skills + shell + compaction** - 长时间运行agent的操作原语（OpenAI 和 Claude）
3. **15 lessons building ChatGPT Apps** - 为 Claude Desktop 和 MCP 改编的来之不易的课程
4. **Platform shifts in 2025** - OpenAI 和 Anthropic 生态系统发生了什么变化

---

## 心智模型

```
flowchart LR
    A[Business intent] --> B[Harness design]
    B --> C[Agent execution]
    C --> D[Validation and evals]
    D --> E[Auto-remediation and learning]
    E --> B
```

您设计地图（说明、边界、工具），而不是一本巨大的手册。您将质量编码为检查，而不是部落评论。您将上下文工件视为基础设施。

这是我最信任的启发式：**anything that helps humans ship better software faster usually helps agents do the same.**

技能使这一点显而易见。如果您使用过内部工程 wiki，那么您已经看过这部电影：针对重复出现的情况而提炼的剧本，易于查找，适用于具体工作。这种模式多年来一直对人类有效。特工技能是相同的模式，现在可以执行。

相同的逻辑也适用于编译器和类型系统、依赖注入、错误值和效果系统。这些抽象概念之所以能够幸存下来，并不是因为它们很时尚。它们之所以能够幸存下来，是因为它们提高了实际代码库的可靠性并改变了速度。agent商也能从这些限制中受益。

我仍然不断看到“语言并不重要，agent只会发出汇编”的说法。我不买。如果一个工作流程对于人类来说很难推理、审查和维护，那么对于大规模的agent来说，它通常也会崩溃。老实说，我厌倦了“AI是外星魔法”的框架。事实并非如此。AI不会抹杀工程基础知识；它放大了它们。强大的系统变得更强。脆弱的系统失败得更快。

**This pattern is tool-agnostic.** 无论您是使用 Codex、Claude code、Gemini 还是 Grok 进行构建，循环都保持不变。

---

## 第 1 课：给agent一张地图，而不是手册

OpenAI 和 Anthropic 团队都吸取了同样的教训：“一个大指令文件”方法的失败是可以预见的。

**Warning**

为什么巨大的指令文件会失败

- **Context is a scarce resource.** 巨大的指令文件挤占了任务、代码和相关文档 - 所以
agent要么错过关键约束，要么开始针对错误的约束进行优化。
- **Too much guidance becomes non-guidance.** 当一切都“重要”时，就没有什么是重要的了。agent最终进行模式匹配
本地而不是有意导航。
- **It rots instantly.** 庞大的手册变成了陈旧规则的坟墓。特工们无法判断什么仍然是真实的，
人类不再维护它，文件就悄悄地变成了一种有吸引力的麻烦。
- **It’s hard to verify.** 单个 blob 不适合进行机械检查（覆盖范围、新鲜度、所有权、
交叉链接），因此漂移是不可避免的。

### AGENTS.md 中实际应该包含哪些内容

现在有基准证据表明这不仅仅是一种风格偏好。在 Addy Osmani 强调的分析（涵盖 SWE-bench 风格agent运行）中，添加自动生成的 `AGENTS.md` 摘要使任务成功率降低了大约 2-3 个百分点，同时使代币成本增加了约 20-30%。

失败模式很简单：agent已经可以检查您的树、推断您的堆栈并按需读取模块文档。如果您预加载与散文相同的信息，则会产生上下文锚定噪音。该模型一直关注房间里的粉红色大象，而不是它应该做出的具体差异。

对 `AGENTS.md` 中的内容使用严格的过滤器：

1. **Undiscoverable operational constraints**：仅从源代码中看不到的设置和工具陷阱
2. **Operational landmines**：“清理”可能破坏生产行为的风险区域
3. **Non-obvious conventions**：故意的本地模式，一般看起来是错误的，但对于这个代码库来说是正确的

其他一切都应该链接起来，而不是重复。如果agent可以通过读取存储库发现它，请将其从指令文件中删除。

将 `AGENTS.md` 视为临时气味跟踪器，而不是永久的知识转储。如果agent反复滥用依赖项、写入错误的文件夹或不断违反架构边界，真正的修复通常是结构性的：重命名模块、改进文件夹语义、添加 linter/hooks、加强测试。然后删除补偿散文。

### 解决方案：索引文件作为目录

**OpenAI 的做法：AGENTS.md**

OpenAI 没有将 `AGENTS.md` 视为百科全书，而是将其视为 **the table of contents**。存储库的知识库位于结构化的 `docs/` 目录中。一个简短的 `AGENTS.md`（大约 100 行）被注入到上下文中，主要用作地图，并指向更深层次的事实来源。

**OpenAI's actual repository structure**

|  |  |
| --- | --- |
| ```  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 ``` | ``` AGENTS.md              (100 lines, injected into context as the map) ARCHITECTURE.md        (top-level map of domains and package layering) docs/ ├── design-docs/ │   ├── index.md │   ├── core-beliefs.md   (agent-first operating principles) │   └── ... ├── exec-plans/ │   ├── active/ │   ├── completed/ │   └── tech-debt-tracker.md ├── generated/ │   └── db-schema.md ├── product-specs/ │   ├── index.md │   ├── new-user-onboarding.md │   └── ... ├── references/ │   ├── design-system-reference-llms.txt │   ├── nixpacks-llms.txt │   └── ... ├── DESIGN.md ├── FRONTEND.md ├── QUALITY_SCORE.md └── SECURITY.md ``` |

**计划是一等工件。** 活动计划、已完成的计划和已知的技术债务都经过版本控制并位于同一位置，允许agent在不依赖外部上下文的情况下进行操作。

**Claude Code 的做法：`CLAUDE.md` + `.claude/` 目录**

[Claude Code uses `CLAUDE.md`](https://code.claude.com/docs/en/best-practices) 作为agent的“宪法” - 它是您的特定存储库如何工作的主要事实来源。与 AGENTS.md 不同，CLAUDE.md 是 **automatically read at the start of each session** 并包含项目特定的说明，否则您将在每个提示中重复。

文件名区分大小写，并且必须完全是 `CLAUDE.md`（大写 CLAUDE，小写 .md）。

**Claude Code 的仓库结构**

|  |  |
| --- | --- |
| ```  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 ``` | ``` CLAUDE.md              (50-120 lines, auto-loaded every session) .claude/ ├── agents/            (specialized subagents) │   ├── reviewer.md │   ├── security.md │   └── test-writer.md ├── commands/          (custom slash commands) │   └── deploy.md ├── hooks/             (lifecycle automation) │   ├── PreToolUse.sh │   ├── PostToolUse.sh │   └── AfterCompaction.sh └── skills/            (versioned procedure bundles)     ├── api-design.md     └── schema-migration.md docs/ ├── architecture/ │   └── decisions.md  (ADRs) ├── evidence/ │   └── current-state.md └── runbooks/     └── deployment.md ``` |

**和 OpenAI 方案的关键差异：**

- `.claude/agents/` 持有 [specialized subagents](https://code.claude.com/docs/en/sub-agents)
在孤立的上下文中运行
- `.claude/hooks/` 在 [15 lifecycle events](https://docs.anthropic.com/en/docs/claude-code/hooks) 机械地强化质量
- CLAUDE.md 更小（50-120 行 vs 100+ 行），因为钩子处理执行
- [MCP servers](https://code.claude.com/docs/en/mcp)
通过标准化协议提供外部工具访问

### 两个生态系统中的渐进式披露

```
flowchart LR
    subgraph openai ["OpenAI: AGENTS.md"]
        A1["AGENTS.md ~100 lines"] -->|points to| A2["docs/design-docs/"]
        A1 -->|points to| A3["docs/exec-plans/"]
        A2 -->|deep dive| A4["core-beliefs.md"]
    end
    subgraph claude ["Claude: CLAUDE.md + .claude/"]
        C1["CLAUDE.md 50-120 lines"] -->|delegates to| C2[".claude/agents/"]
        C1 -->|links to| C3["docs/architecture/"]
        C2 -->|runs| C4["reviewer.md security.md"]
    end
    style openai fill: #d4edda, stroke: #28a745
    style claude fill: #e8f4fd, stroke: #0d6efd
```

agent首先读取索引文件（AGENTS.md 或 CLAUDE.md）。从那里，它会链接到与当前任务相关的任何深度文档。它永远不会一次加载整棵树。

### 机械执法

**OpenAI:** 专用的 linter 和 CI 作业可验证知识库是最新的、交叉链接的且结构正确。定期的“文档园艺”agent会扫描陈旧或过时的文档并打开修复拉取请求。

**Claude code:** [Hooks provide deterministic enforcement](https://aiorg.dev/blog/claude-code-hooks) 。 PreToolUse 挂钩会在危险操作运行之前阻止它们。 PostToolUse 挂钩在代码更改后立即强制格式化。 AfterCompaction 钩子在汇总后注入关键上下文。

与 [Claude Code best practices](https://code.claude.com/docs/en/best-practices) 的主要区别：**If it’s a suggestion, use CLAUDE.md. If it’s a requirement, use hooks.**

**Tip**

如何自己实现

**For OpenAI codex:**

1. 以 `AGENTS.md` 作为索引开始（120 行以下）
2. 按关注点将文档拆分到 `docs/` 子目录中
3. 添加带有日期、状态标记记录的 ADR 决策
4. 创建包含文件路径和量化声明的证据文档
5. 通过 CI 作业机械地执行以获得新鲜度和交叉链接
6. 按计划运行文档园艺agent

**For Claude code:**

1. 在存储库根目录中创建 `CLAUDE.md`（50-120 行，仅限项目特定说明）
2. 专门子agent的结构 `.claude/agents/`（审查、安全、测试编写）
3. 定义 `.claude/hooks/` 以强制执行（PreToolUse 块、PostToolUse 格式）
4. 从 CLAUDE.md 链接到 `docs/architecture/` 以进行深入研究
5. 使用[MCP servers](https://code.claude.com/docs/en/mcp)
用于外部工具访问
6. 让[automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)
使用 AfterCompaction 挂钩管理上下文以固定关键状态

**Universal patterns (both ecosystems):**

- 保持索引文件最少（仅建议）
- 使用深层链接文档获取详细上下文
- 机械地强制质量（linter/hooks，而不是散文）
- 对存储库中的所有决策和计划进行版本控制

> **In flowforge**，我使用两种模式：`AGENTS.md` 与 `docs/adr/INDEX.md` 链接的 Codex 兼容性（25+
> 决策），以及在每次代码更改后运行 scalafix 的 `.claude/hooks/PostToolUse.sh`，确保黄金原则
> 无论我使用哪个agent，都会强制执行。

---

## 第 2 课：自动清理，否则就会陷入其中

完全的agent自主引入了特定的故障模式。agent复制存储库中已存在的模式 - 甚至是不均匀或次优的模式。随着时间的推移，这不可避免地会导致漂移。

**Danger**

跳过此步骤的成本

**OpenAI’s team spent every Friday - 20% of their week - cleaning up “AI slop.”** 那没有规模。如果您不机械地编码清理规则，您将花费一周五分之一的时间来手动完成。

### 黄金原则：编码品味作为执行

这两个生态系统都学会了编码“黄金原则”——固执己见的机械规则，使代码库在未来的agent运行中保持清晰和一致。

**OpenAI 的生产实践示例：**

1. **Prefer shared utility packages** 过度手动帮助以保持不变量集中。
2. **Parse data shapes at the boundary.** 不要探测“YOLO 式”数据 - 验证边界或依赖类型化 SDK。
遵循[“parse, don’t validate”](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)原则。
3. **Statically enforce structured logging**、模式和类型的命名约定、文件大小限制以及
具有自定义 lint 的特定于平台的可靠性要求。
4. **Custom lint error messages inject remediation instructions into agent context.** 当 lint 失败时，错误
消息告诉agent *how* 修复它。

后台 Codex 任务会定期扫描偏差、更新质量等级并打开有针对性的重构 PR。大多数都可以在一分钟内完成审核并自动合并。

**Claude Code 的做法：用 hooks 做确定性约束**

Claude Code 使用 [hooks at 15 lifecycle events](https://docs.anthropic.com/en/docs/claude-code/hooks) 来执行黄金原则。 [production engineering teams](https://www.pixelmojo.io/blogs/claude-code-hooks-production-quality-ci-cd-patterns) 的关键见解：

> **CLAUDE.md rules are suggestions. Hooks are enforcement.** CLAUDE.md 说“不要编辑 .env” → 由 LLM 解析 →
> 与其他背景进行权衡→可能会遵循。 PreToolUse 钩子阻止 .env 编辑 → 始终运行 → 返回退出代码
> 2 → 操作被阻止。

实施黄金原则的示例 `.claude/hooks/PostToolUse.sh`：

|  |  |
| --- | --- |
| ```  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 ``` | ``` #!/bin/bash # 每次写入文件后运行 # 黄金原则 1：生产代码中没有原始 console.log if echo "$TOOL_OUTPUT" | grep -q "console\.log" && [[ ! "$FILE_PATH" =~ 测试]]; then echo "❌ 阻止：使用结构化日志记录 (logger.info) 而不是 console.log" exit 2 # 阻止操作 fi # 黄金原则 2：自动格式化所有代码更改 if [[ "$FILE_PATH" =~ \.(ts | js | tsx | jsx)$]];然后 npx prettier --write "$FILE_PATH" > /dev/null 2>&1 fi # 黄金原则 3：运行 linter 并自动修复 if [[ "$FILE_PATH" =~ \.(ts | tsx)$]];然后 npx eslint --fix "$FILE_PATH" > /dev/null 2>&1 fi exit 0 # 执行后允许操作 ``` |

来自 [production repos](https://github.com/affaan-m/everything-claude-code) 的其他克劳德代码模式：

- **PreToolUse** 钩子可防止危险操作（删除生产数据库、暴露秘密）
- **AfterCompaction** 钩子注入关键上下文（主动计划，黄金原则）
- **SessionEnd** 挂钩生成摘要和清理工件

### agent商青睐钻孔技术

OpenAI 和 Anthropic 团队都喜欢依赖关系和抽象，这些依赖关系和抽象可以完全内化并在回购协议中进行推理。由于可组合性、API 稳定性和训练集中的表示，通常被描述为“无聊”的技术往往更容易让agent建模。

OpenAI 的示例：他们没有引入通用的 `p-limit` 式包，而是实现了自己的带有并发映射的帮助程序 - 与 OpenTelemetry 仪器紧密集成，具有 100% 的测试覆盖率，其行为完全符合其运行时的预期。

### 反馈循环

```
flowchart LR
    A[Agent output] --> B[Golden principles scanner]
    B -->|pass| C[merge]
    B -->|fixable| D[auto-fix PR]
    B -->|not fixable| E[human escalation]
    E --> F[update golden principles]
    F --> B
    D --> G[quality grade updated]
    G --> B
```

**OpenAI:** 审核评论、重构 PR 和面向用户的错误被捕获为文档更新或直接编码到工具中。当文档不足时，规则就会被提升为代码。

**Claude code:** 挂钩执行失败会触发人为升级。该修复成为新的挂钩或更新的 CLAUDE.md 规则。随着时间的推移，安全带会从失败中吸取教训。这就是它变成 [code review operating system](https://vitthalmirji.com/code-review-system/) 的原因，而不仅仅是一个 linter。

**Tip**

如何自己实现

**For OpenAI codex:**

1. 定义 5-10 条黄金原则（您的团队今天手动执行的机械规则）
2. 将它们编码为自定义 linter 或结构测试
3. 使 lint 错误成为规范（告诉agent *how* 修复违规行为）
4. 每天运行漂移扫描，自动打开 PR 以发现低风险违规行为
5. 跟踪手动清理时间（目标：趋向于零）

**For Claude code:**

1. 在 `.claude/hooks/PostToolUse.sh` 中定义黄金原则
2. 使用退出代码 2 阻止违规，自动修复后退出 0
3. 添加 AfterCompaction 挂钩以保留压缩过程中的原则
4. 使用PreTool使用钩子来防止危险操作
5. 跟踪挂钩触发率和人为升级作为指标

**通用模式：**

- 从 5-10 条规则开始，随着模式的出现而扩展
- 青睐无聊的可组合技术（agent对其进行可靠建模）
- 使错误消息具有规定性（不仅仅是“什么”而是“如何”）
- 根据实际失败而非假设风险更新原则

> **In flowforge**，我使用 `.scalafix.conf`（禁止 `var`、`asInstanceOf`、通配符导入）来实现 Codex 兼容性，
> 加上自动运行 scalafix 的 `.claude/hooks/PostToolUse.sh`。我还维护**必须的编译失败测试
> 编译失败** - 证明当合约漂移时管道不会编译的核心承诺。这些工作原理相同
> 跨越两个生态系统。

---

## 第 3 课：上下文是基础设施，而不是文档

来自 OpenAI 的harness engineering帖子：

> 从agent的角度来看，在有效运行时无法在上下文中访问的任何内容都不存在。

系统无法访问 Google 文档、聊天线程或人们头脑中的知识。agent可以看到的全部是存储库本地的、版本化的工件（代码、降价、模式、可执行计划）。

**这一点对 Claude Code 同样成立。** [Claude’s automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) 显示相同的约束：如果上下文不在对话、存储库文件或 MCP 可访问的源中，则它不存在。

```
flowchart LR
    subgraph visible ["What agents CAN see"]
        direction TB
        A["Code + tests"]
        B["Markdown docs ADRs, specs, plans"]
        C["Schemas + configs"]
        D["Exec plans with decision logs"]
        E["Logs, metrics (via tools/MCP)"]
    end
    subgraph invisible ["What agents CANNOT see"]
        direction TB
        F["Slack threads"]
        G["Google Docs"]
        H["People's heads"]
        I["Verbal agreements"]
        J["Undocumented conventions"]
    end
    visible -->|" agent works here "| K["Reliable output"]
    invisible -->|" effectively doesn't exist "| L["Missed constraints wrong assumptions"]
    style visible fill: #d4edda, stroke: #28a745
    style invisible fill: #f8d7da, stroke: #dc3545
```

如果决策不在存储库中，则对于agent来说它不存在。让团队在架构模式上保持一致的 Slack 讨论？就像告诉三个月后加入的新员工一样——除非你写下来，否则他们永远不会知道。

### agent易读性是目标

**OpenAI 的做法：** 由于存储库完全由agent生成，因此它首先针对 **Codex’s legibility** 进行优化。就像团队的目标是提高新工程师的代码可导航性一样，人类工程师的目标是让agent能够推理整个业务领域 **directly from the repository itself**。

**Claude code’s equivalent:** [As software engineering shifts to agent orchestration](https://claude.com/blog/eight-trends-defining-how-software-gets-built-in-2026) ，文件夹和文件结构成为上下文工程的一种形式。 [best practices for Claude Code](https://code.claude.com/docs/en/best-practices) 强调将存储库结构视为agent的主要导航系统。

### 使应用程序本身对agent来说易于理解

**OpenAI’s production setup:**

- **Per-worktree app instances**：应用程序可按 git 工作树启动，因此 Codex 可以在每次更改时启动并驱动一个实例。
- **Chrome DevTools Protocol**：连接到agent运行时，具有处理 DOM 快照、屏幕截图的技能，
和导航。
- **Ephemeral observability per worktree**：通过本地可观察性堆栈公开的日志、指标和跟踪
对于任何给定的工作树都是短暂的。
- **Agents can query logs with LogQL and metrics with PromQL.** 在此上下文可用的情况下，会出现诸如“确保
服务启动在 800 毫秒内完成”变得易于处理。

```
flowchart TD
    A["Agent gets task"] --> B["git worktree created"]
    B --> C["App boots in isolated instance"]
    C --> D["Agent drives app (CDP for Codex, MCP for Claude)"]
    D --> E{"Bug reproduced?"}
    E -->|yes| F["Implement fix"]
    E -->|no| G["Query logs/metrics (LogQL/PromQL for Codex, MCP server for Claude)"]
    G --> D
    F --> H["Validate fix by driving app again"]
    H --> I["Ephemeral observability torn down"]
    style C fill: #d4edda, stroke: #28a745
    style G fill: #fff3cd, stroke: #ffc107
```

**Claude code’s equivalent:**

- **MCP servers for observability**：[Claude Code connects to tools via MCP](https://code.claude.com/docs/en/mcp)
  - 这
模型上下文协议是AI工具集成的开放标准。您无需将可观察性融入 Codex 的运行时，而是通过 MCP 服务器公开它。
- **Example MCP servers** 来自 [Claude Code ecosystem](https://mcpservers.org/claude-skills)
：文件系统访问、Git 操作、数据库查询、HTTP API、浏览器自动化。
- **Per-worktree isolation**：相同的模式适用于克劳德代码。agent创建工作树、启动应用程序、驱动
通过 MCP 连接的工具进行测试、验证、拆卸。
- **Artifacts for state tracking**：[Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)
（最多 20MB）跨会话存储对测试结果、指标历史记录和多会话工作流程有用的结构化数据。

他们经常看到单个 Codex 运行 **upwards of six hours** 的单个任务 - 通常是在人类睡觉时。使用 [Claude Code’s conversation compacting](https://platform.claude.com/docs/en/build-with-claude/compaction) ，长时间运行的会话通过在令牌阈值处自动汇总来保持一致。

### 技术选择有利于智能体的理解

OpenAI 和 Anthropic 团队都喜欢依赖关系和抽象，这些依赖关系和抽象可以完全内化并在回购协议中进行推理。将系统的更多部分拉入agent可以直接检查、验证和修改的表单中，可以使您的输出成倍增加 - 不仅对于 Codex，而且对于在代码库上工作的其他agent（例如 OpenAI 的 [Aardvark](https://openai.com/index/introducing-aardvark/) 或 Anthropic 的 [Claude Code subagents](https://code.claude.com/docs/en/sub-agents)）。

这在实际形式中是相同的心理模型：提高人类可读性和可维护性的选择通常也会改善agent结果。追求汇编级生成而不是有用的抽象并不是加速；而是加速。它正在抛弃我们已经花费多年建立的影响力。

**Tip**

如何自己实现

**Universal patterns (both ecosystems):**

1. **Push decisions from chat to repo.** 每个 Slack 对齐讨论都会成为 ADR。
2. **Create evidence docs, not aspirational docs.** 使用文件路径和行号记录当前状态。
3. **Write unvarnished reviews.** 对差距的残酷诚实。agent商更善于利用​​事实而不是营销。
4. **Define acceptance criteria as measurable checks.** 如果agent无法验证它，则它不是一个标准。
5. **Make the app bootable per worktree.** 每个agent任务都有一个独立的实例。

**For OpenAI codex:**

- 将 Chrome DevTools 协议连接到 Codex 运行时
- 通过每个工作树的短暂可观察性公开日志/指标
- 为agent启用 LogQL/PromQL 查询访问

**For Claude code:**

- 构建[MCP servers](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/)
用于可观察性访问（日志、指标、跟踪）
- 使用[Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)
（最多 20MB）用于存储跨会话的测试结果和指标历史记录
- 启用[automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)
使用 AfterCompaction 钩子来保留关键上下文
- 通过 MCP（Datadog、Grafana、CloudWatch）将 Claude Code 连接到您的可观察性堆栈

> **In flowforge**，我维护 `docs/evidence/unvarnished-review.md` - 引号，如“手卷 codegen JSON 解析器......
> 将在联合、嵌套记录上爆炸” - 以及 v1.0 的 25 个可衡量的验收标准
> `docs/plan/v1.0-readiness.md`。这些对于 Codex 和 Claude Code的工作方式相同，因为它们是简单的降价形式
> 回购协议。

---

## 三个操作原语：技能、shell、压缩

我们正在从单轮助手转变为处理真正知识工作的长期运行agent：读取大型数据集、更新文件和编写应用程序。根据开发人员的反馈和他们自己构建内部agent的经验，OpenAI 和 Anthropic 都发布了三个原语，使长期工作变得实用。

**两个生态系统底层模式一致，区别只在具体实现。**

### 技巧：程序agent按需加载

**OpenAI 的做法：**

技能是一组文件加上一个包含 frontmatter 和说明的 `SKILL.md` 清单。想一想：当需要进行实际工作时，模型可以参考版本化的剧本。技能与agent技能开放标准保持一致。

当技能可用时，平台会将每个技能的 `name`、`description` 和 `path` 公开给模型。该模型使用该元数据来决定是否调用技能。如果是，它会读取 `SKILL.md` 以获取完整的工作流程。

**Claude Code 的做法：**

[Claude code skills](https://code.claude.com/docs/en/skills) 的工作原理相同。技能是一个带有标题（名称、描述、版本）以及分步说明的 Markdown 文件。将技能存储在 `.claude/skills/` 中或通过 [Claude Skills Library](https://mcpservers.org/claude-skills) 安装社区技能。

主要区别：Claude code 的 [skills ecosystem](https://claude.com/blog/skills-explained) 与 MCP 服务器集成以进行外部工具访问，而 OpenAI 的技能则使用 shell 工具来执行。

```
flowchart LR
    subgraph openai ["OpenAI Skills"]
        O1["SKILL.md manifest"] --> O2["Agent reads procedure"]
        O2 --> O3["Executes via shell tool"]
    end
    subgraph claude ["Claude Code Skills"]
        C1[".claude/skills/ manifest"] --> C2["Agent reads procedure"]
        C2 --> C3["Executes via MCP servers"]
    end
    style openai fill: #d4edda, stroke: #28a745
    style claude fill: #e8f4fd, stroke: #0d6efd
```

### Shell：agent的执行

**OpenAI 的做法：**

shell 工具允许模型在真实的终端环境中工作 - 无论是由 OpenAI 管理的托管容器，还是您自己执行的本地 shell 运行时（相同的工具语义，但您控制机器）。托管 shell 通过响应 API 运行，这意味着您的请求带有有状态工作、工具调用、多轮延续和工件。

**Claude Code 的做法：**

[Claude code runs in your local terminal](https://code.claude.com/docs/en/desktop) 默认情况下 - 没有托管容器。你控制机器。 CLI 直接与您的开发环境集成，保持对整个项目的持续了解。

对于远程/云执行，您可以：

- 在 CI/CD 管道中运行 Claude Code CLI
- 使用[Claude Desktop](https://www.producttalk.org/claude-code-what-it-is-and-how-its-different/)
用于基于 GUI 的工作流程
- 通过 SSH + Claude code CLI 连接到远程机器

### 压实：保持长距离运行

**OpenAI 的做法：**

随着工作流程变得更长，它们会遇到上下文窗口限制。服务器端压缩通过管理上下文窗口和自动压缩对话历史记录来保持长时间运行。

两种模式：

- **Server-side compaction**：当上下文超过阈值时，压缩会在流中自动运行。
- **Standalone `/responses/compact` endpoint**：当您想要明确控制压缩何时发生时使用。

**Claude Code 的做法：**

[Automatic conversation compacting](https://platform.claude.com/docs/en/build-with-claude/compaction) 的工作原理相同。当对话接近令牌限制（可配置阈值）时，Claude 会自动总结对话，创建压缩块，并继续压缩上下文。

Claude Code的主要补充：

- **Customize compaction in CLAUDE.md**：“压缩时，始终保留修改文件的完整列表和任何测试
命令”
- **AfterCompaction hooks**：总结后注入关键上下文（积极计划、黄金原则）
- \*\*文物不计入代币限制
\*\*：[Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)（最多 20MB）与对话上下文分开存储代码和输出
- **Manual trigger**: `/compact Focus on the API changes` 用于显式控制

Claude code 的 [automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) 继续改进长会话中的内存使用和关键状态的保存。

### 为什么他们在一起会更好

```
flowchart LR
    subgraph skills ["Skills = the HOW"]
        S1["SKILL.md manifest"]
        S2["Templates + examples"]
        S3["Guardrails + routing"]
    end
    subgraph shell ["Shell/MCP = the DO"]
        SH1["Install dependencies"]
        SH2["Run scripts + tools"]
        SH3["Write artifacts"]
    end
    subgraph compaction ["Compaction = the CONTINUITY"]
        C1["Auto-compress when context fills"]
        C2["Pin immutable constraints"]
        C3["Multi-hour runs stay coherent"]
    end
    skills -->|" model loads procedure "| shell
    shell -->|" long run hits limit "| compaction
    compaction -->|" resumes with full context "| shell
    style skills fill: #e8f4fd, stroke: #0d6efd
    style shell fill: #d4edda, stroke: #28a745
    style compaction fill: #fff3cd, stroke: #ffc107
```

- 技能通过将稳定的过程和示例移动到可重用的包中来减少即时意大利面条。
- Shell/MCP 提供完整的执行环境，让您安装代码、运行脚本和编写输出。
- 压缩可以在长期运行中保持连续性，因此相同的工作流程可以继续执行，而无需手动进行上下文操作。

**The philosophy, summarized:** 使用技能对 *how*（程序、模板、护栏）进行编码。使用 shell/MCP 执行 *do*（安装、运行、写入工件）。使用压缩来保持长时间运行的连贯性（无需手动管理上下文）。

### 真正重要的 10 个技巧

这些内容来自 OpenAI 的开发者博客，来自他们构建 Codex 的工作和 Glean 的生产经验。下面的每个技巧都展示了 OpenAI 实施模式和 Claude Code 的不同之处。

**Applicability to Claude Code:** 技巧 1-5（技能设计）和技巧 10（本地/云奇偶校验）直接适用于相同的模式。技巧 6-9（安全、网络、凭证）使用不同的机制（MCP 权限层与 OpenAI 允许列表），但原则相同。

**Tips 1-5: skill design and routing**

**Tip 1: Write skill descriptions like routing logic, not marketing copy.**

您的技能描述实际上是模型的决策边界。它应该回答：我什么时候应该使用这个？我什么时候应该*not*？产出和成功标准是什么？

**OpenAI + Claude:** 直接在前面的内容描述中包含一个简短的“何时使用与何时不使用”块。

---

**Tip 2: Add negative examples and edge cases to reduce misfires.**

一个令人惊讶的失败模式是，使技能可用最初可以*reduce*正确触发。 Glean 在有针对性的评估中看到了基于技能的路由 **drop by about 20%**，然后在添加负面示例和边缘情况覆盖后恢复。

**OpenAI + Claude:** 在技能正文中明确写出“当……时不要调用此技能”的情况，并建议该怎么做。

---

**Tip 3: Put templates and examples inside the skill, not the system prompt.**

技能内部的模板有两个优点：它们在需要时（调用技能时）可用，并且不会为不相关的查询增加令牌。 Glean 报告称，这种模式推动了他们的一些 **biggest quality and latency gains** 投入生产。

**OpenAI:** 在“示例”部分下的 `SKILL.md` 中包含模板。 **Claude:** 在 `.claude/skills/*.md` 中包含模板，并内联具体示例。

---

**Tip 4: Design for long runs early with container reuse and compaction.**

目光长远的特工很少能一次性成功。从一开始就计划连续性：

- 当您需要稳定的依赖项、缓存文件和中间文件时，跨步骤重复使用相同的容器/会话
输出。
- 传递 `previous_response_id` (OpenAI) 或维护对话线程 (Claude)，以便模型可以继续工作。
- 使用压缩作为默认的长期运行原语，而不是紧急后备。

**Claude-specific:** 使用 [artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)（最多 20MB）跨压缩事件持久保存结构化数据。

---

**Tip 5: For determinism, just tell the model to use the skill.**

“使用`&lt;skill-name&gt;`技能。”就是这样。您可以拉动的最简单的可靠性杠杆。将模糊路由转变为显式契约。

**OpenAI + Claude:** 在两个生态系统中的工作方式相同。

**Tips 6-10: security, networking, and portability**

**Tip 6: Treat skills plus networking as a high-risk combo.**

**Security posture**

**Combining skills with open network access creates a high-risk path for data exfiltration.** 如果您使用网络，请严格遵守网络白名单，假设工具输出不受信任，并避免在面向消费者的流程中使用开放互联网和不受限制的程序。

**OpenAI:** 使用组织级别和请求级别 `network_policy` 允许列表。 **Claude:** 使用 [MCP server vetting](https://www.mintmcp.com/blog/claude-code-security) 和权限级别（拒绝/询问/允许规则）。

两者的强默认姿势：

- 技能：**allowed**
- 壳牌/MCP：**allowed**
- 网络：**enabled only with a minimal allowlist**，每个请求，适用于范围狭窄的任务

---

**Tip 7: Make a standard handoff boundary for artifacts.**

**OpenAI:** 将 `/mnt/data` 视为写入输出的标准位置，您将检索、查看或传递回后续步骤。

**Claude:** 使用 [Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) 获得跨会话持续的结构化输出。对于基于文件的输出，定义一个标准路径，如 CLAUDE.md 中的 `./artifacts/`。

心智模型：工具写入磁盘，模型通过磁盘推理，开发人员从磁盘检索。

---

**Tip 8: Understand allowlists as a two-layer system.**

**OpenAI:** 网络通过组织级别白名单（管理员配置）和请求级别 `network_policy`（必须是组织白名单的子集）进行控制。

**Claude:** [MCP security uses permission tiers](https://www.mintmcp.com/blog/claude-code-security) ：拒绝规则阻止操作，询问规则需要用户批准，允许规则自动批准。挂钩控件允许您禁用会话之间的所有挂钩，以防止持久性恶意代码。

```
flowchart TD
    A["Agent request with network/tool access"] --> B{"In allowlist (OpenAI) or permission tier(Claude)?"}
    B -->|no| C["Blocked"]
    B -->|yes| D{"Needs auth?"}
    D -->|no| E["Request proceeds"]
    D -->|yes| F["Inject credentials (domain_secrets for OpenAI, MCP auth for Claude)"]
    F --> E
    style C fill: #f8d7da, stroke: #dc3545
    style E fill: #d4edda, stroke: #28a745
```

---

**Tip 9: Use secure credential injection for authenticated calls.**

**OpenAI:** 使用 `domain_secrets`，以便模型永远不会看到原始凭证。在运行时，模型会看到占位符（例如 `$API_KEY`），并且 sidecar 仅针对批准的目的地注入真实值。

**Claude:** [MCP servers handle authentication](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/) 与模型分开。凭证存在于 MC​​P 服务器配置中，而不存在于对话上下文中。

---

**Tip 10: Use the same APIs in the cloud and locally.**

**OpenAI:** 技能适用于托管 shell 和本地 shell 模式。 Shell 有一个本地执行模式，您可以自己执行 `shell_call` 并将 `shell_call_output` 返回给模型。

**Claude:** [Claude code runs locally by default](https://code.claude.com/docs/en/desktop) 。对于云执行，请在 CI/CD 中运行 Claude Code CLI 或使用 Claude Desktop 进行 GUI 工作流程。技能和 MCP 服务器在本地/远程的工作方式相同。

实用的开发循环（两个生态系统）：

1. 本地启动（快速迭代、访问内部工具、轻松调试）。
2. 当您需要可重复性、隔离性和部署一致性时，请转向托管/CI。
3. 在两种模式下保持技能相同（即使执行移动，工作流程也保持稳定）。

### 三种构建模式

```
flowchart LR
    subgraph A ["Pattern A: Artifact"]
        direction TB
        A1["Install libs"] --> A2["Fetch data / call API"] --> A3["Write to standard path (/mnt/data or ./artifacts/)"]
    end
    subgraph B ["Pattern B: Repeatable"]
        direction TB
        B1["Load skill"] --> B2["Execute in shell/MCP"] --> B3["Produce artifact deterministically"]
    end
    subgraph C ["Pattern C: Enterprise"]
        direction TB
        C1["Skill = living SOP"] --> C2["Multi-tool orchestration"] --> C3["Consistent execution across org"]
    end
    A -->|" add skills for consistency "| B
    B -->|" scale across teams "| C
    style A fill: #d4edda, stroke: #28a745
    style B fill: #e8f4fd, stroke: #0d6efd
    style C fill: #fff3cd, stroke: #ffc107
```

每个模式都建立在前一个模式之上。从 A 开始以获得快速成功，逐渐升级到 B 以实现可重复的工作流程，在整个组织中扩展到 C。

**Pattern A: Install, fetch, write artifact.**

**OpenAI:** 安装库，调用 API，将报告写入 `/mnt/data/report.md`。 **Claude:** 安装库，通过 MCP 服务器调用 API，写入 Claude 工件或 `./artifacts/report.md`。

这创建了一个干净的审查边界 - 您的应用程序可以向用户显示工件、记录它、比较它或将其提供给后续步骤。

**Pattern B: Skills + shell/MCP for repeatable workflows.**

**OpenAI:** 在 `SKILL.md` 中编码工作流程，挂载到 shell 环境中，确定性地执行。 **Claude:** 在 `.claude/skills/*.md` 中对工作流程进行编码，通过 MCP 服务器执行以进行外部工具访问。

对于电子表格分析、数据集清理和标准化报告生成特别有效。

**Pattern C: Skills as enterprise workflow carriers.**

一种早期模式是单一工具调用和多工具编排之间的差距失去了准确性。技能缩小了这一差距。

Glean 面向 Salesforce 的技能（适用于 OpenAI 和 Claude）：从 **73% to 85%** 提高评估准确性并降低 **time-to-first-token by 18.1%**。实用策略：仔细路由、反面例子、在技巧中嵌入模板。

技能成为活生生的 SOP（标准操作程序）：随着组织的发展而更新，并由两个生态系统中的agent一致执行。

---

## 架构执行：边界，而不是微观管理

agent在 [strict boundaries and predictable structure](https://bits.logic.inc/p/ai-is-forcing-us-to-write-good-code) 环境中最有效。 OpenAI 围绕严格的架构模型构建了他们的应用程序。每个业务域都分为一组固定的层，并具有严格验证的依赖关系方向。

**Claude Code teams are doing the same thing.** [As agentic coding becomes standard in 2026](https://www.teamday.ai/blog/complete-guide-agentic-coding-2026)，共识很明确：**constraints enable speed without decay**。

**The rule (enforced mechanically):**

在每个业务域（例如，应用程序设置）内，代码只能通过一组固定的层“转发”。横切关注点（身份验证、连接器、遥测、功能标志）通过单个显式接口进入：\* *Providers*\*。其他任何事情都是不允许的，并且是机械强制执行的。

```
flowchart LR
    subgraph domain ["Business domain (e.g. App Settings)"]
        direction LR
        T["Types"] --> CF["Config"] --> R["Repo"] --> S["Service"] --> RT["Runtime"] --> U["UI"]
    end
    subgraph providers ["Providers (single entry point)"]
        direction TB
        P1["Auth"]
        P2["Connectors"]
        P3["Telemetry"]
        P4["Feature flags"]
    end
    providers -->|" only allowed cross-cutting edge "| S
    T -.->|" backward dependency BLOCKED by linter "| U
    style domain fill: #e8f4fd, stroke: #0d6efd
    style providers fill: #fff3cd, stroke: #ffc107
    style T fill: #d4edda, stroke: #28a745
```

依赖关系仅从左到右流动。 linter 会阻塞任何后向边缘。提供者是跨领域关注点进入的唯一途径。

**Info**

为什么这很重要？

这种架构通常会被推迟，直到拥有数百名工程师为止。对于编码agent，这是一个早期的先决条件：**the constraints are what allows speed without decay or architectural drift**。

### 通过自定义 lint 和结构测试来执行

**OpenAI:** 自定义 linter 和结构测试（Codex 生成），以及“味道不变量”。它们静态地强制执行结构化日志记录、命名约定、文件大小限制。由于 lint 是自定义的，因此它们会写入错误消息，将修复指令注入agent上下文中。

**Claude code:** [Hooks provide deterministic enforcement](https://github.com/affaan-m/everything-claude-code) 。 PostToolUse 挂钩在代码更改后立即运行 linter。 PreToolUse 挂钩会在违规发生之前阻止它们。生产仓库示例：

|  |  |
| --- | --- |
| ```  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 ``` | ``` #!/bin/bash # .claude/hooks/PostToolUse.sh  # Enforce layered architecture if [[ "$FILE_PATH" =~ ^src/(.+)/types/ ]]; then   LAYER="types" elif [[ "$FILE_PATH" =~ ^src/(.+)/service/ ]]; then   LAYER="service" fi  if [[ -n "$LAYER" ]]; then   # Check for backward dependencies (service importing from UI, etc.)   if grep -q "from.*UI" "$FILE_PATH" && [[ "$LAYER" == "service" ]]; then     echo "❌ Blocked: Service layer cannot import from UI layer"     echo "Fix: Move shared logic to Types or create a Provider"     exit 2   fi fi  # Run language-specific linter npx eslint --fix "$FILE_PATH" exit 0 ``` |

**The philosophy**：集中执行边界，允许地方自治。您非常关心边界、正确性和可重复性。在这些界限内，您允许团队或agent在如何表达解决方案方面拥有很大的自由。

生成的代码并不总是符合人类的风格偏好，但这没关系。只要输出是正确的、可维护的并且对于未来的agent运行来说是清晰的，它就满足标准。

> **In FlowForge**，我通过 sbt 模块定义强制执行这一点（依赖项仅向一个方向流动），
> `.scalafix.conf` 规则（无 `var`、无 `asInstanceOf`、无 `println`）和每个模块覆盖阈值（核心 90%、
> 连接器 80%）。这些规则对于 Codex 和 Claude Code 的作用相同，因为它们是语言级别的强制执行，
> 不是特定于agent的。

---

## 自治阶梯：从起草代码到合并 PR

随着更多的开发循环被直接编码到系统中 - 测试、验证、审查、反馈处理和恢复 - OpenAI 的存储库跨越了一个有意义的阈值，Codex 可以端到端地驱动新功能。

**Claude Code teams are reaching the same milestone.** [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) 显示类似的自主工作流程已准备好投入生产。

给定一个提示，agent现在可以端到端地驱动一个功能：

```
flowchart TD
    A["1. Validate codebase state"] --> B["2. Reproduce reported bug"]
    B --> C["3. Record evidence\n(video for Codex, artifact for Claude)"]
    C --> D["4. Implement fix"]
    D --> E["5. Validate fix by driving app"]
    E --> F["6. Record verification (video/artifact)"]
    F --> G["7. Open PR"]
    G --> H["8. Respond to agent + human feedback"]
    H --> I{"9. Build passes?"}
    I -->|no| J["Remediate build failure"]
    J --> H
    I -->|yes| K{"10. Judgment required?"}
    K -->|yes| L["Escalate to human"]
    K -->|no| M["11. Merge"]
    style A fill: #e8f4fd, stroke: #0d6efd
    style M fill: #d4edda, stroke: #28a745
    style L fill: #fff3cd, stroke: #ffc107
```

单个提示即可完成 11 个步骤。agent循环反馈，直到审阅者满意为止（ [Ralph Wiggum Loop](https://ghuntley.com/loop/)
- agent到agent审核迭代）。人类只会在实际发生时介入
需要判断。

**Implementation differences:**

**OpenAI codex:**

- 使用 Chrome DevTools 协议进行步骤 2（重现错误）和步骤 5（验证修复）
- 录制视频作为证据
- 在本地审核自己的更改，请求agent在云端审核
- 使用 `gh` CLI 进行 PR 操作

**Claude code:**

- 使用 [MCP servers for app interaction](https://code.claude.com/docs/en/mcp)
（浏览器自动化、API 测试）
- 记录证据
在[Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)中（结构化测试结果，屏幕截图）
- [Subagents handle specialized review](https://code.claude.com/docs/en/sub-agents)
（安全性、测试覆盖率、性能）
- 使用 `gh` CLI 进行 PR 操作（与 Codex 相同）

**Note**

警告

这种行为在很大程度上取决于存储库的具体结构和工具，如果没有类似的投资，则不应假设可以概括 - 至少现在还不行。 OpenAI 和 Anthropic 团队花费了数月时间构建线束基础设施。

### agent如何与系统交互

**OpenAI:** 人类几乎完全通过提示进行交互：描述任务、运行agent、允许它打开拉取请求。为了推动 PR 完成，Codex 在本地审核自己的更改，请求额外的agent审核，响应反馈，并进行迭代，直到所有审核者都满意为止。 Codex 直接使用标准开发工具（`gh`、本地脚本、存储库嵌入技能）。

**Claude code:** [The workflow is iterative and conversational](https://medium.com/@billcockerill/coding-with-claude-code-and-claude-desktop-two-ai-workflows-compared-2c70dac52b47) ，感觉就像你旁边坐着一个初级开发人员。 Claude Code 直接与您的开发环境集成，保持对整个项目的持久了解。为了进行审查，[specialized subagents](https://code.claude.com/docs/en/sub-agents) 处理安全扫描、测试覆盖率和代码质量检查。

**Human review is optional, not required.** 随着时间的推移，两个团队都将几乎所有审核工作推向了agent对agent的处理。

### 吞吐量改变了合并理念

As agent throughput increased, many conventional engineering norms became counterproductive. OpenAI 和 Claude Code 团队都使用 **minimal blocking merge gates** 进行操作。拉取请求是短暂的。测试薄片通常通过后续运行来解决，而不是无限期地阻止进度。

在agent吞吐量远远超过人类注意力的系统中，**corrections are cheap, and waiting is expensive**。

这在低吞吐量环境中是不负责任的。在这里，这通常是正确的权衡。

### “agent生成”的实际含义是什么

当他们说代码库是由agent生成时，他们意味着一切：产品代码和测试、CI 配置和发布工具、内部开发人员工具、文档和设计历史、评估工具、审查评论和响应、管理存储库本身的脚本以及生产仪表板定义文件。

人类始终保持在循环中，但在不同的抽象层工作。他们确定工作的优先顺序，将用户反馈转化为验收标准，并验证结果。当agent陷入困境时，他们将其视为一个信号：识别缺少的内容 - 工具、护栏、文档 - 并将其反馈到存储库中，**always by having the agent itself write the fix**。

---

## 构建 ChatGPT Apps的 15 个经验教训（适用于 Claude 生态系统）

Alpic 在三个月内开发了两打 ChatGPT Apps。他们的核心见解是他们所说的**the three body problem**：传统的网络应用程序有两个参与者（用户、UI），但AI应用程序添加了第三个参与者（模型）。困难的部分是管理**context asymmetry**——每个人都有部分知识，没有人能了解全部情况。

**These patterns apply to [Claude Desktop apps and MCP servers](https://www.arsturn.com/blog/claude-desktop-vs-claude-code-should-you-switch-for-mcp-features) .** 无论您是为 ChatGPT 还是 Claude 构建，三体问题都存在。

```
flowchart TD
    subgraph traditional ["Traditional web app"]
        direction LR
        U1["User"] <-->|" clicks, sees "| UI1["UI"]
    end
    subgraph ai ["AI app (ChatGPT or Claude)"]
        direction TB
        U2["User"] <-->|" types, sees "| UI2["Widget/UI/Desktop"]
        UI2 <-->|" tools/MCP "| M["Model"]
        M <-->|" tool results "| U2
    end
    style traditional fill: #f0f0f0, stroke: #999
    style ai fill: #e8f4fd, stroke: #0d6efd
```

三个演员，每个人都有部分知识。用户看到聊天和 UI。 UI 可以看到自己的状态并可以将上下文推送到模型。该模型可以看到工具输出，但看不到 UI 内部结构，除非您明确公开它们。管理谁知道什么——这就是整个游戏。

**Lessons 1-4: system and architecture (ChatGPT + Claude adaptations)**

**Lesson 1: Not all context should be shared.**

不同的部分需要有意识地采用不同的国家观点。在谋杀之谜游戏中，模型需要知道凶手是谁才能正确进行角色扮演，而 UI 和用户则不需要。在“Time’s Up”游戏中，情况正好相反：用户界面显示秘密单词，模型必须保持不知情。

**ChatGPT:** 使用 `structuredContent` 获取模型和小部件所需的数据。使用 `_meta` 来获取仅对小部件可见、对模型隐藏的响应元数据。

**Claude:** 使用 [MCP resources](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/) 仅公开模型需要的内容。使用 [Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) 作为对模型隐藏的客户端状态。

---

**Lesson 2: Lazy-loading doesn’t translate well to AI apps.**

在AI应用程序中，工具调用意味着延迟——由于安全沙箱和模型推理，通常会延迟几秒钟。积极地提前加载：将尽可能多的数据发送到初始工具响应中。

**ChatGPT:** 通过 `window.openai.toolOutput` 使小部件水合。如果小部件可以安全地从公共 API 获取而不与模型共享信息，则经典的 XHR 调用就可以工作。

**Claude:** 使用 [MCP servers for data fetching](https://code.claude.com/docs/en/mcp) 。将数据预加载到 [artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) 或 MCP 资源中。对于公共 API，直接 HTTP 即可工作（模型看不到内部结构）。

---

**Lesson 3: The model needs visibility.**

当用户与小部件交互（选择产品）然后在聊天中提出问题时，模型不知道他们指的是什么。

**ChatGPT:** 使用 `window.openai.setWidgetState(state)` 进行命令式更新。将 `data-llm` 属性直接附加到组件以获取声明性上下文。 Alpic 构建了一个 Vite 插件，可以抓取这些属性并自动更新 widgetState。

**Claude:** 对于 Claude Desktop，通过 [MCP tools](https://code.claude.com/docs/en/mcp) 推送状态更新。对于 Claude Code，在跨会话持续存在的 [artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)（最多 20MB）或 `.claude/context/` 文件中维护状态。

---

**Lesson 4: Different interactions require different APIs.**

小部件到服务器、模型到服务器、小部件到模型——每条路径的存在都是为了支持不同类型的交互。使这些沟通路径变得明确，并有意选择哪个机制处理体验的哪一部分。

**ChatGPT + Claude:** 原理相同。定义清晰的边界：什么是 HTTP（小部件 ↔ 服务器）、什么是工具调用（模型 ↔ 服务器）、什么是状态更新（小部件 → 通过 MCP 或 widgetState 的模型）。

**Lessons 5-8: UI and product behavior (ChatGPT + Claude adaptations)**

**Lesson 5: UI must adapt to multiple display modes.**

**ChatGPT:** 应用程序可以内联显示（保留在对话历史记录中）、全屏显示（占据整个屏幕，聊天栏位于底部）或画中画显示（浮动在顶部）。考虑设备特定的安全区域。

**Claude Desktop:** 存在类似的模式。 [Claude Desktop’s visual diffs](https://medium.com/@billcockerill/coding-with-claude-code-and-claude-desktop-two-ai-workflows-compared-2c70dac52b47) 显示并排比较。工件可以内嵌显示或在单独的窗格中显示。为两者而设计。

**Claude code CLI:** 无 UI 模式（仅限终端），但可以在外部查看器中打开工件。设计文本优先的演示。

---

**Lesson 6: UI consistency matters in an embedded environment.**

**ChatGPT:** 使用 [OpenAI Apps SDK UI Kit](https://github.com/openai/apps-sdk-ui) 来获取与 ChatGPT 设计系统一致的即用型组件。

**Claude Desktop:** 尚无官方 UI 工具包，但 [MCP-connected apps](https://mcpservers.org/claude-skills) 应遵循系统 UI 约定（匹配 macOS/Windows 设计语言）。尽可能使用本机组件。

---

**Lesson 7: Language-first filtering beats traditional UI controls.**

当用户可以说“欧洲阳光明媚的目的地，价格不到 200 美元”时，强迫他们通过复选框和范围滑块会增加摩擦。为模型提供工具参数的 **List of Values (LOV)**，以便将自然语言直接映射到后端 API 要求。

**ChatGPT + Claude:** 相同的模式。在工具/MCP 模式中定义 LOV。让模型做自然语言→结构化参数映射。

---

**Lesson 8: Files unlock richer interactions.**

用户上传产品照片，模型对其进行识别，小部件继续进行产品匹配和发现。

**ChatGPT:** 工具通过 `openai/fileParams` 使用文件。小部件通过 `window.openai.uploadFile` 和 `window.openai.getFileDownloadUrl` 处理文件。

**Claude:** [Claude Desktop handles files natively](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) 。 MCP 服务器可以公开文件上传端点。 [Artifacts store up to 20MB](https://amitkoth.com/claude-artifacts-guide/) 包括图像、PDF 和结构化数据。

**Lessons 9-10: production readiness (ChatGPT + Claude adaptations)**

**Lesson 9: CSPs are the new CORS.**

**ChatGPT:** OpenAI 在双嵌套 iframe 内渲染应用程序。内容安全政策得到严格执行。在应用清单中声明 `connectDomains`、`resourceDomains`、`frameDomains`、`redirectDomains`。

**Claude Desktop:** [MCP servers run with permission tiers](https://www.mintmcp.com/blog/claude-code-security) 。在 MCP 清单中定义允许的操作。拒绝规则阻止危险操作，询问规则需要用户批准，允许规则自动批准。

安全考虑：

- ChatGPT：CSP 违规 = 阻止 iframe 操作
- Claude：MCP 权限违规 = 阻止工具调用

两者都需要明确的外部域/操作白名单。

---

**Lesson 10: Small flags have outsized impact.**

提交时需要 **ChatGPT:** `widgetDomain`。 `widgetAccessible` 控制 widget 是否可以通过 `callTool` 自行调用工具。需要工具注释（`readOnly`、`destructiveHint`、`openWorldHint`）。

**Claude:** [MCP tool definitions](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/) 需要类似的元数据：操作安全级别、所需参数、身份验证要求。 [Hook configurations](https://aiorg.dev/blog/claude-code-hooks) 定义哪些生命周期事件触发哪些自动化。

**Lessons 11-14: iteration velocity (ChatGPT + Claude adaptations)**

**Lesson 11: Fast iteration requires hot reload.**

**ChatGPT:** 长 TTL 资源缓存使标准 HMR 不兼容。 Alpic 构建了一个 Vite 插件，可以拦截资源请求并将实时更新注入 ChatGPT iframe 中。

**Claude Desktop:** [MCP servers support hot reload](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/) 通过服务器重新启动。使用文件观察器在更改时自动重新加载 MCP 配置。

**Claude code CLI:** 支持实时文件观看。对 `.claude/` 目录（技能、挂钩、agent）的更改立即生效或在下一个会话开始时生效。

---

**Lesson 12: Not every test belongs in production environment.**

**ChatGPT:** 构建一个轻量级本地模拟器来模拟 ChatGPT 主机环境。保留真实的 ChatGPT 测试来验证模型交互。

**Claude:** [Claude Code runs locally by default](https://code.claude.com/docs/en/desktop)
- 你的开发环境是
测试环境。使用 [MCP server mocks](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/) 进行集成测试。为端到端工作流程保留 Claude Desktop 测试。

---

**Lesson 13: Mobile testing requires explicit support.**

**ChatGPT:** Vite 的默认本地主机使得其他设备无法访问隧道 URL。通过隧道端口上的域转发进行扩展。

**Claude Desktop:** 桌面应用程序仅适用于 macOS/Windows（无移动设备）。 [Claude mobile apps](https://www.anthropic.com/news/claude-3-5-sonnet) 存在，但重点关注会话用例。设计以桌面优先、移动网络作为后备。

---

**Lesson 14: Familiar abstractions speed delivery.**

**ChatGPT:** Apps SDK 公开低级 JavaScript API。引入 React 友好的抽象 - 像 `useCallTool`、`useWidgetState`、`useLocale` 这样的钩子。

**Claude:** [MCP SDKs exist for Python and TypeScript](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/) 。围绕常见模式（身份验证、缓存、速率限制）构建更高级别的抽象。跨项目重用。

### 编码和重用（第 15 课）

**Lesson 15: Turn lessons into reusable tooling.**

**ChatGPT ecosystem:** Alpic 创建了 [Skybridge Framework](https://github.com/alpic-ai/skybridge)（带有钩子、开发工具、`data-llm` 属性的开源 React 框架）和覆盖整个生命周期的 [Codex Skill](https://github.com/alpic-ai/skybridge/tree/main/skills/chatgpt-app-builder)。

**Claude ecosystem:** [Community is building similar patterns](https://github.com/hesreallyhim/awesome-claude-code) 。来自 [everything-claude-code](https://github.com/affaan-m/everything-claude-code) 的经过生产测试的配置，来自 [claude-code-showcase](https://github.com/ChrisWiles/claude-code-showcase) 的综合示例。 [Claude Skills Library](https://mcpservers.org/claude-skills) 拥有 50 多个即用型 MCP 服务器和技能。

**Note**

图案

这就是应用于应用程序开发的harness engineering。不要不断地重新发现相同的问题 - 将它们编码到可重用的技能、MCP 服务器和挂钩中。在整个生态系统中共享。

---

## 您的运营模式：员工与校长的分割

**Operating principle**：工程师应该优化**agentic system**（指令+工具+检查+上下文），而不仅仅是直接的代码工件。

**This is tool-agnostic.** 无论您使用 Codex 还是 Claude Code，分割都保持不变。

| 等级 | 主要所有权 | 成功信号 | 故障信号 |
| --- | --- | --- | --- |
| 主管工程师 | 团队线束实施 | 对于团队工作流程来说可靠的agent任务 | 重复手动清理，输出不一致 |
| 总工程师 | 组织范围的安全带标准 | 跨团队重用、更少的回归、更快的入职 | 碎片化的约定，特定于模型的漂移 |

**Software engineering track**

- 架构图（AGENTS.md 或 CLAUDE.md + 深层文档）
- 类型合同和政策检查
- CI 集成漂移扫描仪（或 Claude 的钩子）
- 工具痕迹和修复循环

**Data engineering track**

- 数据契约和模式策略（编译时或 CI 时）
- 确定性转换操作手册（纯度执行）
- 线束中的数据质量和沿袭检查
- 针对失败管道和迟到数据的事件手册

---

## 本季度可以实施的蓝图

```
gantt
    title Harness implementation timeline
    dateFormat YYYY-MM-DD
    axisFormat %b W%W
    section Foundation
        Instruction architecture: a1, 2026-03-02, 2w
        Quality enforcement: a2, after a1, 2w
    section Execution
        Skills and procedures: a3, after a2, 2w
        Shell/MCP and execution: a4, after a3, 2w
    section Lifecycle
        Compaction and context: a5, after a4, 4w
```

五层，每层都在最后一层。如果您有团队，您可以并行运行第 1-2 层。

### 第 1 层：指令架构（第 1-2 周）

**For OpenAI codex:**

- 将 `AGENTS.md` 作为索引发送（120 行以下）
- 按关注点构建 `docs/` 目录
- 在 `docs/decisions/` 中创建 ADR 日志

**For Claude code:**

- 在存储库根目录中创建 `CLAUDE.md`（50-120 行，仅限项目特定）
- 专门子agent的结构 `.claude/agents/`
- 从 CLAUDE.md 链接到 `docs/architecture/` 以进行深入研究

**Universal:**

- 使用文件路径和行号定义证据文档
- 对存储库中的所有决策和计划进行版本控制

### 第 2 层：质量执行（第 3-4 周）

**For OpenAI codex:**

- 定义黄金原则（5-10 条规则）
- 部署每日漂移扫描
- 使 lint 错误成为规范性的

**For claude code:**

- 定义 `.claude/hooks/PostToolUse.sh` 进行自动修复
- 定义 `.claude/hooks/PreToolUse.sh` 以阻止违规
- 跟踪挂钩触发率作为指标

**Universal:**

- 自动修复低风险违规行为
- 跟踪手动清理时间（目标：趋向于零）

### 第 3 层：可执行程序/技能（第 5-6 周）

**For OpenAI Codex:**

- 将前 3-5 个工作流程转换为 `SKILL.md` 文件
- 添加模板、示例、反例
- 存储在 `skills/` 目录中并进行版本控制

**For Claude Code:**

- 将工作流程转换为 `.claude/skills/*.md` 文件
- 包括路由逻辑（何时使用、何时不使用）
- 从 [Claude skills library](https://mcpservers.org/claude-skills) 安装社区技能

**Universal:**

- 编写技能描述，例如路由逻辑，而不是营销文案
- 包括负面例子以减少失火
- 测试技能调用准确性（目标>= 90%）

### 第 4 层：执行基板/外壳或 MCP（第 7-8 周）

**For OpenAI Codex:**

- 启用 shell 支持的执行（托管或本地）
- 定义标准工件路径 (`/mnt/data`)
- 仪器轨迹和事件指标

**For Claude Code:**

- 构建[MCP servers for external tools](https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/)
- 定义标准工件路径（Claude 工件或 `./artifacts/`）
- 通过 MCP 连接可观测性堆栈（日志、指标、跟踪）

**Universal:**

- 从本地模式开始（快速迭代）
- 当准备好重复性时转移到托管/CI
- 保持本地/远程技能相同

### 第 5 层：上下文生命周期/压缩（正在进行）

**For OpenAI Codex:**

- 启用服务器端压缩（自动或手动`/responses/compact`）
- 通过 `previous_response_id` 实现多圈连续性
- 在系统消息中固定不可变约束

**For Claude Code:**

- 启用[automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)
带有自定义 CLAUDE.md 说明
- 总结后使用 `.claude/hooks/AfterCompaction.sh` 注入关键上下文
- 存储长期状态
在 [Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) 中（最多 20MB）

**Universal:**

- 定期实施环境卫生
- 跟踪上下文命中率（目标 >= 90%）
- 测量长期一致性（目标：6 小时以上的会话，无漂移）

---

## 真正重要的指标

转变：从测量编写的代码行数到测量利用效率。

| 公制 | 目标 | 它告诉你什么 |
| --- | --- | --- |
| agent任务成功率（成功运行次数/总数） | >= 85% | 线束规格明确 |
| 自动修复率（自动修复/问题总数） | >= 60% | 质量执行是机械的 |
| 上下文命中率（已解决的查找/总数） | >= 90% | 文档结构工作 |
| 手动清理时间（每周小时数） | 趋势为零 | 黄金原则覆盖范围 |
| 技能调用准确率 | >= 90% | 描述+反例有效 |
| 人为升级率 | < 15% | 界限清晰 |

**Leading indicators**（首先测量）：上下文命中率、技能调用准确性、手动清理时间。

**Lagging indicators**（随着工具的成熟而改进）：agent任务成功率、自动修复率、人工升级率。

**How to measure across ecosystems:**

**OpenAI codex:**

- 任务成功：无需人工干预的 PR 合并率
- 自动修复：漂移扫描仪自动修复率
- 上下文点击：AGENTS.md 链接点击（文档中的工具）

**Claude code:**

- 任务成功：`/stats` 命令显示会话指标
- 自动修复：挂钩自动修复与阻止率
- 上下文点击：跟踪 `.claude/agents/` 调用频率

**How OpenAI measures effectiveness**：不是按体积（1M 行令人印象深刻，但不是重点）。按速度（3.5 PR/工程师/天，不断增加）。通过自主（端到端，无需人工干预）。通过放大（人类比实现高一层）。

---

## 当出现问题时：故障场景和修复

**Agent output quality regresses suddenly**

**Symptoms:** PR 质量下降，测试失败更频繁，手动清理时间增加。

**Check in order:**

1. 说明文件（AGENTS.md 或 CLAUDE.md）是否变得太大或矛盾？
2. 未经测试就更改了技能版本吗？
3. 压缩消除了关键限制吗？
4. 评价门槛是否悄然发生了变化？

**Fix (OpenAI):** 审核 AGENTS.md 长度和交叉链接。回滚技能版本。在系统消息中添加压缩保存规则。

**Fix (Claude):** 检查 CLAUDE.md 长度（应为 50-120 行）。回顾 `.claude/hooks/AfterCompaction.sh` - 它是否注入了关键上下文？向 CLAUDE.md 添加保存规则。

**Universal:** 在执行计划中记录结果，让agent实施修复，更新黄金原则。

**Agent keeps missing team conventions**

**Symptoms:** 重复的风格违规，不一致的模式，需要手动更正。

**Root cause:** 知识存在于人们的头脑或聊天线程中。

**Fix (OpenAI):** 将约定移至 `docs/conventions.md`，从 AGENTS.md 链接，添加机器检查（自定义 linter）。

**Fix (Claude):** 将 `.claude/hooks/PostToolUse.sh` 中的约定编码为自动修复。向 CLAUDE.md 添加高级指导。使用 PreToolUse 挂钩来阻止严重违规。

**Universal:** 如果agent不断出错，则该约定不在存储库中。使其机械化。

**Long workflows collapse after many turns**

**Symptoms:** agent在任务中丢失上下文，忘记之前的决定，在相同的问题上循环。

**Root cause:** 压缩删除了关键上下文，或未启用压缩（达到硬限制）。

**Fix (OpenAI):** 确保服务器端压缩处于活动状态。在系统消息中固定不可变的约束。将长目标拆分为具有明确切换边界的检查点子运行。通过 `previous_response_id` 继续。

**Fix (Claude):** 在 API 设置中启用 [automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)。向 CLAUDE.md 添加压缩保留规则：“压缩时，始终保留已修改文件和任何测试命令的完整列表。”使用`.claude/hooks/AfterCompaction.sh`注入临界状态。将长期上下文存储在 [Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) 中（最多 20MB）。

**Universal:** 将压缩事件作为指标进行跟踪。测量压缩前/压缩后的一致性。

**Humans spend 20% of time cleaning up AI slop**

**Symptoms:** 周五手动清理会议，代码风格不一致，重复修复。

**Root cause:** 黄金原则未明确说明。

**Fix (OpenAI):** 定义明确的原则（5-10 条规则）。构建日常漂移扫描仪。自动打开修复 PR。跟踪清理时间作为指标（目标：零）。

**Fix (Claude):** 将 `.claude/hooks/PostToolUse.sh` 中的原则编码为确定性执行。使用退出代码 2 在违规行为发生之前将其阻止。跟踪吊钩触发率和人工超控率。

**Universal:** 目标是趋向于零手动清理。如果它没有下降，那么你的原则还不够具体。

**Skill routing accuracy drops after adding new skills**

**Symptoms:** agent为任务调用错误的技能，跳过相关技能，降低评估分数。

**Root cause:** 模型无法消除相似技能之间的歧义。

**Fix (OpenAI + Claude):** 在每个技能的描述中添加明确的负面示例（“当……时不要调用此技能”）。包括边缘情况覆盖。在 frontmatter 中添加“何时使用与何时不使用”块。

**Evidence from production:** 仅使用此模式，Glean 就恢复了 20% 的准确率下降。

**Universal:** 技能路由是一个领先指标。监控调用准确性（目标 >= 90%）。当您添加新技能时，立即向现有技能添加反面例子。

**Agent can't reproduce bug or validate fix**

**Symptoms:** agent报告“无法验证”或“无法测试”，依赖人工运行应用程序。

**Root cause:** agent无法识别应用程序行为。

**Fix (OpenAI):** 使应用程序可按工作树启动。将 Chrome DevTools 协议连接到agent运行时。通过临时可观测性堆栈（LogQL/PromQL 访问）公开日志和指标。

**Fix (Claude):** 构建 [MCP servers for app interaction](https://code.claude.com/docs/en/mcp) ：浏览器自动化（Playwright/Puppeteer）、API 测试（HTTP 客户端）、日志访问（通过 MCP 的 tail -f）。使应用程序可按工作树启动。使用[Claude artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)存储测试结果和证据。

**Universal:** 该应用程序必须可由agent驱动。如果agent无法启动它、查询它并观察它的行为，它就无法验证修复。

---

## 与工具无关的适应：OpenAI、Claude、Gemini、Grok

无论模型如何，本文中的模式都适用。

```
flowchart TD
    subgraph harness ["Your harness (model-agnostic)"]
        I["Instruction contract AGENTS.md or CLAUDE.md"]
        SK["Skill contract versioned procedures"]
        EX["Execution contract shell or MCP"]
        V["Validation contract evals + policy checks"]
        CX["Context contract compaction + pinned state"]
    end
    I --> SK --> EX --> V --> CX
    harness --> CO["OpenAI Codex"]
    harness --> CL["Claude Code"]
    harness --> GE["Gemini Code Assist"]
    harness --> GR["Grok"]
    style harness fill: #e8f4fd, stroke: #0d6efd
```

制作一次线束后，交换下面的模型。保持这个标准化的接口：

| 合同 | OpenAI 实施 | 克劳德实施 | 通用图案 |
| --- | --- | --- | --- |
| **Instruction** | AGENTS.md（约 100 行）+ docs/ | CLAUDE.md（50-120 行）+ .claude/ | 短索引文件+深层链接文档 |
| **Skill** | SKILL.md 与 frontmatter | .claude/skills/\*.md 与 frontmatter | 具有路由逻辑的版本化工作流程包 |
| **Execution** | Shell 工具（托管或本地） | MCP 服务器（默认为本地） | 带有审计跟踪的沙盒执行 |
| **Validation** | 定制 linter + 结构测试 | 挂钩（工具使用前/后） | 生命周期事件中的机械执行 |
| **Context** | 服务器端压缩 + previous\_response\_id | 自动压缩+AfterCompaction钩子+工件 | 具有关键上下文固定的自动摘要 |

### 迁移路径

**Codex → Claude code:**

1. 重命名 `AGENTS.md` → `CLAUDE.md`（修剪至 50-120 行）
2. 转换自定义 linter → `.claude/hooks/PostToolUse.sh`
3. 转换 shell 工作流程 → MCP 服务器
4. 保持 `docs/` 结构不变（两者都适用）
5. 保持 `skills/` 完整（相同格式）

**Claude code → codex:**

1. 合并 `CLAUDE.md` + `.claude/` 上下文 → `AGENTS.md`（约 100 行）
2. 转换 `.claude/hooks/` → 基于 CI 的执行
3. 转换 MCP 服务器 → shell 工具脚本
4. 保持`docs/`结构不变
5. 将 `.claude/skills/` 保留为 `skills/`（相同格式）

**Universal patterns that port directly:**

- 架构图（docs/architecture/）
- ADR（文档/决定/）
- 证据文档（docs/evidence/）
- 黄金原则（编码为强制执行，而不是散文）
- 技能路由模式（反例、LOV、模板）

目标是收敛。随着 [AGENTS.md](https://aiengineerguide.com/blog/how-to-use-agents-md-in-claude-code/) 、 [MCP](https://code.claude.com/docs/en/mcp) 和 [Skills](https://claude.com/blog/skills-explained) 等标准通过生态系统协作变得成熟，这些模式变得更加可移植。现在，适应您平台的原语，但保留基本原则。

---

## 平台环境：2025 年发生了什么变化

**Info**

背景语境

本节介绍使harness engineering成为可能的平台转变。如果您已经熟悉 2025 年车型格局，则可以跳过。

**Reasoning: OpenAI and Anthropic approaches**

**OpenAI:** 推理模型从 o1/o3 研究演变为生产能力。到 2025 年底，推理将与 GPT-5.2 系列中的通用模型融合，其中 GPT-5.2 Pro 可实现更深层次的推理工作负载。 “更加努力地思考与更快地响应”成为了一个可调的开发人员决策 - 使用专业模型进行复杂的多步骤工作，使用标准模型进行快速迭代。

**Anthropic:** Claude 3 Opus 和 Claude 3.5 Sonnet 通过更大的上下文窗口（200K tokens）和 [extended thinking capabilities](https://www.anthropic.com/news/claude-3-5-sonnet) 优先考虑扩展推理。 Claude Desktop 引入了 [Thinking Mode](https://code.claude.com/docs/en/thinking-mode)（2026 年初）——在回答之前明确显示推理步骤。通过系统提示进行控制：“一步步思考”或“展示你的推理”。

**Convergence:** 现在，两个生态系统都将推理深度视为可调参数，而不是特定于模型的功能。

**Multimodality: OpenAI and Anthropic**

**OpenAI:** 直接在 API 中生成 PDF 和文档（包括无需上传的 PDF-by-URL）。 Whisper 用于语音转文本，TTS 模型用于可控文本转语音。 GPT Image 1.5 用于更高保真度的图像生成和编辑。 Sora 用于生成具有时间一致性的视频。 gpt-realtime 用于低延迟语音对话。通过工具调用将图像生成集成到多轮对话中。

**Anthropic:** [Claude 3.5 Sonnet supports vision](https://www.anthropic.com/news/claude-3-5-sonnet)（图像、PDF、屏幕截图），具有行业领先的 OCR 和图表理解能力。 Claude Desktop 支持图像和 PDF 的拖放。 [Artifacts support rich media up to 20MB](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) 包括图像、交互式图表和结构化文档。宣布 2026 年推出音频和视频功能。

**Key difference:** OpenAI 的多模态以输出为中心（通过 GPT Image 1.5、Sora 生成）。 Claude 以输入为中心（通过视觉理解，OCR）。对于生成，通过 API 使用 GPT Image/Sora，并由 Claude 处理编排和推理。

**Agent-native APIs: OpenAI and Anthropic**

**OpenAI:** Responses API 支持多种输入/输出，包括不同的模式、推理控制和推理过程中的工具调用。适用于 [Python](https://openai.github.io/openai-agents-python/) 和 [TypeScript](https://openai.github.io/openai-agents-js/) 的开源agent SDK
- 与提供商无关，具有非 OpenAI 的记录路径
模型。 [AgentKit](https://openai.com/index/introducing-agentkit/) 添加了 Agent Builder、ChatKit、连接器注册表和评估循环。用于持久线程的对话状态 + 对话 API。用于外部上下文的连接器和 MCP 服务器。 [Apps SDK](https://developers.openai.com/apps-sdk) 扩展了 MCP，让开发人员可以与 MCP 服务器一起构建 UI。

**Anthropic:**[Claude Messages API](https://docs.anthropic.com/en/api/messages)支持多轮对话，包括工具使用、视觉和系统提示。 [Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) 为agent工作流程提供更高级别的抽象。 [MCP (Model Context Protocol)](https://code.claude.com/docs/en/mcp) 是AI工具集成的开放标准 - 提供 50 多个社区服务器。 [Claude Desktop](https://www.anthropic.com/news/claude-desktop) 为agent工作流程提供本机 GUI。 [Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) 将重复前缀的成本降低了 90%。

**Convergence:** 两者都采用MCP作为标准集成协议。两者都提供多轮对话 API 和工具使用。 OpenAI 的 Agents SDK 与提供商无关，并与 Claude 配合使用。

**Coding assistants: Codex and Claude Code**

**OpenAI codex:** 开源 [CLI](https://github.com/openai/codex) 将agent式编码引入本地环境。 AGENTS.md 支持、MCP 集成、沙箱和审批模式使其可以投入生产。通过将 CLI 作为 MCP 服务器运行，可以通过 Agents SDK 来编排 Codex。 CI 中的 Codex Autofix 收紧了循环。

**Claude code:** [Production-ready AI coding assistant](https://code.claude.com/) 理解整个代码库。主要功能：CLAUDE.md 自动加载，[hooks at 15 lifecycle events](https://docs.anthropic.com/en/docs/claude-code/hooks)、[specialized subagents](https://code.claude.com/docs/en/sub-agents)、[MCP server integration](https://code.claude.com/docs/en/mcp)、[automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) 用于长时间会话。 [CLI](https://code.claude.com/docs/en/cli)（终端优先）和[Desktop](https://code.claude.com/docs/en/desktop)（GUI）模式均可用。

**Key difference:** Codex 专注于具有本地回退功能的托管/云优先执行。 Claude Code 注重本地优先执行和持久的项目意识。两者都支持MCP，都与CI/CD集成，都提供审批模式。

**Anthropic: Claude Code ecosystem matured**

[Claude code launched](https://code.claude.com/) 作为一个可用于生产的AI编码助手，可以理解整个代码库。 2025 年/2026 年初的主要里程碑：

- **CLAUDE.md specification:** [Automatic loading of repo-specific instructions](https://code.claude.com/docs/en/best-practices)
在会话开始时
- **MCP standardization:** [Model Context Protocol became the standard for AI-tool integrations](https://code.claude.com/docs/en/mcp)
，随着社区生态系统的不断发展
- **Hooks launched:** [15 lifecycle events for deterministic automation](https://docs.anthropic.com/en/docs/claude-code/hooks)
（2026 年初）
- **Subagents:** [Specialized agents for security, testing, review](https://code.claude.com/docs/en/sub-agents)
- **Artifacts expanded:** [Up to 20MB storage per artifact](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)
，启用有状态应用程序
- **Desktop + CLI convergence:** 两种模式共享 [skills](https://code.claude.com/docs/en/skills)
、 [MCP servers](https://code.claude.com/docs/en/mcp) 和配置
- **Expanded use cases:** Claude Code 用于软件开发之外的一般知识工作

[Eight trends defining how software gets built in 2026](https://claude.com/blog/eight-trends-defining-how-software-gets-built-in-2026)：agent编排取代了人工编写的逻辑作为主要模式。

**Both ecosystems: production concerns became system design**

提示缓存共享前缀的延迟和输入成本。用于长时间运行响应的后台模式，无需保持连接打开。用于事件驱动系统的 Webhook。利率限制已成熟。建筑agent现在与系统设计和提示一样重要。

**OpenAI:** 响应 API，后台模式，`/responses/compact` 端点。 **Claude:** [Automatic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) ，用于状态的工件，用于集成的 MCP。

**Both ecosystems: open standards converged**

**OpenAI:** 推动 [AGENTS.md spec](https://agents.md/) 并参与 [Agentic AI Foundation (AAIF)](https://aaif.io/) 以及 MCP 和技能标准。发布的开放重量模型：[gpt-oss 120b & 20b](https://huggingface.co/collections/openai/gpt-oss)用于自托管，加上[gpt-oss-safeguard 120b & 20b](https://huggingface.co/collections/openai/gpt-oss-safeguard)安全模型。 Apps SDK 扩展了用于 UI 开发的 MCP。

**Claude:** [CLAUDE.md + AGENTS.md compatibility](https://github.com/anthropics/claude-code/issues/6235) 、 [MCP as primary integration protocol](https://code.claude.com/docs/en/mcp) 、 [Skills aligned with Agent Skills standard](https://code.claude.com/docs/en/skills) 。可以通过API或本地部署使用gpt-oss模型。

**Shared infrastructure:** 开放标准（AGENTS.md、MCP、AAIF 技能）、评估框架、强化微调 (RFT)、监督微调、将质量推向更小模型的蒸馏模式。

**Recommended models by task (end of 2025):**

**Info**

模型命名上下文

本部分使用 OpenAI 2025 年底平台更新中的模型命名（准备源材料）。对于 Claude 同等产品，请使用截至 2026 年初的当前生产型号名称。

| 任务 | 开放AI | 人择 | 笔记 |
| --- | --- | --- | --- |
| **General-purpose (text + multimodal)** | GPT-5.2 | 克劳德 3.5 十四行诗 | 用于聊天、长上下文工作和多模式输入 |
| **Deep reasoning / reliability-sensitive** | GPT-5.2 专业版 | 克劳德 3 作品 | 质量值得额外计算的规划和任务 |
| **Coding and software engineering** | GPT-5.2-法典 | 克劳德法典（十四行诗 3.5） | 代码生成、审查、回购规模推理 |
| **Image generation and editing** | GPT 映像 1.5 | 不适用 | 更高保真度的生成和迭代编辑 |
| **Realtime voice** | GPT 实时 | 不适用 | 低延迟语音到语音和实时语音agent |

**Key insights:**

- **OpenAI strengths:** 原生多模态生成（GPT Image 1.5、通过 Sora 的视频）、实时语音 (gpt-realtime)、推理功能 (GPT-5.2 Pro)
- **Claude strengths:** 更长的上下文窗口（200K），卓越的代码理解，更好地遵循复杂的
指令，MCP优先集成
- **For harness engineering:** 两个生态系统都有效。根据您现有的基础设施、成本限制和
具体任务要求。本文中的模式对两者都适用。

---

## 参考

### OpenAI 来源

1. harness engineering：<https://openai.com/index/harness-engineering/>
2. 技能+外壳+压实技巧：<https://developers.openai.com/blog/skills-shell-tips>
3. 构建 ChatGPT Apps的 15 节课程：<https://developers.openai.com/blog/15-lessons-building-chatgpt-apps>
4. 2025 年面向开发人员的 OpenAI：<https://developers.openai.com/blog/openai-for-developers-2025>
5. 法典 CLI：<https://github.com/openai/codex>
6. OpenAI Agents SDK (Python)：<https://openai.github.io/openai-agents-python/>
7. OpenAI Agents SDK (TypeScript)：<https://openai.github.io/openai-agents-js/>
8. agent工具包：<https://openai.com/index/introducing-agentkit/>
9. 土豚：<https://openai.com/index/introducing-aardvark/>
10. Skybridge 框架：<https://github.com/alpic-ai/skybridge>

### 克劳德/人类来源

11. 克劳德代码最佳实践：<https://code.claude.com/docs/en/best-practices>
12. 创建完美的 CLAUDE.md: <https://dometrain.com/blog/creating-the-perfect-claudemd-for-claude-code/>
13. 克劳德代码挂钩指南：<https://aiorg.dev/blog/claude-code-hooks>
14. 通过 MCP 将 Claude Code 连接到工具：<https://code.claude.com/docs/en/mcp>
15. 构建 MCP 服务器：<https://www.sitepoint.com/building-mcp-servers-custom-context-for-claude-code/>
16. 克劳德代码子agent：<https://code.claude.com/docs/en/sub-agents>
17. 克劳德压缩文档：<https://platform.claude.com/docs/en/build-with-claude/compaction>
18. 克劳德技能文档：<https://code.claude.com/docs/en/skills>
19. 克劳德技能库：<https://mcpservers.org/claude-skills>
20. 克劳德文物指南：<https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them>
21. 理解克劳德的谈话
压缩：<https://www.ajeetraina.com/understanding-claudes-conversation-compacting-a-deep-dive-into-context-management/>
22. 克劳德代码安全最佳实践：<https://www.mintmcp.com/blog/claude-code-security>
23. 定义软件的八个趋势
2026：<https://claude.com/blog/eight-trends-defining-how-software-gets-built-in-2026>
24. 与 Claude Agent 一起构建agent
软件开发工具包：<https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
25. 2026 年agent编码完整指南：<https://www.teamday.ai/blog/complete-guide-agentic-coding-2026>

### 社区资源

26. 很棒的克劳德代码：<https://github.com/hesreallyhim/awesome-claude-code>
27. 一切克劳德代码（生产配置）：<https://github.com/affaan-m/everything-claude-code>
28. 克劳德代码展示：<https://github.com/ChrisWiles/claude-code-showcase>
29. 用于生产的克劳德代码挂钩：<https://www.pixelmojo.io/blogs/claude-code-hooks-production-quality-ci-cd-patterns>
30. 如何在Claude代码中使用AGENTS.md：<https://aiengineerguide.com/blog/how-to-use-agents-md-in-claude-code/>

### 通用图案

31. FlowForge：<https://github.com/com-vitthalmirji/flowforge>
32. 代码审查操作系统（配套文章）：[Build a code review operating system](https://vitthalmirji.com/code-review-system/)
33. AGENTS.md 规范：<https://agents.md/>
34. 解析，不验证（Alexis King）：<https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/>
35. AI正在迫使我们编写好的代码：<https://bits.logic.inc/p/ai-is-forcing-us-to-write-good-code>
36. 拉尔夫·维格姆循环：<https://ghuntley.com/loop/>
37. 好的上下文，坏的上下文：编码agent的 AGENTS.md：<https://addyosmani.com/blog/agents-md/>

---

如果你现在是员工或校长，那么你最有影响力的工作就不再是编写优秀的代码了。

在构建一个工具时，优秀的代码将成为系统的默认输出。

OpenAI 在 5 个月内交付了 100 万条线路，每条线路均由agent生成。他们通过投资环境设计而不是实施速度来做到这一点。

**You can build the same harness with Claude Code.** 模式是相同的。工具不同。结果是一样的。

[Read the companion: Build a code review operating system](https://vitthalmirji.com/posts/2026/02/code-review-operating-system/)

您今天就可以开始构建您的安全带。您已经获得了两个生态系统的地图。从您的说明文件（AGENTS.md 或 CLAUDE.md）开始。其余的如下。
