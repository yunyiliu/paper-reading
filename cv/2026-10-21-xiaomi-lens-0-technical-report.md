# Xiaomi-Lens-0 Technical Report

**让模型自己画轮廓看图更细了**

- 论文 Paper: [arXiv 2610.23279](https://arxiv.org/abs/2610.23279)
- 单位 Affiliation: 小米 · Xiaoai Plus、SeerRay 团队
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-21
- 视频 Videos: [YouTube 中文](https://youtu.be/G3MQPjTAt2c) · [YouTube English](https://youtu.be/rSnMzHyD81A) · [B站](https://www.bilibili.com/video/BV1runz6wEL4)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇小米的论文，Xiaomi-Lens-0 技术报告，署名是 Xiaoai Plus 和 SeerRay 团队。

它是一个 4B 参数的多模态大模型，底座是 Qwen3.5。核心想法一句话，让模型从自己的视觉特征里画出 mask，也就是像素级的区域轮廓，用这个信号逼它看得更细。

这张图是它覆盖的六类任务，从按一句话框出或抠出目标，到机器人场景里的指代，再到数数、颜色、位置这类细粒度问答。

### 问题

现在的多模态大模型，整体看图回答问题已经不错，但一到细处就容易错，比如把挨在一起的东西数错，或者认错小区域的属性。

作者认为，原因在训练信号。只用文字监督，训练信号全部来自文字答案，从来没告诉模型，答案依据的是图里哪一块，这一块又延伸到哪里。

分割 mask 正好相反，它逐个像素标出哪些属于目标，是关于区域范围最直接的监督。

### 以前的方法

其实不少多模态模型已经用 mask 训练，做指代分割，也就是按一句话把对应的东西抠出来。但大多数另外挂一个编码器来取空间细节，比如 LISA 和 Sa2VA 用分层的分割编码器，Qwen3-VL-Seg 额外融合一个轻量图像编码器的特征。这样一来，模型自己的视觉表示并不需要学会区域细节。

也有方法从模型自己的视觉特征解码 mask，比如 PixelLM 和 MLLMSeg，但它们把视觉编码器冻住了，mask 的损失传不到共享的视觉特征上。

而且这两类工作，都只把 mask 当成做分割的手段。mask 监督能不能让模型在细粒度感知和视觉推理上也变强，作者说还没人研究过。

### 核心想法

这篇的思路是，不加外挂编码器，让模型直接从自己的视觉表示里预测 mask，这样 mask 的损失会一路传回和语言生成共用的网络。

第二个想法，叫 thinking with visual latents，用视觉隐变量思考。模型推理时每提到一个区域，就生成一个特殊 token，它也能解码成 mask，推理链的中间步骤也就有了像素级监督，而且看得见。

### 方法

先看结构，就是这张图。视觉编码器的输出分两路，一路照常经过空间合并，送进语言模型，另一路是合并之前的 patch 级特征网格，送给一个轻量的 mask 解码器。

为什么取合并之前的特征。作者引用了 LongCat-Next 的分析，空间合并下采样很猛，会丢掉高频细节，而合并之前的网格保住了空间结构，又是共享计算的一部分。

怎么告诉解码器画哪个区域。词表里加了一个特殊的 mask token，每生成一次，就取它在语言模型里的上下文状态，这就叫视觉隐变量。它取决于图像、指令和前面的回答，所以代表的就是当前说到的那个区域。

解码器很简单。patch 特征经过一层 Transformer 和投影，视觉隐变量也投影到同一空间，和每个 patch 打分，相当于一个临时的分类器，再把分数拼回网格，用两层转置卷积放大成 mask。

训练目标是两部分相加，文字做下一个 token 预测，mask 用逐像素的交叉熵加 Dice 损失。关键在于，mask 的损失会同时传回视觉编码器和语言模型。

再说用视觉隐变量思考。推理文字里每提到一个区域，就插一个 mask token。数数时每个实例一个，比较位置时每个物体一个。

要注意，推理时画出的 mask 只是一个输出分支，不会把裁剪图或区域特征喂回语言模型。它让中间步骤可以被检查，也让训练能监督这些状态。

论文给的例子里，问图里有几个画框，推理过程一个一个列出来，每个配一个框和一个 mask，最后得出 22 个。

训练分两步。第一步冻住底座，只训练 mask 解码器。第二步解冻整个模型，文字和 mask 一起训。

数据方面有两条合成流水线。一条是指代数据，先找物体、核对描述、生成 mask，再生成多种说法的指令，最后重新定位一遍，确认这句话只指向这个目标。

另一条是带定位的推理数据。先合成数数、属性、空间关系的问答，让一个独立的检查模型在看不到答案的情况下重做，答对才保留，再让推理模型写出带框的推理过程，给框配上 mask。

他们还做了一个新评测，叫 LensBench，覆盖普通照片、航拍、驾驶和全景四类图像，一共 1200 组、4800 条指令。每组分单目标和多目标，再分说出类别的显式指令，和不说类别、只靠用途或功能来描述的隐式指令，都经过人工审核。

### 实验和效果

先看指代定位，按一句话框出目标，RefCOCO 系列一共 8 个划分，见表 1。和底座 Qwen3.5 比，8 个划分全部提高，平均高 5.6 个百分点，在所有对比模型里 7 个划分第一。

提升最大的是 RefCOCO+，平均高 7.6 个点。这个数据集不用绝对位置词，只能靠外观和属性来区分。

重叠率要求提到 0.95 这么严，它比底座高 15.5 到 16.8 个点，说明框的位置也更精确。

指代分割，也就是直接画 mask，在表 3。8 个划分全是第一，平均 cIoU 81.4，最强的对比模型 Qwen3-VL-Seg 是 78.6。比专门的分割模型 SAM 3 高 7.0 到 12.9 个点。

MLLMSeg 同样从模型自己的视觉编码器解码 mask，Xiaomi-Lens-0 在 8 个划分上比它高 2.8 到 5.5 个点。

在 COCO 2017 上一次框出多个物体，按阈值平均的 F1 是 63.9，所有对比模型里最高。但作者说，优势来自严格阈值下框得更紧，0.5 阈值下它还排在两个模型后面。

推理分割在表 5。ReasonSeg 测试集上，gIoU 比最强的 InstructSAM 高 6.9 个点，cIoU 只高 1.5。机器人场景的 RoboRefIt 上，gIoU 高 4.5 和 7.1 个点。

接下来是这篇最想证明的一点，表 6。这些题只用文字回答，不输出框，也不输出 mask。8 个感知基准上，它全部超过底座，也超过所有对比的开源模型。

数数的 CountQA，从 43.8 提到 50.6。中文的 MME-RealWorld 高 7.2 个点，提升最小的是 HR-4K，只高 1.0 个点。

提升从哪来，看消融，表 8。作者训练了一个只用框的版本，数据完全一样，只关掉 mask 损失。只用框就比底座好不少，RefCOCO+ 验证集从 84.8 提到 89.7。

再加上 mask，6 项指标全都继续上涨，定位多 1.1 到 2.4 个点，感知多 0.2 到 1.4 个点。作者的解释是，框只能粗略表示形状，目标要靠外观和形状跟邻居区分时，mask 帮助最大。

表 9 看推理数据。只用定位数据就拿到了大部分提升，加上带 mask 的推理链，分割再涨 0.5 到 0.8 个点，主要是让最终的 mask 更精细。

### 和以前方法的对比

在自己做的 LensBench 上，对手换成了 GPT-5.6-Sol、Gemini-3.6-Flash、Qwen3.8-Max 这些大模型，同样的指令，直接输出 mask。表 7 里，它在航拍、驾驶、全景三类图像上 cIoU 第一，航拍领先 7.0 个点。

但在普通照片上，Qwen3.8-Max 领先，cIoU 74.2 对 63.9。隐式指令下，它的 cIoU 只有 54.7，排第五。

这张图是对比的例子。比如找咬过一口、露出奶油馅的那块点心，Xiaomi-Lens-0 选对了，SAM 3 却选了好几块别的点心。

我的判断是，这篇最有价值的，是一组干净的消融。同样的数据下，让 mask 损失直接作用在模型自己的视觉表示上，连纯文字回答的感知都变好了，不只是分割分数。

我的判断是，横向对比要打个折扣。对手的底座、规模和训练数据都不一样，它还用了自有的合成数据，架构和数据各占多少，横向表格分不开。能分开的只有消融，而 mask 带来的增量，定位上是 1 到 2 个多点，感知上不到 1.5 个点。

### 局限

论文没有单独的局限章节，但结果写得很清楚。第一，隐式指令仍然难，从显式到隐式，cIoU 掉了 16.7 个点。

第二，全景图里的多目标隐式指令最差，mask 重叠超过一半的比例，从 81.3 掉到 48.7，作者也把它列为还没解决的问题。

### 未来方向

论文没有专门写未来工作，只提出，要把目标找得全不全、mask 范围准不准，和框的定位放在一起考察。

我的判断是，这个思路可以往外推。GUI Agent 要点准一个小按钮，机器人要抓准一个物体，让模型自己的视觉表示学会区域范围，可能比外挂分割模块更划算。不过论文只在图像理解上做了验证。

我的判断是，下一步值得看的是，视觉隐变量能不能真正反馈进推理，让模型画出一块区域后，再回头去看它，而不只是画出来给人检查。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is the Xiaomi-Lens-0 technical report from Xiaomi, credited to the Xiaoai Plus and SeerRay teams.

It's a four-billion-parameter multimodal model built on Qwen3.5. The idea in one line: make the model draw masks, pixel-level outlines, from its own visual features, and use that signal to make it look more carefully.

This figure shows the six task types, from boxing or cutting out what a sentence describes, to referring in robot scenes, to questions about counts, colors and positions.

### Problem

Multimodal models answer questions about whole images well, but they get shaky on details. They miscount objects sitting side by side, or misread the attributes of a small region.

The authors blame the training signal. With text-only supervision, all the learning comes from the text answer. Nothing tells the model which part of the image supports that answer, or how far it extends.

A segmentation mask is the opposite. It labels, pixel by pixel, what belongs to the target, the most direct supervision you can give about a region's extent.

### Prior work

Plenty of multimodal models already train with masks for referring segmentation. But most get their spatial detail from an extra encoder. LISA and Sa2VA use a hierarchical segmentation encoder, and Qwen3-VL-Seg fuses features from a small supplementary image encoder. So the model's own visual representation never has to learn region detail.

Some methods do decode masks from the model's own features, PixelLM and MLLMSeg for example. But both freeze the vision encoder, so the mask loss never reaches the shared visual features.

And both lines of work treat masks purely as a means to segmentation. Whether mask supervision also improves fine-grained perception and visual reasoning, the authors say, has been left unexplored.

### Key idea

So the plan is no extra encoder. The model predicts masks straight from its own visual representations, and the mask loss flows back into the same network that generates language.

The second idea is called thinking with visual latents. Whenever the model mentions a region while reasoning, it emits a special token that can also be decoded into a mask. So the steps of a chain of thought get pixel-level supervision too, and you can see them.

### Method

Here's the architecture. The vision encoder's output takes two paths. One goes through the usual spatial merger into the language model. The other, the patch-level grid from before the merger, goes to a lightweight mask decoder.

Why before the merger? Citing an analysis from LongCat-Next, the authors note that the merger's aggressive downsampling throws away high-frequency detail, while the earlier grid keeps the spatial structure and is still part of the shared computation.

How does the decoder know which region to draw? A special mask token is added to the vocabulary. Each time the model emits it, its contextual state becomes a visual latent. That state depends on the image, the instruction and what's been said so far, so it stands for the region under discussion.

The decoder is small. The patch features pass through one Transformer layer and a projection. The visual latent is projected into the same space and scored against every patch, like a classifier built on the fly. The scores go back into the grid, and two transposed convolutions upsample them into a mask.

Training adds two losses: next-token prediction for text, and pixel-wise cross-entropy plus a Dice loss for masks. The key point is that the mask loss flows back into both the vision encoder and the language model.

Now, thinking with visual latents. Every time the reasoning names a region, a mask token goes in. One per instance when counting, one per object when comparing positions.

To be clear, at inference the masks are only an output branch. No crops or region features go back into the language model. They make the intermediate steps inspectable, and let training supervise those states.

In one of the paper's examples, the question is how many picture frames are in the image. The reasoning lists them one by one, each with a box and a mask, and lands on twenty-two.

Training has two stages. Stage one freezes the backbone and trains only the mask decoder. Stage two unfreezes everything and trains text and masks together.

For data, there are two synthesis pipelines. The first makes referring data: find objects, check descriptions, generate masks, write instructions in many styles, then re-ground each one to confirm it points to that target alone.

The second makes grounded reasoning data. It synthesizes questions about counting, attributes and spatial relations, keeps only the ones an independent checker answers correctly without seeing the reference, then has a reasoning model write traces with boxes, and pairs those boxes with masks.

They also built a benchmark, LensBench, covering general, aerial, driving and panoramic images, twelve hundred units and four thousand eight hundred instructions. Each unit crosses single and multiple targets with explicit instructions that name the category and implicit ones that skip the category and describe targets by function or purpose, and every unit is human-reviewed.

### Results

First, referring expression comprehension, boxing what a sentence describes, across eight RefCOCO-family splits in Table 1. It beats its Qwen3.5 backbone on all eight, by five point six points on average, and ranks first among all compared models on seven.

The biggest gain is RefCOCO+, seven point six points on average. That dataset has no absolute location words, so targets must be told apart by appearance and attributes.

At a strict overlap threshold of zero point nine five, it beats the backbone by fifteen point five to sixteen point eight points. The boxes are tighter, not just better aimed.

Referring segmentation, drawing the mask directly, is in Table 3. It's first on all eight splits, averaging eighty-one point four cIoU against seventy-eight point six for Qwen3-VL-Seg, the strongest baseline. Against SAM 3, a dedicated segmentation model, it's ahead by seven point zero to twelve point nine points.

MLLMSeg also decodes masks from the model's own vision encoder. Xiaomi-Lens-0 beats it by two point eight to five point five points across the eight splits.

On COCO 2017, boxing many objects at once, its threshold-averaged F1 of sixty-three point nine is the best compared. But the authors note the edge comes from tighter boxes at strict thresholds. At the loose zero point five threshold, two models still beat it.

Reasoning segmentation is in Table 5. On the ReasonSeg test set, its gIoU is six point nine above the strongest baseline, InstructSAM, though the cIoU lead is only one point five. On RoboRefIt, robot scenes, gIoU leads by four point five and seven point one.

Next is what this paper most wants to show, Table 6. These questions are answered in text only, no box, no mask. On all eight perception benchmarks, it beats its backbone and every other open-source model compared.

CountQA, a counting test, goes from forty-three point eight to fifty point six. The Chinese MME-RealWorld set gains seven point two, and the smallest gain is HR-4K, at one point zero.

Where does that come from? The ablations, Table 8. The authors trained a box-only version on exactly the same data, with only the mask loss switched off. Boxes alone already help, RefCOCO+ validation goes from eighty-four point eight to eighty-nine point seven.

Add masks back and all six metrics rise further, one point one to two point four on grounding and zero point two to one point four on perception. The authors' explanation: a box captures shape only roughly, so masks help most when a target has to be told from its neighbors by appearance and shape.

Table 9 looks at the reasoning data. Grounding data alone gets most of the gain. Adding mask-grounded reasoning traces lifts segmentation by another zero point five to zero point eight, mostly by sharpening the final masks.

### Comparison

On their own LensBench, the opponents are large models like GPT-5.6-Sol, Gemini-3.6-Flash and Qwen3.8-Max, all with the same instructions, all predicting masks directly. In Table 7, Xiaomi-Lens-0 has the best cIoU on aerial, driving and panoramic images, with a seven-point lead on aerial.

On general photos, though, Qwen3.8-Max leads, seventy-four point two cIoU to sixty-three point nine. And on implicit instructions, its cIoU is fifty-four point seven, only fifth.

This figure shows examples side by side. Asked for the pastry that's been bitten, showing its creamy filling, Xiaomi-Lens-0 gets it, while SAM 3 picks several other pastries.

My read is that the most valuable part is a clean ablation. With the same data, putting the mask loss on the model's own visual representation improves perception answered in plain text, not just segmentation scores.

My read is also that the cross-model tables deserve a discount. Opponents differ in backbone, size and training data, and this model adds in-house synthetic data, so the tables can't separate architecture from data. Only the ablation does, and there the mask adds one point one to two point four points on grounding, and at most one point four on perception.

### Limitations

There's no limitations section, but the results are clear. First, implicit instructions are still hard. Going from explicit to implicit costs sixteen point seven points of cIoU.

Second, implicit instructions with multiple targets in panoramic images are the worst case. The share of masks overlapping the ground truth by more than half falls from eighty-one point three to forty-eight point seven, and the authors call it an open challenge.

### Future work

There's no future-work section either. The authors only argue for judging whether all targets are found, and how accurate the mask extent is, alongside box localization.

My read is that the idea travels. A GUI agent clicking a tiny button, a robot grasping one specific object: teaching the model's own representation about region extent may beat bolting on a segmentation module. But the paper only tests image understanding.

My read on what to watch next: whether these visual latents can feed back into reasoning, so the model draws a region and then looks at it again, instead of drawing it only for a human to check.

That's today's Daily Frontier Papers. The paper is linked in the description.

