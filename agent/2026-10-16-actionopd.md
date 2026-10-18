# ActionOPD: Distilling Decisions, Not Tokens

**让小模型学会大模型的决策**

- 论文 Paper: [arXiv 2610.19820](https://arxiv.org/abs/2610.19820)
- 单位 Affiliation: 清华大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-16
- 视频 Videos: [YouTube 中文](https://youtu.be/W9I0vrd1Y_I) · [YouTube English](https://youtu.be/NlFsrYb9rto) · [B站](https://www.bilibili.com/video/BV1rCJQ6YEPN)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇清华大学的论文，名字叫 ActionOPD，副标题很直白，蒸馏决策，而不是蒸馏 token。

它想解决的问题是，怎么把一个又强又贵的大模型 Agent，教给一个便宜的小模型。

### 问题

先说背景。Agent 和聊天不一样，它每做一个决定，比如调用哪个工具、填什么参数，都会改变它接下来看到的东西。

所以把大模型的本事教给小模型，有两个问题绕不开，老师应该在哪些状态下指导学生，以及师生应该在多细的粒度上对齐。

### 以前的方法

以前主要有两条路。第一条是离线蒸馏，让老师先跑一遍，学生照着老师写出来的每个 token 去模仿。

问题在于，学生自己跑的时候，走到的局面和老师不一样，到了老师没去过的地方，学生就没人教了，这叫状态分布不匹配。

第二条是在线蒸馏，也叫 on-policy distillation，简称 OPD。让学生自己跑，老师在旁边给学生生成的每个 token 打分。

这解决了不匹配的问题，但它要拿到老师每个 token 的概率，通常还要求师生用兼容的分词器。如果老师是闭源 API，或者跟学生不是一个模型家族，就很麻烦，要么拿不到概率，要么得额外对齐分词器。

论文这张图把几种做法放在一起对比。你会发现，不管离线还是在线，监督的单位都是 token。

### 核心想法

作者的观察是，对 Agent 来说，真正重要的不是 token，而是动作。

一次工具调用，表达的是一个决定，就是工具名加参数。同一个决定，可以用完全不同的措辞和推理过程写出来。

所以目标应该从复现老师写的那段文字，变成提高学生做出老师那个动作的概率。这就是 ActionOPD 的出发点。

### 方法

具体怎么做。目标函数写出来是这个式子，在学生自己走到的状态上，让学生做出老师动作的概率尽量大。

这里的概率，指的是学生所有在语义上等于老师动作的输出加起来的总概率，而不是一字不差复现某一句话的概率。

把这个式子变一下形，就得到一个标准的 REINFORCE 策略梯度，奖励就是学生的动作和老师的动作在语义上是否一致。

关键好处是，算这个梯度只需要学生自己的概率，老师只要能生成回答就行，不需要老师的 token 概率，也不需要分词器对齐。

那怎么判断语义一致。每个状态下，先让老师生成三个回答。工具调用用程序比对工具名和规范化后的参数，剩下的差异交给一个冻结的大模型判断。

比如查天气，城市写巴黎、单位写摄氏度，参数顺序换一下，还是同一个调用。对用户说的话，就比较意图和事实内容。

奖励是这样设计的。无效的输出扣一分，有效但和老师都不一致的得零分，和老师一致的至少得一分，和更多老师样本一致，再加一点奖励。

论文举了例子，老师三次分别给出 A、A、B，学生答 A 得 1.25 分，答 B 得 1 分。

训练时还有一个设计，叫状态分组 rollout。在每个状态，学生先生成四个候选，全部和老师的样本比对打分。

环境里只执行得分最高的那一个，但只要这一组候选得分有差别，四个候选都参与训练。这样学生能走到老师引导的轨迹上，同一状态下的几个候选也能互相比较。

最后的优化用的是组内相对优势加 PPO 式裁剪。

### 实验和效果

实验设置。学生是 Qwen3 的 4B 和 8B 两个小模型，老师是 Qwen3.7-Flash 和 DeepSeek-V4-Flash，两两组合，一共四种配置。

训练数据来自 Agent World Model 和 EnvScaler，一共 6697 个工具调用任务，每种配置训练 100 步。

评测用的是训练里没见过的四个基准，BFCL 多轮、ToolSandbox、AppWorld 和 Tau2-Bench。

结果在表 1。四种配置的四项平均分，全部比训练前提高，幅度在 3.3 到 6.9 个百分点。

提升最多的是 4B 学生配 DeepSeek-V4-Flash 老师，平均分从 30.8 提到 37.7。

还有两个规律。在测试的两个尺寸和两个老师里，学生越大，最终分数越高。老师越强，学生学得越好，DeepSeek-V4-Flash 本身的平均分是 71.2，比 Qwen3.7-Flash 的 65.6 高，教出来的学生也更好。

这也说明，用 DeepSeek 的老师教 Qwen 的学生是可行的，师生不需要是同一个模型家族。

### 和以前方法的对比

再看正面对比。作者用同一个 4B 学生、同一个 Qwen3-32B 老师、同样的 Tau2-Bench 训练任务，比较离线蒸馏、token 级 OPD 和 ActionOPD。

在测试集的单次成功率上，离线蒸馏是 37.5，OPD 是 39.6，ActionOPD 是 40.0，比 OPD 高 0.4 个百分点。

但在四次都成功的稳定性指标上，离线蒸馏最高，是 19.2，ActionOPD 是 14.2，也比 OPD 的 17.5 低。作者也承认，这不能说明 ActionOPD 全面更强。

我的判断是，ActionOPD 真正的优势不在分数，而在接口。它只要老师能生成回答，这意味着闭源模型也可以当老师，不过论文本身没有测试闭源老师。

消融实验在表 3。如果选动作时仍然用语义打分，但训练奖励换成只看动作是否有效，测试集单次成功率掉到 30.8，是几个训练变体里最低的。

原因是所有有效的候选拿到一样的奖励，模型学不到哪个动作更好。在这组消融里，去掉语义比对，影响是最大的。

论文还挑了几个案例。比如航空公司场景，用户要改基本经济舱的航班，按规定不能改，训练前的模型四次里有两次直接改了，用 DeepSeek 老师训练后，模型会解释规定，用户要求转接时再转人工。

最后还有一个延伸实验，叫 Self-ActionOPD。老师和学生是同一个模型，只是老师额外看到用户目标、参考动作这些学生看不到的信息。

训练 50 步之后，学生在只看公开信息的情况下，测试集单次成功率提高了 4.8 个百分点。

### 局限

作者自己列了三条局限。

第一，和老师一致，不等于正确。老师犯的错，也会被学生当成目标学过去。

第二，用大模型判断语义是否一致，本身会判错，可能把错的当成对的，或者把对的当成错的，奖励就给错了。

第三，每个状态只采样三个老师回答、四个学生候选。如果合理的决定很多、很分散，长流程任务里就可能这样，可能凑不出匹配，信号就很弱。

### 未来方向

未来方向，作者提到可以增加老师和学生的采样数量，但成本更高，效果还没验证。

他们也认为 Self-ActionOPD 可能指向一条路，就是 Agent 用自己加特权信息来自我改进。代码、数据和模型，论文说会在 GitHub 公开。

我的判断是，这篇的价值在于换了监督的单位。从对齐 token 变成对齐决策，以后用闭源大模型教开源小模型做 Agent，门槛会低很多。

我的判断是，也要看到，训练后的学生离老师还很远，最好的 4B 学生平均 37.7，老师是 71.2。蒸馏能缩小差距，但还远没到替代老师的程度。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Tsinghua University, and the title says it all: ActionOPD, distilling decisions, not tokens.

The question it tackles is simple to state. How do you teach a small, cheap model to behave like a big, expensive agent?

### Problem

Here's why that's harder for agents than for chatbots. Every decision an agent makes, which tool to call and with what arguments, changes what it sees next.

So any distillation method has to answer two questions. At which states should the teacher supervise the student, and at what granularity should their behavior be aligned?

### Prior work

There are two standard answers. The first is offline distillation: the teacher runs the task, and the student imitates every token the teacher wrote.

The catch is that the student, running on its own, drifts into situations the teacher never visited. Once it's off the teacher's path, it never learned what to do there. That's the state-distribution mismatch.

The second answer is on-policy distillation, or OPD. The student runs the task itself, and the teacher scores each token the student produces.

That fixes the mismatch, but it needs the teacher's token probabilities, and usually a compatible tokenizer. If your teacher is a closed API, or comes from a different model family, that gets hard: you need its token probabilities, and extra machinery to line up the tokenizers.

This figure lines the approaches up. Notice that whether it's offline or on-policy, the unit of supervision is always the token.

### Key idea

The authors' observation is that for an agent, the thing that matters isn't the token. It's the action.

A tool call expresses a decision: a tool name plus arguments. And the same decision can be written with very different wording and very different reasoning.

So the goal shifts from reproducing the teacher's exact text to raising the probability that the student takes the teacher's action. That's the core of ActionOPD.

### Method

Here's the objective. On the states the student actually reaches, maximize the probability that the student produces the teacher's action.

And that probability means the total probability of every student output that means the same thing as the teacher's action, not the probability of copying one particular sentence.

Rewrite this and you get a standard REINFORCE policy gradient, where the reward is simply whether the student's action semantically agrees with the teacher's.

The practical payoff is that the gradient only needs the student's own probabilities. The teacher just has to generate answers. No teacher token probabilities, no tokenizer alignment.

So how do you decide two actions agree? At each state, the teacher is sampled three times. Tool calls are compared by program, tool name and canonicalized arguments, and a frozen language model judges whatever differences remain.

For example, a weather call for Paris in Celsius is the same call even if the arguments come in a different order. For messages to the user, the matcher compares intent and factual content.

The reward works like this. An invalid output gets minus one. A valid output that matches no teacher sample gets zero. A match gets at least one, plus a small bonus for matching more of the teacher's samples.

The paper's example: if the teacher said A, A and B, a student answering A earns one point two five, and a student answering B earns one.

Training adds one more trick, which the authors call state-group rollout. At each state, the student proposes four candidates, and all four are scored against the teacher's samples.

Only the highest-scoring candidate is executed in the environment, but all four are used for training, as long as their rewards differ. That steers the student along teacher-guided trajectories, and gives every state a small group of answers to compare.

The update itself uses group-relative advantages with a PPO-style clipped objective.

### Results

The setup. The students are Qwen3 4B and 8B. The teachers are Qwen3.7-Flash and DeepSeek-V4-Flash. That's two by two, four configurations.

Training data comes from Agent World Model and EnvScaler, six thousand six hundred and ninety-seven tool-use tasks in total, with one hundred training steps per configuration.

Evaluation uses four benchmarks that were kept out of training: BFCL multi-turn, ToolSandbox, AppWorld, and Tau2-Bench.

The results are in Table 1. All four configurations improve the four-benchmark average over their starting point, by between three point three and six point nine percentage points.

The biggest jump is the 4B student with the DeepSeek-V4-Flash teacher, going from thirty point eight to thirty-seven point seven.

Two patterns stand out. Across the two student sizes and two teachers tested, bigger students end higher, and stronger teachers produce better students. DeepSeek-V4-Flash scores seventy-one point two on its own, above Qwen3.7-Flash at sixty-five point six, and its students do better too.

It also shows that a DeepSeek teacher can train a Qwen student. Teacher and student don't need to share a model family.

### Comparison

Now the head-to-head. Same 4B student, same Qwen3-32B teacher, same Tau2-Bench training tasks, comparing offline distillation, token-level OPD, and ActionOPD.

On single-attempt success on the test set, offline distillation scores thirty-seven point five, OPD thirty-nine point six, and ActionOPD forty point zero, which is zero point four points above OPD.

But on the reliability metric, succeeding in all four attempts, offline distillation is best at nineteen point two, and ActionOPD's fourteen point two is also below OPD's seventeen point five. The authors say plainly that this doesn't establish uniform superiority.

My take: the real advantage of ActionOPD isn't the score, it's the interface. It only needs the teacher to generate answers, which means a closed model can be the teacher, though the paper itself didn't test one.

The ablations are in Table 3. Keep the semantic scores for picking which action to run, but train on a reward that only checks whether the action is valid, and test single-attempt success drops to thirty point eight, the lowest of the trained variants.

That's because every valid candidate gets the same reward, so the model can't learn which action is better. In this ablation, comparing what actions mean made the biggest difference of any component they removed.

The paper also picks out a few cases. In the airline domain, a user wants to change a basic economy flight, which policy forbids. Before training, the model sometimes changes it anyway, in two of four trials. After training with the DeepSeek teacher, it explains the rule and calls the transfer tool when the user asks for escalation.

Finally, an extension called Self-ActionOPD. Teacher and student are the same model, except the teacher also sees privileged information, like the user's goals and reference actions, that the student can't see.

After fifty steps, the student, using only public information, improves test single-attempt success by four point eight percentage points.

### Limitations

The authors list three limitations.

First, agreeing with the teacher isn't the same as being right. If the teacher makes a mistake, the student learns the mistake.

Second, using a language model to judge semantic agreement can itself be wrong, with false positives and false negatives, so rewards can be misassigned.

Third, there are only three teacher samples and four student candidates per state. When many reasonable decisions exist, as can happen in long-horizon tasks, there may be few matches, and the signal gets weak.

### Future work

Looking ahead, the authors note that more teacher and student samples could help, at a higher cost, but that's untested.

They also see Self-ActionOPD as a possible path toward agents improving themselves with privileged information. Code, data and models are promised on GitHub.

My take: what this paper changes is the unit of supervision. Moving from aligning tokens to aligning decisions could make it much easier to use closed frontier models to train small open agents.

But keep the gap in view. The best 4B student averages thirty-seven point seven, while its teacher scores seventy-one point two. Distillation narrows the gap; it's nowhere near replacing the teacher.

That's today's Daily Frontier Papers. The link to the paper is in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
