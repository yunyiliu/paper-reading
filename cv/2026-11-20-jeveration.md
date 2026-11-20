# Jeveration: System One Models Now Generate

**一次前向78毫秒出图**

- 论文 Paper: [arXiv 2611.24738](https://arxiv.org/abs/2611.24738)
- 单位 Affiliation: 新加坡国立大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-20
- 视频 Videos: [YouTube 中文](https://youtu.be/JBmWykl-PKY) · [YouTube English](https://youtu.be/xJRkfDt1x5g) · [B站](https://www.bilibili.com/video/BV18SypzVEyG)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇新加坡国立大学的论文，名字叫 Jeveration，标题的意思是，System One 模型现在也能生成图片了。

论文报告，它在一张 H200 显卡上，一次前向计算，就生成一张 512 乘 512 的图，端到端 78 毫秒。同一个模型，还能做图生图，和分割、深度估计这类稠密预测。

### 问题

现在的图像生成主要两条路，自回归是一个 token 一个 token 地生成，扩散模型是从噪声开始一步步去噪。两条路都要让模型按顺序跑很多次，延迟就是这么来的。

看表 1，同一张 H200，批大小为 1，自回归的 Janus-Pro-7B 要 576 步，出一张 384 像素的图要 9508 毫秒，20 步的 SANA 也要 587 毫秒。

作者想走第三条路，一次前向，把整张图定下来。出发点是一类叫 System One 的模型，代表是 Jev。这类模型不写长推理，一次并行查询，直接返回有类型的决策，比如选一个选项，或者给一个分数。

这类模型以前用来做选择、定位物体、给机器人动作和图片打分。作者的想法是，它预训练里学到的视觉和语义知识，也可以拿来生成图片。

### 以前的方法

一步出图以前也有人做。连续空间里有 consistency model、MeanFlow，还有把扩散模型蒸馏成一步、几步的 DMD2 和 SANA-Sprint。离散 token 这边，MaskGIT、Muse 这类掩码模型要采样好几轮，DiMO 把这种掩码模型蒸馏成了一步生成器。

另一条线是统一多模态模型，比如 Janus-Pro、Show-o、MMaDA，既能看图也能画图，但出图都要很多步。

Jeveration 借用了 DiMO 这类按 token 匹配分布的蒸馏思路，但生成器换成了一个冻结的视觉语言模型。

### 核心想法

最直接的做法，是让模型对每个位置，同时从码本里选一个码，码本就是图像分词器的词表。作者先这么试了，结果不行。看这张图，标着 no joint decision 的，就是各位置独立选码的结果，拼不成完整的图。

原因是，一句提示词可以对应很多张合理的图。打个比方，一只猫可以趴着，也可以站着，各位置各选各的，有的按趴着选，有的按站着选，拼起来就对不上。论文用公式说明，独立采样学不到位置之间的依赖。

作者的解法是，每张图抽一个共享的随机状态，所有位置都看着它做决定。各位置还是分别采样，但共用一个随机源，就能对齐到同一张图上。换一个随机状态，就是另一张完整的图。

但如果直接拿目标图片监督，模型会无视这个随机状态，因为它和目标图片没有关系。所以作者先训练一个模型学图像的分布，再用它来引导一次出图的决策。

### 方法

这张是架构图。底座是 Qwen3.5-4B 视觉语言模型，全程冻结，只加 LoRA 适配器和输出头，LoRA 就是只训练少量额外参数的微调方法。输入是提示词或图片，后面接一串决策 token，一次前向跑完。

生成图片时，分词器是冻结的 TA-TiTok，一张图是 128 个离散码，所以后面接 128 个决策 token，一个 token 决定一个码。这些 token 的初始值就是随机状态，每个位置有百分之六十的概率被遮住，其余填随机码，再加一点高斯噪声。

输出头有两个分支，一个从码本初始化，一个从零开始，加起来得到每个位置的码的概率。128 个码一次并行采出来，再由解码器还原成 512 像素的图。

做稠密预测时，决策 token 变成和输入图片对齐的空间查询。分割是每个位置选类别，深度和法向是每个位置给数值。

训练分两个阶段。第一阶段当掩码模型训，遮住一部分码，让模型根据剩下的码和提示词去猜，这样学到位置之间的依赖。

第二阶段学一次出图。决策模型拿随机状态一次采出整套码，再随机遮掉一部分，让两个掩码模型去猜。一个是冻结的第一阶段模型，代表真实图像的分布，另一个是辅助模型，一直在决策模型的样本上训练，代表它当前的分布。

只有当决策模型的码真能拼成合理的图，两边的预测才会一致，因为每个被遮位置的预测，取决于其他位置的码。作者用两边预测的差去更新决策模型，逼它通过共享随机状态协调各个位置。

最后还有一步修正，把模型最没把握的一半位置遮掉，让冻结模型补全，再让决策模型直接学这个结果。

冻结模型、辅助模型和修正，都只在训练时用。推理就是一次前向。训练第一阶段大约九万两千步，第二阶段六千七百步，一台四卡 RTX PRO 6000 的机器，大约四天。

### 实验和效果

先看文生图。表 1 的计时包含解码，Jeveration 出一张 512 像素的图是 77.8 毫秒，表里最快，GenEval 是 0.74。GenEval 是一个以物体为中心，评估图文是否对得上的 benchmark。

和统一模型比，它最快，GenEval 和人类偏好分数 HPSv2.1，都高于 JanusFlow、Show-o、Show-o Turbo、MMaDA 和 Janus-Pro-1B。同为 512 像素，它比四步的 Show-o Turbo，GenEval 高 0.11，延迟从 94.1 降到 77.8 毫秒。比 Janus-Pro-1B 和 7B，分别快 98 倍和 122 倍。

和只做生成的模型比，同为 512 像素，它的 GenEval 高于多步模型，0.74 对 0.49 到 0.65，快 5 到 8 倍。也高于 SANA-Sprint、FLUX.1-schnell 和 SDXL-DMD2 这几个一步模型，不过它们出的是 1024 像素。

再看图生图。同一个模型能把边缘图、深度图变成照片，给黑白照片上色，还能转成动漫、吉卜力和水彩风格。每一对左边是输入，右边是输出。

表 2，每个任务 341 个留出样本，HED 边缘 F1 是 0.880，深度 delta 1 是 0.802，上色边缘 F1 是 0.937。每次还是一次前向，91.7 毫秒。

稠密预测在 NYUv2 室内数据集上测。表 3，分割 mIoU 67.7，和同底座的纯稠密模型 67.8、67.3 基本持平，比论文引用的 DeepLabV3+ 高 14.2，深度误差低百分之三十二。

消融实验里，不做第二阶段，掩码模型一次出图，GenEval 只有 0.32。表 4，随机状态换成固定状态，GenEval 从 0.73 降到 0.70，HPS 从 28.27 降到 26.63。去掉修正，HPS 从 28.03 降到 27.92。两个输出分支一起用，也比单用一个好。

表 5，加上图生图和稠密预测一起训练，文生图几乎没掉，GenEval 0.74 对 0.73，HPS 28.06 对 28.28。作者的结论是，多个任务共用一套适配器，互相干扰很小。

### 和以前方法的对比

和统一多模态模型比，最大的区别是从很多步变成一步，一个冻结底座同时做生成和稠密预测。和一步蒸馏方法比，它的生成器不是专门的图像模型，底座参数一个没动，可训练参数大约 10 亿。

我的判断是，最有意思的不是 78 毫秒，而是它展示了，一个做决策的模型，靠共享随机状态和分布匹配，就能一次给出 128 个彼此协调的选择。

我的判断是，速度对比要看清条件。它是 512 像素，SANA-Sprint、FLUX.1-schnell 是 1024 像素，Janus-Pro 是 384 像素，延迟不能直接比。质量也没有全面领先，GenEval 低于 Janus-Pro-7B 的 0.80，DPG 是 78.24，低于 FLUX.1-schnell 的 85.97。

### 局限

论文没有单独的局限章节，下面几点来自它自己的结果。第一，整张图只有 128 个码，作者在图生图实验里也说，这些码抓的是整体结构，细节要靠源图来补。

第二，多任务有代价。表面法向误差 24.2 度，比纯稠密模型的 21.2 和 23.0 都差，作者推测，精细的局部几何对共享容量最敏感。

第三，评测比较窄。稠密预测只测了 NYUv2 一个数据集。上色比的是边缘结构，所以原样复制输入，得分 0.998，反而比模型的 0.937 高。

第四，实验只用了 Qwen3.5-4B 加 TA-TiTok 这一套组合。

### 未来方向

作者在结论里只写了一句，期待这个范式扩展到更多视觉任务和模态。

我的判断是，接下来要看两点，一是更高分辨率、更多码的时候，一次出图还能不能保持一致，二是在 DPG 上，和最强的生成模型还有明显差距，能不能补上。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from the National University of Singapore. It's called Jeveration, and the title says it plainly: System One models now generate.

The authors report a five hundred twelve pixel image in one forward pass, seventy-eight milliseconds end to end on a single H200. The same checkpoint also does image-to-image work and dense prediction, like segmentation and depth.

### Problem

Image generation today mostly takes one of two routes. Autoregressive models write the image token by token, and diffusion models denoise step by step. Either way the model runs many times in sequence, and that's the latency.

Table one makes it concrete. On the same H200 at batch size one, autoregressive Janus-Pro seven B takes five hundred seventy-six steps and about nine and a half seconds for a three hundred eighty-four pixel image. Even twenty-step SANA takes about five hundred eighty-seven milliseconds.

The authors want a third route: decide the whole image in one pass. Their starting point is System One models, with Jev as the example. These skip long reasoning and answer a query in one parallel pass with a typed decision, like picking an option or giving a score.

So far such models have been used for option selection, object localization, and scoring robot actions or images. The authors suggest that the visual and semantic knowledge from their pretraining could be put to work making images too.

### Prior work

One-step generation itself isn't new. In continuous space there are consistency models and MeanFlow, plus distillation methods like DMD2 and SANA-Sprint. On the discrete side, masked models like MaskGIT and Muse sample over several rounds, and DiMO distills such a model into a one-step generator.

The other line is unified multimodal models, such as Janus-Pro, Show-o, and MMaDA, which both understand and generate images, but still need many steps to do it.

Jeveration borrows the token-level distribution matching idea from DiMO, but the generator is a frozen vision-language model.

### Key idea

The naive approach is to have the model pick a codebook entry for every image position at once, the codebook being the image tokenizer's vocabulary. The authors tried that first, and it failed. In this figure, the images labeled no joint decision come from positions choosing independently, and the pieces don't fit.

Here's why. One prompt can match many valid images. Picture a cat that could be lying down or standing up. If each position decides alone, some commit to one cat and some to the other. The paper shows formally that independent sampling can't capture dependencies between positions.

The fix is a shared random state. Each image gets one random state, and every position decides conditioned on it. Positions still sample separately, but sharing one source of randomness lets them land on the same image. A different state gives a different, still coherent image.

But if you train directly on target images, the model just ignores the state, since it has nothing to do with the target. So the authors first learn the image distribution, then use it to guide the one-pass decisions.

### Method

Here's the architecture. The backbone is Qwen3.5-4B, a vision-language model that stays frozen, with LoRA adapters and output heads added. LoRA means training a small set of extra parameters. A prompt or image goes in, followed by decision tokens, all in one forward pass.

For images, the tokenizer is a frozen TA-TiTok with one hundred twenty-eight discrete codes per image, so there are one hundred twenty-eight decision tokens, one per code. They start from the random state: each position is masked with probability sixty percent, the rest get random codes, plus a little Gaussian noise.

The output head has two branches, one initialized from the codebook and one starting at zero, summed into a distribution over codes. All codes are sampled in parallel and decoded into a five hundred twelve pixel image.

For dense prediction, the decision tokens become spatial queries aligned with the input image. Segmentation picks a class per position, and depth and normals are numeric scores.

Training has two phases. Phase one trains a masked model: hide some codes and predict them from the rest and the prompt. That teaches how positions depend on each other.

Phase two learns the one-pass generator. The decision model samples a full set of codes from a random state, part of it is masked again, and two masked models fill it in. One is the frozen phase one model, standing for the real image distribution. The other is an auxiliary model that keeps training on the decision model's samples, tracking its current distribution.

The two only agree when the codes really fit together, because each masked prediction depends on the rest of the sample. Their difference updates the decision model, pushing it to coordinate positions through the shared state.

Finally, a correction step masks the half of the positions the model is least sure about, lets the frozen model fill them in, and trains the decision model to output that directly.

The frozen model, the auxiliary model, and the correction are training only. Inference is one forward pass. Training was about ninety-two thousand steps in phase one and sixty-seven hundred in phase two, roughly four days on one machine with four RTX PRO 6000 GPUs.

### Results

Text-to-image first. Timings in table one include decoding. Jeveration makes a five hundred twelve pixel image in seventy-seven point eight milliseconds, the fastest in the table, with a GenEval of zero point seven four. GenEval is an object-focused benchmark for whether an image matches its prompt.

Among unified models it's fastest, and beats JanusFlow, Show-o, Show-o Turbo, MMaDA, and Janus-Pro one B on GenEval and on HPS v2.1, a human-preference score. Against four-step Show-o Turbo at the same resolution, it's zero point one one higher on GenEval and drops latency from ninety-four point one to seventy-seven point eight milliseconds. It's ninety-eight and one hundred twenty-two times faster than Janus-Pro one B and seven B.

Against generation-only models at five hundred twelve pixels, it beats the multi-step ones on GenEval, zero point seven four versus zero point four nine to zero point six five, while running five to eight times faster. It also beats one-step SANA-Sprint, FLUX.1 schnell, and SDXL-DMD2, though those output ten twenty-four pixels.

Now image-to-image. The same checkpoint turns edge maps and depth maps into photos, colorizes grayscale images, and restyles photos as anime, Studio Ghibli, or watercolor. Input on the left, output on the right.

In table two, on three hundred forty-one held-out examples per task, HED edge F1 is zero point eight eight, depth delta one is zero point eight zero two, and colorization edge F1 is zero point nine three seven. Still one pass, ninety-one point seven milliseconds.

Dense prediction is tested on the NYUv2 indoor dataset. In table three, segmentation hits sixty-seven point seven mIoU, level with same-backbone dense-only models at sixty-seven point eight and sixty-seven point three, and fourteen point two above a reported DeepLabV3 plus result, with thirty-two percent lower depth error.

In the ablations, without phase two the masked model reaches only zero point three two GenEval in one pass. In table four, a fixed state instead of a random one drops GenEval from zero point seven three to zero point seven and HPS from twenty-eight point two seven to twenty-six point six three. Dropping the correction lowers HPS from twenty-eight point zero three to twenty-seven point nine two. Both head branches together beat either alone.

In table five, adding image-to-image and dense tasks barely moves text-to-image: GenEval zero point seven four versus zero point seven three, HPS twenty-eight point zero six versus twenty-eight point two eight. The authors conclude that one set of adapters can host several tasks with little interference.

### Comparison

Versus unified multimodal models, the change is many steps to one, with one frozen backbone doing both generation and dense prediction. Versus one-step distillation, the generator isn't a dedicated image model, its backbone weights never change, and about one billion parameters are trainable.

My read is that the interesting part isn't seventy-eight milliseconds. It's showing that a decision model, with a shared random state and distribution matching, can make one hundred twenty-eight coordinated choices at once.

My read is that the speed numbers need their conditions. Jeveration runs at five hundred twelve pixels, SANA-Sprint and FLUX.1 schnell at ten twenty-four, Janus-Pro at three eighty-four, so latencies aren't like for like. Quality isn't ahead across the board either: GenEval trails Janus-Pro seven B at zero point eight, and DPG at seventy-eight point two four is below FLUX.1 schnell at eighty-five point nine seven.

### Limitations

There's no limitations section, so these come from the paper's own results. First, an image is only one hundred twenty-eight codes, and in the image-to-image experiments the authors note the codes carry global structure while the source image supplies fine detail.

Second, multitasking costs something. Surface normal error is twenty-four point two degrees, worse than dense-only models at twenty-one point two and twenty-three, and the authors suggest fine local geometry is most sensitive to shared capacity.

Third, evaluation is narrow. Dense prediction is tested only on NYUv2. And colorization is scored on edge structure, so just copying the input scores zero point nine nine eight, above the model's zero point nine three seven.

Fourth, the experiments use only one combination, Qwen3.5-4B with the TA-TiTok tokenizer.

### Future work

The conclusion has one line of outlook: the authors expect the paradigm to extend to more visual tasks and modalities.

My read is that two things matter next: whether one-pass generation stays coherent at higher resolution with more codes, and whether the clear DPG gap to the strongest generators can be closed.

That's today's Daily Frontier Papers. The paper is linked in the description.

