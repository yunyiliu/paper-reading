# Tools Need Not Speak in Text: Latent Context Compression for Agents

**工具返回的结果不用再写成文字**

- 论文 Paper: [arXiv 2611.06500](https://arxiv.org/abs/2611.06500)
- 单位 Affiliation: 清华大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-08
- 视频 Videos: [YouTube 中文](https://youtu.be/qAqaayYap7w) · [YouTube English](https://youtu.be/0IZuhHmSV5A) · [B站](https://www.bilibili.com/video/BV1LDWJ6VEiZ)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇清华大学的论文，标题是，工具不必用文字说话，副标题是，面向 Agent 的潜在上下文压缩。方法的名字有点意思，叫 Coca-Cola。

它要解决的，是做 Agent 的人几乎都会碰到的问题，工具返回的东西太长，把上下文塞满了。

### 问题

先说背景。现在主流的 Agent 用的是 ReAct 这种模式，先想一步，调用一个工具，再把工具返回的结果原样贴进上下文，接着想下一步。比如搜索 Agent，每搜一次，就把一大段搜索结果加进来。

论文统计了一下，到第 4 步的时候，工具返回的内容已经占了整个上下文的 96.7%。

这带来两个问题。第一是贵，token 越多，算得越慢，花得越多。第二是会分散模型的注意力，影响推理质量。

论文举了一个例子。问题是，2016 年马拉喀什 ePrix 这场比赛的冠军，是哪一年出生的。搜索结果里有日期、赛道长度、杆位，一大堆细节，但对下一步有用的，只有冠军是 Sébastien Buemi 这一句。知道了名字，下一步去搜他的出生日期就行。

### 以前的方法

最自然的办法是压缩上下文。以前主要有两条路，一条是 token 剪枝，用一个模型判断哪些 token 信息量低，直接删掉，代表是 LLMLingua 这一系列。另一条是文字摘要，用另一个大模型，把工具输出缩写成一段短文字。

作者认为，这两条路有个根本的限制。压缩完的结果还是一串文字，文字得保持通顺、意思连贯，所以能压到多短，是有上限的。

还有一类做法，是把长文本压成少量连续向量，比如 ICAE。但它们主要是为了保留或还原原文，而不是专门留下对 Agent 下一步决策有用的部分。

### 核心想法

作者问了一个问题，工具的输出，一定要用文字来表达吗。这些输出说到底，主要是给 Agent 下一步推理和行动用的。

所以他们的想法是，把工具返回的内容，压成少量 latent token，也就是嵌入空间里的高维向量，直接放进 Agent 的输入里，不再经过文字这一层。

论文还给了一个容量上的分析。一个文字 token，只能从词表里选一个符号，一个向量 token 却有很多维，每一维都能取很多个值，所以同样的位置数量，向量能装的信息上限可以更大。作者也说明，实际能用上多少，要看压缩模型和 Agent 的能力。

### 方法

具体结构看这张图，从左到右分三部分。

第一部分是上下文感知的编码器，一个小语言模型。它同时读任务指令，Agent 当前这一步的推理，和工具返回的原始内容。这样它知道 Agent 想找什么，压缩时就有侧重。

第二部分是一个适配器，叫 latent adapter。编码器和 Agent 的表示空间不一样，适配器把编码器的输出变成固定数量的 latent token，映射到 Agent 的输入嵌入空间，前后加上边界标记，插进 Agent 的输入。

第三部分是训练方法，叫 Action-Preserving Training，可以理解为保动作训练。做法是，把完整的工具输出和压缩后的 latent token，分别喂给同一个冻结的 Agent，比较它对下一步动作的预测分布，让两边尽量一致。

损失函数有两项，一项是 KL 散度，用来对齐两边的动作分布，另一项是交叉熵，用轨迹里记录下来的真实下一步动作做监督。

因为 latent token 是连续的，梯度可以穿过冻结的 Agent，一路传回压缩模型。训练时 Agent 本身不动，只更新编码器的 LoRA 参数和适配器。

这里的关键是，它不要求把原文还原出来，只要求 Agent 拿到压缩版以后，做出的决定和看完整版时一样。

推理的时候，每调用一次工具，就把原始返回替换成 latent token，默认是 128 个，Agent 接着往下走。

### 实验和效果

实验设置。场景是网页搜索 Agent，用了两组模型搭配，一组是 Qwen3.5-35B-A3B 和 Qwen3.5-4B，另一组是 GLM-4.7-Flash 和 GLM-Z1-9B。

评测用四个问答数据集，NQ 和 TriviaQA 是单跳问答，2Wiki 和 HotpotQA 是多跳问答，需要把几条证据串起来。每次搜索最多返回 5 条结果，每个任务最多搜 5 次。

对比的方法有四个，不压缩的原始 Agent，做剪枝的 LongLLMLingua，做摘要的 LLMSummary，还有 ICAE。这里的 ICAE 和 Coca-Cola 结构一样，只是训练目标换成还原原文，正好能看出保动作训练的作用。

先看效率，在表 1 和表 2。和不压缩相比，总 token 消耗减少了 70.01% 到 89.24%。比如 Qwen 这组在 NQ 上，平均输入 token 从 14362 左右，降到 1767 左右，少了 87.7%。

输出 token 也少了，八组设置里减少了 26.9% 到 57.9%，也就是说，Agent 后面要生成的内容也跟着变少了。

端到端运行时间，在所有设置里都是各方法中最短的。Qwen 这组减少了 24.45% 到 32.80%，GLM 这组减少了 41.61% 到 55.97%，比如 GLM 在 2Wiki 上，从 106.37 秒降到 46.83 秒。

再看准确率，在表 3。压缩了这么多，八组设置里，EM 和 F1 两个指标全部高于不压缩的原始 Agent。比如 GLM 这组在 TriviaQA 上，F1 从 44.57 提到 59.41。

作者的解释是，过滤掉冗余信息，可能缓解了长上下文带来的注意力分散。这是作者的推测，论文没有专门做实验验证。

### 和以前方法的对比

和其他压缩方法比，在 16 个数据集和指标的组合里，Coca-Cola 有 13 个拿到最高分。GLM 这组的四个数据集，EM 和 F1 都是第一。

比如 GLM 在 TriviaQA 上，最强的基线是 LLMSummary，F1 是 49.29，Coca-Cola 是 59.41。

但它不是每项都赢。在 Qwen 这组，TriviaQA 上 ICAE 的 EM 和 F1 更高，2Wiki 上 LongLLMLingua 的 F1 更高。作者指出，这两个基线的压缩率明显更低，分别是 66.92% 对 83.68%，和 41.96% 对 70.01%。

压缩率方面，Coca-Cola 在八组里有五组最高，另外三组 LLMSummary 略高，但这三组里它的 EM 和 F1 都更低。

最能说明问题的，是和 ICAE 的对比。两者结构一样，只差训练目标，Coca-Cola 在八组设置里压缩率都更高，比如 GLM 在 2Wiki 上，是 89.24% 对 74.12%。

消融实验在表 4，用的是 GLM 这组的 NQ。去掉推理过程，EM 掉 2.05 分，去掉任务目标，掉 3.48 分，两样都去掉，EM 从 23.50 掉到 18.34。但就算只看工具输出，它的 EM 和压缩率，仍然超过所有基线压缩方法。

latent token 的数量也有影响。从 8 个加到 128 个，F1 从 26.34 稳步升到 31.45，压缩率在 128 个时降到 73.73%。作者默认用 128 个。

论文还给了一个案例。问题是，Jens Risom 引入了哪种以极简和实用为特点的设计。搜索结果里，既有他和中世纪现代风格的关联，也有一条关键证据，说他是最早把斯堪的纳维亚设计带到美国的设计师之一。不压缩的 Agent 分不清，多搜了一次，还是答成中世纪现代，压缩版答对了，斯堪的纳维亚设计。

我的判断是，这篇真正的贡献在训练目标上。不是压得越多越好，而是只保留影响下一步决策的那部分信息，和 ICAE 的对比，把这一点说得比较清楚。

我的判断是，也要看到绝对分数并不高。比如 Qwen 这组在 HotpotQA 上，不压缩的 Agent，EM 只有 11.00，Coca-Cola 是 25.70，相对提升 133.64%。这个百分比很大，有一部分是因为起点太低。

### 局限

论文没有单独写局限这一节，下面三条，是从它的实验设置里能直接看到的。

第一，只测了网页搜索这一种工具，任务也都是问答。作者说方法本身没有针对搜索做设计，可以直接用到别的工具上，但论文里没有这方面的实验。

第二，交互很短，每个任务最多搜 5 次。更长流程的任务效果怎么样，论文没有测。

第三，这个方法要把向量直接插进 Agent 的输入嵌入，训练时梯度还要穿过 Agent，所以必须能拿到 Agent 模型本身，而不只是调它的接口。

### 未来方向

作者在结论里说，他们挑战的是一个默认假设，工具和 Agent 之间不必用文字 token 交流，工具只需要传递足够支撑下一步决策的信息。他们认为，这为更高效、更能扩展的长流程 Agent 打开了一个方向。代码已经在 GitHub 公开。

我的判断是，这个思路最适合自己部署开源模型的团队。只通过 API 调用闭源模型的话，这种接法用不上。

我的判断是，还有一个代价是可解释性。工具结果变成向量以后，人读不出 Agent 看到了什么，出了错也更难排查。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Tsinghua University, and it has a provocative title: Tools Need Not Speak in Text. The method has a memorable name, too. It's called Coca-Cola.

The problem it goes after is one that anyone building agents has hit. Tool outputs are long, and they eat your context.

### Problem

Some background. Most agents today follow the ReAct pattern. Think a step, call a tool, paste the raw result into the context, and think again. A search agent, for example, appends a whole page of search results every time it searches.

The authors measured how fast this adds up. By step four, tool outputs make up ninety-six point seven percent of the agent's entire context.

That costs you twice. First, more tokens means slower and more expensive inference. Second, all that extra text can distract the model and hurt its reasoning.

Here's their example. The question is: what year was the winner of the 2016 Marrakesh ePrix born? The search results come back with the date, the course length, the pole position, the fastest lap, and much more. But the only thing that matters for the next step is one line: the winner was Sébastien Buemi. With that name, the next move is simply to search for his birth date.

### Prior work

The obvious fix is to compress the context, and there have been two main ways to do it. One is token pruning: a model scores which tokens carry little information and deletes them. The LLMLingua family is the standard example. The other is summarization: a second language model rewrites the tool output as a shorter piece of text.

The authors argue both hit the same wall. The compressed result is still text, and text has to stay readable and coherent. That puts a ceiling on how short it can get.

There's also a third line of work that squeezes long text into a handful of continuous vectors, such as ICAE. But those methods mostly aim to preserve, or reconstruct, the original text, not specifically the parts an agent needs for its next decision.

### Key idea

So the authors ask a blunt question. Does a tool's output have to be text at all? It mainly exists to feed the agent's next reasoning step and action.

Their answer is to compress each tool output into a few latent tokens, meaning high-dimensional vectors in the embedding space, and feed those straight into the agent's input. No text in between.

They back this with a capacity argument. A text token picks one symbol from a fixed vocabulary. A vector token has many dimensions, each of which can take many values, so for the same number of positions, vectors can in principle carry more information. The authors are careful to say this is only motivation. How much of that capacity is usable depends on the compressor and on the agent.

### Method

Here's the architecture, in three parts, left to right.

Part one is a context-aware encoder, which is a small language model. It reads three things together: the task instruction, the agent's reasoning at the current step, and the raw tool output. Because it knows what the agent is looking for, it can decide what to keep.

Part two is a latent adapter. The encoder and the agent are different models with different representation spaces. The adapter turns the encoder's output into a fixed number of latent tokens, maps them into the agent's input embedding space, wraps them in special boundary markers, and slots them into the agent's input.

Part three is the training method, which they call Action-Preserving Training. Feed the full tool output and the compressed latent tokens separately into the same frozen agent, look at what it would do next in each case, and push those two next-action distributions to match.

The loss has two terms. A KL divergence aligns the two action distributions, and a cross-entropy term supervises against the actual next action recorded in the trajectory.

Because latent tokens are continuous, gradients can flow back through the frozen agent into the compressor. The agent itself never changes. Only the encoder's LoRA parameters and the adapter are trained.

That's the key move. The compressor is never asked to reconstruct the original text. It only has to make sure the agent decides the same way it would have with the full output.

At inference time, every tool result is swapped for latent tokens, one hundred and twenty-eight by default, and the agent carries on.

### Results

The setup. It's a web-search agent, tested with two model pairings: Qwen3.5-35B-A3B with Qwen3.5-4B, and GLM-4.7-Flash with GLM-Z1-9B.

There are four question-answering benchmarks. NQ and TriviaQA are single-hop. 2Wiki and HotpotQA are multi-hop, where the agent has to chain several pieces of evidence. Each search returns at most five results, and each task allows at most five searches.

Four baselines: the uncompressed agent; LongLLMLingua for token pruning; LLMSummary, which summarizes with a language model; and ICAE. Here, ICAE uses exactly the same architecture as Coca-Cola, with the training objective swapped for text reconstruction. That isolates what action-preserving training contributes.

Efficiency first, in Tables 1 and 2. Compared with no compression, total token use drops by seventy point zero one to eighty-nine point two four percent. With the Qwen models on NQ, for example, average input tokens fall from about fourteen thousand three hundred to about seventeen hundred and seventy, an eighty-seven point seven percent cut.

Output tokens drop too, by twenty-six point nine to fifty-seven point nine percent across the eight settings. So the agent also ends up generating less afterward.

End-to-end runtime is the lowest of any method in every setting. With Qwen, it falls by twenty-four to thirty-three percent. With GLM, by forty-two to fifty-six percent. On 2Wiki with GLM, that's one hundred and six seconds down to forty-seven.

Now accuracy, in Table 3. Despite all that compression, Coca-Cola beats the uncompressed agent on both exact match and F1 in all eight settings. With GLM on TriviaQA, for instance, F1 goes from forty-four point six to fifty-nine point four.

The authors' explanation is that filtering out redundant information may ease the attention dilution that long contexts cause, so the agent focuses on the relevant evidence. That's their hypothesis. The paper doesn't run an experiment to test it directly.

### Comparison

Against the other compression methods, Coca-Cola has the top score in thirteen of the sixteen dataset and metric pairs, and it's first on both metrics on all four datasets with the GLM models.

On TriviaQA with GLM, for example, the strongest baseline is LLMSummary at forty-nine point three F1. Coca-Cola reaches fifty-nine point four.

It doesn't win everywhere. With Qwen, ICAE scores higher on TriviaQA on both metrics, and LongLLMLingua has the higher F1 on 2Wiki. The authors point out that those baselines compress much less: sixty-seven versus eighty-four percent, and forty-two versus seventy percent.

On compression rate, Coca-Cola is highest in five of the eight settings. LLMSummary edges it out in the other three, but in each of those three, LLMSummary's exact match and F1 are lower.

The most telling comparison is with ICAE. Same architecture, different objective, and Coca-Cola compresses more in all eight settings. On 2Wiki with GLM, it's eighty-nine point two percent versus seventy-four point one.

The ablations are in Table 4, on NQ with the GLM models. Drop the reasoning state and exact match falls by two point zero five points. Drop the task goal and it falls by three point four eight. Drop both, so the encoder sees only the tool output, and it goes from twenty-three point five to eighteen point three four. Even so, that stripped-down version still beats every baseline compression method on both exact match and compression rate.

The number of latent tokens matters as well. Going from eight to one hundred and twenty-eight, F1 climbs steadily from twenty-six point three to thirty-one point five, while the compression rate drops to seventy-three point seven percent at one hundred and twenty-eight. That's the default they use.

There's also a case study. The question: Jens Risom introduced what type of design, characterized by minimalism and functionality? The search results mention both his link to Mid-Century Modern design and the key fact that he was one of the first to bring Scandinavian design to America. The uncompressed agent can't separate the two, searches again, and still answers Mid-Century Modern. The compressed agent gets it right: Scandinavian design.

My read is that the real contribution here is the training objective. The point isn't compressing as hard as possible. It's keeping only what changes the next decision, and the ICAE comparison makes that case fairly cleanly.

My read is also that the absolute scores are low. With Qwen on HotpotQA, the uncompressed agent gets an exact match of just eleven, and Coca-Cola gets twenty-five point seven. That's a one hundred and thirty-four percent relative gain, and part of why it looks so big is that the starting point is so low.

### Limitations

The paper doesn't have a limitations section, so here are three that follow directly from its setup.

First, it tests one tool, web search, on question answering only. The authors say nothing in the method is search-specific and it should carry over to other tools, but the paper has no experiments showing that.

Second, the interactions are short, at most five searches per task. How it holds up over much longer tasks isn't tested.

Third, the method injects vectors directly into the agent's input embeddings, and training sends gradients through the agent. So you need the agent model itself, not just an API to call it.

### Future work

In their conclusion, the authors frame this as challenging a default assumption: tools and agents don't have to talk in text tokens, and a tool only needs to pass along what's sufficient for the next decision. They see it as opening a direction toward more efficient, more scalable long-horizon agents. The code is already on GitHub.

My read is that this fits teams that run their own open-weight models. If you only reach a closed model through an API, you can't plug this in.

My read is that there's also a cost in transparency. Once tool results become vectors, nobody can read what the agent actually saw, and failures get harder to debug. That's worth thinking through before putting this into production.

That's today's Daily Frontier Papers. The paper is linked in the description.

