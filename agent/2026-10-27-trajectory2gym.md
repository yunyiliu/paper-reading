# Trajectory2Gym: Decision-Grounded Compositional Interaction Synthesis for Tool-Use Agents

**成功的轨迹不一定值得学**

- 论文 Paper: [arXiv 2610.33568](https://arxiv.org/abs/2610.33568)
- 单位 Affiliation: 西湖大学、蚂蚁国际
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-27
- 视频 Videos: [YouTube 中文](https://youtu.be/FIhU3LAgE9Q) · [YouTube English](https://youtu.be/nqUf17e10-Y) · [B站](https://www.bilibili.com/video/BV1SNxG6mEHB)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇西湖大学和蚂蚁国际的论文，名字叫 Trajectory2Gym，讲的是怎么给会调用工具的 Agent 造训练数据。

它的核心观点就一句话，一条成功的轨迹，不一定是好的训练数据。

### 问题

先看一个场景。让客服 Agent 取消两个订单，再改第三个。它要先查账户，用邮箱查不到，就问姓名和邮编再查，确认订单还在待处理，跟用户确认，最后才执行。每一步都简单，难的是把步骤正确地串起来。

作者把这种前一步的结果决定后一步怎么做的流程，叫做组合式交互。训练这类 Agent，标准做法是收集或者合成轨迹，也就是一次完整交互的记录，再让模型去模仿。

### 以前的方法

现在主流的做法，是按结果筛选轨迹，任务完成了，或者每次工具调用都执行成功了，就留下来当训练数据。

作者认为这有两个问题。第一是覆盖不够。在他们收集的 34772 条通过验证的候选轨迹里，40.9% 最多只有两个需要模仿的决策，只有 10% 在真实报错之后，还有一个通过验证的决策，也就是出错后怎么恢复，数据里很少。

第二是决策没有依据。执行成功，不代表这一步做得有道理。比如采集数据的模型，用了一个 Agent 自己根本看不到的编号，调用照样能成功。

论文图 1 是一个示意。用户先说要换一个大点的水杯和一个暗一点的台灯，后来改口说只换台灯。可是两个换货请求，接口都会接受。只看执行结果，就会把那个违背用户最新要求的换水杯操作，也当成正确答案学进去。

### 核心想法

所以作者的核心主张是，一条轨迹能教会模型什么，不取决于结果，而取决于两件事，它真正体现了哪些步骤之间的依赖，以及每一步决策，能不能从 Agent 当时看得到的信息里推出来。

对应两个要求。一个叫组合覆盖，重要的依赖结构要在数据里得到足够的支撑。一个叫决策有据，每个被监督的动作，都要有学生当时可见的历史作为依据。

### 方法

方法分三步，这张图是总览。第一步造交互，第二步筛选哪些动作可以当训练目标，第三步决定怎么分配数据和训练权重。

第一步，组合式交互合成。作者把交互抽象成路径，比如开头那个场景，就是查询、报错、澄清、重试、读订单、确认、取消、修改。哪些路径在现有数据里太少，就专门围绕它生成任务，再让老师模型在原生环境里执行。

关键在于，只有执行记录里真的出现了这些依赖，这条路径才算数。生成器或者老师出了错，只会变成一条不被认可的结构，不会变成训练目标。

第二步，决策有据的监督。学生能看到的上下文，只包括通过正常接口暴露出来的信息，老师拿到的提示和私有信息，都不会放进去。

然后逐个检查每个动作。一个工具调用要成为训练目标，首先要执行成功，其次这个工具要对 Agent 可见，像编号这样的参数，要能在之前看得到的信息里找到出处。只有采集者知道的编号，不能当训练目标。

最后还要求整条轨迹任务成功，并且必要的操作，按正确的顺序出现。

这里有一个细节。出错的调用不删掉，它和真实的报错信息留在上下文里，但不计算模仿损失。后面根据报错做出的恢复，才是训练目标。这样模型能看到为什么要恢复，但不会学着去犯错。

第三步，结构支撑学习。从通过验证的数据池里挑训练数据，先保证任务和路径的覆盖，再给发现信息、出错恢复、完成任务这类少见的结构补足重复次数。每条路径要几条样本，随模型变大而增加，从 1.7B 的 1 条，到 30B-A3B 的 8 条。

训练时还有一个加权设计，叫按父轨迹归一化的监督微调。一条长的多轮交互，会拆成好几条训练样本，如果按样本平均，长交互就会主导整个损失。

作者让每条完整交互拿到同样的总权重。比如两条交互分别拆成三条和一条样本，按样本平均，权重是四分之三和四分之一，归一化之后各占一半。

### 实验和效果

实验设置。学生是 Qwen3 的五个尺寸，1.7B、4B、8B、14B 和 30B-A3B，每个都从公开的初始模型出发，只做一轮全参数监督微调，没有强化学习，训练数据从 1.16 万到 1.99 万条轨迹。

评测用三个基准。Tau1 和 Tau2 是客服场景，Agent 要和模拟用户多轮对话，BFCL 是伯克利的函数调用排行榜。每个训练后的模型，都和它自己的初始模型在相同条件下对比。

先看表 1，Tau1 和 Tau2。五个尺寸全部提升，两个基准的平均分，1.7B 提高 20 分，4B 以上提高 35 到 41 分。

最有意思的一点是，数据带来的提升，超过了模型尺寸的差距。训练后的 4B 模型，在 Tau2 上拿到 69.17，而没训练的 30B-A3B 只有 28.33。训练后的 30B-A3B 在 Tau2 上是 84.17，和论文引用的 GPT-5.2 的 84.76、Claude-4.5-Opus 的 85.31，差距在 1.2 分以内。

提升也很普遍。五个领域乘五个尺寸，一共 25 组对比，24 组提高，唯一的例外是 1.7B 在 Tau2 航空上，持平在 20 分。变化最大的是 Tau2 电信，30B-A3B 从 12.5 提到 95。

再看表 2，BFCL。总分在每个尺寸都提高，主要来自多轮基础类别，提高 7 到 41 分。4B 以上每个训练后的模型，多轮基础分都超过了论文引用的 GPT-5.5 的 60 分。

但也有代价。单轮的 Live Simple 类别，五个尺寸里有四个下降了 6 到 13 分。作者的猜测是，模型变得更倾向于先收集信息，而这类问题只需要立刻调用一次。

### 和以前方法的对比

和公开的 Agent 训练模型比。作者把五个系列的公开模型，在同样的任务和评分规则下重新评测。他们的 8B 模型在 Tau2 上是 74.17，比每一个都高出至少 28 分，包括 14B 和 32B 的模型。三个基准的平均分是 67.97，其他模型最高是 57.17。

不过作者也说明，这些不是严格对照，训练数据、模型家族和上下文预算都不一样，不能据此判断哪个组件更好。

还有一个对比很能说明问题。EnvFactory-8B 在 BFCL 总分上几乎追平他们的 8B 模型，77.55 对 78.62，但在 Tau2 上落后 47.5 分。作者的结论是，会调函数，不等于会做有状态的多步任务。

消融实验在表 3，都在 8B 上做，每次只换一个设计。四组对比都支持论文的选择。按父轨迹归一化提升最大，三个基准的平均分从 59.94 到 67.97，高 8.04 分。定向扩充数据高 3.55 分，决策有据高 2.75 分，结构化选择高 2.32 分。

结构化选择这组很直观。同样 14819 条轨迹、同样的 token 预算，结构化选择覆盖了 6360 条路径，随机选只有 5418 条，多了 942 条，Tau1 和 Tau2 都提高，BFCL 基本不变。

图 4 是一个同题案例，任务要先改地址、再改商品。Simia 看到了正确地址，却反复查用户资料，最后转人工。MUA-RL 先改商品，订单状态变了，地址就改不了。Trajectory2Gym 先确认，再改地址，最后改商品，任务成功。

我的判断是，这篇真正的贡献，是把造数据拆成三个可以分别检查的层面，有哪些交互结构，哪些决策能被模仿，每条交互拿多少训练权重。其中按父轨迹归一化只是改了损失的加权，在消融里收益却最大，做 Agent 微调的团队值得一试。

### 局限

论文没有单独的局限章节，但有几处值得注意。第一，Tau1 航空的 50 道题，训练时用来构造交互，评测时是新的对话，作者自己说，这属于见过任务的设定，不是任务层面的泛化。

第二，评测规模不大，Tau2 一共 100 道题，航空只有 20 道。消融每组只跑一次，作者自己说，几分的差距只能看作方向性的证据。

第三，更多数据不一定更好。在深度敏感性实验里，每条路径 2 条样本时，三个基准的平均分最高，是 72.28，加到 3 条和 4 条，反而降到 70.55 和 69.44，主要掉在 Tau2。

第四，和闭源模型的比较只是参考，那些分数来自各自报告的结果，不是在同样条件下测出来的。

### 未来方向

作者认为，下一步是把这种逐个决策的验证，扩展到 on-policy 的 rollout，也就是模型自己跑出来的轨迹，以及网页、软件和终端这些类型的 Agent。论文首页给了代码和 Hugging Face 的链接。

他们给的总结很好记，成功应该是监督的门槛，而不是监督的许可证，真实的错误应该留在上下文里，而不是变成训练目标。

我的判断是，这套方法依赖能执行的原生环境和自带的任务检查器，Tau 和 BFCL 正好都具备。换到没有现成检查器的开放任务，能不能照搬，论文还没有验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Westlake University and Ant International. It's called Trajectory2Gym, and it's about how to build training data for agents that use tools.

The whole paper hangs on one idea. A successful trajectory is not necessarily good training data.

### Problem

Picture a support agent asked to cancel two orders and change a third. It looks up the account, the email lookup fails, so it asks for a name and ZIP code and tries again. It checks each order is still pending, confirms with the user, and only then acts. Each call is easy. What has to be learned is how they fit together.

The authors call these compositional interactions: an earlier observation or user update decides a later move. And the standard way to train for them is to collect or synthesize trajectories, full records of an interaction, and have the model imitate them.

### Prior work

The usual filter is the outcome. If the task was completed, or every tool call executed, the trajectory goes into the training set.

The authors see two problems with that. The first is coverage. In their own pool of thirty-four thousand seven hundred and seventy-two certified candidates, forty point nine percent have at most two supervised decisions, and only ten percent have a certified decision after a real error. Recovery from mistakes is rare in the data.

The second is grounding. A call that executes isn't necessarily a call the agent could have justified. The data collector might use an identifier the agent never saw, and the call still succeeds.

Figure 1 is a constructed illustration. A user asks for a bigger water bottle and a dimmer lamp, then says, actually, just the lamp. The API accepts both exchange requests. Filter by execution, and the bottle swap the user just cancelled gets learned as a correct answer.

### Key idea

So the central claim is this. What a trajectory teaches depends not on its outcome, but on which dependencies it actually realizes, and whether each decision can be justified from what the agent could see at the time.

That gives two requirements. Compositional coverage: the important dependency structures get enough support in the data. And decision grounding: every supervised action is backed by the history visible to the learner.

### Method

The method has three stages, shown in this overview. Build the interactions, decide which actions count as training targets, then decide how data and training weight are allocated.

Stage one is compositional interaction synthesis. Interactions are abstracted into paths; the support scenario from the start is lookup, error, clarification, retry, order read, confirmation, cancellation, update. Paths that are too rare in the existing data get tasks generated specifically around them, and a teacher model runs those tasks in the native environment.

The key rule is that a path only counts if its dependencies actually show up in the execution log. A mistake by the generator or the teacher just becomes an uncertified structure, never a training target.

Stage two is grounded supervision. The student's context contains only what the native agent interface exposes. Teacher hints and private events never get in.

Then each action is checked. To become a target, a tool call has to execute without error, the tool has to be available to the agent, and identifier-style arguments have to trace back to something visible earlier. An identifier only the collector knew can't support a target.

On top of that, the trajectory must succeed overall, with the required operations in the right order.

There's a nice detail here. A failed call isn't deleted. It stays in the context along with its real error message, but it gets no imitation loss. The recovery that follows is what gets trained, so the model sees why it had to recover without learning to make the mistake.

Stage three is structural-support learning. Selection from the certified pool first keeps tasks and paths covered, then adds repeats for rare structures like discovery, recovery and completion. How many examples each path gets grows with model size, from one at 1.7B to eight at 30B-A3B.

Training also changes the weighting, with what they call parent-normalized fine-tuning. One long multi-turn interaction gets rendered into several training examples, so under plain per-example averaging, long interactions dominate the loss.

Instead, every full interaction gets the same total weight. Say two interactions render into three examples and one. Per-example averaging gives them three quarters and one quarter; parent normalization gives each one half.

### Results

The setup. The students are five Qwen3 sizes: 1.7B, 4B, 8B, 14B and 30B-A3B. Each starts from its public checkpoint and gets one epoch of full-parameter supervised fine-tuning, no reinforcement learning, on between eleven thousand six hundred and nineteen thousand nine hundred trajectories.

There are three benchmarks. Tau1 and Tau2 are customer-service tasks where the agent talks with a simulated user over many turns, and BFCL is the Berkeley function calling leaderboard. Every trained model is compared with its own starting checkpoint under identical conditions.

Table 1 covers Tau1 and Tau2. All five sizes improve, and the average of the two goes up by twenty points at 1.7B and by thirty-five to forty-one points from 4B upward.

The striking part is that the data gain outweighs model size. The trained 4B model scores sixty-nine point one seven on Tau2, while the untrained 30B-A3B model scores twenty-eight point three three. The trained 30B-A3B reaches eighty-four point one seven on Tau2, within one point two of the reported GPT-5.2 at eighty-four point seven six and Claude-4.5-Opus at eighty-five point three one.

The gains are broad, too. Five domains times five sizes is twenty-five comparisons, and twenty-four improve; the one exception, Tau2 airline at 1.7B, stays flat at twenty. The biggest move is Tau2 telecom at 30B-A3B, from twelve and a half to ninety-five.

Table 2 is BFCL. The overall score rises at every size, driven by the multi-turn base category, up seven to forty-one points. Every trained model from 4B up beats the reported GPT-5.5 score of sixty on multi-turn base.

There is a cost. The single-turn Live Simple category drops six to thirteen points at four of the five sizes. The authors' hypothesis is that the models now lean toward gathering information, on queries that just want one immediate call.

### Comparison

Now against public agent-training models. The authors re-evaluated checkpoints from five families on the same tasks and scorers. Their 8B model scores seventy-four point one seven on Tau2, at least twenty-eight points above every one of them, including 14B and 32B models. Its three-benchmark mean is sixty-seven point nine seven, against at most fifty-seven point one seven for the others.

The authors are clear that these aren't matched comparisons. Training data, model families and context budgets all differ, so you can't attribute the gap to any one component.

One comparison is especially telling. EnvFactory-8B nearly matches their 8B model on BFCL overall, seventy-seven point five five versus seventy-eight point six two, yet trails by forty-seven and a half points on Tau2. The authors' conclusion: function-calling skill doesn't transfer to stateful, multi-step tasks.

The ablations are in Table 3, all at 8B, each changing one design choice. All four favor the paper's choices. Parent normalization gives the biggest gain, lifting the three-benchmark mean from fifty-nine point nine four to sixty-seven point nine seven, eight point zero four points. Targeted expansion adds three point five five, grounding two point seven five, and structural selection two point three two.

The structural-selection pair is easy to read. Same fourteen thousand eight hundred and nineteen trajectories, same token budget: structural selection covers six thousand three hundred and sixty paths, random selection five thousand four hundred and eighteen, so nine hundred and forty-two more. Both Tau benchmarks go up, and BFCL barely moves.

Figure 4 is a same-task case where the address has to change before the items. Simia has already seen the right address, but keeps querying the user profile and hands off to a human. MUA-RL edits the items first, the order status changes, and the address can no longer be updated. Trajectory2Gym confirms, updates the address, then edits the items, and passes.

My read is that the real contribution is splitting data construction into three things you can check separately: which interaction structures exist, which decisions are eligible for imitation, and how much weight each interaction gets. Parent normalization is just a change in loss weighting, yet it gave the largest gain in the ablations, so it's worth trying for anyone fine-tuning agents.

### Limitations

There's no dedicated limitations section, but a few caveats stand out. First, the fifty Tau1 airline tasks were used to build training interactions and then evaluated with fresh dialogues, which the authors themselves call a seen-task setting, not task-level generalization.

Second, the evaluation is small: Tau2 has a hundred tasks, only twenty of them airline. Each ablation arm is a single run, so the authors call gains of a few points directional.

Third, more support isn't automatically better. In the depth sweep, two examples per path gives the best three-benchmark mean, seventy-two point two eight, and three or four drop it to seventy point five five and sixty-nine point four four, mostly on Tau2.

Fourth, the closed-model comparisons are context only: those scores come from each source's reported results, not from identical conditions.

### Future work

The authors see the next step as extending decision-level certification to on-policy rollouts, meaning trajectories the model generates itself, and to web, software and terminal agents. The first page links the code and a Hugging Face page.

Their summary is easy to remember. Success should gate supervision, not license it, and real errors belong in the context, not in the targets.

My read is that the method leans on executable native environments with built-in task checkers, which Tau and BFCL both provide. Whether it carries over to open-ended tasks without a ready-made checker is something the paper hasn't tested.

That's today's Daily Frontier Papers. The paper is linked in the description.

