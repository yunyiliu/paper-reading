# Cache Engineering for Agents

**文字不用删缓存自己管**

- 论文 Paper: [arXiv 2610.28577](https://arxiv.org/abs/2610.28577)
- 单位 Affiliation: 清华大学 · 香港中文大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-23
- 视频 Videos: [YouTube 中文](https://youtu.be/D0reMHcEYUY) · [YouTube English](https://youtu.be/cD9A1eIAzvM) · [B站](https://www.bilibili.com/video/BV1Uent6LE4P)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇清华大学的论文，合作单位还有香港中文大学，题目叫 Cache Engineering for Agents，可以翻译成，给 Agent 做缓存工程。

一句话说，以前 Agent 上下文太长，大家是删文字、做总结，这篇论文换了个思路，文字一个字不动，直接让 Agent 管理自己的 KV 缓存。

### 问题

先补一点背景。大模型每生成一个 token，都要回看前面所有内容。为了不每次从头算，推理系统会把前面 token 算好的中间结果存下来，这就是 KV cache，键值缓存。

Agent 一跑就是几十上百轮，搜索结果、工具返回不断往里堆，缓存越来越大。论文给的一个例子里，读取 KV 缓存占了 API 总成本的 82.4%。

另一个问题是准确率。参与计算的缓存太多，或者位置编号超出模型能稳定处理的范围，Agent 的表现都会明显下降。

### 以前的方法

以前的做法叫上下文工程，context engineering，就是在文字层面，把旧的历史消息删掉，或者总结成一段话。

问题在于，缓存是按顺序算的，后面 token 的缓存依赖前面的内容。改了前面一段文字，后面保留内容的缓存全部作废，要重新算，这叫 re-prefill，重新预填充。改得越早，重算越多。

所以常见的 Agent 框架会先攒着，到了阈值，再一次性把大部分历史总结掉，只留最近几轮。重算是少了，但中间轮次里有用的信息，也可能一起丢了。

作者用 GPT-OSS-120B，在深度搜索 benchmark BrowseComp-Plus 的前 100 个任务上测了这个两难，下面简称 BCP。只保留最近 12 轮工具返回的裁剪方案，成本是完整上下文的 1.85 倍。48K 阈值的总结方案，成本降了大约 7%，准确率却掉了 5 个百分点。

### 核心想法

作者的思路是，文字历史不变，只让选中的那部分缓存，不再参与之后的注意力计算，剩下的原样复用，不用重算。他们把这叫做缓存工程，cache engineering。

看这张图。第一行是全部保留，第二行是上下文工程，删掉前面的文字，后面的缓存要重算。第三行是缓存工程，丢掉无关的缓存，其余的直接复用。

还有一个细节。留下来的缓存，当初计算时看过那些后来被丢掉的内容，作者认为，它们可能还带着那部分的一些信息。

### 方法

具体来说，在普通的 append，也就是追加新 token 之外，论文定义了两个新的缓存操作。

第一个叫 drop，把选中的缓存从之后的注意力里排除，保留下来的缓存和它们的位置编码都不变。

第二个叫 repos，重新定位。因为丢掉缓存并不会让位置编号变小，编号还是跟着完整历史一直往上涨，缓存不大，编号也可能撞到模型的上限。repos 把保留的缓存重新编号，变成从零开始的连续位置。

对于用 RoPE 旋转位置编码的模型，这一步只需要把存好的 key 旋转一个角度，value 不动，也不用重新计算。

那该丢哪些缓存。作者统计了注意力，发现越旧的内容分到的注意力越少，而且在相近的位置，工具返回比模型自己的回答分到的更少。

所以他们的主要策略，叫工具返回滚动丢弃。所有推理内容都留着，工具返回只留最近的 12 轮，更早的缓存陆续丢掉。

但这带来一个服务端的麻烦。以前同样的 token 前缀，缓存一定一样，推理系统靠前缀匹配就能复用。现在不一样了。

看这张图。三条轨迹最后都保留 B 和 C，但第一条先算 C 再丢 A，第二条先丢 A 再算 C，第三条从来没有 A。C 的缓存三条各不相同，只看 token 前缀，就会错误复用。

所以作者做了配套的推理引擎 AKV，基于 SGLang。对外接口是无状态的，Agent 照常发消息，只多一个 cache 字段，写明在哪条消息之后丢掉哪些缓存，在哪里重新定位。

内部它做了三件事。第一是扩展的基数树，也就是 radix tree，在树里记录 drop 和 repos 这些操作节点，匹配时既看 token，也看操作历史。

第二是感知 drop 的回收。一段缓存被所有用到它的请求都丢掉之后，可以提前释放，而且这类缓存会被优先回收。

第三是分段批量注意力。被清出显存的缓存要恢复时，按操作发生的位置把查询分段，每段只看它当时能看到的缓存，一次前向计算就能全部恢复。

### 实验和效果

实验设置。基线是两种基于总结的上下文工程。Standard 阈值较高，GPT-OSS-120B 和 MiniMax-M2.7 是 96K token，MiroThinker-1.7-Mini 是 192K。Text Engineering 统一用 48K。AKV 保留最近 12 轮工具返回的缓存。

模型有三个，MiniMax-M2.7、GPT-OSS-120B 和 MiroThinker-1.7-Mini。任务有四个，深度搜索类的 BCP、xbench-DeepSearch 和 GAIA 的纯文本子集，还有写代码的 SWE-bench Verified。

结果在表 1。三个模型上，AKV 的平均成功率都是最高的，比 Standard 高 1.3 到 2.1 个百分点。平均成本比 Standard 低 27.7%，比 Text Engineering 低 21.9%。

举个例子，GPT-OSS-120B 的平均成功率从 51.0 提到 53.1，每个任务的平均成本从 11.2 美分降到 7.3 美分。

准确率的提升主要出现在长轨迹上。在 BCP 上，轨迹逻辑长度超过 96K token 的那一组，Standard 是 24%，Text Engineering 是 32%，AKV 是 37%。作者认为，这符合一次性大幅压缩会丢信息的解释。

再看速度，表 2。在 SWE-bench Verified 上下文最长的 80 个任务上，用 8 张 A800 跑 GPT-OSS-120B。各自吞吐最高的并发下，AKV 的平均请求延迟快 1.36 倍，P95 延迟，也就是 95% 的请求都能在这个时间内完成，快 1.51 倍，输出吞吐是 Standard 的 1.18 倍。

速度主要来自更短的活跃上下文。附录里，同样 batch size 下，全注意力时间减少 59.8%，每步解码总时间减少 16.0%。

### 和以前方法的对比

消融实验在表 3。只丢工具返回、不用 repos，BCP 是 55.0，还略低于 Standard 的 56.0。加上 repos，涨到 62.0。作者的解释是，不重新定位的话，位置编号会撞到上限，任务被迫中止。

如果连推理内容的缓存也丢，SWE-bench 上的分数从 41.0 掉到 36.5，或者 34.5。说明至少在这组 GPT-OSS-120B 的实验里，丢什么很关键，模型自己的推理最好留着。

论文还试了两种更自动的策略。第一种是递归自我改进，让模型自己搜索更好的缓存策略。用 MiniMax-M2.7 在 20 个 BCP 任务上迭代，再到 100 个没见过的任务上测，准确率不低于 Standard，成本低了 15.6% 到 17.8%。摘要里说，比人工设计的策略又省了 7.1%。

第二种是把缓存管理做成工具，让 Agent 自己决定什么时候丢。在 100 个 BCP 任务上，GPT-OSS-120B 成功率从 52.1 到 61.0，成本也降了。MiniMax-M2.7 成功率 64.0，但成本从 5.4 涨到 5.9 美分，因为跑了更多轮。

同期也有针对 Agent 的缓存淘汰工作，比如 AgentKV 和 IntentKV。作者说，AKV 的不同在于，它提供和具体策略无关的缓存操作，并且把这些依赖历史的缓存，纳入共享复用、显存管理和恢复。

我的判断是，这篇真正的贡献在系统层面。淘汰缓存不是新想法，难的是丢了以后，推理引擎还能正确地共享和复用缓存，AKV 把这个问题讲清楚了，也做出来了。

我的判断是，提升幅度要看清楚。平均成功率只高 1 到 2 个百分点，也不是每项都赢，比如 MiniMax-M2.7 在 SWE-bench 上，AKV 是 74.4，低于 Standard 的 75.2。更稳的收益在成本和延迟上。

### 局限

作者自己写的局限很简短，更多缓存策略还要探索，在这种机制下训练 Agent 还没做，对线性注意力模型的服务支持也还没有。

另外，成本是按 API 价格换算的估算，不是真实账单，repos 按缓存命中的价格计费，作者说这是故意往高了算。服务性能只测了 GPT-OSS-120B 一个模型。

还有，三个问答类任务用 GLM-5.2 当裁判判对错，只有 SWE-bench 用的是官方测试脚本。

### 未来方向

往后看，作者提到的方向是，探索更多缓存策略，以及让 Agent 在缓存工程的机制下训练。代码已经在 GitHub 公开。

我的判断是，这类接口要普及，需要模型服务商在 API 里开放 drop 和 repos 这样的操作。在那之前，能直接用上的，主要是自己部署开源模型的团队。

我的判断是，长期看，Agent 的上下文管理，可能会从改文字，变成文字和缓存分开管，丢什么、留什么，越来越多交给 Agent 自己。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Tsinghua University, with a co-author from the Chinese University of Hong Kong. It's called Cache Engineering for Agents.

The short version: when an agent's context gets too long, the usual fix is to cut or summarize the text. This paper leaves the text alone and lets the agent manage its own KV cache directly.

### Problem

Some background first. Every time a language model produces a new token, it looks back over everything before it. To avoid recomputing all of that each time, serving systems store each earlier token's intermediate results. That store is the KV cache, short for key-value cache.

Agents run for dozens or hundreds of rounds, and search results and tool outputs keep piling in, so the cache keeps growing. In one example the paper gives, reading the KV cache accounts for eighty-two point four percent of the total API cost.

Accuracy suffers too. If the model attends to too many cached states, or the position numbers run past what it handles reliably, agent performance drops sharply. So the active cache has to be kept under control.

### Prior work

The standard answer is context engineering: at the text level, you delete old messages or replace them with a summary.

The catch is that the cache is built in order. Every token's cache depends on what came before it. Edit an early message, and the cache for everything after it is invalid and has to be computed again. That's called re-prefill, and the earlier the edit, the more you recompute.

So common agent harnesses wait. Once the context hits a threshold, they summarize most of the history at once and keep only the last few rounds. That keeps re-prefill short, but useful information from the middle rounds can disappear with it.

The authors measured this trade-off with GPT-OSS-120B on the first one hundred tasks of BrowseComp-Plus, a deep-search benchmark, which I'll call BCP. Trimming to the latest twelve rounds of tool output cost one point eight five times as much as keeping the full context. Summarizing at a forty-eight-thousand-token threshold cut cost by about seven percent, but lost five percentage points of accuracy.

### Key idea

The authors' idea is to keep the text history exactly as it is, and simply stop selected parts of the cache from taking part in future attention. Everything that's kept is reused as is, with no recomputation. They call this cache engineering.

Here's the figure. The top row keeps everything. The middle row is context engineering: delete the early text, then recompute the cache for what follows. The bottom row is cache engineering: drop the irrelevant cache entries and reuse the rest directly.

One subtle point. The cache you keep was computed while the dropped content was still visible, so the authors argue it may still carry some of that information.

### Method

Concretely, on top of the usual append operation, which adds new tokens, the paper defines two new cache operations.

The first is drop. It removes selected cache entries from future attention and leaves the retained entries and their position encodings untouched.

The second is repos, short for reposition. Dropping cache doesn't shrink the position numbers, which keep climbing with the full history, so even a small cache can hit the model's position limit. Repos renumbers the retained cache into consecutive positions starting from zero.

For models that use RoPE, the rotary position encoding, that only means rotating the stored keys. The values stay the same, and nothing is recomputed.

So what should be dropped? The authors looked at attention and found that older content gets less of it, and that at similar positions, tool outputs get less attention than the model's own messages.

That gives their main policy, a tool-response rolling drop. All reasoning stays. Only the twelve most recent tool outputs keep their cache, and older tool-output cache is dropped as the run goes on.

But this creates a problem on the serving side. Normally, the same token prefix always means the same cache, so a server can reuse cache just by matching prefixes. That's no longer true.

Look at this figure. Three trajectories all end up keeping B and C. The first computes C and then drops A. The second drops A and then computes C. The third never had A. C's cache is different in all three, so matching on tokens alone would reuse the wrong one.

So the authors built a matching serving engine called AKV, on top of SGLang. Its interface is stateless. The agent sends messages as usual, plus one extra cache field saying after which message to drop which messages' cache, and where to reposition.

Inside, it does three things. First, an extended radix tree that records drop and repos as nodes in the tree, so matching checks both the tokens and the operation history.

Second, drop-aware eviction. Once every request using a piece of cache has dropped it, that memory can be freed early, and that kind of cache is evicted first.

Third, segment-batch attention. When cache has been pushed out of GPU memory and needs restoring, the queries are split at the points where operations happened, each segment sees only the cache it saw originally, and everything is rebuilt in a single forward pass.

### Results

The setup. The baselines are two summarization-based context engineering methods. Standard summarizes at a high threshold: ninety-six thousand tokens for GPT-OSS-120B and MiniMax-M2.7, and a hundred ninety-two thousand for MiroThinker-1.7-Mini. Text Engineering uses forty-eight thousand for all of them. AKV keeps the cache of the latest twelve tool outputs.

There are three models, MiniMax-M2.7, GPT-OSS-120B and MiroThinker-1.7-Mini, and four benchmarks: BCP, xbench-DeepSearch and the text-only subset of GAIA for deep search, plus SWE-bench Verified for coding. Cost is computed from published API prices.

The results are in Table one. On all three models, AKV has the highest average success rate, one point three to two point one points above Standard. Averaged over everything, it costs twenty-seven point seven percent less than Standard and twenty-one point nine percent less than Text Engineering.

For example, GPT-OSS-120B's average success rate goes from fifty-one point zero to fifty-three point one, while average cost per task falls from eleven point two cents to seven point three.

The accuracy gain shows up mostly on long runs. On BCP, in the group of runs whose logical length goes past ninety-six thousand tokens, Standard scores twenty-four percent, Text Engineering thirty-two, and AKV thirty-seven. The authors say this fits the idea that abrupt, large reductions lose information, while a rolling drop removes it a little at a time.

Now speed, in Table two. On the eighty SWE-bench Verified tasks with the longest contexts, running GPT-OSS-120B on eight A800 GPUs, each method at its best-throughput concurrency: AKV is one point three six times faster in mean request latency, one point five one times faster at P95, the time within which ninety-five percent of requests finish, and gets one point one eight times Standard's output throughput.

Most of that speed comes from a shorter active context. In the appendix, at the same batch size, full-attention time drops by fifty-nine point eight percent and total decode-step time by sixteen percent.

### Comparison

The ablation is in Table three. Dropping tool output without repos scores fifty-five on BCP, slightly below Standard's fifty-six. Add repos and it rises to sixty-two. The authors' explanation is that without repositioning, position numbers hit the limit and the run has to stop.

Drop reasoning cache as well, and SWE-bench scores fall from forty-one to thirty-six point five or thirty-four point five. At least in these GPT-OSS-120B runs, what you drop matters, and the model's own reasoning is worth keeping.

The paper also tries two more automatic approaches. The first is recursive self-improvement, where the system searches for better cache policies itself. Refined with MiniMax-M2.7 on twenty BCP tasks and tested on a hundred unseen ones, the resulting policies match Standard's accuracy at fifteen point six to seventeen point eight percent lower cost. The abstract puts it at seven point one percent cheaper than the hand-designed policy.

The second turns cache management into a tool, so the agent decides when to drop. On a hundred BCP tasks, GPT-OSS-120B goes from fifty-two point one to sixty-one percent success, with cost down from thirteen point four cents to twelve point one. MiniMax-M2.7 reaches sixty-four percent, but its cost rises from five point four to five point nine cents because it runs more turns.

There is concurrent work on cache eviction for agents, such as AgentKV and IntentKV, which use query information and importance estimates to decide what to keep. The authors say AKV differs by offering cache operations that don't depend on any one policy, and by bringing these history-dependent cache states into shared reuse, memory management and recovery.

My read is that the real contribution here is the systems work. Evicting cache isn't a new idea. The hard part is keeping cache sharing and reuse correct after you drop things, and AKV spells that problem out and solves it.

My read is also that the gains need to be seen clearly. Average success rises by only one to two points, and AKV doesn't win everywhere. On SWE-bench with MiniMax-M2.7, for instance, it scores seventy-four point four against Standard's seventy-five point two. The steadier wins are in cost and latency.

### Limitations

The authors' own limitations section is brief: more cache strategies remain to be explored, training agents under this scheme is left for later, and serving support for linear-attention models doesn't exist yet.

A few more things to keep in mind. Costs are estimates from API prices, not real bills, and repos is charged at the cache-hit rate, which the authors say deliberately overstates it. Serving performance was measured on only one model, GPT-OSS-120B, on A800s with BF16 weights.

Also, the three question-answering benchmarks are graded by GLM-5.2 acting as a judge. Only SWE-bench uses the official test scripts.

### Future work

Looking ahead, the authors point to exploring more cache strategies and to training agents under cache engineering. The code is already public on GitHub.

My read is that for this kind of interface to spread, model providers would need to expose operations like drop and repos in their APIs. Until then, the teams who can use it directly are mostly those serving open models themselves.

My read is that over time, agent context management may shift from editing text to managing text and cache separately, with more of the decisions about what to drop and what to keep handed to the agent itself.

That's today's Daily Frontier Papers. The paper is linked in the description.

