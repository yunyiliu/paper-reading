# Letting Models Manage Their Own Context: Where We Stand

**上下文快满了该让谁来删**

- 论文 Paper: [arXiv 2611.34147](https://arxiv.org/abs/2611.34147)
- 单位 Affiliation: NVIDIA（英伟达）
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-28
- 视频 Videos: [B站](https://www.bilibili.com/video/BV19A1MzGEob)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇英伟达的技术报告，题目叫，让模型自己管理自己的上下文，我们现在走到了哪一步。

它问的问题很实际。Agent 干活干到一半，上下文窗口快满了，该删什么、留什么，这个决定应该交给外面的框架，还是交给模型自己。

### 问题

先说背景。Agent 每调用一次工具，工具的输出、模型的回复，都会堆进历史里。历史越长，模型越难找到有用的信息，这叫 context rot，也就是上下文腐烂，再往后，就直接超出上下文窗口了。

现在大部分 Agent 框架，英文叫 harness，就是包在模型外面、负责调工具和管理历史的那层程序，处理办法是自动压缩，英文叫 compaction。用量一过某个阈值，框架就把前面的历史写成一段摘要。

### 以前的方法

作者把这种做法叫框架驱动的上下文管理。什么时候压缩，压缩哪一段，由框架的固定规则决定，模型只负责写摘要。作者认为，这种办法简单，但会丢东西，任务需要的关键信息，可能就在被压掉的那段里。

另一条路已经有人在走。论文提到了 SideQuest、Context Language Models 这些研究，OpenAI 的 Codex 也加了一个实验性的上下文管理模式。

这张图把两种方式放在一起。上面是框架驱动，用量过了 80%，框架把最早的两步压成摘要。下面是模型驱动，模型自己挑出第 3 到第 9 步，写一段摘要换掉它们，原文放进存档。

### 核心想法

这篇论文的想法很直接，把压缩的决定权交给模型。什么时候压缩，压缩哪一段，摘要写什么，都由模型自己决定。

具体是在 NOOA 里做的，这是英伟达开源的 Agent 框架。模型可以查看自己的历史，用一个叫 collapse 的调用，把一段事件换成一条摘要。被压缩的原文不会删除，而是存档，之后模型可以按编号把原文取回来。

作者想回答三个问题。现在的模型，能不能管好自己的上下文。这样做，能不能提高任务表现。推理引擎，能不能把这件事变成更便宜的推理。

### 方法

实验用了两个 benchmark。OfficeQA Pro v2 有 90 道题，问的是美国联邦政府的收入和支出，Agent 要在一大堆文档里搜索和阅读，上下文涨得很快。MLE-bench Lite 是 22 个 Kaggle 竞赛，Agent 要看数据、写代码、调试，最后提交预测。

对比两种设置。框架驱动，用量到预算的 80%，框架就把早先的历史写成摘要，保留最初的指令和最近 10 个事件。模型驱动，模型拿到上下文 API，能实时看到用量，提示词告诉它，用量到 85% 再考虑压缩，快到上限时，还会收到一条警告。

怎么制造上下文压力呢。作者把上下文预算从 8K 一路调到 512K token。预算越小，任务需要的内容和窗口装得下的内容之比就越大，相当于模拟更长的任务。请求超过预算，会先删掉最老的事件再试，失败三次就算答错。

一共测了四个模型，GPT-6.1 Sol、Kimi K3、DeepSeek V4 Pro，和 Claude Opus 5.5。

### 实验和效果

先看主结果。这张图是 OfficeQA Pro v2 上，分数随预算的变化，虚线是框架驱动，实线是模型驱动。

最明显的是 GPT-6.1 Sol。论文报告，在 32K 预算下，框架驱动只有 21%，换成模型自己管理，到了 56%，接近三倍，而且没有做任何额外训练。

但不是每个模型都这样。Kimi K3 在小预算下也领先框架驱动。DeepSeek V4 Pro 的提升要小一些。Claude Opus 5.5 很少调用这个 API，在小预算下反而落后于框架驱动，要等预算足够大，两种方式才差不多。

总的规律是，预算越紧，收益越大，预算放宽，差距就缩小。在 MLE-bench Lite 上，效果小得多，只在最紧的几档预算下看得到，作者推测，是因为这些任务本来就用不了多少上下文。

那模型之间为什么差这么多。作者去看了每个模型怎么用这个 API。同样的 API，同样的说明，GPT、Kimi 和 DeepSeek 在 42% 到 53% 的任务里调用过它，Opus 只在 4% 的任务里用过。

每次压缩的力度也不一样。GPT 压缩一次，请求长度的中位数缩小 3.7 倍，Kimi 是 2.1 倍，DeepSeek 1.8 倍，Opus 1.5 倍。压得狠也有代价，GPT 有 19% 的压缩，事后又回去把存档取了出来，Kimi 是 7%，DeepSeek 是 3%，Opus 是 14%，不过它压缩次数太少，这个数不太可靠。

上下文快满的时候，反应也不同。在 64K 预算下，收到第一次警告以后，Opus 多数时候直接给出答案收尾，图上是 88%。其他模型大多选择压缩，GPT 在 69% 的情况下先去读 API 文档再压缩，DeepSeek 在 46% 的情况下先查看历史，Kimi 在 49% 的情况下直接压缩。

为什么 GPT 用得最好，作者猜测，可能和训练有关，因为 Codex 已经有类似的实验模式。但作者明确说，这只是推测，他们并不知道这些模型是怎么训练的。

还有一个现象，作者叫它上下文焦虑，context anxiety，就是空间还够，模型却过早、过狠地压缩。去掉“到 85% 再考虑压缩”这句提示以后，Kimi 在 128K 预算下第一次压缩时的用量中位数，从大约 84% 掉到 29%，DeepSeek 在 256K 下，从 83% 掉到 40%。Opus 原本几乎不压缩，去掉提示以后，反而经常很早压缩，128K 下中位数只有 16%。

Kimi 的这个习惯，也体现在 token 用量上。在 32K 预算下，两种方式的最大上下文一样，模型驱动累计处理的输入 token 少了 41%，3600 万对 6200 万，分数还略高一点，43% 对 41%。

但少留上下文也有代价。在 128K 及以上的预算，去掉时机提示的 Kimi 和 DeepSeek，比框架驱动低大约 8 个点。作者提到，Kimi K3 训练时，对过长的输出有强化学习惩罚，这可能让它在空间还够的时候，也习惯省 token。

### 和以前方法的对比

最后一部分是推理成本。推理引擎为了不每轮都重算整个提示词，会缓存前缀的计算结果，也就是 KV cache。可一旦在中间改了上下文，比如把 B 段换成摘要，改动之后的所有内容都要重算，哪怕 C 段一个字没变。所以上下文变短，不一定就变便宜。

作者在 SGLang 推理引擎里做了一个原型，编辑之后，把没变的后半段的 KV cache 也接着复用，模型用的是 Qwen3.6-27B，上下文 32K。在模型驱动的设置下，和标准的前缀缓存相比，每个请求的 prefill 计算量，减少了 4% 到 15%，任务准确率没有可测的变化。

省得最多的是 OfficeQA Pro，模型驱动下减少了 15.1%。原因是编辑后保留下来的后半段很长，平均大约 1 万 token，差不多是提示词的 60%。

和已有工作比，这篇没有提出新的训练方法。MEMENTO、AgentFold 这些工作，是训练模型学会压缩，Context Language Models 已经提出了后缀缓存复用。这篇的定位是，在同一个上下文 API 下，对现有模型做一次对照实测。

我的判断是，这篇最值得记住的，不是 GPT 的三倍，而是同一个接口，在不同模型手里结果差别很大。管理上下文更像是一项需要训练的能力，不是给个工具就能用好。

### 局限

作者自己列了四条局限。第一，benchmark 很小，OfficeQA Pro v2 只有 90 题，MLE-bench Lite 只有 22 个竞赛，策略或模型之间的小差别，可能并不显著。

第二，上下文压力是模拟的。缩小预算只是长任务的近似，到上限时删掉最老事件的兜底机制，本身也可能影响分数。

第三，只测了四个模型，也不知道它们是怎么训练的，关于 GPT 为什么更好的解释，只是推测，没有验证。

第四，KV cache 复用只在一个开源模型、一个推理引擎上测过。它是对完整重算的近似，而且测的是 prefill 计算量，不是端到端的延迟或成本。

### 未来方向

作者预计，模型自己管理上下文，会成为前沿模型的标准能力。要做到，需要两件事。一是训练模型学会什么时候压缩、压缩什么。二是把框架和推理引擎打通，编辑上下文时保住能复用的计算，释放不再需要的 KV cache。

我的判断是，如果你在做 Agent，可以试试把压缩交给模型，尤其是上下文很紧的长任务。但一定要按模型分别测，还要盯着提示词，光是去掉一句时机提示，有的模型行为就明显变了。

我的判断是，也要看到，这是英伟达在自家框架上做的一份技术报告，题量小，场景只有两类，最亮眼的接近三倍，是一个 benchmark 在一档预算下的结果，结论还需要更多任务来验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is a technical report from NVIDIA, titled Letting Models Manage Their Own Context: Where We Stand.

The question is a practical one. When an agent's context window fills up halfway through a task, who should decide what to drop and what to keep? The harness around the model, or the model itself?

### Problem

First, the background. Every tool call an agent makes adds tool output and model turns to its history. The longer that history gets, the harder it is for the model to find what matters, which people call context rot. Eventually it simply overflows the context window.

Most agent harnesses, meaning the program wrapped around the model that runs tools and manages history, handle this automatically with what's called compaction. Once usage crosses a threshold, the harness summarizes the earlier history.

### Prior work

The authors call this harness-driven context management. A fixed rule in the harness decides when to compact and which part of the history to cover, and the model only writes the summary. It's simple, but the authors point out it can be lossy, and the one detail the task needed might be in the part that got squashed.

The alternative is already being explored. The paper points to research like SideQuest and Context Language Models, and to an experimental context-management mode that OpenAI's Codex has introduced.

This figure puts the two side by side. On top, harness-driven: usage crosses eighty percent, and the harness folds the first two events into a summary. Below, model-driven: the model itself picks events three through nine, replaces them with a summary, and the originals go to an archive.

### Key idea

The idea is straightforward. Hand the decision to the model. When to compact, what to compact, and what the summary says, all of it is the model's call.

They do this in NOOA, NVIDIA's open source agent harness. The model can inspect its own history and use a call named collapse to replace a range of events with a summary. Nothing is deleted. The originals are archived, and the model can pull any of them back later by its tag.

The authors want answers to three questions. Can current models manage their own context well? Does it improve task performance? And can inference engines turn it into cheaper inference?

### Method

There are two benchmarks. OfficeQA Pro v2 has ninety questions about U.S. federal receipts and expenditures, answered by searching and reading a large document corpus, so context fills up fast. MLE-bench Lite is twenty-two Kaggle competitions, where the agent inspects data, writes and debugs code, and submits predictions.

Two settings are compared. In the harness-driven one, at eighty percent of the budget the harness summarizes earlier history, keeping the initial instructions and the ten most recent events. In the model-driven one, the model gets the context API and a live usage indicator, it's told to consider compacting only once usage reaches eighty-five percent, and it gets a warning as the context nears its limit.

To create pressure, they sweep the context budget from eight thousand all the way to five hundred and twelve thousand tokens. A smaller budget raises the ratio of what the task needs to what the window holds, which stands in for a much longer task. If a request goes over budget, the oldest events are dropped and it retries. Three failures, and the run counts as a wrong answer.

Four models are tested: GPT-6.1 Sol, Kimi K3, DeepSeek V4 Pro, and Claude Opus 5.5.

### Results

Here's the main result. This figure shows score against context budget on OfficeQA Pro v2. The dashed line is harness-driven, the solid line is model-driven.

GPT-6.1 Sol stands out. The paper reports that at a thirty-two thousand token budget it goes from twenty-one percent with harness-driven compaction to fifty-six percent managing its own context. That's nearly three times the score, with no extra training.

But it's not the same for every model. Kimi K3 also beats the harness at small budgets. DeepSeek V4 Pro improves less. Claude Opus 5.5 rarely uses the API, and at small budgets it actually trails the harness, until the budget gets large enough that the two strategies converge.

The pattern is that gains are biggest when the budget is tight, and they shrink as the budget grows. On MLE-bench Lite the effect is much smaller and only shows up at the tightest budgets, probably, the authors say, because those tasks need far less context.

So why do models differ this much? The authors looked at how each one used the API. Same API, same instructions. GPT, Kimi and DeepSeek call it in forty-two to fifty-three percent of tasks. Opus uses it in only four percent.

How hard they compact differs too. When GPT compacts, the request shrinks by a median of three point seven times. For Kimi it's two point one, DeepSeek one point eight, Opus one point five. Compacting harder has a cost: GPT goes back to retrieve archived content after nineteen percent of its compaction calls, against seven percent for Kimi and three percent for DeepSeek. Opus is at fourteen percent, but it compacts so rarely that this number is noisy.

They also react differently when the context is nearly full. At a sixty-four thousand token budget, after the first warning, Opus mostly wraps up and returns an answer, eighty-eight percent of the time in this chart. The others mostly compact. GPT reads the API docs first in sixty-nine percent of cases, DeepSeek inspects its history first in forty-six percent, and Kimi compacts straight away in forty-nine percent.

As for why GPT does best, the authors conjecture that training may play a role, since Codex already has a similar experimental mode. But they're explicit that this is a guess. They don't know how any of these models were trained.

There's another effect they call context anxiety: compacting earlier and harder than needed while space remains. Remove the instruction to wait until eighty-five percent, and Kimi's median first compaction at a one hundred twenty-eight thousand budget drops from about eighty-four percent of the budget to twenty-nine. DeepSeek's, at two hundred fifty-six thousand, drops from eighty-three to forty. Opus, which almost never compacts with the instruction, starts compacting regularly and early without it, at a median of sixteen percent at one hundred twenty-eight thousand.

For Kimi, this shows up in token usage. At a thirty-two thousand budget, both strategies have the same maximum context, yet the model-driven agent processes forty-one percent fewer cumulative input tokens, thirty-six million versus sixty-two million, and scores slightly higher, forty-three percent versus forty-one.

Keeping less context can cost you, though. At budgets of one hundred twenty-eight thousand and above, Kimi and DeepSeek without the timing instruction score about eight points below the harness baseline. The authors note that Kimi K3 was trained with reinforcement learning penalties for overly long rollouts, which may teach it to save tokens even when there's room.

### Comparison

The last part is about serving cost. Inference engines avoid reprocessing the whole prompt every turn by caching the computation for the unchanged prefix, the KV cache. But edit the middle of the context, say replace section B with a summary, and everything after the edit has to be recomputed, even if section C didn't change at all. So a shorter context isn't automatically a cheaper one.

The authors built a prototype in the SGLang serving engine that also reuses the KV cache of the unchanged suffix after an edit, using Qwen3.6-27B at a thirty-two thousand token context. Under model-driven management, compared with standard prefix caching, it cuts prefill compute by four to fifteen percent per request, with no measurable change in task accuracy.

The biggest saving is on OfficeQA Pro, fifteen point one percent under model-driven management. That's because the text that survives an edit is long, about ten thousand tokens, or roughly sixty percent of the prompt on average.

Compared with earlier work, this paper doesn't propose a new training method. Work like MEMENTO and AgentFold trains models to condense their context, and Context Language Models already introduced suffix cache reuse. This paper's role is a controlled measurement of today's models under one shared context API.

My read is that the headline result isn't GPT's near tripling. It's that the same interface gives very different outcomes depending on the model. Managing context looks like a skill that has to be trained, not something a model picks up just because you hand it a tool.

### Limitations

The authors list four limitations. First, the benchmarks are small. OfficeQA Pro v2 has ninety questions and MLE-bench Lite twenty-two competitions, so small differences between strategies or models may not be significant.

Second, the context pressure is simulated. Shrinking the budget is only a proxy for long tasks, and the fallback that drops the oldest events at the limit can itself affect scores.

Third, only four models were tested, with unknown training. The explanation for GPT's stronger results is a conjecture, not a tested claim.

Fourth, KV reuse was tested with one open model in one serving engine. It approximates full recomputation, and what they measured was prefill compute, not end-to-end latency or cost.

### Future work

The authors expect model-driven context management to become a standard capability of frontier models. That takes two things. Training that teaches models when and what to compact. And connecting the harness to the inference engine, so context edits keep reusable computation and release KV cache entries that are no longer needed.

My read is that if you build agents, it's worth trying to let the model handle compaction, especially on long tasks with tight context. But test each model separately, and watch your prompt. Removing a single timing instruction substantially changed how some models behaved.

My read is also that this is NVIDIA's technical report on its own harness, with small benchmarks and two kinds of tasks. The striking near tripling comes from one benchmark at one budget, and the conclusions need more tasks behind them.

That's today's Daily Frontier Papers. The paper is linked in the description.

