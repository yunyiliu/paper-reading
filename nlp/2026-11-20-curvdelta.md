# CurvDelta: Low-Curvature Delta Modeling for Surgical Learning and Unlearning

**学会新本事不丢老本事**

- 论文 Paper: [arXiv 2611.24654](https://arxiv.org/abs/2611.24654)
- 单位 Affiliation: 上海财经大学 · 阿里巴巴 · 南方科技大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-20
- 视频 Videos: [YouTube 中文](https://youtu.be/NHER0F5CN7k) · [YouTube English](https://youtu.be/wbRdMKczQQs) · [B站](https://www.bilibili.com/video/BV1hSypzVEw3)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇上海财经大学、阿里巴巴和南方科技大学的论文，名字叫 CurvDelta，副标题是，用低曲率的参数增量，做外科手术式的学习和遗忘。

它要解决的是一个很常见的麻烦，给大模型教新本事，或者让它忘掉某些东西，结果顺带把原来会的也弄丢了。

### 问题

先说背景。模型预训练完还得不断调整，要学新任务，也可能因为隐私、安全的要求删掉某些知识，这叫 unlearning，选择性遗忘。

每次都从头重训太贵，所以大家用微调。但不管是学还是忘，都要求改动有选择性，目标任务上要进步，无关的知识和能力要保住。

作者把这件事写成一个约束优化问题。在任务数据上，把任务损失降到最低，同时要求模型在一组保护数据上的输出，和原模型相比，偏离不能超过一个事先给定的容忍度。偏离用 KL 散度来衡量，可以理解成两个输出分布有多不一样。

### 以前的方法

以前大致有两条路。一条是把保住旧能力写进训练目标，比如混入旧数据一起训练，叫经验回放，或者加正则项，惩罚重要参数的变化。

另一条是限制模型能往哪儿改，比如把任务梯度投影到某个子空间，或者凭经验，把更新限制在和原有参数或表示正交的方向上。

作者认为，这些办法要么不能保证更新真的满足约束，要么限制和保住能力只是间接相关，学新和保旧之间的取舍不够理想。

最常见的做法，是把两个损失加权相加。问题是，权重没法换算成能容忍多少偏离，要训完才知道偏了多少，而且两个目标的梯度常常打架，遗忘任务里尤其明显。

### 核心想法

核心想法其实很直观。在原模型附近，保护损失可以用一个二次函数来近似，它在各个方向上弯得有多厉害，也就是曲率，由一个矩阵决定。

把参数更新拆到这个矩阵的各个特征方向上，曲率大的方向，走一点点，保护损失就涨很多。曲率小的方向，走同样远，几乎不影响原来的能力。

打个比方，原模型站在一个山谷底，有些方向是陡坡，有些方向是平地。CurvDelta 的做法，就是只允许模型沿着平地走，在这片平地里，找一个能把新任务学好的位置。

论文第一张图画的就是这个。左边不加约束，保护损失很高。右边限制在低曲率空间里，任务损失相当，保护损失低得多。

### 方法

难点在于规模。大模型有几十亿参数，这个曲率矩阵根本存不下，更别说做特征分解。

作者用了一个经典近似，叫 K-FAC。对每一个线性层，把这一块曲率，近似成两个小矩阵的 Kronecker 积，一个是这一层输入激活的协方差，一个是输出梯度的协方差。

这两个小矩阵，只需要在保护数据上，对原模型估计一次，之后就固定不变。

接着是分因子截断。对两个小矩阵分别做特征分解，从大到小取特征向量，直到累计的能量达到一个阈值。这些是高曲率方向，把它们去掉，剩下的就是低曲率方向。

真正加到权重上的更新，是在一个可训练的增量两边，各乘一个投影矩阵，左边去掉输出方向的高曲率部分，右边去掉输入方向的。原模型权重和两个投影都冻结，只训练中间这个增量。

这个增量有两种写法。写成完整的稠密矩阵，就是 CurvDelta。写成 LoRA 那样两个小矩阵相乘，就是低秩版本 CurvLoRA，要训练的参数少很多。

这里有一个细节很关键。投影写进了参数化本身，所以不管用什么优化器，真正加到模型上的更新，始终在低曲率空间里。如果只是投影梯度，用 Adam 这类自适应优化器时，它会对不同坐标做不同的缩放，更新就会跑出这个空间。

作者还给了一个保证。在局部二次近似和 K-FAC 的假设下，保护损失的上界，是每一层保留空间的最大曲率，乘以这层更新幅度的平方，再加起来。曲率是每一步的代价，幅度是走了多远。

同样用 K-FAC 的 CrispEdit，要存完整的特征向量和一个掩码。CurvDelta 分开截断，只存少量特征向量，省了很多存储和计算，代价是丢掉一部分混合方向，作者认为剩下的空间仍然够用。

### 实验和效果

实验分两部分，先看学新能力。数学方面，用 MetaMathQA 里抽的 10 万条数据微调，在 GSM8K 和 MATH 上测。代码方面，用 CodeFeedback 的 10.4 万条数据微调，在 HumanEval 和 MBPP 上测。

通用能力保没保住，看 TriviaQA、NQ-Open 和 MMLU 这三个知识类基准。保护数据只用了 256 条网页文本。模型是 Gemma-3-1B 和 Llama-2-7B 的基础版。

对比的有全量微调、经验回放、CrispEdit、OPLoRA，以及 LoRA 等几种高效微调方法。

先看代码，表 2。在 Llama-2-7B 上，全量微调的代码平均分是 38.5，通用知识平均分，从原模型的 45.6 掉到 40.4。CurvDelta 的代码平均分是 40.5，比全量微调还高，通用知识保住了 44.4，综合分 42.8，是所有方法里最高的。

在 Gemma-3-1B 上，CurvDelta 的综合分 22.1，也是最高，通用知识 23.7，原模型是 24.2，几乎没掉。

再看数学，表 1。在 Gemma-3-1B 上，全量微调的数学平均分 21.3，通用知识从 24.2 掉到 18.1。CurvDelta 数学 21.2，基本持平，通用知识 22.9，综合分 22.2，最高。

但在 Llama-2-7B 上学数学，CurvDelta 的综合分是 39.0，排在经验回放的 40.2 和 CurvLoRA 的 40.1 后面，也低于 OPLoRA 和 MiLoRA。这一组里，经验回放的数学分和保留分，都比 CurvDelta 高。

接下来是遗忘。TOFU 是一个虚构作者的问答数据集，要求模型忘掉其中一部分作者，最理想的参照，是不用这部分数据、从头重训的模型。

表 3 用的是 Llama-3.1-8B-Instruct 和 SimNPO 这个遗忘目标。遗忘质量是一个取了对数的 p 值，越接近 0，越像重训模型。忘掉 5% 时，CurvDelta 的遗忘质量是负 0.1，通用能力 63.3，重训模型是 63.2。全量微调的遗忘质量是负 0.5，通用能力掉到 54.6。

忘掉 10% 时差距更明显，全量微调的通用能力掉到 38.9，CurvDelta 还有 63.2，重训模型是 64.6。

MUSE 是用真实书籍文本测遗忘。CurvDelta 把逐字背诵的程度，从原模型的 99.7 降到 11.6，是各方法里最低的，重训模型是 14.2。

消融实验里，作者把低曲率空间，换成同样维度的高曲率方向或者随机方向。在最大的学习率下，这两种都损失了通用能力，遗忘效果也更弱。说明起作用的是方向选得对，而不只是空间的大小。

另外，估计曲率的保护数据从遗忘集的 0.25 倍到 4 倍，CurvLoRA 的效果都很稳定。

把投影写进参数化，还是只投影梯度，作者也直接比了。两种学习率下，前者的遗忘质量和通用能力都更好，比如学习率较大时，遗忘质量是负 0.193 对负 0.676。

### 和以前方法的对比

直接看保住了多少。在 TOFU 的同一个设置下，测保留数据上新旧模型输出的 KL 散度，CurvDelta 是 0.149，全量微调是 0.624，CrispEdit 是 0.752。低秩方法里，CurvLoRA 是 0.013，LoRA 是 0.043。

成本方面，在一张 L40 显卡上，算上曲率估计，CurvDelta 总共用了 1004 秒，CrispEdit 是 2482 秒，少了百分之五十九点五。但全量微调只要 537 秒，显存也更少，CurvDelta 并不是免费的。

我的判断是，和 CrispEdit 比，它的进步在于约束写进参数化，用 Adam 也不跑偏，而且分因子截断更省，还能用到 LoRA 上。和 OPLoRA 这类用正交子空间，去保护预训练权重或表示的方法比，它的依据是保护数据上实测的曲率。

### 局限

论文没有单独写局限，下面几点来自它自己的实验和假设。第一，那个理论保证，只在局部二次近似和 K-FAC 的假设下成立。附录里，二次近似给出的数值，在十的负十次方到十的负七次方之间，实测的 KL 在 0.01 到 0.07 之间，两者只是变化趋势一致。

第二，效果不是处处领先。Llama-2-7B 学数学时，简单的经验回放两项都更好。MUSE 上，CurvDelta 保留知识的得分是 55.8，离重训模型的 68.6 还有距离，也低于 VILA、OPLoRA 和 CurvLoRA。

第三，学新能力的实验只用了 Gemma-3-1B 和 Llama-2-7B 两个基础模型，每个任务只训练一轮。更大的模型、更长的训练上效果如何，论文没有测。

### 未来方向

论文强调，这套曲率只取决于原模型、保护数据和阈值，算一次，就能在不同任务、不同训练目标，以及稠密和低秩两种写法之间复用。

我的判断是，这种复用很实用，比如模型要不断处理删除数据的请求，保护空间算一次就能一直用。不过论文只测了单次适配，连续改很多次后偏离会不会累积，还要验证。

我的判断是，这篇的价值，在于把保住能力从凭经验加限制，变成先量出哪些方向危险，再只在安全的方向上改。真要落地，还得看更大的模型上能不能复现。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Shanghai University of Finance and Economics, Alibaba and the Southern University of Science and Technology. It's called CurvDelta, and the subtitle is low-curvature delta modeling for surgical learning and unlearning.

The problem it targets is a familiar one. You teach a large model something new, or make it forget something, and it quietly loses abilities it already had.

### Problem

Some background. Models keep changing after pretraining. They need new tasks and new domains, and privacy or safety rules can require removing specific knowledge. That second case is called unlearning.

Retraining from scratch every time is too expensive, so people fine-tune. Whether you're adding or removing, the change has to be selective: make progress on the target, and leave unrelated knowledge and skills alone.

The authors frame this as a constrained optimization problem. Minimize the loss on the task data, subject to a limit: on a separate preservation set, the model's outputs may drift from the original model's by no more than a chosen tolerance. Drift is measured with KL divergence, which is a way of saying how different two output distributions are.

### Prior work

There have been two broad approaches. One puts preservation into the training objective, for example by mixing old data back in, which is called replay, or by adding a regularizer that penalizes changes to important parameters.

The other restricts where the model is allowed to move, for instance by projecting the task gradient into some subspace, or by heuristically keeping updates orthogonal to parts of the existing parameters or representations.

The authors argue that these methods either don't guarantee that the final update actually satisfies the constraint, or impose restrictions that are only indirectly tied to preserving capability. The result is a poor trade-off between learning and keeping.

The most common recipe simply adds the two losses with a weight. The paper points to two problems with that. First, the weight doesn't translate into a tolerance: you only find out how far the model drifted after training. Second, the two objectives' gradients often conflict, especially in unlearning.

### Key idea

The core idea is quite intuitive. Near the original model, the preservation loss can be approximated by a quadratic, and how sharply it bends in each direction, its curvature, is described by one matrix.

Break an update down along that matrix's eigendirections. In high-curvature directions, a tiny step makes the preservation loss jump. In low-curvature directions, a step of the same size barely touches existing abilities.

Think of the original model sitting at the bottom of a valley. Some directions are steep walls, others are flat ground. CurvDelta only lets the model walk along the flat ground, and looks there for a spot that's good at the new task.

The paper's first figure shows exactly this. On the left, unconstrained training gets low task loss but high preservation loss. On the right, with updates kept in the low-curvature space, task loss is comparable and preservation loss is much lower.

### Method

The hard part is scale. A large model has billions of parameters, so that curvature matrix can't even be stored, let alone decomposed.

The authors use a classic approximation called K-FAC. For each linear layer, the curvature block is approximated as a Kronecker product of two small matrices: the covariance of the layer's input activations, and the covariance of its output gradients.

Those two small matrices are estimated once, on the preservation data, at the original model, and then stay fixed.

Next comes factorwise truncation. Each small matrix is eigendecomposed, and eigenvectors are taken from the largest down until their cumulative energy reaches a threshold. Those are the high-curvature directions. Remove them, and what's left is low curvature.

The update actually applied to a weight is a trainable delta multiplied by a projection on each side. The left projection strips out high-curvature output directions, the right one strips out high-curvature input directions. The base weights and both projections are frozen, and only the delta in the middle is trained.

That delta comes in two forms. As a full dense matrix, it's CurvDelta. As a product of two low-rank matrices, the way LoRA does it, it's the low-rank version, CurvLoRA. LoRA is the familiar efficient fine-tuning method that trains only a small pair of matrices, so far fewer parameters are trained.

One detail matters a lot here. Because the projection is built into the parameterization, the update that reaches the model stays in the low-curvature space no matter which optimizer you use. If you only project gradients, an adaptive optimizer like Adam rescales different coordinates by different amounts, and the update drifts out of that space.

The authors also give a guarantee. Under the local quadratic and K-FAC assumptions, the preservation loss is bounded by summing, over layers, the largest curvature left in that layer's space times the squared size of that layer's update. Curvature sets the cost per step, and the update size sets how far you went.

Compared with CrispEdit, which also uses K-FAC, CrispEdit thresholds the full product spectrum element by element, so it has to keep complete eigenvector bases and a mask. CurvDelta truncates each factor separately and keeps only a few leading eigenvectors, which saves a lot of storage and compute. The cost is that some mixed directions get discarded; citing earlier work, the authors argue the remaining space is still large enough.

### Results

The experiments come in two parts. First, learning new skills. For math, they fine-tune on a hundred thousand examples sampled from MetaMathQA and test on GSM8K and MATH. For code, they use a hundred and four thousand examples from CodeFeedback and test on HumanEval and MBPP.

Retention is measured on three knowledge benchmarks: TriviaQA, NQ-Open and MMLU. The preservation set is just two hundred fifty-six web-text examples. The models are the base versions of Gemma 3 1B and Llama 2 7B.

The comparisons include full fine-tuning, replay, CrispEdit, OPLoRA, and the efficient fine-tuning methods LoRA, PiSSA, MiLoRA and CorDA.

Code first, in Table 2. On Llama 2 7B, full fine-tuning reaches a code average of thirty-eight point five, while the knowledge average falls from the original model's forty-five point six to forty point four. CurvDelta gets a code average of forty point five, higher than full fine-tuning, keeps knowledge at forty-four point four, and has the best overall score of all methods, forty-two point eight.

On Gemma 3 1B, CurvDelta also has the best overall score, twenty-two point one, with knowledge at twenty-three point seven against the original model's twenty-four point two. Almost nothing lost.

Now math, in Table 1. On Gemma 3 1B, full fine-tuning gets a math average of twenty-one point three, and knowledge drops from twenty-four point two to eighteen point one. CurvDelta scores twenty-one point two on math, essentially the same, keeps knowledge at twenty-two point nine, and has the best overall score, twenty-two point two.

But on Llama 2 7B math, CurvDelta's overall score is thirty-nine point zero. That's behind replay at forty point two and CurvLoRA at forty point one, and also below OPLoRA and MiLoRA. In this setting, replay beats CurvDelta on both math and retention.

Next, unlearning. TOFU is a question-answering dataset about fictitious authors. The model has to forget some of the authors, and the ideal reference is a model retrained from scratch without them.

Table 3 uses Llama 3.1 8B Instruct with the SimNPO unlearning objective. Forget quality is the log of a p-value, and the closer to zero, the more the model looks like the retrained one. Forgetting five percent of the data, CurvDelta's forget quality is minus zero point one, with model utility at sixty-three point three, against sixty-three point two for the retrained model. Full fine-tuning gets minus zero point five, and its utility drops to fifty-four point six.

At ten percent the gap widens. Full fine-tuning's utility falls to thirty-eight point nine, CurvDelta keeps sixty-three point two, and the retrained model has sixty-four point six.

MUSE tests unlearning on real book text. CurvDelta cuts verbatim memorization from the original model's ninety-nine point seven to eleven point six, the lowest of any method. The retrained model sits at fourteen point two.

In the ablations, the authors swap the low-curvature space for high-curvature or random directions of the same dimension. At the largest learning rate, both alternatives lose utility and still forget less. So what matters is choosing the right directions, not just the size of the space.

Also, the amount of preservation data used to estimate curvature, from a quarter of the forget set's size up to four times it, makes little difference to CurvLoRA. A big preservation set isn't needed.

They also compare building the projection into the parameterization against projecting only the gradients. At both learning rates tested, the first gives better forget quality and better utility; at the higher learning rate, forget quality is minus zero point one nine three versus minus zero point six seven six.

### Comparison

Now look directly at how much is preserved. In the same TOFU setting, they measure the KL divergence between old and new model outputs on retained data. CurvDelta comes in at zero point one four nine, full fine-tuning at zero point six two four, and CrispEdit at zero point seven five two. Among low-rank methods, CurvLoRA gets zero point zero one three, against zero point zero four three for LoRA.

On cost, on a single L40 GPU and including curvature estimation, CurvDelta takes one thousand and four seconds in total, against two thousand four hundred eighty-two for CrispEdit, fifty-nine point five percent less. But full fine-tuning needs only five hundred thirty-seven seconds and less memory. CurvDelta isn't free.

My read is that the advance over CrispEdit comes from two things: putting the constraint into the parameterization, so Adam can't push the update off course, and factorwise truncation, which saves storage and compute and carries over directly to LoRA. Against methods like OPLoRA and MiLoRA, which pick their subspaces from the pretrained weights or representations, CurvDelta picks them from curvature actually measured on preservation data.

### Limitations

The paper has no separate limitations section, so these points come from its own experiments and assumptions. First, the guarantee only holds under the local quadratic and K-FAC assumptions. In the appendix, the quadratic approximation produces values between ten to the minus ten and ten to the minus seven, while the measured KL sits between zero point zero one and zero point zero seven, so the two agree only in their trends.

Second, it doesn't win everywhere. On Llama 2 7B math, plain replay is better on both counts. On MUSE, CurvDelta's retained-knowledge score is fifty-five point eight, well short of the retrained model's sixty-eight point six, and also below VILA, OPLoRA and CurvLoRA.

Third, the skill-learning experiments use only two base models, Gemma 3 1B and Llama 2 7B, trained for one epoch per task. How it behaves on larger models or with longer training isn't tested.

### Future work

The paper stresses that the curvature depends only on the original model, the preservation data and the threshold. Compute it once, and it can be reused across tasks, training objectives, and both the dense and low-rank forms.

My read is that this reuse is valuable in practice. If a model keeps receiving data-deletion requests, or needs several skills added one after another, the protected space only has to be computed once. But the paper tests a single adaptation, and whether drift builds up over many successive edits still needs checking.

My read is that the paper's real contribution is changing how preservation is done: instead of heuristic restrictions, first measure which directions are dangerous, then only move in the safe ones. Whether it holds up depends on reproducing these results on larger models and more varied preservation data.

That's today's Daily Frontier Papers. The paper is linked in the description.

