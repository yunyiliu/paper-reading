# CompoundingKernel: Self-Improving from Zero to First

**瓶颈不是Agent数量而是测量速度**

- 论文 Paper: [arXiv 2611.19230](https://arxiv.org/abs/2611.19230)
- 单位 Affiliation: Amazon（亚马逊）
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-22
- 视频 Videos: [YouTube 中文](https://youtu.be/5EGkBkGXwtY) · [YouTube English](https://youtu.be/-1saH66-HEQ) · [B站](https://www.bilibili.com/video/BV1kUyGzPEcG)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇亚马逊的论文，名字叫 CompoundingKernel，副标题是，从零到第一。

它让一大群 AI Agent 自己写 GPU kernel，结果在英伟达的公开基准 SOL-ExecBench 上，三个公开榜单都排到了第一。

先交代一句，最终运行里所有 Agent 用的都是 Claude Opus 5.5，模型全程不变，变的只是模型外面的搜索系统。

### 问题

先说什么是 kernel。它是在 GPU 上执行某一个运算的程序，kernel 越快，同样的模型计算，花的 GPU 时间就越少。

难点在新硬件。英伟达的 Blackwell，快的路径要用新的 tensor core 指令、tensor memory 和 TMA 数据搬运引擎，而大多数融合算子和低精度形状，还没有现成的库 kernel 能用上这些。

评测用的 SOL-ExecBench，是英伟达给 AI Agent 写 kernel 准备的公开基准，235 个问题，来自 124 个真实模型，分成单算子 L1、融合算子 L2、量化等四个榜单。

打分是这样的。每个输入形状有两个时间，一个是英伟达没有公开的优化版 PyTorch 基线，一个是理论极限，叫 speed of light。和基线一样快得 0.5 分，达到极限得 1 分，算错得 0 分。

更麻烦的是正确性。错的 kernel 通常不会崩溃，而是返回看起来正常的数字。这些 kernel 中位数大约 7000 行，按 Agent 写代码的速度，人审不过来。

而且搜索越强，越会钻检查的空子，也就是 reward hack，不干真活也能拿分。基准作者自己跑 Agent 时，就有 589 个提交，占百分之 14.5，因为 reward hack 被拒。

### 以前的方法

以前大致两条路。一条是用强化学习或微调，专门训练写 kernel 的模型，大多在 KernelBench 上评测。

另一条是模型固定，在外面搭搜索循环。论文说，在 Blackwell 上，追上专家库、或者冲到公开榜单前列的结果，都来自这条路，这篇也是。

论文表 6 列了几个同类系统，比如 Cursor 和英伟达合作的多 Agent 系统，也跑了全部 235 题。它们的区别主要在搜索结构和验证方式。

还有一个前人结论很关键。只给 Agent 一个目标，不加领域知识，本身就是很强的基线。所以复杂的系统，得用测量证明自己有用。

### 核心想法

核心想法是，让每次新尝试都建立在对上一次的可靠测量之上，进步才能一层层叠起来，论文把这叫做会复利的搜索。

所谓从零开始，说的是搜索，不是模型。每个问题只给 PyTorch 定义、形状和数据类型，不给参考 kernel，也看不到官方分数。但模型在公开代码上预训练过，Agent 也能查资料和上网。

整个系统是四层嵌套的循环，越往外越慢，这张图画的就是这四层。

### 方法

最里层是子 Agent。它拿到一个起点 kernel，叫父节点，加一个要尝试的方向，然后改代码、编译，在 B200 上计时，对不对由模型外面的检查来判。最终运行里，子 Agent 拿到第一个正确的 kernel 就停。

第二层是控制器。每个问题一个 LLM 控制器，有空位时它读一遍档案，选一个已有 kernel 接着改，或者开一个全新设计，再给每个子 Agent 分一个不同的方向。

档案是关键。它在模型外面，每个问题一份，记着每个正确的 kernel、计时和父节点。Agent 只能用只读 SQL 查，只有评测能写，而且只写正确的 kernel。

控制器不保留状态，每一轮都重读档案。所以第三层，维护系统的 lead 和 helper Agent，可以热重启，停下 Agent，改提示词和工具，再启动，档案不丢，搜索接着走。

最外层是人。人决定把哪个 kernel 提交到官方榜单，什么时候提交。官方分数只在这一层出现，不进入搜索。

论文也写清了人的参与。从 10 月 25 日起，人按每个问题离第一名的差距，调整它的子 Agent 数量，总数不变，Agent 只收到一个数量。

奖励来自本地计时，是相对官方基线的几何平均加速比。他们没用 SOL 分数当奖励，因为加速比很高时，它会变平。

还有，慢的 kernel 也不扔。控制器可以从档案里任何节点往下扩展，不只是当前最好的。一个结构全新、一开始更慢的 kernel，可能是垫脚石。

正确性由模型外面的检查决定，分五类，官方正确性，额外形状和极端数值下的鲁棒性，防 reward hack，能否在限时内编译，还有复核和官方打分。

### 实验和效果

先看结果。截至 2026 年 11 月 11 日，它在 L1、L2 和量化三个公开榜单都排第一，SOL 平均分大约是 0.823、0.819 和 0.910。

但领先幅度要看清。L1 比第二名高大约 0.01，L2 高不到 0.001，量化榜第一和第二只差 0.000006。

235 个问题，每个都有官方判定正确的 kernel。在三个公开榜单的 209 个问题上，速度是官方基线的 4.40 倍，这是几何平均，中位数 3.78 倍，用的是官方计时。

按单个问题算，209 个里有 89 个第一，193 个进前十。所以榜单第一，主要靠覆盖面广。

加速也不是靠极端值撑的。大多数问题在 2 到 10 倍之间，39 个超过 10 倍，最高 1062 倍，也有两个比基线还慢。

这些也不是库的套壳。235 个 kernel 里，214 个是 CUDA C++。按源码文本匹配，百分之 77 用了 tcgen05 tensor core 指令，百分之 94 含有内联 PTX，也就是很底层的 GPU 指令。

收益大多来得早。运行时间的前百分之 15，每个榜单就拿到了总提升的百分之 73 到 85，这是本地分数。

一个案例能说明垫脚石的思路。量化第 30 题，是一个 NVFP4 解码器层。在 9.71 倍时，控制器让子 Agent 把注意力换成 Blackwell 原生的 CuTe DSL 写法，第一次只跑出 1.52 倍。

下一个子 Agent 查出是寄存器溢出，提到 7.10 倍，还是比父节点慢。第三个子 Agent 把新注意力移到当时最好的节点上，到了 13.97 倍，成为新纪录。这条路径最后到 27.36 倍，都是本地计时。

规模上最重要的发现在这里。一开始，GPU 锁全被占着，每块 GPU 大约五个调用在排队，可 GPU 只有大约百分之 13 的时间在干活，因为锁把 CPU 编译和十几秒的 shell 启动也包进去了。

一次热重启把编译挪出锁，shell 启动降到毫秒级。前后对比，每小时评测的候选数变成 3.6 倍。作者也说明，这次重启捆绑了几个改动，没有拆开测。

### 和以前方法的对比

再看论文自己的对比。同样的 Claude 子 Agent，同样的档案设计，每边 24 小时预算，比较 LLM 控制器，和一个按固定的 UCB 式公式挑节点的程序化搜索树。

相同时间下，LLM 控制器在 38 个问题里赢了 35 个，最好加速比的几何平均高大约百分之 14。

有意思的是，按相同尝试次数比，两边差不多，控制器只赢 15 个，输 23 个。它赢在同样时间里做了 2.4 倍的正确尝试，因为搜索树很快把宽度缩到 4、2、1，位置空着，控制器一直派满。

另一个对比是加指导。把可选的参考文档和技能改成必须用，使用率涨了 4.3 倍，最终加速比却没提高，233 对问题的几何平均比值是 0.981，和只给目标就够的那篇结论一致。

我的判断是，这篇最该记住的不是榜单第一，而是那句话，限制规模的是可靠测量的速度，不是 Agent 数量，不过作者也说，还没有直接改变 Agent 数量来验证。很多人做多 Agent 系统先想加 Agent，这篇的数据提示，瓶颈可能在评测管线上。

我的判断是，榜单第一也要打个折扣。L2 和量化领先很少，而且榜单一直在变，作者自己就记录了几次被其他团队超过，又靠公开更多结果拿回第一。

### 局限

作者列的局限很坦白。证据来自一个基准，一种硬件 B200，一个模型，加一次冷启动的最终运行。

正确性是测出来的，不是证明出来的，他们找过的形式化验证工具，都不支持 Blackwell 的异步 tensor core 操作。鲁棒性检查也只覆盖每次的新最好成绩，不是每个候选。

搜索里也确实出现过 reward hack。量化第 30 题有一段路径，kernel 在第一次运行时先计时再挑最快的版本，后来被规则禁止，改掉以后本地速度掉了百分之 6。

另外，提交什么、何时提交是人定的，算力也按排名差距分配。三个设计对比都只跑了一次，要么在最终版之前的系统上，要么只是短时间的筛选测试，结论不一定适用于最终系统。

代码和 kernel 目前是计划发布，而且不包括 Agent 的提示词。

### 未来方向

作者说下一步要做更便宜的验证，让更多尝试能被可靠测量，也要在其他 GPU 和基准上测试。

他们还要把 kernel 放进推理服务框架，测首个 token 延迟和吞吐。作者预计这个收益会比单个 kernel 的加速小，因为推理框架本来就用着调好的库 kernel。

我的判断是，这篇对做 Agent 系统的人很有参考价值。记忆和检查都放在模型外面，控制器不存状态，系统就能边跑边修。

我的判断是，门槛也很高。一千多个 Agent 同时跑，每个候选都要在 B200 上完整编译和计时，一般团队很难复现这个规模。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Amazon. It's called CompoundingKernel, with the subtitle, self-improving from zero to first.

The pitch: a large fleet of AI agents writes GPU kernels on its own, and on NVIDIA's public SOL-ExecBench benchmark, its kernels rank first on all three public boards it entered.

One thing up front. In the final run, every agent is the same model, Claude Opus 5.5, and the model never changes. What changes is the search system built around it.

### Problem

First, what's a kernel? It's the program that runs one operation on a GPU. Faster kernels mean the same model work takes less GPU time.

The hard part is new hardware. On NVIDIA's Blackwell chips, the fast paths go through new tensor-core instructions, tensor memory, and a copy engine called TMA, and most fused or low-precision shapes don't have a library kernel that uses them yet.

The test is SOL-ExecBench, NVIDIA's public benchmark for kernels written by AI agents. It has two hundred and thirty-five problems drawn from one hundred and twenty-four real models, split across four boards: single operations, called L1, fused operations, called L2, quantization, and FlashInfer-Bench.

Here's the scoring. Every input shape has two published times: an optimized PyTorch baseline that NVIDIA hasn't released, and a speed-of-light bound, the time a perfect kernel would take. Match the baseline and you score zero point five. Hit the bound and you score one. Get it wrong and you score zero.

Then there's correctness. A wrong kernel usually doesn't crash. It returns plausible numbers. And the agents' kernels have a median length of about seven thousand lines, more than a person can audit at the rate agents write them.

Worse, a search strong enough to find fast kernels will also find gaps in the checker. Those are reward hacks, ways to score without doing the real work. In the benchmark authors' own agent runs, five hundred and eighty-nine submissions, fourteen point five percent, were flagged as reward hacks and rejected.

### Prior work

There have been two broad approaches. One trains a kernel-writing model with reinforcement learning or fine-tuning, mostly evaluated on KernelBench.

The other keeps the model fixed and builds a search loop around it. The paper notes that on Blackwell, the results that reach expert libraries or the top of public boards come from this second line, and that's where this work sits.

Table six lists related systems. Cursor's work with NVIDIA, for instance, also ran all two hundred and thirty-five problems with a multi-agent setup. Most keep the model fixed and differ mainly in search structure and validation.

And there's a key earlier finding. An agent given nothing but the goal is already a strong baseline. So any elaborate system has to earn its place by measurement.

### Key idea

The core idea is to make each new attempt build on a reliable measurement of the last one, so progress stacks up. The paper calls these searches that compound.

And "from zero" describes the search, not the model. Each problem starts with only its PyTorch definition, shapes and data types. No reference kernels, and no official scores, ever. But the model was pretrained on public code, and agents could read the CUTLASS headers, the web, and a public wiki.

The system is four nested loops, each slower than the one inside it, and this figure shows all four.

### Method

The innermost loop is a child agent. It gets a starting kernel, its parent, plus one direction to try. It edits the code, compiles it, and times it on a B200. Whether the result is correct is decided by checks outside the model. In the final run, a child stops at its first correct kernel.

The second loop is the controller. Each problem has one LLM controller. Whenever a child slot frees up, it reads the archive, picks an existing kernel to refine or opens a fresh design, and hands each child a distinct direction.

The archive is the key piece. It lives outside the model, one per problem, and records every correct kernel with its timings and its parent. Agents can only query it with read-only SQL. Only an evaluation can write to it, and only for a correct kernel.

The controller keeps no state of its own. It re-reads the archive every round. That lets the third loop, lead and helper agents that maintain the system, do warm restarts: stop the agents, update their prompts and tools, and relaunch, without losing the archive. The search picks up where it left off.

The outermost loop is people. They decide which kernels to send to the official board, and when. Official scores live only in this loop. They never enter the search.

The paper is also clear about where people stepped in. From October twenty-fifth, people set each problem's number of child slots based on its gap to first place. Problems close behind got more, problems safely ahead gave some up, and the total stayed the same. The agents only ever saw a slot count.

The reward comes from local timing: the geometric-mean speedup over the official baseline. They didn't use the SOL score as the reward, because it flattens out at high speedups.

There's one more design choice. Slow kernels aren't thrown away. The controller can expand any node in the archive, not just the current best. A new structure that starts out slower can be a stepping stone to something better.

Correctness is decided by checks outside the model, in five classes: official correctness, robustness on extra shapes and stress values, reward-hack rules, whether the kernel builds within the site's time limit, and review plus the official score. The appendix lists all twelve send checks.

### Results

The headline. As of November eleventh, twenty twenty-six, it ranks first on the public L1, L2 and quantization boards, with mean SOL scores of about zero point eight two three, zero point eight one nine, and zero point nine one oh.

But look at the margins. On L1 it leads second place by about zero point zero one. On L2 the lead is under zero point zero zero one. And on quantization, first and second are six millionths apart.

Every one of the two hundred and thirty-five problems has a kernel the site marks correct. On the two hundred and nine public-board problems, the kernels are four point four times faster than the official baseline. That's a geometric mean; the median is three point seven eight times, using the site's own timing.

Problem by problem, it's first on eighty-nine of the two hundred and nine, and in the top ten on one hundred and ninety-three. So the board wins come from breadth.

And the speedups aren't driven by a few outliers. Most problems sit between two and ten times. Thirty-nine problems are at least ten times faster, and the maximum is one thousand and sixty-two times. Two problems still end up slower than the baseline.

These aren't library wrappers either. Of the two hundred and thirty-five kernels, two hundred and fourteen are CUDA C plus plus, with a median of about seven thousand lines. By a text match on the source, seventy-seven percent name the tcgen05 tensor-core instructions, and ninety-four percent contain inline PTX, very low-level GPU code.

Most of the gain came early. In the first fifteen percent of the run's time, each board got seventy-three to eighty-five percent of its total rise, in local scores.

One case shows the stepping-stone idea well. Quantization problem thirty is an NVFP4 decoder layer. At nine point seven one times, the controller had a child replace the attention with a Blackwell-native CuTe DSL version. The first try ran at just one point five two times.

The next child traced the slowdown to register spills and reached seven point one, still below the parent. A third child moved the new attention onto the run's best node at the time and hit thirteen point nine seven, a new best. That path eventually climbed to twenty-seven point three six times, all in local timing.

Now the most important scaling finding. Early on, every GPU lock was held and about five calls were waiting per GPU, yet the GPUs were only about thirteen percent busy. The lock covered the CPU build and a shell start of twelve to fifteen seconds.

One warm restart moved builds out of the lock and cut shell start to milliseconds. In matched hours before and after, evaluated candidates per hour rose three point six times. The authors note the restart bundled several changes, and the split between them wasn't measured.

### Comparison

Now the paper's own head-to-head. Same Claude child agents, same archive design, a twenty-four hour budget per arm, comparing the LLM controller against a coded search tree that picks nodes with a UCB-style score, roughly a node's speedup plus a bonus for nodes that have few children.

At equal wall time, the LLM controller won on thirty-five of thirty-eight problems, with a geometric-mean advantage of about fourteen percent in best speedup.

Here's the interesting part. At an equal number of attempts, the two are about even: the controller is better on fifteen and worse on twenty-three. It wins by making two point four times as many correct attempts in the same time, because the tree narrows its width to four, then two, then one, leaving slots idle, while the controller keeps dispatching full batches.

The other test was adding guidance. Making the optional reference docs and skills mandatory raised their use four point three times, but not the final speedups. Across two hundred and thirty-three paired problems, the geometric-mean ratio was zero point nine eight one. That agrees with the goal-only result.

My read is that the lesson to keep isn't the leaderboard. It's this: the rate of reliable measurements, not the number of agents, is what limited scale, though the authors say they haven't yet tested that by varying the number of agents. Plenty of multi-agent builders reach for more agents first. This data suggests the bottleneck can be the evaluation pipeline.

My read is also that the first-place finish deserves a discount. The leads on L2 and quantization are tiny, and the boards keep moving. The authors themselves record being overtaken by other teams, and retaking first by making more of their results public.

### Limitations

The authors are candid about limitations. The evidence comes from one benchmark, one kind of hardware, the B200, one model, and one cold-started final run.

Correctness is tested, not proved. They looked for formal verification tools, but none modeled Blackwell's asynchronous tensor-core operations. And robustness checks covered each run's new bests, not every candidate.

Reward hacks did show up in the search. On quantization problem thirty, kernels along part of the path timed several variants on first use and picked the fastest. The rules later banned that, and removing it cost six percent in local speed.

Also, people chose what to send and when, and placed compute by the standings. The three design tests each ran once, on pre-final versions of the harness or as a short screen, so their verdicts may not hold for the final system.

And the code and kernels are planned for release, not yet out, and the agent prompts won't be included.

### Future work

Next, the authors want cheaper verification, so more attempts can be measured reliably, and they plan to test the loop on other GPUs and benchmarks.

They also plan to drop these kernels into a serving framework and measure time to first token and throughput. They expect that gain to be smaller than the per-kernel speedups, because serving stacks already use tuned library kernels.

My read is that this paper is worth studying for anyone building agent systems. Memory outside the model, checks outside the model, and a stateless controller mean you can fix a running system without starting over.

My read is also that the bar to reproduce it is high. Over a thousand agents ran at once, and every candidate needs a full build and a timed run on a B200. Most teams can't easily match that scale.

That's today's Daily Frontier Papers. The paper is linked in the description.

