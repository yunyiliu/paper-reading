# PremiseCheck: Efficient Fact-Checking of Human and AI Peer Reviews

**审稿意见说错了谁来查**

- 论文 Paper: [arXiv 2610.21046](https://arxiv.org/abs/2610.21046)
- 单位 Affiliation: Lehigh University 等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-21
- 视频 Videos: [YouTube 中文](https://youtu.be/KACgr2kNX4E) · [YouTube English](https://youtu.be/86ElQsicrR0) · [B站](https://www.bilibili.com/video/BV1cunz6AEn5)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 Lehigh 大学、不列颠哥伦比亚大学和 Vector 研究所的论文，叫 PremiseCheck，给审稿意见做事实核查。

问题很具体，审稿意见说论文没做某件事，可论文其实做了，怎么又准又便宜地把这种错抓出来。

### 问题

投稿越来越多，不管是人写还是 AI 写的审稿意见，都会把论文内容说错。领域主席不可能为每条意见重读论文，作者也只能在 rebuttal 里纠正。

看一个真实例子。一条意见说，论文只在不公开的 Gopher 模型上测过，可复现性有限。可论文 5.6 节写了，他们用公开材料做了复现，附录给了 OPT-6B 模型的结果。专家把这条标成了事实错误。

任务是这样定义的。给一篇论文和一条意见，系统打分，分越低越可能有事实错误，比如误读论文，或者说论文缺了它其实有的东西。至于某个问题重不重要这种判断分歧，不在任务里。

为此作者做了基准 ReviewFact，覆盖 NLP、机器学习和自然科学，217 篇论文、9267 条意见，标签来自三个已有的专家标注数据集。事实错误很少，NLP 部分占 4.8%，自然科学只有 2.5%。

难在哪里。NLP 部分，一条意见中位数只有 17 个词，论文中位数却有 8500 个 token，反驳证据可能藏在任何地方。

而且意见很少直说。比如问一句，作者测过别的模型吗，其实已经预设了论文没测。

### 以前的方法

第一条路，把整篇论文和意见一起交给前沿大模型判断。准，但每查一条都要重读全文，几万份审稿乘下来很贵。

第二条路，用决策模型，论文里用的是 Jev。它不生成文字，只输出一个陈述在文本里成立的概率，通常不到一秒，成本几乎可以忽略。

评价指标是 AUROC，就是随机拿一条错的和一条对的意见，系统给错的那条打分更低的概率，50 等于瞎猜。

可直接拿整条意见问 Jev，NLP 部分只有 49.9，等于瞎猜。读全文的 GPT-5.5 是 63.2。

也有人先拆成小论断再核查，但还是让大模型读论文，拆得越细，贵的调用越多。

### 核心想法

作者发现，如果人工把意见写成一条条明确的前提，Jev 核查单条前提的准确率和 GPT-5.5 差不多。

所以关键洞察是，搞清楚一条意见在断言什么，需要大模型，但不需要论文。读论文才是最贵的部分，断言一旦写明白，就可以交给便宜的决策模型。

于是分工是，大模型只看意见，把它改写成几条明确前提。决策模型拿每条前提对照论文，整条意见的得分，就是最弱那条前提的得分。

### 方法

一共四步，拆解、分流、核查、汇总，这张图是一个完整例子。

第一步拆解。大模型把意见写成 1 到 6 条独立的陈述句，既要写明说的前提，也要写隐含的，包括提问背后的预设，但不许加审稿人没表达的内容。

图里这条意见说，实验部分没有比较任何 baseline。它被拆成三条，论文提出了一个方法，实验部分没有比较 baseline，以及 baseline 比较是评估方法的标准做法。

第二步分流。有的前提关于论文，有的靠常识。问一条常识前提根据论文是否成立，概率会被错误压低。所以 Jev 先只看前提，估计能不能单凭论文判断，概率不低于 0.5 算论文类，否则算常识类。

第三步核查。Jev 读论文加前提，论文类问根据论文是否成立，常识类只问是否成立。

例子里，第一条和第三条是 0.99 和 0.96，第二条只有 0.03。

第四步汇总。只要有一个前提错，整条意见就有事实错误，所以取最小值。这条意见得 0.03，被标出来交给人看。

为什么便宜。大模型不读论文，输入很短。决策模型不生成文字，同一段输入多问几个问题几乎不加钱。论文报告，NLP 部分所有决策模型调用加起来，每条意见 0.0015 美元。

### 实验和效果

设置是，GPT-5.5 拆解，Jev 核查，每条意见单独检查，就像审稿人刚写完一条就去查。参照系统是读全文、中等推理的 GPT-5.5。

判定标准叫非劣效，配对 AUROC 差值 95% 置信区间的下限高于负 3，就算不比参照差，这个门槛是实验前定的。

结果在表 1。拆解用中等推理时，三个领域分别是 63.0、68.7 和 67.5，参照是 63.2、66.9 和 67.8，成本大约一半。NLP 部分达到非劣效，另外两部分差异不显著。

拆解完全不推理，NLP 部分成本降到四分之一左右，AUROC 63.1，仍然非劣效，耗时只有 45%。另外两个领域掉 1.7 到 3.2 分，不显著。

更便宜的基线都会掉分。Jev 或 GPT-5.4-mini 直接读全文，低 5.7 到 13.4 分。先用 Jev 筛、再把七成可疑意见交给 GPT-5.5 的级联方案，花了大约四分之三的钱，每个领域都不如中等推理的 PremiseCheck。

换成开源权重的决策模型 Kev-27B，NLP 部分 AUROC 和 Jev 一样，配不推理的拆解，API 成本是参照的 17.8%，不算 GPU 开销。

### 和以前方法的对比

准确率从哪来，图 7 拆开看了。同样的前提，用 GPT-5.5 核查并不比 Jev 更准，成本却是参照的 3.1 倍。

四个不同厂商的决策模型，Jev、Kev-27B、Microsoft Decision-1 和 Perplexity Decider，直接判断整条意见只有 50 到 57 分，当核查器都追到 GPT-5.5 的一分以内。作者认为，准确率来自拆解，而不是某个模型。

省钱主要省在推理。推理约占 GPT-5.5 判断成本的一半，去掉推理它在 NLP 部分就掉到瞎猜水平。PremiseCheck 不怕，因为大模型只负责改写。

消融也验证了设计。在一组机器学习前提上，去掉分流，AUROC 从 61.8 掉到 58.8。用平均值代替最小值，掉到 59.4。

还有个几乎免费的技巧。Jev 只按输入收费，作者一次调用问 16 种问法，一半问论文是否支持，一半问是否矛盾，取平均后 AUROC 从 63.1 升到 65.7，提升来自矛盾类问法。

如果审稿已经写完，可以整份一起查。在 NLP 部分，这时不推理的 GPT-5.5 达到 67.8，成本 7.7%，PremiseCheck 是 69.0，成本 20%，差异不显著，GPT-5.5 反而更省。两者融合达到 71.5，显著高于整份查的 GPT-5.5。

实际使用要挑出少数意见给人看。PremiseCheck 挑出的最可疑 5%，命中真错误的比例是随机的两到三倍，GPT-5.5 虽然 AUROC 相近，却只比随机好一点。

举个例子。一条意见希望论文每个方面都有要点总结，Jev 给论文没有要点总结这条前提打了 0.04。作者在 OpenReview 回复，每个实验小节结尾都有总结。

我的判断是，这篇真正的价值在成本结构。最贵的读全文交给只按输入计费的模型，大模型只读意见和提示词，几百个 token，这种分工值得在别的长文档核查里试，不过论文只测了审稿。

我的判断是，绝对水平还不高。AUROC 在 63 左右，错误只占 4.8%，就算命中率翻两三倍，被标出的意见大多仍是对的，它是提醒工具，不是裁判。

### 局限

作者说得很清楚，它只管事实错误。把不重要的问题、不合理的要求也算成错误时，它落后于 GPT-5.5。

人工分析 164 个失败样本发现，NLP 部分的误报，58% 出在拆解，前提加了料、带偏了、或者模糊了原意。自然科学部分的误报，45% 出在核查。

自然科学部分哪些算事实错误，是 GPT-5.5 根据专家理由判的。两位作者复核了 178 条，和 GPT-5.5 标签的一致性 κ 是 0.76 和 0.77，换成人工标签，差异仍不显著。

另外，主力决策模型 Jev 是 API 服务，更多开源决策模型还有待测试。

伦理部分也提醒，误标正确意见可能让审稿人蒙受不公，所以它的定位是把人的注意力引过去，而不是替人判断。

### 未来方向

作者提的方向是，做一个更贴近审稿人原话的拆解器，判断类错误则需要别的方法。

他们设想两个使用时刻。审稿人每写一条就查一条，或者审稿写完后，领域主席或作者整份一起查，这时融合效果最好。

代码和 ReviewFact 数据都已公开，机器学习部分给的是重建脚本。

我的判断是，最可能落地的是写意见时的实时提醒，因为只需指出那条被论文反驳的前提，人一眼就能确认。但误报会不会让审稿人嫌烦，论文还没测。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Lehigh University, the University of British Columbia and the Vector Institute. It's called PremiseCheck, and it fact-checks peer reviews.

The problem is concrete. A review says the paper never did something, but it did. How do you catch that, accurately and cheaply?

### Problem

Submissions keep growing, and review comments, whether written by people or by AI, regularly misstate what a paper says. An area chair can't reread every paper to check every comment, and an author can only push back in the rebuttal.

Here's a real example. A comment says the work was only tested on Gopher, a model that isn't public, so reproducibility is limited. But Section 5.6 says the authors reproduced it with public material, and an appendix reports results with OPT-6B. Experts labeled the comment a factual error.

The task is defined narrowly. Given a paper and one comment, a system outputs a score, and low means a likely factual error, like misreading the paper or claiming it lacks something it contains. Disagreements of judgment, like whether a concern matters, are out of scope.

To measure this, the authors built ReviewFact, spanning NLP, machine learning and the natural sciences, with two hundred seventeen papers and nine thousand two hundred sixty-seven comments, labeled from three existing expert-annotated datasets. Factual errors are rare: four point eight percent of NLP comments, and two point five percent in the natural sciences.

Why is it hard? On the NLP reviews, the median comment is seventeen words, while the median paper is eighty-five hundred tokens, and the counter-evidence can be anywhere.

And comments rarely say things outright. "Did the authors test other models?" already presupposes they didn't.

### Prior work

Option one: give a frontier model the whole paper plus the comment. It's accurate, but it rereads the full paper for every comment, and across tens of thousands of reviews, that's expensive.

Option two: a decision model, here one called Jev. It doesn't generate text. It returns the probability that a statement holds in a document, usually in under a second, at negligible cost.

The metric is AUROC: take one wrong comment and one correct comment at random, and it's the chance the system scores the wrong one lower. Fifty is a coin flip.

But give Jev whole comments, and it scores forty-nine point nine on the NLP reviews. That's chance. GPT-5.5 reading the full paper gets sixty-three point two.

Some earlier work splits comments into smaller claims first, but still has a large model read the paper, so finer splits mean more expensive calls.

### Key idea

The authors noticed that when humans write a comment out as explicit premises, Jev checks those single premises about as well as GPT-5.5.

So the key insight: figuring out what a comment asserts takes a language model, but not the paper. Reading the paper is the expensive part, and once the assertions are explicit, a cheap decision model can do it.

Hence the division of labor. The language model sees only the comment and rewrites it as a few explicit premises. The decision model checks each premise against the paper, and the comment gets the score of its weakest premise.

### Method

Four steps: decompose, route, verify, aggregate. This figure walks through one example.

Step one, decompose. The model rewrites the comment as one to six self-contained statements, covering both explicit and implicit premises, including what a question presupposes, without adding anything the reviewer didn't say.

The comment here says no baselines are compared in the experiments. It becomes three premises: the paper proposes a method, the experiment section compares no baselines, and baseline comparisons are a standard way to evaluate methods.

Step two, route. Some premises are about the paper, others rest on general knowledge. Ask whether a knowledge premise is true according to the paper, and its probability drops for the wrong reason. So Jev first looks at the premise alone and estimates whether the paper can decide it. At zero point five or above it's a paper premise, otherwise a knowledge premise.

Step three, verify. Jev reads the paper with the premise. Paper premises get "true according to the paper?", knowledge premises just "true?".

Here, the first and third premises get zero point nine nine and zero point nine six. The second gets zero point zero three.

Step four, aggregate. One false premise means a factual error, so the score is the minimum. This comment ends up at zero point zero three and gets flagged for a person.

Why is it cheap? The language model never reads the paper, so its input is tiny. The decision model generates no text, so extra questions on the same input cost almost nothing. On the NLP reviews, all decision-model calls together come to about fifteen hundredths of a cent per comment.

### Results

The setup: GPT-5.5 decomposes, Jev verifies, and each comment is checked on its own, the way an assistant would check it right after a reviewer writes it. The reference is GPT-5.5 reading the full paper at medium reasoning.

The bar is non-inferiority: if the lower end of the ninety-five percent interval for the paired AUROC difference stays above minus three, it's no worse than the reference. That margin was set before the experiments.

Table 1 has the results. With a medium-reasoning decomposer, PremiseCheck scores sixty-three point zero, sixty-eight point seven and sixty-seven point five across the three fields, against sixty-three point two, sixty-six point nine and sixty-seven point eight, at roughly half the cost. It's non-inferior on NLP, and not significantly different on the other two.

Turn the decomposer's reasoning off, and on NLP the cost falls to about a quarter. AUROC is sixty-three point one, still non-inferior, in forty-five percent of the time. The other two fields lose one point seven to three point two points, not significantly.

Cheaper baselines lose accuracy. Jev or GPT-5.4-mini reading the full paper land five point seven to thirteen point four points lower. A cascade, Jev first, then the most suspicious seventy percent sent to GPT-5.5, costs about three quarters as much and still trails the medium-reasoning PremiseCheck in every field.

With the open-weight decision model Kev-27B, NLP AUROC matches Jev, and with the no-reasoning decomposer the API cost is seventeen point eight percent of the reference, not counting GPU time.

### Comparison

Where does the accuracy come from? Figure 7 pulls it apart. On the same premises, GPT-5.5 as the verifier is no more accurate than Jev, and costs three point one times the reference.

Four decision models from different providers, Jev, Kev-27B, Microsoft Decision-1 and Perplexity Decider, score fifty to fifty-seven judging whole comments, but all come within a point of GPT-5.5 as verifiers. The authors conclude the accuracy comes from decomposition, not from any one model.

The savings come mostly from reasoning. Reasoning is about half of GPT-5.5's cost, and without it, GPT-5.5 falls to chance on NLP. PremiseCheck doesn't care, because its language model only rewrites.

The ablations back the design. On a set of machine-learning premises, dropping routing lowers AUROC from sixty-one point eight to fifty-eight point eight. Averaging premises instead of taking the minimum gives fifty-nine point four.

There's also a nearly free trick. Jev bills only for input, so one call asks sixteen phrasings, half about support and half about contradiction, and averages them. AUROC rises from sixty-three point one to sixty-five point seven, and the gain comes from the contradiction questions.

For a finished review, you can check every comment at once. On the NLP reviews, GPT-5.5 without reasoning then reaches sixty-seven point eight at seven point seven percent of the cost, and PremiseCheck sixty-nine point zero at twenty percent. That's not a significant difference, so GPT-5.5 is the cheaper choice. Fusing the two reaches seventy-one point five, significantly above batched GPT-5.5.

In practice you'd flag a few comments for a person. PremiseCheck's top five percent hit real errors two to three times as often as random, while GPT-5.5, with a similar AUROC, does little better than random.

One example. A comment asks for a summary of takeaways for each facet, and Jev gives the premise that the paper has none a probability of zero point zero four. On OpenReview, the authors replied that every experiment section ends with one.

My read is that the real contribution is the cost structure. The expensive reading goes to a model that bills only for input, while the language model reads just the comment and its prompt, a few hundred tokens. That split seems worth trying on other long-document checks, though the paper only tests reviews.

My read, too, is that the absolute numbers are modest. AUROC is around sixty-three, and only four point eight percent of comments are wrong, so even at two to three times random, most flagged comments are still correct. It's a prompt for a human, not a referee.

### Limitations

The authors are clear that it only targets factual errors. When labels also count unimportant concerns or unreasonable requests, it falls behind GPT-5.5.

A hand analysis of one hundred sixty-four failures found that fifty-eight percent of NLP false alarms come from decomposition, where a premise adds to, redirects, or blurs what the reviewer said. In the natural sciences, forty-five percent come from verification.

GPT-5.5 decided which natural-science errors were factual, from the experts' rationales. Two authors rechecked one hundred seventy-eight, with kappas of zero point seven six and zero point seven seven against GPT-5.5, and with their labels, the gap is still not significant.

Also, the main decision model, Jev, is an API service, and more open decision models remain to be tested.

The ethics section warns that wrongly flagging a correct comment could unfairly discredit a reviewer. So it's meant to direct human attention, not to replace human judgment.

### Future work

Next, the authors want a decomposer that stays closer to the reviewer's words, and note that errors of judgment need different methods.

They picture two moments of use: checking each comment as a reviewer writes it, or, once the review is done, an area chair or author checking it all at once, where fusion works best.

The code and the ReviewFact data are public, with a rebuild script in place of the machine-learning part.

My read is that the live check while reviewing is the likeliest use, since it only has to point at the one premise the paper seems to refute, and a person can confirm that at a glance. Whether false alarms would annoy reviewers is something the paper hasn't tested.

That's today's Daily Frontier Papers. The paper is linked in the description.

