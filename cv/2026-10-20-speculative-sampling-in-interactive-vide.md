# Speculative Sampling in Interactive Video Generation

**改指令不用等整块播完**

- 论文 Paper: [arXiv 2610.20556](https://arxiv.org/abs/2610.20556)
- 单位 Affiliation: 中国科学技术大学 · 香港中文大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-20
- 视频 Videos: [YouTube 中文](https://youtu.be/5fn3oLl9CVU) · [YouTube English](https://youtu.be/mr5Eod_bN6E) · [B站](https://www.bilibili.com/video/BV1m4nc6KEde)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇中国科学技术大学的论文，合作方有香港中文大学，题目叫 Speculative Sampling in Interactive Video Generation，也就是交互式视频生成里的推测采样。

它讨论的是一个很实际的问题，视频一边在播，用户一边在改指令，怎么让画面尽快跟上，同时又不牺牲画质。

### 问题

先说什么是交互式视频生成。比如一个可以操控的视频世界，你随时改镜头方向或者动作指令，已经生成的视频还在继续播，新指令要尽快体现在画面上。

作者强调，光生成得快还不够。模型再快，如果新内容只能在某个边界才能接进播放，用户还是得等。

论文把响应延迟拆成三段。指令到了，先等生成开始，这段叫 R，然后是生成本身花的时间，叫 C，生成好了，还要等它真正上屏，叫 Q，三段加起来就是延迟 L。

这张图画的就是这个过程。新指令到达以后，要先等下一次生成开始，生成完，还要等到下一个播放边界，才能被看到。

### 以前的方法

现在常见的做法是分块生成。一块叫一个 chunk，块内部的几帧一起生成，前后可以互相参考，这叫双向建模，块和块之间，再一块接一块往后续写。

为了降低延迟，大家倾向于把 chunk 做小，甚至一次只生成一帧。这几年 rollout 训练和少步蒸馏，让这条路越来越实用。

道理很简单。如果新内容只能在块的边界接进播放，论文推导出，平均等待至少是半个块的时长。块越短，这个下限就越低。

但小块也有代价。块小了，一次能一起建模的帧就少，GPU 的并行能力也用不满。所以作者问了一个核心问题，低延迟交互，是不是一定要用更小的双向建模窗口。

### 核心想法

作者的回答是，不一定。关键是把两件事分开，一件是模型一次一起预测多少帧，另一件是系统一次确定交付多少帧。

看这张图。第一行是小块，一帧一帧生成，新指令来了很快就能切换。第二行是大块，要等整块播完才能换。第三行是推测采样，块还是大的，但除了受保护的一小段，后面的未来都可以随时替换。

名字里的推测，指的就是这个。模型提前把后面几帧也算出来，放进缓存，指令不变就直接用，指令一变，还没交付的部分就作废重算。

### 方法

具体做法是这样。先解释一下，潜变量就是视频压缩之后的表示，模型在潜变量上生成，再解码成画面。每次采样，在下一个要交付的潜变量后面，补上 K 减 1 个噪声潜变量当填充，凑成一个 K 个潜变量的双向窗口，一起去噪。

填充位置算出来的预测不扔掉，存进预测缓存，同时记下它们是在哪个指令版本下生成的。交付的粒度是一个潜变量，每次只确定很短的一小段。

调度交给一个只有两个状态的有限状态机。一个叫 Produce，表示有生成任务在跑，一个叫 Consume，表示没有任务，直接消费缓存里的预测，两种状态下，播放都不停。

新指令到了怎么办。控制器在受保护的前缀之后，找最早一个可行的分叉点，锁住前面，把后面还能替换的部分，换成一个新的完整大块。

可行有几个条件，其中一个是，分叉点之前已有的内容，要够一直播到新内容算出来。如果生成任务正在跑的时候又来了新指令，先记下来，多条指令只保留最新的一条。

还有一个工程细节，让原生的双向视频模型也能直接拿来做流式。VAE 是在像素和潜变量之间来回转换的编码器，很多 3D VAE 会在时间上压缩，而接续接口要的是没有压缩的首帧，所以作者把边界那一帧先解码成像素，再走 VAE 的首帧通道重新编码。

理论分析给了两个结论。第一，上屏前的等待不再受块长度限制，只和交付粒度有关，也就是不到一个潜变量的时长。第二，延迟的大头变成了生成时间 C，生成越快，新指令碰上正在跑的任务时，要多等的时间也越少。

代价是，指令一变，没用上的预测就白算了。论文的分析是，没有指令变化时，采样次数和普通分块一样，有变化时，额外的计算量取决于最后被丢掉的比例。

### 实验和效果

实验分两部分。第一部分看块大小本身，用 Wan 1.3B 模型，训练了块大小为 1、3、7、21 的几个版本，在视频生成评测 VBench 上比画质。

结果在这张表里。4 步采样、480p 下，块大小从 1 变成 3，VBench 总分从 79.00 提到 81.66。不过提升不是单调的，最好的块大小要看具体设置。

速度方面，块大小从 1 到 3，当前块要算的潜变量变成 3 倍，但在 H200 上，单次前向时间 480p 只变成 2.05 倍，720p 是 2.74 倍。

多卡并行之后更明显。用 8 卡序列并行，480p 块大小 7 的单次前向是 78.6 毫秒，和单卡块大小 1 的 77.5 毫秒差不多。不过作者也说，卡多了不一定总有帮助。

第二部分是真实的在线服务测试。三个模型，Wan 1.3B、LTX-2.3 和 Sol-H3，每个都有小块和大块两种配置，分别用普通的按块提交，和推测采样来跑。

每组跑 30 条 60 秒的操作轨迹，分低频和高频两档，低频每条改 1 到 2 次指令，高频改 3 到 11 次，测的是从指令到达到画面显示的延迟。

结果在表 3。在主要的大小块对比里，推测采样都降低了延迟，而且块越大，省得越多。比如 Wan 的大块配置，在低频下，中位延迟从 3.23 秒降到 1.32 秒。

实测和理论也对得上。在 Sol-H3 上，小块和大块的平均延迟分别减少 0.493 秒和 2.814 秒，理论预测是 0.438 秒和 2.562 秒。

用了推测采样以后，大块配置的中位延迟，只比小块高 0.26 到 0.39 秒。但大块的 GPU 占用低得多，Wan 是百分之 17 到 20，小块是百分之 50 到 52，LTX 两者差不多。作者认为，空出来的算力可以用来服务更多的并发视频流。

最亮眼的是 Sol-H3。大块加推测采样，低频和高频下的中位延迟是 1.42 秒和 1.41 秒，反而比小块的普通做法 1.87 秒还低，而且没有播放卡顿。

### 和以前方法的对比

和相关工作比，论文自己点了区别。比如 NFD 也做多帧的推测采样，但窗口内部仍然是因果的，用不上双向建模。这篇保留的是块内双向的联合生成，只是把提交变细。

另一条常见的路，是把双向模型改造成因果的流式模型，但论文提到，直接改成因果生成，如果不仔细适配，可能带来画质损失。作者展示的是，大的双向模型不做 Self Forcing 那样的后训练，也能放进低延迟的流式系统里用。

我的判断是，这篇的价值主要在思路。它把块大小从延迟的约束里解放出来，变成画质和算力之间的一个可调参数，以后的优化重点，会更多放在让单块生成更快上。

我的判断是，证据也有明显的短板。在线测试只比了延迟、GPU 占用和卡顿，没有测替换之后视频本身的画质和连贯性，对照组也只是同一个模型的按块提交，没有和其他流式方法正面比较。

### 局限

作者写了两条局限。第一，预测只有在被新指令作废之前用上才划算，如果指令频繁甚至连续变化，每块的大部分都会被丢掉，浪费大量计算。

第二，块内是双向注意力，保留下来的 KV 缓存，可能已经带上了后来被丢掉那段未来的信息，删掉对应的缓存条目，也不一定能完全消除它对新内容的影响。

表 3 里也有不够好的地方。LTX 用 CUDA Graph 的小块配置，推测采样的中位延迟反而比基线高，卡顿比例也从接近零，升到百分之 1.8 和百分之 4.4。LTX 普通执行的小块，两种方法都有大约百分之 20 的时间在卡顿。

另外，LTX 的训练用的是内部数据集，画质评测是内部的 32 个样例，外部不好复现。

### 未来方向

作者在附录里提的下一步，是把 DiT 采样和 VAE 解码当成一条流水线一起优化，目标是缩短到第一组可交付画面的时间。

更大的方向，就是论文反复强调的，与其把块越做越小，不如想办法让高质量的大块生成更快，比如加速 DiT 采样，或者多卡并行。

我的判断是，对做世界模型、游戏和互动视频的团队来说，这主要是调度层的改动，不用为了降延迟专门去训更小块的模型，值得先试，但如果你的场景里指令几乎每秒都在变，就要先算清楚会浪费多少计算。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the University of Science and Technology of China, with The Chinese University of Hong Kong. It's called Speculative Sampling in Interactive Video Generation.

The problem is a practical one. The video keeps playing, the user keeps changing instructions, and you want the picture to react fast without giving up quality.

### Problem

First, what interactive video generation means. Think of a video world you can steer. You change the camera or an action command at any moment, the video already generated keeps playing, and the new command should show up on screen as soon as possible.

The authors' point is that raw speed isn't enough. However fast the model is, if new content can only join playback at certain boundaries, the viewer still waits.

So the paper splits response latency into three parts. A command arrives and waits for generation to start, that's R. Then generation itself takes time, that's C. Then the result waits to actually reach the screen, that's Q. Add them up and you get the latency, L.

This figure shows the timeline. After the new command arrives, it waits for the next generation to start, and once that's done, it still waits for the next playback boundary before anyone sees it.

### Prior work

The usual approach is chunked generation. Frames inside a chunk are generated together and can look at each other in both directions, which is called bidirectional modeling, and chunks are then chained one after another.

To cut latency, people tend to shrink the chunks, sometimes down to a single frame at a time. Recent work on rollout training and few-step distillation has made that route increasingly practical.

The logic is simple. If new content can only enter playback at chunk boundaries, the paper shows the average wait is at least half a chunk. Shorter chunks mean a lower floor.

But small chunks have a cost. Fewer frames get modeled jointly, and the GPU's parallelism goes underused. So the authors ask the central question: does low-latency interaction really require a smaller bidirectional window?

### Key idea

Their answer is no. The trick is to separate two things: how many frames the model predicts jointly, and how many frames the system commits to delivering at a time.

Look at this figure. The top row is small chunks, one frame at a time, so a new command switches quickly. The middle row is a big chunk, where you wait for the whole chunk to finish. The bottom row is speculative sampling. The chunk is still big, but beyond a short protected prefix, the whole future stays replaceable.

That's what "speculative" refers to here. The model computes the next few frames ahead of time and caches them. If the command doesn't change, they get used. If it does, whatever hasn't been delivered is thrown out and recomputed.

### Method

Here's how it works. A quick term first: latents are the compressed representation of the video, the model generates latents, and they're decoded into frames afterwards. On each sampling call, the next latent to be delivered gets K minus one noise latents appended as padding, forming a K-latent bidirectional window that is denoised together.

The predictions at those padding positions aren't discarded. They go into a prediction cache, tagged with the version of the command they were generated under. Delivery happens one latent at a time, so only a short piece is committed at each step.

Scheduling is handled by a finite-state machine with just two states. Produce means a generation job is running. Consume means no job is running and playback draws on the cache. Either way, playback never stops.

So what happens when a new command arrives? The controller finds the earliest feasible branch point after the protected prefix, locks everything before it, and replaces the replaceable part after it with a fresh, complete chunk.

Feasible has a few conditions. One is that the content already in place before the branch must last until the new content is ready. And if a command arrives while a job is running, it's held as pending, and several pending commands collapse into the latest one.

There's also an engineering detail that lets an off-the-shelf bidirectional video model stream. The VAE is the encoder that converts between pixels and latents. Many 3D VAEs compress along time, but the continuation interface expects an uncompressed first frame. So the authors decode the boundary frame to pixels and re-encode it through the VAE's first-frame path.

The analysis gives two conclusions. First, the wait before display no longer depends on chunk length, only on delivery granularity, so it's under one latent's duration. Second, the dominant cost becomes generation time C, and the faster generation is, the less extra time a command waits when it lands on a job that's already running.

The price is that when a command changes, the unused predictions were wasted work. The paper's accounting is that with no command changes, the number of sampling calls matches ordinary chunking, and with changes, the extra compute depends on the fraction of predictions that end up discarded.

### Results

The experiments come in two parts. The first looks at chunk size itself, using the Wan one point three B model trained with chunk sizes of one, three, seven and twenty-one, and scoring quality on the VBench video generation benchmark.

The results are in this table. With four-step sampling at four eighty p, going from chunk size one to three lifts the VBench total from seventy-nine point zero zero to eighty-one point six six. But the gains aren't monotonic, so the best chunk size depends on the setting.

On speed, going from chunk size one to three triples the latents in the current chunk, yet a single forward pass on an H200 only takes two point zero five times as long at four eighty p, and two point seven four times at seven twenty p.

With multiple GPUs it gets better. Using eight-way sequence parallelism, a chunk of seven at four eighty p takes seventy-eight point six milliseconds per forward pass, about the same as seventy-seven point five for a chunk of one on a single GPU. The authors also note that more GPUs don't always help.

The second part is a real online serving test. Three models, Wan one point three B, LTX two point three, and Sol-H3, each in a small-chunk and a large-chunk configuration, run with either ordinary chunk-level commitment or speculative sampling.

Each setup gets thirty sixty-second interaction traces at two levels: low frequency, with one to two command changes per trace, and high frequency, with three to eleven. What's measured is the delay from a command arriving to it appearing on screen.

The results are in Table 3. In the main short and long comparisons, speculative sampling lowers latency, and the bigger the chunk, the more it saves. For Wan's large-chunk setup at low frequency, median latency drops from three point two three seconds to one point three two.

The measurements also line up with the theory. On Sol-H3, mean latency falls by zero point four nine three seconds for small chunks and two point eight one four for large ones, against predicted values of zero point four three eight and two point five six two.

With speculative sampling, the large-chunk median latency is only zero point two six to zero point three nine seconds above the small-chunk one. But GPU activity is far lower for large chunks: seventeen to twenty percent for Wan, versus fifty to fifty-two percent for small chunks, while LTX stays about the same. The authors read that idle capacity as room to serve more concurrent streams.

The standout is Sol-H3. Large chunks with speculative sampling reach a median latency of one point four two and one point four one seconds at low and high frequency, actually below the small-chunk baseline's one point eight seven, with no playback stalls.

### Comparison

Against related work, the paper draws its own lines. NFD, for example, also does speculative multi-frame sampling, but keeps a causal mask inside the window, so it can't use bidirectional modeling. This work keeps bidirectional joint generation inside each chunk and only makes the commitment finer.

The other common route is converting a bidirectional model into a causal streaming one, but the paper notes that a direct conversion can cost quality without careful adaptation. The authors show that a large bidirectional model can run in a low-latency streaming system without Self Forcing style post-training.

My read is that the value here is mainly the framing. Chunk size stops being a latency constraint and becomes a knob between quality and compute, which moves the optimization target toward making each chunk faster to generate.

My read is also that the evidence has clear gaps. The online tests measure latency, GPU activity and stalls, but not the quality or coherence of the video after replacements, and the baseline is only the same model with chunk-level commitment, with no head-to-head against other streaming methods.

### Limitations

The authors name two limitations. First, predictions only pay off if they're used before a new command invalidates them. With frequent or continuous changes, much of every chunk gets thrown away, wasting a lot of compute.

Second, attention inside a chunk is bidirectional, so the retained KV cache may already carry information from the future that was later discarded. Deleting those cache entries may not fully remove its influence on the new content.

Table 3 shows some weak spots too. For LTX's small chunk with CUDA Graph, speculative sampling actually has higher median latency than the baseline, and the stall ratio rises from near zero to one point eight and four point four percent. For LTX's small chunk in ordinary eager execution, both methods spend around twenty percent of the time stalled.

Also, LTX was trained on an internal dataset and evaluated on thirty-two internal cases, which makes it hard to reproduce from outside.

### Future work

The next step the authors propose, in the appendix, is to optimize DiT sampling and VAE decoding together as one pipeline, aiming to cut the time to the first deliverable frames.

The broader direction is the one the paper keeps returning to: rather than shrinking chunks further, make high-quality large-chunk generation faster, through faster DiT sampling or parallelism across GPUs.

My read: for teams building world models, games or interactive video, this is mostly a scheduling-layer change, so you don't need to train a smaller-chunk model just to get latency down. It's worth trying. But if your commands change nearly every second, work out how much compute you'll waste first.

That's today's Daily Frontier Papers. The paper is linked in the description.

