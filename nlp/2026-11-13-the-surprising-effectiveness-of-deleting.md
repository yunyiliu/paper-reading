# The Surprising Effectiveness of Deleting Weights in LLM Post-Training

**删不到万分之五权重大模型上追平全量微调**

- 论文 Paper: [arXiv 2611.10866](https://arxiv.org/abs/2611.10866)
- 单位 Affiliation: 纽约大学 · 卡尔加里大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-13
- 视频 Videos: —

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇纽约大学和卡尔加里大学的论文，标题翻过来是，在大模型后训练里，删权重出人意料地有效。

它问了一个很基础的问题，一个预训练好的大模型，想学会新本事，到底要改动多少。论文给的答案是，不写入任何新数值，只把不到万分之五的权重设成零，在测试的最大规模上，就能追平全量微调。

### 问题

先说背景。大模型预训练完，还要做后训练，比如用可验证奖励的强化学习，监督微调，或者在线蒸馏。最直接的做法是全量微调，每一个参数都可以改。

代价有两个。一是每个任务都要存一份完整的模型，换任务就要换一整个检查点。二是遗忘，学了新任务，原来会的东西会变差。

所以作者问，一个预训练大模型，能不能只靠学习，哪些权重该设成零，就学会新能力。

### 以前的方法

现在省参数的微调，主要有两类。LoRA 学一个低秩的增量，也就是用两个很窄的小矩阵，拼出权重的改动。另一类是稀疏更新，比如 MTH，随机挑一小部分位置，只训练这些位置上的数值。它们的共同点是，都要往模型里写新的浮点数。

删权重这件事，以前主要用在剪枝上。剪枝是在不伤能力的前提下，删掉一部分权重，让推理更省。它是为了保住能力，而不是学新能力。

不过最近有研究发现，微调之后的模型，其实离预训练模型很近，强化学习微调只改动了一个很小的子网络。这篇论文就是把这个观察推到极限。

### 核心想法

核心想法就一句话，删除不写入任何新数值，它只决定，哪些预训练权重留下，哪些清零。

这张图就是整个方法。预训练权重全部冻结，每个权重配一个可训练的分数，叫 logit。部署的时候，分数为正的权重保留原值，为负的直接设成零。所以训练出来的适配器，就是每个权重一个比特，留，或者删。

### 方法

先说删哪里。作者只动每一层 MLP 里的 gate 投影。Llama 和 Qwen 这类模型用的是 SwiGLU 结构，gate 这一路的输出，会以相乘的方式，控制中间每个通道放多少信息过去，有点像一排阀门。

训练时，零和一这种硬开关没法求梯度，所以先用一个 sigmoid 把它变软，每个权重乘上一个零到一之间的系数。再让 sigmoid 的陡峭程度线性升高，强化学习实验里是从 1000 升到 10000，到训练结束，软系数基本就变成了硬开关。作者报告，最后换成硬开关，准确率变化不超过 0.08 个点。

梯度通过软系数传回去，而生成训练样本、做评测的时候，用的是硬开关之后的模型。

初始化借用了剪枝方法 Wanda 的打分，也就是权重的绝对值，乘上它输入激活的大小。打分最低的一小部分先倾向于删除，一开始，大约只有千分之一点八的 gate 权重是零。

训练目标是各自原来的损失，强化学习用 GRPO，监督微调用交叉熵，蒸馏用 SDFT 的损失，前两种再加一个很轻的正则项。作者没有规定要删多少，删几个，由训练自己决定。

存储方面，把这些比特打包，再用 zstd 压缩，Llama 1B 只有 1.0 MB，Llama 8B 是 8.4 MB。换任务的时候，只要加载一个小小的掩码，把对应位置清零就行。

### 实验和效果

实验覆盖三种后训练，主力是强化学习 GRPO。一组是 Llama 的 1B、3B、8B，在 GSM8K 数学应用题上训练和测试。另一组是 Qwen2.5 的 0.5B、3B、7B，用 SimpleRL-Zoo 的数学数据训练，看六个没参与训练的数学基准的平均分。对照组是全量微调，秩为 1 和 4 的 LoRA，以及两种稀疏度的 MTH。

主要结果在表 1。最大的两个模型上，Llama 8B 做 GSM8K，删权重是 83.55，全量微调是 83.24。Qwen 7B 的六项平均，删权重 52.32，全量微调 51.89。而这些实验里，删掉的都不到全部权重的万分之五。

再往大了做，表 2 里，Qwen2.5 14B 是 55.43 对 54.87，Qwen3 的 30B 混合专家模型，是 58.54 对 58.43。这个 30B 模型的掩码只有 29.6 MB，全量微调的检查点是 116471 MB。

但小模型上有差距。Llama 1B 上，删权重是 54.44，全量微调是 60.50。论文的说法是，小模型上能恢复全量微调百分之九十到九十九的准确率，差距随着模型变大而缩小。

那提升是不是来自 Wanda 初始化本身。作者只用 Wanda 剪枝，不做任何训练，试了 8 个稀疏度，取最好的一个，结果离原模型最多 2.4 个点。而从同样起点训练出来的掩码，最多提升了 53.2 个点，Llama 1B 从 1.21 提到了 54.44。所以真正学到东西的，是掩码本身。

它是不是真学到了新能力，而不只是让模型更常答对原本就会的题。表 3 看的是 pass@k，也就是 k 次尝试里至少做对一次的比例。就算让原模型试 128 次，在 Llama 8B 和 Qwen 7B 上，它仍然比删权重之后的模型低 4.3 和 6.1 个点。

最亮眼的是遗忘。看这张图，上面是训练中和原模型之间的 forward KL，可以理解为模型行为偏离原来的程度，下面是六个旧任务平均分的变化。到第 348 步，全量微调的 KL 在 Llama 1B 上到了 1.0，在 8B 上到了 1.5，旧任务掉了 1.3 到 3.1 个点。删权重的 KL 一直不超过 0.05，旧任务始终在原模型上下一个点以内。LoRA 和 MTH 只训练很少的参数，旧任务却最多掉了 5.4 个点。

还有一个对照很说明问题。稀疏度最高的 MTH，只训练万分之五的参数，删权重删掉的，是万分之三点五到四点六，规模相当，但 MTH 照样遗忘。作者据此认为，让遗忘变少的，是删除这个动作本身，而不是改动的数量少。

换成监督微调和蒸馏，结论也大体成立。Llama 3B 在 MMLU-Pro 上，删权重 22.28，全量微调 21.19，在 BBEH 上两边都是 11.39。Qwen 7B 用 SDFT 蒸馏，工具调用是 69.07 对 68.04，化学题是 70.61 对 71.20。

### 和以前方法的对比

和 LoRA、MTH 比，在大模型上，准确率其实都差不多，比如 Llama 8B 上，秩为 1 的 LoRA 是 83.38，删权重是 83.55。真正拉开的是体积。删权重在每个规模上都是最小的适配器，压缩后每个权重大约只占 0.03 比特，而百分之九十九稀疏度的 MTH，在每组实验里都大 50 倍以上。

作者还解释了，为什么光删就够。他们比较了权重改动的几何形状。先看稳定秩，大致是改动用了多少个独立方向，删权重是 66 到 93，LoRA 只有 1 到 2，MTH 是 410 到 650。再看奇异值的漂移，删权重在全量微调的 1.4 倍以内，秩为 1 的 LoRA 最多是 84 倍。改动落在主要权重上的比例，删权重是 0.43 到 0.46，和全量微调的 0.41 到 0.46 吻合。作者的解释是，LoRA 有秩的上限，MTH 的位置是随机定的，而删权重自己选位置，也没有秩的限制，所以更像全量微调。

我的判断是，这篇的卖点不是准确率更高，大模型上几种方法几乎打平。真正有用的是两件事加在一起，几 MB 到几十 MB 的掩码，加上几乎不遗忘。如果要在一个底座模型上挂很多个任务版本，这很实用。

我的判断是，弱点也要看清。小模型上它明显落后，比如 Llama 1B 在 BBEH 上，删权重 5.65，全量微调 8.35。而且表里报告的，都是验证集上最好那个检查点的分数，论文也没有给出多次运行的波动范围，零点几个点的领先，不宜看成真的更强。

### 局限

作者自己提到的局限是训练显存。每个被掩码的权重都要配一个 logit，所以比全量微调省，但比 LoRA 费。他们设想，只给一小部分候选权重配 logit，可以缩小这个差距。

另外，遗忘实验只在 GRPO 和 Llama 三个尺寸上做了，而多个任务接连学习的持续学习场景，论文还没有测，作者把它列为下一步。

### 未来方向

作者提了几个方向。比较不同任务删掉了哪些权重，也许能定位后训练到底改了模型的哪里。几个任务的掩码，也许可以直接取并集，不用再训练就合并技能。如果把掩码限制成 N 比 M、按通道或者按块的结构，同一个掩码，也许既能适配任务，又能压缩模型。

我的判断是，这篇最有意思的地方，是它支持了一种看法，后训练也许更多是在挑选，新能力的原料，大部分已经在预训练权重里了。不过这些结论来自最大 30B 的模型，掩码合并也还只是设想，值得看后续有没有人在更大的模型上复现。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from New York University and the University of Calgary, and its title is a claim: deleting weights is surprisingly effective in LLM post-training.

The question underneath is basic. How much of a pretrained model actually has to change for it to learn something new? The paper's answer: write no new values at all, just set fewer than zero point zero five percent of the weights to zero, and at the largest scales tested you match full fine-tuning.

### Problem

Some context. After pretraining, models go through post-training, such as reinforcement learning with verifiable rewards, supervised fine-tuning, or on-policy distillation. The blunt way to do it is full fine-tuning, where every parameter is free to move.

That has two costs. You store a whole model per task, and switching tasks means swapping an entire checkpoint. And the model forgets: as it learns the new task, it gets worse at things it used to do.

So the authors ask: can a pretrained model pick up new capabilities by learning nothing but which weights to set to zero?

### Prior work

Today's parameter-efficient options fall into two camps. LoRA learns a low-rank update, two thin matrices whose product is the change to the weights. Sparse updates, like MTH, pick a small random set of positions and train only the values there. Either way, you're writing new floating-point numbers into the model.

Deleting weights has mostly meant pruning. Pruning removes weights while trying not to hurt the model, to make inference cheaper. The goal is to keep capability, not to gain it.

But recent work found that fine-tuned models sit very close to their pretrained starting point, and that reinforcement learning changes only a small subnetwork. This paper takes that observation to its limit.

### Key idea

The key idea fits in one sentence: deletion writes no new values; it only decides which pretrained weights stay and which become zero.

This figure is the whole method. The pretrained weights are frozen, and each one is paired with a trainable score, a logit. At deployment, weights with a positive logit keep their value, and weights with a negative logit are zeroed. So the trained adapter is one bit per weight: keep, or delete.

### Method

First, where to delete. The authors only touch the gate projection in each block's MLP. Llama and Qwen use SwiGLU, where the gate's output multiplies the intermediate channels, so it works like a row of valves controlling what flows through.

During training, a hard zero-or-one switch has no useful gradient, so they soften it with a sigmoid: each weight is multiplied by a factor between zero and one. Then the sigmoid's steepness ramps up linearly, from one thousand to ten thousand in the reinforcement learning runs, so by the end the soft factor has become a hard switch. The authors report that swapping in the hard switch at the end changes accuracy by at most zero point zero eight points.

Gradients flow through the soft factors, while sampling rollouts and evaluation both use the hard-switched model.

Initialization borrows the score from Wanda, a pruning method: a weight's magnitude times the size of its input activation. The lowest-scoring sliver starts out leaning toward deletion, so at the start only about zero point one eight percent of gate weights are zero.

The training loss is each regime's usual one: GRPO for reinforcement learning, cross-entropy for supervised fine-tuning, and the SDFT loss for distillation, with a mild regularizer added for the first two. There's no target sparsity. Training decides how many weights to delete.

For storage, the bits are packed and compressed with zstd: one megabyte for Llama 1B, eight point four megabytes for Llama 8B. Switching tasks means loading a tiny mask and zeroing those positions.

### Results

The experiments cover three post-training regimes, with GRPO reinforcement learning as the main one. One track trains and tests Llama 1B, 3B and 8B on GSM8K math word problems. The other trains Qwen2.5 at 0.5B, 3B and 7B on the SimpleRL-Zoo math data and reports the average over six held-out math benchmarks. The baselines are full fine-tuning, LoRA at rank one and rank four, and MTH at two sparsity levels.

The main results are in Table 1. On the two largest models, Llama 8B on GSM8K scores eighty-three point five five with deletion versus eighty-three point two four with full fine-tuning. Qwen 7B's six-benchmark average is fifty-two point three two versus fifty-one point eight nine. And every one of these runs deletes fewer than zero point zero five percent of all weights.

Going bigger, in Table 2, Qwen2.5 14B scores fifty-five point four three against fifty-four point eight seven, and the Qwen3 30B mixture-of-experts model scores fifty-eight point five four against fifty-eight point four three. That 30B model's mask is twenty-nine point six megabytes, against a full fine-tuning checkpoint of about one hundred sixteen thousand megabytes.

Smaller models show a gap, though. On Llama 1B, deletion reaches fifty-four point four four, while full fine-tuning reaches sixty point five. The paper puts it as recovering ninety to ninety-nine percent of full fine-tuning's accuracy at small scale, with the gap closing as models grow.

Is the gain just coming from the Wanda initialization? The authors pruned with Wanda alone, no training, at eight sparsity levels, and took the best. It stayed within two point four points of the base model. The trained mask, starting from the same scores, gained up to fifty-three point two points: Llama 1B went from one point two one to fifty-four point four four. So the learning really is in the mask.

Does it learn new capabilities, or just make the model answer what it already could more often? Table 3 looks at pass at k, the share of problems solved at least once in k tries. Even with one hundred twenty-eight attempts, the base model still trails the deletion-tuned model by four point three points on Llama 8B and six point one on Qwen 7B.

The standout result is forgetting. In this figure, the top row is forward KL against the base model during training, roughly how far the model's behavior has drifted, and the bottom row is the change in average score on six earlier tasks. By step three hundred forty-eight, full fine-tuning's KL reaches one point zero on Llama 1B and one point five on 8B, and the old tasks drop by one point three to three point one points. Deletion's KL stays at or below zero point zero five, and old-task scores stay within one point of the base model throughout. LoRA and MTH train only a few parameters, yet their old-task scores fall by up to five point four points.

One comparison makes the point sharper. The sparsest MTH trains zero point zero five percent of parameters, while deletion removes zero point zero three five to zero point zero four six percent, about the same scale, and MTH still forgets. The authors take this to mean that deletion itself keeps forgetting low, not the small number of edits.

The result mostly holds for supervised fine-tuning and distillation too. Llama 3B on MMLU-Pro scores twenty-two point two eight with deletion versus twenty-one point one nine with full fine-tuning, and on BBEH both land at eleven point three nine. For Qwen 7B with SDFT distillation, tool use is sixty-nine point zero seven versus sixty-eight point zero four, and chemistry questions are seventy point six one versus seventy-one point two.

### Comparison

Against LoRA and MTH, accuracy on the big models is basically a tie. On Llama 8B, rank-one LoRA scores eighty-three point three eight, deletion eighty-three point five five. What separates them is size. Deletion has the smallest adapter at every scale, about zero point zero three bits per weight after compression, while MTH at ninety-nine percent sparsity is more than fifty times larger in every run.

The authors also explain why deletion alone is enough, by comparing the geometry of the weight changes. Stable rank, roughly how many independent directions the change uses, is sixty-six to ninety-three for deletion, one to two for LoRA, and four hundred ten to six hundred fifty for MTH. On singular-value drift, deletion stays within one point four times full fine-tuning, while rank-one LoRA drifts up to eighty-four times more. And the share of changes landing on the principal weights is zero point four three to zero point four six for deletion, matching full fine-tuning's zero point four one to zero point four six. Their explanation: LoRA has a rank cap, MTH's positions are fixed at random, while deletion picks its own positions with no rank cap, so it ends up looking like full fine-tuning.

My read is that the selling point isn't higher accuracy; on large models the methods are close to a tie. What's useful is the combination: a mask of a few to a few tens of megabytes, plus almost no forgetting. If you want many task variants on one base model, that's very practical.

My read is also that the weaknesses deserve attention. On small models it clearly lags: Llama 1B on BBEH scores five point six five with deletion against eight point three five with full fine-tuning. And the tables report the best validation checkpoint, with no run-to-run spread given in the paper, so a lead of a few tenths of a point shouldn't be read as a real advantage.

### Limitations

The limitation the authors name is training memory. Every masked weight needs its own logit, so it uses less memory than full fine-tuning but more than LoRA. They suggest giving logits only to a small set of candidate weights to close that gap.

Also, the forgetting study covers GRPO on three Llama sizes only, and continual learning, where tasks arrive one after another, isn't tested yet. The authors list it as the natural next step.

### Future work

The authors sketch a few directions. Comparing which weights different tasks delete might show where post-training actually changes a model. Masks for different tasks might be combined, say by taking their union, to merge skills with no further training. And restricting masks to N to M, channel or block patterns might let one mask both adapt the model and compress it.

My read is that the most interesting thing here is support for a particular view: post-training may be mostly selection, with the raw material for new skills already in the pretrained weights. But these results top out at a 30B model, and mask merging is still an idea, so watch for whether others reproduce this on larger models.

That's today's Daily Frontier Papers. The paper is linked in the description.

