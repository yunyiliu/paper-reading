# Retrieval Augmented Generation Compositionality

**RAG技巧叠加为何打了折**

- 论文 Paper: [arXiv 2610.29722](https://arxiv.org/abs/2610.29722)
- 单位 Affiliation: IBM、魏茨曼科学研究所
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-24
- 视频 Videos: [B站](https://www.bilibili.com/video/BV122HK6DEov)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 IBM 和魏茨曼科学研究所的论文，题目叫 Retrieval Augmented Generation Compositionality，意思是 RAG 的可组合性。

它问的问题很实际，RAG 里那些单独测都有效的技巧，放到一起还有没有效。作者的答案是，效果往往会大打折扣。

### 问题

先解释一下 RAG。RAG 叫检索增强生成，模型回答问题之前，先从一个文档库里把相关资料找出来，再根据这些资料写答案。

现在一条 RAG 流水线有很多环节。论文研究了八个，embedding，也就是把文字变成向量，然后是切块，建索引时给文本块补充信息，检索，改写查询，重排序，压缩上下文，最后还有 Agent 式的质量检查。

而且 RAG 要同时看两件事。一是检索质量，找回来的文档对不对，论文用 NDCG@10 衡量，简单说就是前十个结果排得好不好。二是忠实度，答案是不是真的有检索到的资料支撑，用 RAGAS 这个工具，让大模型来打分。

### 以前的方法

问题在于，新技巧大多是一个环节一个环节单独测的，脱离了周围的流水线。比如换一个重排序模型，别的都不动，分数涨了，就写进论文。

也有工作做过组合搜索，比如网格搜索、贪心搜索和自动配置搜索，能找到不错的配置。但作者指出，这些做法没有直接测量组件之间的相互作用，也就是一个组件的效果，会不会取决于别的组件，有的只报告最后赢的那组配置。

还有一个默认的假设。文献里大多数 RAG 消融实验，是在 5000 到 5 万篇文档的规模上做的，而 RAG 真正服务的文档库，往往要大一个数量级。

### 核心想法

这篇论文不提出新技巧。作者的思路是，把组合本身当成研究对象，在实验之前先写下三个假设。

第一，单独测出来的提升，组合起来不一定还能叠加。第二，一条简单的流水线，可能在很多场景下就接近最优。第三，不同组合的排名，会随着文档库变大而改变。

### 方法

为了测这些，他们做了一个开源框架，叫 LASAGNA。每个环节都有固定接口，换掉一个组件，不会悄悄改变别的环节。加一个新方法，只要一行代码，就能放进完整的流水线里测。

这张图是框架的总览。框架自带 53 个组件，支持六种向量数据库，也带好了这次用到的八个基准。

实验设计上，八个环节每个有 2 到 5 个选项，全部组合是 30720 种，作者估计要超过 4 万 GPU 小时。所以他们用了一种统计上的实验设计，叫 D-optimal 设计，挑出一个子集，保证每个单独效果和 28 对两两交互都能估计出来，最后实际跑完了 167 组。

再加上 24 组针对性的消融实验，一共 138 个配置，还有从 5000 篇到 20 万篇文档的规模实验，论文说总共测了 450 个配置。

测试数据方面，他们把八个基准合进同一个文档库，覆盖生物医学、金融、科学、辩论和多跳问答，这样检索时要在不同领域的干扰文档里找到对的那篇。答案由 GPT-5.1 生成，忠实度也由 GPT-5.1 打分。论文报告，整个研究花了大约 5200 美元。

作者还从实验里提炼出四套完整配置，叫 profile。SPEED 追求速度，STANDARD 是直接生成答案的标准配置，BALANCED 和 MAX-RELIABLE 则加上了 Agent 式的三轮验证。

### 实验和效果

第一个发现，检索上的提升叠不起来。基线流水线的 NDCG@10 是 0.412。单独测，效果最大的三个技巧，是交叉编码器重排序，关掉 MMR 这个多样性机制，以及给文本块加上文档背景的前缀。

把这三项单独的提升加起来，预计能提高百分之46.6。但把它们组合在一起的 BALANCED 流水线，实际只提高了百分之17.2，比预期少了百分之63。

这张瀑布图，是把五个单独有效的检索技巧都算上的情况，预期的提升损失了百分之72。论文报告，在测过的 17 种组合里，损失从百分之7 到百分之72 不等，中位数是百分之41。

作者的解释是功能重叠。比如重排序本身已经提供了一部分过滤效果，再关掉 MMR，能帮上的空间就少了。

第二个发现，检索好和答案忠实，几乎是两回事。看这张热力图，上半部分是检索，影响最大的是 embedding 和补充信息，下半部分是忠实度，影响最大的换成了改写查询和 Agent 式的质量检查。

在配对的配置上，两个指标的相关系数是负 0.01，基本不相关。有百分之5 的配置忠实度高、检索差，还有百分之6 检索好、忠实度低。所以只看一个指标选流水线，很可能选错。

忠实度这边，情况反而接近可加。四个技巧单独的提升加起来，预计提升百分之28，最好的完整配置实际提升了百分之32。不过作者也说明，这几个数不是同一套组件，只能算探索性的信号。

第三个发现，规模一变，排名就变。他们把文档库从 5000 篇逐步扩大到 20 万篇。这张图里，SPEED 和 STANDARD 的检索分数随规模下降，BALANCED 和 MAX-RELIABLE 基本持平。

5000 篇和 15000 篇的时候，STANDARD 领先，从 25000 篇开始，BALANCED 反超。到 20 万篇，BALANCED 的检索分数比 STANDARD 高百分之13.7，这个差距在统计上是显著的。

有意思的是，忠实度没有随规模下降，三套重视质量的配置都在 5 万篇左右最高，10 万篇回落，到 20 万篇又回升。作者的猜测是，文档库越大，就算标准答案那篇没排进前几名，别的文档里也有相近的内容，足够支撑答案。

最后看四套配置的正面对比，在表 2。SPEED 最快，平均 0.7 秒，但检索和忠实度都最低。STANDARD 检索最好，NDCG@10 是 0.860，平均 5.3 秒。

两套 Agent 配置忠实度最高，BALANCED 是 0.911，MAX-RELIABLE 是 0.916，代价是平均 12 到 14 秒。把检索、忠实度和延迟乘在一起算一个综合指标，在 30 秒的参考响应时间下，STANDARD 最高，没有一套配置在所有指标上都赢。

### 和以前方法的对比

这篇和以前工作的区别，不在分数，而在测法。论文这张表，把它和六项之前的 RAG 评测放在一起，表里只有它同时估计了组件交互，并且预先声明了假设。

还有几个实用的小发现。2019 年的一个 MiniLM 交叉编码器，排在测试的四个新重排序模型前面。HyDE，也就是先让模型写一个假想的答案，再拿去检索，能让 NDCG@10 提高百分之3.1，但第一跳的延迟翻倍。

生成端还有一个反直觉的结果。关掉幻觉检查和答案质量检查，忠实度反而最高，因为检查触发的修正会删掉支撑证据，在一个多跳问题里，15 个文本块删掉了 14 个。

换更强的生成模型，也能看到组合效应。把 GPT-5.1 换成 Claude Sonnet 4.5，最轻的 SPEED 忠实度提高 0.184，BALANCED 只提高 0.027，而且不显著。作者认为，流水线的复杂程度和生成模型的能力，在一定程度上可以互相替代。

我的判断是，这篇最大的价值，是把大家隐约知道的一件事量化了。单独有效的技巧叠在一起会打折，打多少折，取决于看哪个指标、文档库有多大。对做 RAG 的工程师来说，加组件之前，应该先在自己的完整流水线、自己的规模上测一遍。

### 局限

局限作者写得很坦白。整个评测只有一个构造出来的英文测试集，每个实验最多 93 个检索查询、200 个生成查询，集中在五个领域，没有测多语言、真实企业数据，也没有测开源的生成模型。

忠实度的分数依赖大模型裁判。他们让 Claude Opus 4.7 当第三个裁判，和主裁判的相关系数只有 0.44，所以作者的主要结论都依赖排名这类相对比较，绝对分数只当作近似值。

还有一点，他们预先声明的第一个假设检验，其实没有通过。原本要求两两交互能解释至少百分之20 的忠实度差异，结果达不到。后续的专门实验证明交互确实存在，但远低于这个门槛。另外，这些假设是内部哈希存档的，没有在第三方平台正式预注册。

### 未来方向

作者呼吁，新的 RAG 技巧应该放进有代表性的完整流水线里测，两个指标都看，而且在实际要用的规模上测。他们开源了 LASAGNA，希望组合评测成为报告新方法的标准做法。不过论文也说明，设计矩阵和逐条查询的原始分数不在开源代码里，需要向作者索取。

我的判断是，这篇也有明显的弱点。检索打折的核心数字，来自 93 个查询和作者自己的基线，换一套领域和基线，打折的幅度可能不一样，17 种组合的损失从百分之7 到 72，本身就说明了这种不确定性。

我的判断是，接下来值得看的，是有没有人用这个框架，在更大的真实企业文档库和开源模型上，把这些实验重做一遍。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from IBM and the Weizmann Institute of Science, and it's called Retrieval Augmented Generation Compositionality.

The question is a practical one. If every trick in your RAG pipeline helps on its own, do they still help when you stack them? The authors' answer is: often, not nearly as much as you'd expect.

### Problem

Quick refresher. RAG, retrieval-augmented generation, means that before a model answers, it pulls relevant documents from a corpus and writes its answer from them.

A modern RAG pipeline has a lot of stages. The paper studies eight: the embedding model that turns text into vectors, chunking, enriching chunks at index time, retrieval, query augmentation, reranking, context compression, and agentic quality gates at the end.

And RAG gets judged on two separate things. One is retrieval quality, did you fetch the right documents, measured here with NDCG at ten, which roughly asks how well the top ten results are ranked. The other is faithfulness, is the answer actually supported by what was retrieved, scored by an LLM judge through a tool called RAGAS.

### Prior work

The problem is that most new techniques are evaluated one stage at a time, cut off from the rest of the pipeline. Swap in a new reranker, hold everything else fixed, the score goes up, and that's the paper.

Some work has searched over combinations, with grid search, greedy search, or automated configuration search, and that can find good settings. But the authors point out that these approaches don't directly measure interactions, meaning whether one component's benefit depends on what else is in the pipeline. Some only report the winning configuration.

There's a hidden assumption about scale, too. Most RAG ablations in the literature run on five thousand to fifty thousand documents, while the corpora RAG actually serves are often an order of magnitude larger.

### Key idea

This paper doesn't propose a new technique. Instead, it treats composition itself as the thing to study, and the authors wrote down three hypotheses before running anything.

One, gains measured in isolation won't necessarily add up when you combine components. Two, a simple pipeline may be close to optimal in many settings. Three, the ranking of different pipelines changes as the corpus grows.

### Method

To test this, they built an open-source framework called LASAGNA. Every stage sits behind a fixed interface, so swapping one component can't quietly change another stage. Adding a new method takes a single line of code, and it gets tested inside a complete pipeline.

Here's the overview figure. The framework ships with fifty-three built-in components, six vector database backends, and adapters for the eight benchmarks used in the study.

On the design side, the eight stages have two to five options each. The full grid is thirty thousand seven hundred twenty configurations, which the authors estimate would take more than forty thousand GPU hours. So they used a statistical design called D-optimal design to pick a subset that still lets them estimate every main effect and all twenty-eight pairwise interactions. They ended up running one hundred sixty-seven configurations.

On top of that come one hundred thirty-eight configurations across twenty-four targeted ablations, plus a scaling sweep from five thousand to two hundred thousand documents. In total, the paper says, four hundred fifty configurations.

For data, they merge the eight benchmarks into one shared corpus, spanning biomedical, financial, scientific, argumentative and multi-hop questions, so the retriever has to find the right document among distractors from other domains. GPT-5.1 writes the answers and also judges faithfulness. The paper reports the whole study cost about five thousand two hundred dollars.

From all this, the authors distill four complete pipelines they call profiles. SPEED is built for latency. STANDARD is the standard setup that generates the answer directly. BALANCED and MAX-RELIABLE add an agentic loop with three verification iterations.

### Results

Finding one: retrieval gains don't stack. The baseline pipeline scores zero point four one two on NDCG at ten. Measured alone, the three biggest wins are cross-encoder reranking, turning off MMR, which is a diversity mechanism, and adding document-awareness prefixes to chunks.

Add those three separate gains together and you'd predict a forty-six point six percent lift. The BALANCED pipeline, which combines them, actually delivers seventeen point two percent. That's a sixty-three percent shortfall against the prediction.

This waterfall chart counts all five retrieval techniques that helped on their own, and there seventy-two percent of the predicted gain is lost. The paper reports that across seventeen tested combinations, the shortfall ranges from seven to seventy-two percent, with a median of forty-one.

The authors' explanation is functional overlap. Reranking already does part of the filtering you'd get from turning off MMR, so once both are in, each has less room to help.

Finding two: good retrieval and faithful answers are almost separate problems. In this heatmap, the upper triangle is retrieval, where the embedding model and enrichment matter most. The lower triangle is faithfulness, where the biggest factors are query augmentation and agentic gating instead.

Across matched configurations, the correlation between the two metrics is minus zero point zero one. Basically none. Five percent of configurations are faithful with poor retrieval, and six percent retrieve well but aren't faithful. Pick your pipeline on one metric and you can easily pick wrong.

Faithfulness, interestingly, comes closer to adding up. Four separately measured gains predict a twenty-eight percent improvement, and the best complete configuration reaches thirty-two percent. But the authors are careful here: those numbers don't come from one matched component stack, so it's an exploratory signal, not a controlled test.

Finding three: change the scale and the ranking changes. They grow the corpus from five thousand to two hundred thousand documents. In this figure, SPEED and STANDARD lose retrieval quality as the corpus grows, while BALANCED and MAX-RELIABLE stay roughly flat.

At five thousand and fifteen thousand documents, STANDARD leads. From twenty-five thousand on, BALANCED takes over. At two hundred thousand, BALANCED beats STANDARD on retrieval by thirteen point seven percent relative, and that gap is statistically significant.

Here's the odd part: faithfulness doesn't fall with scale. The three quality-focused profiles peak around fifty thousand documents, dip at a hundred thousand, and recover by two hundred thousand. The authors' guess is that a bigger corpus has more coverage, so even when the gold document slips out of the top results, other documents carry similar content that can still ground the answer.

Finally, the head-to-head in Table 2. SPEED is the fastest, at zero point seven seconds on average, but it's lowest on both retrieval and faithfulness. STANDARD has the best retrieval, zero point eight six zero, at about five point three seconds.

The two agentic profiles lead faithfulness, zero point nine one one for BALANCED and zero point nine one six for MAX-RELIABLE, at a cost of roughly twelve to fourteen seconds. Combine retrieval, faithfulness and latency into one volume score with a thirty-second reference, and STANDARD comes out on top. No profile wins on every metric.

### Comparison

What sets this paper apart from earlier work isn't a score, it's the method of measurement. This table lines it up against six previous RAG evaluations, and it's the only one in the table that estimates component interactions and pre-declares its hypotheses.

There are some useful side findings too. A twenty nineteen MiniLM cross-encoder outranks all four newer rerankers they tested. And HyDE, where the model first drafts a hypothetical answer and searches with that, lifts NDCG at ten by three point one percent but doubles first-hop latency.

On the generation side there's a counterintuitive one. Faithfulness is highest with the hallucination and answer-quality gates switched off, because the refinement they trigger can strip out supporting evidence. In one multi-hop example, fourteen of fifteen chunks were removed.

Swapping in a stronger generator shows a composition effect as well. Replacing GPT-5.1 with Claude Sonnet 4.5 raises faithfulness by zero point one eight four for the light SPEED profile, but only zero point zero two seven for BALANCED, and that's not significant. The authors read this as pipeline machinery and generator capability being partly interchangeable.

My read is that the real value here is putting numbers on something practitioners half-know already. Tricks that work alone get discounted when stacked, and how big the discount is depends on which metric you care about and how big your corpus is. If you build RAG systems, test a new component inside your own full pipeline, at your own scale, before you ship it.

### Limitations

The authors are upfront about the limits. Everything runs on one constructed English test bed, with at most ninety-three retrieval queries and two hundred generation queries per experiment, in five focused domains. No multilingual corpora, no real enterprise data, and no open-weight generators.

Faithfulness scores depend on an LLM judge. When they brought in Claude Opus 4.7 as a third judge, its correlation with the main judge was only zero point four four. That's why the authors lean on rankings and relative comparisons, not absolute scores.

One more thing: their first pre-declared hypothesis test actually wasn't met. It required pairwise interactions to explain at least twenty percent of the variance in faithfulness, and they don't. A focused follow-up confirmed the interaction is real, but far below that bar. And the hypotheses were hashed and stored internally, not filed with a third-party pre-registration service.

### Future work

The authors' call is that new RAG techniques should be tested inside representative complete pipelines, on both metrics, at the scale where they'll be used. They release LASAGNA to make compositional evaluation the standard way to report new methods. Note, though, that the design matrix and per-query raw scores aren't in the code release; you have to request them from the authors.

My read is that there's a real weakness too. The headline retrieval shortfall rests on ninety-three queries and the authors' own baseline. With different domains and baselines the discount could look quite different, and the spread across seventeen combinations, from seven to seventy-two percent, already shows how uncertain it is.

My read is that the thing to watch is whether anyone uses this framework to rerun these experiments on larger, real enterprise corpora and with open-weight models.

That's today's Daily Frontier Papers. The paper is linked in the description.

