# Look Only When Needed: Certificate-Guided Visual Routing for Efficient Diffusion VLMs

**多数步骤模型不用看图**

- 论文 Paper: [arXiv 2610.36671](https://arxiv.org/abs/2610.36671)
- 单位 Affiliation: KAIST、AITRICS
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-31
- 视频 Videos: [YouTube 中文](https://youtu.be/DHTL3Wn2-8o) · [YouTube English](https://youtu.be/qO8hVk2Mj6E) · [B站](https://www.bilibili.com/video/BV1JcxC6QE5G)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇韩国科学技术院 KAIST 和 AITRICS 的论文，名字叫 LOWN，全称 Look Only When Needed，意思是，只在需要的时候才看图。

它要解决的，是扩散式多模态模型推理太费算力的问题。核心发现很简单，模型生成回答的大部分步骤，其实不用看图片。

### 问题

先说背景。扩散语言模型和常见的 GPT 那种模型不一样，它不是一个字一个字往后写，而是先把整段回答都盖住，再分很多步，每一步揭开几个位置，这些步骤叫去噪步。

把图片也接进来，就是扩散视觉语言模型，论文简称 DVLM，比如 LLaDA-V。它用的是双向注意力，没法直接用自回归模型那种精确的 KV cache，所以标准做法是，每一步都把图片加文字的整段输入，从头算一遍。

问题在于，图片 token 占了大头。在论文测的基准上，视觉 token 占 LLaDA-V 输入的 89% 到 97%，而去掉图片，只算文字的一次前向，计算量只有原来的 3% 到 10%。也就是说，解码的大部分算力，花在一遍遍处理同一张没变过的图片上。

### 以前的方法

以前加速主要有三条路。第一条是视觉 token 剪枝，删掉一部分图片 token。第二条是缓存中间状态，复用之前的计算。第三条是每一步多揭开几个 token，少走几步。

剪枝的问题是，删掉的信息就找不回来了，剩下的图片 token，还是每一步都要算。作者指出，这些方法要么让带图的计算更便宜，要么复用计算，要么减少步数，但没有一个问过，当前这一步到底需不需要看图。

论文第一张图把几种做法放在一起。剪枝减少的是每次看多少图片 token，LOWN 决定的是，这一步看不看图。

### 核心想法

作者先做了一个测量。在 LLaDA-V 正常解码的每一步，同时跑两次前向，一次带图，叫 FULL，一次把图片 token 拿掉，叫 TEXT-ONLY，再比较两边预测的 token 是不是一样。

结果是，在模型要揭开的那些位置上，十个基准平均有 83.4%，两边给出的第一候选完全一样。还有 85.0% 的最终 token，在被揭开之前，就已经是那个位置的第一候选了。

但是需要图的程度，各个任务差别很大。图片至少改变了一个揭开决定的步骤，平均占 22.8%，最少的 MMBench 只有 7.9%，最多的 DocVQA 有 43.8%。而且十个基准里有八个，需要图的时刻集中在解码前期。

所以作者的结论是，固定的节奏，比如每隔几步看一次图，很难抓住真正需要图的时刻，应该针对每个输入、每一步，自适应地决定。

### 方法

具体怎么做。LOWN 每跑一次带图的 FULL 前向，就把这一次在所有未揭开位置上的预测和置信度存下来，叫做锚点，anchor。

之后每一步，先跑一次便宜的 TEXT-ONLY 前向。把位置按锚点置信度从高到低排好，逐个检查，纯文字的预测和锚点一致，这个 token 就算通过认证，作者叫它 certificate，碰到第一个不一致就停下。

这个一致，同时近似说明两件事，一是这个 token 不需要图，二是回答已经变了之后，锚点仍然有效。整个判断不需要设任何置信度阈值。

论文这张图举了个例子。看过图的锚点预测是 There is a brown bear，一头棕熊。不看图的纯文字前向，前两个词和锚点一致，直接揭开。到了颜色这个词，不看图的模型猜的是 polar，北极熊，和锚点对不上，认证就断在这里。认证不够的那一步，就重新带图跑一次 FULL 前向，换上新锚点，并且先揭开 brown 这个两边有分歧的词。

再说揭开多少个。原模型每一步固定揭开 k 个 token。如果连续通过认证的位置至少有 k 个，这一步就不看图，把这一串全部揭开，可以比 k 多。如果少于 k 个，就跑 FULL 前向，换新锚点，揭开 k 个，而且优先揭开两次前向意见不一致的位置，也就是最依赖图片的那些 token。

因为两种情况每一步都至少揭开 k 个，LOWN 的步数不会超过原模型。整个方法不需要训练，也不改模型参数和注意力结构。

还有一个小细节，叫重复保护。同一步里一起揭开的位置，是各自独立预测的，相邻位置可能给出同一个词，所以碰到和相邻词重复的，认证就停下，避免回答里出现重复的词。

### 实验和效果

实验设置。主模型是 LLaDA-V，另外测了 LaViDa，和视频模型 VidLaDA-8B。十个基准覆盖选择题、文档和图表理解、数学推理和开放式图像描述，包括 MathVista、ScienceQA、ChartQA、DocVQA、MMBench 等。对比方法有两种剪枝，RedVTP 和 ZOO-Prune，还有一种多揭 token 的调度方法，KLASS。

比较方式值得说一下。对手的超参数是离散的网格，作者给每个对手选的，是计算量不低于 LOWN 的最便宜设置，选的时候不看分数。

结果在表 1。LOWN 在十个基准上，平均保留了原模型 99.6% 的分数，解码计算量只有原来的 0.353 倍。同样的算力预算下，十个基准里有八个，它是几种加速方法中的最高分。对比之下，RedVTP 平均保留 93.0%，ZOO-Prune 是 94.2%，KLASS 是 91.9%。

也要看细节。这个 99.6% 里，有四个基准比原模型还高，作者也说明了，如果每个基准最多按百分之百算，平均是 98.6%。单看 MathVista，原模型 59.1 分，LOWN 是 56.0 分，掉了 3.1 分。

实际速度上，论文报告，十个基准平均生成时间减少 57.4%，最多减少 73.2%，是在 H200 GPU 上、batch size 为 1 测的。

LOWN 还能和剪枝叠加。加上保留一半图片 token 的 RedVTP，计算量降到 0.223 倍，平均分保留 98.3%，在同等算力的对比里，十个基准全部最高。

换到 LaViDa 上，规则不用改，保留 99.7% 的分数，计算量是 0.174 倍。视频模型 VidLaDA 上，计算量降到原来的四分之一左右，EgoSchema 分数从 60.6 变成 60.2，MLVU 从 41.8 变成 43.3。

### 和以前方法的对比

一个关键的对照在表 4，检验是不是随便少看几次图都行。随机看图，看图次数和 LOWN 一样，三个任务上，LOWN 的质量都比它高 4.9% 到 13.0%。每四步看一次图，在 MathVista 上计算量和 LOWN 差不多，分数低 4 分。

消融在表 5。只做看图路由，也就是只决定哪一步看图，每步还是揭开 k 个，计算量就减少 27% 到 68%。再加上自适应揭开，又减掉剩下的 21% 到 43%。所以大部分节省，来自跳过看图。

和剪枝比，LOWN 的好处是不丢信息。这张图是个例子，问球场上能看到的最小码线数字是多少，原模型和 LOWN 都答 30，是对的，RedVTP 把关键的图像证据剪掉了，答成 0，话说得很流畅，但答错了。

和 KV cache 方法 Fast-dLLM 组合时，生成时间又减少了 38.1% 到 50.1%。但要注意，组合之后的分数，在三个任务上都比单用 Fast-dLLM 低，LLaVA-Bench 上低了 5 分。

我的判断是，这篇的价值在于找到了一个新的加速维度。以前大家想的是，每次少算一点图，它问的是，这一步要不要算图，而且能和剪枝、缓存叠在一起用。

我的判断是，弱点也很明显。主实验里回答都很短，配置的长度在 16 到 128 个 token 之间，论文的推广结论，也只说到基于 LLaDA 的扩散多模态模型，长回答和其他架构上效果如何，论文没有测。

### 局限

作者自己列了三条局限。

第一，复用的锚点预测，可能和重新带图算一次的结果不一样，锚点越旧越容易出错。论文测到，在 LOWN 自己的运行里，纯文字步骤揭开的 token，有 2.2% 到 4.7% 会被新的带图前向推翻。

第二，图表和文档阅读这类任务，一直离不开图，省得就少。论文举的 DocVQA 例子里，15 步有 14 步都跑了带图前向，计算量还是原来的 0.90 倍。

第三，看图次数是解码过程中才决定的，所以一次回答要花多少算力，事先没法固定。作者说，配合剪枝，可以部分控制成本。

### 未来方向

作者在结论里说，这些结果说明，看图调度可以成为扩散多模态模型高效解码的一个通用方向。他们还算了一个理想调度，只在真正需要图的步骤看图，成本是原来的 0.33 倍，LOWN 实测是 0.35 倍，不过作者也说明，这个理想调度并不是成本的下限。

我的判断是，扩散语言模型的卖点是并行生成，但在多模态场景下，每一步都重算图片，一直是它的软肋。LOWN 这种不用训练、即插即用的办法，如果在长回答上也成立，会让扩散多模态模型在部署成本上更有竞争力。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from KAIST and AITRICS, and it's called LOWN, short for Look Only When Needed.

The target is the compute bill of diffusion vision-language models. And the core finding is simple: for most of the steps that produce an answer, the model doesn't need to look at the image at all.

### Problem

Some background first. A diffusion language model doesn't write left to right like GPT-style models. It starts with the whole answer masked out, then over many denoising steps it reveals a few positions at a time.

Plug an image encoder into that and you get a diffusion vision-language model, which the paper calls a DVLM. LLaDA-V is one. Because it uses bidirectional attention, it can't use the exact key-value cache that autoregressive models rely on, so the standard approach recomputes the entire input, image and text, at every single step.

And the image dominates that input. On the paper's benchmarks, visual tokens make up eighty-nine to ninety-seven percent of what LLaDA-V reads, while a forward pass without them costs only three to ten percent as much. So most of the decoding compute goes into reprocessing the same, unchanged picture.

### Prior work

There have been three main ways to speed this up. Prune some of the visual tokens. Cache intermediate states and reuse them. Or unmask more tokens per step, so you run fewer steps.

Pruning loses whatever it removes, and the remaining image tokens still get processed every step. The authors' point is that all of these make image passes cheaper, reuse them, or run fewer of them, but none asks whether the current step needs the image in the first place.

The paper's first figure lines this up. Pruning changes how many visual tokens each pass reads. LOWN decides whether a pass reads them at all.

### Key idea

It starts with a measurement. At every step of LLaDA-V's normal decoding, the authors run two forward passes: a FULL one with the image, and a TEXT-ONLY one with the visual tokens removed. Then they compare the predictions.

Averaged over ten benchmarks, at eighty-three point four percent of the positions being unmasked, both passes pick the same top token. And eighty-five percent of final tokens were already the top prediction at their position before the step that unmasked them.

But how much the image matters varies a lot by task. The share of steps where the image changes at least one unmasking decision averages twenty-two point eight percent, from seven point nine on MMBench up to forty-three point eight on DocVQA. On eight of the ten benchmarks, the image-dependent moments cluster early in decoding.

So a fixed rhythm, like looking at the image every few steps, is a poor fit. The decision should adapt to each input and each step.

### Method

Here's how LOWN does it. Every time it runs a FULL pass, it stores the predictions and confidences at all still-masked positions. That's the anchor.

At each following step, it runs a cheap TEXT-ONLY pass. It walks the masked positions in order of anchor confidence, and wherever the text-only prediction matches the anchor, that token is certified. The first mismatch ends the walk.

A match approximates two things at once: that this token doesn't need the image, and that the anchor is still valid even though the answer has moved on. And it needs no confidence threshold.

The figure shows an example. The anchor, which saw the image, says "There is a brown bear." The text-only pass agrees on the first two words, so they're unmasked. But for the color, the blind pass guesses "polar," and the walk stops there. When certification runs short, LOWN runs a FULL pass with the image, refreshes the anchor, and unmasks "brown," the word where the two passes disagreed, first.

Now, how many tokens to unmask. The original model unmasks a fixed k tokens per step. If the certified run is at least k long, LOWN skips the image and unmasks the whole run, possibly more than k. If it's shorter, LOWN runs a FULL pass, takes a new anchor, and unmasks k tokens, starting with positions where the two passes disagree, the ones that depend most on the image.

Since both branches unmask at least k tokens, LOWN never takes more steps than the original model. There's no training, and no change to the weights or the attention pattern.

One small detail, a repetition guard. Positions accepted together in one text-only step are predicted independently, so neighbors can come out as the same word. The walk stops at such a repeat, so the answer doesn't stutter.

### Results

The setup. The main model is LLaDA-V, with LaViDa and the video model VidLaDA-8B as extra tests. Ten benchmarks cover multiple choice, document and chart reading, math reasoning and open-ended captioning, including MathVista, ScienceQA, ChartQA, DocVQA and MMBench. The baselines are two pruning methods, RedVTP and ZOO-Prune, and a faster unmasking schedule, KLASS.

The comparison rule matters. Competitors have discrete settings, so each gets the cheapest setting that spends at least as much compute as LOWN, chosen without looking at scores.

The results are in Table 1. Across ten benchmarks, LOWN keeps ninety-nine point six percent of the original model's score at zero point three five three times the decoding compute. At that budget it's the best of the accelerated methods on eight of ten benchmarks. RedVTP keeps ninety-three percent, ZOO-Prune ninety-four point two, and KLASS ninety-one point nine.

Look at the details, though. That average includes four benchmarks where LOWN beats the original model, and the authors note that capping each benchmark at one hundred percent gives ninety-eight point six. On MathVista specifically, the original scores fifty-nine point one and LOWN fifty-six, a drop of three point one points.

In wall-clock time, the paper reports generation time down by fifty-seven point four percent on average and up to seventy-three point two percent, measured on H200 GPUs at batch size one.

LOWN also stacks with pruning. Combined with RedVTP keeping half the visual tokens, compute drops to zero point two two three times, the average score holds at ninety-eight point three percent, and at that budget it's best on all ten benchmarks.

On LaViDa, the rule carries over unchanged: ninety-nine point seven percent of the score at zero point one seven four times the compute. On the video model VidLaDA, compute falls to roughly a quarter, with EgoSchema going from sixty point six to sixty point two, and MLVU from forty-one point eight to forty-three point three.

### Comparison

A key control is Table 4: does skipping the image at random work just as well? Against a random schedule with the same number of image passes, LOWN's normalized quality is four point nine to thirteen percent higher on all three tasks. Looking every fourth step costs about the same as LOWN on MathVista but scores four points lower.

The ablation is Table 5. Visual routing alone, deciding which steps see the image while still unmasking k tokens per step, cuts compute by twenty-seven to sixty-eight percent. Adaptive unmasking then removes another twenty-one to forty-three percent of what's left. So most of the saving comes from skipping the image.

Against pruning, LOWN's edge is that nothing is thrown away. In this example the question is the lowest yard-line number visible on a football field. The original model and LOWN both answer thirty, correctly. RedVTP has pruned the evidence and answers zero, fluently and wrongly.

Combined with the key-value cache method Fast-dLLM, LOWN cuts generation time by a further thirty-eight point one to fifty point one percent. But the combined scores are lower than Fast-dLLM alone on all three tasks, by five points on LLaVA-Bench.

My read is that the contribution here is a new axis for speed. Earlier work asked how to process less of the image each time. This asks whether this step needs the image at all, and it stacks with pruning and caching.

My read is that the weakness is just as clear. The main experiments use short answers, configured at sixteen to one hundred twenty-eight tokens, and the paper's own generalization claim covers LLaDA-based models. How it holds up on long answers or other architectures isn't tested.

### Limitations

The authors list three limitations.

First, a reused anchor prediction can differ from what a fresh image pass would say, and more so as the anchor ages. In LOWN's own runs, two point two to four point seven percent of the tokens unmasked on text-only steps would be overturned by a fresh FULL pass.

Second, chart and document reading keep needing the image, so the savings shrink. In the paper's DocVQA example, fourteen of fifteen steps ran a FULL pass, and the cost was still zero point nine times the original.

Third, because the number of image passes is decided during decoding, the cost of an answer can't be fixed in advance. The authors say pairing it with pruning gives some control.

### Future work

In their conclusion, the authors argue that scheduling image access is a broadly applicable axis for efficient DVLM decoding. They also computed an oracle that looks at the image only on the steps where it changes a decision, and it costs zero point three three times the original, against LOWN's zero point three five, though they note that oracle is not a lower bound.

My read is that parallel generation is the selling point of diffusion language models, but recomputing the image every step has been their weak spot in multimodal use. A training-free, plug-in fix like this, if it holds on longer answers, makes diffusion VLMs more competitive on serving cost.

That's today's Daily Frontier Papers. The paper is linked in the description.

