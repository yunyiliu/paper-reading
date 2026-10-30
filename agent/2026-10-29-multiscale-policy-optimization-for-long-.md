# Multiscale Policy Optimization for Long-Horizon LLM Agents

**最后一个分数该算在谁头上**

- 论文 Paper: [arXiv 2610.35013](https://arxiv.org/abs/2610.35013)
- 单位 Affiliation: 北京大学、IQuest Research 等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-29
- 视频 Videos: [YouTube 中文](https://youtu.be/VWz0WpGVfiI) · [YouTube English](https://youtu.be/uGS2IQsmpc0) · [B站](https://www.bilibili.com/video/BV1wTxP6LEKV)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇北京大学和 IQuest Research 等机构的论文，题目是 Multiscale Policy Optimization，多尺度策略优化，简称 MPO。

它要解决的问题是，用强化学习训练长流程的大模型 Agent，任务结束时只有一个分数，这个分数到底该算在谁头上。

### 问题

先看 Agent 怎么和环境打交道。它一轮一轮地行动，每轮输出一个完整的东西，比如一条搜索查询、一次工具调用，或者最后的回答，环境给出反馈，再进入下一轮。

但模型写这一轮输出的时候，是一个 token 一个 token 往外生成的。所以这里同时有两个尺度，一个是交互的轮次，一个是生成的 token。

训练时通常只有任务结束时的一个奖励。论文的设定里，只有整条轨迹的最后一个 token 拿到这个奖励，之前所有 token 的奖励都是零。

难点就在这里。这个分数既要分到每一轮交互上，看哪一轮帮了忙、哪一轮出了错，又要在一轮里面，分清楚哪些 token 更关键。

### 以前的方法

现在常用的方法分两类。第一类不用价值模型，比如 GRPO、DAPO、GSPO，同一个问题采样一组回答，按组内的相对好坏，给每条完整输出一个统一的分数。

问题是，整条轨迹里所有轮次、所有 token，拿到的是同一个分数，分不出谁的贡献大。

第二类用价值模型，比如 PPO、VAPO、SAO。价值模型也叫 critic，它估计从当前位置出发，最后大概能拿多少奖励，再据此给每个 token 单独算优势，也就是这一步比预期好多少。

它们把每个 token 都当成一个动作，信用分得更细，但没有把一轮内部的生成，和轮与轮之间的环境交互区分开。

### 核心想法

作者先做了一个观察。在这张图的两个例子里，训练中 critic 的打分，在一轮之内比较平稳，到了一些轮次的边界，会明显跳一下。这两个例子，一条是失败的轨迹，一条是成功的轨迹。

他们统计了一次搜索 Agent 训练里的 2048 条轨迹、16323 个轮次。论文报告，轮次平均值之间的差异，中位数能解释 critic 打分方差的 89.5%，在 93.8% 的轨迹里，轮与轮之间的变化大过一轮之内的变化。

所以核心想法是，以轮次为主轴，去传递长距离的信用，同时在一轮里面，保留 token 之间的细微差别。

作者还给了理论上的理由。把一整轮输出看成一个大动作，可以推出一个轮次级别的策略改进下界。如果改用 token 级别的近似，在同样的 KL 约束下，这个下界会多出一项惩罚，来自一轮内部的分布偏移。

### 方法

MPO 有两个部件，这张总览图都画出来了。第一个叫多尺度优势估计，英文缩写 MAE，第二个是轮次级别的重要性采样。

先说 MAE 的第一步，跨轮传递信用。把一轮里所有 token 的时间差分误差加起来，中间的 critic 打分会互相抵消，只剩下这一轮开头和下一轮开头的打分之差，最后一轮再加上奖励。

然后在轮次这条时间轴上做 GAE，也就是广义优势估计。这样信用每经过一轮衰减一次，而不是每个 token 衰减一次，也不受每轮长短和轮内打分的影响。

第二步，在一轮里面分配信用。只在当前这一轮内部，按 token 再做一次局部的优势估计。

两者怎么合起来，作者提了两条要求。第一，这一轮的平均信用，必须等于跨轮算出来的那个值。第二，在这个前提下，尽量保留局部估计里 token 之间的相对差别。

写成一个带约束的最小二乘问题，解出来很简单。每个 token 的优势，等于这一轮的轮次优势，加上一个系数 beta，乘以它的局部优势减去本轮局部优势的平均值。

因为减了平均值，这一轮的平均信用不变，只是在 token 之间拉开差距。beta 取零，就退化成一轮里所有 token 拿同一个分数。

第二个部件是重要性采样。PPO 一类的方法，更新时要用新旧策略的概率比来做修正和裁剪。MPO 用的是整轮输出的概率比，也就是这一轮所有 token 概率比的乘积。

但这里有个坑。如果直接把每个 token 的优势都乘上同一个整轮概率比，刚才减平均值得到的局部修正，在求和的时候会被抵消掉。

作者的办法是 stop-gradient。前向计算时，每个 token 都用整轮的概率比，求梯度时，只对自己求导。这样整轮概率比管加权和裁剪，每个 token 的优势管自己的梯度。

另外，critic 直接回归最终的任务奖励，这一点沿用了 VAPO 的做法。

### 实验和效果

实验用了两个模型，Qwen3-4B-Thinking 和 Qwen3-30B-A3B-Instruct，在三类任务上分别训练，搜索、Claw-Eval 和 Automation Bench。

搜索任务看三个基准，BrowseComp-Plus、GAIA 和 HLE。对比的基线有 GRPO、GSPO、PPO、VAPO 和 SAO。

结果在表 1。论文报告，在两种模型尺寸下，九项指标 MPO 全部是最高的。

具体看。4B 模型上，BrowseComp-Plus 的单次通过率，MPO 是 48.08，最强的基线 SAO 是 43.75，高 4.33 个百分点。GAIA 上，MPO 是 39.47，高 3.85 个百分点。

30B 模型上，BrowseComp-Plus 和 GAIA 分别高 3.59 和 3.80 个百分点。HLE 的提升就小很多，4B 高 0.66 个百分点，30B 只高 0.12 个百分点。

Automation Bench 上，所有方法的绝对分数都不高，4B 模型里，MPO 是 13.33，最好的基线是 12.06。

再把两个部件拆开看。保留 MAE，但把整轮概率比换回 token 概率比，所有分数都会下降，4B 上三个搜索基准的单次通过率，分别掉了 2.36、3.12 和 1.03 个百分点。

不过即使换成 token 概率比，MAE 在六项搜索指标上仍然全部超过 PPO。作者据此认为，信用分配和概率修正，各自都有贡献。

参数消融在表 2 和表 3。beta 取零，也就是一轮里所有 token 拿同一个分数，已经比用 token 级 GAE 的 PPO 好，但比完整的 MPO 低很多，BrowseComp-Plus 单次通过率掉了 6.28 个百分点。

beta 加到 1.5 和 2，分数又逐步下降。所以一轮之内的分配要有，但不能太强。

两个衰减系数也分开调。跨轮的系数取 0.8，大约相当于往回看五轮，轮内的系数取 0.95，大约相当于二十个 token。作者认为，交互和生成，本来就该用不同的传播范围。

多次采样的结果也一致。4B 模型采样 32 次的通过率，BrowseComp-Plus 是 75.90，GAIA 是 69.90，HLE 是 34.00，比 GRPO 和 PPO 里较强的那个，分别高 3.61、5.82 和 5.80 个百分点。30B 模型也有收益，但幅度要看基准和采样次数。

### 和以前方法的对比

和 GRPO 这类方法比，MPO 多训练一个 critic，换来的是能区分每一轮、每个 token 的贡献。和 PPO 这类方法比，它不再把每个 token 当成独立的动作，而是让信用和概率修正都以整轮为单位。

训练曲线上，MPO 的 critic 解释方差，在训练后期比 PPO 更高。但作者自己也说，这个指标是在各自的 rollout 上测的，不能证明 critic 真的更准了。

我的判断是，这篇最有价值的是那个观察。critic 的打分本来就按轮次分层，信用分配就该顺着这个结构走，而且改动不大，还是 PPO 的框架，换掉优势估计和概率比就行。

我的判断是，也要看到弱点。HLE 上的提升很小，30B 只有 0.12 个百分点，论文没有给出多次独立训练之间的波动范围，这么小的差距，很难和随机波动区分开。

### 局限

论文没有单独的局限章节，下面几条来自论文自己交代的设置。

第一，表里的结果，是从每次训练里挑出最好的 checkpoint，再离线评测五次取平均。训练曲线和参数消融，论文说每个设置只跑了一次。

第二，Claw-Eval 和 Automation Bench 的训练数据，是公司内部的专有数据，论文说具体细节不能公开。

第三，理论分析假设轮数是固定的。两种做法的理论比较，比的是最坏情况下的惩罚上界，作者说明，这不代表实际的偏差，也不代表实际能拿到的提升。

第四，实验只覆盖 4B 和 30B 两个尺寸、三类任务，而且每类任务是单独训练的。

### 未来方向

论文的结论是，优化长流程 Agent 时，交互和生成应该被当成两种不同的信用尺度来处理。

我的判断是，这个思路可以推到写代码的 Agent，一轮可能就是一次文件编辑或一次跑测试，但论文没有测这类任务，要看后续有没有人验证。

我的判断是，还有一点要留意，论文里没有提到公开代码。强化学习方法之间几个百分点的差距，没有代码，别人很难复现。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Peking University, IQuest Research and several other groups. It's called Multiscale Policy Optimization, or MPO.

The question behind it: when you train a long-horizon agent with reinforcement learning, and all you get is one score at the very end, who gets the credit?

### Problem

Start with how an agent works. It acts in turns. Each turn is one complete output, like a search query, a tool call, or the final answer, and then the environment responds and the next turn begins.

But inside each turn, the model writes that output one token at a time. So there are two scales at once: the turns of interaction, and the tokens of generation.

Training usually gives a single reward when the task ends. In this paper's setup, only the very last token of the whole trajectory receives it. Every earlier token gets zero.

That's the hard part. The score has to be traced back to individual turns, which one helped and which one went wrong, and then, within a turn, to the tokens that mattered most.

### Prior work

Today's methods fall into two camps. The first skips the value model. GRPO, DAPO and GSPO sample a group of answers to the same prompt and give each complete output one shared score, based on how it ranks in the group.

The problem is that every turn and every token in the trajectory gets that same number. There's no way to tell who actually contributed.

The second camp uses a value model, also called a critic. PPO, VAPO and SAO are examples. The critic predicts how much reward is still to come from each point, and that gives every token its own advantage, meaning how much better that step was than expected.

These methods treat each token as an action. That's finer credit, but it doesn't separate generation inside a turn from interaction with the environment between turns.

### Key idea

The authors start with an observation. In the two examples in this figure, the critic's values during training stay fairly flat within a turn, then jump at some of the boundaries between turns. One is a failed trajectory and one is a successful one.

They measured this across two thousand and forty-eight trajectories and sixteen thousand three hundred and twenty-three turns from one search-agent run. The paper reports that differences between turn averages explain a median of eighty-nine point five percent of the critic's value variance, and between-turn variation beats within-turn variation in ninety-three point eight percent of trajectories.

So the key idea is to carry long-range credit along the turns, while keeping the fine differences between tokens inside each turn.

There's a theory argument too. If you treat a whole turn as one big action, you can derive a policy-improvement bound at the turn level. Switch to a token-level approximation under the same KL constraint, and the bound picks up an extra penalty, caused by distribution shift inside the turn.

### Method

MPO has two parts, and this overview figure shows both. The first is Multiscale Advantage Estimation, or MAE. The second is importance sampling at the level of whole turns.

MAE's first step carries credit across turns. Add up the temporal-difference errors of all the tokens in a turn, and the critic's in-between values cancel out. What's left is the difference between the value at the start of this turn and the start of the next, plus the reward on the last turn.

Then run GAE, generalized advantage estimation, along that turn axis. Credit now decays once per turn instead of once per token, and it no longer depends on how long each turn is or on the values inside it.

The second step allocates credit inside a turn. It runs a separate, local advantage estimate over the tokens of just the current turn.

To combine the two, the authors set two requirements. First, the average credit in a turn must equal the cross-turn value. Second, subject to that, keep as much of the relative difference between tokens from the local estimate as possible.

Written as a constrained least-squares problem, the answer is simple. Each token's advantage is the turn's advantage, plus a coefficient beta times that token's local advantage minus the turn's average local advantage.

Because the local part is centered, the turn's average credit doesn't move. It only spreads the tokens apart. Set beta to zero and every token in the turn gets the same score.

The second part is importance sampling. PPO-style methods correct and clip their updates using the ratio between the new and old policy's probabilities. MPO uses the ratio for the whole turn, which is the product of all its token ratios.

There's a catch. If you multiply every token's advantage by the same whole-turn ratio, the centered local correction cancels out when you sum.

The fix is a stop-gradient. In the forward pass, every token uses the whole-turn ratio. In the backward pass, each token only takes the gradient through itself. So the turn ratio handles weighting and clipping, and each token's own advantage drives its own gradient.

One more detail: the critic regresses directly onto the final task reward, a choice borrowed from VAPO.

### Results

The experiments use two models, Qwen3-4B-Thinking and Qwen3-30B-A3B-Instruct, trained separately on three kinds of tasks: search, Claw-Eval, and Automation Bench.

Search is measured on three benchmarks, BrowseComp-Plus, GAIA and HLE. The baselines are GRPO, GSPO, PPO, VAPO and SAO.

The results are in Table 1. The paper reports that MPO has the top score on all nine metrics, at both model sizes.

Some numbers. On the 4B model, single-attempt pass rate on BrowseComp-Plus is forty-eight point zero eight for MPO, against forty-three point seven five for the strongest baseline, SAO. That's four point three three points. On GAIA, MPO reaches thirty-nine point four seven, three point eight five points ahead.

On the 30B model, the gains on BrowseComp-Plus and GAIA are three point five nine and three point eight points. HLE is a different story: just zero point six six points at 4B, and zero point one two at 30B.

On Automation Bench, every method scores low in absolute terms. At 4B, MPO gets thirteen point three three, and the best baseline gets twelve point zero six.

Now take the two parts apart. Keep MAE, but swap the whole-turn ratio back to per-token ratios, and every score goes down. At 4B, single-attempt pass rate on the three search benchmarks drops by two point three six, three point one two, and one point zero three points.

Even with per-token ratios, though, MAE still beats PPO on all six search metrics. The authors read this as both pieces, the credit assignment and the probability correction, contributing on their own.

The parameter sweeps are in Tables 2 and 3. With beta at zero, so every token in a turn shares one score, MPO already beats PPO with token-level GAE. But it falls well short of full MPO, dropping six point two eight points on BrowseComp-Plus.

Push beta up to one point five and then two, and scores fall again. So credit allocation inside a turn helps, but only in moderation.

The two decay rates are also tuned separately. The cross-turn rate of zero point eight looks back roughly five turns. The within-turn rate of zero point nine five covers roughly twenty tokens. The authors' point is that interaction and generation naturally need different horizons.

Sampling more attempts tells the same story. With thirty-two attempts on the 4B model, MPO reaches seventy-five point nine on BrowseComp-Plus, sixty-nine point nine on GAIA, and thirty-four on HLE. That's three point six one, five point eight two, and five point eight points above the better of GRPO and PPO. The 30B model gains too, but how much depends on the benchmark and the number of attempts.

### Comparison

Compared with GRPO-style methods, MPO pays for an extra critic, and in return it can tell turns and tokens apart. Compared with PPO-style methods, it stops treating each token as its own action. Credit and probability correction both happen per turn, while differences between tokens inside a turn are kept.

On the training curves, MPO's critic shows higher explained variance than PPO's late in training. But the authors are careful here: that metric is measured on each policy's own rollouts, so it doesn't prove the critic is actually more accurate.

My read is that the most valuable thing here is the observation. The critic's values are already layered by turn, so credit assignment should follow that structure. And the change is small: it's still PPO, with a different advantage estimate and a different ratio.

My read is also that there's a weak spot. The HLE gains are tiny, zero point one two points at 30B, and the paper doesn't report variation across independent training runs. A gap that small is hard to separate from noise.

### Limitations

The paper has no dedicated limitations section, so these come from the setup it describes.

First, the tables report the best checkpoint from each training run, evaluated offline five times and averaged. For the training curves and the parameter sweeps, the paper says each setting was run once.

Second, the training data for Claw-Eval and Automation Bench is proprietary internal company data, and the paper says the details can't be disclosed.

Third, the theory assumes a fixed number of turns. And the theoretical comparison between the two approaches compares worst-case penalty bounds, which the authors note are not the actual biases or the improvements you'd actually get.

Fourth, the experiments cover only two model sizes, 4B and 30B, and three task types, each trained separately.

### Future work

The paper's conclusion is that interaction and generation should be treated as two distinct credit horizons when optimizing long-horizon agents.

My read is that this carries over naturally to coding agents, where one turn might be a single file edit or a test run. The paper doesn't test that, so it's worth watching whether someone does.

My read is that one more thing deserves attention: the paper doesn't mention releasing code. Gaps of a few points between reinforcement learning methods are hard for anyone else to reproduce without it.

That's today's Daily Frontier Papers. The paper is linked in the description.

