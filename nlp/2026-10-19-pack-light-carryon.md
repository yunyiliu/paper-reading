# Pack Light, CarryOn: Building Object-Oriented Context for Long-Horizon Agents

**Agent别再反复读历史**

- 论文 Paper: [arXiv 2610.18784](https://arxiv.org/abs/2610.18784)
- 单位 Affiliation: 加州大学圣地亚哥分校
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-19
- 视频 Videos: [YouTube 中文](https://youtu.be/s0fh4hiE0Pg) · [YouTube English](https://youtu.be/r4pLa0oz4jQ) · [B站](https://www.bilibili.com/video/BV1PAJd67E9F)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇加州大学圣地亚哥分校的论文，名字叫 CarryOn，标题前半句是 Pack Light，意思是轻装上阵。

它关心一个很实际的问题，Agent 一个任务要做几百步，上下文越堆越长，怎么给它减负，同时又不丢掉做到一半的进度。

### 问题

先说背景。长任务 Agent 的进度，其实存在两个地方。一个是任务状态本身，比如已经插进幻灯片的图片，已经填进报告的数据。另一个是交互历史，记录的是这些进度是怎么一步步做出来的。

问题在于，结果已经在文件里了，Agent 每走一步，还要把怎么做出来的那段历史再读一遍。任务越长，这种重复就越多。

### 以前的方法

以前的做法主要有两类。一类是压缩历史，比如做摘要，做检索，或者删掉旧的交互。另一类是把历史改写成显式的进度记录，或者结构化的任务状态。

作者认为，这两类方法确实控制住了上下文的增长，但它们让 Agent 依赖一份二手的，挑选过的进度描述，这份描述可能和 Agent 实际能看到，能修改的状态慢慢对不上。

### 核心想法

CarryOn 的想法是换一个组织单位。不围绕历史来组织上下文，而是围绕可以复用的任务对象，比如做好的文件，查到的结论，应用里已经设好的状态。作者把这叫做面向对象的上下文。

具体来说，上下文被分成三块。第一块叫 continuation carrier，可以理解成进度载体，记录一个可复用的东西在哪，做到了哪一步，在什么条件下能接着用。第二块叫执行上下文，保存还没完成的目标，要遵守的约束，和没解决的问题。第三块叫 live tail，就是上次整理之后，新发生的那几步原始交互。

论文第一张图就是例子。一个改幻灯片的任务，整理之后，两张导出的图片和正在编辑的 PPT 变成了三个载体，执行上下文里只留下保持 40% 透明度，别动其他内容，确认保存这几件事，上下文从 4.24 万 token 降到 1.44 万，少了 66%。

### 方法

这个整理动作，论文叫 fold，也就是折叠。CarryOn 跑在 Agent 旁边，每次被触发，分四步完成一次折叠。这张图是整体流程。

第一步，跟踪候选。一个叫 Tracker 的模块读最近的交互，找出可能变成可复用进度的东西，可能是一个新载体，也可能是对已有载体的更新。还没成形的候选会先留着，等以后再处理。

第二步，准备证据。每个候选都要绑定支持它的证据。如果是查到的信息，就检查内容和来源对不对得上。如果是文件或者应用状态，一个叫 Grounder 的模块会做只读检查，比如文件在不在，和初始状态比有没有变化，文档结构是不是完好。

第三步，联合编译。一个叫 Compiler 的模块同时更新载体和执行上下文。两者是配合的，已经由载体承载的进度，不用在执行上下文里重复，但没完成的目标和约束，必须留下来。受影响的载体生成新版本，没受影响的原样保留。

第四步，校验和提交。系统检查这些修改是不是基于最新版本，是不是和授权的证据一致，而且至少要改动一个载体。通过了，就一次性提交，把被覆盖的历史从上下文里撤掉。没通过，就什么都不动。

要注意两点。作者明确说，这些检查只保证运行时内部的一致性，并不验证任务本身做对没有。另外，原始交互并没有删掉，还留在一份只追加的日志里。

实现上，Agent，Tracker 和 Compiler 用的都是 Claude Opus 4.7，没有额外训练。在 OSWorld 2.0 上，通常每积累大约 6000 token 的新交互，就触发一次 Tracker。

### 实验和效果

实验用了两个基准。OSWorld 2.0 是长流程的电脑操作任务，基准估计人完成一个任务的中位时间大约 1.6 小时，论文用了完整评测集，每个任务最多 500 步。BrowseComp-Plus 是需要在多个来源之间反复搜索的问答，论文随机抽了 100 道题。

主结果在表 1。在 OSWorld 2.0 上，同样是 Claude Opus 4.7，加上 CarryOn 之后，部分得分从 43.9% 提到 49.7%，完全做对才算分的二值得分，从 15.7% 提到 19.4%。

成本在表 2。平均每个任务的总 token 从 4010 万降到 1930 万，少了 51.9%。成本从 36.83 美元降到 23.62 美元，少了 35.9%，这已经算上了 CarryOn 自己调用模型的开销。成本是按论文设定的名义 API 价格算的。

在 BrowseComp-Plus 上，准确率从 49% 提到 56%，总 token 少了 22.9%，成本少了 7.1%。

图 3 能看出省在哪里。不加 CarryOn，平均输入上下文随着步数一路上涨，最后超过 20 万 token。加了之后，稳定在 6.5 万 token 左右。成本方面，前期 CarryOn 要额外调用模型做整理，大约 150 步之后两条曲线交叉，到第 500 步，累计成本低了 34.1%。

### 和以前方法的对比

和其他上下文管理方法的正面对比在表 4。这是从 OSWorld 2.0 里随机抽的 40 个任务，所有方法用同一个模型，同样的触发条件，同样的步数预算。

只保留最近消息的滑动窗口，部分得分从基线的 50.32% 掉到 37.41%。通用摘要掉到 44.94%。也就是说，单纯把历史压短，反而会伤害表现。

作者还做了一个对照，叫 CarryOn 引导的摘要，用同样的组织原则，但每次重新写一份完整的摘要，部分得分回到 50.93%。完整的 CarryOn 是 55.93%，而且折叠后保留的文本还短了大约 10%。

不过也要看到，在这 40 个任务上，二值得分 CarryOn 是 22.5%，和不做任何压缩的基线一样，没有提高。

消融实验在表 5。把执行上下文藏起来，部分得分从 55.93% 掉到 44%。把载体藏起来，部分得分掉到 44.62%，同时 Agent 调用次数多了 23.8%，成本多了 22.3%，作者的解释是，Agent 得重新找回之前做过的东西。

附录里有个例子。一个分配奖品的任务，藏起载体之后，Agent 在折叠后开始怀疑价格有没有确认过，于是重新提取 PDF，又打开商品目录，重建价格对应关系。

表 1 里还列了其他前沿模型的成绩，比如 GPT-5.5 部分得分 49.5%，Claude Opus 4.8 是 49.3%，作者称 CarryOn 超过了它们。但这些数字引自 OSWorld 2.0 原论文，不是同一次实验，GPT-5.5 还是批量执行动作的设置，差距也只有零点几个百分点。

我的判断是，这篇真正有说服力的，不是和别的模型比，而是同一个模型，同一套设置下的对比。表 4 那组实验说明，关键不在上下文压得多短，而在压缩之后留下了什么。

### 局限

论文没有专门写局限这一节，但从它给出的设置看，有几点要注意。

第一，所有实验只用了一个模型，Claude Opus 4.7，Agent 和 CarryOn 自己的模块都是它。换成更小的模型，Tracker 和 Compiler 还能不能做好，论文没有测。

第二，BrowseComp-Plus 只用了 100 道题，7 个百分点的提升，就是多答对 7 道题。论文也没有报告多次运行的波动。

第三，方法对比和消融都是在 40 个任务的子集上做的。而且 Grounder 的检查只是文件在不在，有没有变化这类条件，作者把它们定位为运行时内部的一致性保障，运行时的校验也不验证任务是否做对。

还有成本，前 150 步左右，用 CarryOn 反而比不用更贵，短任务未必划算。

### 未来方向

论文没有展开写未来方向。结论里的说法是，这些结果支持面向对象的上下文管理，让任务对象承载已经完成的进度，让显式的上下文只保留往前推进还需要的东西。

我的判断是，值得关注的方向，是把文件和应用状态本身当成 Agent 的记忆，而不是把一切都塞进 prompt。CarryOn 的贡献，是给这件事加上了版本管理和证据检查。

我的判断是，接下来最该看的，是换成开源的小模型来做 Tracker 和 Compiler，这套方法还成不成立，以及在更多任务，多次运行下，二值成功率能不能稳定提高。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from the University of California San Diego. It's called CarryOn, and the first half of the title is Pack Light.

The problem is a practical one. An agent working through a task that takes hundreds of steps keeps piling up context. How do you lighten that load without losing the progress it has already made?

### Problem

Start with where an agent's progress actually lives. There are two places. One is the task state itself: the images already inserted into a slide deck, the numbers already typed into a report. The other is the interaction history, the record of how all that got done.

Here's the waste. The results are already sitting in the files, yet at every step the agent rereads the story of how it produced them. The longer the task, the more of that redundancy piles up.

### Prior work

Existing methods fall into two camps. One compresses the history: summarize it, retrieve from it, or prune old turns. The other translates the history into explicit progress records or a structured task state.

The authors' argument is that both keep context growth in check, but they make the agent rely on a second-hand, selective account of its progress. And that account can drift away from the state the agent can actually inspect and change.

### Key idea

CarryOn changes the unit of organization. Instead of building context around history, it builds it around reusable task objects: finished files, findings with their sources, settings already applied inside an application. The authors call this object-oriented context.

Concretely, the context has three parts. The first is continuation carriers: each one records where a reusable object is, what progress it holds, and under what conditions it can be reused. The second is the execution context, which keeps the open goals, the constraints, and the unresolved issues. The third is the live tail, the raw interactions since the last cleanup.

Figure one shows an example. In a slide-editing task, after the cleanup, two exported images and the working deck become three carriers. The execution context keeps just a few items: keep forty percent transparency, leave other content alone, confirm the save. Active context drops from forty-two point four thousand tokens to fourteen point four thousand, a sixty-six percent cut.

### Method

That cleanup is called a fold. CarryOn runs alongside the agent, and each time it's triggered it performs a fold in four stages. This figure shows the whole pipeline.

Stage one, tracking candidates. A module called the Tracker reads recent interactions and flags progress that could become reusable, either a new carrier or an update to an existing one. Candidates that aren't ready yet stay on the list for a later fold.

Stage two, preparing evidence. Every candidate is bound to the evidence behind it. For information, the runtime checks the content against its recorded sources. For files and application state, a module called the Grounder runs read-only checks: does the file exist, has it changed from its initial state, is the document structure valid.

Stage three, joint compilation. A module called the Compiler updates the carriers and the execution context together. They're coupled: progress held by a carrier doesn't need repeating in the execution context, but unfinished goals and constraints must stay explicit. Affected carriers get a new version; unaffected ones keep their records.

Stage four, validation and commitment. The runtime checks that the edits refer to the current versions, that they match the authorized evidence, and that at least one carrier actually changes. If so, the update is committed in one step and the covered history leaves the context. If not, nothing changes.

Two caveats. The authors state that these checks only keep the runtime consistent; they don't verify that the task is being done correctly. And the raw interactions aren't deleted. They stay in an append-only log.

On implementation: the agent, the Tracker and the Compiler are all Claude Opus 4.7, with no extra training. On OSWorld 2.0, the Tracker normally runs whenever about six thousand tokens of new interaction text have built up.

### Results

Two benchmarks. OSWorld 2.0 is long-horizon computer use, with a median estimated human completion time of about one point six hours per task, and the authors run the full evaluation set with a five-hundred-step budget. BrowseComp-Plus is question answering that needs repeated search across many sources; they use a random subset of one hundred questions.

The main result is Table one. On OSWorld 2.0, with the same Claude Opus 4.7 agent, adding CarryOn raises the partial score from forty-three point nine to forty-nine point seven percent, and the binary score, which only counts fully completed tasks, from fifteen point seven to nineteen point four.

Costs are in Table two. Average total tokens per task fall from forty point one million to nineteen point three million, down fifty-one point nine percent. Cost per task falls from thirty-six eighty-three to twenty-three sixty-two dollars, down thirty-five point nine percent, and that includes CarryOn's own model calls. Costs use nominal API prices set in the paper.

On BrowseComp-Plus, accuracy rises from forty-nine to fifty-six percent, with twenty-two point nine percent fewer tokens and seven point one percent lower cost.

Figure three shows where the savings come from. Without CarryOn, mean input context keeps climbing with each turn, past two hundred thousand tokens. With it, context levels off around sixty-five thousand. On cost, CarryOn starts out slightly behind because of its extra calls, the curves cross at around a hundred and fifty turns, and by turn five hundred cumulative cost is thirty-four point one percent lower.

### Comparison

The head-to-head with other context strategies is Table four: forty randomly sampled OSWorld 2.0 tasks, with the same model, the same trigger condition and the same step budget for every method.

A sliding window that keeps only recent messages drops the partial score from the baseline's fifty point three two to thirty-seven point four one. A generic summary drops it to forty-four point nine four. In other words, just making the history shorter hurts.

The authors also test a CarryOn-guided summary, which follows the same organizing principles but rewrites a full summary at every fold. That gets back to fifty point nine three. Full CarryOn reaches fifty-five point nine three, while keeping about ten percent less text after each fold.

One thing to keep in view: on these forty tasks, CarryOn's binary score is twenty-two point five percent, exactly the same as the uncompressed baseline. No gain there.

The ablations are in Table five. Hide the execution context, and the partial score falls from fifty-five point nine three to forty-four. Hide the carriers, and it falls to forty-four point six two, while agent calls rise twenty-three point eight percent and cost rises twenty-two point three percent. The authors' explanation is that the agent has to recover work it already did.

The appendix has a concrete case. In a prize-distribution task with carriers hidden, after a fold the agent starts doubting whether the prices were confirmed, extracts the PDF again, and reopens the catalogue to rebuild the price mapping.

Table one also lists other frontier models, such as GPT-5.5 at forty-nine point five partial and Claude Opus 4.8 at forty-nine point three, and the authors say CarryOn exceeds them. But those numbers are cited from the OSWorld 2.0 paper, not from the same runs, GPT-5.5 used batched actions, and the margins are a few tenths of a point.

My read is that the convincing evidence here isn't the comparison with other models. It's the same-model, same-setup comparison. Table four suggests that what matters isn't how short the context gets, but what survives the compression.

### Limitations

The paper has no limitations section, but its setup raises a few points.

First, every experiment uses one model, Claude Opus 4.7, for the agent and for CarryOn's own modules. Whether a smaller model could run the Tracker and Compiler well is untested.

Second, BrowseComp-Plus uses one hundred questions, so the seven-point gain is seven more correct answers. The paper doesn't report variation across repeated runs.

Third, the method comparison and the ablations use the forty-task subset. And the Grounder's checks are about conditions like whether a file exists or has changed; the authors describe them as consistency safeguards inside the runtime, and the runtime's validation doesn't verify task correctness.

And on cost, for roughly the first hundred and fifty turns, CarryOn costs more than not using it, so short tasks may not benefit.

### Future work

The paper doesn't lay out future work. Its conclusion says the results support object-oriented context management, where task objects carry finished progress and explicit context keeps only what's still needed to move forward.

My read is that the direction worth watching is treating files and application state as the agent's memory, instead of stuffing everything into the prompt. CarryOn's contribution is adding versioning and evidence checks to that idea.

My read is that the next thing to check is whether this holds up with a small open model running the Tracker and Compiler, and whether the binary success rate improves reliably across more tasks and repeated runs.

That's today's Daily Frontier Papers. The paper is linked in the description.

