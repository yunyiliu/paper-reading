# FastFrames: Video Generation at Interactive Latency

**AI视频一帧只要36毫秒**

- 论文 Paper: [arXiv 2611.23226](https://arxiv.org/abs/2611.23226)
- 单位 Affiliation: Apple · 康奈尔大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-18
- 视频 Videos: [YouTube 中文](https://youtu.be/mukoscp-_xM) · [B站](https://www.bilibili.com/video/BV1n8sh6nEAN)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 Apple 和康奈尔大学的论文，名字叫 FastFrames，副标题是，让视频生成做到交互级的延迟。

先说结论，作者报告，他们 13 亿参数的版本，在一张 B200 显卡上，生成一帧只要 36 毫秒，画质和一个慢了将近 8 倍的最新模型相当。

### 问题

先说背景。自回归视频扩散模型，是一帧一帧往下生成视频的，所以每一步都能接收新的输入，比如方向键、镜头控制，很适合做世界模拟、游戏，还有机器人的预测模型。

问题在于，很多模型说的实时，指的是每秒出多少帧，也就是 FPS。可 FPS 不等于反应快。论文举了个例子，一次性生成 30 帧，花一秒，FPS 是 30，但你按下按键，还是要等整整一秒才看到反应。

人机交互的研究发现，人能察觉的延迟大约是 50 到 60 毫秒。所以作者的目标是，从收到新输入，到出现反映它的第一帧，不超过 50 毫秒。对应的指标叫每秒响应次数，RPS，50 毫秒就是每秒 20 次。

这张图横轴是延迟，纵轴是 VBench 画质分。左边绿色这块是 50 毫秒以内的交互区，论文测的模型里，只有 FastFrames 落在里面。

### 以前的方法

以前提速主要靠蒸馏和少步采样，把要去噪很多步的模型，压成一两步。论文拿来拆解的最快模型，已经只用两步了。

还有两类工作。一类做更轻的 VAE 解码器，论文说最多提速 4 倍，但画质下降。另一类是批处理和流水线并行，提高的是吞吐量，不降低每一帧的延迟。

作者先做了件很朴素的事，测时间。他们拆的是当时最快的少步自回归模型 Causal Forcing++，基于 13 亿参数的 Wan 2.1。在一张 B200 上，更新 KV 缓存要 41.6 毫秒，两步去噪要 83.2 毫秒，VAE 解码要 147 毫秒，一帧加起来 271.8 毫秒。

也就是说，真正做扩散采样的时间只占百分之三十左右。VAE 解码一次，相当于跑 3.5 次主干网络。而把去噪从两步减到一步，延迟只少百分之十五，画质却明显变差。

### 核心想法

所以核心观察是，瓶颈已经不在去噪，而在那些被当成固定实现细节的环节，主要是 VAE 解码和 KV 缓存更新。作者就把这些环节换成轻量的替代品。

具体是三步，换掉 VAE 解码器，换掉专门刷新 KV 缓存的那次完整计算，再做系统层面的优化。图里下面这一行，就是改完之后的流程。

### 方法

先看 VAE 解码器，它负责把压缩后的潜在表示还原成像素。它只有 7300 万参数，但要在空间和时间上反复放大，每层都做密集的三维卷积，占了整条链路一半以上的时间，单它一个就超出预算 3 倍。

作者的问题是，主干网络 DiT 的特征已经很丰富了，还需要这么重的解码器吗。他们先试了最极端的，只用一个线性层，结果画面糊，但能认出场景。

最后的方案是三个因果的三维 ConvNeXt 小模块，都在低分辨率的 token 网格上算，再用一个线性层，把每个 token 展开成 4 帧、16 乘 16 的像素块。参数量 6390 万，和原来差不多，但解码只要 1 毫秒，原来是 147 毫秒。图里右边是新解码器，头发、雨滴这些细节都还在。

训练时主干网络冻结，新解码器去学原来 VAE 解出的画面，对齐像素，也对齐帧间变化，运动区域权重更高。

解码器变轻，训练显存也少得多，论文说更新原来的 VAE 大约要 68 GB，新解码器大约 5 GB。这样就能直接在像素空间加对抗训练，让画面更锐利，判别器用冻结的 DINOv3，数据是他们内部的 1 万段真实视频。

代价是，原来的 Wan VAE 能在同一家族的模型间共用，新解码器每换一个模型都要重新微调。

第二步是 KV 缓存，可以理解成模型对前面画面的记忆。最后一步去噪写进缓存的内容，是从带噪输入算出来的，所以原来的做法，是在干净结果上再完整跑一遍主干网络，只为重写缓存。在两步去噪里，这一遍占了全部主干计算的三分之一。

能不能直接不刷新。作者试了，去掉之后，画面里的物体会随时间变样，就算专门训练模型适应带噪缓存，还是会出瑕疵，比如人脸变了，手指和衣服出问题。

他们的办法是训练一个小网络来修缓存，用主干冻结的 patch 嵌入层，加四个轻量 Transformer 模块，每一层配一个修复头，预测对带噪缓存的修正量。

论文报告，它的效果和原来那遍完整计算相当，13 亿模型上大约 2 毫秒，140 亿模型上大约 6 毫秒，分别快了大约 20 倍和 12 倍。

第三步是系统优化，比如能提前算的都提前算好。逐帧生成时查询 token 太少，GPU 吃不满，就把 KV 序列切成五段并行算，延迟降了百分之十三。再加上 CUDA graph，13 亿模型每次主干计算从 41.6 毫秒降到 16.4 毫秒。

用 8 张 B200 时，他们试了几种并行方式，只有上下文并行划算，13 亿模型的主干计算再降到 5.8 毫秒，140 亿模型从 75.6 降到 20.3 毫秒。

三步合起来，就是图里这一行，单卡一帧大约 35.8 毫秒，比原来快了 7.6 倍。

### 实验和效果

实验设置。两个模型都基于 Wan 2.1。FastFrames-S 是 13 亿参数，直接从 Causal Forcing++ 的模型出发，主干冻结，只训练新解码器和缓存修复网络。FastFrames-L 是 140 亿参数，按 Causal Forcing++ 的流程从基础模型转换和蒸馏。所有模型用同样的提示词和随机种子，延迟取缓存填满后的稳定状态。

结果在表 1。论文测的基线里，没有一个达到 50 毫秒，最快的 Causal Forcing++ 是 272 毫秒，离目标差了 5.4 倍。

FastFrames-S 单卡 36 毫秒，每秒 27.8 次响应。VBench 总分 84.23，Causal Forcing++ 是 84.11，画质分项 85.30 对 85.09。

两者主干权重一样，作者把这点画质提升，归因于像素空间里的对抗训练。

FastFrames-L 在 8 张卡上是 45 毫秒，总分 84.44。同样 140 亿参数的 Krea-Realtime，单卡 1675 毫秒，总分 84.42。FastFrames-L 单卡是 158 毫秒，小模型用 8 张卡能到 14 毫秒。

这张图是画面对比。论文说，小模型清晰度和 Causal Forcing++ 相当，大模型在写实、折纸龙的构图、stay curious 这几个字的清晰度上，和 Krea-Realtime 相当，光影和文字也比小模型好。

作者还把方法用到两个现成的可控模型上，镜头控制的 LingBot，和按物理条件生成的 RealWonder。LingBot 13 亿版本，官方最快是两卡 842 毫秒，用上 FastFrames，8 卡降到 52 毫秒，快 16 倍。RealWonder 从单卡 1141 毫秒降到 8 卡 51 毫秒，快 22 倍，都接近交互目标。

但 LingBot 140 亿版本，8 卡只从 1373 毫秒降到 233 毫秒，还差得远。因为这些模型去噪步数更多，一次生成好几帧，还要额外处理控制条件。

### 和以前方法的对比

放大来看，表 1 里 VBench 总分最高的，其实是双向生成模型 Wan2.2，总分 84.65，但生成一段要 117734 毫秒，将近两分钟。FastFrames 不是总分最高，而是在交互级延迟下，画质和同类最好的相当。

还要看比较条件。小模型 36 对 272 毫秒，都是单卡，比较公平。大模型的 45 毫秒用了 8 张卡，而 Krea-Realtime 的 1675 毫秒是单卡，同样单卡比是 158 对 1675，论文说的是快 10.6 倍。

我的判断是，这篇最有价值的不是某个新模块，而是方法论，先把整条链路的时间测清楚，再砍真正的大头。用每秒响应次数代替 FPS，也提醒大家，实时要看是从按键算起，还是平均算出来的。

我的判断是，画质这边的证据比延迟弱。和基线比，VBench 总分只差零点几，小模型的语义分 79.92 还略低于 Causal Forcing++ 的 80.20，论文也没有报告用户主观评测。而且核心数字都是在 B200 这种数据中心显卡上测的。

### 局限

局限有这么几条。第一是分辨率，目前都是 480p。到了 720p，小模型单卡要 127 毫秒，大模型 8 卡要 116 毫秒，都出了交互区。

第二是硬件。换成 H100 会慢大约百分之二十五到五十，能不能在终端设备上跑，还是开放问题。

第三，新解码器每个模型都要单独训练，论文里用了 32 张 B200，大约一周。而像 LingBot 这类可控模型，设计时没考虑延迟，用了 FastFrames 也只追上一部分。

### 未来方向

作者提到的方向有，不增加延迟地加入动作控制，提高分辨率，支持更轻的硬件，改善长时间的一致性和物理合理性。

论文说，视频、代码和模型权重都在项目主页上。

我的判断是，交互式世界模型要落地，接下来要看新的可控模型，会不会从训练阶段就把延迟当成设计目标，比如论文建议的，把镜头参数拼在通道上，而不是拼在序列上。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Apple and Cornell University. It's called FastFrames, and the subtitle is the goal: video generation at interactive latency.

Here's the headline result. The authors report that their one point three billion parameter model produces a frame in thirty-six milliseconds on a single B200 GPU, with quality matching a recent model that is nearly eight times slower.

### Problem

Some background first. Autoregressive video diffusion models don't generate a whole clip at once. They generate frame by frame, so at every step they can take a new input, like a steering command or a camera move. That makes them a natural fit for world simulation, games, and predictive models for robotics.

The problem is that "real time" in this field usually means frames per second. And frames per second isn't responsiveness. The paper's example: a model that generates a thirty-frame clip as one batch in one second hits thirty frames per second, but after you press a key, you still wait a full second to see anything happen.

Human-computer interaction studies put the threshold for noticing a delay at roughly fifty to sixty milliseconds. So the authors set their target at fifty milliseconds, measured from the moment a new input arrives to the first frame that shows its effect. They report it as responses per second, and fifty milliseconds works out to twenty responses per second.

This plot shows quality on the vertical axis, using the VBench score, against latency on the horizontal axis. The green band on the left is the interactive zone, under fifty milliseconds, and of the models tested, only FastFrames lands in it.

### Prior work

Most past speedups came from distillation and few-step sampling, squeezing a model that needed many denoising steps down to one or two. The fastest model the paper profiles already uses just two steps.

Two other lines of work exist. One builds lighter VAE decoders, which, according to the paper, gain up to four times the speed but lose visual quality. The other uses batching and pipeline parallelism, which raises throughput but doesn't cut the latency of any single frame.

So the authors started with something basic: a stopwatch. They profiled Causal Forcing++, the fastest few-step autoregressive model at the time, built on the one point three billion parameter Wan 2.1. On one B200, the KV cache update takes forty-one point six milliseconds, the two denoising steps take eighty-three point two, and VAE decoding takes one hundred forty-seven. That's two hundred seventy-one point eight milliseconds per frame.

In other words, only about thirty percent of the time goes to actual diffusion sampling. One VAE decode alone costs as much as three and a half passes of the main network. And cutting denoising from two steps to one saves only fifteen percent of the latency, while hurting quality badly.

### Key idea

That's the paper's core observation. The bottleneck is no longer denoising. It's the parts people treat as fixed implementation details, mainly VAE decoding and KV cache updates. The plan is to replace those parts with lightweight alternatives.

There are three moves. Replace the VAE decoder. Replace the full network pass that exists only to refresh the KV cache. And do a round of systems-level optimization. The bottom row of this figure is the pipeline after all three.

### Method

Start with the VAE decoder. The VAE is the piece that compresses video into a small latent representation and expands it back into pixels. Its decoder has only seventy-three million parameters, but it upsamples again and again in space and time, with dense 3D convolutions at every resolution. It takes more than half of the whole pipeline's time, and on its own it blows the fifty millisecond budget three times over.

The authors asked whether such a heavy decoder is needed at all, given how rich the main network's features already are. They first tried the extreme version: a single linear layer mapping features straight to pixels. The result was blurry, but you could still recognize the scene.

The final design is three small causal 3D ConvNeXt blocks, all running on the low-resolution token grid, followed by one linear layer that unfolds each token into a patch of four frames, sixteen by sixteen pixels. At sixty-three point nine million parameters it's about the same size as the VAE, but it decodes in one millisecond instead of one hundred forty-seven. In this figure, the original VAE is on the left, the single linear layer in the middle, and their decoder on the right, keeping fine detail like hair and raindrops.

During training the main network stays frozen, and the new decoder learns to reproduce the original VAE's output, matching each pixel, matching changes from frame to frame, and putting extra weight on regions that move.

A lighter decoder is also much cheaper to train. For the same eighty-one frame video, the paper says updating the VAE decoder takes about sixty-eight gigabytes of GPU memory, against about five for the new one. That makes it practical to add adversarial training directly in pixel space for sharper output, using a frozen DINOv3 discriminator and an internal set of ten thousand real videos.

The cost: the original Wan VAE can be shared across models in the same family, but this decoder has to be fine-tuned for every model.

The second move is the KV cache. Think of it as the model's memory of earlier frames, which later frames attend to. At the last denoising step, the cache entries are computed from a noisy input. So the baselines run the whole network one more time on the clean result, just to rewrite the cache, and throw the output away. With two denoising steps, that extra pass is a third of all the network evaluations.

Can you just skip it? The authors tried. Drop it, and objects in the scene change over time. Even training the model to live with the noisy cache leaves visible artifacts: a face that changes, broken fingers and clothing.

Their fix is a small network that repairs the cache. It reuses the main network's frozen patch embedding, adds four lightweight transformer blocks, and gives each layer a repair head that predicts a correction to its noisy keys and values.

The paper reports it performs about as well as the full pass, while taking roughly two milliseconds on the one point three billion model and six on the fourteen billion model. That's around twenty and twelve times faster.

The third move is systems work. Cache slots are allocated once and written in place, position encodings and conditions are precomputed, and the three attention projections are fused into one matrix multiply. Frame-by-frame generation has too few query tokens to keep the GPU busy, so they split the key-value sequence into five segments processed in parallel, which alone cut latency by thirteen percent. Add CUDA graphs to trim launch overhead, and each network pass on the small model drops from forty-one point six to sixteen point four milliseconds.

On eight B200s, they tried several kinds of parallelism, and only context parallelism paid for its communication cost. That brings a network pass from sixteen point four down to five point eight milliseconds on the small model, and from seventy-five point six to twenty point three on the large one.

Put the three moves together and you get this row: about thirty-five point eight milliseconds per frame on one GPU, seven point six times faster than the original two hundred seventy-one point eight.

### Results

The setup. Both models are built on Wan 2.1. FastFrames-S has one point three billion parameters and starts from the Causal Forcing++ checkpoint, with the main network frozen and only the new decoder and cache-repair network trained. FastFrames-L has fourteen billion parameters, and the authors converted and distilled it from the base model following the Causal Forcing++ recipe. Every model gets the same VBench prompts and seeds on a B200, and latency is measured at steady state with a full cache.

The results are in Table 1. None of the baselines tested reaches fifty milliseconds. The fastest, Causal Forcing++, takes two hundred seventy-two milliseconds, or three point six eight responses per second, five point four times short of the target.

FastFrames-S takes thirty-six milliseconds on one B200, about twenty-eight responses per second. Its VBench total is eighty-four point two three, against eighty-four point one one for Causal Forcing++, and its quality sub-score is eighty-five point three zero against eighty-five point zero nine.

Those two models share the same main network weights, and the authors attribute the small quality gain to the adversarial training in pixel space.

The large model, FastFrames-L, hits forty-five milliseconds on eight GPUs, with a VBench total of eighty-four point four four. Krea-Realtime, also fourteen billion parameters, takes one thousand six hundred seventy-five milliseconds on one GPU and scores eighty-four point four two. FastFrames-L on a single GPU takes one hundred fifty-eight milliseconds, and the small model on eight GPUs gets down to fourteen.

This figure compares frames side by side. According to the paper, the small model is about as sharp as Causal Forcing++, and the large model matches Krea-Realtime on realism, on composing a paper origami dragon, and on rendering the words "stay curious" legibly. The large model also beats the small one on lighting and text.

The authors also applied the method to two existing controllable models: LingBot, which is steered by camera moves, and RealWonder, which is conditioned on physics. LingBot's one point three billion version goes from eight hundred forty-two milliseconds, its fastest official setup on two GPUs, to fifty-two on eight GPUs, sixteen times faster. RealWonder goes from one thousand one hundred forty-one milliseconds on one GPU to fifty-one on eight, twenty-two times faster. Both roughly reach the interactive target.

The fourteen billion LingBot, though, only drops from one thousand three hundred seventy-three to two hundred thirty-three milliseconds on eight GPUs, still far off. These models need more denoising steps, generate several frames per step, and spend extra compute on their control inputs.

### Comparison

Zoom out, and the best VBench total in Table 1 actually belongs to a bidirectional model, Wan2.2, at eighty-four point six five. But it takes one hundred seventeen thousand seven hundred thirty-four milliseconds per clip, nearly two minutes. FastFrames doesn't have the top total score. Its claim is keeping quality at the level of the best comparable models while running at interactive latency.

Watch the comparison conditions too. Thirty-six versus two hundred seventy-two milliseconds for the small model is single GPU against single GPU, so that's fair. The large model's forty-five milliseconds uses eight GPUs, while Krea-Realtime's number is one GPU. GPU for GPU, it's one hundred fifty-eight against one thousand six hundred seventy-five, which the paper describes as ten point six times faster.

My read is that the most valuable thing here isn't any single module. It's the method: time the whole critical path first, then cut where the time actually goes. Swapping frames per second for responses per second is a useful reminder to ask whether "real time" is counted from the keypress or averaged over a batch.

My read is also that the quality evidence is weaker than the latency evidence. Against the baselines, VBench totals differ by tenths of a point, the small model's semantic score of seventy-nine point nine two is slightly below Causal Forcing++ at eighty point two zero, and the paper reports no user study. And every headline number comes from B200s, which are data-center GPUs.

### Limitations

There are several limitations. First, resolution: everything is 480p. At 720p, the small model takes one hundred twenty-seven milliseconds on one B200, and the large model one hundred sixteen on eight, both outside the interactive zone.

Second, hardware. The tests use B200s. On H100s it runs about twenty-five to fifty percent slower, and running on consumer devices is still an open question.

Third, the new decoder has to be trained for every model, and in the paper that took thirty-two B200s for about a week. And controllable models like LingBot weren't designed with latency in mind, so FastFrames only partly closes their gap.

### Future work

The authors' future directions: adding action control without adding latency, higher resolutions, lighter hardware, and better long-term consistency and physical plausibility.

The paper says videos, code and model weights are on the project page.

My read is that if interactive world models are going to be real products, the thing to watch is whether new controllable models treat latency as a design goal from training onward, for example by feeding camera poses in along the channel dimension instead of the sequence, as the paper suggests.

That's today's Daily Frontier Papers. The paper is linked in the description.

