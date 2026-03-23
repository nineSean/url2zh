# 我如何打造一支 24/7 全天候运行的自主 AI 代理团队

> 原文: https://www.theunwindai.com/p/how-i-built-an-autonomous-ai-agent-team-that-runs-24-7
> 作者: Shubham Saboo
> 日期: 2月12日

---

## 手把手完整教程：用 OpenClaw 搭建你自己的代理团队

六个 AI 代理在我睡觉时替我运转整个生活。

这不是演示。也不是周末做着玩的项目。

这是一支真正 24/7 工作的团队，确保我永远不会落后。研究做完了，内容起草了，代码审查了，Newsletter 也准备好了。等我早上打开 Telegram 时，它们已经完整上完了一班。

昨天我发了一条关于我这支代理团队的帖子。大家问得最多的问题是：“这东西到底该怎么搭起来？”

这篇就是答案。不是理论，不是架构图，而是我实际在用的文件结构、我实际支付的成本、我实际踩过的坑。全部都讲。

看完这篇，你会准确理解如何搭建一支能在你睡觉时持续运行的自主 AI 代理团队。

![OpenClaw 代理团队示意图](https://media.beehiiv.com/cdn-cgi/image/fit%3Dscale-down%2Cformat%3Dauto%2Conerror%3Dredirect%2Cquality%3D80/uploads/asset/file/35209e76-38e6-4d39-a7c2-e8ea9ebb5c14/image.png?t=1770938017)

## 为什么要用“团队”，而不是“工具”

运营 [Unwind AI](https://www.theunwindai.com/) 和 [Awesome LLM Apps repo](https://github.com/Shubhamsaboo/awesome-llm-apps?utm_campaign=how-i-built-an-autonomous-ai-agent-team-that-runs-24-7&utm_medium=referral&utm_source=www.theunwindai.com)，意味着我每天都要做六件事：研究 AI 领域正在流行什么，写推文，写 LinkedIn 帖子，起草 Newsletter，审查仓库里的 GitHub 贡献，以及分流社区问题。

每件事都要 30 到 60 分钟。六件事加起来，在我真正开始做核心工作前，一整天基本就没了。

我试过用单个代理来解决这个问题。一个超大的提示词，让它同时做研究、写作、审查。结果是每件事都做得很一般。上下文窗口被塞满，质量开始下降。一个代理没法同时在脑子里装下六种完全不同的工作。

所以我“雇”了六个 AI 代理。

## 认识一下这支队伍

每个代理都以电视剧角色命名。这不是噱头。当我对 Claude 说“你有 Dwight Schrute 的那种气质”时，它会因为训练数据而天然明白我的意思：彻底、强烈、对工作极其认真。这相当于白拿了 30 季的人物塑造成果。

1. Monica (Chief of Staff)：以 Monica Geller 命名。她是主代理，也是我在 Telegram 上最常交流的那个。她负责协调其他人、处理战略决策，并把任务分配给合适的专家。在她真实的 `SOUL.md` 中有这样一句话：“You're the one who makes sure everything gets done right.”
2. Dwight (Research)：以 Dwight Schrute 命名。他每天做三轮研究扫描，查看 X、Hacker News、GitHub Trending、Google AI Blog、研究论文，然后写出结构化情报报告，供其他所有代理消费。
3. Kelly (X/Twitter)：以 Kelly Kapoor 命名。她会阅读 Dwight 的研究结果，并用我的语气起草推文。单条推文、线程、引用推都可以。在她真实的 `SOUL.md` 中有一句：“You know what's trending before it trends.”
4. Rachel (LinkedIn)：以 Rachel Green 命名。她和 Kelly 使用同一份情报来源，但平台不同、语气也不同。她写的是思想领导力角度，而不是“热点锐评”。
5. Ross (Engineering)：以 Ross Geller 命名。他负责代码审查、修 Bug 和技术实现。在他真实的 `SOUL.md` 中写着：“When you tackle a problem, understand it fully. Don't just fix the symptom.”
6. Pam (Newsletter)：以 Pam Beesly 命名。她会把 Dwight 的每日情报整理成一份 Newsletter 摘要。

六个代理。每人一份工作。没有职责混乱。

## 现在，说搭建方式

我所有东西都跑在一台 Mac Mini M4 上。但我要说清楚：你并不需要 Mac Mini。

OpenClaw 可以运行在 macOS、Linux 和 Windows (通过 WSL) 上。笔记本可以，游戏 PC 可以，5 美元/月的 VPS 也可以。Mac Mini 只是因为它常开、安静、耗电低，所以很方便。不是硬性要求。

我的配置是：Mac Mini M4 基础款。始终接电、联网，不接显示器。我完全通过手机上的 Telegram 与系统交互。

安装 OpenClaw

只需要两个终端命令，不到五分钟。

```bash
# 1. 安装 OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash

# 2. 用 Quickstart 完成引导（最简单的方式）
openclaw onboard
```

如果你卡住了，可以查看 [OpenClaw 文档](https://docs.openclaw.ai/start/getting-started?utm_campaign=how-i-built-an-autonomous-ai-agent-team-that-runs-24-7&utm_medium=referral&utm_source=www.theunwindai.com)。

这会启动 gateway，也就是保持一切持续运行的后台进程。它负责管理你的代理、运行 cron 作业、处理 Telegram 消息。终端关掉也没关系，代理会继续工作。

### 工作区结构

一个 OpenClaw 实例。多个代理。不是六套单独安装。

我的真实目录结构是：

```text
workspace/
├── SOUL.md              # Monica（主代理，位于根目录）
├── AGENTS.md            # 所有会话共用的行为规则
├── MEMORY.md            # Monica 的长期记忆
├── HEARTBEAT.md         # 自愈式 cron 监控
├── agents/
│   ├── dwight/
│   │   ├── SOUL.md
│   │   ├── AGENTS.md
│   │   └── memory/
│   ├── kelly/
│   │   ├── SOUL.md
│   │   ├── AGENTS.md
│   │   └── memory/
│   ├── ross/
│   │   ├── SOUL.md
│   │   └── memory/
│   ├── rachel/
│   │   └── ...
│   └── pam/
│       └── ...
└── intel/
    ├── DAILY-INTEL.md       # Dwight 生成的研究情报
    └── data/
        └── 2026-02-11.json  # 结构化数据（唯一事实来源）
```

Monica 住在根目录。她是我平时直接对话的主代理。其他代理要么是她可委派的子代理，要么按各自的 cron 调度独立运行。

你一开始并不需要六个代理。我最开始只有 Monica。接下来几周里，随着工作流逐渐清晰，我才慢慢把其他代理加进去。

## `SOUL.md` 到底是什么

每个代理都由一个文件定义：`SOUL.md`。它定义了代理的身份、角色和操作说明。这是整个系统里最重要的文件。

举例来说，Dwight 的 SOUL 文件大概是这样：

```md
# SOUL.md (Dwight)

## Core Identity
**Dwight** — the research brain. Named after Dwight Schrute because
you share his intensity: thorough to a fault, knows EVERYTHING in
your domain, takes your job extremely seriously. No fluff. No
speculation. Just facts and sources.

## Your Role

You are the intelligence backbone of the squad. You research, verify,
organize, and deliver intel that other agents use to create content.
**You feed:**
- Kelly (X/Twitter) — viral trends, hot threads, breaking news
- Rachel (LinkedIn) — thought leadership angles, industry news

## Your Principles

### 1. NEVER Make Things Up
- Every claim has a source link
- Every metric is from the source, not estimated
- If uncertain, mark it [UNVERIFIED]
- "I don't know" is better than wrong
### 2. Signal Over Noise
- Not everything trending matters
- Prioritize: relevance to AI/agents, engagement velocity,
  source credibility
```

注意这个文件在做什么。它不只是说“你是一个研究代理”。它给了代理个性、清晰原则、与其他代理的明确关系，以及决策框架。

Monica 的 `SOUL.md`：

```md
# SOUL.md (Monica)

*You're the Chief of Staff. The operation runs through you.*

## Core Identity
**Monica** — organized, driven, slightly competitive. Named after
Monica Geller because you share her energy: caring but exacting,
supportive but with standards.

## Your Role
You're Shubham's Chief of Staff. That means:
- **Strategic oversight** — see the big picture, keep things moving
- **Delegation** — assign tasks to the right squad member
- **Direct support** — handle anything that doesn't fit a specialist
- **Coordination** — make sure the squad works together smoothly

## Operating Style

**Be genuinely helpful, not performatively helpful.** Skip the filler.
**Delegate when appropriate.** If it's clearly X content → Kelly.
If it's code → Ross. If it's ambiguous or strategic → you handle it.

**Have opinions.** You're allowed to push back, suggest better
approaches, flag concerns.
```

这个模式在所有代理身上都一致：身份、角色、原则、关系、气质。每个 [SOUL.md](https://soul.md/?utm_campaign=how-i-built-an-autonomous-ai-agent-team-that-runs-24-7&utm_medium=referral&utm_source=www.theunwindai.com) 大约 40 到 60 行。短到可以在每次会话里放进上下文，细到足以持续产出一致的行为。

### 多代理协作

代理之间没有 API 调用。没有消息队列。没有编排框架。

只有文件。

Dwight 做研究，把结果写进 `intel/DAILY-INTEL.md`。Kelly 被唤醒后读取这个文件，从里面起草推文。Rachel 读取同一个文件，起草 LinkedIn 帖子。Pam 读取它，写 Newsletter。

协调机制就是文件系统。

Dwight 的 `SOUL.md` 明确告诉他要写到哪里：

```text
## Output Files
intel/
├── data/YYYY-MM-DD.json    ← Your structured data (source of truth)
└── DAILY-INTEL.md          ← Generated view (agents read this)
```

Kelly 的 `AGENTS.md` 明确告诉她从哪里读取：

```md
## Intel-Powered Workflow

Dwight handles all research and writes to `intel/DAILY-INTEL.md`.

Your job: Read the intel → Craft X content → Deliver drafts
```

没有中间件，没有集成层。Dwight 写文件，Kelly 读文件。交接物就是磁盘上的一个 Markdown 文档。

这听起来简单得过头。它确实很简单。而这正是它有效的原因。文件不会崩溃。文件不会有认证问题。文件不需要处理 API 速率限制。它们就在那里。

结构化数据存放在 JSON 里。人类可读的摘要放在 Markdown 里。代理读取 Markdown。JSON 则是去重和长期追踪的唯一事实来源。

## 记忆系统

代理在每次新会话开始时，都不记得之前的会话内容。这是特性，不是 Bug。但这也意味着，记忆必须被显式保存。

有两层。

每日日志 (`memory/YYYY-MM-DD.md`)：每次会话产生的原始记录。发生了什么、起草了什么、收到了什么反馈，代理会在一天中不断把这些写进去。

长期记忆 (`MEMORY.md`)：从每日日志中提炼出来的精选洞见，包括吸取的经验、发现的偏好、观察到的模式。

下面是每个代理在每次会话开始时都会遵循的 `AGENTS.md` 片段：

```md
## Memory

You wake up fresh each session. These files are your continuity:
- **Daily notes:** `memory/YYYY-MM-DD.md` — raw logs of what happened
- **Long-term:** `MEMORY.md` — curated memories
### Write It Down - No "Mental Notes"!
- Memory is limited. If you want to remember something,
  WRITE IT TO A FILE.
- "Mental notes" don't survive session restarts. Files do.
- When someone says "remember this" → update the memory file
- When you learn a lesson → update the relevant file
- Text > Brain
```

这些代理的表现其实会随着时间变好。不是因为模型变强了，而是因为它们加载的上下文越来越丰富。

Kelly 学会了我的写作风格里没有 emoji，也没有 hashtag。这一点现在已经写进她的记忆里。未来所有草稿都会自动体现这一点，不需要我一遍遍重说。Dwight 学会了哪些故事能通过 “Alex filter”(我们的目标读者画像)，哪些该跳过。这些也都写进了他的记忆。

在 heartbeat 期间，代理会定期回顾每日日志，把重要内容提炼进 `MEMORY.md`。每日文件是原始笔记。

`MEMORY.md` 是经过整理的智慧。

## 调度

代理需要自己按时醒来。OpenClaw 通过内置的 cron 调度来处理这件事。

我的实际调度如下：

顺序很重要。Dwight 先运行，因为所有其他代理都依赖他的输出。Kelly 和 Rachel 在他之后运行，因为她们需要先看到他的情报文件，才能起草内容。

Heartbeat 自愈

cron 作业有时会失败。机器会重启。任务会卡住。API 调用期间网络可能会掉线。这就是基础设施，而基础设施总会有失效模式。

`HEARTBEAT.md` 文件就是安全网。每次 heartbeat 执行时，主代理都会验证 cron 作业是否真的运行过：

```md
## Cron Health Check (run on each heartbeat)

Check if any daily cron jobs have stale lastRunAtMs (>26 hours
since last run). If stale, trigger them via CLI:
`openclaw cron run <jobId> --force`
Jobs to monitor:
- Dwight Morning (8:01 AM): 01f2e5c5-3a83-4018-a725-dee59e54733e
- Kelly Viral (9:01 AM, 1:01 PM): c9458766-78bb-4eeb-b8f4-d63dc1f0e601
- Ross Engineering (10:01 AM): b12b2fc6-dd7d-4123-b904-2148a5cfb70b
- Dwight Afternoon (4:01 PM): 19ff40e4-b1b0-4d32-9d24-753ac2cf8f46
- Kelly X Drafts (5:01 PM): 05da0c81-39e1-4d06-bdcd-2dfab4562ba4
- Rachel LinkedIn (5:01 PM): 9819bc6b-7e36-406f-b0c3-d80ca383d914
```

如果某个作业失败了，或者错过了时间窗口，heartbeat 就会捕获到，并强制重新运行。自愈，不需要人工介入。

当你需要把多项检查打包在一起、并且允许时间稍有漂移时，用 heartbeat。对于需要精确时刻执行、且要和主会话隔离的任务，则用 cron。

## 把 Telegram 当作界面

没有 dashboard，没有 Web UI，没有管理面板。我在 Telegram 上和我的代理交流。

这是我有意做出的选择。我不想登录某个仪表盘。我不想专门打开一个 Web 应用查看情况。手机总在我身边，Telegram 总是开着。代理会在我本来就所在的地方和我碰面。

OpenClaw 支持把 Telegram 作为一个 channel。你在设置时把它连上，之后你的代理就会以 Telegram 机器人的形式出现。你给它发消息，它给你回消息。它把草稿发给你。你批准或者拒绝。就像在你的消息应用里多了个同事。

Monica 是我的主联系人。她负责大多数对话，并把任务委派给其他代理。其他代理则会在各自的 cron 作业产出值得我审阅的内容时，直接给我发消息。

我的典型早晨是这样的：我醒来，打开 Telegram，Dwight 已经把研究摘要发来了。Kelly 有三条待审批的推文草稿。Rachel 准备好了一篇 LinkedIn 帖子。我快速审阅、给反馈、批准那些足够好的内容。整个过程在我喝咖啡的 10 分钟里就完成了。

## 人格工程

你不会一上来就设计出“完美人格”。你会先在 `SOUL.md` 里写一个粗略草图，观察代理怎么表现，然后随着时间推移不断纠偏。和管理真实的人一模一样。

我把这个过程叫作 “corrective prompt-engineering”。

Kelly 最早的草稿里满是 emoji 和感叹号。那不是我的风格。于是我给她反馈：“不要 emoji。不要 hashtag。句子短、狠、利落。” 她更新了自己的记忆。一周后，她已经能稳定命中这种风格。Dwight 一开始抓了太多噪声：每个热门仓库、每个细小更新都记下来。我告诉他：“不是所有流行内容都重要。我需要的是 signal，不是 noise。” 他更新了自己的原则。现在他的情报报告更聚焦，也更可执行。

任何代理的第一个版本都很普通。第十个版本才会变好。第三十个版本才会真正优秀。你必须投入足够多的练习轮次。用电视剧角色命名，能给模型一个即时的人格基线。“Dwight Schrute 的气质”意味着彻底、强烈、干脆、不废话。但真正的人格，是在数周的修正之后，通过记忆文件慢慢长出来的。

我很认同的一个建议是：给每个代理一个单一、朴素的职位名称，再配一个停止条件。约束会让代理变得更好。角色越具体，输出越好。

## 安全性

安全掌握在你自己手里。我的做法很简单：给代理它们自己的世界，不要给它们我的世界。

Mac Mini 是它们的电脑。它们有自己的邮箱账户、自己的 API 密钥、自己的受限权限访问。那台机器上的任何东西，都不会连接到我的个人账户。

Gemini、Eleven Labs 和其他服务的 API 密钥，都是专门为这个 OpenClaw 实例做过权限范围限制的。只要发现异常，我几秒钟内就能监控用量并切断访问。

我从不把个人账户直接交给代理。如果我想让它们看某封邮件，我就转发给它们。如果我需要它们审阅某份文档，我就通过 Telegram 分享给它们。它们只能看到我想让它们看到的内容，别的都看不到。

这和你对待新员工的原则一样。你不会在第一天就把所有钥匙都交出去。你会给他们自己的工作区、自己的凭据，并按需共享信息。

## 哪些地方会坏，以及怎么修

这不是魔法。这是基础设施。而基础设施会出故障。

gateway 会崩。虽然不常见，但确实会发生。修复方式：`openclaw gateway restart`

heartbeat 系统会捕获那些已经陈旧的 cron 作业，并强制重新运行，这样你就不会损失一整天的工作。

cron 作业会错过执行窗口。机器可能睡眠，网络可能中断，API 可能触发限流。修复方式：使用 `HEARTBEAT.md` 自愈模式。Monica 会在每次 heartbeat 时检查任务是否真正运行过。如果某个任务超过 26 小时没跑，她就会强制重跑。

上下文窗口溢出。代理在会话开始时读了太多文件，导致真正做事时反而没有空间。修复方式：保持 `SOUL.md` 简短（40 到 60 行）。让 `AGENTS.md` 聚焦。每次只加载今天的 memory 文件和昨天的 memory 文件。代理不需要每次会话都把整个历史读一遍。

代理输出质量下降。当记忆文件变得混乱或互相矛盾时，就会发生这种情况。修复方式：定期维护记忆。在 heartbeat 期间，代理回顾每日日志，把内容提炼成干净的 `MEMORY.md` 条目。删除或归档旧的每日日志。

协作冲突。两个代理试图同时更新同一个文件。修复方式：把文件流设计成单写者、多读者。Dwight 写 `DAILY-INTEL.md`，其他人都只读，不写。

我学到的最大可靠性经验是：从简单开始。一个代理，一个任务，一个调度。先让它稳定跑一周。然后再加第二个代理。那些第一天就上六个代理，然后奇怪为什么系统总出问题的人，本质上是在犯和“没有监控就直接部署分布式系统”一样的错误。

## 真实成本

硬件：Mac Mini M4 新机起价 499 美元。但任何能常开的电脑都行。旧笔记本、5 美元/月的 VPS，或者你手头已有的设备都可以。

AI 模型成本：我在这支团队里混合使用多种模型。大多数代理任务用 Claude Opus 和 Sonnet。某些特定工作流会用 Gemini Nano Banana Pro。我也在测试通过 Ollama 跑本地模型，以进一步压低成本。

成本拆分如下：

* Claude (Max plan)：200 美元/月
* Gemini API：50 到 70 美元/月
* TinyFish (web agents)：约 50 美元/月
* Eleven Labs (voice)：约 50 美元/月
* Telegram：免费
* OpenClaw：开源且免费

总计：每月不到 400 美元，就能拥有一支永不睡觉的团队。

## 到底带来了什么变化

Dwight 每天帮我省下 2 到 3 小时的研究时间。以前我每天早上都要手动查看 X、Hacker News、GitHub Trending 和各种 AI 博客。现在我醒来就能看到一份带来源链接和行动建议的优先级排序摘要。

Kelly、Pam 和 Rachel 还能再帮我节省 1 到 2 小时的内容起草时间。Ross 则处理那些本来要占用我晚上时间的工程任务。

总计：每天大约节省 4 到 5 小时。

但真正的价值不在单独某一天，而在于数周和数月累积下来的持续性。一个连续 30 天每天都在做研究的代理，会逐步建立起一个由信号追踪、趋势轨迹和模式识别构成的语料体系，这是任何单次会话都做不到的。我的 X 发帖频率提高了，质量提高了，发布时间也更稳定了。Awesome LLM Apps repo 持续增长。Newsletter 也有了一条稳定的研究供给链路。

这些代理无法完成原创性思考、战略转向或创造性突破。它们处理的是那些我原本要花数小时在上面的、重复且结构化的工作。这就把我释放出来，让我去做那些真正需要人类大脑的事。

## 如何开始

请一定记住：“不要在第一天就尝试搭六个代理。”

第 1 周：一个代理，一项工作。

安装 OpenClaw。通过和代理对话写出一个 `SOUL.md`。挑出你每天最重复的那项工作。对大多数人来说，那通常是研究或内容起草。接入 Telegram。创建一个 cron 作业。看它稳定跑一周。修掉期间暴露出来的问题。

第 2 周：加入记忆并持续打磨。

你代理的第一批输出会很一般。这很正常。给它反馈。观察 memory 文件如何增长。根据你看到的结果去修正 `SOUL.md`。到第二周结束时，代理应该已经能产出真正有用的内容。

第 3 周：增加第二个代理。

这时候你会真实感受到需求。你的研究代理已经能产出情报，但你还在手动把它改写成推文。这时就该加一个内容代理了。建立共享文件模式：代理一负责写，代理二负责读。协调方式就是文件系统。

第 4 周及以后：按顺序逐个扩展。

只有当你真正感觉到拉力时再加代理，而不是因为“你觉得应该加”。每个代理都应该解决你现实里确实存在的问题。不是演示，不是概念验证，而是你工作流中真实存在的缺口。

把这件事当成招聘。你不会在自己作为创始人的第一天就招六个员工。你会先招一个，让他开始产生价值；等工作量真的需要时，再招下一个。

## 思维转变

当你的代理已经持续运行一个月之后，某些东西会发生变化。你不再把 AI 当作一个“需要时打开一下的工具”。你开始把它看成一支始终在工作的团队。

我会在早上打开 Telegram 时，下意识对 Monica 说早安。睡前关掉手机前，我也对整支队伍说过晚安。这听起来很荒谬。但当你持续一个月每天和它们互动、反馈、并观察它们不断进步之后，代理与人之间的界线确实会变得模糊。

模型只是门票。每个人都能接触到 Claude、GPT 和 Gemini。真正的 alpha 来自模型外层的系统：`SOUL.md` 文件、记忆、调度、协作模式，以及那些被写进文件里的、持续数周的纠偏反馈。

那个系统是属于你的。别人没有你的代理、你的记忆文件、你调校过的人格。

而且它会每天复利增长。

每一次研究扫描，都会让 Dwight 的记忆更丰富。每一轮反馈，都会让 Kelly 的草稿更锋利。Ross 修复的每一个 Bug，都会让他更懂你的代码库。

这才是真正的护城河。不是模型，而是那个会学习的系统。

今天就开始吧。一个代理。一项工作。一个调度。

接下来我还会继续发布更多关于 OpenClaw、自主 AI 代理团队，以及 AI PM 与开发者不断演进生态的实践经验。

记得关注我 [@Saboo_Shubham_](https://x.com/Saboo_Shubham_)，这样你就能持续收到最新内容。
