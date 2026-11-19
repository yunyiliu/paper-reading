# FastFrames: Video Generation at Interactive Latency

**按键就有画面每帧36毫秒**

- 论文 Paper: [arXiv 2611.23226](https://arxiv.org/abs/2611.23226)
- 单位 Affiliation: Apple · 康奈尔大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-19
- 视频 Videos: —

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇苹果的论文，第一作者在苹果实习，来自康奈尔大学，论文叫 FastFrames，题目是，视频生成做到可交互的延迟。

它要回答的问题很具体，AI 一帧一帧往后生成视频，能不能快到你按一下键，画面几乎马上就跟着变。

### 问题

先说背景。自回归视频扩散模型，就是一段一段往后生成视频，每生成一段，都可以接收新的输入，比如方向键或者镜头控制。所以它被看作做交互体验的好材料，比如世界模拟、游戏，还有机器人的预测模型。

但作者指出，近来有些工作报的是每秒帧数，也就是 FPS，这个数说明不了能不能实时响应。论文举的例子是，一次批量生成 30 帧，用了一秒，FPS 是 30，可用户还是要等一秒，才看到反应。

所以他们换了一个指标，从新输入到达，到第一帧反映它的画面出现，中间的延迟，再换算成每秒响应次数，叫 RPS。人机交互研究报告过，人能察觉的延迟，大约在 50 到 60 毫秒，作者把目标定在 50 毫秒以内，也就是每秒 20 次响应。

这张图横轴是延迟，纵轴是 VBench 画质总分，越靠左上越好。论文的结论是，现有方法里，没有一个进到 50 毫秒以内的区域。

### 以前的方法

以前提速，主要靠蒸馏和少步采样，把去噪的步数压到很少。也有人专门压缩 KV cache，或者做更小更快的 VAE 解码器，但论文说，这类小解码器最多快 4 倍左右，画质会下降。

作者先做了一件事，把当时最快的少步自回归模型，Causal Forcing++，推理过程拆开计时。它基于 Wan 2.1 的 1.3B 模型，在一块 B200 GPU 上，生成一帧要 271.8 毫秒。

拆开看，两步扩散采样，每步 41.6 毫秒。另外还要再跑一次完整的 DiT，只为了刷新 KV cache，又是 41.6 毫秒。最后 VAE 解码要 147 毫秒，一项就占了一半以上。论文说，真正花在扩散采样上的时间，只占大约 30%。

### 核心想法

所以这篇的核心想法是，瓶颈已经不在去噪步数，而在那些平时被当成固定实现细节的部件，比如 VAE 解码和 KV cache 更新。作者的做法是，质疑这些部件是不是非要不可，然后换成轻量的替代品。

先解释两个词。VAE 解码器，负责把压缩后的潜空间表示，还原成像素画面。KV cache 是模型对前面帧的记忆，后面的帧，通过它回看前面的内容。

### 方法

第一步，换掉 VAE 解码器。作者先试了最极端的办法，只用一个线性层，直接把 DiT 的特征映射成像素。结果画面是糊的，但看得出是什么场景，说明渲染像素，不一定需要很贵的解码器。

最后的方案，是三个因果的 3D ConvNeXt 风格小模块，全部在 DiT 原本的低分辨率 token 网格上计算，最后用一个线性层，把每个 token 变成一小块像素。参数量差不多，6390 万对 7330 万，但耗时从 147 毫秒，降到 1 毫秒左右。

训练分两步。先冻结 DiT，让新解码器去模仿原来 VAE 解出来的画面。因为新解码器很轻，训练时显存大约 5 GB，原来的 VAE 要 68 GB 左右，所以可以直接在像素空间加对抗训练，用冻结的 DINOv3 做判别器的骨干，让画面更锐利、更真实。

第二步，加速 KV cache 更新。原来的问题是，最后一步去噪时，写进缓存的键和值，是从带噪声的输入算出来的。所以基线要再跑一遍完整的 DiT，只为把缓存修干净，这一遍的输出直接丢掉。

作者试过直接去掉这一遍，画面会漂移、出伪影，让模型学着用带噪的缓存，也还是有明显伪影。于是他们训练了一个小网络，一个共享的特征提取器，加上每层一个修复头，预测对带噪键值的残差修正。在 1.3B 模型上大约 2 毫秒，比原来那遍 DiT 快大约 20 倍，论文报告效果和原来相当。

第三步是系统层面的优化。复用固定的缓存缓冲区，预先算好位置编码和条件，把 QKV 投影合成一次矩阵乘法，把很长的 KV 序列切成几段并行算注意力，再用 CUDA graph 减少调度开销。这些加起来，单次 DiT 从 41.6 毫秒降到 16.4 毫秒。

再往上，用 8 块 B200 做上下文并行，把查询 token 分到各张卡上。1.3B 模型的单次 DiT，从 16.4 降到 5.8 毫秒，14B 模型从 75.6 降到 20.3 毫秒。

### 实验和效果

模型有两个尺寸，都基于 Wan 2.1。小的 FastFrames-S 是 1.3B，从 Causal Forcing++ 的检查点出发，DiT 冻结，只训练新的解码器和 KV 修复网络。大的 FastFrames-L 是 14B，按 Causal Forcing++ 的训练配方，从双向的基础模型转成自回归，再做蒸馏。两个都是两步采样，每次生成一个潜空间帧。

结果在表 1。所有方法用一样的 VBench 提示词和随机种子，分辨率都在 480p 左右，自回归模型测的是缓存满了之后的稳态延迟。FastFrames-S 在一块 B200 上，每帧 36 毫秒，大约每秒 28 次响应，VBench 总分 84.23。同样是 1.3B 的 Causal Forcing++，是 272 毫秒，总分 84.11。

FastFrames-L 单卡是 158 毫秒，放到 8 卡节点上是 45 毫秒，总分 84.44。作者在图 4 的对比里说，14B 比 1.3B 的光影更真实，文字也更清楚。

作者还把这套方法，套到别人现成的交互模型上。镜头控制模型 LingBot 1.3B，官方最快的配置，用 2 张卡，是 842 毫秒，用上 FastFrames 的方法，8 张卡上是 52 毫秒，快了 16 倍。物理条件控制的 RealWonder，从官方单卡的 1141 毫秒，降到 8 卡的 51 毫秒，快了 22 倍。论文说，这两个都大致到了 50 毫秒的目标。

### 和以前方法的对比

横向比，表 1 里最快的基线 Causal Forcing++，每秒 3.68 次响应，离每秒 20 次的目标差 5.4 倍。同为 14B 的 Krea-Realtime，单卡是 1675 毫秒，FastFrames-L 单卡是 158 毫秒。

画质上，FastFrames-S 和 Causal Forcing++ 用的是同一套 DiT 权重，VBench 总分略高，作者把这归因于像素空间的对抗训练。不过在 VBench 的语义分上，FastFrames-S 是 79.92，比 Causal Forcing++ 的 80.20 略低。

作者还指出一个测量问题。比如 Self-Forcing 报的延迟，是对整段序列取平均，早期缓存没满的帧更快，所以不代表稳态。这篇的数字，都是在缓存满的稳态下测的，而且给了分项。

我的判断是，这篇最有价值的，不是某一个新模块，而是那张拆开计时的图。它说明少步蒸馏之后，瓶颈已经转移到解码和缓存这些部件上，只盯着去噪步数，提速空间有限。

我的判断是，也要看清代价。FastFrames-L 的 45 毫秒，要一整台 8 卡 B200 的节点。表 1 里，FastFrames 和同骨干基线的 VBench 总分，只差 0.02 到 0.12 分，单凭这个分数，很难说清画质的差别，还要看视频本身。

### 局限

局限，作者自己写得很清楚。第一，主要实验都在 480p。到 720p，FastFrames-S 单卡每帧要 127 毫秒，FastFrames-L 用 8 卡也要 116 毫秒，都超出了 50 毫秒的交互区间。

第二，测量用的是 B200 这种高端 GPU，换成 H100，大约慢 25% 到 50%。作者把在设备端运行，列为一个令人期待的可能，现在还做不到。

第三，新解码器不能像 Wan 的 VAE 那样，在同一家族的不同模型之间共用，每个模型都得单独微调。训练也不便宜，像素解码器的重建训练，用了 32 块 B200，大约一周。

### 未来方向

未来方向，作者提了几件事，在不增加延迟的前提下加入动作条件，比如把镜头位姿拼在通道维度，而不是序列维度，还有提高分辨率，支持更轻的硬件，以及改进长时间的一致性和物理合理性。

对 LingBot 和 RealWonder 这类模型，作者认为，还应该把它们蒸馏到更少的采样步数，每次生成更少的帧。论文说，项目主页提供视频、代码和模型权重。

我的判断是，这篇把交互式视频生成，从能不能做，推进到了在服务器上能实时做。下一步要看的是，同样的思路，能不能让 720p 和单卡也进到 50 毫秒以内。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Apple, with the first author, an Apple intern, also at Cornell. It's called FastFrames, and the subtitle is the whole pitch: video generation at interactive latency.

The question is simple to state. When an AI model generates video frame by frame, can it react fast enough that you press a key and the picture changes almost immediately?

### Problem

Some background. Autoregressive video diffusion models generate a video chunk by chunk, and each new chunk can take fresh input, like a steering command or a camera move. That makes them a natural fit for interactive things: world simulation, games, and predictive models for robots.

But the authors point out that some recent work reports real-time speed in frames per second, and that number doesn't tell you whether a model can respond in real time. Their example: a model that produces a thirty-frame clip in one batch, taking one second, scores thirty frames per second, yet the user still waits a full second to see any reaction.

So they measure something else: the delay from a new input arriving to the first frame that shows its effect, and they report its inverse as responses per second. Human-computer interaction studies put the threshold where people notice delay at roughly fifty to sixty milliseconds. The authors aim for fifty milliseconds or less, which is twenty responses per second.

This figure plots latency against VBench quality score, with the upper left being best. The paper's point is that no existing method lands inside the fifty-millisecond zone.

### Prior work

Earlier speedups came mostly from distillation and few-step sampling, cutting the number of denoising steps to just a handful. Others compress the KV cache or build smaller, faster VAE decoders, but according to the paper those small decoders top out at about four times faster and lose visual quality.

The authors started by timing every stage of the fastest few-step autoregressive model at the time of writing, Causal Forcing plus plus. It's built on the one point three billion parameter Wan two point one backbone, and on a single B200 GPU it takes two hundred seventy-one point eight milliseconds per frame.

Break that down. Two diffusion steps at forty-one point six milliseconds each. Then one more full pass through the diffusion transformer, the DiT, just to refresh the KV cache: another forty-one point six. Then the VAE decoder takes one hundred forty-seven milliseconds, more than half the total on its own. Only about thirty percent of the time is actually spent on diffusion sampling.

### Key idea

So the key idea is that the bottleneck is no longer the number of denoising steps. It's the parts usually treated as fixed implementation details, like VAE decoding and KV cache updates. The authors question whether those parts are really necessary, and swap them for lightweight replacements.

Two quick definitions. The VAE decoder turns the compressed latent representation back into pixels. The KV cache is the model's memory of earlier frames; each new frame looks back through it.

### Method

Step one: replace the VAE decoder. They first tried the most extreme option, a single linear layer mapping DiT features straight to pixels. The output was blurry but recognizable, which suggests rendering pixels doesn't need an expensive decoder.

The final design is three small causal 3D ConvNeXt-style blocks that all run on the DiT's own low-resolution token grid, followed by one linear layer that turns each token into a small patch of pixels. The parameter count is similar, sixty-three point nine million versus seventy-three point three million, but decoding drops from one hundred forty-seven milliseconds to about one.

Training happens in two stages. First, with the DiT frozen, the new decoder learns to match what the original VAE produces. Because the decoder is so light, training it needs about five gigabytes of GPU memory instead of roughly sixty-eight for the VAE, so they can add adversarial training directly in pixel space, with a frozen DINOv3 backbone inside the discriminator, to make frames sharper and more realistic.

Step two: speed up the KV cache update. The issue is that during the last denoising step, the keys and values written to the cache are computed from a noisy input. So the baselines run one more full DiT pass just to clean up the cache, and throw that pass's output away.

Simply dropping that pass makes the video drift and pick up artifacts, and training the model to live with the noisy cache still leaves visible artifacts. Instead, they train a small network: a shared feature extractor plus one repair head per layer, predicting residual corrections to the noisy keys and values. On the one point three billion model it takes about two milliseconds, around twenty times faster than the DiT pass it replaces, and the paper reports comparable results.

Step three is systems work. Persistent cache buffers, precomputed position tables and conditioning, the query, key and value projections fused into one matrix multiply, the long KV sequence split into segments processed in parallel, and CUDA graphs to cut launch overhead. Together, these bring a single DiT evaluation from forty-one point six milliseconds down to sixteen point four.

On top of that, they spread the work over eight B200 GPUs with context parallelism, splitting the query tokens across cards. A DiT evaluation drops from sixteen point four to five point eight milliseconds for the small model, and from seventy-five point six to twenty point three for the fourteen billion model.

### Results

There are two sizes, both on Wan two point one. FastFrames-S, at one point three billion parameters, starts from the Causal Forcing plus plus checkpoint; the DiT stays frozen and only the new decoder and KV repair network are trained. FastFrames-L, at fourteen billion, follows the Causal Forcing plus plus recipe: convert the bidirectional base model to autoregressive, then distill it. Both use two sampling steps and generate one latent frame at a time.

The results are in Table one. Every method uses the same VBench prompts and seeds at roughly four-eighty p, and for the autoregressive models latency is measured at steady state with a full cache. FastFrames-S runs at thirty-six milliseconds per frame on one B200, about twenty-eight responses per second, with a VBench total of eighty-four point two three. Causal Forcing plus plus, same size, takes two hundred seventy-two milliseconds and scores eighty-four point one one.

FastFrames-L takes one hundred fifty-eight milliseconds on a single GPU and forty-five on an eight-GPU node, with a total of eighty-four point four four. In the side-by-side comparisons of Figure four, the authors say the fourteen billion models give more realistic lighting and more legible text than the small ones.

They also applied the same techniques to other teams' interactive models. For the camera-controlled LingBot one point three billion, the fastest official setup, on two GPUs, takes eight hundred forty-two milliseconds; with FastFrames' methods on eight GPUs it's fifty-two, sixteen times faster. The physics-conditioned RealWonder goes from one thousand one hundred forty-one milliseconds on one GPU to fifty-one on eight, twenty-two times faster. The paper says both roughly reach the fifty-millisecond target.

### Comparison

Against the field: the fastest baseline in Table one, Causal Forcing plus plus, manages three point six eight responses per second, five point four times short of the twenty target. Krea-Realtime, also fourteen billion parameters, takes one thousand six hundred seventy-five milliseconds on one GPU, against one hundred fifty-eight for FastFrames-L.

On quality, FastFrames-S uses the very same DiT weights as Causal Forcing plus plus and scores slightly higher on the VBench total, which the authors credit to the adversarial training in pixel space. On the VBench semantic score, though, FastFrames-S is slightly lower, seventy-nine point nine two against eighty point two.

The authors also flag a measurement problem. Self-Forcing, for example, reports latency averaged over a whole sequence, where early frames are cheaper because the cache isn't full yet, so it doesn't reflect steady state. Every number in this paper is measured at steady state with a full cache, broken down by operation.

My read is that the most valuable thing here isn't any single new module, it's that timing breakdown. It shows that after few-step distillation, the bottleneck has moved to decoding and caching, so work that only cuts denoising steps has limited room left.

My read is also that the cost should be clear. FastFrames-L's forty-five milliseconds needs a full node of eight B200s. And in Table one, FastFrames and its same-backbone baselines differ by only zero point zero two to zero point one two points on VBench total, so that score alone can't settle quality differences; you have to watch the videos.

### Limitations

The authors are upfront about limitations. First, the main experiments are at four-eighty p. At seven-twenty p, FastFrames-S needs one hundred twenty-seven milliseconds per frame on one GPU, and FastFrames-L needs one hundred sixteen even on eight, both outside the interactive range.

Second, everything is measured on high-end B200 GPUs; on H100s it's roughly twenty-five to fifty percent slower. The authors call running on-device an exciting possibility, which is to say it's not there yet.

Third, unlike Wan's VAE, which can be shared across models of the same family, the new decoder has to be fine-tuned for each model. And training isn't cheap: the reconstruction stage of the pixel decoder ran on thirty-two B200s for about a week.

### Future work

For future work, the authors list adding action conditioning without raising latency, for instance by concatenating camera poses along the channel dimension instead of the sequence dimension, along with higher resolution, lighter hardware, and better long-term consistency and physical plausibility.

For models like LingBot and RealWonder, they argue for distilling to fewer sampling steps and fewer frames per generation. The paper says videos, code and model weights are on the project page.

My read is that this paper moves interactive video generation from "can it be done" to "it runs in real time on a server". The next thing to watch is whether the same approach gets seven-twenty p, or a single GPU, under fifty milliseconds.

That's today's Daily Frontier Papers. The paper is linked in the description.

