# Intrinsic Distribution Matching

**两步出图也要把字写对**

- 论文 Paper: [arXiv 2610.39633](https://arxiv.org/abs/2610.39633)
- 单位 Affiliation: Inclusion AI · 西湖大学 · 清华大学 等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-30
- 视频 Videos: [YouTube 中文](https://youtu.be/i-acSbKayrk) · [YouTube English](https://youtu.be/NhkUrwCy2P8) · [B站](https://www.bilibili.com/video/BV1snxq6yEp3)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 Inclusion AI 和西湖大学等单位的论文，名字叫 Intrinsic Distribution Matching，简称 IDM，可以翻译成内在分布匹配。

它关心的是，文生图模型只走一两步就出图的时候，怎么还能把提示词里的细节画对，尤其是图里大段的中文和英文文字。

### 问题

先说背景。现在主流的扩散模型和 flow matching 模型，生成一张图要反复调用网络很多次，论文用 NFE 表示调用次数。比如论文的表里，Z-Image 基础版大约要 100 次。

一步或几步生成，就是把这个过程压缩成一两次调用，速度快很多。难点是，只有一两次机会，提示词里写的细节很容易丢。

作者选了长文本渲染来检验，也就是在图里写一大段字。很多字都要清楚、正确，还要和画面协调，这对快速生成是很严格的考验。

### 以前的方法

一个主流做法叫分布匹配蒸馏，简称 DMD。这里先解释一个词，score，可以理解成在某个位置上，概率密度往哪个方向升高。

DMD 用一个预训练的老师模型，提供参考分布的 score，再单独训练一个 fake-score 网络，估计学生当前生成分布的 score。两者的差，就告诉学生，生成的样本应该往哪边挪。

这个额外的网络要一直跟着学生变化，训练很麻烦。后来的 TwinFlow 和 APEX 做了简化，让生成器自己在内部分出两个速度分支，一个代表参考分布，一个代表模型自己的分布，就不需要单独的 fake-score 网络了。

IDM 沿用了 APEX 这种内部构造。作者接着问，生成器能不能再给自己提供一个跨样本的统计约束。

### 核心想法

作者先回到 DMD 背后的数学。DMD 的更新方向，来自反向 KL 散度的梯度，也就是两个 score 的差。

他们把这个方向拆成两部分。一部分只由均值和协方差决定，叫高斯矩分量。另一部分是均值和协方差之外的所有结构，叫非高斯残差。

这张图画的就是这个拆分，左边是完整的场，中间是矩分量，右边是残差。图里也说明，两个分布的均值和协方差一样，残差仍然可以不为零。

由此得出两个结论。均值、方差这些量，可以单独拿出来显式约束。但只对齐它们不够，完整的 DMD 修正还得保留。

作者的理由是，理论上精确的 DMD 已经包含全部信息，但实际用的 score 是学出来的估计，训练也是有限的，统计量不一定对得准，额外加一个约束可能有帮助。

### 方法

所以 IDM 的总损失是三项相加，一个基础的生成损失，一个 DMD 损失，再加一个新的特征矩损失。默认权重是 DMD 取 1，特征矩取 0.5。

这张图是两种损失的流程。上面是 DMD 损失，把预测出的干净图片重新加噪，用两个内部分支的速度差，构造一个固定的修正目标，同时监督速度预测和干净图片的预测。

下面是新加的特征矩损失。特征从哪来呢，作者用的是生成器自己的 EMA 副本，也就是参数的滑动平均版本。把它最后一层的隐藏状态做平均池化，每张图得到一个特征向量。

生成图片和参考图片，都先在高噪声水平下加噪，再送进这个 EMA 副本提取特征。然后要求两边特征的均值一样，每一维的标准差也一样。

论文说明，这个损失就是两个对角高斯分布之间 2-Wasserstein 距离的平方。均值项把整体往参考的中心拉，标准差项调整散开的程度。

为了让统计量更稳，作者用了两个队列，各存最多 16384 个历史特征，一个存生成样本，一个存参考样本，生成样本这边还会加上当前批次，一起估计均值和方差。

这样设计的好处是，不需要额外的判别器，不需要单独的 score 网络，也不需要 DINOv2 这样的外部图像编码器，两种监督信号都来自生成器和它的 EMA 副本。特征提取和队列只在训练时用，推理时一次前向就能出图。

### 实验和效果

实验设置。作者从两个模型出发，60 亿参数的 Z-Image-Turbo，和 200 亿参数的 Qwen-Image，分别做全参数微调和 LoRA 微调，LoRA 的秩是 32 和 64。

训练数据除了公开数据集，还有用 Qwen-Image-2512 和 Z-Image-Turbo 合成的样本，其中 60 万张来自 FLUX-Reason-6M 的提示词，500 万张来自文字渲染的提示词。

训练上，Z-Image-Turbo 跑 15000 步，Qwen-Image 的 LoRA 跑 10000 步，全局批大小是 64。

最主要的结果是长文本渲染，在表 5。两次网络调用下，IDM 在中文 LongText 上是 94.25，英文是 90.20，论文报告这是这个预算下对比方法里最高的。

它比 Z-Image-Turbo++ 高 2.63 和 0.32 个百分点，比 APEX 高 3.87 和 1.44 个百分点。作为参照，同样只走两步，原版 Z-Image-Turbo 中文只有 78.48，英文是 72.89。

综合质量在表 1。Qwen-Image 全参数版本走两步，GenEval 是 0.91，APEX 是 0.90。FID 是 6.34，APEX 是 6.44，FID 越低越好。在单张 A100 上，单张图的延迟分别是 0.44 秒和 0.45 秒，几乎一样。

同一个模型也能只走一步，GenEval 是 0.90，延迟 0.35 秒。用秩 64 的 LoRA，只训练 4 亿参数，两步的 GenEval 是 0.90，和全参数的 0.91 很接近。

### 和以前方法的对比

那新加的特征矩损失到底贡献多少，要看控制变量的消融，都在 Z-Image 全参数、两步的设置下做。

在同一个 DMD 基线上加特征矩损失，GenEval 从 77.46 到 77.81，中文 LongText 从 93.87 到 94.25，英文从 89.79 到 90.20，论文报告的五个综合指标都有提升。

反过来，去掉 DMD 损失，GenEval 掉到 76.90，两个都去掉是 76.24。作者自己的结论是，矩匹配有帮助，但 DMD 修正仍然是核心。

特征用什么也做了比较。用外部的 DINOv2 编码器，GenEval 是 77.75，中文 LongText 是 94.13。用 EMA 特征是 77.81 和 94.25，两者很接近。直接用正在训练的模型自己的特征是 77.51，用老师的特征是 77.36。

这张图是定性对比，一张寿司海报，一张咖啡店菜单。论文说，完整的 IDM 在密集文字上保持得更稳定，尤其是多行的菜单。

和 APEX 的比较要注意一点。论文自己说明，APEX 用的训练数据更少，所以这个对比反映的是整套训练设置的差别。而且在 DPG-Bench 上，APEX 是 88.64，反而高于 IDM 的 86.82。

我的判断是，这篇最站得住的结论是，生成器自己的 EMA 特征可以替代外部编码器做矩匹配，效果接近 DINOv2，还省掉一个模型。至于长文本上的领先，有多少来自新损失，有多少来自 500 万张文字渲染数据，外部对比里分不开，能直接归给新损失的，是消融里不到半个百分点的提升。

### 局限

论文没有单独的局限章节，下面几点来自论文自己的说明和表格。

第一，特征矩损失只对齐所有提示词混在一起的均值和每一维的标准差，不区分具体条件，也不匹配完整的特征分布。作者也说了，矩相同，分布不一定相同。

第二，队列里的特征来自之前的模型状态，算出来的统计量只是当前分布的近似。

第三，两步出图在部分指标上仍有差距。表 5 里，IDM 两步的英文 LongText 是 90.20，还低于 Z-Image-Turbo 走 8 步的 91.74。表 2 里，IDM 在 Z-Image 上的 GenEval 是 0.78，Z-Image 基础版大约走 100 步，是 0.84。

第四，评测都是自动基准，论文没有报告人工评测。

### 未来方向

论文也没有专门写未来方向。作者在结论里说，在最大 200 亿参数的模型上的实验，支持把生成器的预测和表示一起用于少步文生图。作者称代码已经公开。

我的判断是，这里值得关注的是自给自足的思路，生成器既当学生，又给自己提供评判用的特征。如果这在视频生成这种更贵的场景也成立，少步蒸馏的成本会更低，不过论文只做了图像。

我的判断是，下一步该看的是，把同样的数据换给 APEX 这类方法，长文本上的差距还剩多少，这决定了这个新损失本身的分量。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Inclusion AI, Westlake University and several partners. It's called Intrinsic Distribution Matching, or IDM.

The question it asks: when a text-to-image model gets only one or two steps to draw a picture, how do you keep the details from the prompt intact, especially long passages of Chinese and English text inside the image?

### Problem

Some background first. Diffusion and flow-matching models build an image by calling the network over and over, and the paper counts those calls as NFE, the number of network evaluations. In the paper's tables, the Z-Image base model takes about a hundred of them.

One-step and few-step generation squeezes that whole process into one or two calls, which is far faster. The catch is that with only one or two chances, the details in the prompt are easy to lose.

The authors use long-text rendering as the stress test: writing a large block of text inside the image. Many characters have to come out legible and correct, and still fit the scene. That's a hard bar for fast generation.

### Prior work

One popular approach is distribution matching distillation, or DMD. It's built on something called a score, which you can think of as the direction, at any point, in which probability density goes up.

In DMD, a pretrained teacher supplies the score of the reference distribution, and a separate fake-score network is trained to estimate the score of whatever the student is currently generating. The difference between the two tells the student which way to move its samples.

That extra network has to keep chasing the student as it changes, which makes training awkward. Two later methods, TwinFlow and APEX, simplified this. The generator itself carries two internal velocity branches, one standing in for the reference distribution and one for its own, so no separate fake-score network is needed.

IDM keeps APEX's internal setup. The authors' next question is whether the generator can also supply an explicit statistical constraint across samples.

### Key idea

They start from the math behind DMD. DMD's update direction is the gradient of the reverse KL divergence, which works out to the difference between two scores.

They split that direction into two parts. One part depends only on the means and covariances, the Gaussian moment component. The other part is everything beyond means and covariances, the non-Gaussian residual.

This figure shows the split: the full field on the left, the moment part in the middle, the residual on the right. It also makes the point that two distributions can share means and covariances and still leave a nonzero residual.

Two conclusions follow. Means and variances are quantities you can pull out and constrain directly. But matching them alone isn't enough, so the full DMD correction has to stay.

The authors' reasoning is that exact DMD already contains all of this, but in practice the scores are learned estimates and training is finite, so the statistics may not line up. An extra constraint might help close that gap.

### Method

So IDM's loss has three terms: a base generation loss, the DMD loss, and the new feature-moment loss. By default, DMD gets a weight of one and the feature-moment term gets zero point five.

This figure shows both losses. On top is the DMD loss. The predicted clean image is noised again, the difference between the two internal velocity branches gives a fixed correction target, and that target supervises both the velocity prediction and the clean-image prediction.

Below is the new feature-moment loss. Where do the features come from? From the generator's own EMA copy, a moving average of its weights. The last layer's hidden states are average-pooled into one feature vector per image.

Generated images and reference images are both noised at a high noise level and passed through this EMA copy. The loss then asks the two sets of features to have the same mean, and the same standard deviation in every dimension.

The paper notes that this loss is exactly the squared two-Wasserstein distance between two diagonal Gaussians. The mean term pulls the average toward the reference, and the standard-deviation term adjusts the spread.

To keep those statistics stable, the authors keep two queues of up to sixteen thousand three hundred and eighty-four past features each, one for generated samples and one for reference samples, and on the generated side they add in the current batch.

The payoff of this design: no discriminator, no separate score network, and no external image encoder like DINOv2, since both training signals come from the generator and its EMA copy. Feature extraction and the queues are only used in training, and at inference, one forward pass produces the image.

### Results

The setup. The authors start from two models, Z-Image-Turbo with six billion parameters and Qwen-Image with twenty billion, and train each with full fine-tuning and with LoRA at ranks thirty-two and sixty-four.

Besides public datasets, the training data includes images synthesized with Qwen-Image-2512 and Z-Image-Turbo: six hundred thousand from FLUX-Reason-6M prompts, and five million from text-rendering prompts.

Training runs fifteen thousand steps on Z-Image-Turbo and ten thousand steps for the Qwen-Image LoRA, with a global batch size of sixty-four.

The headline result is long-text rendering, in Table 5. At two network calls, IDM scores ninety-four point two five on Chinese LongText and ninety point two on English, which the paper reports as the best among the compared methods at that budget.

That's two point six three and zero point three two points above Z-Image-Turbo++, and three point eight seven and one point four four above APEX. For reference, plain Z-Image-Turbo at the same two steps scores just seventy-eight point four eight in Chinese and seventy-two point eight nine in English.

Overall quality is in Table 1. The fully tuned Qwen-Image version at two steps scores zero point nine one on GenEval, against zero point nine zero for APEX. Its FID is six point three four versus six point four four for APEX, and lower is better. On a single A100, latency per image is zero point four four seconds versus zero point four five, essentially the same.

The same checkpoint can also run in a single step, scoring zero point nine zero on GenEval at zero point three five seconds. And a rank sixty-four LoRA, training only zero point four billion parameters, reaches zero point nine zero at two steps, close to the zero point nine one of full tuning.

### Comparison

So how much does the new feature-moment loss actually add? For that we need the controlled ablations, all run on fully tuned Z-Image at two steps.

Adding the feature-moment loss on top of the same DMD baseline lifts GenEval from seventy-seven point four six to seventy-seven point eight one, Chinese LongText from ninety-three point eight seven to ninety-four point two five, and English from eighty-nine point seven nine to ninety point two. All five aggregate metrics the paper reports go up.

Going the other way, removing the DMD loss drops GenEval to seventy-six point nine, and removing both gives seventy-six point two four. The authors' own conclusion is that moment matching helps, but the DMD correction remains central.

They also compared feature sources. An external DINOv2 encoder gives seventy-seven point seven five GenEval and ninety-four point one three on Chinese LongText. EMA features give seventy-seven point eight one and ninety-four point two five, very close. The live features of the model being trained give seventy-seven point five one, and teacher features seventy-seven point three six.

This figure is the qualitative comparison, a sushi poster and a coffee-shop menu. According to the paper, full IDM keeps dense text more consistent, especially in the multi-line menu.

One caution on the APEX comparison. The paper itself says APEX was trained on a smaller collection, so the comparison reflects the full training setup, not just the method. And on DPG-Bench, APEX actually scores higher, eighty-eight point six four against IDM's eighty-six point eight two.

My read is that the most solid finding here is that the generator's own EMA features can stand in for an external encoder in moment matching, performing close to DINOv2 while dropping a model. As for the long-text lead, the outside comparisons can't separate how much comes from the new loss and how much from five million text-rendering images. What can be pinned on the loss itself is a gain of under half a point in the ablations.

### Limitations

The paper has no dedicated limitations section, so these points come from its own statements and tables.

First, the feature-moment loss matches only means and per-dimension standard deviations pooled across all prompts. It doesn't condition on the prompt, and it doesn't match the full feature distribution. As the authors say themselves, equal moments don't mean equal distributions.

Second, queued features come from earlier model states, so the statistics only approximate the current distribution.

Third, on some metrics two steps still have a gap to close. In Table 5, IDM's two-step English LongText score of ninety point two is below Z-Image-Turbo at eight steps, ninety-one point seven four. In Table 2, IDM on Z-Image scores zero point seven eight on GenEval, while the Z-Image base model, at about a hundred steps, scores zero point eight four.

Fourth, every evaluation is an automatic benchmark. The paper reports no human evaluation.

### Future work

The paper doesn't spell out future work either. In the conclusion, the authors say their experiments on models up to twenty billion parameters support using the generator's predictions and representations together for few-step text-to-image generation. They say the code is available.

My read is that the interesting idea here is self-sufficiency: the generator is the student and also supplies the features used to judge it. If that holds for more expensive settings like video, few-step distillation gets cheaper, though this paper only tests images.

My read is that the next thing to watch is what happens when methods like APEX get the same training data. How much of the long-text gap survives will tell us how much this new loss really carries.

That's today's Daily Frontier Papers. The paper is linked in the description.

