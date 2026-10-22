# Looking Ahead: Modeling Future Evolution in Autoregressive Video Generation

**逼视频模型学会往前看**

- 论文 Paper: [arXiv 2610.29989](https://arxiv.org/abs/2610.29989)
- 单位 Affiliation: 南京大学、腾讯 ARC Lab、浙江大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-22
- 视频 Videos: [YouTube 中文](https://youtu.be/b4M3LSbEHjw) · [YouTube English](https://youtu.be/zlxB0Q6pKo0) · [B站](https://www.bilibili.com/video/BV1dRn26CELW)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇南京大学和腾讯 ARC Lab 的论文，浙江大学也有参与，题目叫 Looking Ahead，往前看，讲的是怎么让自回归视频生成模型，学会建模未来的变化。

它问的问题很具体，视频模型能一段接一段地往下生成，看起来也挺顺，但它到底有没有学会，画面里的东西接下来会怎么变。

### 问题

先解释一下自回归视频生成。它不是一次生成一整段视频，而是把视频切成很多小段，论文叫 chunk，每次根据前面已有的画面，生成下一小段，这样视频就能一直往下接。

训练这类模型，常用的办法叫 teacher forcing。意思是训练时，前面的历史画面直接用真实视频，模型只负责把当前这一段从噪声里还原出来。

问题就出在这里。视频前后两段的外观和动作通常非常像，模型只要照着前面的样子往下顺，就能把当前这段还原得不错。

作者认为，这就成了一条捷径。模型靠局部的连续性就能交差，去学场景到底怎么演变的压力就小了，所以能续写，不等于懂得接下来会发生什么。

### 以前的方法

以前提升自回归视频质量，主要有几条路。比如把一次生成整段的双向模型，改造成按顺序生成、步数很少的生成器，或者训练时让模型看自己生成的历史，再或者在推理时管理历史上下文，比如滑动窗口、缓存和压缩。

还有一类做法叫表征对齐，把视频模型中间层的特征，去对齐一个外部预训练好的视觉模型。作者指出，这类方法依赖外部编码器，扩展到大规模视频数据可能更麻烦。

和这篇最接近的，是同期的另一篇工作 Next Forcing，也是让模型预测更远的未来。作者说两者是独立、同时做的，实验里也做了对比。

### 核心想法

这篇的核心想法很简单，既然紧挨着的下一段太好猜，那就让模型去猜更远的未来。

目标离已知的历史越远，局部的外观和动作连续性能帮的忙就越少，模型就得真正理解，时间是怎么推进的。

而且这个监督信号就来自训练视频本身，不需要额外的预训练模型。作者把这个方法叫 Future Chunk Prediction，未来片段预测，简称 FCP。

### 方法

具体来说，正常训练是给定前 k 段，预测第 k 加 1 段。FCP 额外要求，用同样的信息去预测更后面的第 k 加 delta 段，中间隔着的那几段，故意不给模型看。

这张图是整体结构。左边是主生成器，也就是原来的视频扩散模型，右边是几个额外的小预测器，它们读取主生成器中间层的隐藏状态，各自去预测未来的某一段。

为什么要单独做预测器。作者在早期实验里发现，如果主生成器和未来预测共用一套参数，时间上的连贯性反而变差，可能是两个目标在抢同一套参数。

但预测器如果完全独立训练，主生成器就什么也学不到。所以预测的误差会沿着这些隐藏状态，反向传回主生成器，逼着主生成器的中间表征，把未来怎么变的信息编码进去。

在主生成器里，同一段视频有两种身份，作为历史时它是干净的画面，作为正在生成的目标时它带着噪声，FCP 把这两种状态的表征，都拿来做未来预测。

预测器只看当前这一段的隐藏状态，不看全部历史，这样表征要更紧凑，计算也更省。它也不看文字提示，因为文字描述可能直接泄露后面会发生什么，作者希望它只靠画面的表征去推。

还有一个设计，是并行而不是串联。这张图左边是串联，每个预测器接着上一个预测器往下传，右边是并行，每个时间距离一个独立的预测器，都直接从主生成器的表征出发。

作者担心串联会留后门。扩散模型训练时，每个预测器拿到的是加了噪声的未来画面，串起来以后，中间那一段的信息，可能顺着链条漏给后面的预测器，又变成一条捷径。

训练时，主生成器原来的损失，加上各个未来预测器的损失，一起优化。到了推理阶段，这些预测器全部去掉，生成过程和原来一模一样，没有额外开销。

### 实验和效果

实验设置。主生成器从开源视频模型 Wan 的官方权重初始化，用了 1.3B 和 5B 两个尺寸，做的是图生视频，每个预测器只有三层 DiT，也就是扩散 Transformer 的模块，从主生成器复制过来初始化。

训练数据是内部视频，加上公开的 WISA-80K，主要是普通的真实世界视频，没有用物理模拟器的数据。

评测重点是物理合理性，主要的 benchmark 叫 Physics-IQ Verified，它拿生成的视频和真实拍摄的视频直接比数值，不靠视觉语言模型来打分。

最关键的是同等条件下的对比，同样的数据，同样的设置，只差一个 FCP。1.3B 模型的总分 IQ，从 27.6 提到 30.9。

5B 模型在表 2 里，图生视频从 30.4 提到 33.8，视频续写，也就是给一段视频接着往下生成，从 40.6 提到 44.4。

放到更大的榜单里看，表 1。作者的 5B 模型总分 33.8，是表里自回归模型的最高分，也超过了几个更大的双向模型，比如 Wan2.2 的 14B 和 HunyuanVideo-1.5。

但要注意，表里最高的仍然是双向模型 Grok Imagine Video，34.8，比它高 1 分。作者自己也说，不同模型的训练数据和设置都不一样，跨模型的差距不能都算到方法头上。

这张图是两个例子。左边是多米诺骨牌，FCP 生成的骨牌沿着该倒的方向倒下去，原方法在中间出现了反常的塌陷，右边是冰块融化，FCP 生成的水位跟着上升了，原方法没有体现出来。

在另外三个物理相关的 benchmark 上，FCP 也基本都比原方法好，比如 WorldModelBench 总分从 7.22 到 7.64，不过有的提升很小，PhyGround 两项只多了 0.03 和 0.05。作者的解释是，这几个 benchmark 靠视觉语言模型打分，对细微的物理变化可能不够敏感。

训练成本也很关键，看这张图。横轴是训练算力，纵轴是 IQ 分数，在差不多的算力下，FCP 的分数比原方法高，作者据此认为，提升不是靠多花算力换来的。

作者还看了模型内部的表征。这张图里，选画面里的一个点，看其他位置和它有多像，原方法的相似区域会散到不相关的地方，FCP 的更集中在被选中的物体和同类物体上。

跨帧来看，相似区域也会跟着物体一起移动。作者认为，这种结构更清楚的表征，可能对以后做世界模型有用。

### 和以前方法的对比

先看和 Next Forcing 思路的对比。Next Forcing 让预测器看全部历史，作者按这个思路做了一个看全部历史的 FCP 变体来比，注意这不是 Next Forcing 原本的模型。要达到 29 分和 30 分，作者的版本估计分别少用 42.6% 和 50.7% 的训练算力。

再看并行和串联。在 1.3B 上，并行比串联高 0.8 分。到了 5B，两个串联版本在图生视频上只比原方法高一点点，在视频续写上没有提升，并行版本则都有提升。

作者的解释是，模型越大，可能越容易利用串联里漏过来的未来信息，捷径的问题也就可能越明显。

两个消融也说明了设计的道理。给预测器加上文字提示，分数反而降了 1.1 分。只约束干净历史的表征，或者只约束带噪声目标的表征，分别提升 1.2 分和 2.3 分，两个都约束，提升 3.3 分。

我的判断是，这篇的价值在于它很省事。不改推理，不加外部模型，只在训练里多加一个往前看的任务，就在同等条件下稳定提分，别的自回归视频模型也很容易照着加。

### 局限

论文的局限，作者自己写了两条。第一，受计算资源限制，没有试更大的模型，也没有在长得多的视频上训练。

第二，所有模型都是从双向模型改过来的，不是从头预训练的。作者说改造过程中，早期性能会先下降，这种初始化可能限制了 FCP 的效果能发挥多少。

另外，评测主要集中在物理合理性上，视频演变的其他方面，这篇没有专门去测。

训练还用了内部数据，完整数据不能公开，作者说用同样的数据完全复现训练可能做不到。代码已经公开，但论文说源码包里不含模型权重。

### 未来方向

未来方向，作者提了几条。一是把 FCP 当成原生的训练目标，从头大规模预训练自回归的世界模型。

二是把那些辅助预测器在推理时用起来。这张图是它们的预测，没看中间的画面，也能大致预测出后面的运动趋势，但画质还不够，而且越远越差。作者设想用它们在早期去噪阶段加速采样，或者和主模型的预测结合起来，不过都还没验证。

三是加入物理模拟器的数据，作者认为这可能进一步提升物理合理性。

我的判断是，这篇最值得关注的，是它点出的问题，teacher forcing 让模型太容易抄近路。往前多看几步这个办法，代价很小，后面很可能会被更多自回归视频和世界模型的工作借鉴。

我的判断是，也要看到，提升的绝对幅度不算大，同等条件下大约 3 到 4 分，最好的 5B 模型仍然低于 Grok Imagine Video，而且评测几乎都围绕物理合理性，它对长视频和其他能力的帮助，还需要更多验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Nanjing University and Tencent's ARC Lab, with Zhejiang University, and it's called Looking Ahead. It's about teaching autoregressive video generators to model how a scene changes over time.

The question is a sharp one. A video model can keep extending a clip, chunk after chunk, and the result looks smooth. But has it actually learned what happens next?

### Problem

First, what autoregressive video generation means. Instead of producing a whole clip in one shot, the model cuts the video into short pieces the paper calls chunks, and generates each new chunk conditioned on everything before it. That's what lets the video keep going.

The standard way to train these models is called teacher forcing. During training, the history is taken straight from the real video, and the model only has to denoise the current chunk.

And that's where the trouble starts. Neighboring chunks in a video look almost the same, with similar appearance and similar motion. So the model can do a decent job on the current chunk just by continuing what it already sees.

The authors argue this becomes a shortcut. Local continuity is enough to pass, so there's little pressure to learn how the scene actually evolves. Being able to continue a video isn't the same as understanding what comes next.

### Prior work

Earlier work attacked rollout quality from a few directions. Adapting bidirectional models, which generate a whole clip at once, into causal few-step generators. Training the model on its own generated history. Or managing the history at inference time, with sliding windows, caches and compression.

Another line of work is representation alignment, where the video model's intermediate features are pulled toward an external, pretrained vision model. The authors point out that this depends on outside encoders, which can make scaling to large video corpora harder.

The closest work is a concurrent paper called Next Forcing, which also has the model predict further into the future. The authors say the two were developed independently, and they compare against that idea in their experiments.

### Key idea

The core idea is simple. If the very next chunk is too easy to guess, make the model guess something further away.

The further the target is from the observed history, the less local appearance and motion can help. The model has to actually capture how time moves the scene forward.

And the supervision comes straight from the training video itself, with no extra pretrained model. The authors call it Future Chunk Prediction, or FCP.

### Method

Concretely, normal training predicts chunk k plus one from the first k chunks. FCP adds a second task: from that same information, predict a later chunk, k plus delta, with the chunks in between deliberately hidden.

Here's the overall architecture. On the left is the main generator, the original video diffusion model. On the right are a few small extra predictors that read the generator's intermediate hidden states, and each one predicts a particular future chunk.

Why separate predictors? In early experiments, the authors found that sharing parameters between the generator and the future-prediction task actually hurt temporal coherence, possibly because the two objectives compete for the same weights.

But a predictor trained in isolation would leave the generator untouched. So the prediction loss flows back through those hidden states into the generator, forcing its internal representations to encode how the scene will change.

Inside the generator, each chunk plays two roles. As history, it's clean. As the current target, it's noisy and being denoised. FCP attaches future prediction to both of those representations.

Each predictor sees only the current chunk's hidden state, not the full history, which keeps the representation compact and the compute low. It also gets no text prompt, because a caption could simply give away what happens next, and the authors want the predictor to rely on what the video representation already holds.

One more design choice is parallel rather than chained prediction. On the left of this figure, the chained version passes hidden states from one predictor to the next. On the right, the parallel version gives each time offset its own predictor, all starting directly from the generator's representation.

The worry with chaining is a back door. In diffusion training, each predictor receives a noisy version of its future chunk, so in a chain, information about an intermediate chunk can leak to the later predictors and create a new shortcut.

During training, the generator's usual loss and all the future-prediction losses are optimized together. At inference, every predictor is thrown away, so generation runs exactly as before with no extra cost.

### Results

The setup. The generators start from the official Wan video model checkpoints, in two sizes, 1.3B and 5B, working on image-to-video. Each predictor is just three DiT blocks, that is, diffusion transformer blocks, copied from the generator to start.

Training data is a mix of internal videos and the public WISA-80K set, mostly ordinary real-world footage, with no physics-simulator data.

The evaluation focuses on physical plausibility. The main benchmark, Physics-IQ Verified, compares generated videos numerically against real recordings, rather than asking a vision-language model to judge them.

The most important results are the controlled ones: same data, same settings, the only difference is FCP. For the 1.3B model, the overall IQ score goes from twenty-seven point six to thirty point nine.

For the 5B model, in Table 2, image-to-video goes from thirty point four to thirty-three point eight, and video-to-video, where the model continues a given clip, goes from forty point six to forty-four point four.

Now the wider leaderboard, Table 1. The 5B FCP model scores thirty-three point eight, the best of the autoregressive models in the table, and above several larger bidirectional models, including Wan2.2 14B and HunyuanVideo-1.5.

But the top of the table is still a bidirectional model, Grok Imagine Video, at thirty-four point eight, one point higher. The authors themselves caution that different training data and settings mean cross-model gaps can't all be credited to the method.

This figure shows two examples. On the left, dominoes: with FCP they topple in the expected direction, while the baseline produces an odd collapse in the middle. On the right, melting ice: FCP shows the water level rising, and the baseline misses it.

On three other physics-related benchmarks, FCP generally beats the baseline, for example WorldModelBench total goes from seven point two two to seven point six four, though some gains are tiny, with the two PhyGround scores up just zero point zero three and zero point zero five. The authors suggest these benchmarks, which use vision-language model judges, may not be sensitive to subtle physical changes.

Training cost matters too, so look at this chart. Training compute on the horizontal axis, IQ score on the vertical. At comparable compute, FCP scores higher than teacher forcing, which the authors take as evidence the gain isn't just from spending more compute.

The authors also looked inside the model. In this figure, they pick one point in the frame and map how similar every other position is to it. With teacher forcing, the similar regions spill into unrelated areas. With FCP, they concentrate on the chosen object and similar objects.

Across frames, the similar regions also move along with the object. The authors believe representations this structured may be useful for building world models.

### Comparison

First, against the Next Forcing idea. Next Forcing conditions its predictors on the full history, so the authors built a full-history version of FCP to compare, which is their variant, not the original Next Forcing model. To reach IQ scores of twenty-nine and thirty, their version is estimated to need forty-two point six and fifty point seven percent less training compute.

Next, parallel versus chained. At 1.3B, parallel beats chained by zero point eight points. At 5B, the two chained versions barely improve on the baseline for image-to-video and don't improve video-to-video at all, while the parallel versions improve both.

The authors' explanation is that a bigger model may be better at exploiting the future information that leaks through the chain, so the shortcut problem may get worse with scale.

Two ablations back up the other design choices. Giving the predictors the text prompt lowers the score by one point one. Regularizing only the clean-history representation, or only the noisy-target one, gains one point two and two point three points, and doing both gains three point three.

My read is that the appeal here is how cheap it is. No change at inference, no external model, just one extra look-ahead task during training, and it gives consistent gains under matched conditions. Other autoregressive video models could adopt it easily.

### Limitations

The authors list two limitations themselves. First, compute limits meant no larger models and no training on much longer videos.

Second, every model starts from a bidirectional checkpoint rather than being pretrained from scratch. The authors saw early performance drop during that conversion, and they suspect this initialization may cap how much FCP can help.

Beyond that, the evaluation is centered on physical plausibility, and other aspects of how video evolves aren't specifically tested here.

Training also used internal data that can't be fully released, and the authors say exact reproduction with identical data may not be possible. The code is public, but the paper says model checkpoints aren't included in the source package.

### Future work

Looking ahead, the authors mention a few directions. One is to use FCP as a native objective in large-scale from-scratch pretraining of autoregressive world models.

Another is to put those auxiliary predictors to work at inference. This figure shows their predictions: without seeing the intervening frames, they still capture the broad motion trend, but visual quality isn't there yet, and it gets worse the further out they predict. The authors suggest using them for early denoising steps to speed up sampling, or combining their predictions with the main generator's, but neither has been tested.

A third is adding physics-simulator data, which the authors think could further improve physical plausibility.

My read is that the most important thing here is the problem it names: teacher forcing makes it too easy for a video model to take a shortcut. Looking a few steps ahead is a cheap fix, and I'd expect more autoregressive video and world-model work to borrow it.

My read is also that the gains are modest in absolute terms, roughly three to four points under matched conditions, the best 5B model still trails Grok Imagine Video, and the evaluation is almost entirely about physical plausibility. Whether it helps long videos and other abilities still needs more evidence.

That's today's Daily Frontier Papers. The paper is linked in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
