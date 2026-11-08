# On the Generalization Limitation of On-Policy Distillation for Post-RL Consolidation

**蒸馏几乎满分本事没学全**

- 论文 Paper: [arXiv 2611.05609](https://arxiv.org/abs/2611.05609)
- 单位 Affiliation: 字节跳动等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-06
- 视频 Videos: [YouTube English](https://youtu.be/cK5CH2dZIVg) · [B站](https://www.bilibili.com/video/BV1Co4z6DEFo)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇字节跳动的论文，合作者来自芝加哥大学、杜克大学等高校，研究的是在线策略蒸馏，在强化学习之后做能力整合时，泛化上的局限。

一句话结论是，蒸馏在训练范围内几乎拿满分，不代表学生真的学全了老师的本事，训练范围之外的能力，可能悄悄丢了。

### 问题

先说背景。现在不少大模型的后训练流程是这样的，先针对数学、代码、Agent 这些能力，分别用强化学习训出几个专家模型，最后再把它们的本事，合并到一个模型里。论文说，大规模后训练流程里已经在用这一步。

整合想要的结果有三个，学到老师的提升，保住底座模型原有的能力，还能把这些能力用到训练没覆盖的任务上。

所以作者关心的是训练分布之外，也就是 OOD 的表现，并且分成两种。一种叫外推泛化，比如训练用的是简单数学题，测的是更难的竞赛题。另一种叫组合泛化，分别学了两种能力，看能不能在一个任务里一起用上。

### 以前的方法

合并用的主流方法叫 on-policy distillation，简称 OPD，在线策略蒸馏。学生自己生成回答，老师在学生写出的每个位置，给出它认为下一个 token 的分布，学生往老师的分布靠。它结合了强化学习在自己轨迹上学习的特点，和知识蒸馏逐个 token 的密集监督。

写成公式是这样，在学生自己生成的轨迹上，每个位置都算一次学生到老师的反向 KL，然后最小化。

多个老师的版本叫 MOPD，每个数据来源分配一个对应能力的老师，把几个 OPD 目标按比例加起来。

以前的工作报告，OPD 能很快、几乎完整地追上老师在训练分布内的表现。和这篇最接近的两项研究，一项诊断 MOPD 里的能力冲突，但能力都还在训练分布内，另一项看的是从数学到科学这种跨领域迁移，和这篇说的泛化形式不一样。

论文还拿 mixed RL 做对照，就是把几个任务的数据混在一起，各用各的奖励，直接做强化学习。

### 核心想法

作者的出发点来自 grokking 这个现象，模型在训练集上拟合之后很久，泛化能力才出现。所以 OPD 在训练范围内收敛得快，不能说明它也继承了老师的泛化能力，这两件事要分开测。

为了分清问题出在哪，作者设计了两种老师。一种是自 RL 专家老师，和学生同一个底座，各自用强化学习训出来。另一种是外部通才老师，比如更大的模型，通常不光在目标能力上，在别的能力上也比学生强。

### 方法

先做一个可控的小实验。随机初始化一个 1 亿参数的 Qwen2 小模型，用需要 2 到 10 步运算的数学题预训练，得到底座。老师从底座出发，在 11 到 14 步的题上做强化学习，学生从同一个底座出发，向老师做 OPD，最后用 15 到 17 步的题测 OOD。

正式实验的学生是 Qwen3-8B。数学用 GSM8K、MATH、DAPO 训练，在 GSM8K、MATH500、AMC23 和 AIME24、AIME25 上测。代码、工具调用和长上下文，也各有自己的训练集和测试集。

数学这边怎么划分训练范围内外，作者用了一个操作性的规则，三个 RL 老师都超过底座 97% 分数的基准，算训练范围内。这样 GSM8K 和 MATH500 是范围内，AMC23 和两个 AIME 是范围外。

组合泛化的测试，长上下文数学用 GSM-Infinite 和 DocMath，长上下文写代码用 LongBench-v2 Code，Agent 式编程用 SWE-bench Verified。

### 实验和效果

先看小实验，这张图。纵轴是蒸馏比例，就是学生相对底座的提升，占老师提升的多少。预训练范围内是 96% 到 104%，后训练范围内是 93% 到 96%，到了 OOD，掉到 76% 到 86%。

再看真实任务。用自 RL 专家老师做单老师 OPD，三个训练集平均，学生拿到老师训练范围内分数的 97.10%，但 OOD 分数比底座相对下降了 10.78%，也就是 8.56 个百分点。老师自己在 OOD 上也有遗忘，而且更严重，平均比底座低 11.43 个点。

多老师也是一样。MOPD 在所有组合上的 OOD 平均分，比底座低 14.07%，也就是 11.18 个点。

有人会说，学生的遗忘是从老师那里学来的。所以作者换成更强的外部老师，用 Qwen3-32B 教 Qwen3-8B。学生在训练范围内达到老师的 99.34%，只差 0.62 分，但 OOD 仍然比底座低 3.73%，也就是 2.96 个点。

接下来是组合泛化。三个设置平均，MOPD 在训练范围内比底座提高了 9.24%，但在组合任务上，比最好的老师低 23.82%。

代码加 Agent 这一组，SWE-bench Verified 从底座的 6.63 提到 8.05，比 mixed RL 的 7.79 高，但低于老师的 9.84。

最典型的是代码加长上下文，看这张图。MOPD 在三个组成基准上都提高了，可是长上下文写代码这个组合任务，LongBench-v2 Code 停在底座的 44.00，两个老师分别是 50.00 和 58.00。

数学加长上下文，结果也不一致。DocMath 从 55.38 提到 60.63，GSM-Infinite 反而从 50.88 降到 49.75，两项都低于最好的老师。

### 和以前方法的对比

和强化学习比，OPD 有一个好消息。在同样的设置下，OPD 比它的自 RL 老师少遗忘 25.09%，MOPD 比 mixed RL 少遗忘 28.92%，mixed RL 平均比底座低 15.73 个点。

组合任务上，mixed RL 也有差不多的差距，比最好的老师低 26.10%，MOPD 的组合任务平均分比 mixed RL 高 3.08%，也就是 1.07 个点。

为什么 OPD 遗忘得少，作者看了参数更新的形状，见表 1。OPD 的更新范数只有 RL 的 55.4%，0.154 对 0.278。更新能量里，前 64 个奇异方向占了 69.0%，RL 只有 39.9%，也就是说 OPD 的改动更小、更集中。

多老师的情况，MOPD 的更新有 11.0% 到 18.6% 能被单老师 OPD 的更新解释，mixed RL 这个比例不到 0.240%。作者强调，这只是初步的解释，更新几何和泛化之间的因果关系还没有建立。

那能不能补救。第一个办法，老师和数据一起放宽。还是 Qwen3-32B 当老师，训练数据换成 90% 的 GSM8K 加 10% 的 OpenR1，学生在五个基准上都超过了底座，AIME24 和 AIME25 的 pass@8 追平老师的 86.67 和 83.33，AMC23 到了 100，超过老师的 95。

第二个办法是训练后打补丁，叫 Procrustes-attention patching。在选定的注意力权重上，把 OPD 的更新和 RL 老师的更新做正交对齐，再减掉对不上的那部分。补丁强度取 0.4 时，AIME25 从 60.83 提到 65.00，离底座只差 0.83 个点，GSM8K 保持在 93.48。

第三个办法是保底回放。训练时混进一些额外的提示，由底座模型自己当老师，这里用的是代码题。和普通 OPD 比，八个 OOD 分数里有五个提高，AIME25 提高 6.67 个点，到 67.50，超过了底座和 RL 老师，GSM8K 只降了 0.18 个点。

我的判断是，这篇最有价值的地方，不是某个新方法，而是对评测习惯的提醒。用 OPD 合并专家之后，只看训练范围内的分数接近满分就宣布成功，是不够的，训练范围外和组合任务都要测。

### 局限

作者自己列了几条局限。主实验的学生只有 Qwen3-8B，最大的外部老师是 Qwen3-32B，学生容量、老师强度和模型家族的影响，还没有充分研究。

公开数据集方便复查，但比不上前沿模型真实后训练数据的广度。而且这里的 OOD，是相对后训练数据说的，不代表预训练没见过。

MOPD 和 mixed RL 都没有精调混合比例、奖励平衡和课程。受算力限制，也没有对每个实验重复多个随机种子，数学上只在 10% 的题目上用四个采样种子重复评测，这说明不了多次独立训练的稳定性。

另外，MOPD 比 mixed RL 好，并不是普遍规律。在一个 3 层 Transformer 的合成算术实验里，先做 SFT 预热再做强化学习的 mixed RL，在组合任务上反而超过了 MOPD。五个种子平均，它在两种组合上的准确率是 93% 和 90%，从同样先预热再强化学习的专家老师蒸馏出来的 MOPD 是 78% 和 74%。

### 未来方向

作者提出，更新几何和泛化的关系，需要做因果检验，在控制更新幅度、训练预算和训练范围内表现的前提下，去操纵更新方向。也可以在训练过程中，逐步投影或正则化参数更新，再在固定预算下，一起研究老师分配、数据覆盖和保底回放。

我的判断是，对做后训练的团队，这篇给的建议很实用，蒸馏数据别太窄，加一点更广的数据，或者回放底座自己的行为。但也要看到，三个补救办法，论文展示的结果都来自 GSM8K 这一个数学训练设置，能不能推广到代码和 Agent，还需要更多实验。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from ByteDance, with co-authors at the University of Chicago, Duke, UIUC and Northwestern. It looks at on-policy distillation when it's used to merge capabilities after reinforcement learning, and where its generalization falls short.

The one-line takeaway: a student that scores almost perfectly on what it was distilled on hasn't necessarily learned everything the teacher knows. What lies outside the training range can quietly get lost.

### Problem

Some context first. A common post-training recipe today goes like this. You train several specialist models with reinforcement learning, one for math, one for code, one for agents, and then you fold all of them into a single model. The paper notes this consolidation stage is already used in large-scale post-training pipelines to combine domain specialists.

What you want from that step is three things. Learn the teachers' improvements, keep what the base model could already do, and carry those skills to tasks the training never covered.

So the authors focus on out-of-distribution performance, OOD for short, and they split it two ways. Extrapolative generalization: train on easier math, test on harder competition problems. Compositional generalization: learn two skills separately, then see if the model can use them together in one task.

### Prior work

The standard merging tool is on-policy distillation, or OPD. The student generates its own answers, and at every position the teacher supplies its distribution over the next token, which the student moves toward. It combines reinforcement learning's habit of learning on its own trajectories with the dense, token-by-token signal of knowledge distillation.

As an equation, it's this. Along the student's own trajectories, compute the reverse KL from student to teacher at each position, and minimize it.

The multi-teacher version is called MOPD. Each data source gets a teacher with the matching skill, and the OPD objectives are mixed with fixed weights.

Earlier work reports that OPD catches up with the teacher in-distribution quickly and almost completely. The two closest studies differ from this one. One diagnoses capability conflicts in MOPD, but all inside the training distribution. The other looks at transfer across domains, like math to science, which is a different kind of generalization.

As a baseline, the paper uses mixed RL: pool the task data, keep each task's own reward, and run reinforcement learning directly.

### Key idea

The starting point is grokking, where a model fits its training set and only generalizes much later. So OPD converging fast in-distribution doesn't tell you the student inherited the teacher's generalization. Those are separate questions, and they need separate tests.

To tell the causes apart, the authors use two kinds of teacher. Self-RL specialist teachers share the student's base model, each trained with reinforcement learning. External generalist teachers, such as a larger model, typically outperform the student not just on the target skill but on others too.

### Method

It starts with a controlled toy study. A randomly initialized one-hundred-million-parameter Qwen2 model is pretrained on math problems needing two to ten operations. A teacher is trained from that base with reinforcement learning on eleven to fourteen operations, the student distills from it starting from the same base, and fifteen to seventeen operations serve as the OOD test.

The main experiments use Qwen3-8B as the student. Math trains on GSM8K, MATH and DAPO, and tests on GSM8K, MATH500, AMC23, AIME24 and AIME25. Coding, tool use and long context each get their own training and test sets.

For math, in versus out of distribution is set by a working rule: a benchmark counts as in-distribution if all three RL teachers score above ninety-seven percent of the base. That puts GSM8K and MATH500 inside, AMC23 and both AIMEs outside. The authors are clear that this is relative to the post-training data only.

The compositional tests are long-context math on GSM-Infinite and DocMath, long-context coding on LongBench-v2 Code, and agentic coding on SWE-bench Verified.

### Results

First the toy study, in this figure. The y-axis is the distillation ratio: how much of the teacher's gain over the base the student recovers. It's ninety-six to a hundred and four percent on the pretraining range, ninety-three to ninety-six on the post-training range, and on OOD it drops to seventy-six to eighty-six.

Now the real tasks. With self-RL specialists and single-teacher OPD, averaged over three training sets, the student reaches ninety-seven point one percent of the teacher's in-distribution score. But its OOD score is ten point seven eight percent below the base, eight point five six points. The teachers forget too, and worse, eleven point four three points below the base on average.

Multiple teachers, same story. MOPD's average OOD score across the mixtures is fourteen point zero seven percent below the base, or eleven point one eight points.

You might think the student just inherits the teacher's forgetting. So the authors switch to a stronger external teacher, Qwen3-32B teaching Qwen3-8B. In-distribution, the student hits ninety-nine point three four percent of the teacher, only zero point six two points short. OOD, it's still three point seven three percent below the base, or two point nine six points.

Next, compositional generalization. Averaged over three settings, MOPD improves in-distribution scores by nine point two four percent over the base, but on the composed tasks it lands twenty-three point eight two percent below the best teacher.

For code plus agent, SWE-bench Verified goes from the base's six point six three to eight point zero five. That beats mixed RL at seven point seven nine, but trails the teacher at nine point eight four.

The clearest case is code plus long context, in this figure. MOPD improves all three component benchmarks, yet on the combined task, LongBench-v2 Code, it sits at the base's forty-four. The two teachers score fifty and fifty-eight.

Math plus long context is mixed. DocMath rises from fifty-five point three eight to sixty point six three, while GSM-Infinite slips from fifty point eight eight to forty-nine point seven five. Both stay below the best teacher.

### Comparison

Against reinforcement learning there's good news for OPD. It forgets twenty-five point zero nine percent less than its self-RL teachers, and MOPD forgets twenty-eight point nine two percent less than mixed RL, which averages fifteen point seven three points below the base.

On composed tasks, mixed RL shows a similar gap, twenty-six point one percent below the best teacher, and MOPD's composed-task average is three point zero eight percent higher than mixed RL's, or one point zero seven points.

Why does OPD forget less? The authors look at the shape of the weight updates, in Table 1. OPD's update norm is fifty-five point four percent of RL's, zero point one five four versus zero point two seven eight. Its top sixty-four singular directions hold sixty-nine percent of the update energy, against thirty-nine point nine for RL. Smaller, more concentrated changes.

With multiple teachers, eleven to eighteen point six percent of MOPD's update is explained by the single-teacher OPD updates. For mixed RL it's under a quarter of one percent. The authors stress this is a preliminary interpretation, and no causal link between this geometry and generalization has been established.

So can it be fixed? Option one: relax the teacher and the data together. Keep Qwen3-32B as teacher, but train on ninety percent GSM8K plus ten percent OpenR1. The student now beats the base on all five benchmarks, matches the teacher's AIME24 and AIME25 pass at eight, eighty-six point six seven and eighty-three point three three, and reaches a hundred on AMC23, above the teacher's ninety-five.

Option two is a patch after training, called Procrustes-attention patching. On selected attention weights, the OPD update is orthogonally aligned to the RL teacher's update, and the mismatched part is subtracted. At a patch strength of zero point four, AIME25 rises from sixty point eight three to sixty-five, within zero point eight three points of the base, while GSM8K holds at ninety-three point four eight.

Option three is base-preserving replay. During training, extra prompts are mixed in with the base model itself as the teacher, here coding prompts. Compared with plain OPD, five of eight OOD scores go up, and AIME25 gains six point six seven points to sixty-seven point five, above both the base and the RL teacher, for a GSM8K cost of just zero point one eight points.

My read is that the main value here isn't a new method. It's a warning about evaluation habits. If you merge specialists with OPD and declare victory because in-distribution scores are near perfect, that's not enough. Test outside the training range, and test combined tasks.

### Limitations

The authors list several limitations. The main student is only Qwen3-8B, and the largest external teacher is Qwen3-32B, so the effects of student size, teacher strength and model family are under-explored.

Public datasets keep things inspectable, but they don't match the breadth of real frontier post-training data. And OOD here is relative to the post-training data, not to what the model saw in pretraining.

Neither MOPD nor mixed RL had their mixture weights, reward balance or curricula extensively tuned. Compute limits also kept them from repeating every experiment with multiple sampling seeds. For math, they re-ran evaluation with four sampling seeds on a ten percent subset, which says nothing about stability across independent training runs.

And MOPD beating mixed RL isn't a universal rule. In a synthetic arithmetic task with a three-layer Transformer, mixed RL with a supervised warmup overtook MOPD on the composed tasks. Over five seeds, it averaged ninety-three and ninety percent on the two compositions, versus seventy-eight and seventy-four for MOPD distilled from the matching warm-RL specialists.

### Future work

The authors call for causal tests of the link between update geometry and generalization: manipulate update directions while controlling update size, training budget and in-distribution performance. They also suggest projecting or regularizing updates step by step during training, and studying teacher assignment, data coverage and replay together under a fixed budget.

My read is that the practical advice for post-training teams is concrete. Don't distill on data that's too narrow. Add some broader data, or replay the base model's own behavior. But note that all three fixes are demonstrated on a single GSM8K math setup, so whether they carry over to code and agents still needs more experiments.

That's today's Daily Frontier Papers. The paper is linked in the description.

