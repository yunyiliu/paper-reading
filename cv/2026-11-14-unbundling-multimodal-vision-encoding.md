# Unbundling Multimodal Vision Encoding

**视觉编码器该不该保留**

- 论文 Paper: [arXiv 2611.17334](https://arxiv.org/abs/2611.17334)
- 单位 Affiliation: Meta FAIR · 纽约大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-14
- 视频 Videos: [YouTube 中文](https://youtu.be/3Jn48nSZ_Tc) · [YouTube English](https://youtu.be/ERMMyCTrBrI) · [B站](https://www.bilibili.com/video/BV1owsG6dELu)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 Meta FAIR 和纽约大学的论文，名字叫 Unbundling Multimodal Vision Encoding，意思是，把多模态模型里的视觉编码器拆开来看。

它问的是，多模态大模型该怎么看图，视觉编码器还要不要，要的话，到底是哪一部分在起作用。

### 问题

先说背景。现在大多数多模态模型，是先拿一个单独预训练好的视觉编码器，比如做图文对齐的那一类，把图片变成一串向量，再交给语言模型处理。

但最近的做法开始分化。有的模型，比如 Kimi K3，从零开始把视觉编码器和语言模型一起训练。还有的干脆不要编码器，叫 encoder-free，把图片切成小块，直接送进语言模型。

问题在于，这几条路线谁好谁坏，从发布的模型上看不出来，因为它们的数据、架构、算力全都不一样。

### 以前的方法

以前也有对照实验，但大多是在比较哪种预训练编码器更好，而不是问编码器到底需不需要预训练。

比较有编码器和没编码器的研究，又常常一下子改了好几个变量，而且只算多模态训练的算力，把编码器预训练当成免费的。论文举的例子是，SigLIP 2 预训练用了 400 亿个图文样本。

还有一篇 2025 年 Shukor 等人的工作，两种设计都从零训练，结论是 encoder-free 能追平，甚至略好于有编码器的模型。这篇论文的结论正好相反，后面会讲原因。

### 核心想法

作者的核心想法是，视觉编码器其实捆了三个设计选择。第一，上游预训练带来的视觉先验。第二，视觉专用的容量，也就是只有图像 token 才经过的参数。第三，这部分容量放在融合之前，图像和文字在主干网络里相遇之前，就先处理完。

这样一拆，几条路线就清楚了。预训练编码器三样都有，从零训练的编码器去掉了先验，encoder-free 三样全没有。作者还加了第四种，叫 in-fusion，只保留专用容量，但把它放进主干网络内部。

然后一次只拆一个，回答三个问题。编码器需要单独预训练吗，视觉需要专用容量吗，这部分容量必须放在融合之前吗。

### 方法

为了公平，所有模型都从零训练，连语言主干也是。文本统一来自 Nemotron-CC，图文对来自 MetaCLIP，一种编码器，一种主干，一套优化设置。

主干仿照 Qwen3 的结构，主要实验用 1.7B 和 4B 两个尺寸，视觉专用参数固定在 0.4B，也就是 4 亿。编码器是一个 27 层的 ViT。

最关键的设定叫 lifetime compute，全生命周期算力。就是把所有训练阶段的算力加在一起，包括编码器预训练和语言预训练，只是不算最后的指令微调。各种方案在同样的总算力下比较。

评测分两类。视觉理解用 Cambrian 的 16 个视觉问答 benchmark，分四类取平均，语言能力用 12 个文本 benchmark。测视觉问答之前，每个检查点先做三轮指令微调。

in-fusion 怎么做，看这张图。在主干的某些层里，注意力的投影矩阵和前馈网络 FFN 保留共享权重，再加一份只给图像 token 用的权重，输出直接加到共享的输出上。

这些新权重初始化时输出为零，训练从共享模型出发。它们只对图像 token 起作用，纯文本输入的计算和原来完全一样。

### 实验和效果

第一个问题，编码器要不要单独预训练。在 1.7B 主干、固定总算力下，作者试了四种预训练目标。预训练只占预算的 7.4% 或 15% 时，四种都和从零训练的编码器差不多。占到 60% 时，视觉和文本两项平均分都明显更低。

更有意思的是，就算预训练算力免费，也没用。MetaCLIP 编码器的预训练算力加到原来的 6 倍多，ImageNet 线性探测准确率从 80.4% 升到 84.4%，但视觉问答平均分，反而是预训练最短的最高，从零训练的编码器也能追平。识别变强，不等于视觉问答变强。

第二个问题，视觉要不要专用容量。看这张图，encoder-free 在视觉问答上明显落后于从零训练的编码器，1.7B 和 4B、各个预算都是这样，文本能力则差不多。按图上的箭头，两个尺寸上，encoder-free 分别要用编码器 4.4 倍和 3.0 倍的算力才能追上。

有人会说，encoder-free 少了 4 亿参数，当然吃亏。作者就把这 4 亿参数加回主干，加宽、加深都试了。表 2 里，从零编码器视觉问答 44.4，encoder-free 39.0，加了参数的版本在 38.6 到 38.8，没追上。差距最大的是 OCR 和图表。

在 224 分辨率下，换更复杂的 patch embedding，也就是图片小块进模型的那一层，也不行，两层投影是 40.5，分层 MLP 是 35.9。同样的参数，做成视觉专用，比做成共享的更有用。

第三个问题，专用容量必须放在融合之前吗。作者把这 4 亿参数挪进主干，做成视觉专用的前馈分支，试了几种放法。表 3 里，每层放一点是 39.4，均匀分在 8 层是 37.6，放在最后 8 层是 35.1，都没比 encoder-free 好。只有放在最前面 8 层，是 45.0，和从零编码器的 44.4 相当。

还有一组对照，同样的参数放在前 8 层，但让所有 token 都用。加宽共享 FFN 是 40.1，单独一个分支、所有 token 共用，是 42.3，都不如视觉专用的 45.0。关键在专用两个字。

训练更久，结论也一样。1.7B 主干，从零编码器 48.9，in-fusion 49.2，encoder-free 44.8。4B 主干，编码器 54.5，in-fusion 54.3，encoder-free 51.7。

作者又做了从 0.6B 到 6.4B 的五档尺寸，拟合规模定律。每个尺寸上，encoder-free 都要大约 3 倍的算力，才能在视觉问答上追平编码器，而 in-fusion 基本和编码器重合。

实际中，大家常常是在训好的语言模型上加视觉。作者先训一个 1.7B 纯文本模型，再继续做多模态训练。in-fusion 是 50.1，从零编码器 48.6，不算预训练成本的 MetaCLIP 编码器反而只有 43.5，encoder-free 是 39.8。

最后是速度。在一张 H200 上，16 个并发请求，每个带一张 448 像素的图，从零编码器的首 token 延迟是 210 毫秒，in-fusion 是 182 毫秒，快了 13%，只加前馈分支的版本是 171 毫秒，快了 18%。原因是不用先等一个单独的编码器跑完。训练上，4B 时编码器方案比 encoder-free 慢大约 3%，in-fusion 一样快。

### 和以前方法的对比

和 Shukor 那篇的矛盾，作者在附录里找了原因。一是对方编码器的学习率可能太小，作者在自己的设置里把编码器学习率降到主干的 0.01 倍，1B 尺寸上它的视觉问答就和 encoder-free 差不多了。二是对方用文本、图文交错和 caption 几种 loss 的平均来拟合规模定律，视觉的信号被淹没了。

再和 MoT 比，MoT 就是给视觉 token 一整套单独的主干参数。在继续训练语言模型的设定下，全层 MoT 的视觉专用参数多得多，视觉问答却只有 43.3，不如只复制前 8 层的 48.5。

我的判断是，这篇最有用的结论是给 encoder-free 路线的。论文的建议是，去掉编码器的话，把它的参数改成最前面几层里的视觉专用模块，而不是加到共享参数里，视觉能力保得住，首 token 也更快。

### 局限

作者自己列了几条局限。第一，为了控制变量，编码器预训练用的图片和多模态训练是同一批，所以不能排除，用更多额外数据训练的现成编码器会有帮助。

第二，实验都是稠密模型，最大 6.4B，训练图片只有 224 的分辨率。第三，图文数据配比是按从零编码器选的，对其他方案未必最优。图文交错数据和视频也还没测。

### 未来方向

作者认为最值得做的下一步，是扩展到稀疏的混合专家主干，也就是 MoE，因为专家结构天然适合放这种按模态分配的容量。

我的判断是，这篇的价值在于把一个争了很久的问题拆清楚了。视觉编码器不一定要单独预训练，也不一定要放在主干之前，但视觉专用的参数不能省。

我的判断是，也要看到它的边界。4B 模型视觉问答平均 54 分出头，这些都是作者自己控制的小规模设置，分数不能直接和发布的大模型比。更大的模型、更高的分辨率下还成不成立，要等更大规模的验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Meta FAIR and New York University. It's called Unbundling Multimodal Vision Encoding.

The question sounds basic, but nobody has settled it. How should a multimodal language model take in images? Does it still need a vision encoder, and if it does, which part of that encoder is actually doing the work?

### Problem

Some context first. Most multimodal models start from a vision encoder that was pretrained separately, often on image-text alignment. It turns an image into a sequence of vectors, and the language model takes it from there.

Lately, recipes have split. Kimi K3 trains its vision encoder from scratch, jointly with the language model. Other models drop the encoder entirely. These are called encoder-free models, and they feed raw image patches straight into the language backbone.

The trouble is that you can't tell which approach is better by comparing released models. They differ in data, architecture and compute all at once, so any gap could come from anywhere.

### Prior work

There are controlled studies, but most of them compare one pretrained encoder against another. They change which visual prior the encoder brings in, not whether it needs one at all.

Studies of encoder versus encoder-free usually change several things at the same time. They also count only the multimodal training compute, and treat the encoder's pretraining as free. The paper's example is SigLIP 2, which was trained on forty billion image-text samples.

And one 2025 study by Shukor and colleagues trained both designs from scratch and found encoder-free models match or slightly beat the encoder. This paper reaches the opposite conclusion, and we'll get to why.

### Key idea

The key idea is that a vision encoder bundles three separate design choices. One, a visual prior from upstream pretraining. Two, dedicated vision capacity, meaning parameters that only image tokens pass through. Three, placing that capacity before fusion, so images are processed before they ever meet the text inside the shared backbone.

Seen that way, the recipes line up neatly. A pretrained encoder keeps all three. A from-scratch encoder gives up only the prior. Encoder-free gives up all three. The authors add a fourth recipe, in-fusion, which keeps only the dedicated capacity and moves it inside the backbone. This table lays it out.

Then they take the choices apart one at a time, with three questions. Does the encoder need separate pretraining? Does vision need dedicated capacity? And does that capacity have to come before fusion?

### Method

To keep things fair, every model is trained from scratch, language backbone included. They share one data pool, with text from Nemotron-CC and image-text pairs from MetaCLIP, one encoder design, one backbone design, and one optimization recipe.

The backbone follows the Qwen3 architecture. The main experiments use one point seven billion and four billion parameter backbones, with dedicated vision capacity fixed at zero point four billion. The encoder is a twenty-seven layer vision transformer.

The most important choice is what they call lifetime compute. It adds up the compute of every training stage, including encoder pretraining and any language-only pretraining, but not the final instruction tuning. All recipes are compared at the same total.

For evaluation, visual understanding is the average over sixteen Cambrian visual question answering benchmarks, grouped into general, knowledge, OCR and charts, and vision-centric. Language ability is the average over twelve text benchmarks. Each checkpoint is instruction tuned for three epochs on Cambrian-7M before the visual tests.

Here's how in-fusion works, in this figure. In selected backbone layers, the attention projections and the feed-forward network keep their shared weights, and each gets an extra set of weights that only image tokens use. Their output is simply added to the shared output.

The new weights start out producing zero, so training begins from the plain shared model. Because they only act on image tokens, text-only input is processed exactly as before, and attention still mixes image and text tokens together.

### Results

Question one, does the encoder need separate pretraining? With the one point seven billion backbone and a fixed total budget, the authors try four pretraining objectives: contrastive, self-supervised, and two captioning variants. When pretraining takes seven point four or fifteen percent of the budget, all four are on par with the from-scratch encoder. At sixty percent, both the visual and the text averages fall clearly below it.

The more striking part is that free pretraining doesn't help either. They raise MetaCLIP pretraining from one point four to nine times ten to the twenty-one FLOPs. ImageNet linear-probe accuracy climbs from eighty point four to eighty-four point four percent. But visual question answering is highest after the shortest pretraining, and a from-scratch encoder matches it. Better recognition doesn't translate into better VQA.

Question two, does vision need dedicated capacity? In this figure, encoder-free trails the from-scratch encoder on visual question answering at both sizes and every budget, while text scores stay about the same. The arrows show how much compute encoder-free needs to catch up, relative to the encoder: four point four times at one point seven billion, and three times at four billion.

You might object that encoder-free simply has zero point four billion fewer parameters. So the authors add them back to the backbone, wider, deeper, or both. In Table 2, the from-scratch encoder averages forty-four point four on VQA, encoder-free thirty-nine. The three enlarged versions land between thirty-eight point six and thirty-eight point eight. No catch-up at all, though their text scores rise, because text tokens use those parameters too. The biggest gap is on OCR and charts.

At two hundred twenty-four pixel training images, a richer patch embedding, the layer where image patches enter the model, doesn't fix it either. A two-layer projection gets forty point five, and a four-layer hierarchical MLP gets thirty-five point nine. The same parameters are worth more when they belong to vision alone.

Question three, does that capacity have to come before fusion? The authors move the encoder's zero point four billion parameters into the backbone as vision-only feed-forward branches, and try several placements. In Table 3, a narrow branch in every layer scores thirty-nine point four. Eight branches spread across depth, thirty-seven point six. The last eight layers, thirty-five point one. None beat encoder-free. Only the first eight layers work, at forty-five, on par with the from-scratch encoder's forty-four point four.

There's also a control that gives the same parameters in the first eight layers to every token. Widening the shared feed-forward layers scores forty point one, and a separate branch used by all tokens scores forty-two point three. Both fall short of the vision-only forty-five. The word that matters is dedicated.

The picture holds with longer training. At one point seven billion, the from-scratch encoder scores forty-eight point nine, in-fusion forty-nine point two, encoder-free forty-four point eight. At four billion, encoder fifty-four point five, in-fusion fifty-four point three, encoder-free fifty-one point seven.

They also run a ladder of five backbone sizes, from zero point six to six point four billion, and fit scaling laws. At every size, encoder-free needs roughly three times the encoder's compute to match it on VQA, while in-fusion tracks the encoder.

In practice, people often add vision to an already trained language model. So the authors pretrain a one point seven billion text-only model and continue it with multimodal data. In-fusion reaches fifty point one, the from-scratch encoder forty-eight point six. A MetaCLIP encoder whose pretraining isn't counted gets only forty-three point five, and encoder-free thirty-nine point eight.

Finally, speed. On one H200 with sixteen concurrent requests, each carrying a four hundred forty-eight pixel image, the from-scratch encoder takes two hundred ten milliseconds to first token, while in-fusion takes one hundred eighty-two, thirteen percent faster, and the feed-forward-only version one hundred seventy-one, eighteen percent faster. The backbone no longer waits for a separate encoder to finish. In training, the encoder recipe is about three percent slower than encoder-free at four billion, while in-fusion trains just as fast.

### Comparison

On the clash with Shukor's study, the authors trace it to that paper's setup. First, its encoder learning rate may have been too small: when they lower the encoder's rate to one hundredth of the backbone's, the encoder is on par with encoder-free on VQA at one billion parameters. Second, it fit scaling laws to an average of text, interleaved and caption losses, which drowns out the vision signal.

Then there's MoT, mixture-of-transformers, which gives vision tokens their own full copy of the backbone's weights. When continuing a pretrained language model, all-layer MoT has far more vision-only parameters, yet it scores forty-three point three, below the forty-eight point five of a version that copies only the first eight layers.

My read is that the most useful takeaway is for the encoder-free camp. The paper's own suggestion is that if you drop the encoder, you should turn its parameters into vision-only modules in the earliest layers, not add them to the shared weights. You keep the visual ability and get a faster first token.

### Limitations

The authors list several limitations. First, to keep things controlled, encoders are pretrained on the same images as multimodal training. So the results don't rule out a benefit from released encoders trained on additional data.

Second, every backbone is dense, up to six point four billion parameters. Third, training uses two hundred twenty-four by two hundred twenty-four images, so higher resolution and text-heavy data are left open. Fourth, the image-text mixture was tuned with the from-scratch encoder at one point seven billion, so it may not be best for the other recipes. Interleaved image-text data and video are also untested.

### Future work

The authors see sparse mixture-of-experts backbones as the most compelling next step, since experts are a natural home for capacity that is routed by modality.

My read is that the value here is clarity on a long-running debate. The practical answer: the vision encoder needn't be pretrained separately, and needn't sit before the backbone, but vision does need parameters of its own.

My read is also that the limits matter. These are small models in the authors' controlled setup, with the four billion models averaging around fifty-four on VQA, so the scores aren't comparable to large released models. Whether the conclusions hold for much larger models at higher resolution still needs testing at scale.

That's today's Daily Frontier Papers. The paper is linked in the description.

