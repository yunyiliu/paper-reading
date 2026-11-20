# The Resolution Hypothesis: Agent Evolution's Key Is Identifying Improvements

**Agent进化分数涨了能力却没涨**

- 论文 Paper: [arXiv 2611.23068](https://arxiv.org/abs/2611.23068)
- 单位 Affiliation: 上海科技大学 · 字节跳动
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-20
- 视频 Videos: [YouTube 中文](https://youtu.be/wPwb71UFOwU) · [YouTube English](https://youtu.be/7TPraGuSanU) · [B站](https://www.bilibili.com/video/BV18SypzVEC1)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇上海科技大学和字节跳动等单位的论文，题目叫 The Resolution Hypothesis，可以叫它“分辨率假说”。副标题是，Agent 进化的关键，在于认出哪些改动真的是改进。

这篇不提新方法，它做的是一次大规模的复查。作者想回答一个很多人都困惑的问题，为什么让 Agent 自己进化，有时候效果很好，有时候怎么都起不来。

### 问题

先解释什么叫 Agent 进化。模型权重不动，改的是模型外面的东西，比如提示词、技能文档、记忆，甚至整套 Agent 的外壳代码，也就是 harness。论文把这些统称为 artifact，我们就叫它“产物”。

流程是一个循环。提议者写出一批候选产物，装到 Agent 上，在一组任务上跑分，分数更高的留下来，进入下一轮。

问题在于，这个方向的结论很不稳定。同一个方法，换个 benchmark 就从有效变成无效，不同论文里的排名也来回颠倒。在 Terminal-Bench 这类编程 Agent 的 benchmark 上，已经有好几篇工作报告说，进化会停滞。

### 以前的方法

以前大家的主要精力，放在设计新的进化方法上，有的进化提示词，有的进化技能和记忆，有的直接进化可执行的代码。

也有一些分析注意到这个方向很脆弱，比如对运行之间的随机性和任务顺序很敏感，加进去的技能还会引发新的失败。作者说，他们这篇要做的，是把背后的机制单独拆出来验证。

### 核心想法

作者的假说是，Agent 进化的瓶颈，不是想不出更好的产物，而是认不出哪个更好。

为什么难认。因为权重是冻结的，产物只能通过改变模型推理时的行为来起作用，而 Agent 每次做同一个任务，轨迹本来就不一样。真正的行为变化和随机波动，很难分开。

作者为此定义了一个概念，叫评测的分辨率，就是一条评测流程能可靠地和噪声区分开的最小真实改进。真实改进一旦小于这个分辨率，挑选就退化成碰运气，改进也就积累不起来。

论文第一张图讲的就是这个。噪声大的时候，分数最高的那个候选，实际上可能更差。噪声小的时候，才能留下真正最好的那个。

### 方法

为了验证，作者做了一次规模很大的实验。七种进化方法，六个模型，四个 Agent benchmark，另外还在九个 benchmark 上复查了以前论文报告的提升。

方法包括进化提示词的 GEPA，进化技能的 SkillOpt-Lite 和 Trace2Skill，进化记忆的 ReasoningBank，还有三种进化 harness 的方法。模型有 GPT-5.4 nano、GPT-5.6 的两个版本、DeepSeek-V4-Flash、Kimi-K3 和 GLM-5.2。

关键在于重复。每个方法在每个模型和 benchmark 上独立进化 8 次，每个产物最多再评测 16 次。合计 1030 次独立进化，超过 210 万条测试轨迹，整个研究用了超过一千万次大模型调用。

分析工具是一个方差分解。分数的波动有两个来源，一个记作 sigma，是同一个产物重复评测时的噪声，另一个记作 tau，是同一个方法每次进化出来的产物本身就不一样。比较一个固定产物，只受 sigma 限制，比较两个方法，两种波动都要算进去。

### 实验和效果

先看方法层面。评测噪声 sigma，在 SpreadsheetBench、OfficeQA 和 ALFWorld 上是 1 到 3.7 个百分点，在 Terminal-Bench 上到了 4.5 个百分点。一个方法进化一次、评测一次，典型的标准差在 1.7 到 4.1 个百分点之间。

结果是，在同一个模型、同一个 benchmark 上，超过六成的配置两两相比，平均分的差距还不到单次比较的一个标准差。

再看已发表论文里的差距。作者统计了九篇 Agent 进化论文主表里的 171 个结果，新方法领先最强对手的中位数是 3.4 个百分点。这些结果没有一个报告方差，只有 6 个对多次进化取了平均。

要可靠地分辨 3.4 个百分点，按论文的统计功效计算，在 Terminal-Bench 上每个方法大约要跑 23 次。只跑一次的话，想从四种文本类方法里挑出真正最好的那个，在 Terminal-Bench 上只有 33% 的概率，而随便猜也有 25%。

那为什么有的 benchmark 上能看到 10 到 40 个百分点的大提升。作者借助一个 LLM Agent 加人工核对，逐条检查进化出来的产物，只把最关键的一条规则加到没进化过的基线上。论文报告，在提升超过 3 个百分点的 27 个模型和 benchmark 组合里，这一句话就拿回了提升的中位数 90%。

这些规则大多是在钻评测的空子。比如 SpreadsheetBench 随方法发布的评测，读公式单元格时不重新计算，所有公式都被当成空值。作者从产物里找出的关键规则就是，别写公式，直接写算好的数值。

作者把这类评测修好，比如像官方评测那样先重算，或者按意思而不是按字面判分，再给同样的产物打分。提升的中位数从 17.6 个百分点降到 0.8 个百分点，在修好的评测上重新跑文本类进化方法，提升也没有回来。

接着看候选层面。作者让一个提议者在 Terminal-Bench 上写出 100 个技能，每个在 10 个没见过的任务上各跑 16 次。在 2 小时的执行预算下，最好的通用技能成功率是 57.5%，候选的平均是 49.2%，说明好的候选确实存在。

但挑选这一步不行。看这张表，用一次执行的分数做两两比较，选中更好那个的概率是 49.7%，和抛硬币的 50% 在统计上没有区别。换成大模型当裁判也一样，几种设置都在 45% 到 49% 之间。

更糟的是，退步的技能只占候选的 30.5%，却占了被接受候选的 62.1%。而且候选从 1 个加到 100 个，所有实际的挑选方法，基本都在随机挑选的水平上打转。

最后，作者把 SkillOpt-Lite 和 GEPA 各 8 次进化里的每个中间产物，都在同一个验证集上重新评测 8 次。这张图里，进化过程中自己记录的分数涨了 6.9 到 9.0 个百分点，重新评测后，变化只在正 0.4 到负 3.2 个百分点之间。

原因是，每一轮都接受了分数碰巧偏高的那个候选，分数越涨越高，能力并没有变强。而且每一轮的选错累积下来，同一个方法的几次运行越走越远，到最后差别和论文之间的方法差距一样大。

### 和以前方法的对比

和以前关于进化不稳定的分析相比，作者说他们的不同在于，把评测噪声和运行之间的差异分开测量，并且用固定的候选池，分别检验提议和挑选这两个环节。

我的判断是，这篇对整个方向的提醒很直接。如果一篇 Agent 自我进化论文只领先三四个点，又只跑了一次，那这个排名很可能说明不了什么。

我的判断是，做产品的人也该注意，自动优化提示词或技能时，评测集上的分数上涨，不一定是能力变强，可能只是挑中了运气好的那次，或者学会了评分器的格式偏好。

### 局限

作者自己列了几条局限。第一，只研究权重冻结的情况，用强化学习更新权重的方法，面对的约束可能不一样。

第二，候选层面的证据只来自 Terminal-Bench v2.1，两个候选池各 100 个技能，跨轮次的重新评测也只覆盖两种方法和两个模型。

第三，更强的模型噪声可能更低，作者不能排除这一点，不过在他们的实验里，Terminal-Bench 上三个模型的噪声中位数都在 3.5 到 4 个百分点。另外，像 GPU kernel 优化这种用确定性程序打分的进化，不在这次研究范围内。

### 未来方向

作者给了几个改进方向。一是补充更细的过程指标，比如检查命令的失败信息有没有被掩盖，初步分析里，这个指标的评测噪声比最终的成功失败低 13 到 57 倍。二是把提议者和挑选者分开评。三是发论文时同时报告 sigma 和 tau，让读者自己判断差距是否超过了分辨率。

作者说，实验材料和轨迹数据会在论文被接收后公开。

我的判断是，这篇也有没解决的地方。过程指标还只是初步分析，作者也承认它本身不能解决挑选问题。附录里试过每个候选评测 16 次再取平均，进化效果确实变好了，但作者也说这样算力成本很高，更便宜的办法，论文还没有给出。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from ShanghaiTech University, ByteDance and collaborators. It's called The Resolution Hypothesis, and the subtitle makes the claim: the key to agent evolution is identifying improvements.

There's no new method here. It's a large-scale audit, aimed at a puzzle many people in this area have run into: why does letting an agent evolve itself sometimes work well, and sometimes never get off the ground?

### Problem

First, what agent evolution means. The model weights stay frozen. What changes is everything around the model: the prompt, skill documents, memories, even the agent's whole harness code. The paper calls these artifacts.

The process is a loop. A proposer writes candidate artifacts, the agent runs a set of tasks with each one, and whichever scores higher is kept for the next round.

The trouble is that results in this area don't hold still. The same method looks effective on one benchmark and useless on another, and rankings swap places from paper to paper. On Terminal-Bench and other coding-agent benchmarks, several groups have reported that evolution simply stalls.

### Prior work

Most of the effort so far has gone into new evolution methods: some evolve prompts, some evolve skills or memories, and some evolve executable code directly.

Some analyses have flagged how fragile this is, for instance sensitivity to run-to-run variance and task order, or skills that cause new failures. The authors say their contribution is to isolate the mechanism behind it and test it directly.

### Key idea

The hypothesis: the bottleneck in agent evolution isn't proposing better artifacts. It's identifying which ones are better.

Why is that hard? The weights are frozen, so an artifact can only help by changing the model's behavior at inference time, and an agent's trajectory already varies every time it attempts the same task. Telling a real behavioral change apart from that noise is difficult.

So the authors define the resolution of an evaluation: the smallest true improvement it can reliably separate from noise. Once real improvements fall below that resolution, selection turns into a coin toss, and little improvement accumulates.

The paper's first figure shows the idea. Under high noise, the candidate with the top score can actually be a worse one. Only under low noise does the best candidate reliably win.

### Method

To test this, the authors ran a very large study: seven evolution methods, six models, four agent benchmarks, plus nine more benchmarks where they re-examined gains reported in earlier papers.

The methods include GEPA for prompts, SkillOpt-Lite and Trace2Skill for skills, ReasoningBank for memory, and three methods that evolve the harness. The models are GPT-5.4 nano, two versions of GPT-5.6, DeepSeek-V4-Flash, Kimi-K3 and GLM-5.2.

The key is repetition. Each method is evolved eight separate times per model and benchmark, and each resulting artifact is evaluated up to sixteen times. That's one thousand and thirty independent evolution runs, more than two point one million test trajectories, and over ten million model calls across the whole study.

The analysis rests on a variance decomposition. Score swings come from two places. Sigma is evaluation noise, how much the same artifact's score moves when you test it again. Tau is run-to-run variance, because each evolution run produces a different artifact. Judging one fixed artifact is limited by sigma alone, but comparing two methods means paying for both.

### Results

Start at the method level. Evaluation noise, sigma, runs from one to three point seven percentage points on SpreadsheetBench, OfficeQA and ALFWorld, and reaches four point five on Terminal-Bench. A method evolved once and evaluated once has a typical standard deviation of one point seven to four point one points.

Here's the consequence. For over sixty percent of pairs of configurations on the same model and benchmark, the gap between their averages is smaller than one standard deviation of a single-run comparison.

Now compare that to the literature. Across one hundred seventy-one results in the main tables of nine agent-evolution papers, the new method beats its strongest competitor by a median of three point four points. None of those results reports a variance estimate, and only six average over more than one evolution run.

To reliably detect a three point four point margin, the paper's power calculation says you need about twenty-three runs per method on Terminal-Bench. With a single run, a results table picks the truly best of the four text-artifact methods only thirty-three percent of the time there. A random guess gets twenty-five.

So where do the occasional huge gains of ten to forty points come from? The authors went through evolved artifacts instruction by instruction, with an LLM agent flagging candidates and a human checking each one, then added only the single most responsible rule to the unevolved baseline. Across the twenty-seven model and benchmark pairs with a reported gain above three points, that one sentence recovered a median of ninety percent of the gain.

Most of those rules exploit an evaluation shortcut. Take SpreadsheetBench as released with one method: its grader reads formula cells without recalculating them, so every formula scores as empty. The key rule the authors pulled out is simply, don't write formulas, write the computed values.

Then the authors repaired the shortcuts, for example recalculating as the official evaluation does, or grading answers by meaning instead of exact wording, and re-scored the same artifacts. The median gain fell from seventeen point six points to zero point eight. Re-running the text-artifact methods under the repaired evaluations didn't bring the gain back.

Next, the candidate level. A proposer wrote one hundred skills on Terminal-Bench, and each was run sixteen times on ten unseen tasks. Under a two-hour execution budget, the best shared skill reached fifty-seven point five percent success, against a pool average of forty-nine point two. So good candidates do exist.

Selection is where it breaks. In this table, picking the better of two skills from one execution score gets it right forty-nine point seven percent of the time, statistically no different from a coin flip. LLM judges don't help either: the settings tested land between forty-five and forty-nine percent.

It gets worse. Regressions make up thirty point five percent of the pool, but sixty-two point one percent of the candidates that get accepted. And as the number of candidates grows from one to a hundred, every practical selector stays roughly flat at the level of random choice.

Finally, the authors re-evaluated every intermediate artifact from eight SkillOpt-Lite runs and eight GEPA runs, eight times each, on the same validation set. In this figure, the scores the runs recorded for themselves climbed six point nine to nine points, while the re-evaluated change stayed between plus zero point four and minus three point two.

The reason is that each round accepts whichever candidate got a lucky high score. The recorded score keeps rising while the agent gets no better. And those per-round mistakes add up, pushing separate runs of the same method apart until they differ as much as published methods differ from each other.

### Comparison

Compared with earlier analyses of why evolution is unstable, the authors say what's new is measuring evaluation noise and run-to-run variance separately, and using fixed candidate pools to test proposal and selection one at a time.

My read is that the lesson for the field is blunt. If an agent self-evolution paper leads by three or four points from a single run, that ranking may not mean much.

My read is that product teams should take note too. When you auto-tune prompts or skills, a rising score on your eval set may not be a better agent. It may be a lucky draw, or a learned quirk of the grader's answer format.

### Limitations

The authors list several limitations. First, they only study frozen weights. Methods that update weights with reinforcement learning may face different constraints.

Second, the candidate-level evidence comes only from Terminal-Bench v2.1, with pools of one hundred skills, and the round-by-round re-evaluation covers just two methods and two models.

Third, stronger future models might have lower noise, and the authors can't rule that out, though in their grid the median noise on Terminal-Bench stayed at three point five to four points for all three models. Evolution scored by deterministic programs, like GPU kernel optimization, is also outside the study.

### Future work

The authors suggest a few fixes. One is finer process metrics, such as whether failure diagnostics in check commands are masked; in a preliminary analysis that metric showed thirteen to fifty-seven times lower evaluation noise than final pass or fail. Another is evaluating proposers and selectors separately. A third is reporting sigma and tau alongside scores, so readers can judge whether a margin exceeds the resolution.

The authors say materials and trajectory data will be released once the paper is accepted.

My read is that the paper leaves the hardest part open. The process metrics are preliminary, the authors admit they don't fix selection by themselves, and the one fix tested in the appendix, averaging sixteen validation rollouts per candidate, did improve evolution but is expensive, as the authors note. A cheaper fix isn't on offer yet.

That's today's Daily Frontier Papers. The paper is linked in the description.

