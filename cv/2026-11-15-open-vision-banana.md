# Open Vision Banana: A Fully Open Recipe for Generative Visual Perception

**会画图的模型也能看懂图**

- 论文 Paper: [arXiv 2611.15844](https://arxiv.org/abs/2611.15844)
- 单位 Affiliation: 纽伦堡工业大学 等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-15
- 视频 Videos: [YouTube 中文](https://youtu.be/XrisON_3SSs) · [YouTube English](https://youtu.be/lsm5hpLNMQU) · [B站](https://www.bilibili.com/video/BV1uQsJ6iEmR)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇纽伦堡工业大学等机构的论文，题目叫 Open Vision Banana，一个完全公开的生成式视觉感知配方。

它想回答的问题是，一个画图的模型，能不能直接拿来做看图的工作，比如分割、深度估计，而且数据、代码、权重全部公开。

### 问题

先说背景。最近有一种看法，图像生成模型为了画得像，其实已经学到了不少对真实世界的理解。

此前有一项工作叫 Vision Banana，做法是把视觉任务改写成图像编辑，让模型把预测结果直接画成一张图片。

论文说，它在分割和几何任务上能和专门模型比，还基本保留了原来的编辑能力。但它的底座模型、数据和代码都不公开，外人没法知道是哪些设计在起作用。

### 以前的方法

用生成模型做感知，以前大概有几条路。一条是只抽取生成模型内部的特征，再接一个专门的感知头去训练，预测并不是生成模型自己画出来的。

另一条是像 Marigold、Lotus 这样，利用生成模型的先验，专门做深度和法线这类几何预测，但它们不打算保留通用的生成能力。

还有 DICEPTION、SenseNova-Vision 这类通用模型，也想把生成和感知统一起来。这篇论文说，自己的不同在于每个组件都公开，而且把关键设计一个个做了对照实验。

### 核心想法

核心做法很简单。不加任何任务专用的输出头，给模型一张图和一句指令，它生成一张 RGB 图片，再用一个固定的、按任务写好的规则，把这张图读回成预测结果。

所以难点不在网络结构，而在怎么把任务画成图，用什么颜色，提示词怎么写，调哪些参数，数据怎么配。论文发现，这些底层选择影响很大，还会互相牵制，单独有用的选择，合在一起可能反而不好。

### 方法

底座用的是 FLUX.2 Klein，一个开放权重、能以输入图片为条件的生成模型，有 4B 和 9B 两个尺寸。训练时整个 transformer 都更新，VAE 和文本编码器冻结。

分割怎么画。提示词里列出要找的类别，每类配一个随机颜色，可以写成 RGB 数值、十六进制，或者颜色名字，黑色留给背景。模型把每一类涂成对应的颜色，读回的时候，每个像素归到最接近的那个颜色。

训练时，百分之七十的样本只列出图里的一部分类别，百分之十五列出全部，还有百分之十五会故意加上图里没有的类别，教模型不要凭空画出不存在的东西。

实例分割一次只问一个类别，提示词给出 32 种颜色，按从左到右的顺序，给每个实例涂一种颜色。深度编码成相对的逆深度，越近越亮。法线则是把三个方向分量，分别对应到 RGB 三个通道。

数据只用公开数据集。感知数据之外，还混入了公开的文生图、图像编辑和原图重建数据，用来保住生成能力。零样本版本用了 210 万对训练样本，非零样本版本用了 380 万对。

训练目标就是底座原来的 rectified flow 目标，因为输出都是图片，不需要改损失。非零样本版本里，几何标注没有定义的区域，用一个有效区域掩码排除在损失之外。

完整模型训练 10 万步。非零样本版本，4B 在 16 张 GH200 上跑了大约 73 小时，9B 在 32 张上跑了大约 79 小时。

这里说的零样本，是指评测用的数据集没有出现在训练里，但任务本身训练过。非零样本版本，则额外用了部分评测数据集的训练集。

推理时还有一个技巧，叫 legend ensembling，图例集成。同一张图跑很多遍，每遍换一套类别和颜色的对应关系，随机种子固定，最后每个像素做多数投票。这一步不需要重新训练。

### 实验和效果

先看语义分割。在城市街景数据集 Cityscapes 上，零样本的 9B 模型加上图例集成，mIoU 达到 73.9。论文报告，Vision Banana 是 69.9，最强的零样本专门模型 SAM 3 是 65.2。

要注意，73.9 是集成之后的分数，单次推理的平均是 65.6。用上 Cityscapes 训练数据的非零样本 4B 模型，拿到 OVB 最好的结果，76.4。

在 80 个类别的 COCO-Object 上，零样本 9B 集成后是 53.0，超过表里所有专门方法，包括 CorrCLIP 的 49.4。但单次平均只有 38.7，和 DiffSegmenter、iSeg 差不多。

再看推理分割，指令是一句需要想一想的话，比如钢琴家应该坐的地方。零样本 4B 模型配上 Gemma4 改写查询，gIoU 是 67.9，超过 SegZero 和 RSVP-GPT-4o，但低于用 Gemini 2.5 Pro 做推理的 SAM 3 的 77.0，和 Vision Banana 的 79.3。

这里有个有意思的对照。没微调的底座，加上查询改写，只有 24.1，微调之后，改写能多带来 26.7 分。把 Gemma4 换成 Gemini 3.1 Pro，非零样本 4B 还是 74.6。作者的结论是，用哪个推理模型，远没有生成底座本身的感知能力重要。

差距比较大的是指代分割和实例分割。RefCOCOg 上，零样本 9B 是 57.4，Vision Banana 是 73.8。实例分割 SA-Co/Gold 上，零样本 9B 加一个 Qwen3-VL 判断类别是否存在，集成后是 37.4，非零样本的 SAM 3 是 54.1，Vision Banana 是 47.5。

几何任务上，NYUv2 深度的 δ1 指标，零样本 9B 是 97.4，Marigold V2 是 98.0。论文提醒，Vision Banana 预测的是绝对深度，OVB 是逐图对齐后的相对深度，两者不能直接比。

生成能力保留得怎么样。图像编辑基本守住了，4B 在 ImgEdit 基础项上还从 3.66 提到 3.78。但文生图的分数全部下降，零样本配比下，GenAI-Bench 的 VQAScore，4B 降了大约 0.05，9B 降了大约 0.03。

再看图例集成。Cityscapes 上，9B 从第一遍的 65.4 起步，大约 10 到 15 遍后稳定在 74 左右。论文报告，最多能提 8.7 个点，在 COCO-Object 上最多提 15.0 个点。

为什么有用。作者做了一个诊断，把每块预测区域按真实标注重新贴上类别，分数一下到了 85.9。也就是说，区域本身分得很准，主要错在颜色和类别绑错了。集成之后，纠正的主要就是这种绑定错误。

投票还有个副产品。各遍之间的一致程度，可以当作每个像素的置信度。Cityscapes 上一致度低于 0.9 的像素只占百分之六，却贡献了百分之五十八的错误。

另外，微调之后，模型的冻结特征也变好了。9B 在无监督物体发现上是 56.3，超过 DINOv3 7B 的 54.8。

### 和以前方法的对比

论文还有很大一部分是对照实验，我挑几个最有信息量的。第一个是颜色怎么分配。如果训练时每类固定一种颜色，比如车永远是蓝色，按同样的颜色评测有 53.9，但评测时颜色一换，就掉到 7.1，说明模型是死记硬背。

训练时每个样本随机换颜色，模型就能跟着提示词走，换了颜色也有 51.0。调色板也有讲究，从 200 种颜色里只用区分度最高的 64 种，六项设置的平均最好。

第二个是调哪些参数。只调最后几层，除了编辑以外各项都落后，所以最终方案更新整个 transformer。附录里还试了 LoRA，深度、法线和编辑能追平全量微调，但语义分割差了大约 12 个 mIoU。

第三个是数据配比。没有一个设置在所有任务上都最好，趋势也不单调。比如去掉文生图数据，法线误差最低，但深度变差。最后定为百分之十五的生成数据，百分之二十五的编辑和重建数据。

我的判断是，这篇的价值不在刷榜，而在把一个闭源系统的配方拆开，一项一项告诉你，哪些地方真的影响结果。对想复现这类方法的团队，这些对照实验比最终分数更有用。

我的判断是，领先的那几个分割数字要打个折看，它们依赖图例集成，Cityscapes 用了 20 遍，COCO-Object 用了 90 遍，相当于同一张图要生成几十次，单次推理的分数要低不少。

### 局限

论文自己说，主要局限是开源底座的感知能力，还比不上 Nano Banana Pro 这样的闭源生成器。

从结果看，非零样本版本在指代分割和推理分割上，离 Vision Banana 不到五个点，但实例分割还差得更多。

另外，文生图质量有可以测到的下降，9B 非零样本版本在 UGE 编辑测试上，也从 4.03 降到 3.68。作者也承认，几何数据怎么分配的结论来自缩小规模的实验，完整规模上哪种分配最好，还没有定论。

成本也不低，完整微调 4B 模型，需要 16 张 GH200 跑大约三天。

### 未来方向

论文公开了数据处理流程、训练和评测代码，以及最终、中间和消融实验的模型权重，作者希望以此推动后续研究。

作者还估计，LoRA 版本放到单个 4 卡节点上，靠梯度累积也能跑，只是更慢。

我的判断是，接下来值得看两件事。一是更强的开源图像生成底座出来以后，这套配方能不能把差距补上，论文已经表明，底座的感知能力是主要瓶颈。二是多画几遍再投票的思路，能不能用更少的遍数达到同样效果。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by the University of Technology Nuremberg, with partners including the University of Tübingen, TUM and valeo.ai. It's called Open Vision Banana, a fully open recipe for generative visual perception.

The question is simple. Can a model built to draw pictures be turned into a model that reads them, doing segmentation and depth estimation, with the data, code and weights all public?

### Problem

Some background first. There's a growing view that image generators, in learning to draw convincingly, pick up a lot of real-world understanding along the way.

An earlier system called Vision Banana put that to work by recasting vision tasks as image editing. The model draws its prediction directly as an image.

According to this paper, Vision Banana rivals specialist models on segmentation and geometry while mostly keeping its editing skills. But its base model, data and code are proprietary, so nobody outside can tell which design choices actually produce those results.

### Prior work

There have been a few routes to perception with generators. One extracts the generator's internal features and trains a separate perception head on top, so the prediction isn't something the generator draws itself.

Another, like Marigold and Lotus, uses generative priors specifically for geometry such as depth and surface normals, without trying to keep general image generation.

And generalists like DICEPTION and SenseNova-Vision also try to unify generation and perception. The authors say what sets their work apart is that every component is public, and that they test the key design choices one at a time.

### Key idea

The core move is plain. No task-specific output heads. You give the model an image and an instruction, it generates an RGB image, and a fixed, task-specific rule reads that image back into a prediction.

So the hard part isn't architecture. It's how to draw each task as a picture: which colors, how to word the prompt, which parameters to tune, how to mix the data. The paper finds these low-level choices matter a lot, and they interact. Choices that help on their own can hurt when combined.

### Method

The base is FLUX.2 Klein, an open-weight generator that conditions on an input image, at four billion and nine billion parameters. The whole transformer is trained, while the VAE and the text encoder stay frozen.

Here's how segmentation is drawn. The prompt lists the classes to find, each with a randomly assigned color, written as RGB values, hex codes or color names, with black reserved for background. The model paints each class in its color, and at readout every pixel goes to the nearest prompted color.

In training, seventy percent of examples list only some of the classes present, fifteen percent list all of them, and fifteen percent add classes that aren't in the image, to teach the model not to paint things that aren't there.

Instance segmentation asks about one category at a time and gives thirty-two colors, painting instances in left-to-right order. Depth is encoded as relative inverse depth, so nearer is brighter. Surface normals map each vector component to one of the three color channels.

The data is public only. Alongside perception data, they mix in public text-to-image, image editing and identity reconstruction data to preserve generation. The zero-shot models use two point one million training pairs, the non-zero-shot models three point eight million.

The training objective is just the base model's rectified-flow loss, since every output is an image. For the non-zero-shot models, undefined regions in the geometry labels are masked out of the loss.

The full models train for one hundred thousand steps. The non-zero-shot runs took about seventy-three hours on sixteen GH200 GPUs for the four billion model, and about seventy-nine hours on thirty-two for the nine billion.

A note on terms. Zero-shot here means the evaluation datasets were never in training, though the tasks themselves were. The non-zero-shot models also train on the training splits of some evaluation datasets.

There's also an inference trick called legend ensembling. Run the same image many times, change the class-to-color legend on every pass, keep the random seed fixed, and take a per-pixel majority vote. No retraining needed.

### Results

Start with semantic segmentation. On the Cityscapes street-scene benchmark, the zero-shot nine billion model with legend ensembling reaches seventy-three point nine mIoU. The paper reports Vision Banana at sixty-nine point nine, and the strongest zero-shot specialist, SAM 3, at sixty-five point two.

Keep in mind that seventy-three point nine is after ensembling. The mean single pass is sixty-five point six. With Cityscapes training data, the non-zero-shot four billion model gets the best OVB result, seventy-six point four.

On COCO-Object, with eighty classes, the ensembled zero-shot nine billion model scores fifty-three point zero, ahead of every specialist in the table, including CorrCLIP at forty-nine point four. But its mean single pass is thirty-eight point seven, on par with DiffSegmenter and iSeg.

Next, reasoning segmentation, where the instruction takes some thought, like "the place where piano players should sit". The zero-shot four billion model, with Gemma 4 rewriting the query, scores sixty-seven point nine gIoU. That beats SegZero and RSVP with GPT-4o, but trails SAM 3 at seventy-seven point zero and Vision Banana at seventy-nine point three, both of which use Gemini 2.5 Pro as their reasoner.

There's a telling comparison here. The untuned base model, even with query rewriting, gets only twenty-four point one. After fine-tuning, rewriting adds twenty-six point seven points. Swap Gemma 4 for Gemini 3.1 Pro, and the non-zero-shot four billion model stays at seventy-four point six. The authors conclude the choice of reasoner matters far less than the base generator's perception.

The bigger gaps are in referring and instance segmentation. On RefCOCOg, the zero-shot nine billion model scores fifty-seven point four, against Vision Banana's seventy-three point eight. On SA-Co/Gold instance segmentation, the zero-shot nine billion model, with a Qwen3-VL gate that checks whether the category is present, reaches thirty-seven point four after ensembling, while the non-zero-shot SAM 3 scores fifty-four point one and Vision Banana forty-seven point five.

On geometry, NYUv2 depth accuracy, the delta-one metric, is ninety-seven point four for the zero-shot nine billion model, against ninety-eight point zero for Marigold V2. The paper warns that Vision Banana predicts metric depth while OVB is aligned relative depth per image, so those two aren't directly comparable.

How much of the generator survives? Editing mostly holds up, and the four billion model's basic ImgEdit score even rises from three point six six to three point seven eight. But text-to-image scores drop for every model. With the zero-shot mix, GenAI-Bench VQAScore falls by about zero point zero five for four billion and about zero point zero three for nine billion.

Now legend ensembling. On Cityscapes, the nine billion model starts at sixty-five point four on the first pass and levels off around seventy-four after ten to fifteen passes. The paper reports gains of up to eight point seven points there, and up to fifteen points on COCO-Object.

Why does it work? The authors ran a diagnostic: relabel each predicted region with its true majority class, and the score jumps to eighty-five point nine. So the regions themselves are mostly right. The errors come from colors bound to the wrong class, and that's mainly what ensembling fixes.

The vote has a side benefit. Agreement across passes works as a per-pixel confidence. On Cityscapes, the six percent of pixels with agreement below zero point nine account for fifty-eight percent of all errors.

And fine-tuning improves the frozen features too. The nine billion model scores fifty-six point three on unsupervised object discovery, above DINOv3 7B at fifty-four point eight.

### Comparison

A large part of the paper is controlled experiments, and I'll pick the most informative. First, color assignment. Train with a fixed color per class, say cars always blue, and you get fifty-three point nine when evaluated the same way. Change the colors at test time and it collapses to seven point one. The model memorized the mapping.

Randomize colors per training example and the model learns to follow the prompt, scoring fifty-one point zero with unseen assignments. The palette matters too. Using only the sixty-four most distinct of two hundred colors gave the best average over six settings.

Second, which parameters to tune. Tuning only the last few blocks falls behind on everything except editing, so the final models update the full transformer. In the appendix, LoRA matches full fine-tuning on depth, normals and editing, but trails by about twelve mIoU on semantic segmentation.

Third, the data mix. No setting wins on every task, and the trends aren't monotonic. Dropping text-to-image data gives the lowest normal error, but worse depth. They settle on fifteen percent generation data and twenty-five percent editing and reconstruction data.

My read is that the value here isn't leaderboard numbers. It's taking a closed system's recipe apart and showing, piece by piece, what actually moves the results. For a team trying to build something like this, those controlled experiments are worth more than the final scores.

My read is also that the headline segmentation numbers deserve a discount. They rely on legend ensembling, with twenty passes on Cityscapes and ninety on COCO-Object, which means generating the same image dozens of times, and the single-pass scores are noticeably lower.

### Limitations

The authors name the main limitation themselves: open base models still perceive less well than proprietary generators like Nano Banana Pro.

The results bear that out. The non-zero-shot models come within five points of Vision Banana on referring and reasoning segmentation, but fall further behind on instance segmentation.

Text-to-image quality also drops measurably, and the nine billion non-zero-shot model's UGE editing score falls from four point zero three to three point six eight. The authors also note that their conclusions on how to split the geometry data come from reduced-scale runs, and don't establish the best split at full scale.

And it isn't cheap. Fully fine-tuning the four billion model takes sixteen GH200 GPUs for about three days.

### Future work

The paper releases the data pipeline, the training and evaluation code, and the final, intermediate and ablation checkpoints, to support further research.

The authors also estimate that the LoRA version should fit, more slowly, on a single four-GPU node using gradient accumulation.

My read is that two things are worth watching. First, whether stronger open image generators close the gap with this same recipe, since the paper points to base-model perception as the main bottleneck. Second, whether the draw-it-many-times-and-vote idea can get the same gains with far fewer passes.

That's today's Daily Frontier Papers. The paper is linked in the description.

