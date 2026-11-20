# Learning What Actions Change: Action Effect Matching for Dynamics-Faithful World Models

**动作差一点世界模型看懂了吗**

- 论文 Paper: [arXiv 2611.25980](https://arxiv.org/abs/2611.25980)
- 单位 Affiliation: KAIST · RLWRLD
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-20
- 视频 Videos: [YouTube 中文](https://youtu.be/-YFAY7lidZg) · [YouTube English](https://youtu.be/-ncUOnMDJwY) · [B站](https://www.bilibili.com/video/BV1b9ypz6EPe)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 KAIST 的论文，合作单位是 RLWRLD 公司，题目叫学习动作改变了什么，方法名叫 AFFECT。

一句话说，它想让机器人的世界模型真正看懂，动作差一点点，结果会差多少。

### 问题

先说背景。世界模型，就是给它当前画面和一串动作，它预测接下来会发生什么。现在机器人的世界模型，大多建立在视频扩散模型上，画面已经很逼真，前后也连贯。

大家想拿它评估策略、规划动作、生成训练数据。尤其是评估，不用每个策略都上真机器人，就能比较谁好谁坏。

问题在于，机械臂靠近物体的轨迹只差一点点，结果就可能完全不同，可能抓住了，可能抓空了，也可能把方块推走了。论文这张图画的就是这个。

而现有模型往往生成一段看似合理的视频，却忽略这些差别。动作不同，预测结果却差不多，拿它评估策略就不可靠，做规划也会选错动作。

### 以前的方法

作者先指出，单看画面质量不够。模型可能场景画得很好，手臂也动得像样，却画错了夹爪有没有碰到物体，任务成败也就判断错了。

最近也有一批工作在处理这个问题。一类要求能从生成视频里反推出动作，或者把不同动作的预测拉开。作者认为，光靠手臂动得不一样就能满足这类目标，不保证机器人和物体的交互是对的。

另一类直接监督结果，比如在仿真里重放偏离专家的动作，或者额外标注结果和奖励。问题是，这在真机器人上很难规模化。

### 核心想法

作者的思路是，把动作敏感性当成一个分布问题。不只要求模型把录下的那段未来复现得像，而是要求同一个场景里，动作变一变，预测的未来也像真实世界那样跟着变。

怎么知道真实世界怎么变。办法是采数据时主动给动作加噪声，并把噪声记下来。这样不需要从同一个初始状态反复执行，也不需要人工标注结果。

然后在这批数据上，加一个辅助训练目标，就是 AFFECT。它用的是冻结的编码器提出来的特征，不改模型结构，可以直接加到现有的世界模型上。

### 方法

第一步叫扰动玩耍数据。先用人类遥操作的玩耍数据训练一个玩耍策略，玩耍就是操作员随意摆弄桌上的东西，不做固定任务。然后让策略自己在机器人上跑，每段动作都叠加随时间相关的噪声。

为什么要随时间相关。附录解释，每步独立的白噪声在一段动作里会互相抵消，只让手臂原地抖。夹爪则不加噪声，因为那会产生任何策略都不会出现的失败。

采集也省人。物体留在上一次的位置，只有掉出工作区才要人捡回来。抓空、碰撞、碰倒都保留，因为这正是由动作决定、模型要学的结果。

第二步是怎么比较。这张总览图里，左边是采数据，中间两个固定的编码器，分别从视频和噪声里提特征，右边让真实视频和生成视频的动作效果对齐。

这些特征都是手工定的，很简单。噪声这边，比如一段时间里噪声的总和或平方和，视频这边，比如潜空间里某个通道的均值变了多少。编码器全程冻结，作者解释，如果一起训练，它输出常数就能把损失降到零，模型什么也学不到。

关键的一步叫中心化。把每个噪声特征减去它的期望值，让它围绕零上下波动，比预期大就是正，比预期小就是负。这样做是为了防止损失去监督背景这类跟动作无关的内容。

接下来定义动作效果，就是中心化后的噪声特征乘以视频特征，在很多段数据上取平均。比如推得越用力，物体被推得越远，这个平均值就是正的。AFFECT 要求生成视频和真实视频的动作效果一致，每一对特征都要对上。

有个细节。生成视频必须从全新的噪声出发，只看当前画面和动作，不能看真实的未来。否则模型不理会动作也能对上。因为这样生成比较贵，这个损失每 8 次迭代才算一次。

从统计上看，动作效果就是同一场景里，噪声和画面变化的协方差。所以只跟场景有关的误差，比如画面偏暗，会被抵消，损失只看动作带来的变化方向和强度。

论文举了个例子。真实物体往右移，模型却总画成往左移，只比平均特征，两边分布一样，看不出错。乘上噪声之后，符号反了，错误就暴露了。作者还说明，这个损失是一种有限特征的 MMD，也就是最大均值差异。

### 实验和效果

实验设置。一个仿真环境，是 OGBench 里的 UR5e 机械臂。两个真机器人，一个是 DROID 硬件配置的 Franka 机械臂，一个是双臂的 ALOHA-2。AFFECT 加在两个世界模型底座上，DreamDojo 和 Cosmos3。

数据量不大，附录里写，仿真用了大约 10 小时的扰动数据，两个真机器人的设置用了 5 小时。所有对照都用同样的数据和同样的训练预算，区别只在有没有这个辅助损失。

核心指标是相关性。让几个策略分别在真机器人上和世界模型里跑，看两边成功率对不对得上，用皮尔逊相关系数衡量。世界模型里每段视频的成败，由人来判定。

先看真机器人，在表 2。DROID 上，DreamDojo 的相关系数从 0.468 提到 0.790，Cosmos3 从 0.425 提到 0.583。FVD 衡量生成视频和真实视频的差距，越低越好，这两个模型分别下降了百分之二十三和百分之十八。

ALOHA-2 上，Cosmos3 的相关系数从 0.298 提到 0.694，是所有方法里最高的。DreamDojo 从 0.325 提到 0.490，不过它的 FVD 从 183 微升到了 188。

再看仿真，在表 1。两个底座的每一项指标都变好了。DreamDojo 的相关系数从 0.451 到 0.686，加了 AFFECT 之后，在所有对比方法里每一项都是最好。Cosmos3 从 0.091 提到 0.334。

然后是规划，在图 3。几个策略 checkpoint 各提候选动作，世界模型想象出每个候选的未来，再由训练好的成功率打分器，挑最高分的去执行。

UR5e 的 5 个抓取放置任务，平均成功率，随机挑是百分之四十六点七，用原来的 DreamDojo 挑是百分之六十，用加了 AFFECT 的挑是百分之六十八点三，多了 8.3 个百分点。ALOHA-2 是一个长任务，先打开垃圾桶，再扔掉桌上两样东西，平均多了 8.4 个百分点。

作者还检查了手工特征有没有看到动作效果。图 4 是热力图，在机械臂、方块和影子经过的区域，响应比随机打乱配对的基线强 1.6 到 2.2 倍，在静止的桌面和背景上，只是随机水平。

### 和以前方法的对比

基线有三个，自回归的 iVideoGPT，在它上面加了反事实一致性约束的 CoCo，作者只复现了其中一部分，还有专门评估策略的 Ctrl-World。在两个真机器人上，这三个基线的相关系数都在 0.4 以下。

真正的区别在训练目标。作者指出，以前那类目标，奖励从视频里认出动作，或者奖励不同动作的预测分得更开，反应越大越占便宜。AFFECT 瞄准的是真实反应本身，忽略动作、反应过头、方向反了，都会被罚。

消融在表 3。把噪声特征在不同片段之间打乱，或者干脆不乘噪声，只匹配平均的画面特征，结果都比只用原来的训练损失还差，FVD 从 284 涨到了 460 和 451。用正确的 AFFECT，是 256。

只用百分之二十的数据时，AFFECT 仍有提升，但幅度小了，FVD 从 405 降到 393。

我的判断是，这篇最有价值的是它盯住的问题。拿世界模型评估策略，要紧的不是画面多清晰，而是不同动作的后果画得对不对。AFFECT 用加噪声、记下来这种便宜办法，把这个信号放进了训练。

### 局限

论文正文没有单独的局限一节，下面几条来自附录和实验本身。

第一，损失只能约束特征看得见的东西。作者写道，如果视频特征主要盯着手臂，就主要监督手臂的运动，而不是物体的运动，任何特征都没察觉到的反应，就完全不受约束。

第二，只跟场景有关的错误，比如画面偏暗，AFFECT 管不了，还得靠原来的生成损失。而且两个损失要完全一致，需要当前画面和动作包含结果的全部信息，作者承认数据只是近似满足，因为玩耍策略看到的比世界模型多。

第三，评估规模不大。每种机器人只涉及 4 个策略 checkpoint 和 2 个任务，世界模型里的成败也是人工判定的。

第四，结果也不是全面领先。仿真里，Cosmos3 加了 AFFECT 之后，画面指标仍然明显不如 Ctrl-World。UR5e 的规划实验里，有一个任务加了 AFFECT，也只是和随机挑持平。

### 未来方向

论文没有专门写未来工作。作者在结论里说，规划实验的结果进一步表明，这些提升能够转化成更好的动作选择。

从数据量的对比，作者还推断，动作分布覆盖得越广，AFFECT 带来的收益越大。

我的判断是，这套做法最大的吸引力是便宜。不用重置场景，不用标注结果，也不用从同一状态反复执行，而且在两个结构差别很大的底座上都接上了。

我的判断是，也要看到边界。证据来自两个底座、三种机器人、每种几小时的数据，评估也只有少量策略和任务。要成为评估策略的常规做法，还需要更大规模的验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from KAIST and the company RLWRLD, and it's called Learning What Actions Change: Action Effect Matching for Dynamics-Faithful World Models. The method is named AFFECT.

In one line: it tries to make a robot's world model understand that a small change in the action can mean a very different outcome.

### Problem

Some background first. A world model takes the current frame and a sequence of actions, and predicts what happens next. Robotics world models today are mostly built on video diffusion models, and their videos already look photorealistic and temporally coherent.

People want to use them for a few things: evaluating policies, planning, and generating training data. Evaluation is the big one. You could compare policies without running every one of them on a real robot.

Here's the problem. A gripper's approach can change only slightly, and the outcome can be completely different. It grasps the cube, it misses, or it pushes the cube away. That's what this figure shows.

And current models tend to generate a plausible-looking video that ignores those differences. Different actions, roughly the same predicted outcome. That makes them unreliable for judging policies, and it leads planners to pick the wrong actions.

### Prior work

The authors' first point is that visual quality alone isn't enough. A model can keep the scene intact and render a convincing arm motion, yet get wrong whether the gripper touched the object or whether the object moved, and so get the task outcome wrong.

There's a growing line of work on this. One family of methods asks that the action be recoverable from the generated video, or pushes predictions under different actions apart. The authors' objection is that robot motion alone can satisfy those objectives, so they don't guarantee the robot and the object interact correctly.

Another family supervises outcomes directly, by replaying off-expert actions in simulation, or by adding outcome labels and rewards. The catch is that this is hard to scale on real robots.

### Key idea

The authors treat action sensitivity as a distributional problem. It's not enough for the model to reproduce the one recorded future. In a given scene, when the action shifts, the predicted future should shift the way the real world does.

So how do you learn how the real world shifts? You add noise to the actions while collecting data, and you log that noise. No repeated executions from the same starting state, and no outcome labels.

On top of that data sits an auxiliary training objective, which is AFFECT. It works on features from frozen encoders, needs no change to the model's architecture, and can be added to existing world models.

### Method

Step one is what they call perturbation play. A play policy is trained on human teleoperated play data, where an operator just fiddles with objects on the table instead of doing one fixed task. That policy then runs on the robot by itself, and every action chunk gets temporally correlated noise added to it.

Why correlated noise? The appendix explains that independent per-step noise averages out within a chunk, and only makes the arm jitter in place. The gripper isn't perturbed at all, because gripper noise creates failure modes no policy would ever produce.

Collection is cheap on human time, too. Objects stay wherever the last rollout left them, and a person only steps in when something leaves the workspace. Failed grasps, collisions and knocked-over objects are all kept, because those are exactly the action-dependent outcomes the model has to learn.

Step two is the comparison. In this overview, data collection is on the left. In the middle, two fixed encoders extract features, one from the video and one from the noise. On the right, the action effects of real and generated videos are pulled into alignment.

The features are simple and hand-specified. On the noise side, things like the sum of the noise over a window, or its sum of squares. On the video side, things like how much one channel's mean in the latent space changed. The encoders stay frozen. The authors point out that a trainable video encoder could just output a constant, drive the loss to zero, and teach the model nothing.

The key move is centering. Each noise feature has its expected value subtracted, so it swings around zero: positive when the perturbation was bigger than expected, negative when smaller. That keeps the loss from supervising things the action doesn't touch, like the background.

Then comes the action effect: the centered noise feature times the video feature, averaged over many windows. If pushing harder tends to move the object farther, that average comes out positive. AFFECT asks that the action effect measured on generated videos match the one measured on real videos, for every pair of features.

One detail matters here. The generated video has to start from fresh noise, conditioned only on the current frame and the actions, never on the real future. Otherwise the model could match the real video without paying attention to the action. Because that sampling is expensive, the loss is applied only once every eight iterations.

Statistically, this action effect is a covariance between the noise and the change in the video, within the same scene. So errors that depend only on the scene, like rendering everything a bit too dark, cancel out. The loss looks only at the direction and strength of the response to the action.

The paper gives an example. Suppose the real object moves right but the model always draws it moving left. Compare average features and the two distributions look identical, so you can't see the mistake. Multiply by the noise and the sign flips, which exposes it. The authors also show the loss is a finite-feature MMD, or maximum mean discrepancy.

### Results

The setup. One simulated environment, a UR5e arm from OGBench. Two real robots, a Franka arm in the DROID hardware configuration, and a bimanual ALOHA-2. AFFECT is added to two world-model backbones, DreamDojo and Cosmos3.

The data is modest: per the appendix, about ten hours of perturbed play data in simulation, and five hours for the two real-robot settings. Every comparison uses the same data and the same training budget, and differs only in the auxiliary loss.

The headline metric is correlation. Run several policies on the real robot and inside the world model, and check whether the success rates line up, using the Pearson correlation. Success inside the world model is labeled by human evaluators, video by video.

Real robots first, in Table 2. On DROID, DreamDojo's correlation rises from zero point four six eight to zero point seven nine zero, and Cosmos3 goes from zero point four two five to zero point five eight three. FVD, a measure of how far generated videos are from real ones, lower is better, drops by twenty-three and eighteen percent.

On ALOHA-2, Cosmos3's correlation goes from zero point two nine eight to zero point six nine four, the highest of any method. DreamDojo goes from zero point three two five to zero point four nine zero, though its FVD ticks up slightly, from one eighty-three to one eighty-eight.

Simulation, in Table 1. Both backbones improve on every metric. DreamDojo's correlation goes from zero point four five one to zero point six eight six, and with AFFECT it's the best of all compared methods on every column. Cosmos3 goes from zero point zero nine one to zero point three three four.

Then planning, in Figure 3. Several policy checkpoints each propose candidate actions, the world model imagines the future for each candidate, and a trained success estimator picks the highest-scoring one to execute.

On five UR5e pick-and-place tasks, average success is forty-six point seven percent when picking at random, sixty percent with plain DreamDojo, and sixty-eight point three percent with DreamDojo plus AFFECT, a gain of eight point three points. On ALOHA-2, a long task where the robot opens a trash bin and throws away two items, the average gain is eight point four points.

The authors also check whether the hand-made features actually see the action's effect. Figure 4 is a heat map. Where the arm, the cube and its shadow pass through, responses are one point six to two point two times stronger than a baseline with shuffled pairings. On the static table and backdrop, they stay at chance.

### Comparison

There are three baselines: iVideoGPT, an autoregressive model; CoCo, which adds counterfactual-consistency constraints on top of iVideoGPT; and Ctrl-World, built for policy evaluation. CoCo was reimplemented by the authors from its paper, and only partly. On both real robots, all three baselines stay below zero point four in correlation.

The real difference is the objective. The authors note that earlier objectives reward being able to recover the action from the video, or reward separating predictions under different actions, so a bigger response always helps them. AFFECT targets the real response itself. Ignoring the action, overreacting, or responding in the wrong direction are all penalized.

The ablations are in Table 3. Shuffle the noise features across episodes, or drop the noise entirely and just match average video features, and results get worse than training with the original loss alone. FVD goes from two eighty-four up to four sixty and four fifty-one. The real AFFECT gets two fifty-six.

With only twenty percent of the data, AFFECT still helps, but less: FVD goes from four hundred five to three ninety-three.

My read is that the paper's real contribution is the question it focuses on. When you evaluate policies in a world model, what matters isn't how sharp the video is. It's whether the consequences of different actions come out right. AFFECT gets that signal into training cheaply: add noise, and write it down.

### Limitations

The paper has no dedicated limitations section, so these come from the appendix and the experiments themselves.

First, the loss only constrains what its features can see. As the authors put it, a video feature that mostly tracks the arm will mostly supervise arm motion rather than object motion, and a response that no feature detects isn't constrained at all.

Second, errors that depend only on the scene, like a too-dark render, are outside AFFECT's reach and stay the job of the original generation loss. And in theory the two losses fully agree only if the current frame and actions carry all the information about the outcome, which the authors say their data meet only approximately, because the play policy sees more than the world model does.

Third, the evaluation is small. Each robot involves just four policy checkpoints and two tasks, and success inside the world model is judged by people.

Fourth, it doesn't win everywhere. In simulation, Cosmos3 with AFFECT still trails Ctrl-World clearly on video quality. And in UR5e planning, one task with AFFECT only ties random selection.

### Future work

The paper doesn't have a future work section. In the conclusion, the authors say the planning results further suggest these gains can translate into better action selection.

From the data-size comparison, they also infer that AFFECT's gains grow as the actions cover more of the space.

My read is that the big appeal here is cost. No scene resets, no outcome labels, no repeated runs from the same state, and the paper plugs it into two backbones that are architecturally very different.

My read is also that the evidence has limits. It comes from two backbones, three robots, a few hours of data each, and an evaluation with only a handful of policies and tasks. Before this becomes standard practice for policy evaluation, it needs larger-scale tests on more tasks.

That's today's Daily Frontier Papers. The paper is linked in the description.

