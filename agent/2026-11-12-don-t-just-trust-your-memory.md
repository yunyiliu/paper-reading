# Don't Just Trust Your Memory: Evidence-Linked Memory Reduces Memory-Induced Degradation in LLM Agents

**Agent的记忆也会帮倒忙**

- 论文 Paper: [arXiv 2611.14320](https://arxiv.org/abs/2611.14320)
- 单位 Affiliation: KAIST、DeepAuto.ai
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-12
- 视频 Videos: [YouTube 中文](https://youtu.be/v5PnqdIE2Hw) · [YouTube English](https://youtu.be/sv7pmJH36Ow) · [B站](https://www.bilibili.com/video/BV1Z3Wd61EgQ)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 KAIST 和 DeepAuto.ai 的论文，标题是，别只相信你的记忆。它研究的是，Agent 的记忆为什么有时候反而会帮倒忙。

作者做了一个叫 WikiTrace 的记忆系统，让 Agent 用一条经验之前，可以回头查一查它是从哪次经历里总结出来的。论文报告，它在五个基准的八个测试集上全部第一。

### 问题

先说背景。现在很多 Agent 都带外部记忆，把做过的任务总结成几条简短的经验，遇到新任务拿出来参考，不改模型参数，也能越做越好。

但记忆不总是有用。作者把一种情况叫做记忆导致的退化，就是一个任务，不带记忆本来能做对，带上记忆反而做错了。

所以作者把效果拆成两个指标。Gain，是记忆把做错的任务变成做对的比例。Regression，是把做对的变成做错的比例。只看平均准确率，分不出这两者。

他们挑出不带记忆三次都能做对、带了记忆却做错的任务，用大模型当裁判找原因。结果是，52% 的失败能追溯到 Agent 照着做的某条记忆，最常见的是这条记忆本身就是错的，占 20%。

举个例子。在家务模拟环境 ALFWorld 里，记忆里有条规则说，台灯类任务，只要物体放在亮着的台灯旁边就算完成。可这条规则来源的那次成功经历里，Agent 是开了灯，又把铅笔拿起来，任务才结束。照着错规则，Agent 把一个本来能做对的任务做错了。

错的经验从哪来。每次经历只有一个标签，成功或者失败，它不会告诉你那么多动作里，是哪一个导致了结果。总结经验只能去猜，猜错了，就是一条误导人的经验。

### 以前的方法

以前的方法，经验的形式有反思、技能、操作手册、工作流，还有推理策略。论文对比了三个代表，AWM、ExpeL 和 ReasoningBank。

AWM 把成功经历抽象成工作流，ExpeL 维护一份规则清单，ReasoningBank 把成功和失败都提炼成推理策略，按任务相似度检索。

作者认为，它们共同的问题是，只给经验，不给出处。能识破错误经验的证据，往往还留在原来的经历里，可 Agent 没办法回头核对。

### 核心想法

WikiTrace 的想法很简单，经验和证据不要分开。经验简洁，告诉 Agent 该做什么。原始经历记录了实际发生的事，但太长，不可能每个决定都从头读。所以把两者连起来，平时看经验，有疑问再顺着链接往下查。

这张图是整体结构。记忆分三层，最上面是 insight，一条简短的经验。中间是 compilation，可以理解成一份诊断，总结那次为什么成功或失败。最底层是原始轨迹，每一步的动作和观察。三层之间都有链接。

### 方法

建记忆时，一个叫 compiler 的模型读一次过去的经历，包括任务、轨迹和成败标签，写出一份 compilation 和最多三条 insight。

每条 insight 是一个单独的文档，写明该做什么、什么时候适用、来自哪次经历、那次成没成功。insight 写好就不再修改，所以经验之间可能重复，甚至矛盾，该信哪条，交给 Agent 判断。

原始轨迹原样保存，每一步的推理、动作和观察都在，每条 insight 同时链接到它的 compilation 和原始轨迹。

用的时候，Agent 一开始就拿到全部 insight，外加三个和当前任务最相似的过去经历，作为查找的起点。

提示词要求 Agent 在三种情况下往下查，经验缺少执行需要的细节，经验来自失败的经历，或者和另一条经验矛盾。每当一个动作失败，也要再查一次。作者把这叫做 descent，往下钻。

查的方式很朴素。记忆以只读文件挂给 Agent，它用 read 和 bash 这些终端命令去搜，找能拍板的那个事实，比如一条命令的具体格式。

### 实验和效果

实验覆盖五个基准、八个测试集。三个是文字版模拟世界，做家务的 ALFWorld，做科学实验的 ScienceWorld，Minecraft 合成的 PlanCraft。两个是工具调用环境，AppWorld 和 Tau3-bench 的零售场景。

所有方法用同一个 Agent，底层模型是 DeepSeek-V4-Flash，都从同一批不带记忆的训练经历建记忆，不许额外跑任务来验证记忆。每组跑三次取平均。

最关键的对照组叫 Insights Only。它拿到的 insight 和 WikiTrace 完全一样，只是没有 compilation 和原始轨迹，也没有往下查的提示。两者唯一的区别，就是能不能回到证据。

主结果在表 1。八个测试集平均，不带记忆是 79.5，Insights Only 是 87.0，WikiTrace 是 90.2，比不带记忆高 10.7 分，比 Insights Only 高 3.2 分。

而且，在所有带记忆的方法里，只有 WikiTrace 在每个测试集上都不低于不带记忆。比如 PlanCraft 一个测试集上，不带记忆是 56.3，ExpeL 掉到了 51.0。

再看这张图。WikiTrace 的 Regression 是 4.5%，Insights Only 是 6.5%。作为参照，不带记忆重跑一遍，因为模型本身的随机性，也有 9.7% 做对的任务会做错。

Gain 这边，WikiTrace 比重跑基线高 35 个百分点，Insights Only 高 27 个，以前的记忆方法只高 7 到 14 个。作者把 WikiTrace 和 Insights Only 的差距，归功于能回查原始经历。

那什么时候会往下查。在三个文字世界里，任务越难查得越多，简单任务 22% 的运行会查，难任务是 44%。查了的运行比没查的，领先 Insights Only 更多，任务越难，差距越大。不过作者提醒，查不查是 Agent 自己选的，两组运行不只差在这一点。

为什么查。大模型裁判读 Agent 的推理打标签。出了问题之后的查找，比如尝试被卡住，或者怀疑某条经验，占 36%，领先 Insights Only 10.4 分。没出问题时的查找，只领先 3.7 分。

查完有什么用。往下查过的运行里，43% 改变了原来的计划，比如修正一条经验，或者补上漏掉的细节。这些运行领先 Insights Only 8.3 分，计划没变的只领先 4.4 分。

这张图是一个具体例子。在 PlanCraft 里，任务是合成一种木台阶。记忆里有条来自失败经历的经验说，如果只放一个材料就已经合成出了东西，配方可能做不了，考虑放弃。

其实放一块木板后，输出栏显示的只是预览。Insights Only 照着错经验，拿走预览，耗掉一块木板，最后宣布做不了。WikiTrace 发现这条经验和一条来自成功经历的经验矛盾，就把两边的原始经历都翻出来，看到成功的那次是一块一块加木板，直到台阶出现，于是修正经验，完成了任务。

失败经历值得存吗。在 PlanCraft 和 ScienceWorld 上，只用失败经历比只用成功经历效果好，两者都用最好。但只给经验时，失败经历的退化风险最高，Insights Only 的 Regression 到了 9.3% 和 9.4%，WikiTrace 降到 7.1% 和 8.9%。

### 和以前方法的对比

WikiTrace 是不是赢在建记忆花得多。作者加大了对手的预算，AWM 建九遍，ExpeL 和 ReasoningBank 各建三遍，平均分在 82.8 到 83.8 之间，WikiTrace 仍然最高。

测试时，WikiTrace 每个任务的成本平均是 Insights Only 的 1.17 倍，换来 3.2 分的提升。

换成 GLM-5.3-flash 做底层模型，在三个测试集上，WikiTrace 平均 92.7，比 Insights Only 高 1.8 分，比不带记忆高 21.5 分。而以前的每种记忆方法，都至少在一个测试集上低于不带记忆。

我的判断是，这篇最有价值的，是把记忆帮倒忙这件事量化了出来。给 Agent 加记忆的团队，都应该单独看 Regression，而不是只看平均分。

我的判断是，也要看清提升来自哪里。比不带记忆高出的 10.7 分里，有 7.5 分 Insights Only 也能拿到，靠的是作者这套逐次提炼经验的做法。真正归功于回查证据的，是剩下的 3.2 分。

### 局限

作者列了几条局限。第一，记忆来自只有成败标签的经历，compilation 对失败原因的诊断可能是错的。链接让 Agent 能对照原始经历检查经验，但不等于验证了它。

第二，所有 insight 都放进第一条消息，提示词会随记忆变大而变长。记忆大到放不下时，需要 Agent 自己找经验，这个检索问题论文没有解决。

第三，退化原因和查找原因，都是大模型裁判读推理过程标出来的，反映的是推理里怎么说，不一定是结果的真正原因。

另外从实验设置看，每个基准的训练集只有 38 到 54 个任务，记忆规模很小，换模型的实验也只做了三个测试集。

### 未来方向

作者明确留下的问题，是记忆变大之后怎么检索，让 Agent 在没法把经验全放进提示词时，也能找到需要的那几条。

论文的结论是，Agent 的记忆不仅要存经验，还要保留能重新评估经验的证据。代码已经在 GitHub 公开。

我的判断是，这个思路很容易用到实际的 Agent 里。不用训练模型，只要在经验旁边留一条指向原始记录的链接，代价是测试时多读一些文件。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from KAIST and DeepAuto.ai, and its title is a warning: don't just trust your memory. It's about why an agent's memory sometimes makes it worse.

The authors built a memory system called WikiTrace, which lets an agent check where a lesson came from before acting on it. According to the paper, it comes out on top on all eight test suites across five benchmarks.

### Problem

Some background. Many agents now carry an external memory: past tasks are boiled down into short lessons, which the agent gets on new tasks, so it improves from experience without any change to its weights.

But memory doesn't always help. The authors call one failure mode memory-induced degradation: the agent fails a task with memory that it would have solved without it.

So they split the effect into two numbers. Gain is how often memory turns a failure into a success. Regression is how often it turns a success into a failure. Average accuracy can't tell them apart.

They took tasks the agent solved in all three runs without memory but failed with it, and had a language model judge find the cause. In fifty-two percent of cases, the failure traced back to a memory item the agent followed, and most often, about twenty percent, the item itself was wrong.

Here's an example. In ALFWorld, a simulated household, memory held a rule that a lamp task is done once the object sits next to the lit lamp. But in the successful run behind that rule, the agent turned on the lamp and then picked up the pencil before the episode ended. Following the rule, the agent failed a task it could solve on its own.

Where do wrong lessons come from? A trajectory has one label, success or failure, which doesn't say which of its many actions caused the outcome. Anyone distilling a lesson has to guess, and a wrong guess becomes a misleading lesson.

### Prior work

Earlier methods store lessons as reflections, skills, manuals, workflows or reasoning strategies. The paper compares against three of them: AWM, ExpeL and ReasoningBank.

AWM turns successful runs into workflows, ExpeL keeps a list of rules, and ReasoningBank distills successes and failures into reasoning strategies retrieved by task similarity.

The authors' point is that they all hand over the lesson without its source. The evidence that could expose a bad lesson is often still in the original trajectory, and the agent has no way to look.

### Key idea

The idea behind WikiTrace is to keep lesson and evidence together. A lesson is short and says what to do. A trajectory records what really happened, but it's too long to reread for every decision. So you link them: read the lesson by default, and follow the link down when something seems off.

This figure shows the structure. Memory has three layers. On top is the insight, one short lesson. In the middle is the compilation, a diagnosis of why that run succeeded or failed. At the bottom is the raw trajectory, every action and observation. All three are linked.

### Method

To build the memory, a model called the compiler reads one past run, its task, trajectory and outcome label, and writes a compilation plus at most three insights.

Each insight is its own document: what to do, when it applies, which run it came from, and whether that run succeeded. Insights are never edited once written, so the pool can hold duplicates and contradictions, and the agent decides which one applies.

The raw trajectory is kept unchanged, with every turn's reasoning, action and observation, and each insight links to both its compilation and its trajectory.

On a new task, the agent starts with every insight, plus the three past runs most similar to the current task as starting points for a search.

The prompt tells it to go down to the source run when an insight lacks the details needed to act, comes from a failed run, or disagrees with another insight. It also asks for another look whenever an action fails. The authors call this a descent.

The mechanics are plain. The memory is mounted as read-only files, and the agent searches them with terminal tools like read and bash for the one fact that settles the question, such as a command's exact format.

### Results

The evaluation spans eight suites across five benchmarks. Three are text worlds: ALFWorld for household chores, ScienceWorld for science experiments, and PlanCraft for Minecraft crafting. Two are tool-use environments: AppWorld, and the retail domain of tau three bench.

Every method uses the same agent on DeepSeek-V4-Flash, builds its memory from the same no-memory training runs, and may not run extra tasks to validate it. Every setting is run three times and averaged.

The key comparison is called Insights Only. It gets exactly the same insights as WikiTrace, but no compilations, no raw trajectories, and no prompt to descend. The only difference is access to the evidence.

The main results are in Table 1. Averaged over eight suites, no memory scores seventy-nine point five, Insights Only eighty-seven point zero, and WikiTrace ninety point two: ten point seven points above no memory and three point two above Insights Only.

And of all the memory methods, WikiTrace is the only one that matches or beats no memory on every suite. On one PlanCraft suite, for instance, no memory scores fifty-six point three and ExpeL drops to fifty-one point zero.

Now this figure. WikiTrace's Regression is four point five percent, against six point five for Insights Only. For reference, simply rerunning the agent without memory flips nine point seven percent of solved tasks to failures, because the model itself is noisy.

On Gain, WikiTrace beats that rerun baseline by thirty-five percentage points, Insights Only by twenty-seven, and earlier memory methods by only seven to fourteen. The authors credit the gap between WikiTrace and Insights Only to access to source runs.

When does the agent descend? In the three text worlds, harder tasks get more descents: twenty-two percent of runs on easy tasks, forty-four percent on hard ones. Runs that descend lead Insights Only by more than runs that don't, and the difference grows with difficulty. The caveat, which the authors make themselves, is that the agent chooses when to descend, so the two groups differ in more than that.

Why does it descend? A language model judge labeled the agent's reasoning. Descents after a problem, like a blocked attempt or a doubt about an insight, are thirty-six percent of the total and lead Insights Only by ten point four points. Descents with nothing gone wrong lead by just three point seven.

What does the evidence change? In forty-three percent of descending runs, it changes the plan, for example by revising an insight or adding a missing detail. Those runs lead Insights Only by eight point three points, against four point four when the plan stays the same.

This figure walks through one case. In PlanCraft, the task is to craft a wooden slab. A lesson from a failed run says that if placing a single item already crafts something, the recipe may be impossible, so consider giving up.

In fact, what shows in the output slot after one plank is only a preview. Insights Only follows the bad lesson, takes the preview, burns a plank, and declares the task impossible. WikiTrace notices the lesson conflicts with one from a successful run, opens both source runs, sees the successful run keep adding planks until the slab appears, revises the lesson, and finishes the task.

Are failed runs worth keeping? On PlanCraft and ScienceWorld, memory from failures alone beats memory from successes alone, and both together is best. But with lessons alone, failures carry the most regression risk: Insights Only reaches nine point three and nine point four percent, and WikiTrace brings that to seven point one and eight point nine.

### Comparison

Is WikiTrace just spending more on building memory? The authors gave the rivals bigger budgets, building AWM nine times and ExpeL and ReasoningBank three times each. They landed between eighty-two point eight and eighty-three point eight, and WikiTrace still had the best average.

At test time, WikiTrace costs one point one seven times as much per task as Insights Only, for an average gain of three point two points.

With GLM-5.3-flash as the backbone, on three suites, WikiTrace averages ninety-two point seven, one point eight above Insights Only and twenty-one point five above no memory. Every earlier memory method falls below no memory on at least one of those suites.

My read is that the most useful thing here is putting a number on memory hurting the agent. Anyone adding memory to an agent should track Regression separately, not just the average.

My read is also that it's worth being clear about where the gains come from. Of the ten point seven points over no memory, seven point five are already there with Insights Only, the authors' own per-run lesson extraction. The part credited to checking the evidence is the remaining three point two.

### Limitations

The authors list several limitations. First, memory comes from runs labeled only success or failure, so a compilation's diagnosis of a failure can be wrong. A link lets the agent check a lesson against its run, but that isn't verifying it.

Second, every insight goes into the first message, so the prompt grows with the memory. A memory too big to fit would need the agent to find its own lessons, a retrieval problem the paper leaves open.

Third, the causes of degradation and the reasons for descent are labeled by a language model reading the agent's reasoning. They describe what that reasoning says, not necessarily what caused the outcome.

The setup is also small: each benchmark has only thirty-eight to fifty-four training tasks, so the memories are small, and the second backbone was tested on just three suites.

### Future work

Looking ahead, the open problem the authors name is retrieval: how an agent finds the few lessons it needs once memory is too big for the prompt.

Their conclusion is that agent memory should keep not only lessons but the evidence needed to reassess them. The code is already on GitHub.

My read is that this transfers easily to real agents. It needs no training, just a link from each lesson back to the original record, at the cost of reading a few more files at test time.

That's today's Daily Frontier Papers. The paper is linked in the description.

