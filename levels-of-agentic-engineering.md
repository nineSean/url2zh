# Agentic Engineering 的 8 个层级

> 原文: https://www.bassimeledath.com/blog/levels-of-agentic-engineering
> 作者: Bassim Eledath
> 日期: March 10, 2026 (Updated March 12, 2026)

---

AI 的编码能力，正在以快于我们有效驾驭它的速度提升。这就是为什么 SWE-bench 分数刷得再高，也没法和工程管理真正关心的生产力指标对齐。当 Anthropic 的团队能在 10 天内交付像 Cowork 这样的产品，而另一个团队用着同样的模型却连一个坏掉的 POC 都推进不了时，差别就在于：前者弥合了能力与实践之间的鸿沟，后者没有。

这道鸿沟不会一夜之间消失。它是分层级被填平的，一共 8 个。读到这里的多数人，很可能已经越过前几个层级；你也应该渴望迈向下一个，因为每往上一个层级，产出都会出现巨大的跃升，而模型能力的每一次提升，又会进一步放大这些收益。

你还应该关心它的另一个原因，是多人协作效应。你的产出，受队友所处层级的影响，可能比你想象得更大。假设你已经是 7 级高手，睡觉时后台 agents 还在帮你提好几个靠谱的 PR；但如果你的仓库要求必须等同事审批才能合并，而那位同事还停留在 2 级、仍在手工 review PR，你的吞吐量就会被卡住。所以，从自己的利益出发，你也该把整个团队一起往上带。

基于我和多支团队、多个 AI 辅助编程实践者的交流，下面是我观察到的一条层级演进路径，顺序并不绝对严格：

![The 8 Levels of Agentic Engineering](https://www.bassimeledath.com/images/blog/agentic_thumbnail_notext.jpg)

## Level 1 & 2: Tab Complete 和 Agent IDE

这两个层级我会快速带过，主要是为了存档。随便略读就好。

一切始于 Copilot 和 tab complete。按一下 Tab，代码自动补全。对很多人来说，这可能早就成了遥远记忆；而对后来进入 agentic engineering 的新人来说，甚至可能直接跳过了这一阶段。这个阶段更偏爱有经验的开发者，因为他们能先把代码骨架搭出来，再让 AI 把空白补上。

像 Cursor 这样的 AI 导向 IDE 改变了游戏规则：它把聊天能力接入代码库，让跨文件修改变得容易得多。但它的上限始终是上下文。模型只能帮助它“看得见”的东西，而令人抓狂的是，它往往要么没看到正确的上下文，要么看到了太多错误的上下文。

这个层级的大多数人，也会开始尝试自己所用编码 agent 的 plan mode：先把一个模糊想法翻译成给 LLM 的分步计划，再不断迭代这份计划，最后触发实现。在这个阶段，这种做法效果不错，也是维持控制权的合理方式。不过后面你会看到，对 plan mode 的依赖会越来越少。

## Level 3: Context Engineering

作为 2025 年的年度热词，context engineering 之所以兴起，是因为模型终于可靠到可以在“合适数量的指令 + 恰到好处的上下文”条件下稳定完成任务。噪声上下文和上下文不足一样糟糕，于是努力方向就变成了提升每个 token 的信息密度。“Every token needs to fight for its place in the prompt” 成了这阶段的口号。

![Same message, fewer tokens — information density was the name of the game (source: humanlayer/12-factor-agents)](https://www.bassimeledath.com/images/blog/context-eng.png)

在实践中，context engineering 影响的范围比很多人意识到的更广。它包括你的 system prompt 和规则文件 (`.cursorrules`、`CLAUDE.md`)；包括你如何描述工具，因为模型会读取这些描述来决定要调用哪些工具；包括你如何管理对话历史，避免一个长时间运行的 agent 在第十轮时就偏离主线；还包括你每一轮到底暴露哪些工具，因为工具太多会像把人丢进选项爆炸里一样，让模型也无所适从。

这些天你已经没那么常听到 context engineering 这个词了。天平已经偏向那些能容忍更嘈杂上下文、能在更混乱地形中推理的模型了，更大的上下文窗口也帮了忙。即便如此，清楚什么东西会吞噬上下文，依然很重要。几个仍然会踩坑的场景：

- **更小的模型对上下文更敏感。** 语音应用常用较小模型，而上下文大小还和首 token 延迟相关，这会直接影响时延。
- **高 token 消耗的工具和模态。** 像 Playwright 这样的 MCP，以及图像输入，都会非常快地烧掉 token，让你比预期更早进入 Claude Code 的 “compact session” 状态。
- **拥有几十个工具访问权的 agents。** 模型可能把 token 都花在解析工具 schema 上，而不是做真正有价值的工作。

更广义地说，context engineering 并没有消失，它只是进化了。关注点已经从“过滤掉坏上下文”，转向“确保正确的上下文在正确的时间出现”。而这个转变，正是 Level 4 的铺垫。

## Level 4: Compounding Engineering

context engineering 提升的是当前这个 session；[compounding engineering](https://every.to/source-code/my-ai-had-already-fixed-the-code-before-i-saw-it) 提升的是之后的每一个 session。这个概念由 Kieran Klaassen 推广开来。对我和很多人来说，它都是一个拐点，让我们意识到 “vibe coding” 的上限远不止原型开发。

它是一个 “plan, delegate, assess, codify” 的循环。你先为任务准备足够的上下文，让 LLM 有机会成功；然后把任务委派出去；接着评估结果；最关键的是，把你学到的东西编码下来：什么有效，什么失效，下次该遵循什么模式。

![The compounding loop: plan, delegate, assess, codify — each cycle makes the next one better](https://www.bassimeledath.com/images/blog/compounding-eng.png)

正是 `codify` 这一步，让它具备了复利效应。LLM 是无状态的。如果它今天重新引入了你昨天明确删掉的依赖，那么只要你不提醒它，明天它还会再犯。最常见的闭环方式，就是更新你的 `CLAUDE.md` (或同类规则文件)，把这次经验烘焙进未来所有 session 里。顺带提醒一句：把一切都塞进规则文件里的冲动，可能会适得其反，指令太多和没有指令，效果差不多。

更好的做法，是构建一种环境，让 LLM 可以自己轻松发现有用上下文。比如维护一个始终最新的 `docs/` 目录 (Level 7 里会再谈)。

做 compounding engineering 的人，通常会对喂给 LLM 的上下文异常敏感。当 LLM 出错时，他们的第一反应往往是“是不是上下文缺了什么”，而不是先责怪模型不够聪明。正是这种直觉，使 Level 5 到 Level 8 成为可能。

## Level 5: MCP 和 Skills

Level 3 和 4 解决的是上下文问题，Level 5 解决的是能力问题。MCP 和自定义 skills 让你的 LLM 能访问数据库、API、CI 流水线、设计系统、用于浏览器测试的 Playwright，以及用于通知的 Slack。模型不再只是“思考”你的代码库，它现在可以直接对其采取行动。

关于 MCP 和 skills 是什么，已经有大量不错的资料，所以我不重复解释。我更想讲讲自己怎么用它们。我的团队共享一个 PR review skill，而且我们一直在持续迭代它。这个 skill 会根据 PR 的性质，有条件地拉起不同的 subagents：一个负责数据库集成安全，一个做复杂度分析、标记冗余和过度设计，另一个检查 prompt 健康度，确保 prompts 符合团队的标准格式。

它还会跑 linters 和 Ruff。

![A single PR triggers a review skill that fans out into specialized subagents — each checking a different dimension of quality](https://www.bassimeledath.com/images/blog/review_skill.jpg)

为什么要在 review skill 上投入这么多？因为当 agents 开始批量生产 PR 时，人类 review 才会成为瓶颈，而不是质量关口。[Latent Space 的一篇文章](https://www.latent.space/p/reviews-dead) 很有说服力地指出：我们熟悉的代码评审已经死了。取而代之的，将是自动化、一致、由 skills 驱动的 review。

在 MCP 这边，我会用 Braintrust MCP，让我的 LLM 可以查询评测日志并直接做修改。我也会用 DeepWiki MCP，让 agent 能访问任意开源仓库的文档，而不需要我手动把文档塞进上下文。

当团队里多个人都在各自写同一种 skill 的不同版本时，就值得把它们收敛成共享注册表了。[Block](https://engineering.block.xyz/blog/3-principles-for-designing-agent-skills) (向你们致哀) 有一篇很好的文章：他们搭建了一个内部 skills marketplace，里面有 100 多个 skill，以及面向特定角色和团队的精选组合包。skills 被像代码一样对待：有 pull request、有 review、有版本历史。

还有一个值得特别指出的趋势：越来越多的 LLM 开始直接使用 CLI 工具，而不是 MCPs。看起来几乎每家公司都在发布自己的 CLI，比如 [Google Workspace CLI](https://justin.poehnelt.com/posts/rewrite-your-cli-for-ai-agents/)，Braintrust 也快出了。原因在于 token 效率。MCP servers 会在每一轮把完整的工具 schema 注入上下文，不管 agent 实际会不会用；CLI 则相反，agent 只在需要时运行一个定向命令，只有相关输出才会进入上下文窗口。正因为这个原因，我大量使用 `agent-browser`，而不是 Playwright MCP。

**继续之前还要强调一点。** Level 3 到 Level 5 是后续一切的积木。LLM 在某些事情上会好得出奇，在另一些事情上却会差得离谱；你需要逐步形成对这些边界的直觉，然后才能把更多自动化安全地叠上去。如果你的上下文很嘈杂、prompts 规格不清或描述错误、工具说明又写得很差，那么 Level 6 到 Level 8 只会把这堆混乱成倍放大。

## Level 6: Harness Engineering & Automated Feedback Loops

这时，火箭才算真正开始起飞。

context engineering 关注的是模型“看见什么”；[harness engineering](https://openai.com/index/harness-engineering/) 关注的是构建完整的环境、工具链和反馈回路，让 agents 能在没有你介入的情况下稳定地完成工作。给 agent 的不只是一个编辑器，而是一整套反馈机制。

![OpenAI's Codex harness — a full observability stack wired into the agent so it can query, correlate, and reason about its own output (source: OpenAI)](https://www.bassimeledath.com/images/blog/harness-eng.png)

OpenAI 的 Codex 团队把 Chrome DevTools、可观测性工具和浏览器导航都接进了 agent runtime，这样它就能截图、驱动 UI 路径、查询日志，并验证自己修复的问题。只给一个 prompt，agent 就可以复现 bug、录视频、实现修复；然后它还能通过实际驱动应用来做验证、打开 PR、响应 review 反馈、完成合并，只有在确实需要判断时才升级给人处理。agent 做的已经不只是写代码。

它还能看到代码运行出来的结果，并像人一样围绕结果继续迭代。

我的团队在做用于技术故障排查的语音和聊天 agents，所以我写了一个叫 `converse` 的 CLI 工具，让任意 LLM 都能和我们的后端 endpoint 进行逐轮对话。LLM 一边改代码，一边用 `converse` 对线上系统做真实对话测试，再继续迭代。有时候，这种自我改进循环会连续跑上好几个小时。

当结果是可验证的，这种模式尤其强大。比如，一段对话必须遵循某种流程，或者必须在某些情形下调用指定工具 (例如升级到人工客服)。

支撑这一切的概念是 [backpressure](https://latentpatterns.com/glossary/agent-backpressure)：类型系统、测试、linters、pre-commit hooks 之类的自动反馈机制，让 agents 能在没有人工介入的情况下发现并修正错误。如果你想要 autonomy，就必须有 backpressure；否则你得到的只会是一台稳定制造垃圾的机器。这一点也延伸到安全层面。[Vercel CTO 的这篇文章](https://vercel.com/blog/security-boundaries-in-agentic-architectures) 指出，agents、它们生成的代码，以及你的 secrets，应该处在彼此隔离的信任域里。因为一个埋在日志文件中的 prompt injection，就可能骗过 agent，让它在共享安全上下文里把你的凭据泄露出去。安全边界本身就是一种 backpressure：它限制的不是 agent “应该”做什么，而是它在失控时“能”做什么。

有两件事特别有帮助：

- **为吞吐量而设计，而不是为完美而设计。** 如果每一次提交都要求完美，agents 就会围着同一个 bug 反复堆人、相互覆盖彼此的修复。更好的方式是允许一些不会阻塞发布的小错误存在，把真正的质量把关放到发布前的最后一道关。我们对人类同事也是这么做的。
- **约束大于指令。** 那种 “先做 A，再做 B，再做 C” 的分步 prompting，正在快速过时。以我的经验看，定义边界比给清单更有效，因为 agents 很容易盯死清单，而忽略任何不在清单上的事情。更好的 prompt 是：“这是我想要的结果，做到通过这些测试为止。”

harness engineering 的另一半，是确保 agent 不依赖你也能在仓库中自如导航。OpenAI 的做法是：把 `AGENTS.md` 控制在约 100 行左右，充当一份目录，指向别处结构化的文档；同时把文档新鲜度纳入 CI，而不是依赖随缘更新、最后慢慢过期。

当你把这些都搭起来后，一个很自然的问题就会出现：如果 agent 已经能验证自己的工作、能在仓库中导航、还能在没有你帮助的前提下纠正错误，那你为什么还必须坐在椅子前盯着它？

提前提醒一下：如果你还处在前几个层级，下一节听起来可能会有点陌生。但没关系，先收藏，之后再回来读。

## Level 7: Background Agents

一个也许有点刺耳的观点：plan mode 正在走向消亡。

Claude Code 的创造者 Boris Cherny 直到今天，仍然有 [80% 的任务](https://youtu.be/PQU9o_5rHC4?si=8j25qad3-J_DbCNX) 是从 plan mode 开始的。但随着每一代新模型出现，规划之后一次成事的成功率都在持续上升。我认为，我们正在接近这样一个拐点：plan mode 作为一个独立的人类在环步骤，会慢慢淡出。不是因为规划不重要了，而是因为模型已经越来越擅长自己完成规划。当然有个很大的前提：你必须先把 Level 3 到 Level 6 的基础工作做扎实。

如果你的上下文足够干净、约束足够明确、工具描述足够清楚、反馈回路足够紧，模型就能在不经过你预审计划的情况下稳定规划。反之，如果这些工作没做好，你仍然得盯着它的 plan。

说清楚一点，规划作为一种通用实践不会消失，它只是换了一种形态。对新手来说，plan mode 依然是正确的入门点，就像 Level 1 和 2 描述的那样。但到了 Level 7，复杂功能的 “planning” 更像是探索：在代码库里探路、在 worktrees 里试原型、描摹解空间。而且越来越多时候，这种探索正在由 background agents 替你完成。

这很关键，因为它正是 background agents 得以成立的前提。如果一个 agent 能产出靠谱计划、执行过程中又不需要你逐步签字，它就能在你做别的事时异步运行。真正的变化，是从“我手忙脚乱地同时盯着多个标签页”，变成“工作在我不盯着的时候也持续发生”。

[Ralph loop](https://ghuntley.com/loop/) 是很多人进入这个阶段时最常见的入口：一个自治 agent 循环，反复运行编码 CLI，直到所有 PRD 条目都完成；而每一轮都会拉起一个带干净上下文的新实例。以我的经验，想把 Ralph loop 调好很难，PRD 中任何描述不足或规格错误，都会在后面狠狠咬你一口。它还是有点过于 fire-and-forget 了。

你当然可以并行跑多个 Ralph loop，但 agents 一多，你就会越来越明显地意识到自己真正把时间花在了哪里：协调它们、安排先后顺序、检查输出、不断 nudging。你已经不是在写代码了，你是在做中层管理。你需要一个 orchestrator agent 来负责 dispatch，这样你才能把注意力放在意图上，而不是物流调度上。

![Dispatch launching 5 workers across 3 models in parallel — your session stays lean while agents do the work](https://www.bassimeledath.com/images/blog/dispatch.png)

我最近大量使用的工具是 [Dispatch](https://github.com/bassimeledath/dispatch)，这是[我自己构建的一个 Claude Code skill](https://www.bassimeledath.com/blog/dispatch)。它会把你的 session 变成一个指挥中心。你留在一个干净的主 session 里，而 workers 在隔离上下文中完成重活。dispatcher 负责规划、委派和追踪，因此主上下文窗口能保留下来做 orchestration。当某个 worker 卡住时，它会抛出一个澄清问题，而不是悄悄失败。

Dispatch 在本地运行，所以非常适合快速开发场景：反馈更快，更容易交互式调试，也没有基础设施负担。[Ramp 的 Inspect](https://builders.ramp.com/post/why-we-built-our-background-agent) 则是它的互补方案，适合更长时间运行、自治程度更高的工作：每个 agent session 都会在云端托管、带完整开发环境的沙箱 VM 中启动。产品经理在 Slack 里发现一个 UI bug，丢个标记过去，Inspect 就会在你合上笔记本之后继续推进。

它的代价是更高的运维复杂度，包括基础设施、快照、权限安全等；但它换来的是本地 agents 很难做到的规模化与可复现性。我的建议是，两种都用，本地 background agents 和云端 background agents 都应该有。

这个层级里有一种出奇有效的模式：不同工作用不同模型。最强的工程团队并不是由一群复制人组成的，而是由思考方式不同、训练经历不同、各有长处的人构成。LLM 也一样。这些模型的后训练路径不同，因此确实带有不同“性格”与偏好。

我经常把 Opus 派去做实现，把 Gemini 派去做探索式研究，再让 Codex 来做 review。最终累计起来的产出，会比任何单一模型单打独斗更强。有点像群体智慧，只不过对象变成了代码。

还有一个关键点：必须把实现者和评审者解耦。我已经吃过太多次亏了。如果同一个模型实例既实现自己的工作、又评估自己的工作，它天然会带偏见。它会忽略问题，甚至在任务根本没完成时也告诉你“一切都做好了”。这不是恶意，而是因为它就像人不会给自己批改试卷一样。让另一个模型，或者至少另一个带 review 专用 prompt 的实例，来完成评审，你得到的信号质量会明显提高。

![Don't let the same model grade its own exam — separate the implementer from the reviewer](https://www.bassimeledath.com/images/blog/model_separation_implement_review.jpg)

background agents 也打开了把 CI 和 AI 结合起来的闸门。一旦 agents 能在没有人坐镇的情况下持续运行，你就可以直接从现有基础设施触发它们。比如一个 docs bot，在每次 merge 后重新生成文档，并提一个 PR 去更新 `CLAUDE.md` (我们就是这么干的，节省了大量时间)；一个安全审查 agent，扫描 PR 并自动开修复；一个依赖机器人，不只是提醒升级，而是真的替你升级包并跑完整测试。

良好的上下文、能复利积累的规则、足够强的工具，再加上自动化反馈回路，现在全都能自治运行起来了。

## Level 8: Autonomous Agent Teams

目前还没有人真正掌握这个层级，但已经有少数团队开始逼近它了。这是眼下的前沿地带。

在 Level 7 中，是一个 orchestrator LLM 用 hub-and-spoke 模式把工作分发给多个 worker LLM；到了 Level 8，这个瓶颈会被拿掉。agents 彼此直接协调，自主认领任务、共享发现、标记依赖、解决冲突，不再要求所有事情都必须通过单一 orchestrator 中转。

Claude Code 的实验性 [Agent Teams](https://code.claude.com/docs/en/agent-teams) 功能，是这方面的早期实现：多个实例并行处理同一个共享代码库，每个“队友”都有自己的上下文窗口，并且彼此直接通信。Anthropic 曾用 16 个并行 agents 从零构建一个能编译 Linux 的 C 编译器。Cursor 则连续几周运行了数百个并发 agents，来从零搭建一个浏览器，并把自家代码库从 Solid 迁移到 React。

但你仔细看就会发现，这里面的缝线还很明显。Cursor 发现，如果没有层级结构，agents 会变得过度规避风险，在原地打转却不推进。Anthropic 的 agents 则不断破坏既有功能，直到他们加入了 CI 流水线来防止回归。所有在这个层级做实验的人，最后都会说出同一句话：multi-agent coordination 是一个很难的问题，而且目前没有人接近最优。

说实话，我并不认为现在的模型已经准备好在大多数任务上承受这种程度的 autonomy。即便它们的智能水平够了，它们在速度和 token 成本上也仍然太昂贵，不足以在编译器、浏览器这种 moonshot 项目之外成为经济可行的方案。这些项目很惊艳，但距离“干净、稳定、可持续”的工程现实还很远。对我们大多数人日常做的工作来说，真正的杠杆点仍然在 Level 7。

我也不会惊讶 Level 8 最终会变成主流模式，但就现在来说，除非你是 Cursor，而“突破本身就是业务”，否则我会把更多精力放在 Level 7。

## Level ?

接下来不可避免的问题就是：下一层会是什么？

一旦你已经能比较顺畅地编排 agent 团队，交互界面就没有理由永远停留在纯文本上。voice-to-voice，甚至未来可能是 thought-to-thought 式地与编码 agent 互动，也许会成为自然的下一步。那将是“可对话的 Claude Code”，而不是简单的语音转文字输入。你看着自己的应用，用嘴描述一连串改动，然后看着这些改动直接在眼前发生。

现在有不少人在追逐那个完美的一次成型：你说出想要什么，AI 就在单次执行里把一切完美拼出来。问题在于，这种想象默认了我们人类总是清楚知道自己想要什么。但事实并非如此，从来都不是。软件一直是迭代出来的，我相信未来也仍然如此。只不过这个迭代过程会变得容易得多，也会远远超出纯文本交互的范畴，而且速度会快得多。

所以，你现在处在哪个层级？你又在做什么，来把自己带到下一个层级？
