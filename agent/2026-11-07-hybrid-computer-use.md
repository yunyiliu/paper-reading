# Hybrid Computer Use: A Unified Programmatic, Semantic, and Visual Interface for General Agents

**AI用电脑不必学人类**

- 论文 Paper: [arXiv 2610.41432](https://arxiv.org/abs/2610.41432)
- 单位 Affiliation: 微软研究院
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-07
- 视频 Videos: [YouTube 中文](https://youtu.be/x0X_x31qcxo) · [YouTube English](https://youtu.be/emDMygWu5H8) · [B站](https://www.bilibili.com/video/BV1vm4s6hEFQ)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇微软研究院的论文，合作单位还有复旦大学和清华大学，名字叫 Hybrid Computer Use，简称 HCU，意思是混合式的电脑操作。

它想回答一个问题，让 AI Agent 操作电脑，是不是一定要像人一样，看屏幕、点鼠标、敲键盘。

### 问题

先说背景。computer-use Agent，就是让大模型直接操作电脑软件，替你把事做完。常用的评测基准叫 OSWorld，到了 2.0 版本，任务已经从日常桌面操作，扩展到很长的专业软件工作流。

任务一长，效率就和成功率一样重要。可现在大多数接口，都在模仿人的用法，先看截图，找到按钮在哪，再用鼠标键盘去点。

作者认为，这等于给模型加了一个人类交互的先验。比如一个输入框，软件明明已经告诉系统它叫什么、填了什么，Agent 还要靠屏幕坐标去找。本来写代码就能处理的数据，Agent 却在反复点击拖拽。

### 以前的方法

其实电脑本来就提供三种交互通道。第一种是视觉通道，就是图形界面，看截图、点鼠标。第二种是程序通道，用代码和命令行，直接读写文件和数据。第三种是语义通道，叫 accessibility，简称 AX，也就是无障碍接口。

AX 原本是给读屏软件这类辅助技术用的，它把软件界面里的控件，连同名字、当前的值、能做哪些操作，整理成一棵结构化的树，现在也被拿来做自动化。

以前的工作，有的把代码和截图结合起来，但很少用到 AX。也有系统三种都支持，比如开源的 CUA Driver，但它优先在后台用 AX 操作，为的是不打扰用户正在用的桌面。CUA Driver 也是这篇论文主要的对比基线。

### 核心想法

这篇论文的想法很直接，把三种通道都交给模型，让模型自己决定什么时候用哪一种，怎么组合。

这张图是整体结构。HCU 不加规划器，不加子 Agent，也不改 Agent 的循环。它通过 MCP 协议，给现有的 Agent 多接两个桌面工具，一个管 AX，一个管截图和鼠标键盘，代码和命令行还用 Agent 原来的工具。所以它能直接接到不同的 Agent 框架上。

另外还配了一份使用说明，论文叫 skill，告诉模型每种通道能做什么，什么时候最合适。

### 方法

先看 AX 通道。直接用 AX 有三个难点。第一，覆盖不全，暴露哪些信息由软件自己决定。第二，控件会变，可能重名、移动、消失，或者被重建。第三，系统报告动作执行了，不代表界面真的变成了想要的样子。

HCU 的第一招是精简视图。它在内部保留完整的 AX 记录，给模型看的却是压缩过的版本，去掉空的包装层，合并重复的内容。每个控件有一个短编号，编号在压缩之前就绑定好了，所以精简不会让编号指错对象。

压缩有多明显，附录里有个例子。同一个 Obsidian 笔记软件的窗口，OSWorld 2.0 自带的 AX 数据是 30526 个 token，HCU 返回给模型的文本，只有 1237 个。

第二招是批量动作加等待。模型可以在一次调用里连着做几个动作，比如点击、输入、回车，中间还能插入等待步骤，等复选框真的被勾上了，再往下走。中途出错或者超时，就立刻停下，告诉模型停在了哪一步。

第三招是动作之后直接给反馈。动作执行完，HCU 默认返回一份 AX diff，就是界面前后哪里变了，哪个值改了，哪个窗口开了或者关了。这样模型不用再专门花一步去看一眼。

它还会区分两种情况，输入确定没有送达，和输入可能送达了、但效果未知，不会把发出了输入，当成已经成功。

三种通道作用在同一台电脑上，就带来一个新问题，一个通道的动作，会让另一个通道手里的信息过期。比如用 AX 打开了一个对话框，正好盖住了之前截图里要点的位置，模型照着老坐标点下去，就点错了地方。

HCU 的办法叫动作守卫。AX 动作执行前，先按编号找到当前真实的控件，不见了、有歧义、或者被替换了，就拒绝执行，也不会换成一个同名的控件。鼠标点击之前，检查窗口布局和上一张截图是否一致，不一致就不点，返回一张新截图，让模型重选坐标。

最后是那份 skill。它是作者从开发过程里反复出现的错误中总结出来的，比如参数格式写错、编号用错。论文强调，里面没有任何针对具体任务的解法。

### 实验和效果

实验设置。主实验在 OSWorld 2.0 的全集上，一共 108 个任务，每个任务最多 500 步。指标有两个，Partial 是平均得分，Binary 是拿到满分的任务比例。

模型方面，GPT-6 Astra 跑在 Codex 里，Claude Opus 5 跑在 Claude Code 里，都用各自最高的推理强度。对照基线是同样用 GPT-6 Astra、同样在 Codex 里跑的 CUA Driver。

结果在表 1。HCU 配 GPT-6 Astra，Partial 是百分之 72.37，是论文对比的全集结果里最高的。HCU 配 Claude Opus 5，Binary 是百分之 35.19，也是最高。

和同样设置下的 CUA Driver 比，Partial 高 3.55 个百分点，Binary 只高 0.9 个点，工具调用次数少了大约 30%，平均用时少了 21%，每个任务的成本几乎一样，都是 10 美元出头。

配 Claude Opus 5 的时候，和这个模型在官方排行榜上的成绩比，Partial 和 Binary 都高了 3.7 个点多，成本低了 20%。

在性能和成本的这张图上，HCU 配 GPT-6 Astra，和 StateAct 一起构成了帕累托前沿。StateAct 每个任务大约 7.8 美元，更便宜，但 Partial 只有 61.6。帕累托前沿的意思是，其他系统都没法在分数和成本上同时胜过它们。

作者还在另外三个平台上，用 GPT-5.4 测了后来发布的版本。Linux 上的 WeaveBench，是 53.25 对 46.37，Windows Agent Arena 是 80.86 对 71.89，macOS 上一个 28 个任务的子集，多完成了 1 个，每个任务的成本都低了 46% 到 51%。

### 和以前方法的对比

接下来是拆解实验，看每个部件的贡献，在 OSWorld 1.0 上，用 GPT-5.4，每种配置跑三次取平均，结果在表 2。完整的 HCU 比 CUA Driver 高 3.29 分，步数少了大约 30%，输入 token 少了 57%，成本少了 55%。

贡献最大的是 AX。在代码和图形界面之外加上 AX，分数提高 2.79 分，步数、token、时间和成本都下降。去掉图形界面，分数变化还在运行间的波动范围内，但步数多了 12%。也就是说，分数提升主要来自 AX，截图和鼠标主要帮着省步数。

skill 的作用也很明显。有了它，AX 和图形界面的调用里，因为参数格式错误被拒绝的比例，从百分之 17.37 降到百分之 1.36。

再看换模型。表 3 在 OSWorld 1.0 上比了四个模型，GPT-5.4、Claude Opus 5.5、Gemini 3.8 Flash 和 Grok 4.7。每个模型用了 HCU，分数都比 CUA Driver 高，幅度从 2.94 分到 12.90 分，提升最大的是 Gemini。输入 token 少了 18% 到 57%，成本低了 8% 到 55%。例外是 Claude，步数基本没变，用时还多了 6%。

有意思的是，不同模型的用法很不一样。在 HCU 的桌面调用里，GPT 有 80% 走 AX，Grok 是 68%，Gemini 是 51%。Claude 却有 94% 用的是截图和鼠标，而且更喜欢一次批量做好几个动作。

还有一种常见的组合，用截图来看，用 AX 来做。GPT、Gemini 和 Grok 在用到图形界面的任务里，有 29% 到 31% 的任务，图形界面只用来截图，动作全部走 AX。

我的判断是，HCU 最有价值的地方，不是多了几个工具，而是把接口这件事讲清楚了。给模型更合适的通道，再把验证和反馈放进同一次调用，就能同时提高成功率、降低成本，而且不需要重新训练模型。

### 局限

这篇论文没有单独的局限章节，下面几点，都来自论文自己的数字和说明。

第一，AX 的覆盖取决于软件本身。软件不暴露的信息，AX 就看不到，这时候还得退回截图。

第二，主实验里，和同设置的 CUA Driver 比，Binary，也就是拿满分的比例，只高了 0.9 个点，提升主要体现在部分得分上。

第三，表 1 里 Claude Opus 5 官方成绩那一行，系统写的是 computer-use loop，而 HCU 在 Claude Code 里跑，框架不同，不是严格的对照。另外，OpenAI 自己报告的 GPT-6 Astra 是 72.60，比 HCU 还高一点，只是那是离线子集的结果，论文没有放进排名。

第四，主实验一共 108 个任务。拆解实验里，去掉图形界面那一项，三次运行里有两次比完整版略低，有一次反而高了接近 3 分，说明一些小的差别，还在随机波动的范围里。

### 未来方向

作者在结论里说，照着人的习惯设计的接口，会让 Agent 在性能和效率上都吃亏。他们联系到 Rich Sutton 的 bitter lesson，一个固定了电脑该怎么用的、像人的接口，可能会限制越来越强的模型。

下一步，既然 AX 把控件状态暴露成了代码能读写的东西，就可以把桌面当成一个可编程的环境，把桌面操作放进代码执行里，HCU 的批量动作加等待就是基础。论文首页给出了 GitHub 代码地址。

我的判断是，这对做 Agent 产品的人很实用，同样的模型，换个更好的接口，就能多拿几个点，成本往往还更低，而且 Linux、macOS 和 Windows 都有实现。但最好的结果里，拿满分的任务也只有三分之一多一点，长流程的电脑操作，离真正可靠还很远。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Microsoft Research, with co-authors at Fudan and Tsinghua. It's called Hybrid Computer Use, or HCU.

The question behind it: when an AI agent operates a computer, does it really have to do it the way we do, by looking at the screen and moving a mouse?

### Problem

Some background first. A computer-use agent is a model that drives desktop software directly to get a task done. The standard benchmark is OSWorld, and version two point zero has moved from everyday desktop chores to long workflows in professional software.

When tasks get that long, efficiency matters as much as success. Yet most interfaces copy the human routine: take a screenshot, find the button, click it.

The authors call this a human-interaction prior. An app may already report a text field's name and value, and the agent still hunts for it by screen coordinates. Data that a few lines of code could process gets handled with click after click and drag after drag.

### Prior work

A computer actually offers three ways in. The visual channel is the graphical interface: screenshots, mouse and keyboard. The programmatic channel is code and the command line, working on files and data directly. And the semantic channel is accessibility, usually shortened to AX.

AX was built for assistive technology like screen readers. It exposes an app's controls as a structured tree, with names, current values, and the operations each control supports, and it's now used for automation too.

Earlier systems often combined code with screenshots but made little use of AX. Some support all three, like the open-source CUA Driver, but it favors acting through AX in the background so it doesn't disturb the user's desktop. CUA Driver is the main baseline in this paper.

### Key idea

The idea here is simple. Expose all three channels, and let the model decide which to use, and how to combine them.

This figure shows the design. HCU adds no planner, no sub-agents, and no loop of its own. Through the Model Context Protocol, it gives an existing agent two desktop tools, one for AX and one for screenshots and mouse-and-keyboard input, while code and shell stay with the agent's own tools. That's why it plugs into different agent harnesses.

On top of that comes a guide the paper calls a skill, which tells the model what each channel can do and when it works best.

### Method

Start with the AX channel. Using AX directly has three problems. Coverage is incomplete, because each app decides what to expose, and custom-drawn content may have no accessible element at all. Targets change: controls can share names, move, disappear, or get recreated. And the system reporting that an action ran doesn't mean the interface actually reached the intended state.

HCU's first fix is a compact view. It keeps the full AX records internally, but shows the model a condensed version, with empty wrappers removed and repeated content merged. Each control gets a short reference, bound before the condensing, so shortening the view never changes what a reference points to.

How big is the saving? In one appendix example, the same Obsidian window takes thirty thousand five hundred and twenty-six tokens in OSWorld two point zero's stock accessibility data, and twelve hundred and thirty-seven in HCU's text.

The second fix is batched actions with wait steps. In one call, the model can chain several actions, say, click a search box, type, press Enter, and insert a step that waits until a state actually appears, like a checkbox becoming checked. If something fails or times out, the batch stops and reports where.

The third fix is feedback with the action. By default, HCU returns an AX diff after acting: what changed, which values, which windows opened or closed. So the model doesn't need a separate step just to look.

It also separates two cases: input known not to have been delivered, and input that may have been delivered with an unknown effect. Sending input is never treated as proof of success.

Because all three channels act on the same machine, a new problem appears. An action in one channel can make another channel's evidence stale. Say an AX action opens a dialog right over the spot an earlier screenshot pointed to. Click those old coordinates and you hit the wrong thing.

HCU's answer is action guards. Before an AX action, it resolves the reference against the live element, and if the target is missing, ambiguous, or replaced, it refuses, rather than redirecting to something with the same name. Before a mouse action, it checks that the window layout still matches the last screenshot. If not, it blocks the input and returns a fresh screenshot so the model can pick new coordinates.

Last, the skill. The authors distilled it from mistakes that kept recurring during development, like malformed arguments and misused references. The paper stresses that it contains no task-specific solutions.

### Results

The setup. The main experiment uses the full OSWorld two point zero set, one hundred and eight tasks, with a limit of five hundred steps. Two metrics: Partial is the mean task score, and Binary is the share of tasks with full credit.

For models, GPT-6 Astra runs in Codex and Claude Opus 5 runs in Claude Code, each at its highest reasoning effort. The matched baseline is CUA Driver, also with GPT-6 Astra in Codex.

Results are in Table 1. HCU with GPT-6 Astra reaches seventy-two point three seven percent Partial, the highest among the full-set results compared. HCU with Claude Opus 5 reaches thirty-five point one nine percent Binary, also the highest.

Against the matched CUA Driver, Partial is up three point five five points but Binary only zero point nine. Tool calls drop by about thirty percent, mean time by twenty-one percent, and cost per task is essentially the same, just over ten dollars.

With Claude Opus 5, compared to that model's official leaderboard result, both Partial and Binary are up by more than three point seven points, at twenty percent lower cost.

On this performance-versus-cost chart, HCU with GPT-6 Astra and StateAct together form the Pareto frontier. StateAct is cheaper, around seven dollars eighty a task, but scores sixty-one point six Partial. The frontier means no other system beats them on both score and cost at once.

The authors also tested the later released version on three more platforms with GPT-5.4. On WeaveBench on Linux, it's fifty-three point two five versus forty-six point three seven. On Windows Agent Arena, eighty point eight six versus seventy-one point eight nine. On a twenty-eight-task macOS subset, one more task completed. And cost per task is forty-six to fifty-one percent lower.

### Comparison

Next, the ablations, on OSWorld one point zero with GPT-5.4, three runs per configuration, in Table 2. Full HCU beats CUA Driver by three point two nine points, with about thirty percent fewer steps, fifty-seven percent fewer input tokens, and fifty-five percent lower cost.

AX contributes most. Adding it on top of code and the graphical channel raises the score by two point seven nine points and lowers steps, tokens, time and cost. Removing the graphical channel leaves the score within run-to-run variation but adds twelve percent more steps. So the score gain comes mainly from AX, while screenshots and mouse input mostly save steps when the task needs a visual.

The skill matters too. With it, the share of AX and graphical calls rejected for malformed arguments falls from seventeen point three seven percent to one point three six.

Then the model comparison. Table 3 runs four models on OSWorld one point zero: GPT-5.4, Claude Opus 5.5, Gemini 3.8 Flash, and Grok 4.7. Every one scores higher with HCU than with CUA Driver, by two point nine four to twelve point nine points, with Gemini gaining the most. Input tokens drop eighteen to fifty-seven percent, and cost eight to fifty-five percent. The exception is Claude, whose steps barely change and whose time goes up six percent.

What's interesting is how differently the models use the same tools. Of HCU's desktop calls, GPT sends eighty percent through AX, Grok sixty-eight, and Gemini fifty-one. Claude instead uses screenshots and the mouse for ninety-four percent, and it batches several actions in one call much more often.

A common pattern is to look with screenshots and act with AX. For GPT, Gemini and Grok, on twenty-nine to thirty-one percent of the tasks where they used the graphical channel, every graphical call was a screenshot, and all actions went through AX.

My read is that HCU's real value isn't a few more tools. It's a clear point about interfaces: give the model better channels, put verification and feedback into the same call, and you get higher success and lower cost together, with no model retraining.

### Limitations

The paper has no separate limitations section. The points below all come from its own numbers and statements.

First, AX coverage depends on the app. Whatever an app doesn't expose, AX can't see, and the agent has to fall back to screenshots.

Second, in the main experiment, against the matched CUA Driver, Binary, the full-credit rate, rises only zero point nine points. Most of the gain is in partial credit.

Third, in Table 1, the official Claude Opus 5 row is listed under a computer-use loop, while HCU runs inside Claude Code, so the harnesses differ and it isn't a strict head-to-head. Also, OpenAI's own report for GPT-6 Astra is seventy-two point six, slightly above HCU's seventy-two point three seven, though that was on an offline subset and the paper leaves it unranked.

Fourth, the main experiment has one hundred and eight tasks. In the ablations, removing the graphical channel scored slightly below full HCU in two runs and almost three points above it in the third, so small differences are still within random variation.

### Future work

In their conclusion, the authors argue that interfaces modeled on how people use software, looking at the screen and using a mouse and keyboard, cost agents both performance and efficiency. They connect this to Rich Sutton's bitter lesson: a human-like interface that fixes how the computer is used may limit increasingly capable models.

As a next step, they note that because AX exposes control states that code can read and act on, the desktop can be treated as a programmable environment, with desktop work folded into code execution and HCU's batched actions and wait steps as the foundation. The paper's first page links the code on GitHub.

My read is that this is practical news for anyone building agent products. Same model, better interface, a few more points and often noticeably lower cost, with implementations for Linux, macOS and Windows. But keep perspective. Even the best result gets full credit on only about a third of tasks. Long computer workflows are still far from reliable.

That's today's Daily Frontier Papers. The paper is linked in the description.

