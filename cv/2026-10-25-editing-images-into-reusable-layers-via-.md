# Editing Images into Reusable Layers via Compositional Edit-Prior Distillation

**改完一张图图层也拆好了**

- 论文 Paper: [arXiv 2610.24738](https://arxiv.org/abs/2610.24738)
- 单位 Affiliation: 华南理工大学、清华大学、京东等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-25
- 视频 Videos: [YouTube 中文](https://youtu.be/QYZEW3liCOk) · [YouTube English](https://youtu.be/5ItBEnlzV6A) · [B站](https://www.bilibili.com/video/BV1NxH56ME4d)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇华南理工大学、京东等单位的论文，题目是，通过组合式编辑先验蒸馏，把图像编辑成可复用的图层。模型叫 LayerEditor。

一句话说，你让它改一张海报，它不只交回一张改好的图，还同时交回一组可以单独拿出来再改的透明图层。

### 问题

现在的指令式修图模型，比如你让它把海报里的城市照片从阿姆斯特丹换成威尼斯，它输出的是一张拍平的图，文字、物体和背景全揉在一起。

论文说，这对电商内容制作和广告设计这类场景很不方便。想再改一个细节，只能整张重新生成，既增加开销，又可能把不相关的地方也改掉，边缘还容易出瑕疵。

### 以前的方法

一个自然的办法是先编辑，再用图层分解模型把结果拆开。论文提到的 Agent Banana，就是把编辑和分解串成一套流程。

问题在于，这两步是分开优化的。编辑模型只管图像看起来对，不管图层怎么分，分解模型事后去拆，又不知道这次编辑要的是什么。结果图看着没问题，拆出来的图层却可能缺东西、分得乱，还得反复调用模型，再加手工修。

### 核心想法

作者的思路是，把编辑和出图层合成一件事。在同一次生成里，同时产出编辑后的整图，和一组有顺序的 RGBA 图层。RGBA 就是红绿蓝三个颜色通道，再加一个表示透明度的 Alpha 通道。

这里有个难点。直接把训练好的编辑模型改成输出图层，它原来的编辑能力会变差，比如文字写坏、结构变形，或者改到不该改的地方。所以第二个想法是，让原来的编辑模型当老师，在训练里一直带着学生，把编辑能力保住。

### 方法

先看整体结构，这张图是总览。输入是原图、编辑指令和想要的图层数 N，输出一张直接生成的编辑结果，作者叫它 Direct 画布，再加上 N 个从后往前叠放的 RGBA 图层。

底座是 FireRed-Image-Edit-1.1，一个多模态扩散 Transformer，用 rank 256 的 LoRA 来微调。扩散模型的意思是，从一团噪声出发，一步步去噪，最后得到图像。

第一个设计叫 In-Context RGB-Alpha Token Modeling。Direct 画布，加上每个图层的颜色和透明度，各自是一段 token，全部从噪声出发，放在同一个序列里一起去噪，彼此通过注意力互相参考。

这些内容在画面上的坐标是一样的，为了区分，作者给位置编码加了一个槽位坐标。原图是负 1，Direct 画布是 0，每一层的颜色和透明度依次往后排。这样图层数可以变，模型结构不用改。

第二个设计，是一个专门的 Alpha 解码器。它从原来的颜色解码器初始化，单独训练，用来还原细线条和半透明边缘这类连续变化的透明度。

第三个设计，就是标题里的 C-EPD，组合式编辑先验蒸馏。老师是冻结住的原始编辑模型。

训练时，先让学生自己跑一条去噪轨迹，在中途挑几个状态去问老师。这叫 on-policy 蒸馏，好处是老师指导的，正好是学生自己会走到的地方。

老师给两种监督。第一种，学生在 Direct 画布上的去噪方向，要和老师预测的方向一致。

第二种更关键。把学生此刻预测的各个图层解码出来，按透明度从后往前叠在白底上，合成一张图，再要求这张合成图在像素上接近老师预测的编辑结果。

这样，梯度能穿过解码和叠图这一步，直接传回图层分支，老师的编辑能力就同时传给了整图和图层。

训练流程是，先用普通的 flow matching 训练两个 epoch，之后每 5 次更新加一次 C-EPD，权重逐步升到 0.1。推理时跑 30 步，不再需要老师。

数据方面，作者做了 LayerEditor-100K，一共 10 万条编辑样本，来自 Crello、PrismLayersPro 和 ScaleEdit。其中文字编辑 48329 条，物体编辑 36858 条，海报整体风格编辑 14813 条，每条 2 到 10 层。

评测用两个自建的测试集，Crello-Testset 有 2994 条，ScaleEdit-Testset 有 1000 条，和训练数据严格不重叠。

### 实验和效果

对比对象有字节的 Seedream5-Pro，通过官方 API 调用，分先编辑再分解，和一次调用同时完成两种用法，还有 FireRed 编辑器加 Qwen-Image-Layered 分解的几种串联方案。

主要看的是合成图，也就是把输出的图层叠回去之后，和目标图有多像。因为图层叠回去之后，编辑结果应该依然正确。

结果在表 1。在 Crello 测试集上，和 Seedream5-Pro 一次调用的版本相比，合成图的 SSIM 提高了百分之 5.7，FID 降低了百分之 38.8。SSIM 越高越像，FID 越低越好。

在 ScaleEdit 测试集上，合成图 SSIM 从 0.8365 提到 0.8839，衡量透明度准不准的 Alpha IoU，从 0.6583 提到 0.6802。

还有一组数字很能说明问题。Seedream5-Pro 先编辑再分解，直接出的图 SSIM 是 0.8933，拆成图层再叠回去，掉到 0.7901，降了百分之 11.6。LayerEditor 从 0.8925 到 0.8839，只降了百分之 1。也就是说，它的图层叠回去，基本还是那张图。

作者还用 Gemini-3.1-Pro 按五个维度打 1 到 10 分。在 Crello 测试集上，LayerEditor 五项平均 6.00，对手里最高的是 Seedream5-Pro 一次调用版，4.50。

控制能力方面，表 2 把同一张编辑结果，按不同粒度重新分成 2 到 10 层。LayerEditor 的 Alpha IoU 比 Seedream5-Pro 一次调用版高 0.0493，语义分组得分高 0.97 分。

这张图是指定图层内容的例子。比如把海报里的 MUCH 换成 VALUE，同时要求一层背景加五层文字，或者把两只鸟改成鲜红色，并且把鸟单独分层。

还有一个附带的能力。只给原图和一句保持原样的指令，它就能把图拆成图层，没有为这个任务专门微调。在 1171 张测试图上，合成图 SSIM 是 0.8761，Qwen-Image-Layered 是 0.7963。

### 和以前方法的对比

消融实验在表 3。去掉 C-EPD，编辑一致性从 6.14 掉到 4.67，图层分割从 6.83 掉到 5.01。

有意思的是，两种老师监督只留一种，编辑一致性会好一些，但画质和图层分割反而更差，两种一起用，四项指标才全部提升。

专门的 Alpha 解码器也有用。换回原来的颜色解码器、把三个通道取平均，RGBA 误差从 0.0350 涨到 0.0578。

也要看到短板。只看直接生成的那张图，Seedream5-Pro 先编辑再分解的版本，SSIM 在两个测试集上都略高于 LayerEditor。LayerEditor 赢的是图层这一块。

另外，作者自己说明，串联方案里的 FireRed 编辑器没有在他们的数据上微调，而 LayerEditor 微调过，所以这是系统层面的对比，没法把联合建模的好处，和训练数据的好处分开。

我的判断是，这篇真正的贡献，是把叠回去之后还对不对，直接变成了训练目标。以前分解是事后补救，这里老师直接监督叠出来的那张图，所以图层和编辑结果从训练开始就是绑在一起的。

### 局限

作者列的局限不少。最多只试了 10 层，更多层没有测。细线条、复杂透明和半透明边缘，在移动图层或重新合成时，还会出瑕疵。

文字仍然是像素图，不是能改字体的矢量文字，做排版还是受限。

评测方面，没有系统测试图层被挪动之后的效果，Gemini 打分也没有和人工评分对照过。标注和打分靠自动流程，可能有错和偏差。编辑能力有没有保住，也只在自建测试集上看过，没用独立的通用编辑基准。

### 未来方向

作者在伦理声明里提醒，图层让后期修改更方便，也可能被用来做误导性的改图，部署时要考虑内容溯源和版权。

我的判断是，对电商和广告这种要反复改稿的场景，一次给出能直接复用的图层，比再提升一点画质更实用。接下来最值得看的，是图层被挪动、替换之后，画面还能不能保持干净。

我的判断是，也要看到，所有分数都来自作者自建的测试集，其中一部分还是模型打分，论文也没有提到代码和数据会不会公开，实际效果还要等更多独立验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from South China University of Technology, JD.com and collaborators. It's called Editing Images into Reusable Layers via Compositional Edit-Prior Distillation, and the model is LayerEditor.

The pitch in one line: ask it to edit a poster, and you don't just get back an edited picture. You also get a stack of transparent layers you can pull out and edit on their own.

### Problem

Today's instruction-based editors work like this. You say, swap the city photo on this poster from Amsterdam to Venice, and you get back one flattened image, with text, objects and background all fused together.

The paper points out that this is painful for things like e-commerce content and ad design. Want to tweak one more detail? You regenerate the whole image, which costs time, risks changing things you didn't touch, and tends to leave artifacts at the edges.

### Prior work

The obvious workaround is to edit first, then run a layer-decomposition model to split the result apart. Agent Banana, which the paper cites, chains editing and decomposition into one workflow.

The catch is that the two steps are optimized separately. The editor only cares about the picture looking right, not how it should be layered. The decomposer comes in afterwards without knowing what the edit was for. So an edit can look fine and still produce layers that are incomplete or badly organized, which means more model calls and more manual cleanup.

### Key idea

The authors' answer is to make editing and layering one job. In a single generation, the model produces the edited image and an ordered stack of RGBA layers. RGBA just means red, green and blue color channels, plus an alpha channel for transparency.

There's a snag, though. If you simply retrain a good editing model to output layers, its editing gets worse: garbled text, distorted structure, changes to regions it should have left alone. So the second idea is to keep the original editor around as a teacher during training, to protect that editing skill.

### Method

Here's the overview figure. The inputs are the source image, the instruction, and a requested number of layers, N. The outputs are a directly generated edited image, which the authors call the Direct canvas, plus N RGBA layers stacked from back to front.

The starting point is FireRed-Image-Edit-1.1, a multimodal diffusion transformer, adapted with rank two fifty-six LoRA. A diffusion model, briefly, starts from noise and removes it step by step until an image appears.

The first piece is called In-Context RGB-Alpha Token Modeling. The Direct canvas, and the color and the transparency of every layer, each become a block of tokens. All of them start as noise, sit in one sequence, and are denoised together, attending to each other along the way.

Since all of these share the same pixel coordinates, the authors add a slot coordinate to the position encoding. The source image is minus one, the Direct canvas is zero, and each layer's color and transparency fields follow in order. That lets the number of layers vary without changing the architecture.

The second piece is a dedicated alpha decoder. It's initialized from the original color decoder and trained separately, to recover smooth transparency for thin strokes and semi-transparent edges.

The third piece is the C-EPD of the title, compositional edit-prior distillation. The teacher is the original editing model, frozen.

During training, the student first runs its own denoising trajectory, and the teacher is queried at a few states along the way. That's on-policy distillation, and the benefit is that the teacher gives guidance exactly where the student actually goes.

The teacher supervises two things. First, the student's denoising direction for the Direct canvas should match the teacher's prediction.

The second is the important one. Take the layers the student is predicting at that moment, decode them, stack them back to front on a white background using their alpha, and require that composite to match the teacher's predicted edit, pixel by pixel.

Gradients flow back through the decoding and the stacking into the layer branch. So the teacher's editing skill reaches both the full image and the layers.

The schedule: two epochs of plain flow matching first, then C-EPD added every fifth update, with its weight ramping up to zero point one. Inference takes thirty steps, with no teacher involved.

On data, the authors built LayerEditor-100K, one hundred thousand editing examples drawn from Crello, PrismLayersPro and ScaleEdit. That's forty-eight thousand three hundred twenty-nine text edits, thirty-six thousand eight hundred fifty-eight object edits, and fourteen thousand eight hundred thirteen poster-level style edits, each with two to ten layers.

Evaluation uses two benchmarks they built, Crello-Testset with two thousand nine hundred ninety-four cases and ScaleEdit-Testset with one thousand, kept strictly separate from the training data.

### Results

The competitors include ByteDance's Seedream5-Pro through its official API, used two ways, edit then decompose, or both in one call, plus several cascades of the FireRed editor with Qwen-Image-Layered decomposition.

The main yardstick is the composite: stack the output layers back together and see how close that is to the target. The requested edit should still be correct once the layers are recombined.

The numbers are in Table 1. On Crello-Testset, against the one-call version of Seedream5-Pro, composite SSIM improves by five point seven percent and composite FID drops by thirty-eight point eight percent. Higher SSIM means more similar; lower FID is better.

On ScaleEdit-Testset, composite SSIM goes from zero point eight three six five to zero point eight eight three nine, and alpha IoU, which measures how accurate the transparency is, from zero point six five eight three to zero point six eight zero two.

One comparison is especially telling. Seedream5-Pro's edit-then-decompose pipeline scores zero point eight nine three three SSIM on its direct output, but once split into layers and restacked, that falls to zero point seven nine zero one, an eleven point six percent drop. LayerEditor goes from zero point eight nine two five to zero point eight eight three nine, a one percent drop. In other words, its layers add back up to essentially the same picture.

The authors also had Gemini-3.1-Pro score outputs from one to ten on five dimensions. On Crello-Testset, LayerEditor averages six point zero zero, against four point five zero for the best baseline, the one-call Seedream5-Pro.

For control, Table 2 takes the same edited result and regroups it into anywhere from two to ten layers. LayerEditor beats one-call Seedream5-Pro by zero point zero four nine three in alpha IoU and by zero point nine seven points in semantic grouping.

This figure shows layer content being specified in the prompt. One example replaces MUCH with VALUE on a poster and asks for one background layer plus five text layers. The other turns two birds vivid red and asks for the birds on their own layers.

There's also a bonus capability. Give it only the source image and an instruction to keep it unchanged, and it splits the image into layers, with no fine-tuning for that task. On one thousand one hundred seventy-one test images, composite SSIM is zero point eight seven six one, versus zero point seven nine six three for Qwen-Image-Layered.

### Comparison

The ablations are in Table 3. Remove C-EPD, and edit consistency falls from six point one four to four point six seven, and layer segmentation from six point eight three to five point zero one.

Interestingly, keeping only one of the two teacher signals helps edit consistency a bit, but makes visual quality and layer segmentation worse. Only with both together do all four metrics improve.

The dedicated alpha decoder matters too. Swap back to the original color decoder and average its three channels, and RGBA error rises from zero point zero three five zero to zero point zero five seven eight.

Now the weak spot. On the directly generated image alone, Seedream5-Pro's edit-then-decompose version has slightly higher SSIM than LayerEditor on both benchmarks. LayerEditor's win is in the layers.

The authors also flag that the FireRed editor in the cascades wasn't fine-tuned on their data, while LayerEditor was. So these are system-level comparisons, and they can't separate the benefit of joint modeling from the benefit of the training data.

My read is that the real contribution here is turning "does it still look right when you stack the layers back up" into a training objective. Decomposition used to be cleanup after the fact. Here the teacher supervises the restacked image directly, so the layers and the edit are tied together from the start of training.

### Limitations

The authors list quite a few limitations. They only go up to ten layers; larger stacks are untested. Thin strokes, complex transparency and semi-transparent edges can still produce artifacts when layers are moved or recombined.

Text is still rasterized pixels, not editable fonts or vector shapes, which limits typography work downstream.

On evaluation, they don't systematically test what happens after layers are moved around, and the Gemini scores aren't validated against human ratings. Annotations and scores come from automated pipelines and may contain errors and biases. And whether the editing skill was preserved is only checked on their own benchmarks, not on an independent general editing test.

### Future work

In their ethics statement, the authors note that layers make later edits easier, which could also be used for deceptive edits, so deployment should consider content provenance and copyright.

My read is that for e-commerce and advertising, where designs get revised over and over, getting reusable layers in one pass matters more than a small bump in image quality. The thing to watch next is whether the layers stay clean after they're moved or replaced.

My read is also that every score here comes from benchmarks the authors built, some of it model-graded, and the paper doesn't say whether code or data will be released. Real-world performance needs more independent checking.

That's today's Daily Frontier Papers. The paper is linked in the description.

