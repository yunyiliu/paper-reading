# Your Benchmark Is an Empirical Measure Over Difficulty

**评测分数不动不代表没进步**

- 论文 Paper: [arXiv 2610.29927](https://arxiv.org/abs/2610.29927)
- 单位 Affiliation: 伊利诺伊大学厄巴纳-香槟分校、加州大学圣地亚哥分校、纽约大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-31
- 视频 Videos: [YouTube 中文](https://youtu.be/5gxWuAqPpQQ) · [YouTube English](https://youtu.be/C4IaguexS2I) · [B站](https://www.bilibili.com/video/BV1EcxC6QEDb)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇伊利诺伊大学厄巴纳香槟分校的论文，合作单位还有加州大学圣地亚哥分校和纽约大学。题目直译是，你的 benchmark，是难度上的一个经验测度，论文首页标注为 NeurIPS 2026 评测与数据集赛道。

题目听着很数学，但问题很实际，我们天天看的排行榜分数，还有训练时盯着的评测曲线，到底能信几分。

### 问题

跑 benchmark 看起来很简单，一堆题，让模型做一遍，算个平均分。这个分数有两个用处，在排行榜上比模型，在训练中看进步。

作者指出，结论不只取决于模型有多强，还取决于这些题在难度上是怎么分布的。

这张示意图画了两种失败。左边，两个模型在简单题上差不多，在难题上差很多，如果 benchmark 里大多是简单题，两个模型看起来就打成了平手。

右边，模型在某一段难度上进步很大，但 benchmark 在这一段正好没有题，总分几乎不动，真实的进步就被漏掉了。

### 以前的方法

benchmark 靠不靠得住，以前已经有不少研究，比如数据污染，就是测试题混进了训练数据，还有研究发现，结果对解码方式、prompt 和实现细节很敏感。

给题目估难度，也早有人做。一类靠人工标注，比如 MATH 把题分成五个难度等级。作者认为，这样成本高，而且反映的是人觉得难不难，不是模型觉得难不难。

另一类用项目反应理论，也就是 IRT，这是心理测量学里给考试题定难度的方法，已经常用来分析大模型 benchmark。这篇的新东西，是把 benchmark 当成难度轴上的一个分布，让前面两种失败能被严格定义。

### 核心想法

核心想法一句话，benchmark 不只是一堆题，它在难度轴上撒了一把点，每道题在自己的难度位置上，占同样的权重。

这样一来，benchmark 分数就等于，模型在每个难度上的表现，按这把点加权求平均。问题就变成了两件事，这把点撒得对不对，撒得匀不匀。

撒得不对，叫测度错配，也就是 benchmark 的难度分布，和你真正关心的难度分布不一样。撒得不匀，叫覆盖缺口，难度轴上有一段没有题。据作者所知，这是第一次把这两种问题定义成 benchmark 评测的两种基本失败模式。

### 方法

针对这两个问题，作者给了两个诊断指标。第一个叫 DDM，难度分布错配，就是 benchmark 的难度分布和目标分布之间的 Wasserstein 距离。直观理解是，要把一种分布搬成另一种，平均要把多少分量搬多远。

定理 2.1 说，在性能差随难度平滑变化的假设下，这个距离限定了 benchmark 上的分差，和目标分布上的分差，最多能差多少。如果两个模型在目标分布上的差距大于这个界，排名就不会翻转。

那目标分布选什么。作者证明，如果完全不知道两个模型的差别出现在哪个难度，均匀分布是最坏情况下最稳妥的唯一选择。作者也强调，这不是说均匀分布永远最好，只是没有先验时的中性默认。

第二个指标是最大缺口。把所有题按难度排好，相邻两道题之间最大的空隙有多宽。

定理 2.3 说，如果模型的进步只发生在一段比最大缺口还窄的难度区间里，最坏情况下，不管重复评测多少次，都检测不出来。进步的区间比最大缺口宽，评测次数够多，才能检测出来。

难度怎么估。作者用的是 IRT 的单参数模型，每个模型一个能力值，每道题一个难度值，模型做对的概率，由能力减难度决定。

对非 Agent 类的 benchmark，作者挑了 12 个模型，参数量从 1.7B 到 397B，每道题最多跑 32 次。Agent 类的 benchmark 通常需要很强的闭源模型，就直接用 HAL 排行榜上的评测结果。

拟合用的是 MCMC 贝叶斯推断，前后做了 7 项诊断，25 个 benchmark 全部通过。非 Agent 类的，每次去掉一个模型重新拟合，难度估计也很稳定。

### 实验和效果

作者分析了 25 个常用 benchmark，覆盖数学、代码、Agent、逻辑、科学、指令遵循和医学，7 个领域。结果在表 1。

先看错配。25 个里有 10 个的 DDM 大于等于 0.1，意思是平均要把每道题在难度轴上挪动全长的百分之十以上，才能变成均匀分布。比如 MATH-500 是 0.164，ScienceQA 是 0.178，IFEval 是 0.159。

作者的解读是，这些 benchmark 隐含了很强的难度权重，它们的排行榜结论，不能当成难度中性的比较，用之前要先看看这个配比，是不是你关心的那种。

再看缺口。25 个里有 8 个的最大缺口大于等于 0.1，有 9 个的前五大缺口加起来大于等于 0.3。

这张图是 AIME24 和 AIME25 的难度分布，阴影是最大的五个缺口。作者指出，这两个 benchmark 前五大缺口加起来接近 0.5，难度轴上很大一部分分辨不清，而它们恰恰被大量用来监控 RLVR 训练，也就是用可验证奖励做强化学习。

缺口也不只是小 benchmark 的问题。HumanEval 有 164 道题，最大缺口是 0.165，BBH 5 objects 有 250 道题，最大缺口是 0.128。

诊断完，作者给了一个补救办法，叫 BenchFill，往缺口里补题。

思路是，找到一个缺口，把目标难度定在缺口正中间，然后让大模型改写已有的题，生成一道难度刚好落在那里的新题。新题是加进去的，不替换原来的题。

具体是一个迭代搜索。记忆库里存着题目和难度，每轮挑难度最接近目标的几道当父题，再给改写模型看几道目标上下的参考题，让它改出一批子题。

比如这张图里，原题说，n 个骰子的点数相乘，能得到 550 种不同的乘积，问 n 是多少，改写后加一个条件，乘积必须是完全平方数，难度就变了。

每道新题先由 GPT-5.4 检查答案对不对，再让那批模型实际去做，估出难度，放回记忆库，下一轮接着改。最后选最接近目标的那一道，还要经过人工专家审核。

实验设置是，8 个 benchmark，每个补最大的三个缺口，改写模型是 MiniMax-M2.7，最多搜 8 轮。对照组是预算相同的基线，随机挑父题改写。

结果在表 2。每个 benchmark 只加 3 道题，前三大缺口之和就缩小了百分之 21.09 到百分之 38.88，平均百分之 32.67。基线平均只有百分之 17.52，理想上限是百分之 35.06。在三个 AIME 上，BenchFill 达到了这个上限。

选出来的题，和目标难度的平均偏差，每个 benchmark 都在 0.01 以内，基线平均是 0.048。

补题有没有实际用处。附录里，作者在三个 AIME 上各选一个缺口，用轻量的 SFT 让模型在这一段进步，再看能不能检测出来。

原版和加了基线题的版本，三个案例都没检测出显著提升，加了一道 BenchFill 题的版本，三个都检测出来了。

作者还把 AMC23 和三个 AIME 补成了新版本公开，题数从 40、30、30、30，扩到 80、50、52、53，所有新题都经过人工审核。

### 和以前方法的对比

对比主要有两组。补缺口这件事，是和预算相同的随机父题改写比，BenchFill 平均缩小的缺口，接近基线的两倍。

另一组在附录 D，按均匀分布把 benchmark 剪掉一部分题，看排名是不是和同领域其他 benchmark 更一致。在 8 个数学 benchmark 上，Spearman 相关平均从 0.890 升到 0.930，随机剪掉同样多的题只有 0.895。

我的判断是，这篇最有用的不是 BenchFill，而是表 1，它给 25 个常用 benchmark 做了一次体检，而且把难度估计公开了，任何人都能拿来查自己在用的 benchmark。

### 局限

作者自己写的局限，主要是一条，这套框架是事后的。先有 benchmark，再去估难度、查问题、补缺口。

还有几点要注意。难度是用 IRT 单参数模型估的，作者用 BIC 做过比较，25 个 benchmark 里有 21 个更支持单参数模型，剩下 4 个并不支持。

均匀分布只是默认目标，作者自己也说，如果难度配比符合你的评测目标，不均匀本身不是问题。

另外，补题帮助检测进步的实验只有三个案例，而且 SFT 是专门让模型在目标缺口附近进步的，是构造出来的场景。

### 未来方向

作者提的方向是，把这套思路前移到 benchmark 的构建阶段，从一开始就设计难度结构，既对准评测目标，又在难度轴上保持足够的分辨率。

他们也认为，测度这个视角，还能用来定义更多 benchmark 的问题和设计目标。

我的判断是，对做训练的人，最直接的启发是，别只盯着 AIME 这种几十道题的 benchmark 看进步，曲线不动，不一定是没进步，可能是 benchmark 在那一段正好没有题。

我的判断是，理论和诊断很扎实，但补题要靠大模型改写，再让一批模型反复做题定难度，成本不低，而且只在数学、BBH、MMLU-Pro 生物和 IFEval 上做过，Agent 和代码类 benchmark 还没有补题结果。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the University of Illinois Urbana-Champaign, with co-authors at UC San Diego and NYU. The title is a mouthful: your benchmark is an empirical measure over difficulty. According to its first page, it appears in the NeurIPS 2026 Evaluations and Datasets track.

It sounds abstract, but the question is very practical. How much should you trust a leaderboard score, or the eval curve you watch during training?

### Problem

Running a benchmark looks simple. Take a set of questions, run the model, average the results. That number gets used in two ways: to compare models on leaderboards, and to track whether a model is improving during training.

The authors' point is that the conclusion depends not only on how good the models are, but on how the questions are spread across difficulty.

This figure is the authors' sketch of two ways it goes wrong. On the left, two models are tied on easy questions and far apart on hard ones. If the benchmark is mostly easy questions, they look like a tie.

On the right, a model improves a lot in one band of difficulty, but the benchmark has no questions in that band. The score barely moves, and real progress goes unnoticed.

### Prior work

People have questioned benchmarks before. There's contamination, where test questions leak into training data, and work showing results swing with decoding, prompting and implementation details.

Estimating question difficulty isn't new either. One approach uses human labels: MATH, for instance, sorts problems into five levels. The authors argue that's costly and measures what humans find hard, not what models find hard.

The other approach is item response theory, or IRT, the psychometrics method for calibrating exam questions, already common on language model benchmarks. What this paper adds is treating the benchmark as a distribution along the difficulty axis, so both failures can be defined precisely.

### Key idea

Here's the core idea in one line. A benchmark isn't just a pile of questions. It scatters points along a difficulty axis, and each question gets equal weight at its own difficulty.

Seen that way, the benchmark score is just the model's performance at each difficulty, averaged with those points as weights. So the questions become: are the points in the right places, and are they spread evenly?

Points in the wrong places is what they call measure mismatch: the benchmark's difficulty distribution differs from the one you actually care about. Points spread unevenly is a coverage gap: a stretch of difficulty with no questions at all. To their knowledge, this is the first work to formalize these two as fundamental failure modes of benchmark evaluation.

### Method

They propose one diagnostic for each. The first is DDM, difficulty distribution mismatch: the Wasserstein distance between the benchmark's difficulty distribution and a target distribution. Intuitively, it's how much mass you'd have to move, and how far, to turn one distribution into the other.

Theorem 2.1 says that if the performance difference changes smoothly with difficulty, this distance caps how far the score gap on the benchmark can drift from the score gap under the target. And if two models differ by more than that bound under the target, their ranking can't flip.

So which target should you pick? The authors prove that if you have no idea where along the difficulty axis two models differ, the uniform distribution is the unique maximin choice, the safest choice in the worst case. They're careful to say that's not a claim that uniform is always best. It's a neutral default when you have no prior information.

The second diagnostic is the maximum gap. Sort the questions by difficulty and measure the widest empty stretch between neighbors.

Theorem 2.3 says that if a model's improvement is confined to a band narrower than that maximum gap, then in the worst case no test can detect it, however many times you re-run the evaluation. Only when the band is wider than the gap does repeated evaluation eventually catch it.

How do they estimate difficulty? With the one-parameter IRT model. Each model gets an ability score, each question gets a difficulty score, and the chance of a correct answer depends on ability minus difficulty.

For non-agentic benchmarks they picked twelve models, from one point seven billion to three hundred ninety-seven billion parameters, with up to thirty-two runs per question. Agentic benchmarks typically need strong closed models, so there they used published results from the HAL leaderboard.

The fit uses Bayesian inference with MCMC, with seven diagnostic checks, and all twenty-five benchmarks pass. For the non-agentic ones, leaving out any one model and refitting barely changes the difficulty estimates.

### Results

They analyze twenty-five popular benchmarks across seven domains: math, coding, agentic tasks, logic, science, instruction following and medicine. The numbers are in Table 1.

First, mismatch. Ten of the twenty-five have a DDM of at least zero point one, meaning that on average, questions would have to move at least ten percent of the difficulty range to match a uniform spread. MATH-500 is at zero point one six four, ScienceQA at zero point one seven eight, IFEval at zero point one five nine.

The authors' reading is that these benchmarks carry a strong built-in difficulty weighting. Their leaderboards shouldn't be read as difficulty-neutral, so check whether that weighting matches what you care about.

Now the gaps. Eight of the twenty-five have a maximum gap of at least zero point one, and nine have their five largest gaps summing to at least zero point three.

This figure shows AIME24 and AIME25, with the five largest gaps shaded. The authors point out that those five gaps add up to close to zero point five, leaving large parts of the difficulty axis unresolved, and these are exactly the benchmarks widely used to monitor RLVR, reinforcement learning with verifiable rewards.

And it isn't only a small-benchmark problem. HumanEval has one hundred sixty-four questions and a maximum gap of zero point one six five. BBH five objects has two hundred fifty questions and a maximum gap of zero point one two eight.

After the diagnosis comes a fix, which they call BenchFill: adding questions inside the gaps.

The idea is to pick a gap, set the target difficulty at its midpoint, and have a language model rewrite existing questions into a new one that lands right there. The new question is added; nothing is replaced.

It's an iterative search. A memory holds questions and their difficulties. Each round, the ones closest to the target become parents, the rewriter also sees a few reference questions just above and below the target, and it writes a batch of children.

In this figure, the parent says the numbers on n dice can multiply to five hundred fifty distinct products, and asks for n. One rewrite adds the condition that the product must be a perfect square, which changes the difficulty.

Each child is first checked by GPT-5.4 for a correct answer. Then the panel of models actually attempts it, its difficulty is estimated, and it goes back into memory for the next round. The child closest to the target wins, and human experts review it.

The setup: eight benchmarks, filling the three largest gaps in each, with MiniMax-M2.7 as the rewriter and at most eight rounds. The baseline has the same budget but picks parents at random.

The results are in Table 2. Adding just three questions per benchmark shrinks the sum of the three largest gaps by twenty-one point zero nine to thirty-eight point eight eight percent, thirty-two point six seven on average. The baseline averages seventeen point five two percent, and the idealized upper bound is thirty-five point zero six. On the three AIME benchmarks, BenchFill hits that bound.

The selected questions land close to target: the mean deviation is under zero point zero one on every benchmark, versus zero point zero four eight on average for the baseline.

Does filling gaps actually help? In the appendix, they pick one gap in each of the three AIME benchmarks, apply mild fine-tuning so the model improves around that gap, and test whether the improvement is detectable.

On the original benchmark, and on the version with the baseline's added question, none of the three cases shows a significant gain. With one BenchFill question added, all three do.

They've also released gap-filled versions of AMC23 and the three AIME sets, growing them from forty, thirty, thirty and thirty questions to eighty, fifty, fifty-two and fifty-three. Every new question was reviewed by hand.

### Comparison

There are two main comparisons. For gap filling, against a same-budget rewriter with random parents, BenchFill closes nearly twice as much of the gaps on average.

The second is in Appendix D: prune a benchmark toward a uniform spread and check whether its model ranking agrees better with other benchmarks in the same domain. Across eight math benchmarks, average Spearman correlation rises from zero point eight nine zero to zero point nine three zero, while randomly pruning the same number of questions gives zero point eight nine five.

My read is that the most useful part isn't BenchFill, it's Table 1. It's a health check on twenty-five widely used benchmarks, and the difficulty estimates are public, so anyone can audit the benchmarks they rely on.

### Limitations

The authors' own stated limitation is mainly one thing: the framework is post hoc. The benchmark exists first, and only then do you estimate difficulty, diagnose it and patch the gaps.

A few more things to keep in mind. Difficulty comes from a one-parameter IRT model. The authors checked this with BIC: twenty-one of the twenty-five benchmarks favor it, and the other four don't.

The uniform target is only a default. As the authors themselves say, non-uniform difficulty isn't a problem if it matches your evaluation goal.

And the experiment showing that filled gaps help detect progress covers only three cases, with fine-tuning deliberately aimed at the target gap. It's a constructed scenario.

### Future work

The direction the authors propose is moving this into benchmark construction itself: designing the difficulty structure from the start, so it matches the evaluation goal and keeps enough resolution along the difficulty axis.

They also see the measure view as a way to formulate more benchmark failures and design goals.

My read is that the most practical takeaway for anyone training models is this: don't judge progress only on benchmarks like AIME with a few dozen questions. A flat curve doesn't always mean no progress. The benchmark may simply have no questions where your model improved.

My read is also that the theory and diagnostics are solid, but gap filling relies on a model rewriting questions and a panel of models repeatedly solving them, which isn't cheap. And it's only been run on math, BBH, MMLU-Pro biology and IFEval, with no gap-filling results yet for agentic or coding benchmarks.

That's today's Daily Frontier Papers. The paper is linked in the description.

