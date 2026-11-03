# FrameForge: Harnessing Priors of Video Foundation Models for Unified Image Generation

**把图像生成当成视频的一帧**

- 论文 Paper: [arXiv 2610.36545](https://arxiv.org/abs/2610.36545)
- 单位 Affiliation: 中国科学技术大学 · FrameX.AI
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-03
- 视频 Videos: [YouTube 中文](https://youtu.be/myTF2aM8AcA) · [YouTube English](https://youtu.be/FMVxlhkLa3k) · [B站](https://www.bilibili.com/video/BV1h44g6UEkz)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇中国科学技术大学和 FrameX.AI 的论文，名字叫 FrameForge，意思是，借视频生成模型学到的先验，来做统一的图像生成。

它想回答的问题是，一个模型，能不能同时做好主体生成、线稿生图和图像编辑这几种很不一样的任务，而且所有任务只共用一个小小的微调插件。

### 问题

先说什么叫条件图像生成。给模型一张输入图和一段文字，让它生成一张新图。输入图里要保留什么，每个任务都不一样。

比如主体生成，是给一只狗的照片，让它出现在新场景里，要保住的是这只狗本身的样子。线稿生图，是给一张边缘图，也就是 canny 图，要保住的是轮廓和结构。图像编辑，是按指令改一张图，要保住的是没让改的地方。

难点在于，这几种任务里，输入和目标之间的关系差别很大，要在一个统一的框架里同时建模，并不容易。

### 以前的方法

以前的做法，主要以图像模型为中心。一类是在大规模多任务数据上训练一个统一模型，比如 OmniGen。另一类是在现成的图像生成器上，加控制分支或者轻量的 adapter，比如 OminiControl 和 EasyControl。

作者认为，这些方法通常只是把条件当成额外的信号塞进去，没有明确建模输入和目标差多少，以及怎么从一个变成另一个。

另一条路，是借用视频生成模型。视频模型见过大量画面随时间的变化，所以有研究把输入图和目标图，当成视频里的两帧来生成，比如 RealGeneral、DRA-Ctrl 和 ChronoEdit。其中 RealGeneral，是这篇论文部分作者之前的工作。

问题在于，这些方法通常把输入和目标放在固定的时间位置上，不管什么任务、什么样本，两帧之间的距离都一样。作者认为，这个固定间隔，会限制一个共享 adapter 能做到的程度。

### 核心想法

作者的关键观察是，不同任务里输入和目标的差别，可以理解成视频里隔了多少帧。

这张图是他们的动机。上面是自然视频里，隔不同帧数的两帧之间的差异，用 LPIPS 衡量，LPIPS 是一种看两张图在感知上差多少的指标。下面是三种任务里，输入图和目标图的差异分布。

可以看到，图像编辑的差异比较小，主体生成和线稿生图的差异更大，而且这些差异的范围，和自然视频里帧与帧之间的变化范围大幅重叠。

同一个任务内部，分布也很宽。所以作者认为，光靠任务标签，定不了该隔多远，应该根据每个输入自己来决定，这就是输入自适应的虚拟时间间隔。

另外，知道隔多远还不够，还要知道输入是怎么变成目标的。FrameForge 的两个模块，就分别对应这两件事。

### 方法

先看整体结构，就是这张图。底座是腾讯混元的 HunyuanVideo 1.5 视频扩散模型，主干权重全部冻结，只插入一个所有任务共享的 LoRA，LoRA 是一种只训练少量参数的微调方法。

目标图被当作视频里要预测的那一帧，输入图和文字作为条件，一起送进模型。

第一个模块叫 MoTE，时间专家混合。视频模型用一种叫 3D RoPE 的位置编码，告诉模型每一帧在时间轴上的位置。MoTE 准备了三套这样的时间位置编码，叫三个专家，对应三种虚拟间隔，分别相当于视频里的 17 帧、69 帧和 121 帧。

输入图放在第一个位置，目标图放在最后一个位置，选哪个专家，就等于决定两张图在时间上隔多远。

由谁来选呢，一个轻量的路由器。它看文字、输入图和当前带噪声的目标图，每个样本只选一个专家，而且在去噪的每一步都重新选。

只选一个是离散操作，没法直接求梯度，作者用了直通估计，前向只用选中的专家，梯度走软概率。另外还加了一个负载均衡损失，防止路由器总是选同一个专家。

第二个模块叫 CoTR，时间推理链。它在输入图和目标图之间，插入一个从随机噪声初始化的中间隐变量，形成输入、推理、目标这样一条链。

这个中间变量没有任何直接监督，也不会被解码成图，它只是在网络里不断和文字、输入、目标交互，充当传递信息的通道。

但光插进去，模型不一定会用它，因为目标还能直接看到文字和输入图。所以在训练的中间阶段，作者以一半的概率加一个注意力掩码，切断目标和文字、输入之间的直接注意力，逼着信息从这个中间变量走。

训练分三个阶段。第一阶段只训练路由器，第二阶段加入中间变量，同时训练路由器和 LoRA，并使用掩码，第三阶段去掉掩码，用完整注意力继续训练，和推理时的设置一致。

### 实验和效果

实验设置。三个任务各用 17 万个训练样本，一共 51 万，图片都缩放到 512 乘 512。可训练参数 1.82 亿，在两张 A800 显卡上训练了大约 6 天。

评测用三个基准。主体生成用 DreamBench，线稿生图用 COCO2017 验证集的 5000 张图，图像编辑用 GEdit-Bench 英文部分的 606 个样本。主体生成和编辑的分数，分别由 GPT-4o 和 GPT-4.1 来打。

先看主体生成，在表 1。FrameForge 的总分是 0.739，是所有对比方法里最高的，第二名 ChronoEdit 是 0.729。

不过在主体一致性这一项，它是 0.739，排第二，ChronoEdit 是 0.757。画面质量这一项，它也不是最高，EasyControl 和 IP-Adapter 等方法更高，但这两个的主体一致性低很多。

线稿生图在表 2。结构相似度 SSIM 是 0.447，最高，比第二名 EasyControl 相对高百分之 7.2。图文一致性 CLIP-T 是 0.323，和 ControlNet 持平。

但 FID 是 23.47，不是最好，表里五个对比方法，有四个比它低，最低的是 ControlNet 的 20.42。FID 衡量的是生成图整体分布和真实图有多接近，越低越好。作者的解释是，FID 会受底座模型自身画面统计特征的影响，所以要和其他指标一起看。

图像编辑在表 3，这是提升最明显的地方。在开源方法里，FrameForge 的总分最高，从 ChronoEdit 的 5.08 提到 6.12。

动作变化这一类编辑，比如把猫改成睡觉的姿势，分数从 5.76 提到 6.84，超过了闭源的 Gemini 2.5 和 Seedream 4.0，但还不如 GPT-4o 的 7.81。

论文还指出，用图像模型做底座的方法，在动作变化上普遍表现很差，比如 ICEdit 只有 1.13。作者认为，这说明视频模型学到的动态先验确实有用。

再看路由器到底学到了什么，就是这张图。线稿生图主要选长间隔专家，尤其在去噪早期。主体生成和编辑一开始偏向中等间隔，随着目标图的内容逐渐成形，都转向短间隔专家，编辑转得更早、更明显。

也就是说，同一个任务里，选择也会随去噪过程变化，而不是每个任务固定一个间隔。这和前面图 1 的观察对得上。

### 和以前方法的对比

消融实验在表 4，这里能看清每个模块的贡献。每个任务单独训一个 LoRA，编辑得分是 5.99。三个任务共用一个 LoRA，主体生成反而从 0.676 涨到 0.722，但编辑掉到 4.94，说明编辑受任务干扰最大。

单加 MoTE，编辑回到 5.77，单加 CoTR 是 5.56，都还不如单独训练。两个一起加，编辑到 6.12，FID 降到 23.47，三个任务都是这一组里最好的。

其他消融也有意思。专家从 1 个加到 3 个，线稿和编辑变好，加到 5 个、7 个，主体生成略好，另外两项变差。中间隐变量用 1 个最好，用 2 个、3 个反而下降。

换一个底座，HunyuanVideo 1.0，FrameForge 也比共用 LoRA 的基线三项都好，编辑从 4.38 提到 5.00。

我的判断是，这篇最有价值的不是某个分数，而是一个思路，把不同的图像任务看成视频里不同的时间间隔，让模型自己挑。它不改视频主干的权重，却在最受任务干扰的编辑上，带来了最明显的提升。

我的判断是，和 ChronoEdit 的比较要谨慎看。两边底座不一样，一个是 HunyuanVideo 1.5，一个是 Wan2.1 的 14B 模型，论文没有在同一个底座上比较这两种方法，所以分数差距不完全是方法本身带来的。

### 局限

论文没有单独写局限，下面几点，来自论文里的数字和设置。

第一，只测了三种任务，图片都在 512 乘 512，更高分辨率和更多任务类型，论文没有验证。

第二，编辑的整体分数，和闭源模型还有明显差距，FrameForge 是 6.12，GPT-4o 是 8.01，Gemini 2.5 是 7.89。编辑的画面质量这一项，它也低于 ChronoEdit。

第三，评测主要靠 GPT 模型自动打分，和 FID 这类指标，论文没有报告人工评测，也没有报告生成速度，推理用的是 50 步去噪。

### 未来方向

论文没有专门讨论未来方向。作者在结论里强调，这个框架让视频基础模型保持冻结，就能同时适配三种任务。

我的判断是，下一步值得看的是，这种虚拟时间间隔的思路，能不能扩展到更多任务，比如多张参考图，以及换到更强的视频底座上，收益还在不在。

我的判断是，更大的信号是，视频生成模型正在变成图像任务的一种底座，尤其是需要动作和空间变化的编辑，视频先验的优势最明显。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the University of Science and Technology of China and FrameX.AI. It's called FrameForge, and the idea is to borrow what a video generation model already knows and use it for image generation.

The question it asks: can one model handle subject-driven generation, edge-to-image generation and image editing, three quite different jobs, with a single small adapter shared across all of them?

### Problem

First, what conditional image generation means. You give the model an input image and some text, and it produces a new image. What it has to keep from the input depends on the task.

In subject-driven generation, you show it a photo of a dog and ask for that dog in a new scene, so the dog's identity has to survive. In edge-to-image, the input is a canny edge map, and the outline and structure have to survive. In editing, you give an instruction, and everything you didn't ask to change has to survive.

The hard part is that the relationship between input and target is very different across these tasks, and modeling all of them inside one framework is not easy.

### Prior work

Most earlier approaches are built around image models. One line trains a unified model on large multi-task datasets, like OmniGen. The other bolts control branches or lightweight adapters onto an existing image generator, like OminiControl and EasyControl.

The authors argue that these methods mostly inject the condition as an extra signal, without explicitly modeling how far the target is from the input, or how one turns into the other.

A second route uses video generation models. Video models have seen a huge amount of visual change over time, so some work treats the input and target images as two frames of a video, for example RealGeneral, DRA-Ctrl and ChronoEdit. RealGeneral is earlier work by some of the same authors.

The catch is that these methods put the input and target at fixed time positions. Whatever the task or the sample, the two frames are always the same distance apart. The authors think that fixed gap limits what one shared adapter can do.

### Key idea

The key observation is that the gap between input and target in different tasks can be read as how many frames apart they are in a video.

This figure is the motivation. The top shows how different two frames of a natural video are, at different frame gaps, measured with LPIPS, a metric for how perceptually different two images look. The bottom shows the same distance between input and target for the three tasks.

Editing has small differences, while subject-driven and edge-to-image have larger ones, and those ranges overlap heavily with the frame-to-frame changes in natural video.

Within a single task the spread is wide too. So the authors argue a task label can't tell you how far apart the frames should be. Each input should decide for itself, and that's what they call an input-adaptive virtual time interval.

Knowing how far apart is only half of it, though. You also need to know how the input turns into the target. FrameForge has one module for each.

### Method

Here's the overall architecture. The backbone is Tencent Hunyuan's HunyuanVideo one point five video diffusion model. Its weights stay frozen, and only one LoRA, a way of fine-tuning a small number of extra parameters, is added and shared by every task.

The target image is treated as the video frame to be predicted, and the input image and text go in as conditions.

The first module is MoTE, a mixture of temporal experts. Video models use a positional encoding called 3D RoPE to tell the model where each frame sits in time. MoTE builds three versions of that temporal encoding, called experts, each for a different virtual gap, equivalent to video spans of seventeen, sixty-nine and one hundred twenty-one frames.

The input goes in the first position and the target in the last, so picking an expert means picking how far apart in time the two images are.

The choice is made by a lightweight router. It looks at the text, the input image and the current noisy target, picks one expert per sample, and picks again at every denoising step.

Picking exactly one is a discrete choice with no direct gradient, so the authors use a straight-through estimator: the forward pass uses the chosen expert, and gradients flow through the soft probabilities. A load-balancing loss keeps the router from always choosing the same expert.

The second module is CoTR, a chain of temporal reasoning. It inserts an intermediate latent between the input and the target, initialized from random noise, forming a chain of input, reasoning, target.

This latent gets no direct supervision and is never decoded into an image. It just keeps interacting with the text, input and target inside the network, acting as a channel for information.

But inserting it doesn't mean the model will use it, because the target can still look straight at the text and the input. So in the middle stage of training, with probability one half, an attention mask cuts the direct attention between the target and the text and input, forcing information to pass through the latent.

Training has three stages. Stage one trains only the router. Stage two adds the latent, trains the router and the LoRA together, and applies the mask. Stage three drops the mask and keeps training with full attention, which matches the setup used at inference.

### Results

The setup. Each task gets a hundred seventy thousand training samples, five hundred ten thousand in total, all resized to five hundred twelve by five hundred twelve. There are one hundred eighty-two million trainable parameters, trained on two A800 GPUs for about six days.

There are three benchmarks. Subject-driven generation uses DreamBench, edge-to-image uses five thousand images from the COCO twenty seventeen validation set, and editing uses six hundred six samples from the English part of GEdit-Bench. Subject and editing scores are judged by GPT-4o and GPT-4.1 respectively.

Subject-driven first, in Table one. FrameForge's overall score is zero point seven three nine, the highest of all compared methods. Second is ChronoEdit at zero point seven two nine.

On subject consistency alone, though, it's zero point seven three nine and ranks second, behind ChronoEdit at zero point seven five seven. On perceptual quality it isn't the top either, with EasyControl and IP-Adapter among those scoring higher, but those two keep the subject much less well.

Edge-to-image is Table two. Structural similarity, SSIM, is zero point four four seven, the best, a seven point two percent relative gain over the runner-up, EasyControl. Text alignment, CLIP-T, is zero point three two three, level with ControlNet.

FID, though, is twenty-three point four seven, not the best. Four of the five compared methods score lower, and ControlNet is lowest at twenty point four two. FID measures how close the overall distribution of generated images is to real ones, and lower is better. The authors' explanation is that FID is affected by the backbone's own appearance statistics, so it should be read alongside the other metrics.

Editing is Table three, and that's where the gain is largest. Among open-source methods FrameForge has the best overall score, up from ChronoEdit's five point zero eight to six point one two.

On motion-change edits, like turning a cat into a sleeping pose, it goes from five point seven six to six point eight four, ahead of the closed-source Gemini two point five and Seedream four point zero, but still behind GPT-4o at seven point eight one.

The paper also points out that methods built on image backbones do badly on motion change, with ICEdit at just one point one three. The authors take this as evidence that the motion priors learned by video models really help.

Now, what did the router actually learn? That's this figure. Edge-to-image mostly picks the long-gap expert, especially early in denoising. Subject-driven and editing start out favoring the medium gap, then shift to the short-gap expert as the target's content takes shape, and editing shifts earlier and more strongly.

So even within one task, the choice changes over the denoising process rather than each task getting one fixed gap. That lines up with the observation in Figure one.

### Comparison

The ablation in Table four shows what each piece contributes. Training a separate LoRA per task gives an editing score of five point nine nine. Sharing one LoRA across all three tasks actually lifts subject-driven from zero point six seven six to zero point seven two two, but editing drops to four point nine four, so editing suffers most from task interference.

Adding MoTE alone brings editing back to five point seven seven, and CoTR alone gives five point five six, both still below separate training. With both, editing reaches six point one two and FID falls to twenty-three point four seven, the best of this group on all three tasks.

The other ablations are interesting too. Going from one expert to three improves edge-to-image and editing, while five or seven experts help subject-driven slightly and hurt the other two. One reasoning latent works best, and two or three make things worse.

On a different backbone, HunyuanVideo one point zero, FrameForge also beats the shared-LoRA baseline on all three tasks, with editing going from four point three eight to five point zero zero.

My read is that the value here isn't any single score, it's the framing: treat different image tasks as different time gaps in a video, and let the model choose. It leaves the video backbone's weights untouched and still gives its biggest gain on editing, the task hurt most by interference.

My read is that the comparison with ChronoEdit needs care. The backbones differ, HunyuanVideo one point five versus a fourteen billion parameter Wan two point one model, and the paper doesn't compare the two methods on the same backbone, so the gap isn't purely down to the method.

### Limitations

The paper has no limitations section, so these points come from its own numbers and setup.

First, only three tasks are tested, all at five hundred twelve by five hundred twelve. Higher resolutions and other kinds of tasks aren't evaluated.

Second, the overall editing score is still well behind closed-source models, six point one two against eight point zero one for GPT-4o and seven point eight nine for Gemini two point five. On editing perceptual quality it also trails ChronoEdit.

Third, evaluation relies on GPT-based automatic judging and metrics like FID. The paper reports no human evaluation and no generation speed, and inference uses fifty denoising steps.

### Future work

The paper doesn't discuss future directions explicitly. In the conclusion, the authors stress that the framework adapts to all three tasks while keeping the video foundation model frozen.

My read is that the thing to watch is whether this virtual-time-gap idea extends to more tasks, such as multiple reference images, and whether the gains hold on stronger video backbones.

My read is that the bigger signal is video generation models becoming a foundation for image tasks, with the clearest advantage on edits that need motion and spatial change.

That's today's Daily Frontier Papers. The paper is linked in the description.

