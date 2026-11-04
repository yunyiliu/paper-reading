# Order-Only On-Policy Distillation

**小模型只学老师的排序**

- 论文 Paper: [arXiv 2611.02112](https://arxiv.org/abs/2611.02112)
- 单位 Affiliation: 孟菲斯大学、密苏里大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-04
- 视频 Videos: [YouTube 中文](https://youtu.be/tn80AhxfTjY) · [YouTube English](https://youtu.be/LYuNjecyLW8) · [B站](https://www.bilibili.com/video/BV12AxQ6pEJ5)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇孟菲斯大学和密苏里大学的论文，题目叫 Order-Only On-Policy Distillation，简称 O3PD，可以理解成，只学排序的在线蒸馏。

它问了一个很朴素的问题，大模型教小模型的时候，小模型到底该学老师的什么，是每个词的具体概率，还是只要知道老师更偏向哪个词，就够了。

### 问题

先说蒸馏。蒸馏，就是拿一个强的大模型当老师，把本事教给一个弱一些的小模型。2015 年 Hinton 等人的工作说明，老师给出的完整概率分布，比单个标准答案更有用，因为它还告诉学生，其他候选有多大可能。

从那以后，让学生去拟合老师的概率分布，就成了蒸馏的默认做法。

但作者认为，老师的概率里混着两种信息。一种是老师更偏向哪些词，另一种是老师对这个偏好有多自信。前一种是学生要学会推理必须掌握的，后一种反映的是老师自己的能力和校准，小得多的学生，可能根本学不来。

实际里也能看到问题。在线蒸馏对老师的温度非常敏感，最合适的温度，每一对师生都不一样，而且训练久了，学生还可能退步，掉到训练前的水平以下。这里的温度，是用来把概率分布调尖或者调平的一个参数。

### 以前的方法

以前主要有两类做法。第一类是序列级蒸馏，让老师写出完整回答，学生做监督微调，照着学。

它有两个弱点。每个位置只给一个目标词，老师对其他候选的偏好就丢了。而且学生训练时看的是老师写的前文，自己生成时看的是自己写的前文，一旦出错，就走进了没练过的局面。

第二类是在线蒸馏，英文是 on-policy distillation，简称 OPD。让学生自己生成，老师在学生写出的每个位置上给出概率分布，学生去对齐这个分布。它现在已经是蒸馏推理模型的标准做法之一。

论文这张图，把老师能给的监督分成三层。最少的一层，只给一个词，最多的一层，给完整的概率，中间一层，是一个排好序的候选列表。以前的方法，要么在最底层，要么在最顶层。

### 核心想法

作者的主张是，真正值得传给学生的，是中间这一层，也就是老师对候选词的排序。剩下的具体概率数值，与其说是信号，不如说是一种约束。

这样做还有一个顺带的好处。logits，就是模型给每个词打的原始分数，把它除以任何一个正的温度，排序都不会变。所以只学排序，天然就和温度无关，也就不用再调温度了。

### 方法

具体怎么做，看这张总览图。学生先自己生成一段回答。在每个位置，把学生写到这里的前文交给老师，老师选出它认为最可能的前 K 个词，按概率从高到低排好。

然后学生用一个相邻排序损失来学。对老师排在第 i 位和第 i 加 1 位的两个词，要求学生给前一个打更高的分。老师的概率只用来挑出候选、排出顺序，它的数值从来不进入损失函数。

为什么只比相邻的两个。因为相邻的 K 减 1 对都排对了，靠传递性，整个顺序也就排对了。这是描述一个排序最少的约束，而两两全比，要比 K 乘 K 减 1 再除以 2 对。

每一对的损失，用的是经典的 Bradley-Terry 模型，只看学生给这两个词的分数差。排反了，梯度最大，排对以后，梯度慢慢变小。它鼓励拉开差距，但不规定差距要多大。而对齐概率的做法，要学生把老师的概率复现得一模一样，梯度才会消失。

论文还强调，只学排序是一个原则，不绑定某一个损失。全部两两比较、按名次加权比较，或者 ListMLE 这种整体排序损失，用的都是同样的监督信息，后面实验里都测了。

### 实验和效果

实验设置。模型全部来自 Qwen3 家族，学生有 1.7B、4B、8B 三个尺寸，每个学生配上所有比它大的老师，老师有 4B、8B、14B，一共六组。训练数据是 OpenThoughts 里的数学推理部分。

评测用三个竞赛级数学基准，HMMT25、AIME24 和 AIME25，每道题采样 12 次，取平均准确率。主要对手是 OPD，它和 O3PD 的配置完全一样，都看老师的前 8 个候选，唯一的区别，就是概率数值进不进损失。

结果在表 3。六组配置里，O3PD 的三项平均分全部高于 OPD。8B 学生只高 0.6 分，小一些的学生提升更大，4B 学生分别高 2.3 和 1.9 分，1.7B 学生配 8B 和 4B 老师，分别高 2.6 和 2.9 分。

例外是 14B 老师教 1.7B 学生，只高 0.3 分。作者的解释是，师生差距最大的时候，连老师的排序都更难学。

提升主要集中在 AIME 上。AIME25 每一组都提高了，平均 3.4 分，最多 5.6 分。全部 18 项单项对比里，O3PD 赢了 14 项。

它也超过了监督微调 SFT 和强化学习方法 GRPO。比如 1.7B 学生，O3PD 平均 40.9 分，GRPO 是 37.7，SFT 是 35.8。要注意，SFT 和 GRPO 的数字引自另一篇论文，不是作者自己重跑的，不过那篇用的是同样的学生、数据和评测方案。

训练过程也更稳。这张图是 8B 老师教 1.7B 学生的训练曲线。OPD 在第 20 步就到顶，38.1 分，之后一路下滑，跌到训练前的 37.7 以下。O3PD 总体一直在涨，最后一步拿到最高的 40.7 分。

作者特意说明，表里报的是每种方法最好的检查点，这样选，其实对会退步的 OPD 更有利。在这一组上，如果都比最后一步，两者差距会比 2.6 分还大。

还有一个实用的结论，小老师就够了。1.7B 学生用 4B 老师加 O3PD，得到 40.9，比用 14B 老师加 OPD 的 39.1 还高。4B 学生也一样，8B 老师加 O3PD 是 63.2，14B 老师加 OPD 是 60.9。

温度问题有多严重，看附录这张热力图。OPD 的最佳温度从 0.25 到 2 都有，每对师生不一样。只训练 3 步，光是温度不同，1.7B 及以上的学生，HMMT25 就能差出 6.4 分。0.6B 的学生在高温下直接崩掉，14B 老师教 0.6B 学生，从温度为 1 时的 11.9 分，掉到温度为 8 时的 1.1 分。而 O3PD 在任何温度下，拿到的目标都是一样的。

### 和以前方法的对比

消融实验在表 4，都在 8B 老师教 1.7B 学生上做，训练 100 步。把损失换成全部两两比较、按名次加权，或者 ListMLE，连同默认的相邻比较，全都比 OPD 高 1.3 到 2.6 分。相邻比较和 ListMLE 最好，只差 0.1 分，而相邻比较只用了 28 对里的 7 对。

候选数 K 取 4、8、16，都比 OPD 好，8 最好，说明对 K 不太敏感。结合这两组结果，作者认为，好处来自排序监督本身，而不是某个特定的损失函数。

那学生到底学到了什么。这组图测的是师生在每个位置上的一致程度。看概率数值，OPD 和老师更接近，O3PD 离得更远，而且越训越远。但看排序和候选集合，情况反过来了，O3PD 的 Kendall 相关系数大部分时间更高，前 8 个候选的重合度几乎每次评估都更高，第一名的一致率两者差不多。

也就是说，概率对得更准，并没有换来更好的推理。OPD 和老师的分布距离更小，O3PD 的准确率更高。

再看这张图，它把每个检查点按离初始学生多远、离老师多远摆出来。对 1.7B 学生，OPD 几乎没离开起点，O3PD 走得远好几倍。4B 学生两者差不多，8B 学生反而是 OPD 走得更远。作者的解释是，小学生学不来大老师的概率，只学排序，就放开了它，去重塑自己的分布。

我的判断是，这篇论文的价值，在于把一个默认假设拆开了。从 Hinton 那篇开始，蒸馏几乎都默认要对齐概率，它用一个很简单的改动说明，至少在这个设置下，排序就够了，还顺手省掉了调温度这一步。

我的判断是，也要看清提升的幅度。平均分的差距在 0.3 到 2.9 分之间，每道题只采样 12 次，论文没有给出置信区间或者显著性检验，单项上也有掉分，比如 4B 老师教 1.7B 学生，HMMT25 反而低了 3.1 分。

### 局限

作者明确写到的局限主要是一条。实验只做了数学推理，没有评估这种蒸馏对安全对齐、或者开放式生成里其他行为的影响。

从实验设置还能看出几点。所有师生都来自 Qwen3 一个家族，最大的老师只有 14B，既没有测跨家族蒸馏，也没有测更大的模型。

训练也很短，只有 100 步，用 LoRA 微调，学生每次最多生成 1024 个 token，而且老师开着思考模式，学生是非思考模式。更长的训练、完整长度的推理下，结论是否还成立，论文没有回答。

还有，14B 老师教 1.7B 学生这组几乎没有提升，提示师生差距特别大的时候，这个方法的优势会变小。

### 未来方向

论文没有单独写未来方向，但结论里把只学排序当成一个蒸馏的通用原则，不绑定某一个损失函数。

我的判断是，接下来最值得看的有两件事。一是换到代码、Agent 这类任务，以及跨模型家族的蒸馏，结论还成不成立。二是它和 OPD 能不能互补，比如什么时候该传概率，什么时候只传排序。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the University of Memphis and the University of Missouri. It's called Order-Only On-Policy Distillation, or O3PD for short.

It asks a deceptively simple question. When a big model teaches a small one, what exactly should the student learn? The teacher's precise probability for every word, or just which words the teacher prefers?

### Problem

Quick background. Distillation means using a strong model as a teacher to train a weaker, smaller student. Back in 2015, Hinton and colleagues showed that the teacher's full output distribution is more useful than a single correct answer, because it also tells the student how plausible the alternatives are.

Ever since, training the student to match the teacher's probabilities has been the default recipe.

The authors argue those probabilities actually mix two things. One is which tokens the teacher prefers. The other is how confident the teacher is about that preference. The first is what a student needs to reason like the teacher. The second reflects the teacher's own capacity and calibration, and a much smaller student may simply be unable to reproduce it.

You can see the problem in practice. On-policy distillation is very sensitive to the teacher's temperature, and the best temperature differs from one teacher and student pair to the next. Worse, as training goes on, the student can get worse, even dropping below where it started. Temperature here is the knob that sharpens or flattens a probability distribution.

### Prior work

There are two main families of methods. The first is sequence-level distillation: the teacher writes full answers, and the student is fine-tuned to copy them.

That has two weaknesses. Each position gives the student just one target token, so the teacher's views on the alternatives are lost. And the student trains on the teacher's text but generates from its own, so once it makes a mistake, it wanders into situations it never practiced.

The second family is on-policy distillation, or OPD. The student generates its own text, the teacher gives a probability distribution at every position the student wrote, and the student is trained to match it. It has become a standard way to distill reasoning models.

This figure arranges teacher supervision into three levels. The least informative is a single chosen token. The most informative is the full probability distribution. In between sits a ranked list of candidates. Existing methods live at either the bottom or the top.

### Key idea

The authors' claim is that the middle level is what matters: the teacher's ranking of candidate tokens. The exact probability values on top of that, they argue, act more like a constraint than a signal.

There's a neat bonus. Logits are the raw scores a model gives each token, and dividing them by any positive temperature never changes their order. So a method that only learns the ranking is immune to temperature by design, and there's nothing left to tune.

### Method

Here's the overview. The student first generates a response. At every position, the text so far goes to the teacher, which picks its top K most likely next tokens and sorts them from most to least likely.

The student then learns with an adjacent ranking loss. For the tokens the teacher ranks i and i plus one, the student should score the first one higher. The teacher's probabilities are used only to pick and order the candidates. Their values never enter the loss.

Why only adjacent pairs? Because if all K minus one neighboring pairs are in the right order, transitivity puts the whole list in order. That's the smallest set of constraints that pins down a ranking, compared with K times K minus one over two pairs if you compare everything.

Each pair uses the classic Bradley-Terry model, which only looks at the gap between the student's scores for the two tokens. A reversed pair gets the biggest gradient, and once the order is right, the gradient fades. Bigger margins are still welcome, but there's no target margin. Probability matching, by contrast, only stops pushing when the student reproduces the teacher's probabilities exactly.

The paper stresses that order-only is a principle, not one specific loss. Comparing all pairs, weighting pairs by rank, or a listwise loss like ListMLE all use the same supervision, and all of them are tested later.

### Results

The setup. Every model is from the Qwen3 family. Students come in 1.7B, 4B and 8B, and each is paired with every larger teacher, 4B, 8B or 14B, for six pairs in total. Training data is the math reasoning subset of OpenThoughts.

Evaluation uses three competition-level math benchmarks, HMMT25, AIME24 and AIME25, averaging accuracy over twelve samples per problem. The main baseline is OPD, run with exactly the same configuration. Both methods see the teacher's top eight candidates. The only difference is whether the probability values enter the loss.

The results are in Table 3. O3PD beats OPD on the three-benchmark average in all six pairs. The 8B student gains just zero point six points. Smaller students gain more: two point three and one point nine for the 4B student, and two point six and two point nine for the 1.7B student with the 8B and 4B teachers.

The exception is the 14B teacher with the 1.7B student, which gains only zero point three. The authors suggest that when the size gap is largest, even the teacher's ranking gets hard to reproduce.

Most of the gains are on AIME. O3PD improves AIME25 in every pair, by three point four points on average and up to five point six. Across all eighteen individual comparisons, O3PD wins fourteen.

It also beats supervised fine-tuning and the reinforcement learning method GRPO. For the 1.7B student, O3PD reaches forty point nine, against thirty-seven point seven for GRPO and thirty-five point eight for SFT. One caveat: the SFT and GRPO numbers are taken from another paper rather than rerun, although that paper used the same students, data and evaluation protocol.

Training is also more stable. This figure shows training curves for the 8B teacher and 1.7B student. OPD peaks at step twenty with thirty-eight point one, then slides below the starting score of thirty-seven point seven. O3PD keeps climbing overall and hits its best, forty point seven, at the final step.

The authors point out that the table reports each method's best checkpoint, which actually favors OPD, since OPD declines. For this pair, comparing final checkpoints would make the gap even bigger than two point six points.

There's a practical takeaway too: a smaller teacher is enough. The 1.7B student with a 4B teacher and O3PD scores forty point nine, beating OPD with the 14B teacher at thirty-nine point one. Same for the 4B student: sixty-three point two with an 8B teacher and O3PD, versus sixty point nine with the 14B teacher and OPD.

How bad is the temperature problem? Look at this heat map from the appendix. OPD's best temperature ranges from zero point two five to two depending on the pair. After just three training steps, temperature alone moves HMMT25 accuracy by up to six point four points for students of 1.7B and up. A 0.6B student collapses at high temperature: with the 14B teacher, it falls from eleven point nine at temperature one to one point one at temperature eight. O3PD gets the exact same targets at every temperature.

### Comparison

The ablations are in Table 4, all on the 8B teacher and 1.7B student, trained for one hundred steps. Swap in all-pairs, rank-weighted, or ListMLE losses, and every variant, like the default adjacent loss, still beats OPD, by one point three to two point six points. Adjacent pairs and ListMLE come out on top, within zero point one of each other, even though the adjacent loss uses only seven of the twenty-eight pairs.

Candidate list sizes of four, eight and sixteen all beat OPD, with eight the best, so the method isn't fussy about K. Taken together, the authors conclude the benefit comes from ordinal supervision itself, not from any particular loss.

So what does the student actually learn? These plots measure teacher and student agreement position by position. On probability values, OPD stays closer to the teacher, and O3PD drifts further away as training goes on. But on ranking and candidate sets, it flips. O3PD has the higher Kendall correlation for most of training, its top-eight overlap is higher at nearly every evaluation, and agreement on the single top token is about the same.

In other words, matching the probabilities more closely did not buy better reasoning. OPD ends up closer to the teacher's distribution, but O3PD ends up more accurate.

This plot places each checkpoint by how far it moved from the starting student and how far it sits from the teacher. For the 1.7B student, OPD barely leaves the starting point, while O3PD moves several times further. For 4B the two are similar, and for 8B, OPD actually moves further. The authors' explanation: a small student can't reproduce a big teacher's probabilities, and learning only the order frees it to reshape its own distribution.

My read is that the paper's value is in pulling apart an assumption almost everyone made. Since Hinton's paper, distillation has largely meant matching probabilities. A very small change shows that, at least in this setting, the ranking is enough, and it removes temperature tuning along the way.

My read is also that the gains should be kept in proportion. Average improvements range from zero point three to two point nine points, with twelve samples per problem and no confidence intervals or significance tests reported. Some individual scores go down too, like HMMT25 for the 4B teacher and 1.7B student, which drops three point one points.

### Limitations

The one limitation the authors spell out is scope. The experiments cover only mathematical reasoning, so they don't evaluate how this kind of distillation affects safety alignment or other behavior in open-ended generation.

The setup itself suggests more. Every teacher and student is from the Qwen3 family, and the largest teacher is only 14B, so there's no cross-family distillation and no test with larger models.

Training is also short: one hundred steps of LoRA fine-tuning, with student rollouts capped at one thousand and twenty-four tokens, and the teacher in thinking mode while the student is not. Whether the result holds with longer training and full-length reasoning, the paper doesn't say.

And the 14B teacher with the 1.7B student barely improved, which suggests the advantage shrinks when the gap between teacher and student is very large.

### Future work

The paper has no separate future-work section, but its conclusion frames order-only as a general distillation principle, not tied to any single loss.

My read is that two questions matter most next. First, whether the result holds for code, for agents, and for distilling across model families. Second, whether it complements OPD, in other words, when it's worth passing on probabilities and when the ranking alone is better.

That's today's Daily Frontier Papers. The paper is linked in the description.

