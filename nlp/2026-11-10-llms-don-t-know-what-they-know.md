# LLMs Don't Know What They Know: Benchmarking Temporal Self-Knowledge for Financial Backtesting

**AI说不知道其实它知道**

- 论文 Paper: [arXiv 2611.06808](https://arxiv.org/abs/2611.06808)
- 单位 Affiliation: The Fin AI、曼彻斯特大学 等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-10
- 视频 Videos: [YouTube 中文](https://youtu.be/F0fWqp_qT24) · [YouTube English](https://youtu.be/d6HEXOsK3Vc) · [B站](https://www.bilibili.com/video/BV1xRW36BEym)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 The Fin AI 联合曼彻斯特大学、佛罗里达大学等机构的论文，标题很直接，大模型不知道自己知道什么。

它问的是金融圈一个很实际的问题，用大模型做历史回测的时候，你怎么确定，模型不是早就知道了答案。

### 问题

先说什么是回测。拿过去的某个时间点，让模型去预测之后的股价、通胀、利率，再和真实发生的结果对比，看它预测得准不准。

麻烦在于，这些所谓的未来，对模型来说其实是过去，结果很可能就在它的训练数据里。模型可能不是在预测，而是在背答案，论文管这叫事后信息泄漏，它会让预测能力看起来比实际更强。

实践里大家常用三道防护。第一，只在厂商公布的知识截止日期之后做评测。第二，在提示词里告诉模型，假设今天是过去某一天。第三，直接问模型，你知不知道这件事。

这篇论文要检验的，就是这三道防护到底靠不靠得住。

### 以前的方法

以前的工作大致分几类。一类是限制输入，只给模型当时能看到的资料，但这删不掉已经记在模型参数里的结果，作者引用的研究显示，模型被告知不要用，照样会回忆出来。

另一类是用困惑度或者模型的行为，去测知识到底在哪里结束，这些研究发现，实际的截止和公布的截止往往不一样。但它们没有问，模型自己知不知道这条边界在哪，而自我报告和声明日期这两道防护，恰恰依赖这一点。

还有一类是绕开问题。比如只做前瞻评测，等结果出来再算分，但这样没法评历史。或者用只拿某个日期之前数据训练的 point-in-time 模型，但这类模型目前都还很小。

### 核心想法

这篇的思路是，在同一批带日期的事实上，同时测两件事，一是模型实际知道什么，二是模型说自己知道什么。

把这两条线放到时间轴上，就能分别找到，知识真正结束的地方，和模型自称知识结束的地方，再和厂商公布的截止日期放在一起比。这张图就是全文结论的缩影。

关键设计是，同一组候选答案，一次单独给，一次配上我知道或者我不知道的声明一起给。这样，模型的答案和它对这个答案的说法，来自同一次选择。

### 方法

数据集叫 FinDendro，一共 2226 条美国的带日期事实，从 1999 年到 2026 年。包括美联储经济数据库 FRED 里 12 个月度宏观指标，比如通胀、失业率、国债利率和油价，50 家美国大公司的月末收盘价，还有美联储每个月的利率决定。

每个数字都按当时第一次公布的值来打分，不用后来修订过的数据，股价也把后来的拆股调整还原回去。这样答对了，说明模型记住的是当时的结果。

每条事实生成四个提示。任务一，直接做选择题，问这个值是多少，测的是模型实际知道多少。

任务二，每个候选值出现两次，一次配我知道，一次配我不知道、这是猜的，分回忆和预测两种问法。任务三，在预测问法前面加一句，假设今天是上个月的最后一天，按那天的情况回答。

这里有两个控制，挺重要。数字题如果直接选最接近上个月的那个值，有 47% 的题就能蒙对，所以这部分题不计入知识得分。

美联储的决定里，不变占了 78% 的月份，所以改用平衡准确率，每类结果权重一样，一直选不变只能拿三分之一。两个控制加起来，随机水平大约是 0.27。

控制之后，用来算知识得分的有 1340 条事实。知识结束点，是两个季度滚动准确率还高于一条线的最后一个季度，这条线取模型 2015 到 2020 年的水平和随机水平的中点。自称的边界，是声称知道的比例还没掉到 2015 到 2020 年水平一半以下的最后一个季度。

评测的是四个前沿模型，GPT-5、DeepSeek-V3.2、GPT-4o 和 Llama-3.3-70B，都通过 API 调用，支持的模型温度设为 0，每个提示只跑一次。另外还测了两个本地运行的 Qwen 小模型，和三类 point-in-time 模型。

### 实验和效果

第一个发现，前沿模型知道的比它说的多。结果在表 4，GPT-5 在控制后的题目上答对 76%，但只在 38% 的回答里声称自己知道。GPT-4o 答对 50%，声称知道的只有 20%。

这张图横轴是实际答对率，纵轴是声称知道的比例，对角线以下就是知道得多、说得少。两个 Qwen 小模型正好反过来，准确率只有 0.36 到 0.37，离随机水平 0.27 不远，却声称知道大约 70%。

第二个发现，声称知道，可以比真正知道持续得更久。Llama-3.3-70B 的准确率在 2021 年第二季度之后就掉到随机水平，但它一直声称知道到 2023 年第一季度，中间差了 7 个季度。

它的厂商公布截止是 2023 年 12 月，比测出来的知识边界晚了 10 个季度，在这个例子里，按公布日期选回测区间反而偏保守，会白白排除一大段模型其实已经不知道的时期。GPT-5 则不同，两条边界都在 2024 年第三季度，和厂商公布的截止一致。

第三个发现最关键，模型说不知道，不代表真不知道。GPT-5 声称知道的题，答对 95%，声称不知道的题，也答对了 61%。

作者模拟了一种常见做法，只保留模型说不知道的题来回测。表 6 显示，在 GPT-5 的知识窗口内，这样过滤后还留下 46% 的题，准确率仍然是 0.75，而知识边界之后只有 0.33。换句话说，靠我不知道来过滤，泄漏还在。

表 5 是几个具体例子。英特尔 2021 年 1 月的收盘价，GPT-5 说不知道，然后猜了 55.51，正好是正确答案。让它假设今天是 2020 年 12 月 31 日再预测，它还是说不知道，还是选了 55.51。

第四个发现，声明日期只改说法，不改知识。让 GPT-5 假设今天是过去某一天，声称事先知道的比例从 0.39 掉到 0.01，但准确率几乎没变，从 0.76 到 0.75。Llama 和 GPT-4o 也类似，DeepSeek-V3.2 的声称比例反而从 0.37 升到 0.46，准确率同样没变。

放到模拟回测里看，表 7，GPT-5 在知识窗口内把 99% 的回答标成预测，这些预测的准确率是 0.82，知识边界之后只有 0.39。也就是说，加了日期提示，背出来的答案会被包装成预测，让回测成绩虚高。

按事实类型拆开看，差距最大的是股价，GPT-5 股价题答对 59%，但只有 5% 的时候说自己知道。美联储的决定上，知道和说法就比较一致。

那用 point-in-time 模型行不行。作者测的这类模型只有 1 到 1.5B 参数，连选择题格式都跟不好，改用候选答案的似然来打分后，训练截止前后的准确率都在 0.25 到 0.30，接近随机，看不出截止前后有差别。

### 和以前方法的对比

这篇本身是一个 benchmark，没有新方法比旧方法高几个点的那种对比，它比的是三道常用防护和实测行为。论文的结论是，厂商截止日期、模型自我报告、声明日期，这三道防护都没法把已知结果和未知结果分开。

和 HindsightBench、Dated Data 这些测知识边界的工作相比，作者在表 1 里强调，FinDendro 多做的是，把模型的知识声明和实际答题放在同一批事实上比，并且直接检验这些防护。

我的判断是，这篇最有用的是一个提醒，模型说我不知道，只是一种说法，不是一种保证，而且越强的模型越是这样。做金融回测，在模型本身上测一遍知识边界，比相信它的自述可靠得多。

我的判断是，也要看到评测面不宽，前沿模型只有四个，其中 GPT-4o 和 Llama-3.3-70B 在图 1 里的公开发布都在 2024 年，结论能不能推到更新的模型上，还需要再测。

### 局限

作者列的局限有几条。第一，只覆盖美国一个经济体、三类月度金融事实，其他地区、其他类型的知识，或者文字类的事件，边界可能不一样。

第二，边界判断用的是固定阈值，换个阈值，估出来的季度会动。比如把阈值从一半改成三分之一，GPT-4o 的差距会从 1 个季度变成 6 个季度。

第三，模型不靠记忆也可能答对，比如顺着趋势外推，持续性控制只去掉了最简单的那种策略。另外，这里的知识声明是选择题形式，换成自由回答或者用概率表达，结果可能不同。

第四，前沿模型是通过 API 路由调用的，每个提示只看一次回答，线上服务的版本也可能随时间变化。point-in-time 模型又太小，还没法说明一个严格截止的大模型会不会诚实。

### 未来方向

作者给的建议很具体。做回测之前，先在目标领域的带日期事实上测出模型的知识边界，用它来选回测区间，并给边界的不确定性留出余量，模型更新了还要重新测。

对模型厂商，作者建议在公布训练截止日期之外，再补一份在带日期、分领域事实上的行为测评。数据池、题目文件和构建代码，论文说会随 benchmark 一起公开。

我的判断是，这件事不只关乎金融。任何拿历史数据评大模型预测能力的场景都有同样的问题，假设今天是某年某月这种提示，挡不住模型已经记住的东西。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from The Fin AI, with the University of Manchester, the University of Florida and others, and its title is blunt: LLMs don't know what they know.

The question is a practical one from finance. When you backtest a language model on history, how do you know it isn't just remembering the answer?

### Problem

Quick refresher on backtesting. You pick a past date, ask the model to forecast what came next, a stock price, inflation, a rate decision, and then score it against what actually happened.

The trouble is that the future you're asking about is the model's past. The outcome may well be sitting in its training data. So the model might not be forecasting at all, it might be recalling. The paper calls this hindsight leakage, and it makes forecasting skill look better than it is.

Practitioners lean on three safeguards. One, only evaluate after the vendor's stated knowledge cutoff. Two, tell the model to answer as if today were some past date. Three, just ask the model whether it knows the answer.

This paper tests whether any of those three can actually be trusted.

### Prior work

Earlier work falls into a few camps. One restricts the input, feeding the model only what was available at the time. But that can't remove outcomes stored in the weights, and the work the authors cite shows models recall them even when told not to.

Another camp measures where knowledge really ends, using perplexity or behavior, and finds that effective cutoffs differ from reported ones. What those studies don't ask is whether the model itself knows where that boundary is. And that's exactly what self-reports and declared dates depend on.

A third camp sidesteps the problem. Prospective evaluation waits for outcomes to arrive, so it can't score history. Point-in-time models, trained only on data before a fixed date, avoid leakage, but the ones available today are small.

### Key idea

The idea here is to measure two things on the same dated facts: what the model actually knows, and what it says it knows.

Put both on a timeline and you get two boundaries, where knowledge really ends and where the model claims it ends, which you can then set against the vendor's stated cutoff. This figure is the whole paper in one picture.

The key design trick: the same candidate answers are offered once on their own, and once paired with a statement, I know this, or I'm guessing. That way the model's answer and its claim about that answer come from a single choice.

### Method

The benchmark is called FinDendro. It has two thousand two hundred twenty-six dated U.S. facts from 1999 to 2026. That's twelve monthly macro series from the Fed's FRED database, things like inflation, unemployment, Treasury yields and oil, plus month-end closing prices for fifty large U.S. companies, plus monthly Federal Reserve rate decisions.

Every number is scored against the value first published at the time, not later revisions, and stock prices have later split adjustments reversed. So a correct answer means the model remembers what was reported back then.

Each fact turns into four prompts. Task one is a plain multiple-choice question: what was the value? That's the measure of what the model actually knows.

Task two shows each candidate value twice, once as I know this, and once as I don't know, here's my guess, in two wordings, Recall and Predict. Task three takes the Predict wording and adds a line in front: assume today is the last day of the previous month, and answer as you would on that date.

Two controls matter a lot here. For numerical facts, just picking the value closest to last month's would get forty-seven percent of them right, so those facts don't count toward the knowledge score.

For Fed decisions, unchanged is the right answer in seventy-eight percent of months. So the paper uses balanced accuracy, weighting each outcome equally, and always saying unchanged scores one third. With both controls, chance sits at about zero point two seven.

After the controls, one thousand three hundred forty facts count toward knowledge. Knowledge ends at the last quarter where two-quarter rolling accuracy stays above the midpoint between the model's 2015 to 2020 level and chance. Claims end at the last quarter where the share of I know answers stays above half its 2015 to 2020 level.

Four frontier models are tested, GPT-5, DeepSeek V3.2, GPT-4o and Llama 3.3 70B, all through APIs, at temperature zero where supported, one run per prompt. There are also two small Qwen models run locally, and three families of point-in-time models.

### Results

Finding one: frontier models know more than they say. In Table 4, GPT-5 gets seventy-six percent of controlled facts right, but claims to know the answer only thirty-eight percent of the time. GPT-4o gets fifty percent right and claims twenty.

On this chart, the x-axis is actual accuracy and the y-axis is how often the model says it knows. Below the diagonal means it knows more than it says. The two small Qwen models go the other way: accuracy of zero point three six to zero point three seven, not far above chance at zero point two seven, yet they claim to know about seventy percent.

Finding two: claims can outlast knowledge. Llama 3.3 70B's accuracy drops to chance after the second quarter of 2021, but it keeps claiming knowledge until the first quarter of 2023. That's a seven-quarter gap.

Its vendor-stated cutoff, December 2023, is ten quarters past its measured knowledge boundary. In this case, picking backtest periods by the stated date is actually too cautious, and throws away a long stretch where the model no longer knows the outcomes. GPT-5 is different: both of its boundaries land on the third quarter of 2024, matching its stated cutoff.

Finding three is the big one. I don't know doesn't mean the model doesn't know. On facts GPT-5 claims to know, it's right ninety-five percent of the time. On facts it says it doesn't know, it's still right sixty-one percent of the time.

The authors simulate a common fix: keep only the facts the model disclaims, and backtest on those. Table 6 shows that inside GPT-5's knowledge window, that filter keeps forty-six percent of facts, and accuracy on them is still zero point seven five, versus zero point three three after the boundary. In other words, filtering on I don't know leaves the leak in place.

Table 5 has concrete cases. Intel's closing price for January 2021: GPT-5 says it doesn't know, then guesses fifty-five point five one, which is exactly right. Tell it today is December thirty-first, 2020, and ask for a prediction, and it again says it doesn't know, and again picks fifty-five point five one.

Finding four: a declared date changes the words, not the knowledge. Telling GPT-5 to answer as of a past date drops its claims of prior knowledge from zero point three nine to zero point zero one, while accuracy barely moves, from zero point seven six to zero point seven five. Llama and GPT-4o behave the same way. DeepSeek V3.2's claims actually rise, from zero point three seven to zero point four six, again with accuracy unchanged.

In a simulated dated backtest, Table 7, GPT-5 labels ninety-nine percent of its in-window answers as predictions, and those predictions are right zero point eight two of the time, versus zero point three nine after its boundary. So a date prompt lets remembered answers pass as forecasts, and inflates the backtest.

Split by fact type, the biggest gap is stock prices. GPT-5 gets fifty-nine percent of them right but says it knows only five percent. On Fed decisions, what models know and what they claim line up much more closely.

What about point-in-time models? The ones tested are only one to one and a half billion parameters and struggle with the multiple-choice format. Scored by candidate likelihood instead, they sit near chance, zero point two five to zero point three, both before and after their training cutoffs, with no detectable change at the boundary.

### Comparison

This is a benchmark paper, so there's no new method beating old ones by a few points. What it compares is three standard safeguards against measured behavior. The verdict: a stated cutoff, a self-report and a declared date each fail to separate known outcomes from unknown ones.

Against boundary-measuring work like HindsightBench and Dated Data, the authors point to Table 1: what FinDendro adds is putting the model's knowledge claims and its actual answers side by side on the same facts, and testing the safeguards directly.

My read is that the real takeaway is a warning. A model saying I don't know is a statement, not a guarantee, and that's truer the stronger the model is. For financial backtests, measuring the knowledge boundary on the model itself is far more trustworthy than taking its word for it.

My read is also that the evaluation is narrow. There are only four frontier models, and two of them, GPT-4o and Llama 3.3 70B, were publicly released in 2024 according to Figure 1. Whether this holds for newer models still needs testing.

### Limitations

The authors list several limitations. First, it covers one economy and three families of monthly financial facts. Boundaries for other regions, other kinds of knowledge, or text-based events may differ.

Second, the boundary rule uses fixed thresholds, and other thresholds move the estimated quarters. Going from one half to one third of the distance to chance stretches GPT-4o's gap from one quarter to six.

Third, a model can be right without recall, for example by extrapolating a trend, and the persistence control only removes the simplest version of that. Also, the knowledge claims here are multiple choice; free-form or probabilistic statements might behave differently.

Fourth, the frontier models were reached through an API router, with one response per prompt, and served versions can change over time. And the point-in-time models are too small to show whether a large model with a strict cutoff would be honest.

### Future work

The authors' advice is concrete. Before backtesting, measure the model's knowledge boundary on dated facts from your own domain, choose evaluation periods from that, leave a margin for uncertainty, and re-measure whenever the model is updated.

For model developers, they suggest publishing behavioral knowledge assessments on dated, domain-specific facts alongside the stated training cutoff. The fact pool, question files and construction code, the paper says, will be released with the benchmark.

My read is that this isn't just a finance problem. Any time you score a model's forecasting on historical data, the same issue applies, and a prompt saying assume today is some past date won't block what the model already remembers.

That's today's Daily Frontier Papers. The paper is linked in the description.

