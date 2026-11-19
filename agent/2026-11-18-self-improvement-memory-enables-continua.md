# Self-Improvement Memory Enables Continual Recursive Self-Improvement

**换了模型优化别从头来**

- 论文 Paper: [arXiv 2611.18856](https://arxiv.org/abs/2611.18856)
- 单位 Affiliation: Google Cloud AI · Google DeepMind · 宾夕法尼亚州立大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-18
- 视频 Videos: [YouTube 中文](https://youtu.be/6ciq8AO6RCg) · [B站](https://www.bilibili.com/video/BV16yst6VEXq)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 Google 和宾夕法尼亚州立大学的论文，题目直译过来是，自我改进记忆，让持续的递归自我改进成为可能。

一句话概括，它不是给干活的 Agent 加记忆，而是给负责修改 Agent 的那个优化器加记忆。

### 问题

先解释一个词，harness。大模型 Agent 除了模型本身，外面还包着一层框架，负责管理上下文、工具和执行流程，这层框架就叫 harness。

改 harness 不用动模型权重，也能提高成绩，但人工改很费工夫。所以最近有一类方法，叫递归自我改进，简称 RSI，让大模型根据执行反馈，反复提出修改、测试、挑选，自动把 harness 越改越好。

问题在于，这种优化通常在固定的模型和基准上做。可现实里，模型会换代，Agent 也会碰到新任务，优化就得接着做。

而且，直接把改好的 harness 搬过去，可能还不如不搬。论文里，在 Claude Opus 5.5 上优化出来的 harness，换给 Gemini 3.8 Flash 用，在办公类任务上只有 68.41 分，比原始 harness 的 73.89 分还低。

换数据也一样。文本分类实验里，源任务上优化好的 harness，到了新的数据集上，起点是 72.92 分，也低于原始 harness 的 75.72 分。

作者认为，光有 harness 代码，回答不了三个问题，这个修改当初为什么提，想修什么，测下来效果怎样。没有这段历史，新一轮优化可能重复老路，或者改掉自己不明白用途的组件。

### 以前的方法

以前的工作，比如 Meta-Harness 和 AHE，是拿执行轨迹和分数，在 harness 代码上做搜索。优化好的 harness 换到别的任务或模型上，有时能保住收益，但要看模型和 harness 怎么配合。

也有工作会存下候选版本、修改历史或者文字笔记。这篇论文的主要对照组，是以前研究里用过的起点，直接从优化好的 harness 接着优化，叫 Harness Transfer。

### 核心想法

这篇的核心想法是，记忆不给执行任务的模型，而是给负责改 harness 的优化器。作者管它叫自我改进记忆，简称 SIM。

SIM 按反复出现的问题来归类修改。每个问题下面，记着每次改了什么、为什么改、预计修好哪些任务、实际修好了哪些，以及在哪个模型、哪个基准、哪一轮做的。

换了模型或数据之后，这份记忆跟着 harness 一起带过去，优化器就能判断，继承来的修改哪些该留，哪些该改、该换，或者该删。

### 方法

先看优化循环，就是这张图。一共跑 N 轮，每一轮，评测器在训练任务上运行当前 harness，返回分数、轨迹和报错。

提议者根据反馈和历史，生成 k 个候选 harness。挑选者先筛一遍，再从跑过的候选里选一个，除了分数，还看复杂度、是不是只对某个领域有效，会不会让别的任务退步。一个都没选中，就保留原来的。

提议者和挑选者都是大模型扮演的 harness 工程师，执行任务的模型权重是冻结的。而且只有训练集的反馈能进入循环，测试集完全不参与。

SIM 的每一条，对应一个根因假设，比如过早宣布任务完成。下面按顺序挂着每次尝试的修改和理由、预测和实际结果，还有出处。关键是，预测要在评测之前写下，评测之后再核对。

每一轮，提议者在系统提示词里拿到完整的 SIM，挑选者也读同一份，不检索，也不截断。每个提议，要么引用已有的根因，要么命名一个新的。

只有被选中的修改会记进去，即使它让分数退步，或者没修好预测的任务，也照样记录，旧的尝试不删。

实验比较四种条件。冷启动，从原始 harness 开始，不带记忆。Harness Transfer，从优化好的 harness 开始，不带记忆。只带根因的记忆。还有完整的 SIM。后三种起点相同，四种的目标模型、任务、轮数和候选数都一样。

### 实验和效果

实验覆盖四个领域。编程，从 Terminal-Bench 2.1 换到 Terminal-Bench 4。文本分类，从三个数据集换到八个没用过的数据集。办公类 Agent 任务，从 Harvey LAB 换到另外三个基准的平均。还有工程设计，从 EngDesign 换到 Frontier-Eng。

换数据的实验，执行任务的一直是 Gemini 3.8 Flash。换模型的实验，源优化阶段执行任务的是 Claude Opus 5.5，之后换成 Gemini 3.8 Flash 接着优化。每次优化都是 10 轮，每轮 5 个候选。

先看换数据，结果在这张图。六个设置里，完整 SIM 的最终分数全部最高，比 Harness Transfer 高 2.43 到 4.35 分。

只带根因的记忆效果不稳定，相对 Harness Transfer，从低 1.82 分到高 2.40 分都有。作者的解读是，光记住诊断不够，修改历史和评测证据才帮得上忙。

有个例子很说明问题。文本分类的 FewShotAll 设置里，直接接着优化只到 74.12 分，连冷启动的 75.92 分都不如，带完整 SIM 到了 77.92 分。

再看换模型，从 Opus 5.5 换到 Gemini 3.8 Flash。四个领域里，Opus 优化出来的 harness 一上来都不如 Gemini 的原始 harness，直接接着优化，有三个领域最后还是不如冷启动。

完整 SIM 在四个领域都最高，比 Harness Transfer 高 3.60 到 11.33 分。差距最大的是办公类任务，都从 68.41 分起步，带 SIM 到了 84.75 分，不带只有 73.42 分。

还有一个跨代实验。在 Terminal-Bench 2.1 上，把 harness 和 SIM 从 Gemini 2.5 Flash 一路传到 Gemini 4 Argon，和每一代从头优化比较。后面三代里有两代更好，3.5 Flash 高 3.5 分，3.8 Flash 高 1.6 分，但到了 4 Argon，反而低了 0.8 分。

前面这些，每个条件只跑了一次。为了看是不是偶然，作者在编程上把源优化独立重复了 10 次，每个结果分别带 SIM 和不带 SIM 各接着优化一次。

结果在表 3。10 对里有 9 对是带 SIM 的更高。平均起点 18.36 分，带 SIM 最后是 21.70 分，不带是 19.33 分，平均多 2.36 分，配对检验的 p 值是 0.003。

### 和以前方法的对比

SIM 到底怎么帮上忙。论文追踪了一个例子，完成门控。源任务里有个任务要搭 git 服务器，Agent 没创建要求的用户账号，用已有账号在本地测了一遍，就宣布完成，结果外部验证失败。

原始 harness 只会问一句，你确定要标记完成吗。源优化于是加了一个中间件，上一轮有解析错误，或者根本没有命令执行过，就拒绝完成。这个任务从 5 次成功 1 次，变成了 5 次全部成功。

换到 Terminal-Bench 4，出了新问题。有个光波导布线任务，检查脚本只看约束满不满足，真正的得分却是布线成本。Agent 找到一个可行解，检查通过，就宣布完成，继承来的门控也拦不住。

带 SIM 的运行，提议者读到了门控当初的记录，知道它是防止没验证就完成的，于是保留门控，再加一条提醒，可行不等于最优，要拿分数和理论界比一比。

不带 SIM 的运行，把同一个门控改了四次，其中两次是在修完成环节的卡死。第 3 轮的说明里，还把这个继承来的门控，当成了自己第 1 轮加的。

不过，针对这个新问题的两次修改虽然都被选中，预测要修好的任务却一个也没修好。这个结果也如实记在了 SIM 里。

我的判断是，和直接搬 harness 相比，SIM 补上的是修改背后的理由和证据。它不改优化流程，也不动模型，只是给优化器一份带出处的实验记录，更像团队的变更日志，而不是一堆没有注释的代码。

### 局限

作者自己说了几条局限。独立重复只在编程上做了，其他领域每个条件只跑一次。而且只考虑 harness 接口兼容的情况，优化器的规划和搜索方法也是固定的。

另外，各条件只对齐了轮数和候选数，token 用量和运行时间可能不一样。办公类任务的分数，也可能带着评委模型的偏差。

作者还提醒，会改自己的 harness，可能学会满足评测器，而不是真正解决任务，所以优化器完全接触不到测试集的反馈。

### 未来方向

未来方向，作者提到要做更多次重复运行，测更长的迁移链条，还要让规划、token 分配和工具接口也跟着调整。

我的判断是，这篇的价值在于提了一个真问题。模型几个月就换一代，harness 优化不该每次从头来，留一份带理由和结果的改进记录，思路简单，也容易照搬。

我的判断是，也要看到证据的分量。在 Terminal-Bench 4 上，各方法的分数只有 20 分上下，大部分对比是单次运行，跨代实验到了 Gemini 4 Argon 还输了 0.8 分。所以它更像一个有希望的方向，还不是定论。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Google, with authors at Google Cloud AI, Google DeepMind and Penn State. The title: Self-Improvement Memory Enables Continual Recursive Self-Improvement.

The short version: it gives memory not to the agent doing the work, but to the optimizer that keeps rewriting the agent.

### Problem

First, a term. An LLM agent has two parts: the model itself, and the code wrapped around it that manages context, tools and execution. That wrapper is called the harness.

Improving the harness can raise scores without touching the model's weights, but doing it by hand means endless rounds of engineering and evaluation. So a recent line of work automates it. It's called recursive self-improvement, or RSI: a model uses execution feedback to propose harness edits, test them, and keep the best ones, over and over.

The catch is that this optimization usually happens in one fixed setting, one model and one benchmark. In practice, models get replaced and agents meet new tasks, so the optimization has to keep going.

And simply carrying the evolved harness over can be worse than not carrying it at all. In the paper, a harness evolved on Claude Opus 5.5, handed to Gemini 3.8 Flash on workspace tasks, starts at sixty-eight point four one. The plain, unevolved seed harness scores seventy-three point eight nine.

Changing the data does the same thing. In text classification, a harness evolved on the source suite starts the new target suite at seventy-two point nine two, below the seed's seventy-five point seven two.

The authors' diagnosis: the harness code alone can't tell you why an edit was made, what it was meant to fix, or what happened when it was tested. Without that history, the next optimizer may repeat old attempts, or change inherited parts without knowing what they're for.

### Prior work

Earlier methods like Meta-Harness and AHE search over harness code using execution traces and scores. Other work found that evolved harnesses can keep some of their gains on new tasks or models, but how well that works depends on how model and harness interact.

Some methods also keep records of the process: candidate versions, edit histories, free-text notes. The main control in this paper is the starting point studied in earlier work: just continue optimizing from the evolved harness. They call it Harness Transfer.

### Key idea

The key idea is to give memory to the harness optimizer, not to the model solving the tasks. The authors call it Self-Improvement Memory, or SIM.

SIM groups edits by the recurring harness problem they address. Under each problem, it records every attempt: what was changed and why, which tasks it was predicted to fix, which it actually fixed, and the model, benchmark and round it came from.

When the model or the data changes, this memory travels with the harness, so the optimizer can decide which inherited changes to keep, which to revise or replace, and which to remove.

### Method

Start with the optimization loop itself, shown in this figure. A run has N rounds. In each round, an evaluator runs the current harness on the training tasks and returns scores, traces and errors.

A proposer uses that feedback and the run's history to generate k candidate harnesses. A selector screens them, then picks among the ones that were run, weighing not just performance but complexity, domain specificity and the risk of regressions. If nothing is accepted, the old harness stays.

The proposer and selector are LLMs acting as harness engineers. The policy model that actually does the tasks stays frozen. And only feedback from the training split ever enters this loop. The held-out set never does.

Each SIM entry is one root-cause hypothesis, say, premature completion. Under it, attempts are stored in order: the change and its rationale, predicted and observed outcomes, and provenance. The important detail is that predictions are written down before evaluation and checked afterward.

Every round, the proposer gets the complete SIM in its system prompt, and the selector reads the same record. No retrieval, no truncation. Each proposal either cites an existing root cause or names a new one.

After selection, only the chosen edit is recorded, including its measured outcome, even when it made things worse or failed to fix what it predicted, and earlier attempts are never deleted.

The experiments compare four conditions. Cold Start begins from the seed harness with no memory. Harness Transfer begins from the evolved harness with no memory. Problem-Only Memory keeps just the root causes, without the edits and outcomes. And Full SIM keeps everything. The last three share the exact same starting harness, and all four get the same target model, tasks, rounds and candidate budget.

### Results

There are four domains. Coding moves from Terminal-Bench two point one to Terminal-Bench four. Text classification moves from a three-dataset suite to eight datasets the source run never saw. Agentic workspace tasks move from Harvey LAB to the average of three other benchmarks. And engineering design moves from EngDesign to Frontier-Eng.

In the cross-dataset runs, Gemini 3.8 Flash does the tasks throughout. In the cross-model runs, Claude Opus 5.5 does the tasks during source optimization, and then Gemini 3.8 Flash takes over. Every run is ten rounds with five candidates per round.

First, changing the data. Results are in this figure. In all six settings, Full SIM finishes highest, between two point four three and four point three five points above Harness Transfer.

Keeping only the root causes is hit or miss. Against Harness Transfer, it lands anywhere from one point eight two points below to two point four points above. The authors' reading is that the diagnosis alone isn't enough. What helps is the edit history and the evidence.

One case makes the point. In the FewShotAll text-classification setting, the evolved harness starts below the seed. Continuing without memory only reaches seventy-four point one two, still below Cold Start at seventy-five point nine two. Full SIM reaches seventy-seven point nine two.

Next, changing the model, from Opus 5.5 to Gemini 3.8 Flash. In all four domains, the Opus-evolved harness starts out worse than Gemini's own seed harness. And continuing without memory still ends below Cold Start in three of the four.

Full SIM finishes highest in all four domains, three point six to eleven point three three points above Harness Transfer. The biggest gap is in workspace tasks. Both start at sixty-eight point four one. With SIM, the run reaches eighty-four point seven five. Without it, seventy-three point four two.

There's also a generations experiment. On Terminal-Bench two point one, the harness and SIM are passed from Gemini 2.5 Flash to 3.5 Flash, 3.8 Flash, and then Gemini 4 Argon, and compared with optimizing from scratch at each generation. In two of the three later generations, continuing wins, by three point five points on 3.5 Flash and one point six on 3.8 Flash. On 4 Argon it finishes zero point eight points lower.

Everything so far is one run per condition. To check that the result isn't luck, the authors repeated source optimization in coding ten independent times, and continued each resulting harness twice, once with SIM and once without.

The results are in Table 3. SIM wins in nine of ten pairs. From an average start of eighteen point three six, runs with SIM end at twenty-one point seven, runs without at nineteen point three three. That's a mean gain of two point three six points, with a paired p-value of zero point zero zero three.

### Comparison

So how does SIM actually help? The paper follows one concrete case: a completion gate. On the source benchmark, one task asks the agent to set up a git server. The agent never creates the required user account, tests everything locally with an existing account, and declares the task complete. The external check then fails.

The seed harness only asks, are you sure you want to mark the task as complete? So source optimization added middleware that refuses completion if the last turn had a parse error, or if no command actually ran. On that task, successful runs went from one out of five to five out of five.

Then on Terminal-Bench four, a new problem shows up. In a photonic waveguide routing task, the bundled checker only tests whether the constraints are met, while the real score is the routing cost. The agent finds a feasible route, the check passes, it declares completion, and the inherited gate has no reason to stop it.

In the run with SIM, the proposer reads the gate's original entry, so it knows the gate exists to stop completion without verification. It keeps the gate and adds a reminder: feasible isn't optimal, so compare your score against the theoretical bound.

In the run without SIM, the same gate gets edited four times, twice just to fix completion deadlocks. And its round-three rationale credits the inherited gate to its own first round.

To be fair, the two edits aimed at this new problem were both selected, yet neither fixed the tasks it predicted it would. That outcome is recorded in SIM too.

My read is that compared with simply carrying a harness forward, SIM adds the reasons and the evidence behind each change. It doesn't touch the propose, evaluate and select procedure, or the model. It gives the optimizer a lab notebook with provenance, more like an engineering team's changelog than a pile of uncommented code.

### Limitations

The authors list several limitations. Independent repeats were only done for coding. The other domains have one run per condition. They only study cases where the harness interface stays compatible, and the optimizer's planning and search procedure is kept fixed.

Also, the conditions are matched on rounds and candidates, but token use and runtime can differ. And workspace scores may carry the biases of the judging model.

The authors also warn that a self-modifying harness could learn to satisfy the evaluator without solving the task, which is why the optimizer never sees held-out feedback.

### Future work

For future work, the authors point to more repeated runs, longer chains of transitions, and letting planning, token allocation and tool interfaces adapt too.

My read is that the paper's value is in naming a real problem. Models turn over every few months, and harness optimization shouldn't restart from zero each time. Keeping a record of edits with reasons and outcomes is a simple idea and easy to copy.

My read is also that the evidence has limits. On Terminal-Bench four, every method scores around twenty, most comparisons are single runs, and in the generations chain, the last model, Gemini 4 Argon, came out zero point eight points worse. It looks like a promising direction, not a settled result.

That's today's Daily Frontier Papers. The paper is linked in the description.

