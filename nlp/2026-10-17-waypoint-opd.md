# Waypoint-OPD: Transferring RL-Induced Policy Shifts via Gated On-Policy Distillation

**4B老师教出超过它的学生**

- 论文 Paper: [arXiv 2610.21601](https://arxiv.org/abs/2610.21601)
- 单位 Affiliation: 阿里云
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-17
- 视频 Videos: [YouTube 中文](https://youtu.be/HeU54BvYnX4) · [YouTube English](https://youtu.be/xsnf7WVH4pA) · B站「每日前沿文章」

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇阿里云的论文，叫 Waypoint-OPD。Waypoint 是路标、途经点的意思，OPD 是在线蒸馏。

它要解决的问题是，一个模型用强化学习练出来的进步，怎么转移给别的模型。而且在这篇论文里，老师只有 4B，学生反而是 8B 到 30B 的更大模型。

论文第一页这张图就是结论。三个学生模型，在 7 个数学竞赛基准上的平均分，新方法比原来的 Direct-OPD，分别高了 2.18、1.59 和 1.89 个百分点。

### 问题

背景是这样的。强化学习，简称 RL，能提升模型的数学和代码推理，但它很贵，要反复生成答案，验证对错，再更新模型。

所以一个自然的想法是，老师模型花大力气做完 RL，能不能把这份进步复用到别的模型上，不用每个模型都重新练一遍。论文要回答的是，老师策略里那么多局部变化，到底哪些应该拿来指导学生。

### 以前的方法

最基本的做法叫在线蒸馏，on-policy distillation，简称 OPD。学生自己写答案，老师在学生写出来的每一个位置，给出下一个 token 的概率分布，学生去贴近老师。

问题是，这样学到的是老师的绝对水平，老师自身能力的上限，也一起被绑定了。

后来有了 Direct-OPD。它不学老师本身，而是学老师在 RL 前后的变化。对同一个 token，拿 RL 之后的老师给的对数概率，减去 RL 之前的老师给的，差值为正，说明 RL 让老师更偏好这个 token，这个差值就当作奖励给学生。

还有一种叫 OPD 平方，全名 On-Policy Delta Distillation，它用的是推理调优后的老师和它的基础模型之间的差别，只在两种信号符号一致时才保留。

但后面这两种方法都只看两个端点，训练前和训练后，或者老师和基础模型。中间发生了什么，它们看不到。

### 核心想法

作者的观察是，同样一个终点变化，可能来自两种完全不同的过程。

一种是前半段和后半段往同一个方向走，一路加强。另一种是前半段往一个方向走得很远，后半段又掉头往回拉，最后剩下的净变化，和第一种一模一样。

只看终点，这两种情况分不出来。作者的假设是，前后两段方向一致，这个变化就更值得传给学生，前后方向相反，就应该压下去。

怎么看到中间过程呢，办法很简单。从老师的同一次 RL 训练里，多留一个中途保存的 checkpoint，这就是题目里的 Waypoint，路标。

### 方法

具体流程在这张图里，一共四步。

第一步，学生自己生成答案。在每一个位置，取学生认为最可能的 16 个候选 token，按学生自己的概率给它们分配权重。

第二步，用三个冻结的老师给这些候选打分，分别是 RL 之前的 Initial，中途的 Waypoint，和最后的 Final。这样每个候选都有两段变化，前半段是 Waypoint 减 Initial，后半段是 Final 减 Waypoint。

这里有一个细节，叫中心化。如果所有候选的分数一起涨了同样多，候选之间的相对偏好其实没变，所以先减去加权平均，只保留相对变化。

第三步是这篇的核心，叫一致性门控。在每个位置，算前后两段变化的加权余弦相似度，也就是两个方向有多一致。方向相反，门就关上，这个位置的信号直接去掉，部分一致，就按一致的程度缩小。如果某一段几乎没有变化，方向没法定义，就保留原来的信号。

要注意，门只决定权重。真正传给学生的，仍然是从训练前到训练后的总变化，中间的 Waypoint 只当证据用，不当学习目标。

另外还有一步批量缩放，目的是让不同批次之间，门的平均强度保持稳定。每一批数据都乘一个系数，让缩放后门的平均值保持在 0.8 左右。论文附录的统计也显示，训练越往后，缩放之前门的平均值确实越小。

第四步，把这些加权后的信号当作每个候选的优势值，用 PPO 式的方法更新学生，同时加一个 KL 惩罚，不让学生离自己的初始模型太远。

成本方面，和 Direct-OPD 比，只多了一次冻结老师的打分，不需要训练新的老师，蒸馏时也不调用答案验证器。

### 实验和效果

实验设置。老师是 Qwen3-4B，数学和代码各做一次 GRPO 强化学习，训练 1200 步。Initial 是训练前，Waypoint 取第 600 步，Final 取第 1200 步。

学生是三个更大的模型，Qwen3-8B，Qwen3-14B，和混合专家架构的 Qwen3-30B-A3B，每个学生只训练 100 步。

数学评测用了 7 个竞赛基准，包括三届 AIME，三场 HMMT，以及 AMC 2023，每道题生成 32 次，取平均正确率。代码用 MBPP、MBPP+，和 LiveCodeBench 的三个版本。

数学结果在表 1。三个学生里，新方法的平均分都是所有蒸馏方法中最高的。比最强的基线 Direct-OPD，分别高 2.18、1.59 和 1.89 个百分点，21 项单项对比里，赢了 19 项。

有意思的是，老师自己的平均分是 56.13。8B 学生学完是 56.14，基本持平，14B 学生是 59.20，30B 学生是 57.85，都超过了老师。14B 学生在全部 7 个数学基准上都超过了老师。

代码结果在表 2。新方法比 Direct-OPD，平均分高了 2.66、1.52 和 1.19 个百分点，15 项单项对比全部提升。

训练过程也有记录。在 8B 学生的数学训练里，从第 60 步开始，新方法在图上三个基准的每一次评测，都领先 Direct-OPD。

还有一个出乎意料的结果，多了一次打分，训练反而更快。8B 数学训练 100 步，新方法用了 8 小时 2 分，Direct-OPD 用了 8 小时 21 分。论文同时观察到，新方法生成的回答在大多数步都更短，回答写到长度上限被截断的比例，从百分之 14.83 降到了百分之 11.97。

### 和以前方法的对比

再看消融实验，在表 3，都是在 8B 学生的数学任务上做的。把门全部打开，也就是不做一致性筛选，平均分从 56.14 掉到 53.50。

更说明问题的是打乱实验。门的数值分布完全不变，只是随机分给别的位置，平均分掉到 51.20，比完全不用门还低 2.30。这支持了作者的说法，每个门必须对上产生它的那个位置。

去掉批量缩放，平均分是 55.22，少了 0.92，但仍然比 Direct-OPD 高 1.26。作者也提醒，剩下的增益不能全部算到门控头上，因为中心化等其他差别也保留着。

中途的 checkpoint 选在哪一步，影响很小。换成第 500 步和第 700 步，平均分分别是 56.18 和 56.13，和第 600 步的 56.14 几乎一样。

和直接模仿老师的 OPD 比，差距更明显，8B 学生用 OPD，数学平均分只有 51.04。OPD 平方在少数代码单项上更高，比如 14B 学生的 LiveCodeBench v5，但每个学生的代码平均分，仍然是新方法最高。

我的判断是，这篇的思路很巧。它不需要更强的老师，也不需要更多数据，只是多看了一眼老师训练的中途，就拿到了终点差值里没有的信息。

我的判断是，提升幅度本身不算大，大约两个百分点左右，而且所有实验的老师都是 Qwen3-4B，学生也都来自同一个 Qwen3 家族。

### 局限

先说论文自己写明的几点。前后方向一致，不等于这个变化就是对的，一致性只是一种加权依据，不保证监督一定有益。

第二，方法要求老师和学生共用同一个分词器，才能直接比较同一个 token 的概率。

第三，它需要三个来自同一次 RL 训练的老师 checkpoint，训练前、中途和训练后都要有。

第四，消融实验只在 8B 学生的数学任务上做过，而且作者也说，这些实验没有单独分离出中心化的作用。

另外，挑选学生 checkpoint 用的验证集是 AIME 2024，而 AIME 2024 同时也在报告的成绩表里。论文也没有报告多次重复训练的波动范围。

### 未来方向

论文结论里说，这些结果说明，老师训练中途的 checkpoint，可以为学生学习时怎样给局部变化加权，提供证据。代码已经在 GitHub 公开。

作者在引言里也点出一个更大的方向，复用老师的 RL 成果，可以把一次昂贵的训练，分摊给多个目标模型。这篇里，一个 4B 老师的 RL 成果，就转移给了从 8B 到 30B 的三个学生。

我的判断是，接下来值得看两点。一是只用一个中途点够不够，多留几个 checkpoint，会不会给出更细的判断。二是换成更大的老师，或者跨模型家族，这个门控还灵不灵。

我的判断是，如果你在做 RL 训练，这篇给了一个很便宜的提醒，训练中途的 checkpoint 先别删，后面做蒸馏的时候，它们可能有用。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Alibaba Cloud, and it's called Waypoint-OPD. A waypoint is a marker along a route, and OPD stands for on-policy distillation.

The question is how to take the gains one model got from reinforcement learning and hand them to other models. And here's the twist: the teacher is only a 4B model, while the students are bigger, from 8B up to 30B.

The first figure is the headline. Across three student models, averaged over seven math competition benchmarks, the new method beats the previous approach, Direct-OPD, by two point one eight, one point five nine, and one point eight nine percentage points.

### Problem

Some background. Reinforcement learning, or RL, can improve a model's math and code reasoning. But it's expensive: you generate answers over and over, check them, and update the model.

So the natural idea is to reuse it. If a teacher has already paid for RL, can other models inherit that improvement without being trained from scratch? The paper's question is which of the many local changes in the teacher's policy should actually guide the student.

### Prior work

The baseline is on-policy distillation, or OPD. The student writes its own answer, and at every position the teacher supplies its distribution over the next token, which the student tries to match.

The catch is that the student learns the teacher's absolute behavior, so it's tied to the teacher's own limits.

Direct-OPD changes the target. Instead of copying the teacher, it copies how RL changed the teacher. For each token, take the log probability under the post-RL teacher minus the log probability under the pre-RL teacher. A positive gap means RL made the teacher prefer that token, and the gap becomes the student's reward.

A third method, OPD squared, short for On-Policy Delta Distillation, uses the difference between a reasoning-tuned teacher and its base model, and keeps a token's signal only when two signals agree in sign.

But these last two look at two endpoints only: before and after training, or teacher and base model. They can't see what happened in between.

### Key idea

The authors' observation is that the same net change can come from two very different histories.

In one, the first half and the second half of training push in the same direction, and the change builds up. In the other, the first half overshoots in one direction and the second half pulls back, leaving exactly the same net change.

Looking only at the endpoints, you can't tell these apart. The authors' hypothesis is that changes where the two halves agree are worth passing on, and changes where they reverse should be suppressed.

How do you see the middle? Simple: keep one extra checkpoint from partway through the same teacher RL run. That's the waypoint in the title.

### Method

The pipeline is in this figure, in four steps.

Step one, the student generates its own answer. At every position, take the sixteen tokens the student finds most likely, weighted by the student's own probabilities.

Step two, score those candidates with three frozen teachers: Initial before RL, Waypoint in the middle, and Final at the end. Now every candidate has two shifts, an early one from Initial to Waypoint and a late one from Waypoint to Final.

There's a detail here called centering. If every candidate's score goes up by the same amount, the relative preferences haven't changed at all, so the weighted mean is subtracted first and only relative changes remain.

Step three is the core, an agreement gate. At each position, compute a weighted cosine similarity between the early and late shifts, which measures how well their directions agree. If they point in opposite directions, the gate closes and that position's signal is dropped. If they partly agree, the signal is scaled down accordingly. And if one shift is too small to have a direction, the original signal is kept.

Note that the gate only sets the weight. What gets transferred is still the full change from before to after training. The waypoint is evidence, not something to imitate.

There's also a batch scaling step, meant to keep the average gate strength steady from batch to batch. Each batch is multiplied by a factor so the average scaled gate stays around zero point eight. The appendix shows that the average gate before scaling does shrink as training goes on.

Step four, these weighted signals become per-candidate advantages for a PPO-style update of the student, with a KL penalty that keeps it close to its own starting point.

On cost: compared with Direct-OPD, this adds one more scoring pass with a frozen teacher. No new teacher training, and no answer verifier during distillation.

### Results

The setup. The teacher is Qwen3-4B, trained with GRPO reinforcement learning for twelve hundred steps, once for math and once for code. Initial is before training, Waypoint is step six hundred, and Final is step twelve hundred.

The students are three larger models: Qwen3-8B, Qwen3-14B, and the mixture-of-experts Qwen3-30B-A3B. Each student trains for just one hundred steps.

Math is evaluated on seven competition benchmarks, three AIME exams, three HMMT rounds, and AMC 2023, averaging correctness over thirty-two attempts per problem. Code uses MBPP, MBPP plus, and three versions of LiveCodeBench.

The math results are in Table 1. For all three students, the new method has the highest average among the distillation methods. It beats the strongest baseline, Direct-OPD, by two point one eight, one point five nine, and one point eight nine points, and wins nineteen of twenty-one individual comparisons.

Here's the interesting part. The teacher itself averages fifty-six point one three. The 8B student lands at fifty-six point one four, essentially a tie, while the 14B student reaches fifty-nine point two and the 30B student fifty-seven point eight five, both above the teacher. The 14B student beats the teacher on all seven math benchmarks.

The code results are in Table 2. The new method beats Direct-OPD on average by two point six six, one point five two, and one point one nine points, and improves all fifteen individual comparisons.

Training curves were recorded too. For the 8B student on math, from step sixty onward the new method leads Direct-OPD at every evaluation on all three benchmarks shown.

And a surprise: despite the extra scoring pass, training was faster. One hundred steps of 8B math took eight hours and two minutes, versus eight hours and twenty-one minutes for Direct-OPD. The paper also observes shorter responses at most steps, with the share of responses cut off at the length limit dropping from fourteen point eight three percent to eleven point nine seven percent.

### Comparison

Now the ablations, in Table 3, all on the 8B student in math. Open every gate, meaning no agreement filtering, and the average drops from fifty-six point one four to fifty-three point five.

The shuffle test is more telling. Keep exactly the same gate values but assign them to random positions, and the average falls to fifty-one point two, two point three points below using no gate at all. That supports the authors' point that each gate has to stay with the position that produced it.

Remove batch scaling and the average is fifty-five point two two, down zero point nine two, but still one point two six above Direct-OPD. The authors caution that the remaining gain can't all be credited to gating, since centering and other differences are still in place.

Where you put the waypoint barely matters. Moving it to step five hundred or step seven hundred gives fifty-six point one eight and fifty-six point one three, nearly identical to fifty-six point one four at step six hundred.

Against plain OPD, which imitates the teacher directly, the gap is wider: the 8B student scores only fifty-one point zero four in math. OPD squared does win a few individual code columns, like the 14B student on LiveCodeBench v5, but the new method keeps the best code average for every student.

My read is that the idea is clever. It needs no stronger teacher and no extra data, just one look at the middle of the teacher's training, and that reveals information the endpoint difference simply doesn't contain.

My read is also that the gains are modest, around two points, and every experiment uses Qwen3-4B teachers, with students from the same Qwen3 family.

### Limitations

Start with what the paper itself states. Agreement between the two halves doesn't make a change correct. It's a weighting criterion, not a guarantee that the supervision helps.

Second, teacher and student must share a tokenizer, so that probabilities for the same token can be compared directly.

Third, you need three teacher checkpoints from the same RL run: before, during, and after training.

Fourth, the ablations were run only on the 8B student in math, and the authors note they don't isolate the effect of centering.

Also, student checkpoints were selected using AIME 2024 as the validation set, and AIME 2024 also appears in the reported results. And the paper doesn't report variation across repeated training runs.

### Future work

The paper concludes that intermediate teacher checkpoints provide evidence for how to weight local policy changes while the student learns. The code is already public on GitHub.

The introduction also points to a bigger theme: reusing a teacher's RL gains spreads one expensive training run across many target models. Here, a 4B teacher's RL gains carried over to three students from 8B to 30B.

My read is that two questions come next. Is one waypoint enough, or would more checkpoints give a finer signal? And does the gate still hold up with a bigger teacher, or across model families?

My read for practitioners: if you run RL, this is a cheap reminder not to delete your intermediate checkpoints. They may be useful for distillation later.

That's today's Daily Frontier Papers. The paper is linked in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
