# Proving and Improving the Platonic Representation Hypothesis with JEPAs

**图文对齐不只靠规模更要选对正样本**

- 论文 Paper: [arXiv 2610.39302](https://arxiv.org/abs/2610.39302)
- 单位 Affiliation: 布朗大学 · 纽约大学 · AMI Labs
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-01
- 视频 Videos: [YouTube 中文](https://youtu.be/ZLYAVt_KOwU) · [YouTube English](https://youtu.be/7__pB138PX0) · [B站](https://www.bilibili.com/video/BV1BRx466Etq)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇布朗大学、纽约大学和 AMI Labs 的论文，作者是 Randall Balestriero 和 Yann LeCun，题目是，用 JEPA 证明并改进柏拉图表征假说。

它回答一个老问题，只看图片的模型和只看文字的模型，分开训练，会不会学到同一种对世界的表示。作者的答案是，会，前提是正样本选对了，而靠下一个 token 预测堆规模，是一条很低效的路。

### 问题

先说背景。有一篇很有名的论文观察到，视觉模型和语言模型的表征，随着规模变大会越来越像，并提出了柏拉图表征假说，简称 PRH，意思是不同模态的模型，最终会收敛到同一个对现实的统计模型。

问题在于，这一直只是观察。作者指出，对实际在用的自监督目标，特别是 JEPA，没有证明，也说不清什么时候成立，所以大家主要的手段，还是把模型和数据往大了堆。

### 以前的方法

以前有几条线索。分开训练的不同语言的词向量，被发现只差一个旋转。理论上，PRH 原论文依赖一个理想化的对比学习论证，已知最接近的证明，只覆盖深度线性网络。

还有不少工作，研究怎么把两个现成的表征空间对齐。这篇要问的是另一件事，什么样的训练目标，能让编码器本来就可以对齐。

### 核心想法

先解释 JEPA，联合嵌入预测架构，一类自监督方法。给模型一对它认为是同一个东西的输入，叫正样本对，让两者的向量靠近，再加一项防坍缩的约束，防止所有输入都挤到同一个点。

作者的关键洞察是，PRH 其实是一个关于正样本的命题。正样本如果是同一场景的两次观察，比如同一张照片的两条描述，叫好视角，学到的表征属于世界。如果只是同一次观察的两种破坏，比如同一句话遮住不同的词，叫坏视角，学到的表征属于这个模态自己。

论文第一张图的左边就是这个意思。好视角下，图片和文字各训练一个 JEPA，结果只差一个旋转。下面是坏视角，遮掉一个词，或者前半句接下一个词。

### 方法

第一个结果，作者叫它柏拉图表征定理。在一个有限的世界里，每个模态都一一对应地呈现世界状态。如果所有模态共用同一种正样本分布，最优解又在旋转意义下唯一，那么每个模态学到的，都是同一个世界嵌入，只差一个旋转。

证明的思路很直接，一一对应的呈现只是换了坐标，各个模态的训练，其实是同一个问题，写在不同的坐标系里。作者还证明，带白化这类约束的 JEPA，只要特征值之间有间隔，解就只差一个旋转。

现实中，不同模态的正样本不会完全一样。第二个定理给了上界，错位多少，只取决于两边正样本画出的图差多少，和谱的顶部分得多开。在 300 个随机生成的小世界上，误差都在上界以下，最多到上界的 18%。

第三个结果解释坏视角为什么差。每次观察都带着只属于本模态的干扰，比如一条描述用了哪些词，照片从哪个角度拍。坏视角的两个正样本共享这个干扰，如果保留的干扰比保留的场景信息还多，模型就会把一部分维度花在干扰上，这部分，任何旋转都对不上另一个模态。

然后是和 NLP 关系最大的一步。作者证明，下一个 token 预测，形式上就是一个 JEPA。上下文是一边，下一个 token 是另一边，softmax 的归一化项，扮演防坍缩的角色。

它的正样本，是同一篇文档里的前缀和后续，共享的是这篇文档的用词和文风，所以按作者的说法，它是一个坏视角 JEPA。作者也强调，这一步是启发式的，是有待实验检验的预测。

看这张图。好视角把同一场景的描述连在一起，从不连到别的场景。下一个 token 预测，场景内的连接很窄，还会通过高频词串到别的场景。作者认为，这就是语言模型要靠规模才能和视觉对齐的原因。

### 实验和效果

实验设置很统一。图像和文字编码器分开训练，用同一个 LeJEPA 目标，数据是 Flickr30k，每张图 5 条描述。文字端是 3300 万参数的 BERT，从零训练，没有一个编码器见过另一个模态。

文字端有三个版本，只差正样本。坏视角，取同一条描述的片段。好视角，取同一张图的不同描述。第三个，同一个 BERT 改用下一个 token 预测。衡量对齐，主要看只用一个旋转把两边对上之后，用图片检索描述，第一名就对的比例。

结果在表 1。好视角的命中率是 4.4%，是随机水平的 44 倍。坏视角是 2.7%，下一个 token 预测是 2.1%。论文说，这个排序在每个随机种子下都成立。

一个关键对照，把同一张图的两条描述拼起来，交给下一个 token 预测，命中率只从 2.1% 到 2.3%。作者的结论是，提升来自目标函数本身，而不是知道两条描述属于同一张图。

接下来是规模。文字端换成网上的 Recap-DataComp，每张网图有一条原始 alt-text，和两条由图像描述模型写的描述，三条互为好视角。

第一个结论是样本效率。同一个 1.09 亿参数的编码器，同一份语料，好视角在 2160 万个样本时，在 PRH 原论文的 WIT 基准上达到 0.109，下一个 token 预测要 9 亿个样本，才到 0.111，多了 42 倍。需要说明，下一个 token 预测只评了 9 亿样本这一个检查点。

第二个结论最抢眼。用 9 亿个样本训练这个好视角 JEPA，在 WIT 上和 DINOv2 的对齐是 0.184，高于作者测的所有语言模型，包括 Qwen2.5-72B 的 0.154，和 Llama-3-70B 的 0.151。同样文字上的下一个 token 预测只有 0.111。

换成 CLIP 的图像编码器，它依然领先，0.230 对 0.210。作者说，这个模型读了 1160 亿个文字 token，赢了参数大 660 倍、训练数据多 100 多倍的模型。

作者也写了保留。第四个随机种子换了正则权重，只到 0.121，四个种子平均 0.168，仍高于所有语言模型。另外，写大部分训练文字的描述模型，是通过 CLIP 的视觉编码器看图的。

第三个结论是，对齐也有缩放定律。作者按不同比例混合好视角和坏视角，拟合了一个公式，然后在训练前预测 85 个新实验，误差 0.0093，而同一实验换种子重跑，差异是 0.0096。

不过外推到 3.2 亿样本时，它高估了有好视角的结果。作者也说，它只是一个配方的经验总结。

最后是两个应用。第一个，图像描述器。解码器只用文字训练，描述图片时，用冻结的 DINOv2 和 DINOv3 编码，经过一个旋转，再解码，只有这个旋转用了图文配对。在 COCO 上 CIDEr 是 0.838，论文列出的基于 CLIP 的描述器，从 0.345 到 0.918 不等。

第二个应用，一个配对都不用，只靠两边的几何结构找旋转。ImageNet-100 的零样本 top-5 准确率，好视角 JEPA 是 64.4%，同样条件下重跑 Qwen3-8B 是 47.6%。不过 CLIP 的文字编码器更高，68.7%，而同样文字上的下一个 token 预测，一半的运行都失败了。

### 和以前方法的对比

放在一起看。PRH 原论文是观察加一个理想化的论证，这篇给了 JEPA 下一个可证明的充分条件，正样本对上了，表征就只差一个旋转。同样的文字、同样的编码器，好视角在论文跑的绝大多数对照里，都比下一个 token 预测对齐得好。

我的判断是，这篇最有意思的地方，是把下一个 token 预测放进了 JEPA 的框架，用正样本的好坏，解释了语言模型为什么要靠规模才能和视觉对齐。

我的判断是，这不等于说下一个 token 预测要被替代。这篇只比了一件事，和视觉模型的对齐程度，没有比生成，也没有比推理。

### 局限

作者自己写了几条局限。第一，定理是理想化的，假设一一对应的呈现，而真实的模态是有损的，也没有一个实验完全满足定理一的条件。

第二，也是作者说的主要保留，好视角从哪里来。网络数据上，三条文字有两条是图像描述模型生成的，带着从图文训练中蒸馏来的信息。不用描述模型，只用 alt-text 和它的纯文本改写，WIT 上只有 0.060，用生成的描述是 0.107。

所以和语言模型、和 CLIP 的大规模对比，依赖这些生成的文字。而同一份文字上不同视角之间的对比，不受影响。

第三，超参数只为好视角 JEPA 调过，坏视角和下一个 token 预测，没有单独调。

### 未来方向

作者希望，正样本的选择，而不只是规模，成为构建柏拉图表征的杠杆，并推广到音频、视频和动作这些模态。

我的判断是，这篇的价值，是把一个常被挂在嘴边的假说，变成了可以设计的东西，想要对齐，就去选对正样本。

我的判断是，也要看到，最亮眼的那组数字，靠的是用图文数据训练过的描述模型来生成好视角。没有现成配对信息的模态，好视角从哪里来，决定了这条路能走多远。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Brown University, New York University and AMI Labs, by Randall Balestriero and Yann LeCun. The title: proving and improving the Platonic Representation Hypothesis with JEPAs.

It takes on an old question. If you train one model only on images and another only on text, do they end up with the same picture of the world? The answer here is yes, if you choose the right positive pairs, and that scaling up next-token prediction is a very inefficient way to get there.

### Problem

Some background. An influential earlier paper observed that vision models and language models grow more similar as they scale. It proposed the Platonic Representation Hypothesis, or PRH: models of different modalities converge toward one shared statistical model of reality.

The trouble is that it has stayed an observation. For the self-supervised objectives people actually use, JEPAs in particular, there's no proof and no prediction of when it holds. So the main lever has been to make models and data bigger.

### Prior work

There were hints. Word embeddings trained separately on different languages turned out to differ by just a rotation. On the theory side, the original PRH argument rests on an idealized contrastive learner, and the closest proof the authors know of covers deep linear networks.

Plenty of work also aligns two existing embedding spaces after the fact. This paper asks a different question: which training objective makes encoders alignable in the first place?

### Key idea

First, what a JEPA is: a joint-embedding predictive architecture, a family of self-supervised methods. You give the model two inputs it should treat as the same thing, a positive pair, pull their embeddings together, and add an anti-collapse term so everything doesn't land on one point.

The key insight is that the PRH is really a statement about positive pairs. If the positives are two observations of one scene, say two captions of the same photo, the authors call them good views, and the representation belongs to the world. If they're two corruptions of one observation, like one sentence with different words masked, those are bad views, and the representation belongs to the modality.

That's the left half of Figure 1. With good views, an image JEPA and a text JEPA land on the same embedding, up to one rotation. Below are the bad views: a masked word, or a prefix and its next token.

### Method

The first result is what the authors call the Platonic Representation Theorem. Take a finite world, with each modality rendering its states one to one. If every modality shares the same distribution of positive pairs, and the optimum is unique up to rotation, then every modality learns the same embedding of the world, up to one rotation each.

The proof is short: a one-to-one rendering just changes coordinates, so each modality's training problem is the same problem written in a different coordinate system. They also show that JEPAs with a whitening-style constraint have optima unique up to rotation, as long as there's a gap between eigenvalues.

In practice, modalities won't share exactly the same positives. The second theorem bounds the misalignment by two things: how different the graphs drawn by the two sets of positives are, and how well separated the top of the spectrum is. On three hundred randomly generated small worlds, the error always stayed below the bound, at eighteen percent of it at most.

The third result explains why bad views hurt. Every observation carries a nuisance of its own modality, like which words a caption happened to use, or the angle a photo was taken from. Bad-view positives share that nuisance, and when they keep more of it than of the scene, the encoder spends dimensions on it, and no rotation can line those up with the other modality.

Then comes the step that matters most for NLP. The authors show that next-token prediction has the form of a JEPA. The context is one side, the next token is the other, and the softmax normalizer plays the anti-collapse role.

Its positives are a prefix and its continuation inside one document, which share that document's wording and style. So, in the authors' framing, it's a bad-view JEPA. They stress that this step is heuristic, a prediction to test rather than a corollary.

Look at this figure. Good views connect every description of a scene, and never another scene. Next-token prediction makes narrow links within a scene, and spills across scenes through frequent tokens. The authors read this as the reason language models align with vision mostly through scale.

### Results

The setup is deliberately uniform. Image and text encoders are trained separately with the same LeJEPA objective, on Flickr30k, five captions per image. The text side is a thirty-three-million-parameter BERT trained from scratch, and no encoder ever sees the other modality.

There are three text variants that differ only in their positives. Bad views take spans of one caption. Good views take different captions of the same image. And the same BERT, made causal, trained with next-token prediction. The headline metric: fit a single rotation between the two spaces, then check how often an image retrieves its own caption first.

Table 1 has the results. Good views retrieve the right caption first four point four percent of the time, forty-four times chance. Bad views get two point seven, next-token prediction two point one. The paper says that ranking holds in every seed.

A key control: train next-token prediction on two captions of the same image joined together. It barely moves, from two point one to two point three. The authors conclude that the gain comes from the objective itself, not from knowing which captions go together.

Then scale. The text side moves to web data, Recap-DataComp. Each web image has its original alt-text and two captions written by an image captioner, and those three texts are each other's good views.

First finding: sample efficiency. With the same hundred-and-nine-million-parameter encoder and the same corpus, good views reach point one zero nine on the PRH paper's WIT benchmark after twenty-one point six million samples, while next-token prediction needs nine hundred million samples to reach point one one one, forty-two times more. One caveat: next-token prediction was only scored at that nine-hundred-million checkpoint.

Second finding, the eye-catching one. Trained on nine hundred million samples, the good-view JEPA scores point one eight four against DINOv2 on WIT. That's above every language model the authors evaluated, including Qwen2.5-72B at point one five four and Llama-3-70B at point one five one. Next-token prediction on the same texts gets point one one one.

Swap in CLIP's image encoder and it still leads, point two three zero against point two one zero. In the authors' words, this model read a hundred and sixteen billion text tokens and beat models six hundred and sixty times larger, trained on over a hundred times more.

They flag caveats. A fourth seed, with a different regularization weight, reached only point one two one; all four seeds average point one six eight, still above every language model. And the captioner that wrote most of these texts sees images through CLIP's vision encoder.

Third finding: alignment has a scaling law. Mixing good and bad views at different rates, they fit a formula, then predicted eighty-five new runs before training them. The prediction error was point zero zero nine three, while rerunning one experiment with another seed differs by point zero zero nine six.

But extrapolated to three hundred and twenty million samples, it overestimates the runs with good views. The authors call it an empirical summary of one recipe.

Two applications close the paper. First, a captioner. The decoder is trained on text alone. To caption an image, embed it with frozen DINOv2 and DINOv3, apply one rotation, and decode; only the rotation is fitted on image–caption pairs. On COCO it reaches a CIDEr of point eight three eight. The CLIP-based captioners the paper lists report between point three four five and point nine one eight.

Second, alignment with no pairs at all, finding the rotation from the geometry of the two spaces alone. On zero-shot ImageNet-100 top-five accuracy, the good-view JEPA gets sixty-four point four percent, versus forty-seven point six for Qwen3-8B rerun under identical conditions. CLIP's text encoder still scores higher, at sixty-eight point seven, while next-token prediction on the same texts fails in half its runs.

### Comparison

Putting it together. The original PRH was an observation plus an idealized argument. This paper gives a provable sufficient condition for JEPAs: match the positives, and the representations agree up to one rotation. With the same texts and the same encoder, good views beat next-token prediction in all but a few of the comparisons the paper ran.

My read is that the most interesting move here is folding next-token prediction into the JEPA framework, which turns "language models align with vision through scale" into a statement about weak positive pairs.

My read is also that this doesn't say next-token prediction should be replaced. The paper measures one thing, alignment with vision models. It doesn't compare generation, and it doesn't compare reasoning.

### Limitations

The authors list several limitations. First, the theorems are idealized: they assume one-to-one renderings, while real modalities are lossy, and no experiment meets the conditions of Theorem 1 exactly.

Second, and the main caveat in their own words: where do good views come from? On web data, two of each image's three texts come from an image captioner, carrying information distilled from image–text training. Without any captioner, using the alt-text and a text-only paraphrase, WIT drops to point zero six zero, against point one zero seven with the recaptions.

So the large-scale comparisons with language models and CLIP rely on those generated texts. The comparisons between view types, on identical texts, don't.

Third, hyperparameters were tuned only for the good-view JEPA, not for the bad-view and next-token baselines.

### Future work

The authors hope positive views, and not scale alone, become the lever for building platonic representations, extended to modalities like audio, video and actions.

My read is that the value here is turning a much-quoted hypothesis into something you can design for. If you want alignment, choose the right positives.

My read is also that the most striking numbers depend on a captioner trained on image–text data to produce the good views. For modalities with no paired data to lean on, where good views come from will decide how far this goes.

That's today's Daily Frontier Papers. The paper is linked in the description.

