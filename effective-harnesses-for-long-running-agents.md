# 长时运行 agent 的高效 harness

> 原文: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
> 作者: Justin Young
> 日期: 2025-11-26

---

agent 在跨越多个上下文窗口工作时，仍然面临不少挑战。我们从人类工程师的工作方式中获得灵感，为长时运行的 agent 设计出了一种更高效的 harness。

随着 AI agent 能力不断增强，开发者越来越希望它们承担复杂任务，而这类任务往往需要持续数小时，甚至数天。不过，如何让 agent 在多个上下文窗口之间持续、稳定地推进工作，仍然是一个尚未解决的问题。

长时运行 agent 的核心挑战在于，它们必须以离散的 session 方式工作，而每个新 session 一开始都不记得之前发生过什么。你可以把它想象成一个软件项目由轮班工程师接手，但每位新来的工程师都完全不记得上一班做过什么。由于上下文窗口有限，而大多数复杂项目又无法在单个窗口内完成，因此 agent 需要一种机制来跨越不同编码 session 之间的断层。

我们开发出了一套双管齐下的方案，让 [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) 能在多个上下文窗口中高效工作：第一次运行时由一个 initializer agent 负责初始化环境，之后每个 session 都由 coding agent 负责逐步推进，并为下一次 session 留下清晰的工作痕迹。相关代码示例可以在配套的 [quickstart](https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding) 中找到。

## 长时运行 agent 问题

Claude Agent SDK 是一个强大的通用 agent harness，擅长编码，也适用于那些需要模型使用工具来收集上下文、规划和执行的任务。它具备诸如 compaction 之类的上下文管理能力，使 agent 能在不耗尽上下文窗口的情况下持续处理任务。从理论上讲，在这样的设定下，agent 应该能够在任意长的时间里持续产出有效工作。

但 compaction 本身并不够。开箱即用时，即便是像 Opus 4.5 这样的前沿编码模型，运行在 Claude Agent SDK 上并在多个上下文窗口中循环执行，如果只给它一个高层 prompt，例如“构建一个 [claude.ai](https://claude.ai) 的克隆版”，最终也无法做出生产级的 Web 应用。

Claude 的失败主要表现为两种模式。第一，agent 往往会试图一次做太多事，几乎等于想“一把梭”把整个应用做完。结果通常是模型在实现过程中耗尽上下文，导致下一个 session 接手时面对的是一个只做了一半、且没有文档说明的功能。接手的 agent 只能猜测之前发生了什么，并花费大量时间试图先把基础应用重新跑通。

这种情况即便在使用 compaction 时也会发生，因为 compaction 并不总能向下一个 agent 传递足够清晰的指令。

第二种失败模式通常发生在项目后期。当已经做出一些功能后，后续的某个 agent 实例会四处查看，发现“似乎已经有不少进展”，然后直接宣布任务完成。

这就把问题拆成了两个部分。第一，我们需要先搭建一个初始环境，为 prompt 所要求的全部功能打好基础，让 agent 能够按步骤、按功能逐项推进。第二，我们应当让每个 agent 在每个 session 中都只做增量推进，同时在结束时把环境收拾到“干净状态”。

这里所说的“干净状态”，指的是那种可以直接合并到主分支的代码状态：没有重大 bug，代码整洁且有文档说明，总体上开发者无需先清理一堆无关的烂摊子，就能直接开始开发新功能。

在内部实验中，我们通过两部分方案来解决这些问题：

1. Initializer agent：第一个 agent session 使用专门的 prompt，要求模型搭建初始环境，包括一个 `init.sh` 脚本、一个用于记录 agent 工作日志的 `claude-progress.txt` 文件，以及一个展示新增文件的初始 git commit。
2. Coding agent：此后的每个 session 都要求模型做增量推进，并留下结构化更新。[^1]

这里的关键洞见，是让 agent 在一个全新的上下文窗口里也能迅速理解当前工作状态。这通过 `claude-progress.txt` 文件与 git 历史共同实现。我们之所以想到这些做法，很大程度上来自对高效软件工程师日常工作方式的观察。

## 环境管理

在更新后的 [Claude 4 prompting guide](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices#multi-context-window-workflows) 中，我们分享了多上下文窗口工作流的一些最佳实践，其中包括一种 harness 结构：在“第一个上下文窗口使用不同的 prompt”。这个“不同的 prompt”会要求 initializer agent 搭建好环境，把后续 coding agent 高效工作所需的上下文都准备出来。下面我们更深入介绍这种环境中的几个关键组成部分。

### 功能列表

为了避免 agent 试图一次性完成整个应用，或者过早认为项目已经做完，我们要求 initializer agent 写出一份完整的功能需求文件，把用户最初的 prompt 展开成细化需求。

在 [claude.ai](https://claude.ai) 克隆示例中，这意味着要写出 200 多个功能点，例如“用户可以打开一个新聊天，输入问题，按下回车，然后看到 AI 回复”。这些功能在一开始都被标记为“失败”，这样后续的 coding agent 就能清楚知道完整功能应该长什么样。

```json
{
  "category": "functional",
  "description": "新聊天按钮会创建一个全新的对话",
  "steps": [
    "进入主界面",
    "点击 'New Chat' 按钮",
    "确认已创建新对话",
    "检查聊天区域显示欢迎状态",
    "确认该对话出现在侧边栏中"
  ],
  "passes": false
}
```

我们会要求 coding agent 只能通过修改 `passes` 字段的状态来编辑这个文件，并用措辞强烈的指令提醒它，比如“绝对不能删除或修改测试，因为这可能导致功能缺失或引入 bug”。经过一些实验后，我们最终选择用 JSON 来保存这些内容，因为相比 Markdown，模型更不容易不恰当地修改或覆盖 JSON 文件。

### 增量推进

有了这些初始环境脚手架后，下一版 coding agent 就只被要求一次处理一个功能。事实证明，这种增量式方法对于解决 agent 总想一次做太多事的问题至关重要。

即便已经采用增量式开发，模型在改完代码后仍然必须把环境留在一个干净状态。在实验中，我们发现，最有效的做法是要求模型把自己的进展提交到 git，并使用清晰的 commit message，同时把进展摘要写入一个进度文件。这样一来，模型就可以利用 git 回滚错误改动，并恢复到代码库的可工作状态。

这些方法也提高了效率，因为 agent 不再需要猜之前到底发生了什么，也不必把大量时间花在先把基础应用修回可用状态这件事上。

### 测试

我们观察到的最后一个主要失败模式，是 Claude 倾向于在没有充分测试的情况下就把某个功能标记为完成。如果没有明确 prompt，Claude 虽然会修改代码，甚至会跑单元测试，或者对开发服务器执行 `curl` 命令，但它往往无法意识到功能在端到端层面其实并没有真正工作。

在构建 Web 应用时，一旦明确要求 Claude 使用浏览器自动化工具，并像真人用户那样完成所有测试，它在端到端验证功能方面就表现得相当不错。

![Claude 通过 Puppeteer MCP server 测试 claude.ai 克隆版时截取的截图。](https://cdn.sanity.io/images/4zrzovbb/website/f94c2257964fb2d623f1e81f874977ebfc0986bc-1920x1080.gif)

向 Claude 提供这类测试工具显著提升了表现，因为 agent 能发现并修复那些仅凭代码本身并不明显的 bug。

不过仍然存在一些问题，例如 Claude 的视觉能力以及浏览器自动化工具本身的局限，使它难以识别所有类型的 bug。举例来说，Claude 无法通过 Puppeteer MCP 看到浏览器原生的 alert 模态框，因此依赖这些模态框的功能往往会更容易出错。

## 快速进入状态

具备以上这些基础之后，每个 coding agent 在开始工作时，都会被提示按一系列步骤先“找准位置”。其中有些步骤看起来很基础，但依然非常有帮助：

1. 运行 `pwd`，确认当前工作的目录。你只能编辑这个目录中的文件。
2. 阅读 git 日志和进度文件，了解最近完成了什么工作。
3. 阅读功能列表文件，选择尚未完成且优先级最高的功能作为当前要处理的目标。

这种方法还能帮 Claude 在每个 session 里节省一些 token，因为它不需要再自己摸索该如何测试代码。我们也会要求 initializer agent 写一个 `init.sh` 脚本，用于启动开发服务器，并在实现新功能之前先跑一遍基础的端到端测试。

在 claude.ai 克隆版这个案例里，这意味着 agent 总是先启动本地开发服务器，再使用 Puppeteer MCP 新建一个聊天、发送一条消息并接收回复。这样 Claude 就能迅速判断应用是否被留在了一个损坏状态中，并立刻修复已有 bug。如果 agent 反过来先去实现新功能，问题往往只会更严重。

基于这些做法，一个典型 session 往往会以如下 assistant 消息开始：

```text
[Assistant] 我会先确认当前环境，弄清楚项目此刻所处的状态。
[Tool Use] <bash - pwd>
[Tool Use] <read - claude-progress.txt>
[Tool Use] <read - feature_list.json>
[Assistant] 让我看看 git log，了解最近做了哪些工作。
[Tool Use] <bash - git log --oneline -20>
[Assistant] 现在我来检查是否有用于重启服务的 init.sh 脚本。
<启动开发服务器>
[Assistant] 很好。接下来我会访问这个应用，并验证一些基础功能是否仍然正常。
<测试基础功能>
[Assistant] 根据刚才的验证测试，我可以看到基础功能运行良好。核心聊天功能、主题切换、对话加载和错误处理都工作正常。现在我会更全面地查看 tests.json 文件，以了解下一步应实现什么功能。
<开始开发新功能>
```

Agent 失败模式与解决方案

| 问题 | Initializer Agent 行为 | Coding Agent 行为 |
| --- | --- | --- |
| Claude 过早宣布整个项目已经完成。 | 建立功能列表文件：基于输入规格，建立一个结构化 JSON 文件，列出端到端功能描述。 | 在 session 开始时读取功能列表文件。选取一个单独功能作为起点。 |
| Claude 让环境停留在有 bug 或进展未文档化的状态。 | 写出初始 git 仓库和进度说明文件。 | session 开始时先阅读进度说明文件和 git commit 日志，并对开发服务器运行基础测试，以捕获未被记录的 bug。session 结束时写入 git commit 和进度更新。 |
| Claude 过早把功能标记为完成。 | 建立功能列表文件。 | 对所有功能做自我验证。只有在认真测试之后，才能把功能标记为“passing”。 |
| Claude 需要花时间摸索如何运行应用。 | 写一个可以启动开发服务器的 `init.sh` 脚本。 | session 开始时先阅读 `init.sh`。 |

以上总结了长时运行 AI agent 中四种常见的失败模式及其解决方案。

## 未来工作

这项研究展示了一种可能的长时运行 agent harness 方案，使模型能够跨越多个上下文窗口，以增量方式持续推进工作。不过，仍然有一些开放问题尚待回答。

最值得关注的一点是，我们仍不清楚：在跨上下文工作时，单个通用 coding agent 是否就是最佳方案，还是说多 agent 架构能取得更好表现。看起来很合理的一种猜想是，像测试 agent、质量保障 agent、代码清理 agent 这样的专门化 agent，可能会在软件开发生命周期中的各类子任务上表现得更好。

另外，这个 demo 目前针对的是全栈 Web 应用开发。未来的一个方向，是把这些发现推广到其他领域。我们很可能会发现，其中部分甚至全部经验，也能适用于其他需要长时运行 agent 的任务类型，例如科研或金融建模。

### 致谢

本文由 Justin Young 撰写。特别感谢 David Hershey、Prithvi Rajasakeran、Jeremy Hadfield、Naia Bouscal、Michael Tingley、Jesse Mu、Jake Eaton、Marius Buleandara、Maggie Vo、Pedram Navid、Nadine Yasser 和 Alex Notov 的贡献。

这项工作体现了 Anthropic 内多个团队的共同努力，正是这些团队让 Claude 能够以安全的方式进行长周期的自主软件工程，尤其要感谢 code RL 与 Claude Code 团队。欢迎有兴趣参与的候选人前往 [anthropic.com/careers](https://anthropic.com/careers) 申请加入。

### 脚注

[^1]: 我们在这里把它们称为不同的 agent，仅仅是因为它们收到的初始用户 prompt 不同。除此之外，system prompt、工具集合以及整体 agent harness 都是相同的。
