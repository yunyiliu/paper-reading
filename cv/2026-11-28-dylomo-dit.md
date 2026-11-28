# DyLoMo-DiT: Dynamic Looped Mixture-of-Experts Diffusion Transformers

**算力用在刀刃上扩散模型新架构**

- 论文 Paper: [arXiv 2611.30844](https://arxiv.org/abs/2611.30844)
- 单位 Affiliation: Lernie Li 等（论文未列单位）
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-28
- 视频 Videos: [B站](https://www.bilibili.com/video/BV19A1MzGEEW)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇图像生成架构方面的论文，作者是 Lernie Li 等四位研究者，论文名叫 DyLoMo-DiT，意思是动态循环的混合专家扩散 Transformer。

它想回答一个很实际的问题，给定同样多的参数、同样多的计算量，扩散模型能不能把这些预算用得更聪明。

论文首页的这些图，是它放大之后的文生图模型画的。论文首页还列出了代码和模型权重的链接。

### 问题

先说背景。现在主流的图像生成模型，很多是 DiT，也就是用 Transformer 来做扩散去噪。

DiT 想变强，通常就是加宽或者加层。问题是，加宽会同时增加参数和每个 token 的计算量，加层也会同时增加深度和参数。参数、计算、深度，三样东西被绑在了一起。

还有一点。扩散模型生成一张图，要从纯噪声一步步去噪。论文引用以前的研究指出，噪声大的那些步主要在定整体结构，噪声小的步在补细节。但 DiT 在每一步用的都是同样深的网络，算力是平均分配的。

### 以前的方法

以前有两类改进。第一类是混合专家，也就是 MoE，比如 DiT-MoE。每个 token 只激活少数几个专家，参数可以放很大，计算量基本不变，代价是要存很多参数。

第二类是循环深度，把同一组层反复用好几遍，深度增加了，参数不用多存一份。已经有几篇工作把循环用在 DiT 上，但它们的前馈网络还是稠密的，参数和计算依然绑在一起。

作者说，据他们所知，还没有扩散 Transformer 同时解开这两种绑定，并且按噪声水平分配深度。

而且单纯加循环不一定有用。论文的消融里，在 DiT-MoE-B 的预算下，每一步都固定跑两遍循环，FID 几乎没变。

### 核心想法

所以这篇的核心想法是，把参数、计算、深度这三样资源拆开，分别分配。

专家负责在不同 token 之间分配参数，共享权重的循环负责用很少的参数换来深度，省下的参数再拿去加专家。最后，循环跑几遍，由噪声水平决定，算力往需要的地方放。

### 方法

具体看这张结构图。左边是普通 DiT 和 MoE-DiT，右边是 DyLoMo-DiT，它的物理层数更少，中间一段层共享权重，会被重复执行好几遍。

第一个部件是多头稀疏专家。除了第一层和最后一层，每层的前馈网络都先把输入切成几个头，每个头有自己的路由器，从自己的专家里挑几个。另外还有一个永远开着的共享专家。

以 XL 规模为例，4 个头，每个头 16 个专家里选 6 个，所以每个 token 每层会用到 24 个路由专家。负载均衡不用额外的损失函数，而是借鉴 DeepSeek-V3 的偏置调节方法。

第二个部件是共享权重的循环。网络分成前段、中段、后段，中段的层重复执行 r 遍，这几遍共用同一套参数。每一遍写回残差时乘以 r 分之一，防止更新叠加过头。每一遍都会重新选专家。

第三个部件是按噪声分配循环次数。推荐的设置是，噪声小的一半时间步跑 3 遍，噪声大的一半只跑 1 遍，平均下来正好等于每步跑 2 遍的算力。以 XL 为例，一个 token 在低噪声时要经过 30 层，高噪声时只经过 16 层。

这个循环次数是模型的一部分，训练和采样都用同一个设置。如果训练时随机用 1 到 3 遍，同一套权重就可以在推理时换不同的设置。

### 实验和效果

实验设置。所有类别条件模型都在 ImageNet 256 分辨率上，用和 DiT 一样的流程训练 40 万步，用 FID 评估，越低越好。每个 DyLoMo-DiT 都和对比模型对齐总参数量和单次前向的计算量。

先看和 DiT 的对比，在表 2。在 B、L、XL 三个规模上，FID 分别降低了 11%、14% 和 24%。XL 规模上，DiT 是 5.57，DyLoMo-DiT 是 4.22。

表里还有一行，是同样配方但不加循环的版本。它解释了一部分提升，加上循环和按噪声分配之后，FID 又分别降了 3%、8% 和 16%，规模越大，循环的贡献越大。

再看和 DiT-MoE 的对比，在表 3。在 DiT-MoE 自己的预算下，FID 在 S、B、L 三个规模上分别降低了 36%、56% 和 56%。L 规模上，从 7.92 降到 3.50。

作者还把对手放大来比。DiT 放大到三倍参数和三倍计算，FID 是 4.75，仍然比原规模的 DyLoMo-DiT-XL 差 13%。原版 DiT-MoE-XL 参数是它的 6.2 倍，FID 是 6.87，差了 63%。

表 4 是一步步加上设计的消融。从 DiT-MoE-B 的 16.06 开始，换成不循环的单遍版本，降到 7.93。加上每步固定两遍循环，是 7.94，几乎不动。改成按噪声分配次数，降到 7.01。改成三个路由头，到 5.61，再把专家切得更细，到 5.28。

论文还报告，按噪声分配，大约相当于多跑一遍循环。比如在两遍的算力下，FID 是 7.01，接近每步都跑三遍的 6.72。反过来，把多的循环放在高噪声那一半，B 规模的 FID 会变差 33%。

图 7 有个有意思的发现。路由器没有直接拿到时间步信息，但它自己学会了按噪声分工，专家的使用在时间步 550 到 560 之间突然切换，47% 的专家有八成以上的调用集中在其中一段。

最后是文生图。作者训练了一个放大版，总参数 154.8 亿，每个图像 token 激活 10.5 亿。在 GenEval 上，用作者自己的评测流程，它得到 0.714，FLUX.1-schnell 是 0.673，FLUX.1-dev 是 0.671。论文报告它的单次前向计算量只有两个 FLUX 模型的 24% 和 22%。

### 和以前方法的对比

和以前的方法比，区别在于以前的 DiT 和 MoE-DiT 每个噪声水平用同样的深度，以前的循环 DiT 用稠密前馈网络。这篇把专家、循环和按噪声分配放在同一个预算里一起配。

不过要注意几个细节。DiT-MoE 的结果，是作者在自己的训练流程下重新实现的，不是 DiT-MoE 原论文报告的数字。40 万步之后的长训练曲线，是单次运行的结果。

我的判断是，消融里最有说服力的，是在这个预算下，固定两遍循环几乎没有收益，按噪声分配之后才有用，这说明收益来自把算力放对地方，而不只是网络更深。

我的判断是，文生图那组对比要谨慎看。作者自己说只训练了一个模型，没有做消融，GenEval 差距在 0.03 以内不做解读。而且作者重算的 FLUX.1-schnell 分数比公开值低了 0.02 到 0.04，按公开值算，领先就小很多。

### 局限

论文没有单独的局限章节，下面是从正文里能看到的几点。

第一，论文对齐的计算量是乘加次数，没有报告实际生成速度。专家路由和循环在真实硬件上要多花多少时间，论文没有给。

第二，混合专家省的是计算，不是存储。文生图模型总参数 154.8 亿，比 FLUX 的 120 亿还多，存放这些权重需要的显存并没有变小。

第三，循环次数的设置要在训练时定下。想在推理时自由切换，就得用随机次数训练，论文报告这样跑两遍时 FID 是 4.68，比专门按两遍训练的 4.36 差 7%。

### 未来方向

作者在附录里做了一个初步实验，不改结构，也不加参数，直接用文生图的权重做起点，训练图生视频。论文报告训练能正常收敛，图像阶段训练得越久，视频的验证损失越低。他们也在一个内部视频模型上看到了多跑几遍循环的收益。

我的判断是，按噪声水平分配深度，这个思路很可能会被更大的图像和视频模型借鉴。真正要看的是，在实际推理速度和显存上，它还能不能保持这样的优势。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is about image generation architecture, from Lernie Li and three co-authors. It's called DyLoMo-DiT, short for Dynamic Looped Mixture-of-Experts Diffusion Transformers.

The question is a practical one. If you hold the parameter count and the compute fixed, can a diffusion model spend that budget more wisely?

The images on the first page come from the scaled-up text-to-image version of the model. The authors also list links to their code and weights.

### Problem

Some background first. Many of today's image generators are DiTs, diffusion models that use a Transformer to do the denoising.

To make a DiT stronger, you usually make it wider or deeper. But going wider raises both the parameter count and the compute per token, and adding layers raises both depth and parameters. Parameters, compute and depth are tied together.

There's a second issue. A diffusion model builds an image by denoising step by step, starting from pure noise. Citing earlier work, the paper notes that the high-noise steps mostly set the global structure, while the low-noise steps refine detail. Yet a DiT uses the same depth at every step, so compute is spread evenly.

### Prior work

There have been two lines of improvement. The first is mixture of experts, or MoE, as in DiT-MoE. Each token activates only a few experts, so the parameter count can grow while compute stays roughly flat. The price is storing all those parameters.

The second is recurrent depth: reuse the same group of layers several times, so the network gets deeper without storing another copy of the weights. A few papers have looped DiTs this way, but they keep dense feed-forward layers, so parameters and compute stay coupled.

The authors say that, to their knowledge, no diffusion Transformer has removed both couplings while also allocating depth by noise level.

And looping alone doesn't automatically help. In the paper's own ablation, at the DiT-MoE-B budget, running a fixed two passes at every step leaves FID essentially unchanged.

### Key idea

So the core idea is to pull the three resources apart, parameters, compute and depth, and allocate each one deliberately.

Experts spread parameter capacity across tokens. A weight-shared loop buys depth for very few parameters, and the parameters it saves go into more experts. Finally, the number of loop passes depends on the noise level, so compute goes where it's needed.

### Method

Here's the architecture figure. On the left are a plain DiT and an MoE-DiT. On the right is DyLoMo-DiT, with fewer physical layers, and a middle segment of shared-weight layers that runs several times.

The first component is multi-head sparse experts. In every layer except the first and last, the feed-forward input is split into heads. Each head has its own router that picks a few of its own experts, and there's also one shared expert that is always on.

At the XL scale, that's four heads, each choosing six of its sixteen experts, so every token uses twenty-four routed experts per layer. Load balancing uses no auxiliary loss; it follows the bias-adjustment approach from DeepSeek-V3.

The second component is the weight-shared loop. The network has a front, a middle and a back segment, and the middle layers run r times with one set of parameters. Each pass writes to the residual scaled by one over r, so updates don't pile up. Experts are re-selected on every pass.

The third component allocates passes by noise level. The recommended schedule runs three passes on the low-noise half of the timesteps and one pass on the high-noise half, which averages out to the compute of two passes everywhere. At the XL scale, a token goes through thirty layers at low noise and sixteen at high noise.

The schedule is part of the model, used the same way in training and sampling. If you instead train with one to three passes drawn at random, a single set of weights can run under different schedules at inference.

### Results

The setup. All class-conditional models are trained on ImageNet at two fifty six resolution for four hundred thousand steps, using the DiT recipe, and scored by FID, where lower is better. Each DyLoMo-DiT is matched to its baseline in total parameters and forward compute.

First, against DiT, in Table 2. At the B, L and XL scales, FID drops by eleven, fourteen and twenty-four percent. At XL, DiT scores five point five seven and DyLoMo-DiT four point two two.

The table also has a no-loop variant with the same recipe. It explains part of the gain. Adding the loop with noise-dependent passes cuts FID by a further three, eight and sixteen percent, so the loop matters more as the model grows.

Next, against DiT-MoE at its own budget, in Table 3. FID drops by thirty-six, fifty-six and fifty-six percent at the S, B and L scales. At L, that's seven point nine two down to three point five.

The authors also scale the baselines up. A DiT with three times the parameters and compute reaches four point seven five, still thirteen percent worse than DyLoMo-DiT-XL at its original size. The original DiT-MoE-XL, with six point two times the parameters, reaches six point eight seven, sixty-three percent worse.

Table 4 adds the design pieces one at a time. Starting from DiT-MoE-B at sixteen point zero six, the single-pass version gets to seven point nine three. A fixed two-pass loop gives seven point nine four, no change. Allocating passes by noise level brings it to seven point zero one. Three routing heads take it to five point six one, and finer experts to five point two eight.

The paper also reports that noise-based allocation is worth about one extra pass. At two-pass compute it reaches seven point zero one, close to the six point seven two of three passes everywhere. And flipping the allocation, putting the extra passes on high noise, makes the B-scale FID thirty-three percent worse.

Figure 7 has an interesting finding. The router never gets the timestep directly, yet it learns to split work by noise level. Expert usage switches abruptly between timesteps five fifty and five sixty, and forty-seven percent of experts get more than eighty percent of their calls in one of the two ranges.

Finally, text to image. The scaled model has fifteen point four eight billion total parameters, with one point zero five billion active per image token. On GenEval, using the authors' own evaluation pipeline, it scores zero point seven one four, against zero point six seven three for FLUX.1-schnell and zero point six seven one for FLUX.1-dev. The paper reports its forward pass uses twenty-four and twenty-two percent of the compute of the two FLUX models.

### Comparison

Compared with earlier work: DiT and MoE-DiT use the same depth at every noise level, and earlier looped DiTs use dense feed-forward layers. This paper configures experts, loops and noise-based allocation together under one budget.

A few details matter here. The DiT-MoE numbers come from the authors' own reimplementation under their training protocol, not from the DiT-MoE paper. And the long training curve beyond four hundred thousand steps is from single runs.

My read is that the most convincing result is in the ablation: at this budget, a fixed two-pass loop adds essentially nothing, and the gain only appears once passes are allocated by noise level. That says the win comes from putting compute in the right place, not just from a deeper network.

My read is that the text-to-image comparison needs care. The authors trained only one such model, with no ablation, and they don't interpret GenEval differences within zero point zero three. Their recomputed FLUX.1-schnell score is also zero point zero two to zero point zero four below its published values, and against those published values the lead gets much smaller.

### Limitations

The paper has no separate limitations section, so these are the points visible in the text.

First, compute is matched in multiply-accumulate operations, and the paper doesn't report actual generation speed. How much extra time routing and looping cost on real hardware isn't given.

Second, mixture of experts saves compute, not storage. The text-to-image model has fifteen point four eight billion total parameters, more than FLUX's twelve billion, so the memory needed to hold the weights doesn't shrink.

Third, the pass schedule is fixed at training time. To switch freely at inference you have to train with random pass counts, and the paper reports that this model, run at two passes, scores four point six eight FID, seven percent worse than a model trained for two passes, at four point three six.

### Future work

In the appendix, the authors run a first experiment that starts image-to-video training directly from the text-to-image weights, without changing the architecture or adding parameters. They report that training converges normally, and that the longer the image stage ran, the lower the video validation loss. They also see returns from extra passes in an in-house video model.

My read is that allocating depth by noise level is an idea larger image and video models are likely to borrow. The real test is whether the advantage holds up in measured inference speed and memory.

That's today's Daily Frontier Papers. The paper is linked in the description.

