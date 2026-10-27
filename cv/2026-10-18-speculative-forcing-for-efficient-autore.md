# Speculative Forcing for Efficient Autoregressive Video Generation

**画面每一块不必算满步数**

- 论文 Paper: [arXiv 2610.22347](https://arxiv.org/abs/2610.22347)
- 单位 Affiliation: KAIST AI、NAVER AI Lab
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-18
- 视频 Videos: [YouTube 中文](https://youtu.be/8XdmQb-twkM) · [YouTube English](https://youtu.be/_XzByiXY1a8) · [B站](https://www.bilibili.com/video/BV1Yvn366EBr)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇韩国 KAIST 和 NAVER AI Lab 的论文，名字叫 Speculative Forcing，讲的是怎么让实时视频生成跑得更快。

它要回答的问题很具体，生成视频的时候，画面里的每一小块，是不是都得算满所有步数。

### 问题

先说背景。现在有一类视频模型是自回归的，就是一段一段往后生成，每一段都参考前面已经生成好的内容。这样就能边生成边播放，也能做成交互式的世界模型，根据动作或者相机轨迹，实时生成接下来的画面。

这类模型生成每一段，都要从噪声出发，去噪好几步。经过蒸馏，步数已经压得很少，论文里用的模型是 4 步。但每一步，画面里所有的 token 都要完整算一遍。这里的 token，指的是画面在潜空间里切出来的一个个小块。

作者问了一个很朴素的问题，每个 token 真的都需要去噪到最后一步吗。

这张图给了答案。图里亮的地方，表示某一步的结果和最终 4 步结果还差多少。在普通视频里，和前面画面相似的背景，往往很早就基本定型了。在世界模型里，之前见过的区域很早收敛，新露出来的区域还要继续算。

### 以前的方法

以前给扩散模型加速，常见的办法是缓存，也就是复用前面步骤算过的中间结果。还有一些方法，只重算模型关注的区域，或者让不同 token 在同一套流程里走不同的步数。

但论文指出，后两类方法里，每个 token 最后都还是要走到流程的终点，没有哪个 token 会被宣布已经完成。跳过的计算留下的误差，还得靠定期的全量更新来控制。

另一类方法，比如 DSA 和 Surprise Forcing，是给一整帧或者一整段决定走几步。这样一来，已经定型的区域和还在变化的区域，只能分到一样的步数。

### 核心想法

这篇的想法，借鉴了大语言模型里的投机解码。投机解码的思路是，先出一个草稿，再验证，验证通过的直接收下。作者借用了这个思路，但没有照搬它的接受规则。

放到视频里，这些少步数的模型，在每一步都会给出一个对干净画面的估计，不只是最后一步。作者就把这个中间估计当成草稿。

然后用一个验证器判断，哪些 token 的草稿已经和最终结果足够接近。通过的 token 直接定稿，不再去噪，没通过的继续算。

### 方法

整体流程就是这张图。上面是原来的做法，所有 token 走满全部步数。下面是 Speculative Forcing，每一步之后验证一次，打勾的 token 被收下，下一步只算剩下的。

这里有个细节很关键。被收下的 token 不再参与去噪，但它的 key 和 value，也就是注意力计算要用的缓存，会保留下来，还在去噪的 token 仍然可以参考它。

一段生成完，模型再对拼好的干净结果做一次前向计算，刷新缓存，留给下一段用。

那什么叫足够接近。论文定义了一个收敛差距，就是当前草稿和完整 4 步结果之间的距离，再除以这一段里 token 大小的中位数。差距不超过一个阈值 epsilon，就算收敛。

问题是，推理的时候拿不到完整 4 步的结果，不然就没必要省了。所以需要验证器去预测这个差距。作者给了两种验证器。

第一种不用训练。第一步的时候，只有一个草稿，就看它和前面已经生成好的干净内容有多像，找最相似的那个 token 算距离，足够像就收下。

从第二步开始，看同一个 token 前后两次草稿变了多少，变化足够小就收下。两个阈值，都在留出的数据上校准。

第二种是学出来的验证器。作者认为，像不像历史、变没变，都只是收敛的间接信号。一个 token 可能和历史很像，却还需要细化，也可能两步之间变化很小，离最终结果却还很远。

所以他们直接读生成模型最后一层的中间特征，用一个很浅的小网络去预测收敛差距。这个小网络是层归一化加三层 3D 卷积，只有 411 万参数。训练时生成模型冻结，只训练这个小头。

推理的时候，预测的差距不超过 epsilon 就收下。把 epsilon 调大就更快，调小就更稳，不用重新训练。

另外，两种验证器的打分在做决定之前，都会在空间和时间上做一次平滑，让相邻 token 的决定保持一致，避免画面不连贯。

### 实验和效果

实验设置。作者把方法用在两个 13 亿参数、4 步去噪的模型上，一个是视频生成模型 LongLive，一个是世界模型 Lingbot-World 2.0。两个生成模型都完全不改，速度都在单张 H100 上测。

评测主要用 VBench，这是一个视频生成的自动评测基准。LongLive 测 5 秒短视频和 30 秒长视频。世界模型用 200 个留出的场景，每个场景配一条随机的相机轨迹，还额外测生成的画面是不是按指令的相机路径在走。

先看 5 秒短视频，表 1。原来的 4 步模型每秒生成 26.4 帧，VBench 总分 83.08。用学习验证器，平均每个 token 只走 1.68 步，速度到每秒 41.74 帧，总分 83.21。

免训练的验证器，平均 1.65 步，每秒 43.61 帧，总分 83.12，也和 4 步持平。

再看 30 秒长视频，表 2。4 步模型每秒 24.5 帧，总分 82.69。用学习验证器，平均 1.72 步，每秒 37.8 帧，总分 82.81。

长视频还有一个指标叫 Drift，衡量画质随时间的波动，越低越好。两种验证器的 Drift，都比 4 步模型低。

最难的是世界模型，表 3。相机一直在动，新内容不断出现，所以平均要 2.55 步，比视频生成多。速度从每秒 18.1 帧到 24.5 帧，质量分 82.65，和 4 步的 82.62 持平，相机位姿的误差也基本一样。

论文第一页这张图，把三个场景放在一起。横轴是平均步数，灰线是原来均匀步数的做法，红点是 Speculative Forcing。论文报告的提速，短视频 1.58 倍，长视频 1.54 倍，世界模型 1.35 倍。

### 和以前方法的对比

更关键的，是和别的省步数办法比，表 5。在世界模型上，作者借鉴 MotionCache 和 RAS 两种已有方法，把验证信号换成相邻帧的差异，或者通道方差。平均步数差不多，质量分只有 80.97 和 78.23，明显低于学习验证器的 82.65。

粒度也很重要。用同一个学习验证器，但改成整段一起收，质量分降到 82.54，Drift 从 2.839 升到 3.214。原因是整段一起收，分不清哪些 token 已经定型，哪些还在变。

作者还做了一个理想对照，直接算出真实的收敛差距再决定收不收，叫 oracle。oracle 的质量分是 82.61，学习验证器是 82.65。论文的结论是，学习验证器不用算完整结果，质量就和 oracle 相当。

附录里，作者还在 LongLive 上实现了 Surprise Forcing 的步数分配策略来对比，它是按整段分配步数的方法。在 5 秒视频上，Surprise Forcing 平均 3.59 步，每秒 27.38 帧，学习验证器 1.68 步，每秒 41.74 帧，总分 83.11 对 83.21，非常接近。

我的判断是，这里最有说服力的，不是比 4 步快，而是比同样快的简单办法更好。比如直接均匀走 2 步，5 秒视频每秒 44.54 帧，总分 82.46，Speculative Forcing 速度接近，总分是 83.21。

我的判断是，也要看到，和 4 步相比，这些分数的差距都很小，很多在零点一分上下，而且评测全是 VBench 这类自动指标，论文没有报告人工评测。所以更准确的说法是质量持平，而不是更好。

### 局限

作者在附录里写了局限。能加速多少，取决于底层模型和生成场景。如果模型对减少步数很敏感，或者场景更难，能提前收下的 token 就少。世界模型平均要 2.55 步就是例子，所以不保证在别的模型和场景上有同样的加速。

还有两个额外开销。每一段生成完，都要多做一次完整前向来刷新缓存。而且一段里只要还有一个 token 没收下，这一步就得跑，所以实际的速度提升，比按平均步数算出来的要少。

此外，免训练验证器的阈值要针对每个设置单独校准，论文里两个模型的学习验证器，也是分开训练的。

### 未来方向

未来方向，作者提到，可以借鉴复用带噪缓存的方法，省掉那次刷新缓存的前向，把理论加速再往上推。

他们也认为，这个方法在很多内容已经被条件确定的场景里可能特别有效。比如说话人视频，背景基本不动，主要是脸和上半身在动。还有基于真实环境的世界模型，以及 AR、VR 这类应用。

我的判断是，这篇的价值在于，把先出草稿、再验证的思路，做到了视频扩散里每个 token 的粒度，而且不改生成模型的权重，可以套在现有的少步数模型上。

我的判断是，对实时世界模型来说，从每秒 18.1 帧提到 24.5 帧，可能比画质再提高零点几分更实用。但它能省多少，取决于画面里有多少内容是已经见过的。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from KAIST AI and NAVER AI Lab. It's called Speculative Forcing, and it's about making real-time video generation faster.

The question behind it is simple. When a model generates a video, does every little piece of the frame really need the full number of steps?

### Problem

Some background first. A growing class of video models is autoregressive. They generate a video chunk by chunk, each chunk conditioned on what came before. That lets you stream video as it's generated, and it's the basis of interactive world models, which produce the next frames from actions or a camera path.

Each chunk starts from noise and is denoised over several steps. Distillation has already cut that to a handful, four steps for the models in this paper. But at every step, every token in the frame gets computed. A token here is one small patch of the frame in the model's latent space.

So the authors ask a plain question. Does every token need to be denoised all the way to the final step?

This figure is their answer. The bright areas show how far a given step's result still is from the final four-step output. In ordinary video, background that resembles earlier frames tends to settle early. In the world model, regions the camera has already seen converge early, while newly revealed regions keep changing.

### Prior work

The usual way to speed up diffusion is caching: reuse intermediate results from earlier steps. Other methods recompute only the regions the model attends to, or run different tokens for different numbers of steps along the same schedule.

But as the paper points out, in those last two approaches every token is still carried to the end of the schedule. No token is ever declared finished, and the error from skipped computation has to be kept in check with periodic full updates.

Another line of work, including DSA and Surprise Forcing, picks the number of steps for a whole frame or chunk at once. That means settled regions and still-changing regions get the same budget.

### Key idea

The idea here borrows from speculative decoding in language models: produce a cheap draft, verify it, and keep whatever passes. The authors take that draft-and-verify idea, though not its exact acceptance rule.

In video, these few-step models produce an estimate of the clean frame at every step, not just the last one. The authors treat that intermediate estimate as a draft.

A verifier then decides which tokens' drafts are already close enough to the final result. Accepted tokens are frozen and get no more denoising. The rest keep going.

### Method

This figure shows the whole pipeline. On top is the baseline, where every token runs every step. Below is Speculative Forcing: after each step there's a verification, checked tokens are accepted, and the next step only computes the ones left.

One detail matters a lot. An accepted token stops being denoised, but its keys and values, the cached state that attention reads, stay in place, so the tokens still being refined can keep attending to it.

Once a chunk is done, the model runs one clean forward pass over the assembled result to refresh the cache for the next chunk.

So what counts as close enough? The paper defines a convergence gap: the distance between the current draft and the full four-step output, divided by the median token norm in that chunk. If the gap is at or below a threshold, epsilon, the token has converged.

The catch is that at inference time you don't have the four-step output, otherwise there'd be nothing to save. So a verifier has to predict that gap, and the authors offer two.

The first needs no training. At step one there's only a single draft, so it asks how similar that draft is to clean content that's already been generated. It finds the most similar earlier token, and if the distance is small enough, the token is accepted.

From step two on, it looks at how much a token's draft changed between consecutive steps, and accepts it if the change is small enough. Both thresholds are calibrated on held-out data.

The second verifier is learned. The authors argue that resembling the history, or not changing much, are only proxies for convergence. A token can match the history and still need refinement, or barely change between two steps and still be far from its final value.

So they read the generator's own hidden states from its final block, and train a shallow head to predict the convergence gap directly. The head is a layer norm plus three 3D convolutions, about four point one million parameters. The generator stays frozen, and only the head is trained.

At inference, a token is accepted when its predicted gap is at or below epsilon. Raise epsilon to go faster, lower it to play safe, with no retraining.

Both verifiers also smooth their scores across space and time before making a decision, so neighboring tokens get consistent calls and the frame stays coherent.

### Results

The setup. The method is applied to two one-point-three-billion-parameter models with four denoising steps: LongLive for video generation, and the world model Lingbot-World 2.0. Neither generator is modified, and speed is measured on a single H100.

Quality is measured mainly with VBench, an automatic benchmark for video generation. LongLive is tested on five-second and thirty-second videos. The world model is tested on two hundred held-out scenes, each with a random camera path, and the authors also check whether the camera follows the commanded trajectory.

Short videos first, in Table 1. The original four-step model runs at twenty-six point four frames per second with a VBench total of eighty-three point zero eight. With the learned verifier, tokens average just one point six eight steps, speed rises to forty-one point seven four frames per second, and the total is eighty-three point two one.

The training-free verifier averages one point six five steps, runs at forty-three point six one frames per second, and scores eighty-three point one two, also on par with four steps.

For thirty-second videos, Table 2, the four-step model runs at twenty-four point five frames per second and scores eighty-two point six nine. The learned verifier averages one point seven two steps, runs at thirty-seven point eight, and scores eighty-two point eight one.

Long videos also get a metric called Drift, which measures how much image quality wobbles over time. Lower is better, and both verifiers come in below the four-step model.

The hardest case is the world model, Table 3. The camera keeps moving and new content keeps appearing, so tokens need two point five five steps on average, more than in plain video. Speed goes from eighteen point one to twenty-four point five frames per second, the quality score is eighty-two point six five against eighty-two point six two for four steps, and camera pose error is essentially unchanged.

The figure on the first page puts all three together. The horizontal axis is mean steps, the gray line is uniform-step sampling, and the red points are Speculative Forcing. The reported speedups are one point five eight times for short videos, one point five four for long videos, and one point three five for the world model.

### Comparison

The more telling comparison is against other ways of saving steps, in Table 5. On the world model, the authors swap in verification signals adapted from two existing methods, MotionCache and RAS: frame-to-frame temporal difference, and channel variance. At similar mean steps, those score eighty point nine seven and seventy-eight point two three, well below the learned verifier's eighty-two point six five.

Granularity matters too. Use the same learned verifier, but accept whole chunks at once, and quality drops to eighty-two point five four while Drift rises from two point eight three nine to three point two one four. Chunk-level acceptance can't tell settled tokens from ones that are still changing.

There's also an oracle baseline, which computes the true convergence gap before deciding. The oracle scores eighty-two point six one, and the learned verifier eighty-two point six five. The paper's conclusion is that the learned verifier matches the oracle's quality without ever computing the full-step output.

In the appendix, the authors also implement Surprise Forcing's step-allocation policy on LongLive for comparison; it allocates steps per chunk. On five-second videos, Surprise Forcing averages three point five nine steps at twenty-seven point three eight frames per second. The learned verifier averages one point six eight steps at forty-one point seven four. Totals are close, eighty-three point one one versus eighty-three point two one.

My read is that the strongest result isn't beating four steps on speed. It's beating the simple option at a similar speed. Uniform two-step sampling on five-second videos runs at forty-four point five four frames per second and scores eighty-two point four six. Speculative Forcing is close in speed and scores eighty-three point two one.

My read is also that the gaps against four steps are tiny, many around a tenth of a point, and every quality number comes from automatic metrics like VBench. The paper reports no human evaluation. So the fair claim is that quality holds, not that it improves.

### Limitations

The authors list their limitations in the appendix. How much speedup you get depends on the base generator and the domain. If a model is sensitive to fewer steps, or the domain is harder, fewer tokens can be accepted early. The world model needing two point five five steps is an example, so the same speedup isn't guaranteed on other models or domains.

There are two overheads as well. Every chunk needs an extra full forward pass to refresh the cache. And a step has to run as long as any single token in the chunk is still active, so the real speedup falls short of what the mean step count implies.

On top of that, the training-free thresholds are calibrated separately for each setting, and the learned verifier was trained separately for each of the two models.

### Future work

Looking ahead, the authors point to methods that reuse noisy cached states, which could skip that cache-refresh pass and push the ideal speedup higher.

They also suggest the approach may be especially effective where much of the content is already pinned down by the conditions. Think talking-head video, where the background barely moves and the motion is around the face and upper body, environment-grounded world models, and augmented and virtual reality.

My read is that the contribution is taking draft-and-verify down to the level of individual tokens in video diffusion, without touching the generator's weights, so it can sit on top of existing few-step models.

My read is that for real-time world models, going from eighteen point one to twenty-four point five frames per second may matter more than another fraction of a point in quality. But how much it saves depends on how much of the scene has already been seen.

That's today's Daily Frontier Papers. The paper is linked in the description.

