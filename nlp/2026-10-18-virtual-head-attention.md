# Virtual Head Attention

**KV缓存砍到四分之一质量还略有提升**

- 论文 Paper: [arXiv 2610.17959](https://arxiv.org/abs/2610.17959)
- 单位 Affiliation: 百度、北京大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-18
- 视频 Videos: [YouTube 中文](https://youtu.be/oP41R39kpes) · [YouTube English](https://youtu.be/YabY6Swwta0) · [B站](https://www.bilibili.com/video/BV1Yvn366EbA)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇百度和北京大学的论文，名字叫 Virtual Head Attention，虚拟头注意力，论文页脚标注的是 NeurIPS 2026。

它要解决的是大模型里一个老问题，注意力机制怎么设计，才能同时做到省显存、省算力，效果还不掉。

### 问题

先说背景。模型生成文字的时候，要把前面每个 token 的 key 和 value 存下来，这叫 KV cache。上下文越长，它占的显存越多，直接限制一张卡能同时服务多少请求，生成也会变慢。

作者把这件事叫做不可能三角，三个角分别是 KV cache 的大小，训练和 prefill 的计算量，还有模型质量。prefill 就是模型一次性读入提示词的那个阶段。

这张图中间就是这个三角。右边是作者在 1.7B 和 4B 两个尺寸上的对比，横轴是 KV cache，纵轴是训练计算量，气泡越大，平均准确率越高。

### 以前的方法

现在最常用的是 GQA，分组查询注意力，意思是多个 query 头共用一组 key 和 value。用 8 组 KV 的 GQA，效果不错，但每层每个 token 要存 4 KB 的 cache。

直接把 KV 组数降到 2，cache 能压到 1 KB，但同一组 key 和 value 要被更多 query 头共用，注意力的多样性就下降了，在 1.7B 规模上，平均准确率掉了 1 个百分点。

另外两条路。MLA 把 KV 压进一个低秩的隐空间，cache 降到 1.12 KB，但在多个尺寸上质量都是最低的。MFA 把每个头的维度做大，KV 头减到一个，cache 也是 1 KB，质量接近 GQA，代价是在 1.7B 上，训练和 prefill 的计算量多了 49.8%。

还有一点，MLA 在推理时需要定制的 kernel，和一套非标准的推理流程，部署起来要额外适配。

### 核心想法

VHA 的思路有点反直觉。它不是把 KV 加回去，而是压得更狠，KV 头只留 2 个，物理的 query 头也砍掉一半。

然后用两个很轻的线性操作，把表达能力补回来。一个叫 Q Premix，在 query 这一侧，把砍掉一半的头重新扩展成虚拟头。另一个叫 Linear Postmix，在注意力输出这一侧，让不同的头之间交换信息。

关键在于，这两个操作都是纯线性的，没有激活函数。所以训练完以后，可以直接乘进原来的 query 投影矩阵和输出投影矩阵，模型就变回一个标准的、只有 2 组 KV 的 GQA，vLLM、SGLang 这些推理框架里现成的 GQA 优化，都能直接用。

这张图左边就是 VHA 的结构，图下面那行字说的就是，推理时两个模块都被吸收进投影矩阵。

### 方法

先看 Premix。每个物理 query 头，会分别乘上两个 KV 组各自的一个小矩阵，变成两个虚拟头，各自去看对应那一组的 key 和 value。这样物理头砍了一半，虚拟头的总数又回到原来 GQA 的 query 头数。

这个小矩阵初始化成接近单位矩阵，也就是说，一开始虚拟头只是物理头的轻微扰动，保证训练初期稳定，之后随着训练慢慢分化，产生不同的注意力模式。

那为什么这样还能省计算。作者算了一下，在 1.7B 规模上，query 投影这部分的参数和计算量，差不多减少一半，这是训练计算量下降的主要来源。

再看 Postmix。注意力算完以后，在头这个维度上做一次低秩变换，形式是单位矩阵，加上两个小矩阵的乘积，1.7B 模型里秩只有 4。其中一个小矩阵初始化为零，所以一开始它什么都不做，训练中慢慢学会怎么跨组混合信息。

有意思的是，作者自己证明了，Postmix 在数学上是多余的，完全可以合并进输出矩阵。但实验里，在 Premix 的基础上加上 Postmix，1.7B 的平均准确率又提高了 1.19 个百分点。作者的解释是，拆开的参数改变了优化路径，低秩约束也起到了正则化的作用，类似 LoRA 和 RepVGG。

最后算一下 cache。每层每个 token 存 1 KB，比 dense 模型常用的 8 组 GQA 小 4 倍，比 MoE 模型常用的 4 组 GQA 小 2 倍。

### 实验和效果

实验分三层。第一层是小规模预训练，用 Qwen3 的架构，训了 0.6B、1.7B、4B 三个 dense 模型，和一个 8B-A1B 的混合专家模型，每个都用 FineWeb-Edu 的 500 亿 token，4K 上下文，在 8 个常见的选择题基准上，测零样本准确率。

结果在表 1。四个尺寸上，VHA 的平均分都是最高的，cache 也是最小的一档，和 MFA 并列。和 GQA 比，提升从 0.05 到 0.81 个百分点，比如 4B 模型，GQA 是 56.90，VHA 是 57.71。

最能说明问题的是表 2。同样在 1.7B，把 GQA 的 KV 组从 8 降到 2，掉 1 个点，再把 query 头也砍一半，变成和 VHA 一样的形状，总共掉 1.71 个点。而同样形状的 VHA，不但把这些全补回来，还比原来的 GQA 高 0.41 个点。

计算量在表 5。序列长度 4096 时，四个尺寸上，VHA 的训练计算量比 GQA 少 6.5% 到 8.1%。而 cache 同样是 1 KB 的 MFA，计算量要多出大约 48% 到 74%。

第二层是工业规模。一个 ERNIE 风格的 30B-A3B 混合专家模型，用内部数据训了 5000 亿 token，8K 上下文，FP8 精度，在 12 个基准上测。VHA 平均 50.36，MLA 是 49.66，GQA 是 47.76，而 VHA 的 cache 只有这个 GQA 的一半。

第三层是长上下文。一个 10B-A1B 的混合专家模型，全注意力层和滑动窗口层按 1 比 3 交替。两个模型只有全注意力层不同，VHA 用 2 组 KV，GQA 用 4 组，训练一路扩展到 128K 上下文。

在长文本基准 RULER 上，从 8K 到 128K，每个长度 VHA 都领先，平均 84.4 对 73.0。差距最大的是变量追踪这项任务，在 32K 到 128K，VHA 在 83 到 92.6 分之间，GQA 只有 3 到 4.2 分。

其他长文本测试里，LongBench-v2 是 24.2 对 18.9，BABILong 是 38.2 对 31.1，InfiniteBench 基本打平。

### 和以前方法的对比

再看消融，在表 4。1.7B 上，只加 Premix，比 GQA 低 0.78，只加 Postmix，低 0.14，两个都加，才高出 0.41。4B 上，单独一个模块最多高 0.32，两个一起是 0.81。

他们还试了把线性换成 SiLU 这种非线性激活，三种放法都更差，而且一旦有非线性，推理时就没法合并掉了。Postmix 的秩也试了 4、32 和 128，秩越大，训练损失越低，下游反而越差。

训练出来的矩阵也确实有结构。Premix 明显偏离了单位矩阵，在中后层最明显，两个 KV 组学到的变换也不一样。Postmix 主要在共用同一组 KV 的头之间重新分配信息，再加上少量跨组的连接。

我的判断是，这篇最实用的地方不在涨了多少分，而在部署成本几乎为零。训练时多两个小模块，推理时就是一个标准的 GQA-2，现有的推理框架不用改。相比之下，MLA 需要定制 kernel，MFA 在 1.7B 上就要多付将近一半的计算量。

我的判断是，短板也要看到。小规模实验的提升都不到 1 个点，8B-A1B 上只有 0.05，主表没有给误差范围，只有 0.6B 做了重复实验。30B 模型的代码类平均分，VHA 比 MLA 低 1.14 个点。解码提速 2 到 4 倍，论文写的是理论值，没有给实测的吞吐数据。

### 局限

作者自己写的局限。最大的验证是 30B-A3B、5000 亿 token，扩展到更大的混合专家模型和更多数据，比如 20 万亿 token，还没有做。

长上下文只在一个 10B-A1B 的模型上验证到 128K，别的长上下文架构和训练方案还没探索。另外，VHA 在强化学习训练里表现如何，也还不知道。

附录里还能看到一点，在长文本评测 HELMET 里，VHA 在上下文学习这一类上落后于 GQA。

### 未来方向

作者提到，VHA 和量化、稀疏化这些 KV cache 压缩方法是正交的，可以叠加，可能进一步省推理成本。代码已经放在 PaddlePaddle 的 GitHub 上。

我的判断是，如果这个结果在更大规模上站得住，VHA 会是新模型选注意力结构时一个很顺手的选项，因为推理侧几乎不用改。接下来值得看两件事，一是有没有独立团队复现，二是有没有实测的服务吞吐。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Baidu and Peking University. It's called Virtual Head Attention, and its footer marks it as a NeurIPS 2026 paper.

It goes after an old problem in language models. How do you design attention so it saves memory, saves compute, and doesn't lose quality?

### Problem

Some background first. When a model generates text, it stores a key and a value for every earlier token. That's the KV cache. The longer the context, the more GPU memory it eats, which limits how many requests a GPU can serve at once and slows generation down.

The authors call this an impossible triangle. The three corners are KV cache size, training and prefill compute, and model quality. Prefill is the stage where the model reads the whole prompt in one pass.

The triangle is in the middle of this figure. On the right, the authors compare methods at 1.7B and 4B: KV cache on the horizontal axis, training compute on the vertical, and bigger bubbles mean higher average accuracy.

### Prior work

Today's default is GQA, grouped-query attention, where several query heads share one set of keys and values. GQA with eight KV heads works well, but costs four kilobytes of cache per layer per token.

Cut it to two KV heads and the cache drops to one kilobyte. But now many more query heads share the same keys and values, attention gets less diverse, and at the 1.7B scale average accuracy falls by a full point.

There are two other routes. MLA squeezes keys and values into a low-rank latent space. Its cache is one point one two kilobytes, but its quality comes out lowest across several scales. MFA uses bigger heads and a single KV head. Its cache is also one kilobyte and quality is close to GQA, but at 1.7B, training and prefill compute go up by forty-nine point eight percent.

One more thing: MLA needs custom kernels and a non-standard inference pipeline, so it takes extra work to deploy.

### Key idea

VHA's move is a little counterintuitive. Instead of adding KV heads back, it cuts deeper: only two KV heads, and half the physical query heads.

Then it wins back expressiveness with two light linear operations. Q Premix works on the query side and expands the halved heads back into virtual heads. Linear Postmix works on the attention output and lets heads exchange information.

The key point is that both operations are purely linear, with no activation functions. So after training they can be multiplied straight into the existing query and output projection matrices, and the model becomes a standard GQA with two KV heads. All the existing GQA optimizations in serving frameworks like vLLM and SGLang just work.

The left side of this figure is the VHA architecture, and the line underneath says it plainly: at inference, both modules are absorbed into the projection matrices.

### Method

Start with Premix. Each physical query head is multiplied by a small matrix belonging to each of the two KV groups, giving two virtual heads, each attending to its own group's keys and values. So the physical heads are halved, but the number of virtual heads is back to GQA's original query head count.

That small matrix starts out close to the identity. At first the virtual heads are just slight perturbations of the physical ones, which keeps early training stable, and as training goes on they drift apart into different attention patterns.

Why does this also save compute? By the authors' count, at the 1.7B scale the query projection's parameters and compute shrink by roughly half, and that's the main source of the training-compute savings.

Now Postmix. After attention, it applies a low-rank transform across the head dimension: the identity plus the product of two small matrices, with rank just four in the 1.7B model. One of those matrices starts at zero, so Postmix begins as a no-op and gradually learns how to mix information across groups.

Here's the interesting part. The authors prove that Postmix is mathematically redundant; it could be merged into the output matrix. Yet in experiments, adding Postmix on top of Premix lifts the 1.7B average accuracy by another one point one nine points. Their explanation is that the separate parameters change the optimization path, and the low-rank constraint acts as a regularizer, much like LoRA and RepVGG.

Finally, the cache. One kilobyte per layer per token: four times smaller than the eight-group GQA typical of dense models, and two times smaller than the four-group GQA typical of MoE models.

### Results

The experiments come in three layers. First, small-scale pretraining on the Qwen3 architecture: dense models at 0.6B, 1.7B and 4B, plus an 8B-A1B mixture-of-experts model, each trained on fifty billion tokens of FineWeb-Edu at 4K context, and scored zero-shot on eight standard multiple-choice benchmarks.

That's Table 1. At all four scales, VHA has the best average and the smallest cache, tied with MFA. Over GQA, the gains run from zero point zero five to zero point eight one points. At 4B, for example, GQA scores fifty-six point nine, and VHA fifty-seven point seven one.

The most telling result is Table 2. At 1.7B, cutting GQA from eight KV heads to two costs one point. Halving the query heads as well, to match VHA's shape, costs one point seven one in total. VHA with that same shape wins all of it back and ends up zero point four one above the original GQA.

Compute is in Table 5. At sequence length four thousand ninety-six, VHA's training compute is six point five to eight point one percent below GQA across the four scales. MFA, at the same one-kilobyte cache, needs roughly forty-eight to seventy-four percent more.

Second layer, industrial scale: an ERNIE-style 30B-A3B mixture-of-experts model trained on five hundred billion tokens of internal data at 8K context in FP8, and tested on twelve benchmarks. VHA averages fifty point three six, MLA forty-nine point six six, and GQA forty-seven point seven six, with VHA using half that GQA's cache.

Third layer, long context: a 10B-A1B mixture-of-experts model that alternates full-attention and sliding-window layers one to three. The two models differ only in the full-attention layers, two KV heads for VHA and four for GQA, and training extends all the way to 128K context.

On the long-context benchmark RULER, VHA leads at every length from 8K to 128K, averaging eighty-four point four against seventy-three. The biggest gap is variable tracking: from 32K to 128K, VHA scores between eighty-three and ninety-two point six, while GQA gets three to four point two.

On other long-context tests, LongBench-v2 is twenty-four point two to eighteen point nine, BABILong thirty-eight point two to thirty-one point one, and InfiniteBench is essentially a tie.

### Comparison

Now the ablations in Table 4. At 1.7B, Premix alone lands zero point seven eight below GQA, Postmix alone zero point one four below, and only both together get zero point four one above. At 4B, either module alone gets at most zero point three two above GQA, and the pair gets zero point eight one.

They also swapped the linear maps for a SiLU nonlinearity. All three placements did worse, and any nonlinearity would stop the modules from being merged at inference. They tried Postmix ranks of four, thirty-two and one hundred twenty-eight too: a higher rank gave lower training loss but worse downstream scores.

The trained matrices do show structure. Premix moves clearly away from the identity, most of all in the middle and later layers, and the two KV groups learn different transforms. Postmix mostly redistributes information among heads that share the same KV group, plus a few cross-group links.

My read is that the most practical part of this paper isn't the score gain, it's that deployment costs almost nothing. You add two small modules during training, and at inference you have a standard GQA-2 that existing serving frameworks already handle. By contrast, MLA needs custom kernels, and MFA, at 1.7B, pays nearly half again in compute.

My read is also that the weak spots are real. The small-scale gains are all under one point, just zero point zero five at 8B-A1B, and the main table has no error bars; only the 0.6B run was repeated. On the 30B model, VHA's coding average is one point one four below MLA. And the two to four times faster decoding is a theoretical figure; the paper reports no measured throughput.

### Limitations

The authors' own limitations. Their largest validation is 30B-A3B on five hundred billion tokens. Scaling to bigger mixture-of-experts models and much more data, say twenty trillion tokens, hasn't been done.

Long context was validated to 128K on just one 10B-A1B model, and other long-context architectures and training recipes are unexplored. How VHA behaves under reinforcement learning is also an open question.

The appendix shows one more thing: on the HELMET long-context suite, VHA trails GQA in the in-context learning category.

### Future work

The authors point out that VHA is orthogonal to other KV cache compression methods like quantization and sparsification, so they could be stacked for further inference savings. The code is already on GitHub under PaddlePaddle.

My read is that if these results hold up at larger scale, VHA becomes an easy option when choosing attention for a new model, because almost nothing changes on the inference side. Two things to watch: whether an independent team reproduces it, and whether anyone measures real serving throughput.

That's today's Daily Frontier Papers. The paper is linked in the description.

