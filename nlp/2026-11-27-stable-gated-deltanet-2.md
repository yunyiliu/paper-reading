# Stable Gated DeltaNet-2: Understanding and Mitigating Recurrent State Instability

**线性注意力为什么会训崩**

- 论文 Paper: [arXiv 2611.34051](https://arxiv.org/abs/2611.34051)
- 单位 Affiliation: MemTensor · 中国科学技术大学 · 香港科技大学 · 上海交通大学等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-27
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1Em1xzKEuZ)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 MemTensor 联合中国科学技术大学、香港科技大学、上海交通大学等单位的论文，名字叫 Stable Gated DeltaNet-2，副标题是，理解并缓解循环状态的不稳定。

一句话说，有一种很新的线性注意力结构，在长上下文训练里可能会突然崩掉，这篇论文找到了原因，还给出了一个带数学保证的修法。

### 问题

先说背景。标准的 softmax 注意力，序列越长，计算越贵。线性注意力的思路是，把看过的所有内容，压缩进一个固定大小的矩阵里，这个矩阵叫循环状态，这样计算量随长度线性增长，生成时占的内存也不变。

这条路线这两年进化很快。DeltaNet 用一种叫 delta rule 的更新方式，先把记忆里和当前 key 相关的旧内容擦掉一部分，再写入新内容。Gated DeltaNet 在这之前加了一个遗忘门，让旧记忆先衰减。

最新的一代叫 Gated DeltaNet-2，简称 GDN-2。它把擦除和写入拆成两个独立的、按通道控制的门，表达能力更强，上下文检索也更好。

但问题来了。作者在 64K 长度的训练里发现，GDN-2 有时会训练失败，loss 直接变成 NaN，也就是数值溢出，训练没法继续。

看表 1。纯线性模型，4 次预训练失败 1 次，4 次后训练也失败 1 次。混合模型更糟，预训练 4 次失败 2 次，后训练 4 次失败 3 次。

更麻烦的是，就算训练顺利跑完，状态里的单个数值也可能超过 10 的 36 次方，而 loss 曲线上几乎看不出预警。只盯着 loss，是发现不了这个问题的。

### 以前的方法

这个问题，在上一代的 GDN 和 Kimi Delta Attention，也就是 KDA 身上没有出现。原因是它们擦除的时候，读取旧内容的方向和修改的方向是同一个，都是 key。这种对称的擦除，在数学上保证不会把状态放大。

针对 GDN-2 的不稳定，已经有两种稳定办法，论文拿它们当基线。一种叫 D-Clip，把衰减因子压低，让每一步整体不放大。另一种叫 Beta-Guard，在需要的时候，缩小擦除门各个通道之间的差异。两种都保留了 GDN-2 原来的写入方式。

### 核心想法

作者认为，问题的根源在于 GDN-2 擦除时的不对称。它读取旧内容，用的是被擦除门加权过的 key，但修改还是沿着原来的 key 去做。两个方向不一致，就会产生一种剪切效应。

看这张图。左边是 GDN-2，读和改的方向错开，一部分状态被保留下来，同时又在另一个方向上多出一个分量，整体就可能变大。右边是作者的版本，读和改对齐，每一步只会缩小或者不变。

论文的定理 3.1，给出了单步擦除最大放大倍数的精确公式。结论是，只要读和改的方向不平行，这一步擦除就一定会在某个方向上把状态放大，哪怕所有门的取值都在 0 到 1 之间。

单步放大一点还不要紧，麻烦在于累积。如果上一步放大的方向，正好对上下一步会放大的方向，增益就会一步步叠起来。推论 3.2 构造了一个例子，只在第一个 token 写入一次，之后完全不写，状态也会指数增长，直到无穷大。

作者还回放了一次真实的失败。图 2 里，这次失败出现在训练第 2058 步。在触发失败的那条序列上，GDN-2 的状态范数一路上涨，到第 52736 个 token 溢出，而 GDN 和 KDA 在同样的回放里没有这种增长。

再做一个实验，只保留前 64 个 token 的写入，之后的写入全部清零。GDN 和 KDA 的状态会衰减下去，GDN-2 却继续增长。同时，GDN-2 状态的有效秩，可以理解成状态实际用上了多少个维度，很快掉到接近 1，能量全挤在一个方向上。

### 方法

修法分两步。第一步叫对称擦除。把擦除门开平方根，分别乘到 key 的两边，得到一个新向量 u。读取、修改、写入都用这同一个 u，擦除矩阵就变成对称的了。

但这样有个副作用。某个通道的擦除门接近 0 的时候，这个通道的擦除和写入会一起消失。想写新东西，就得同时擦掉旧东西，写入被绑住了。

第二步叫衰减调节的残差写入。沿着 key 减去 u 的方向，补一条额外的写入，它的幅度由衰减决定。衰减越弱，补写越少，衰减越强，允许的补写越多。

这张架构图里，状态更新就是三步，先按通道衰减，再做对称擦除，最后加上主写入和残差写入。

理论上，作者给了两个保证。命题 4.1 说，没有新写入的时候，状态范数每一步只会缩小或者不变。命题 4.2 说，在 value 向量有界、状态从零开始的条件下，状态范数的增长，不超过序列长度平方根的量级。

工程上，它保留了和 GDN、KDA 一样的分块并行训练结构。残差写入只增加一些矩阵乘法，块内要解的三角方程组规模不变，复杂度仍然随序列长度线性增长。

### 实验和效果

实验设置。模型都是 20 层，大约 15 亿到 16 亿参数，分两种。一种全是线性注意力层，论文叫循环设置。另一种每三层线性注意力配一层完整注意力，叫混合设置。

训练数据是 ProLong，按 64K 长度打包，大约 828 亿 token，每个模型跑一个 epoch，用 16 张 H800。对比对象有四个，GDN-2、KDA、D-Clip 和 Beta-Guard。

先看长上下文检索，用的是 RULER 基准，在表 3。循环设置下，Stable GDN-2 在 32K、64K、128K 三个长度上，平均分都是第一，不过 32K 只比 D-Clip 高 0.01。

差距最大的是 128K。Stable GDN-2 是 11.62，原版 GDN-2 只有 4.82，高了 6.8 分。D-Clip 是 7.02，Beta-Guard 是 10.82，KDA 是 9.96。

混合设置下就没那么一边倒。32K 是第一，31.72。64K 是 18.69，略低于 Beta-Guard 的 19.13。到了 128K 是 8.78，低于 Beta-Guard 的 10.52，也略低于 D-Clip 的 8.85。

再看九个常识推理任务的平均准确率，在表 2。循环设置下是 50.39，和 GDN-2 持平。混合设置下是 50.53，比 GDN-2 高 0.62 个百分点，是这一组里最高的。

语言建模的困惑度有好有坏。循环设置下，WikiText 困惑度在 8K 到 64K 的各个窗口都比 GDN-2 低，但比 KDA 高。混合设置下，比 GDN-2、D-Clip 和 KDA 都略高。

还有一项，循环设置下的 LAMBADA 困惑度，Stable GDN-2 是 9.333，比 GDN-2 的 7.856 高不少。

### 和以前方法的对比

消融实验在图 4，看两个部件各自的作用。去掉残差写入，常识推理平均从 50.39 掉到 49.43，四个大海捞针检索任务的平均分，从 25.43 掉到 22.22。

如果保留残差写入，但去掉衰减调节，检索掉得更厉害，降到 13.53。作者的解读是，多写进去的这部分能量，必须有东西管住。

和 D-Clip、Beta-Guard 比，作者的说法是，那两种方法靠限制状态传播或者擦除强度来换稳定，而 Stable GDN-2 保留了按通道擦除和灵活写入。

我的判断是，这篇最有价值的部分其实是诊断。它说明 loss 曲线正常，不代表循环状态健康，做线性注意力的团队，值得把状态范数和有效秩放进训练监控里。

我的判断是，绝对分数也要看清楚。128K 的 RULER 平均分最好也只有 11.62，这些 16 亿参数级别的小模型，离真正好用的长上下文检索还很远。

### 局限

论文没有单独的局限章节，从实验设置看，有几点要注意。第一，主要对比只用了一组随机种子。失败统计跑了四组种子，但那是对 GDN-2 做的，Stable GDN-2 在同样四组种子下的表现，论文没有给出同样的统计。

第二是规模。循环模型大约 16 亿参数，混合模型大约 15.5 亿，训练长度 64K，更大的模型、更长的训练，论文没有测试。

第三，理论保证有前提，平方根量级的那个界，要求 value 向量有界，而且状态从零开始。作者也说明，定理证明的是放大的轨迹存在，实际训练里会不会放大、放大多少，取决于数据和学到的参数。

第四，不是所有指标都赢。混合设置下的困惑度，和混合设置下 128K 的检索，它都不是最好的。

### 未来方向

论文的结论是，控制循环状态的动态很重要，而且数值稳定性可以明显改善，同时不牺牲循环记忆的表达能力。

我的判断是，下一步最该看的，是更大规模、更长上下文下，它能不能继续稳定，以及在混合架构里，检索上的优势能不能也拉开。

我的判断是，这篇的思路值得借鉴。先从训练失败的现象，追到数学上的根因，再用一个改动很小、能证明的结构把它堵上，而不是事后去压参数。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from MemTensor, working with USTC, HKUST, Shanghai Jiao Tong University and others. It's called Stable Gated DeltaNet-2, and the subtitle is understanding and mitigating recurrent state instability.

In one line: a very new linear attention design can blow up during long-context training, and this paper finds out why and offers a fix with a mathematical guarantee.

### Problem

Some background first. Standard softmax attention gets more expensive as the sequence grows. Linear attention instead compresses everything it has seen into a fixed-size matrix, called the recurrent state, so compute grows linearly with length and memory stays constant during generation.

This line of work has moved fast. DeltaNet uses an update called the delta rule: it first erases part of the old content tied to the current key, then writes the new content. Gated DeltaNet adds a forget gate that decays the old memory before that correction.

The newest generation is Gated DeltaNet-2, or GDN-2. It splits erasing and writing into two independent gates that work channel by channel, which makes the update more expressive and improves in-context retrieval.

Here's the catch. Training at a context length of sixty-four K, the authors found that GDN-2 can fail outright. The loss turns into NaN, meaning the numbers have overflowed and training can't continue.

Look at Table 1. For the purely linear model, one of four pretraining runs failed, and one of four post-training runs. The hybrid model was worse: two of four pretraining runs and three of four post-training runs failed.

Worse still, even runs that finish can hide individual state values above ten to the thirty-sixth power, with little warning in the loss curve. If you only watch the loss, you won't see this coming.

### Prior work

The earlier models, GDN and Kimi Delta Attention, or KDA, don't have this problem. When they erase, they read the old content and modify the state along the same direction, the key. That symmetric erasure is mathematically guaranteed not to amplify the state.

Two fixes for GDN-2's instability already exist, and the paper uses them as baselines. One, called D-Clip, lowers the decay factors so that each full step can't amplify. The other, Beta-Guard, shrinks the differences between the erase gate's channels when needed. Both keep GDN-2's original write term.

### Key idea

The authors trace the problem to an asymmetry in how GDN-2 erases. It reads old content with a key that has been reweighted by the erase gate, but it applies the change along the original key. When those two directions differ, you get a shear.

This figure shows it. On the left is GDN-2: read and modify directions are misaligned, so part of the state is kept while an extra component appears along another direction, and the total can grow. On the right is the authors' version: the directions line up, and each step can only shrink the state or leave it unchanged.

Theorem three point one gives the exact maximum gain of a single erase step. The upshot is that whenever the read and modify directions aren't parallel, that erase step amplifies the state in some direction, even though every gate value sits between zero and one.

A small gain in one step isn't the real danger. Accumulation is. If the direction amplified at one step lines up with a direction the next step expands, the gains stack up. Corollary three point two builds an example where there's a single write at the first token and nothing after, and the state still grows exponentially, without bound.

The authors also replayed a real failure. In Figure 2, the run fails at training step two thousand and fifty-eight. On the sequence that triggered it, the GDN-2 state norm climbs steadily and overflows at token fifty-two thousand seven hundred and thirty-six, while GDN and KDA show no such growth in the same replay.

Then a cleaner test: keep the writes from the first sixty-four tokens and zero out every write after that. GDN and KDA decay away, but GDN-2 keeps growing. Meanwhile, the effective rank of the GDN-2 state, roughly how many dimensions the state is really using, quickly drops to about one, with all the energy squeezed into a single direction.

### Method

The fix has two parts. The first is symmetric erasure. Take the square root of the erase gate, apply one factor to each side of the key, and call the result u. Reading, modifying and writing all use that same u, so the erase matrix becomes symmetric.

But that has a side effect. When the erase gate for a channel approaches zero, both erasing and writing in that channel vanish. You can't write something new without also erasing, so writing gets tied down.

The second part is residual writing regulated by decay. It adds an extra write along the direction of the key minus u, with a size set by the decay. Weaker decay means less extra writing, and stronger decay allows more.

In this architecture diagram, the state update has three steps: per-channel decay, then symmetric erasure, then the main write plus the residual write.

On the theory side, the authors prove two guarantees. Proposition four point one says that with no new writes, the state norm can only shrink or stay the same at each step. Proposition four point two says that with bounded value vectors and a state starting at zero, the state norm grows at most in proportion to the square root of the sequence length.

On the engineering side, it keeps the same chunkwise parallel training structure as GDN and KDA. The residual write just adds some matrix products, the triangular system solved inside each chunk stays the same size, and cost still grows linearly with sequence length.

### Results

The setup. All models have twenty layers and about one point five to one point six billion parameters, in two flavors. One uses only linear attention layers, which the paper calls the recurrent setting. The other pairs every three linear attention layers with one full attention layer, the hybrid setting.

Training data is ProLong, packed into sixty-four K sequences, about eighty-two point eight billion tokens, one epoch per model on sixteen H800 GPUs. There are four baselines: GDN-2, KDA, D-Clip and Beta-Guard.

Start with long-context retrieval on the RULER benchmark, in Table 3. In the recurrent setting, Stable GDN-2 has the best average at all three lengths, thirty-two K, sixty-four K and one hundred twenty-eight K, though at thirty-two K it beats D-Clip by just zero point zero one.

The biggest gap is at one hundred twenty-eight K. Stable GDN-2 scores eleven point six two, against just four point eight two for the original GDN-2, a gain of six point eight points. D-Clip gets seven point zero two, Beta-Guard ten point eight two, and KDA nine point nine six.

The hybrid setting is more mixed. At thirty-two K it's first, at thirty-one point seven two. At sixty-four K it scores eighteen point six nine, just under Beta-Guard's nineteen point one three. At one hundred twenty-eight K it scores eight point seven eight, below Beta-Guard's ten point five two and slightly below D-Clip's eight point eight five.

Next, average accuracy across nine common-sense reasoning tasks, in Table 2. In the recurrent setting it's fifty point three nine, tied with GDN-2. In the hybrid setting it's fifty point five three, zero point six two points above GDN-2 and the best in that group.

Language modeling perplexity is a mixed picture. In the recurrent setting, WikiText perplexity is lower than GDN-2 at every window from eight K to sixty-four K, but higher than KDA. In the hybrid setting, it's slightly higher than GDN-2, D-Clip and KDA.

One more number: in the recurrent setting, LAMBADA perplexity is nine point three three three for Stable GDN-2, clearly worse than GDN-2's seven point eight five six.

### Comparison

The ablations are in Figure 4, testing each part on its own. Remove the residual write, and average reasoning accuracy falls from fifty point three nine to forty-nine point four three, while the average over four needle-in-a-haystack retrieval tasks drops from twenty-five point four three to twenty-two point two two.

Keep the residual write but drop the decay regulation, and retrieval falls much further, to thirteen point five three. The authors' reading is that the extra energy coming in through that write has to be kept in check.

Against D-Clip and Beta-Guard, the authors' argument is that those methods buy stability by constraining state propagation or erasure strength, while Stable GDN-2 keeps channel-wise erasure and flexible writing.

My read is that the most valuable part of this paper is the diagnosis. A healthy-looking loss curve doesn't mean a healthy recurrent state, and teams building linear attention should probably add state norms and effective rank to their training dashboards.

My read is also that the absolute scores deserve a clear look. The best one hundred twenty-eight K RULER average here is eleven point six two. These models, around one point six billion parameters, are still a long way from long-context retrieval you'd rely on.

### Limitations

The paper has no dedicated limitations section, but a few things stand out from the setup. First, the main comparisons use a single random seed. The failure counts cover four seed settings, but only for GDN-2, and the paper doesn't report the same count for Stable GDN-2 across those four seeds.

Second, scale. The recurrent model has about one point six billion parameters, the hybrid about one point five five billion, trained at sixty-four K, and larger models or longer training aren't tested.

Third, the theory comes with conditions: the square-root bound needs bounded value vectors and a state that starts at zero. The authors also note that the theorem shows amplifying trajectories exist, and whether and how much amplification happens in real training depends on the data and the learned parameters.

Fourth, it doesn't win everywhere. In the hybrid setting it's not the best on perplexity, and not the best on one hundred twenty-eight K retrieval.

### Future work

The paper's conclusion is that controlling recurrent state dynamics matters, and that numerical stability can be substantially improved without giving up the expressive power of recurrent memory.

My read is that the next thing to watch is whether it stays stable at larger scale and longer context, and whether its retrieval edge can also open up in hybrid architectures.

My read is that the approach is worth copying. Trace a training failure back to its mathematical root cause, then close it with a small, provable change to the structure, rather than tightening knobs after the fact.

That's today's Daily Frontier Papers. The paper is linked in the description.

