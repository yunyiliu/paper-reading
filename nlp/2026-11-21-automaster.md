# AutoMaster: Autonomous Post-Training by Making the Same Model a Better Teacher

**做对了还不够要教对过程**

- 论文 Paper: [arXiv 2611.26510](https://arxiv.org/abs/2611.26510)
- 单位 Affiliation: 中国人民大学高瓴人工智能学院
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-21
- 视频 Videos: [YouTube 中文](https://youtu.be/c6Ahp4qzPgw) · [YouTube English](https://youtu.be/08AjImnvZ9A) · [B站](https://www.bilibili.com/video/BV1Rzy3znE1A)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇中国人民大学高瓴人工智能学院的论文，名字叫 AutoMaster，副标题是，让同一个模型变成更好的老师。

它问的是一个很实际的问题，拿大模型做成功的轨迹去训练小模型 Agent，光看任务做没做成，到底够不够。

### 问题

先说背景。现在训练 Agent，常见的做法是让一个强的老师模型去做任务，把做成功的轨迹挑出来，再拿去给小模型做 SFT，也就是监督微调。轨迹，就是模型一步步思考、调用工具、拿到结果的完整记录。

问题在于，任务成功，不代表过程里每个行为都值得学。作者举的例子有，没有依据的假设，没用的重复，还有走过场的验证。这些毛病会导致失败，但也会出现在最后做对了的轨迹里。

把这些轨迹一起拿去微调，坏习惯就会和真正有用的动作一起被强化。反过来，一些好的行为，在成功的轨迹里本来就很少见。

论文开头这张图是个例子。任务是修一个有时会卡住的并行处理程序，没有引导的老师和有引导的老师，修完都能处理全部 100 条记录。区别在于，前者只测了默认的 4 个 worker 这一种配置，后者把 worker 数换成 1、2、4、8，测了四种配置。

### 以前的方法

以前的做法大致有几类。最常见的是直接在成功轨迹上做 SFT。再进一步，有的方法专门挑展示了好行为的轨迹，有的方法针对老师做失败的任务，逐个优化提示词，把失败的示范救回来。

另一条线是用自然语言写的技能，也就是 skill，来改进 Agent。有的只改提示词，不动模型参数，有的把技能训练成专门的适配器，还有的让带着技能的老师，给学生自己生成的 token 打分。

还有一类是让 AI Agent 自己去跑训练流程，根据评测反馈，反复修改训练代码、数据和策略，这叫自主后训练。

### 核心想法

这篇的核心想法是，与其只在老师的输出里挑，不如在老师生成轨迹的时候，就主动去塑造它的行为。

具体做法叫 Skill-FT，技能引导微调。用自然语言写的技能，压住老师的坏习惯，引出那些平时很少出现的有效行为，就连老师本来就能做对的任务，也去改它的做法。

然后让一个大模型 Agent 来自动完成这件事，从分析轨迹、设计技能、挑选数据，到看学生的评测结果再修改，这个系统就叫 AutoMaster。

### 方法

先说 Skill-FT。技能是一段文字，说明在什么情况下该怎么思考、怎么行动，可以是行为原则、具体步骤，也可以是领域知识。老师生成轨迹的时候，把技能接在系统提示词的后面。

训练学生用的是标准的 SFT，而且保留老师当时看到的技能说明。但评测学生的时候，不给任何生成的技能，完全按基准的官方流程来。

AutoMaster 有两层循环，看这张图。内层循环里，Agent 先分析老师和学生的初始轨迹，找出坏的模式和好的模式，然后设计技能，分配给训练任务的不同子集。

接着先小规模试跑，看技能有没有引出想要的行为，有没有带来新的问题。有效就扩大规模批量生成，再由 Agent 挑出值得训练的轨迹，它也可以挑没加技能、但本身就不错的轨迹。

外层循环负责训练和评测。每一轮都从同一个初始学生模型开始，用挑出来的数据做 SFT，评测分数和学生的轨迹再反馈给 Agent，由它决定下一轮是改技能、换任务范围，还是换数据。

工程上，Agent 通过 MCP 协议提交老师生成和训练任务，每次评测都能追溯到对应的训练集和技能。

### 实验和效果

实验设置。一共三个领域，终端任务用 Terminal-Bench 2.1 评测，搜索任务用 BrowseComp 的一个 185 题的子集，软件工程用 Doc2Repo。每个领域随机抽 1000 个训练任务，作为所有方法共用的固定任务池。

学生是 Qwen3.5 的 4B 和 9B。老师是 DeepSeek-V4-Pro-Preview。负责做实验的 Agent，是在 Codex 里运行的 GPT-6 Astra。

对比的基线叫 Vanilla SFT，就是从老师没加引导、并且做成功的轨迹里，随机抽 512 条来训练。AutoMaster 同样只用 512 条，训练配置完全一样，外层循环最多跑五轮。

结果在表 1。六个设置里，AutoMaster 的平均分全部最高，比 Vanilla SFT 高 1.42 到 8.99 个百分点。三个领域里，9B 的提升都比 4B 大。

最明显的是 9B 学生在 Terminal-Bench 2.1 上。Vanilla SFT 只把准确率从 25.47 提到 26.59，同一个老师、同样的任务池和数据量，AutoMaster 做到了 35.58。

作者还做了配对 bootstrap，估计这些提升的置信区间。9B 的三个基准，百分之九十五置信区间的下限都大于零。但 4B 在 BrowseComp 和 Doc2Repo 上，提升比较小，置信区间包含零，也就是说，不能排除这两项其实没有提升。

消融实验在图 4。去掉技能引导，只让 Agent 从普通轨迹里挑，或者保留技能、但不让 Agent 挑数据，六个设置里平均分都会下降。

两部分谁更重要，要看任务。4B 学生在 Terminal-Bench 上，只用技能是 20.22，只挑数据是 17.60，两个都低于 Vanilla SFT 的 22.47，合在一起才到 28.09。到了 BrowseComp，顺序反过来，只挑数据是 45.77，只用技能是 43.78。

还有一个有意思的发现，在图 6。同样的轨迹、同样的训练配置，训练时保留技能说明，比删掉它，平均分高 2.15 到 4.01 个百分点，六个设置全是这样。而评测的时候，学生是看不到这些技能的。

按理说，删掉技能，训练输入和评测输入更一致，应该更好才对。作者给的可能解释是，保留技能，等于把行为规则和它的适用条件写在输入里，学生更容易学会，什么时候该这么做。

最后看多轮实验，在图 8。六个设置里，最高分都出现在第二到第四轮，比第一轮高 2.16 到 4.87 个百分点。比如 9B 在 BrowseComp 上，前三轮都在 49 左右，到第四轮才到 51.53。

### 和以前方法的对比

最关键的一组对照，是排除一个疑问，提升会不会只是因为 AutoMaster 挑了更合适的训练任务。作者构造了 Task-Matched SFT，用同一个老师没加引导的轨迹，任务和每个任务的样本数，都和 AutoMaster 最终的训练集一模一样。

结果，AutoMaster 在六个设置里仍然全部更高，差距在 1.08 到 10.11 个百分点。

再看训练数据本身。在终端和搜索任务里，有百分之 79.3 到 91.2 的配对样本，没引导的老师和 AutoMaster 选的示范都做成功了。单看成功这个标签，分不出两者的差别。

图 5 给了三个例子。数值计算任务里，两个解都达标，但没引导的老师声称做到了四阶精度，实测收敛阶只有二阶左右。搜索任务里，两条轨迹都找对了答案，没引导的老师反复猜奖项的名字，有引导的老师先从剧集信息里找到导演，再去核对线索。软件工程任务里，没引导的老师为了通过一个写错的自测，把排序规则反了过来，38 个测试过了 32 个，有引导的老师改正自测，38 个全过。

我的判断是，这篇最有价值的地方，是把注意力从结果对不对，挪到了过程好不好。按它的对照实验，只按成功率筛轨迹，会低估同一个老师还能教出多少东西。

我的判断是，和只改提示词的方法比，它的技能最后是训练进模型里的，评测时不用再带着。和单纯挑数据的方法比，它多了一步，主动去改老师的行为。

### 局限

论文没有专门写局限，但从它自己的数据能看出几点。第一，4B 的提升不太稳定。除了那两个包含零的置信区间，多轮实验里，4B 只在 Terminal-Bench 上五轮都超过 Vanilla SFT。作者也认为，模型更大，可能更能稳定拿到这些收益。

第二，实验范围比较窄。只有一个老师，学生只有 Qwen3.5 一个系列的两个尺寸。论文报告的标准差，按它自己的说法，只反映同一个 checkpoint 三次评测之间的波动，不包括重新训练带来的差异。

第三，最终用哪个模型，是 AutoMaster 根据累积的评测反馈自己挑的，而这些基准评测的分数和轨迹，本来就会反馈给后续实验。另外，论文没有报告整套系统花了多少算力和 API 成本。

### 未来方向

论文也没有专门讨论未来方向。结论里，作者的说法是，一个固定的模型，通过行为塑造，可以变成更好的老师，就算在它本来能做对的任务上也是这样。保留技能上下文，也可能帮学生更好地把这些行为学进去。

我的判断是，下一步最值得看的，是在实验 Agent 没见过的测试集上，结果能不能保持，以及换别的老师和学生，是不是一样有效。

我的判断是，对做 Agent 训练数据的人，最直接能借鉴的有两点。第一，别只按成功率筛轨迹，要看过程。第二，用提示词引导老师生成的数据，训练时可以把那段提示留着，在这篇的实验里，留着比删掉更好。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the Gaoling School of Artificial Intelligence at Renmin University of China. It's called AutoMaster, and the subtitle is: making the same model a better teacher.

The question behind it is a practical one. When you train a small agent on a big model's successful runs, is "the task succeeded" really enough of a filter?

### Problem

Some background. A common way to train agents today is to let a strong teacher model attempt tasks, keep the runs that succeed, and fine-tune a small model on them with supervised fine-tuning, or SFT. A run here is a trajectory: the full record of the model's reasoning, tool calls and results, step by step.

The catch is that a successful outcome doesn't mean every behavior along the way is worth copying. The authors' examples are unsupported assumptions, unproductive repetition, and superficial verification. Those habits can cause failures, but they also show up in runs that end up correct.

Fine-tune on those runs, and the bad habits get reinforced right alongside the useful actions. Meanwhile, some good behaviors are rare even among the successful runs.

The paper's first figure shows an example. The task is to fix a parallel pipeline that sometimes freezes. The unguided teacher and the guided teacher both end up processing all one hundred records. The difference: the first only tests the default setting of four workers, while the second varies the worker count, one, two, four and eight, and tests four configurations.

### Prior work

Prior work falls into a few camps. The most common is plain SFT on successful trajectories. Going further, some methods select trajectories that show beneficial behaviors, and others optimize the prompt task by task to recover verified demonstrations from the teacher's failures.

A second line uses natural-language skills to improve agents. Some only refine the prompt, without touching model weights. Some train skill-specific adapters. And some have a skill-conditioned teacher score the tokens of the student's own runs.

A third line has AI agents run the training pipeline themselves, revising training code, data and strategy in response to evaluation feedback. That's called autonomous post-training.

### Key idea

The key idea here: instead of only picking from what the teacher produces, actively shape the teacher's behavior while it's generating the trajectories.

The method is called Skill-FT, for skill-guided fine-tuning. Natural-language skills suppress the teacher's bad habits and draw out effective behaviors that rarely appear on their own, even on tasks the teacher already solves.

Then an LLM agent automates the whole thing: analyzing trajectories, designing skills, selecting data, and revising based on how the student scores. That system is AutoMaster.

### Method

First, Skill-FT. A skill is a piece of text saying how to reason or act in certain situations. It can be a behavioral principle, a concrete procedure, or domain knowledge. During teacher rollouts, the skill is appended to the system prompt.

The student is trained with standard SFT, and the skill instructions the teacher saw are kept in the training input. But at evaluation, the student gets no generated skills at all and follows each benchmark's official protocol.

AutoMaster runs two nested loops, shown in this figure. In the inner loop, the agent first analyzes the initial trajectories of the teacher and the student, looking for bad patterns and good ones. Then it designs skills and assigns them to subsets of the training tasks.

Next come small pilot rollouts, to check whether a skill brings out the intended behavior without creating new problems. If it works, the agent scales up to batched rollouts and selects the trajectories worth training on. It can also pick unguided runs that already show useful behavior.

The outer loop handles training and evaluation. Every round starts from the same initial student, fine-tunes it on the selected data, and feeds the scores and the student's trajectories back to the agent. The agent then decides whether to revise a skill, change which tasks it covers, or pick different data.

On the engineering side, the agent submits rollout and training jobs through MCP, and every evaluation can be traced back to its training set and the skills behind it.

### Results

The setup covers three domains. Terminal tasks are evaluated on Terminal-Bench two point one, search on a one-hundred-eighty-five-question subset of BrowseComp, and software engineering on Doc2Repo. Each domain gets one thousand randomly sampled training tasks, a fixed pool shared by every method.

The students are Qwen3.5 4B and 9B. The teacher is DeepSeek-V4-Pro-Preview. The agent running the experiments is GPT-6 Astra inside Codex.

The baseline is Vanilla SFT: five hundred twelve trajectories sampled at random from the teacher's successful, unguided runs. AutoMaster also gets exactly five hundred twelve, with the same training recipe, and a budget of five outer-loop rounds.

Results are in Table 1. AutoMaster has the highest mean score in all six settings, beating Vanilla SFT by one point four two to eight point nine nine percentage points. And in all three domains, the 9B model gains more than the 4B.

The biggest jump is the 9B student on Terminal-Bench. Vanilla SFT only lifts its accuracy from twenty-five point four seven to twenty-six point five nine percent. With the same teacher, task pool and data size, AutoMaster reaches thirty-five point five eight.

The authors also ran a paired bootstrap to get confidence intervals on these gains. For 9B, all three ninety-five percent intervals sit above zero. But for 4B on BrowseComp and Doc2Repo, the gains are smaller and the intervals include zero, so a zero gain can't be ruled out there.

The ablations are in Figure 4. Remove skill guidance, so the agent only selects from unguided runs, or keep the skills but drop the agent's data selection, and the mean score falls in all six settings.

Which part matters more depends on the task. For the 4B student on Terminal-Bench, skills alone give twenty point two two and selection alone gives seventeen point six. Both are below Vanilla SFT at twenty-two point four seven, and only together do they reach twenty-eight point zero nine. On BrowseComp the order flips: selection alone gives forty-five point seven seven, skills alone forty-three point seven eight.

Here's an interesting finding, in Figure 6. Same trajectories, same recipe: keeping the skill instructions in the training input beats removing them by two point one five to four point zero one points, in all six settings. And remember, the student never sees these skills at evaluation.

You'd expect removing them to help, since training then looks more like evaluation. The authors' possible explanation is that keeping the skill puts the behavioral rule and the conditions for using it right in the input, which may help the student learn when an action applies.

Finally, the rounds, in Figure 8. In all six settings, the best score shows up in round two, three or four, two point one six to four point eight seven points above round one. For example, 9B on BrowseComp hovers around forty-nine for three rounds before reaching fifty-one point five three in round four.

### Comparison

The most important control rules out one suspicion: maybe AutoMaster just picked better training tasks. So the authors built Task-Matched SFT, from the same teacher's unguided runs, with the same task IDs and the same number of samples per task as AutoMaster's final training set.

AutoMaster still comes out ahead in all six settings, by one point zero eight to ten point one one points.

Now look at the training data itself. In the terminal and search corpora, seventy-nine point three to ninety-one point two percent of the paired examples succeed both ways, unguided and AutoMaster. On those pairs, the success label alone can't tell the two apart.

Figure 5 gives three cases. In a numerical task, both solutions meet the error threshold, but the unguided teacher claims fourth-order accuracy when the measured convergence is roughly second order. In search, both runs find the right answer, but the unguided teacher keeps guessing award names, while the guided teacher identifies the director from the episode credits and then checks the clues. In software engineering, the unguided teacher reverses the sorting rule to pass a flawed self-test and passes thirty-two of thirty-eight tests, while the guided teacher fixes the self-test and passes all thirty-eight.

My read is that the real value here is shifting attention from whether the outcome is right to whether the process is good. By the paper's own controls, filtering on success alone underestimates what a fixed teacher can teach.

My read is that, compared with prompt-only methods, the skills end up trained into the model, so they aren't needed at test time. And compared with pure data selection, it adds a step: actively changing how the teacher behaves.

### Limitations

The paper has no limitations section, but its own numbers point to a few. First, the 4B gains are less consistent. Besides those two intervals that include zero, across the rounds the 4B model beats Vanilla SFT in all five only on Terminal-Bench. The authors suggest a larger model may get these gains more consistently.

Second, the evaluation is narrow. There's one teacher, and the students are two sizes from a single family, Qwen3.5. And the reported standard deviations, by the paper's own description, only reflect variation across three evaluation runs of a fixed checkpoint, not retraining.

Third, the final checkpoint is chosen by AutoMaster itself based on accumulated evaluation feedback, and those benchmark scores and trajectories are what feed the later experiments. The paper also doesn't report what the whole system cost in compute or API spend.

### Future work

There's no future-work section either. In the conclusion, the authors say a fixed model can become a more effective teacher through behavioral shaping, even on tasks it already solves, and that preserving skill context may help the student internalize those behaviors.

My read is that the next thing to watch is whether the gains hold on a test set the experimenting agent never saw, and whether the method works as well with other teachers and other students.

My read is that for anyone building agent training data, two lessons are directly usable. Don't filter trajectories on success alone, look at the process. And if you steer a teacher with a prompt, consider keeping that prompt in the training input. In this paper, keeping it beat removing it.

That's today's Daily Frontier Papers. The paper is linked in the description.

