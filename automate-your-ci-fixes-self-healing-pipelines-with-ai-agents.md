# 自动修复你的 CI：用 AI 代理打造自愈式流水线

> 原文: https://dagger.io/blog/automate-your-ci-fixes-self-healing-pipelines-with-ai-agents/
> 作者: Kyle Penfound
> 日期: 2025-04-23

---

![Automate Your CI Fixes: Self-Healing Pipelines with AI Agents](https://dagger.io/assets/blog/automate-your-ci-fixes-self-healing-pipelines-with-ai-agents/hero.png)

#### 停止手动修 CI 故障：用 Dagger 构建一个 AI 代理

如果你的 CI 流水线不只是告诉你哪里坏了，而是真的帮你把它修好，会怎样？想象一下：你刚推送完一次改动，看到一个 lint 错误冒出来；片刻之后，你就在自己的 pull request 里看到了一个可以直接提交的修复建议，已经把问题改好了。这不是科幻，而是今天就能实现的事情，只要把 AI 代理与 Dagger 的可组合能力结合起来。

#### 我们都很熟悉的 CI 失败之痛

持续集成（Continuous Integration, CI）系统就像代码库尽职尽责的守门人。它们会运行 linter、执行测试、做安全扫描，总体上确保所有传入变更都符合项目标准。我们依赖它们在代码合并之前帮我们发现错误。

不过，一旦 CI 标记出某个问题，比如烦人的 lint 违规或测试失败，负担就又回到了开发者身上。即便报错信息已经很清楚，这个过程通常仍然很繁琐：

1. 阅读并理解：解析 CI 输出，定位到底哪里出了问题。
2. 切换上下文：拉取最新改动，再回到编辑器。
3. 实施修复：对代码做必要修改。
4. 提交并推送：暂存这些改动并重新推上去。
5. 等待：再看一遍 CI 流水线跑起来，祈祷这次能全部变绿。

这个循环会消耗宝贵时间，也会打断开发流程。如果我们能把整个过程直接在 CI 里自动化掉，会怎样？

#### 解决方案：一个能自动修复问题的 AI 代理

这篇文章介绍了一个用 Dagger 构建的 AI 代理，它能在 CI 环境中自动处理 lint 和测试失败。当 pull request 中检测到失败时，这个代理会：

1. 分析失败输出。
2. 使用工具与项目代码库和测试套件交互。
3. 反复尝试修复，并重新运行测试或 linter 来验证结果。
4. 生成一份包含已验证修复内容的代码 diff。
5. 直接把这些修复作为代码建议发布到 pull request 中，让开发者只需点一下就能审查并接受。

![](https://dagger.io/assets/blog/automate-your-ci-fixes-self-healing-pipelines-with-ai-agents/image-1.webp)

#### 技术深潜：如何用 Dagger 构建这个代理

我们当然可以尝试把整个项目源码和失败日志一股脑喂给大语言模型（Large Language Model, LLM），然后让它“把问题修好”。但这种方式通常过于宽泛。LLM 面对过量上下文或过多可执行动作时，往往会给出不可靠甚至荒谬的结果。反过来，如果你只提供错误信息，再让它吐出一段代码片段，又缺少足够的上下文和工具，LLM 也无法有效验证自己的建议。

关键在于找到合适的平衡点：既给代理足够的上下文和能力，又不让它被信息淹没。这正是 Dagger 擅长的地方。

1. 核心代理：`DebugTests` 这个 Dagger Function

整个方案的核心，是一个名为 `DebugTests` 的 Dagger Function。我们并没有让 LLM 在整个容器文件系统里自由发挥，而是定义了一个专注于当前任务的受限环境。这个环境通过项目中的一个自定义 Dagger Module 来实现，我们可以把它叫作 `Workspace`。

`Workspace` 模块封装了代理所需的具体动作：

* `ReadFile(path string) (string, error)`：读取指定文件的内容。
* `WriteFile(path, content string) error`：向文件写入新内容。
* `ListFiles(path string) ([]string, error)`：遍历目录结构。
* `RunTests() (string, error)`：执行项目测试套件（或相应检查），并返回输出。
* `RunLint() (string, error)`：执行项目的 linter，并返回输出。

关键的一点是，因为我的项目本来就已经做了 Dagger 化（Daggerized），`Workspace` 模块里的 `RunTests` 和 `RunLint` 函数，其实只是去调用开发者和 CI 原本就在使用的那些 Dagger Functions！我们不需要为了代理重新实现测试执行逻辑；只需要把现有的 Dagger 动作作为工具暴露出来即可。

2. 创建代理运行环境

在 `DebugTests` 函数内部，我们会实例化这个 `Workspace`，并基于它创建一个 Dagger `Environment`。这个 `Environment` 实际上会把 `Workspace` 的这些函数（例如 `ReadFile`、`WriteFile`、`RunTests`、`RunLint`）作为 LLM 可调用的“工具”暴露出来。

然后，我们把这个 `Environment` 连同一段精心设计的提示词（prompt）一起提供给 LLM。提示词会告诉 LLM 它的目标是什么（修复初始输入中给出的测试或 lint 失败），解释它有哪些可用工具（即 `Workspace` 里的函数），并引导它进行推理。

接下来，代理会在一个循环中运行：

* 接收最初的失败输出。
* 使用 `ReadFile` 和 `ListFiles` 工具理解相关代码。
* 假设一种修复方式，并通过 `WriteFile` 将其应用到代码中。
* 使用 `RunTests` 或 `RunLint` 检查修复是否生效。
* 如果仍然失败，就分析新的输出并重复这一过程。
* 一旦所有失败都被解决，代理就会计算原始源码与修改后源码之间的 diff，并将其返回。

`DebugTests` 函数负责协调整个交互过程，最终返回经过验证的 `diff`。

3. CI 集成：从 Diff 到 Pull Request 建议

虽然 `DebugTests` 是核心代理逻辑，但真正负责接入 CI 的，是第二个 Dagger Function。这个函数会在 CI 工作流中的 lint 或测试阶段失败时被触发。

它会执行以下步骤：

1. 利用 Dagger 的 Git 能力获取 pull request 的源码。
2. 调用 `DebugTests` 函数，并传入源码和捕获到的失败输出。
3. 接收代理返回的、包含修复内容的 `diff`。
4. 使用平台 API（例如 GitHub API），把这个 `diff` 格式化后发布为 pull request 相关代码行上的代码建议。

这些内容并不只是普通评论，而是绑定到具体代码行上的可执行建议。开发者可以逐条审查这些变更，并点击 “Commit suggestion” 直接应用它们，大幅简化修复流程。

![](https://dagger.io/assets/blog/automate-your-ci-fixes-self-healing-pipelines-with-ai-agents/image-2.png)

#### 总结：用 Dagger 和 AI 打造更聪明的 CI

借助 Dagger，我们构建了一个能够自动修复 lint 和测试失败的 AI 代理。关键在于：通过自定义 `Workspace` 模块设计出一个聚焦的运行环境，把特定能力（文件 I/O、测试执行）作为工具暴露给 LLM。由于项目原本就已经把测试和 lint 定义成了 Dagger Functions，代理可以直接复用这些能力，这也展示了 Dagger 在组合与复用方面的强大优势。

把这个代理集成进 CI，会显著改变开发者体验。开发者不再需要手动排查并修复那些常规错误，而是能在 pull request 中直接收到已经验证过、可立即提交的修复建议。这既节省时间，也减少了上下文切换，让工程师能更专注于功能开发，而把这些纠错类琐事交给 Dagger 和 AI 去处理。

我很期待继续探索这个方向，也很想听听你们在开发工作流中使用 AI 代理的经验。你们有没有实现过类似方案？过程中遇到过哪些挑战？欢迎到 [Dagger Discord](https://discord.com/invite/dagger-io) 一起讨论。你也可以在 [这里](https://github.com/kpenfound/greetings-api/blob/main/DEBUGGER_AGENT.md) 查看我演示项目的代码。
