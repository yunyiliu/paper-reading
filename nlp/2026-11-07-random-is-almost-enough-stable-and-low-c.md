# Random Is Almost Enough? Stable and Low-Cost Benchmark Compression via BestRand

**评测只跑20%分数还准吗**

- 论文 Paper: [arXiv 2611.00883](https://arxiv.org/abs/2611.00883)
- 单位 Affiliation: 博世研究院、卡内基梅隆大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-07
- 视频 Videos: [YouTube 中文](https://youtu.be/TZbZF679-BQ) · [B站](https://www.bilibili.com/video/BV1em4s6hE7f)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇博世研究院和卡内基梅隆大学的论文，标题是个问句，随机抽题，是不是差不多就够了？他们提出的方法叫 BestRand。

它要解决的问题很实际，给大模型跑评测太贵了，能不能只跑一小部分题，就把完整的分数估出来。

### 问题

先说背景。给大模型、多模态模型跑 benchmark，要花大量时间和算力，因为这些 benchmark 题目往往很多，里面还可能有重复的题。

所以有一类方法叫 benchmark 压缩，从完整题库里挑一个小子集，比如只留 20% 的题，用模型在子集上的平均分，去估计它在全集上的分数。好坏用平均绝对误差来衡量，也就是估出来的分数和真实分数，平均差几个百分点。

### 以前的方法

以前的做法大致两类。一类是最简单的随机抽题。另一类是利用历史数据，拿一批已经测过的模型的答题记录，给题目做聚类、估难度，再挑有代表性的题，比如 Anchor Points、TinyBenchmarks、ZipBench，还有 SparseEval 这种直接训练一个预测器的。

后一类听起来更聪明，但好几种都要先拿到模型在整个题库上的答题记录，构造成本不低。而论文的表 1 显示，随机抽题平均下来其实已经很强，在 70 个基准上，误差比好几个复杂方法都小。

但随机抽题有个毛病，不稳定。大部分时候误差很小，偶尔抽到一个很偏的子集，误差会突然变大。你只抽一次，就不知道自己是不是刚好倒霉。

### 核心想法

这篇论文没有急着提新方法，而是先回答一个问题，随机抽题的误差，到底是什么分布。作者推出了一个解析近似，结论是长尾分布，大多数子集误差很小，少数子集误差大得多。上面这一排图，是语言模型、多模态模型和 VLA 模型各一个基准上，实测分布和理论曲线的对比，基本对得上。

长尾从哪来？关键是一个量，叫共同偏移。如果抽到的子集碰巧比全集简单，所有模型的分数会一起往上偏，碰巧难，就一起往下偏。

论文的定理 1 说，误差超过某个阈值的概率有一个下界，这个下界只由共同偏移的幅度决定。而这个幅度，会随着模型之间的相关性变大而变大。

这里的相关性，指的是不同模型在同样的题上，一起做对、一起做错的程度。论文统计的这些基准里，相关系数全部大于零，平均是 0.35。作者的解释是，现在的模型骨干相近，训练数据重叠，还有知识蒸馏让彼此的行为更接近，所以容易在同样的题上一起翻车。

由此得出一个有点反直觉的结论。只要相关性大于零，就算被测模型的数量趋于无穷，这个共同偏移也不会消失。多测几个相似的模型，抵消不掉它们共同的偏差。

定理 2 则解释了随机抽题为什么差不多够用。模型多的时候，误差分布的峰值，也就是最常见的那个误差，贴近这个分布的下端。所以大多数随机子集，本来就已经不错。

目标就清楚了，保留随机抽题的低成本，只想办法砍掉那条长尾。

### 方法

BestRand 的做法很简单，一共四步。第一步，从题库里随机抽一个题池，默认是目标子集的 4 倍大，让 5 个便宜的探针模型，也就是用来试水的模型，在题池上跑一遍。

第二步，算出探针模型在整个题池上的平均分，当作参照。第三步，从题池里随机抽 128 个候选子集，每个都算一下，探针模型在这个子集上的平均分，和参照差多远。

第四步，挑差得最小的那个子集，之后所有要评测的模型，都只在这个子集上跑。

关键在于，第三步不需要任何新的模型调用，只是对缓存好的分数求平均，论文说只用 CPU，不到 0.1 秒就能选完。额外的成本，只有探针模型在题池上跑的那一次。

为什么这样有用？定理 3 把共同偏移拆成两块，一块是探针模型的分数能解释的，一块是解释不了的残差。让子集上的探针平均分贴近全集，就能压住前一块，而且不需要知道探针和目标模型之间的具体关系，也不用先跑目标模型。

理想情况下，探针能解释的那部分偏移可以基本被消掉，只剩下解释不了的那部分。作者也说明，这是理想化的极限，在有限的题池里，并不保证一定能缩到这么小。

### 实验和效果

实验规模不小，一共 70 个基准，文本 44 个、多模态 16 个、Agent 10 个，包括 MMLU、MMMU、OSWorld 和 tau-bench 这些常见的。保留比例从 2% 到 50%，一共六档。

设置上，每个基准只用 5 个模型来构造子集，再在另一批完全没参与构造的模型上测误差。随机抽题和 BestRand 各重复了一万次。

结果在表 1。文本类所有档位平均，随机抽题的误差是 3.47 个百分点，BestRand 是 2.92。Agent 类，从 6.14 降到 5.43。和随机抽题、ZipBench、TinyBenchmarks 这三个有竞争力的方法相比，作者称平均误差降低了 16.4%。

不过要说清楚，绝对差距不大。在 20% 的保留比例下，文本类随机抽题的误差是 2.11，BestRand 是 1.73，差了不到 0.4 个百分点。

更明显的是稳定性。这张图是 20% 保留比例下，六个基准上两种方法的误差分布。蓝色的随机抽题拖着一条长尾巴，橙色的 BestRand 集中在零附近，图上标出的 95 分位误差，降了 36% 到 57%。

作者还专门量了高误差风险，就是误差落进随机抽题最差 30% 区间的概率。文本类，随机抽题自己是 29.2%，BestRand 是 12.8%。三类基准上 BestRand 都最低，和三个强基线相比，作者称高误差的概率降低了 50.2%。

成本方面，BestRand 只让探针模型跑题池，构造时用到的答题记录，平均只有那些依赖历史数据的方法的 51.3%，在 2% 保留比例下，只有 8%。

再看分布外测试，也就是构造子集用的模型，和被测的模型不是一类，比如用老模型构造、测新模型，或者换一个模型家族。表 2 显示，那些从历史记录里学习的方法，在这些设置下全都不如随机抽题，而 BestRand 在五种设置下误差都最低，平均 1.69，随机抽题是 1.95。

作者还把方法用到了 VLA 评测上，比如自动驾驶，这类评测每道题都是一次完整的交互式任务执行，特别贵。在 Bench2Drive 上，删掉 65% 的路线时，平均误差从随机抽题的 2.84 降到 2.46。

### 和以前方法的对比

和以前的方法比，BestRand 的思路是反着来的。别人用历史数据去学一个更聪明的挑题规则，它基本还是随机，只是多抽几次，再用几个探针模型，挑一个最不偏的。

消融实验也说明它不挑剔。匹配时用 L1、L2 还是最大值距离，结果没有显著差别。探针模型怎么选影响也不大，唯一要避免的，是只用分数低的模型当探针。

不过有一点要注意。图 4 里，当可以用来构造的模型增加到几百、上千个时，学习型的方法会追上来，BestRand 只是和最强的方法差不多。它的优势，主要在手里模型少的场景。

我的判断是，这篇最有价值的不只是 BestRand 本身，而是那条理论结论，模型越像，随机抽题的长尾风险越大，而且多测模型也消不掉。这让只跑一部分题这件事，有了一个能估算的风险。

我的判断是，它的短板也很明显，在排名上基本没有提升。附录表 8 里，文本类的 Spearman 排名相关，BestRand 是 0.922，随机抽题是 0.921。如果你主要关心的是模型之间谁排在前面，它帮不上太多忙。

### 局限

作者自己写了两条局限。第一，误差不可能为零，压缩后的基准不能完全替代完整评测，作者建议最终结果在可行时还是报告全量评测。

第二，构造成本不是零，探针模型还是要在一部分题上完整跑一遍推理。

从附录还能看到两点。一是检验规模，每个基准用来测试的模型并不多，很多是 20 个，少的只有 5 个。二是理论建立在简化假设上，比如所有模型两两之间相关程度相同，作者也说，这是有用的近似，而不是对数据的精确描述。

### 未来方向

往后看，作者已经把它扩展到多指标的场景，在 Bench2Drive 上同时匹配成功率、驾驶分数等四个指标，删掉 50% 路线时，平均误差从 1.89 进一步降到 1.47。作者表示，会公开全部代码和数据。

我的判断是，这个思路对需要频繁评测的团队很实用，比如训练过程中隔一段就要看一次分数。先用几个模型挑好一个子集，之后反复用，成本很低。但真正要对外发布的数字，还是应该跑全量。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Bosch Research and Carnegie Mellon University, and its title is a question: is random almost enough? The method it proposes is called BestRand.

The problem is a practical one. Evaluating large models is expensive, so can you run just a small slice of a benchmark and still recover the full score?

### Problem

Some background. Running benchmarks on language and vision-language models takes a lot of time and compute, because benchmarks tend to have many test items, and some of those items may be redundant.

That's where benchmark compression comes in. You keep a small subset, say twenty percent of the items, and use a model's average on that subset to estimate its full-benchmark score. Quality is measured by mean absolute error: on average, how many percentage points the estimate is off.

### Prior work

Existing approaches fall into two camps. One is plain random sampling. The other uses history: response records from models that were already evaluated, which methods like Anchor Points, TinyBenchmarks and ZipBench use to cluster items and pick representative ones, while SparseEval trains a predictor outright.

The second camp sounds smarter, but several of these methods need full-benchmark responses before they can build anything. And Table 1 shows that random sampling is already strong on average: across seventy benchmarks, it beats several of the more elaborate methods.

The problem with random sampling is stability. Most draws are fine, but every so often you get a lopsided subset and the error jumps. If you only draw once, you can't tell whether you got unlucky.

### Key idea

So before proposing anything, the paper asks a basic question: what does the error distribution of random sampling actually look like? The authors derive an analytical approximation, and the answer is a long tail. Most subsets give small errors, a few give much larger ones. The top row of this figure compares measured and theoretical distributions on three benchmarks, one each for language, vision-language and VLA models, and they line up closely.

Where does the tail come from? The key quantity is what the authors call the common shift. If the subset you drew happens to be easier than the full benchmark, every model's score drifts up together. If it's harder, they all drift down.

Theorem one says the probability of the error exceeding a given threshold has a lower bound, and that bound depends only on the size of this common shift. And that size grows as the models become more correlated with each other.

Correlation here means how much different models get the same items right and the same items wrong. Across the benchmarks the paper examines, that correlation is positive every time, with a mean of zero point three five. The authors' explanation is that today's models share similar backbones and overlapping training data, and distillation pulls their behavior even closer, so they tend to fail on the same items.

That leads to a slightly counterintuitive result. As long as correlation is above zero, the common shift doesn't vanish, even as the number of evaluated models goes to infinity. Evaluating more similar models doesn't average away their shared error.

Theorem two explains the other half, why random is almost enough. With many models, the most likely error, the peak of the distribution, sits close to the low end of that distribution. So most random subsets are already good.

That makes the goal clear. Keep the low cost of random sampling, and cut off the long tail.

### Method

BestRand itself is simple, four steps. Step one: draw a random pool of items, by default four times the target subset size, and run five cheap probe models, models used only to test the waters, on that pool.

Step two: take the probes' average scores over the whole pool as a reference. Step three: draw one hundred twenty-eight random candidate subsets from the pool, and for each one, measure how far the probes' average on that subset is from the reference.

Step four: keep the candidate with the smallest gap, and evaluate every target model on that subset alone.

The key is that step three makes no new model calls; it just averages cached scores, and the paper says selection runs on a CPU in under a tenth of a second. The only extra cost is that single probe pass over the pool.

Why does this help? Theorem three splits the common shift into two parts, the part the probe scores can explain and a residual they can't. Matching the probe averages on the subset to the full set controls the first part, without knowing exactly how the probes relate to the target models, and without running the target models first.

In the ideal limit, the probe-explained part of the shift is essentially removed, leaving only the residual. The authors are careful to say this is an idealized limit, not a guaranteed reduction when selecting from a finite pool.

### Results

The evaluation is broad: seventy benchmarks, forty-four text, sixteen multimodal and ten agent benchmarks, including familiar ones like MMLU, MMMU, OSWorld and tau-bench. Retention ratios run from two to fifty percent, six levels in all.

For each benchmark, only five models are used to build the subset, and the error is measured on a separate set of models that played no part in building it. Random sampling and BestRand are each repeated ten thousand times.

The results are in Table 1. Averaged over all retention levels, random sampling on text benchmarks is off by three point four seven percentage points, and BestRand by two point nine two. On agent benchmarks, the error drops from six point one four to five point four three. Against the three competitive baselines, random sampling, ZipBench and TinyBenchmarks, the authors report a sixteen point four percent reduction in average error.

To be clear, the absolute gap is small. At twenty percent retention on text, random sampling's error is two point one one and BestRand's is one point seven three, a difference of under zero point four points.

The bigger difference is stability. This figure shows the error distributions of both methods on six benchmarks at twenty percent retention. Random sampling, in blue, trails a long tail; BestRand, in orange, stays bunched near zero, and the ninety-fifth percentile errors marked on the plots drop by thirty-six to fifty-seven percent.

The authors also measure high-error risk directly: the chance of landing in the worst thirty percent of random sampling's errors. On text, random sampling itself sits at twenty-nine point two percent, and BestRand at twelve point eight. BestRand is lowest in all three categories, and against the three strong baselines, the authors report a fifty point two percent cut in high-error probability.

On cost, because BestRand only runs probes on the pool, it uses on average fifty-one point three percent of the construction records that history-based methods need, and just eight percent at two percent retention.

Then there's out-of-distribution testing, where the models used to build the subset differ from the models being tested: older models to build and newer ones to test, for example, or a different model family. Table 2 shows that every method that learns from historical records does worse than plain random sampling in these settings, while BestRand has the lowest error in all five, averaging one point six nine against random's one point nine five.

The authors also take the method to VLA evaluation, such as autonomous driving, where every item is a full interactive task rollout and therefore expensive. On Bench2Drive, with sixty-five percent of routes removed, the average error falls from random sampling's two point eight four to two point four six.

### Comparison

Compared with earlier methods, BestRand goes the opposite way. Others use historical data to learn a cleverer selection rule; BestRand stays essentially random, and simply draws several times and lets a few probe models pick the least skewed subset.

The ablations show it isn't fussy. Matching with L1, L2 or max-norm distance makes no significant difference. How you pick the probe models matters little too; the one thing to avoid is using only low-scoring models as probes.

One caveat, though. In Figure 4, as the number of models available for construction grows to hundreds or thousands, the learning-based methods catch up, and BestRand is merely comparable to the best of them. Its edge is mainly when you have few models to work with.

My read is that the most valuable part of this paper isn't just BestRand, it's the theoretical result: the more alike the models are, the heavier the tail risk of random subsets, and testing more models won't fix it. That gives running a partial benchmark a risk you can actually estimate.

My read is also that there's a clear weakness: it barely improves rankings. In appendix Table 8, rank correlation with the full benchmark on text is zero point nine two two for BestRand and zero point nine two one for random sampling. If what you mostly care about is which model ranks above which, it won't help much.

### Limitations

The authors list two limitations. First, the error is never zero, compressed benchmarks can't fully replace a complete evaluation, and they recommend reporting full-benchmark results for final evaluations when feasible.

Second, construction isn't free: the probe models still have to run full inference on part of the benchmark.

The appendix shows two more things. One is the size of the test panel: each benchmark is tested on relatively few models, often twenty, and as few as five. The other is that the theory rests on simplifying assumptions, such as every pair of models being equally correlated, which the authors describe as useful approximations rather than exact descriptions of the data.

### Future work

Looking ahead, the authors already extend the method to multiple metrics: on Bench2Drive, matching four metrics at once, including success rate and driving score, cuts the average error from one point eight nine to one point four seven with half the routes removed. They say all code and data will be made public.

My read is that this is genuinely useful for teams that evaluate constantly, for example checking scores every so often during training. Pick a subset once with a few models, then reuse it cheaply. But numbers you publish should still come from the full benchmark.

That's today's Daily Frontier Papers. The paper is linked in the description.

