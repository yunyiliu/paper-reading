# Evolving Infra: Agent-Adaptive LLM Serving at Scale

**几百个Agent一起跑算力显存怎么分**

- 论文 Paper: [arXiv 2611.12304](https://arxiv.org/abs/2611.12304)
- 单位 Affiliation: 佐治亚理工 · 伯克利 · MIT 等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-13
- 视频 Videos: —

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇佐治亚理工、伯克利、MIT 等机构合作的论文，叫 Evolving Infra，讲的是面向 Agent 的大模型推理服务。

它关心的问题很实际，几百个 Agent 同时在跑，背后的推理集群该怎么分配算力和显存。论文报告，和英伟达的推理框架 Dynamo 相比，平均端到端延迟最多降低 4.1 倍，P95 首 token 时间最多降低 14 倍。

### 问题

先补背景。模型处理请求分两步，prefill 把输入一次读进去，算出 KV cache，也就是模型对前文的中间结果，decode 再逐个 token 往外生成。前文的 KV 缓存过，prefill 就只算新增部分。

Agent 是一步一步干活的，每一步是一次模型请求，加上它触发的工具调用。前后两次请求共享很长的前缀，所以 KV 可以大量复用。

主流集群用 PD 分离，一部分节点专做 prefill，一部分专做 decode。问题是，不同 Agent 的需求差得很远。电脑操作 Agent 每一步都加一张截图，prefill 很重。数学 Agent 每次生成很长的推理，decode 很重。这张图里，三种 Agent 最合适的分配，从 2 个 prefill 节点配 7 个 decode 节点，到 6 配 3 都有。

而且集群里各种 Agent 的比例会随时间变。负载从数学 Agent 过渡到电脑操作 Agent，每十分钟的最优分配从 3 配 6 变到 7 配 2。就算一直用最好的固定分配，某些时段也会损失多达 24% 的吞吐。

还有一层，Agent 会压缩历史、分叉、派出子 Agent。论文在 Claude Code 的真实轨迹上看到，一次 359 秒的子 Agent 调用之后，下一个请求有 98% 能复用之前的 KV，一次 12 秒的压缩之后，只有 14%。最近用过的，不一定还有用。

### 以前的方法

现有系统靠三个部件。planner 决定多少节点做 prefill，多少做 decode，router 决定请求发给谁，内存池负责共享和淘汰 KV。作者认为，在 Agent 场景下，三个部件各有问题。

第一，切换角色太贵。Dynamo 每次给节点换角色都要重启，在 Claude Code 轨迹上平均一次 681 秒，期间节点不接请求。Arrow 能原地切换，但切完节点上的 KV 就用不上了，作者复现的 Arrow 丢掉了 55.5% 本可复用的 KV。

第二，算力需求估不准。planner 看队列长度这类粗信号，几个子 Agent 一起启动，造成短暂高峰，就可能触发没必要的切换。有的还把已缓存的前缀也算成 prefill 工作量。结果 Dynamo 的 planner 和 Arrow，吞吐比最好的固定分配分别低 49% 和 65%。

第三，KV 复用判断错。按缓存命中路由，请求会挤到少数节点上，P95 首 token 时间到 27 秒和 34 秒。发到别处，又要重算或远程取，P95 反而涨到 65 秒和 95 秒。淘汰按最近最少使用，会删掉正在等工具的分支的 KV，却留着被压缩替换的旧历史。

### 核心想法

作者把问题归到两个根源。一是算力分配和 KV 缓存绑在一起，节点一换角色，KV 就丢了或够不着了。二是三个部件只看粗信号，看不到 Agent 真正的工作量和上下文管理。

所以核心想法分两层。先把算力分配和 KV 缓存解耦，再让算力分配跟着 Agent 的工作量走，KV 管理跟着 Agent 的上下文管理走。

看这张总览图，同一批 GPU 既能做 prefill 也能做 decode，下面是所有节点的主机内存拼成的共享 KV 池，每块 KV 标着有几个分支还在引用它。

### 方法

方法分三块。第一块是基础设施。作者在 SGLang 上做了一个对称引擎，一个调度循环同时支持两种角色，节点换角色不用重启。

所有节点的主机内存拼成共享 KV 池，prefill 节点可以通过 RDMA 直接从别的节点拉 KV。节点换角色，它的 KV 留在池里，别人照样能读。

还有一个细节，以前是 prefill 开始时才把远端 KV 加载进显存，会拖慢 prefill。现在请求排队时就把 KV 取完，取数据的时间被排队时间盖住了。

第二块是全集群的 KV 目录。它是一棵前缀树，记录每块 KV 在哪个节点上，以及有几个活动分支在引用它。

计数要 Agent 框架配合。框架给每个请求标上分支编号，分支结束时通知 router。请求只是追加内容，就只给新增的块加计数。上下文被压缩改写，旧块减计数，新块加计数。

淘汰时，先删计数为零的块，比如被压缩替换的历史、已结束子 Agent 的上下文、传输留下的重复副本，删完了才轮到还在用的块。这样，等工具、等用户的分支，KV 会排到最后才删。

第三块是调度。作者给 prefill 和 decode 各建了一个迭代时间模型，思路来自 roofline，prefill 主要受算力限制，decode 主要受显存带宽限制。关键是，prefill 模型把已缓存和未缓存的 token 分开算。

模型系数完全在线拟合，不需要离线 profiling。论文报告，在拟合以外的设置上，平均误差低于 7%。

planner 每个规划窗口结束时，用这段时间请求的平均工作量，预测每种分配的吞吐。只有新分配的收益在统计上站得住，平均增益超过两倍标准误，才切换。这样短暂的高峰不会让集群来回折腾。

router 也不再追着缓存命中跑，而是预测请求在每个 prefill 节点上的首 token 时间，选最短的，KV 趁排队时从共享池取。

### 实验和效果

实验设置。数学证明用 PutnamBench，电脑操作用 OSWorld 2.0，还有真实用户的 Claude Code 轨迹回放，作者说这是最动态的一种。模型是 GLM-5.2 和 Kimi K2.6，9 个节点，一共 36 张 GB300，规划窗口 600 秒。主要基线是 Dynamo 1.5.1。

结果在这张图里。固定负载下和 Dynamo 比，吞吐是它的 1.1 到 2.73 倍，丢掉的 KV 复用少了 81% 到 95%，平均延迟降低 1.15 到 4.12 倍。

分场景看，Claude Code 收益最大，平均延迟降低 3.8 到 4.12 倍，90 个并发 Agent 时，P95 首 token 时间从 142 秒降到 10.2 秒。OSWorld 降低 2.34 到 2.69 倍，数学 Agent 只有 1.15 到 1.23 倍。

切换也快了很多。Evolving Infra 一次切换平均 6.2 秒，包括等正在跑的请求完成，Dynamo 重启一次平均 754 秒。

再看负载随时间变化的实验，三个小时里从数学 Agent 过渡到电脑操作 Agent。Evolving Infra 的分配基本跟上了每十分钟的最优分配，吞吐是 Dynamo 的 2.14 倍，是最好固定分配的 1.28 倍。

消融实验里，从四种初始分配出发，平均延迟差别不超过 1.8%。在最优固定分配上打开在线调整，吞吐只掉 0.3%。不调分配，只用新的基础设施和 KV 管理，吞吐也有最好固定分配的 1.37 倍。

规模上，从 36 张卡扩到 256 张 GB300，固定分配的 P95 首 token 时间从 142 秒涨到 418 秒，Evolving Infra 一直在 10 到 14 秒之间。

### 和以前方法的对比

和以前的方法比，Dynamo 能调分配，但要重启。Arrow 能原地切，但 KV 用不上，还把缓存的前缀算成工作量。Continuum、ThunderAgent 这类面向 Agent 的系统会保留等工具的 Agent 的 KV，但假设上下文只会变长，也不追踪重复副本。

我的判断是，这篇最有意思的地方，是把分支编号、何时压缩、子 Agent 何时结束这些 Agent 框架里的信息，直接交给推理系统。

我的判断是，收益在不同负载上差别很大。数学 Agent 这种 decode 为主、没有上下文管理的场景，平均延迟只改善 1.15 到 1.23 倍，224 和 288 个并发时，P95 首 token 时间反而比 Dynamo 高。论文的解释是，planner 为了吞吐只留了一两个 prefill 节点。

### 局限

论文没有单独的局限章节，但从设置能看出几点。第一，需要 Agent 框架配合，给请求打分支编号，分支结束时通知 router，KV 目录的计数就靠这些信号。

第二，主要对比对象是 Dynamo。Arrow 代码没公开，用的是作者的复现。Llumnix 跑不了 GB300 节点的 Arm CPU，只在 H200 上比了一组，那组里 Evolving Infra 的吞吐比最好的固定分配高 9%。

第三，3072 张卡的结果来自模拟，模拟器假设取 KV 不花时间。Claude Code 轨迹是回放的，输出长度和工具耗时都固定为录制值。

### 未来方向

附录里讨论了更大的集群。几千张卡时单个 router 管不过来，作者把模拟集群分成 8 个分区，各有自己的 router、planner 和 KV 目录，各分区的 planner 跟着本地需求调整，吞吐是固定分配的 1.42 倍。

我的判断是，Agent 成了推理需求的大头，推理系统和 Agent 框架之间，很可能需要一个标准接口来传递分支、压缩这类信息。代码已开源，值得看的是主流推理框架会不会跟进。

我的判断是，数字也要冷静看。4.1 倍和 14 倍来自上下文管理最复杂的 Claude Code 负载，数学 Agent 上收益小得多，普通单轮对话论文没有测。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from a group spanning Georgia Tech, Berkeley, MIT and others. It's called Evolving Infra, and it's about serving language models to agents.

The question is a practical one. When hundreds of agents run at once, how should the inference cluster behind them split its compute and memory? Compared with NVIDIA's Dynamo serving framework, the paper reports up to four point one times lower mean end-to-end latency, and up to fourteen times lower P95 time to first token.

### Problem

Some background. Serving a request has two phases. Prefill reads the input in one pass and computes the KV cache, the model's intermediate state for the context. Decode then generates output one token at a time. If the earlier context's KV is cached, prefill only processes the new part.

Agents work in steps. Each step is one model request plus the tool calls it triggers. Consecutive requests share long prefixes, so there's a lot of KV to reuse.

Large deployments use prefill-decode disaggregation: some nodes only prefill, the rest only decode. But agents need very different mixes. A computer-use agent adds a screenshot at every step, so it's prefill-heavy. A math agent writes long reasoning traces, so it's decode-heavy. In this figure, the best split for three agent types ranges from two prefill and seven decode nodes to six and three.

And the mix on a cluster shifts over time. As the load moves from math agents to computer-use agents, the best split for each ten-minute window goes from three-and-six to seven-and-two. Even the best fixed split loses up to twenty-four percent of throughput in some periods.

There's another layer. Agents manage their own context. They compact history, fork, and spawn sub-agents. On real Claude Code traces, after a three-hundred-fifty-nine-second sub-agent call, ninety-eight percent of the next prompt could reuse earlier KV. After a twelve-second compaction, only fourteen percent could. Recent use doesn't mean the KV is still needed.

### Prior work

Existing systems use three parts. A planner decides how many nodes prefill and how many decode, a router picks a node for each request, and a memory pool shares and evicts KV. The authors argue each part struggles with agents.

First, switching roles is expensive. Dynamo relaunches a worker for every role change, which takes six hundred eighty-one seconds on average on the Claude Code traces, with the node out of service. Arrow switches in place, but the node's cached KV becomes unreachable, and the authors' reimplementation lost fifty-five point five percent of available KV reuse.

Second, compute demand is misjudged. Planners watch coarse signals like queue depth, so a few sub-agents starting together can trigger a pointless switch. Some also count cached prefixes as prefill work. Dynamo's planner and Arrow end up with forty-nine and sixty-five percent less throughput than the best fixed split.

Third, KV reuse is misjudged. Route by cache hits, and requests pile onto a few nodes, pushing P95 time to first token to twenty-seven and thirty-four seconds. Spread the load, and other nodes must recompute or fetch the prefix, and P95 climbs to sixty-five and ninety-five seconds. Least-recently-used eviction drops the KV of branches waiting on tools, while keeping history that compaction already replaced.

### Key idea

The authors see two root causes. Compute allocation and the KV cache are tied together, so changing a node's role loses its KV or puts it out of reach. And all three parts rely on coarse signals that don't show the agents' real work or context management.

So the idea has two layers. Decouple compute allocation from the KV cache. Then let compute allocation follow the agent workload, and KV management follow the agents' context management.

In this overview, the same GPUs can serve prefill or decode. Underneath is a shared KV pool built from every node's host memory, with each block labeled by how many branches still reference it.

### Method

The method has three pieces. First, infrastructure. The authors extend SGLang into a symmetric engine with one scheduling loop for both phases, so a node changes roles without a restart.

Every node's host memory joins one shared KV pool, and a prefill node can pull KV from a peer over RDMA. When a node switches roles, its KV stays in the pool for others to read.

One detail matters here. Remote KV used to be loaded into GPU memory only when prefill started, which slowed prefill down. Now the fetch finishes while the request waits in the queue, so the transfer hides behind queueing time.

Second, a cluster-wide KV directory. It's a prefix tree recording which node holds each block, and how many active branches reference it.

The counts need the agent harness to cooperate. The harness tags each request with its branch ID and reports when a branch finishes. An appending request only adds counts to its new blocks. When compaction rewrites a context, old blocks lose their counts and new ones gain them.

Eviction removes zero-count blocks first: replaced history, contexts of finished sub-agents, and duplicate copies left by transfers. Only then does anything still in use go. So a branch waiting on a tool or a user is the last to lose its KV.

Third, scheduling. The authors build iteration-time models for prefill and decode, following the roofline idea: prefill is mostly compute-bound, decode mostly memory-bound. The key is that the prefill model treats cached and uncached tokens separately.

The coefficients are fitted entirely online, with no offline profiling. The paper reports a mean error below seven percent outside the fitted settings.

At the end of each planning window, the planner uses the average work per request to predict the throughput of every split. It switches only if a new split's gain is statistically solid, with a mean gain above twice its standard error. Short bursts don't make the cluster thrash.

The router stops chasing cache hits too. It predicts the request's time to first token on every prefill node, picks the shortest, and the KV is fetched from the pool while the request waits.

### Results

The setup. Math proofs on PutnamBench, computer use on OSWorld 2.0, and replayed Claude Code traces from real users, which the authors call the most dynamic. The models are GLM-5.2 and Kimi K2.6, on nine nodes with thirty-six GB300 GPUs, and a six-hundred-second planning window. The main baseline is Dynamo version one point five point one.

Here are the results. Under steady load, versus Dynamo, throughput is one point one to two point seven three times higher, lost KV reuse drops by eighty-one to ninety-five percent, and mean latency falls by one point one five to four point one two times.

Claude Code gains the most, with mean latency down three point eight to four point one two times, and at ninety concurrent agents, P95 time to first token falls from one hundred forty-two seconds to ten point two. OSWorld improves two point three four to two point six nine times, and the math agents only one point one five to one point two three times.

Switching gets much faster. An Evolving Infra switch takes six point two seconds on average, including waiting for running requests, while a Dynamo relaunch takes seven hundred fifty-four seconds.

Next, a workload shifting over three hours from math agents to computer-use agents. Evolving Infra's split tracks the best split of each ten-minute window, and throughput is two point one four times Dynamo's and one point two eight times the best fixed split.

In the ablations, starting from four different initial splits changes mean latency by at most one point eight percent. Turning on online adaptation at the best fixed split costs only zero point three percent of throughput. And with the split held fixed, the new infrastructure and KV management alone reach one point three seven times the best fixed split's throughput.

At scale, going from thirty-six to two hundred fifty-six GB300 GPUs, the fixed split's P95 time to first token rises from one hundred forty-two seconds to four hundred eighteen, while Evolving Infra stays between ten and fourteen.

### Comparison

Against earlier work: Dynamo can change the split, but has to relaunch. Arrow switches in place, but loses the KV and counts cached prefixes as work. Agent-aware systems like Continuum and ThunderAgent keep the KV of agents waiting on tools, but assume contexts only grow and don't track duplicate copies.

My read is that the most interesting move is handing harness information, like branch IDs, compactions, and when a sub-agent ends, straight to the serving system. The serving layer no longer sees just a stream of unrelated requests.

My read is also that the gains vary a lot by workload. On decode-heavy math agents with no context management, mean latency improves only one point one five to one point two three times, and at two hundred twenty-four and two hundred eighty-eight agents, P95 time to first token is worse than Dynamo's. The paper explains that the planner keeps only one or two prefill nodes to maximize throughput.

### Limitations

There's no separate limitations section, but the setup points to a few. First, the harness has to cooperate, tagging requests with branch IDs and reporting when branches finish. The directory's counts depend on those signals.

Second, the main comparison is against Dynamo. Arrow's code isn't public, so it's the authors' reimplementation. Llumnix can't run on the GB300 nodes' Arm CPUs, so it's compared once, on H200 GPUs, where Evolving Infra beats the best fixed split by nine percent in throughput.

Third, the three-thousand-and-seventy-two-GPU results are simulated, and the simulator assumes KV fetches take no time. And the Claude Code traces are replayed, with output lengths and tool durations fixed to recorded values.

### Future work

The appendix looks at bigger clusters. With thousands of GPUs, one router can't track every node, so the authors split a simulated cluster into eight partitions, each with its own router, planner and KV directory. Each partition's planner follows local shifts in demand, for one point four two times the throughput of a fixed split.

My read is that as agents become the bulk of inference demand, serving systems and agent harnesses will likely need a standard way to pass along things like branches and compactions. The code is open source, and the thing to watch is whether mainstream serving frameworks pick this up.

My read is also that the headline numbers need a calm look. Four point one times and fourteen times come from Claude Code, the workload with the most complex context management. The math agents gain much less, and plain single-turn chat wasn't tested.

That's today's Daily Frontier Papers. The paper is linked in the description.

