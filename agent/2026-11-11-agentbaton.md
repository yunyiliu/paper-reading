# AgentBaton: Efficient Model Collaboration in Agentic Tasks

**强模型开局便宜模型收尾**

- 论文 Paper: [arXiv 2610.35959](https://arxiv.org/abs/2610.35959)
- 单位 Affiliation: NVIDIA
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-11
- 视频 Videos: [YouTube 中文](https://youtu.be/OQsSvD28AjQ) · [YouTube English](https://youtu.be/cTkQs8g9C_w) · [B站](https://www.bilibili.com/video/BV17gWy6mEHG)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 NVIDIA 的论文，叫 AgentBaton，Baton 就是接力棒。

它研究的问题很实际，一个很长的 Agent 任务，能不能让又贵又强的模型和便宜的模型接力完成，钱省下来，正确率不掉。

### 问题

先说背景。Agent 做长任务，是一轮一轮来的，调用工具，看反馈，再改。

最可靠的结果，往往来自云上又强又贵的模型，每一轮都调用，成本很高。便宜的模型也越来越多，有低价的云端模型，也有能在本地跑的开源小模型，但单独干，正确率明显低。

### 以前的方法

以前把不同模型组合起来，主要是按请求来路由，或者做级联，也就是看一眼题目，决定这道题交给哪个模型。

问题在于，Agent 每做一个动作，都会改变工作环境，需要什么能力是一轮一轮在变的，开头看一眼题目，没法提前定好。

也有更细的做法，每一轮都重新选模型，比如论文拿来对比的 MTRouter。也有工作研究过在任务中途换模型，但用的是事先定好的切换点。

### 核心想法

这篇论文先不急着提方法，而是做了一组系统的实验，研究两件事，交接的方向，还有交接时怎么把前面的历史传过去。

方向有两种。一种是强模型先做出关键的中间结果，再交给便宜模型。另一种反过来，便宜模型先收集信息，再交给强模型攻坚。两种听起来都说得通。

历史怎么传，也有两种，这张图画的就是。一种是把前面的完整对话，直接放进接手模型的上下文。另一种是写成日志文件，接手的模型自己用 cat、grep 这些命令去查。两种情况下，前面改过的文件都留在同一个工作区里。

实验结果是一个明显的不对称。便宜模型先干、强模型接手，正确率通常比不上强模型从头干到尾，有时候还更贵。强模型先干、便宜模型在合适的阶段接手，正确率基本保得住，成本却大幅下降。

所以作者的结论是，只交接一次，从强到弱，挑对时机，就够了。AgentBaton 就是一个专门判断时机的小模型。

### 方法

先看实验怎么做的。基准有两个，Terminal-Bench 2.1，是各种终端里的工作流程，还有 DeepSWE v1.1，是在活跃开源仓库里的长流程软件工程任务。

分析实验里，强模型主要用 GPT-5.6 Sol、Kimi K3 和 GLM-5.3，推理强度设为 max。便宜模型用 Qwen3.8-27B，编程和推理能力不错，又能在本地工作站上部署。Agent 框架是简洁的 mini-SWE-agent。

这里要注意一个记账口径。云端模型按官方 token 价格算钱，能在本地部署的模型，成本记为零。后面说的省钱比例，都是按这个口径算的 API 成本。

第一组实验，作者先让每个模型单独跑一遍，再把工作区恢复到它跑到 30% 和 75% 的位置，换另一个模型接着干。

结果在表 1。便宜模型交给强模型时，日志文件的方式更好，正确率一般不低，多数情况还更省钱。反方向，GLM 和 Kimi 交给 Qwen，直接放上下文多数更好，GPT-5.6 Sol 有好有坏，作者推测是因为 Sol 不公开完整的思维链。

第二组实验，把交接点扩展到 15% 到 90% 的六个位置，看 DeepSWE 上正确率和成本怎么变。

便宜模型先干的方向，交给 GPT-5.6 Sol，在 30% 处交接，正确率 70.8%，比 Sol 单独的 68.1% 还略高，但成本只省了 6.1%。交给 GLM 和 Kimi，越晚交接越省钱，但正确率也跟着掉。

反方向就好得多。Kimi K3 在 45% 处交给 Qwen，正确率 72.57%，接近 Kimi 单独跑的 73.45%，成本降了 61.5%。GLM-5.3 在 30% 处交接，正确率 61.1%，和它单独跑持平，成本降了 82.4%。

那什么时候交接合适。作者找了这样的轨迹，Qwen 在早一点接手会失败，晚一步接手就成功，然后看强模型在这中间做了什么。

比如一个写压缩器的任务。GLM 跑到 30% 时，编码器能输出文件，但解压会崩溃，这时交给 Qwen，失败了。GLM 继续往下，把问题缩小到符号位和算术编码，编码器其实还是错的，这时交给 Qwen，Qwen 重写编码器，成功了。

作者总结出三个信号。第一，已经有了实质性的实现或者产物。第二，真的运行、测试或检查过，拿到了和任务相关的反馈，失败也算，只要它把问题缩小了。第三，剩下的工作已经很局部，主要是排查、修补、验证，或者准备最终输出。

能不能让强模型自己决定什么时候交接。作者用提示词试了，结果在表 2，Kimi K3 一次都没交，GLM-5.3 在 113 个任务里只交了 6 次。GPT-5.6 Sol 每个任务都交了，但平均成本还有 3.75 美元，单独跑是 5.32 美元。

所以作者训练了一个专门的判断模型。数据来自 SWE-rebench V2 里随机抽的 2000 个任务，每个任务随机挑两个强模型各跑一遍，得到 4000 条轨迹。

标注交给 Codex，按刚才那三个信号，回头看整条轨迹，找出最早适合交接的那一轮。找不到明确交接点的，单独标成一类。

判断模型本身是 Qwen 3.5 2B，很小。每一轮它只输出一个判断，还没准备好，或者可以交接了。标注点之后的轮次也算可以交接，这样它就算错过时机，后面也还会触发。

实际运行时，强模型先开始，每走一轮，AgentBaton 看一遍更新后的轨迹。一旦判断可以交接，就交给便宜模型，继承工作区，一直做到结束，不再换回来。论文报告，在一张 H100 上，判断一次的延迟中位数是 338 毫秒。

### 实验和效果

正式实验又加了两个强模型，Claude Opus 5 和 Gemini 3.8 Flash，和一个便宜模型 GPT-5.6 Luna，一共 11 组模型搭配，两个基准。

结果在表 3。论文报告，AgentBaton 平均把 API 成本降低了 65.9%。在 22 个模型搭配和基准的组合里，有 12 个正确率提升，或者相对下降不到 5%。

举个例子，在 DeepSWE 上，Kimi K3 交给 Qwen，正确率 72.6%，Kimi 单独跑是 73.5%，成本降了 72.9%。

接手的模型强一点，效果更好。换成 GPT-5.6 Luna 接手，DeepSWE 上五个强模型里有四个，正确率反而超过了自己单独跑。比如 Claude Opus 5 交给 Luna，正确率从 67.3% 升到 76.1%，成本降了 70.5%。

最后一组实验是调推理强度。图 1 把各模型不同推理强度下单独跑的结果，和 AgentBaton 的组合画在一起，横轴是成本，纵轴是正确率。

论文报告，Kimi K3 用 high 推理强度，再交给 Luna，正确率 79.6%，每道题 1.6 美元。这比这组实验里最好的单个模型还高，那是 Opus 5 开 xhigh，正确率 75.2%，每道题 8.9 美元。

### 和以前方法的对比

先和按题目路由比。作者训练了一个小模型，看题目判断 Qwen 能不能独立做出来。它在训练集上准确率 99.55%，实际却只把一道题分给了 Qwen，等于全程用强模型，几乎省不了钱。

再和每轮路由的 MTRouter 比，模型搭配相同。论文报告，四组设置里，AgentBaton 正确率持平或更高，成本比 MTRouter 低 46.7% 到 86.6%，数字在表 4。

还有两个简单对照。按固定轮数交接，正确率一直更低。按固定进度比例交接，等于用了未来的信息，因为要事先知道强模型总共走多少轮，它的正确率和 AgentBaton 接近，在 DeepSWE 上略高，但 AgentBaton 成本相同或更低。

我的判断是，这篇最有用的不是那个 2B 的小模型，而是那条不对称的结论，强模型开局，便宜模型收尾，比反过来靠谱得多。对搭 Agent 系统的人，这是一个很容易落地的分工原则。

### 局限

论文没有单独写局限这一章，但从它自己的数据里，能看到几点。

第一是成本口径。本地模型记为零成本，所以交给 Qwen 的那些组合，省下的比例没有算本地硬件和电费。

第二，代价不是零。22 个组合里，有 10 个正确率相对下降 5% 或以上。最大的是 DeepSWE 上 GPT-5.6 Sol 交给 Qwen，正确率从 68.1% 掉到 60.2%，相对下降 11.6%。

第三，在 Terminal-Bench 上，交接的效果没有 DeepSWE 稳定，论文附录也说，不同模型、不同交接点之间，结果的一致性更差。

我的判断是，还有两点要注意。评测只有编程和终端任务，判断模型也是用软件工程任务训练的，换到别的 Agent 场景，论文没有测。另外 DeepSWE 一共 113 道题，差一道题就接近一个百分点，零点几个点的差距，不宜过度解读。

### 未来方向

论文的结论是，任务进度本身，可以作为动态分配算力的信号，调推理强度和模型交接，是两种可以互补的手段。

我的判断是，接下来值得看两件事。一是把本地模型的真实成本算进去，账还划不划算。二是交接效果依赖前一个模型留下多少上下文，像 Sol 这样不公开完整思维链的闭源模型，能交出去的信息有限。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from NVIDIA, and it's called AgentBaton, as in a relay baton.

The question is a very practical one. On a long agent task, can an expensive, capable model and a cheap one run it as a relay, saving money without losing accuracy?

### Problem

Some background first. Agents work through long tasks one turn at a time: call a tool, read the feedback, revise.

The most reliable results tend to come from capable cloud models, and calling one of those on every single turn adds up fast. Meanwhile, cheaper options keep appearing, from lower-tier cloud models to small open-weight models you can run on local hardware. On their own, though, they solve noticeably fewer tasks.

### Prior work

Most earlier work on mixing models routes or cascades at the level of the request. You look at the query once and decide which model gets the whole thing.

The trouble is that in agent tasks, every action changes the workspace. What the task demands shifts from turn to turn, so a single look at the initial query can't settle it.

There are finer-grained approaches that pick a model every turn, like MTRouter, which this paper compares against. And there's work on switching models mid-task, but with switch points fixed in advance.

### Key idea

Before proposing anything, the authors run a systematic study of two questions: which direction the handoff should go, and how to pass the history along.

Two directions are possible. The capable model could start, producing a plan or key intermediate results, and then hand over. Or the cheap model could do the early legwork, like gathering information, before the capable one takes on the hard part. Both sound reasonable.

There are also two ways to pass the history, shown in this figure. Either you put the full prior conversation into the new model's context, or you write it to a log file that the new model can inspect with commands like cat and grep. Either way, the receiving model inherits the same workspace, with every file change already in place.

What they find is a clear asymmetry. Cheap first, capable second usually does worse than running the capable model alone, and sometimes costs more. Capable first, with the cheap model taking over at the right stage, largely keeps the accuracy while cutting cost substantially.

So the conclusion is that a single, well-timed handoff from strong to cheap is good enough. AgentBaton is a small model whose only job is to call that timing.

### Method

First, the setup. There are two benchmarks: Terminal-Bench 2.1, which covers a range of command-line workflows, and DeepSWE v1.1, which has long software engineering tasks in active open-source repositories.

For the analysis, the capable models are mainly GPT-5.6 Sol, Kimi K3 and GLM-5.3, at max reasoning effort. The cheap model is Qwen3.8-27B, chosen because it codes and reasons well and is small enough for a local workstation. The agent scaffold is the minimal mini-SWE-agent.

One accounting detail matters here. Cloud models are billed at official token prices, and locally deployable models are counted as zero cost. Every saving you'll hear is in API cost under that convention.

In the first experiment, each model runs on its own. Then the workspace is restored to the point thirty percent or seventy-five percent of the way through, and the other model takes over.

Table 1 has the results. When the cheap model hands off to a capable one, the log file generally matches or beats putting history in context, and is clearly cheaper in most cases. In the other direction, GLM and Kimi handing to Qwen mostly do better with history in context, while GPT-5.6 Sol is mixed. The authors suspect that's because Sol doesn't expose its full chain of thought, so there's less reasoning to pass along in context.

The second experiment extends the handoff point to six positions, from fifteen to ninety percent, and tracks accuracy and cost on DeepSWE.

Going cheap first and handing to GPT-5.6 Sol at thirty percent gets seventy point eight percent, slightly above Sol's sixty-eight point one on its own, but the cost falls by only six point one percent. Handing to GLM or Kimi, later handoffs save more money but lose accuracy along the way.

The reverse direction looks much better. Kimi K3 handing to Qwen at forty-five percent reaches seventy-two point five seven percent, close to Kimi's own seventy-three point four five, at sixty-one point five percent lower cost. GLM-5.3 handing off at thirty percent gets sixty-one point one percent, matching its solo score, at eighty-two point four percent lower cost.

So when is the right moment? The authors pulled out trajectories where Qwen fails after one handoff point but succeeds after the next, and looked at what the capable model did in between.

Take a task to write a compressor. At thirty percent, GLM's encoder produces a compressed file, but decompression crashes, and Qwen taking over from there fails. GLM keeps going and narrows the fault to sign-bit handling and arithmetic coding, though the encoder is still wrong. Hand over at that point, and Qwen rewrites the encoder and passes.

The authors draw out three signals. First, a substantive implementation or artifact exists. Second, it has actually been run, tested or inspected, with task-relevant feedback, and a failure counts if it narrows the problem down. Third, what's left is local work: diagnosis, repair, validation, or preparing the final output.

Could the capable model just decide for itself when to hand off? The authors tried prompting it, and Table 2 shows how that went. Kimi K3 never handed off, and GLM-5.3 did so on only six of one hundred thirteen tasks. GPT-5.6 Sol handed off every time, but still averaged three dollars seventy-five per task, against five dollars thirty-two running alone.

So they trained a dedicated detector. The data comes from two thousand randomly sampled SWE-rebench V2 tasks, each attempted separately by two of five capable models, for four thousand trajectories.

Codex labeled them, using the three signals to look back over each trajectory and find the earliest turn suitable for a handoff. Trajectories with no clear handoff point got a separate label.

The detector itself is Qwen 3.5 2B, a small model. Each turn it outputs one verdict: not ready yet, or ready to hand off. Training also marks the turns after the labeled point as ready, which pushes it to still trigger later if it missed the ideal moment.

At run time, the capable model starts, and after every turn AgentBaton reads the updated trajectory. As soon as it says ready, control passes to the cheap model, which inherits the workspace and runs to the end, with no switching back. The paper reports a median latency of three hundred thirty-eight milliseconds per decision on one H100.

### Results

The main experiments add two more capable models, Claude Opus 5 and Gemini 3.8 Flash, and a second cheap model, GPT-5.6 Luna. That's eleven model pairs across two benchmarks.

Table 3 has the results. The paper reports an average API cost reduction of sixty-five point nine percent. Of the twenty-two combinations of model pair and benchmark, twelve either improve accuracy or lose less than five percent of it in relative terms.

One example: on DeepSWE, Kimi K3 handing to Qwen scores seventy-two point six percent, against seventy-three point five for Kimi alone, at seventy-two point nine percent lower cost.

A stronger receiving model helps. With GPT-5.6 Luna taking over, four of the five capable models on DeepSWE actually beat their own solo accuracy. Claude Opus 5 handing to Luna, for instance, goes from sixty-seven point three to seventy-six point one percent, at seventy point five percent lower cost.

The last experiment varies reasoning effort. Figure 1 puts every model's solo results at different effort levels next to the AgentBaton combinations, with average cost per task on the horizontal axis and accuracy on the vertical.

The paper reports that Kimi K3 at high effort, handing to Luna, reaches seventy-nine point six percent at a dollar sixty per task. That beats the best single model in the sweep, Opus 5 at xhigh effort, with seventy-five point two percent at eight dollars ninety.

### Comparison

First, against routing by the query. The authors trained a small router to predict from the task description whether Qwen could solve it alone. It hit ninety-nine point five five percent accuracy on its training set, yet sent just one task to Qwen, so it effectively ran the capable model throughout and saved almost nothing.

Then against per-turn routing with MTRouter, using the same model pairs. The paper reports that across four settings, AgentBaton matches or beats its accuracy at forty-six point seven to eighty-six point six percent lower cost, with the numbers laid out in Table 4.

There are two simple baselines too. Handing off after a fixed number of turns is consistently less accurate than AgentBaton. Handing off at a fixed fraction of progress actually uses future information, since you'd need to know in advance how long the capable model's run will be. Its accuracy is close to AgentBaton's, a bit higher on DeepSWE, but AgentBaton's cost is equal or lower.

My read is that the most useful thing here isn't the 2B detector. It's the asymmetry: let the strong model open, let the cheap model close, and that works far better than the reverse. For anyone building agent systems, it's a division of labor you can put into practice quickly.

### Limitations

The paper has no dedicated limitations section, but its own numbers show a few.

First, the cost accounting. Local models count as free, so the savings for the Qwen pairs leave out local hardware and electricity.

Second, the trade isn't free. Ten of the twenty-two combinations lose five percent or more of their accuracy in relative terms. The worst is GPT-5.6 Sol handing to Qwen on DeepSWE, which drops from sixty-eight point one to sixty point two percent, a relative loss of eleven point six percent.

Third, on Terminal-Bench the handoffs are less stable than on DeepSWE. The appendix says results there are less consistent across models and handoff points.

My read is that two more things deserve caution. The evaluation covers only coding and terminal tasks, and the detector was trained on software engineering tasks, so whether this holds for other kinds of agents is untested. And DeepSWE has one hundred thirteen tasks, so a single task is worth close to one percentage point. Gaps of a fraction of a point shouldn't be read into too much.

### Future work

The paper concludes that task progress itself is a useful signal for allocating compute, and that reasoning effort and model handoff are complementary ways to do it.

My read is that two things are worth watching. One is whether the economics still hold once the real cost of running the local model is counted. The other is that a handoff depends on how much context the first model leaves behind, and closed models like Sol that don't expose their full chain of thought have less to pass on.

That's today's Daily Frontier Papers. The paper is linked in the description.

