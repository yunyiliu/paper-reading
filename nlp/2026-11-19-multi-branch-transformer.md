# Multi-Branch Transformer

**模型不只能加深还能分叉**

- 论文 Paper: [arXiv 2611.19800](https://arxiv.org/abs/2611.19800)
- 单位 Affiliation: 伊利诺伊大学香槟分校、康奈尔大学等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-19
- 视频 Videos: [YouTube 中文](https://youtu.be/cZDlEtom0vk) · [YouTube English](https://youtu.be/WZkr2ZCcwiQ) · [B站](https://www.bilibili.com/video/BV1oWsi6fEZ4)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇伊利诺伊大学香槟分校、莱斯大学、密歇根大学和康奈尔大学合作的论文，题目叫 Multi-Branch Transformer，多分支 Transformer。

它问的问题很基础，语言模型要变大，除了加宽和加深，还有没有第三条路。作者给的答案是分叉，让几串完整的 Transformer 块并排着跑。

### 问题

先说背景。现在的语言模型，基本都是一个个 Transformer 块串起来的。想让模型更强，常规做法就两个，把每一块做宽，或者多叠几块。

这些年注意力、MoE、残差连接都有不少改进，但块一个接一个串着，这一点没变。而生成每个 token 时，串着的块要依次算完，块越多，固定开销越大。

另一头是 KV cache，也就是模型为已经读过的每个 token 存下的键和值。上下文越长、同时处理的请求越多，读这块缓存的时间就越占大头。

### 以前的方法

以前也有人在 Transformer 里做并行。比如 PaLM，把一个块里面的注意力和前馈层并排算，这是块内并行。

还有一类是调宽度和深度的比例，用更宽、更浅的堆叠。另外像 ParScale，是让同一个主干并行跑几遍，每一遍加不同的可学习前缀。

多分支结构在计算机视觉里早就有，比如 Inception 和 ResNeXt。但作者认为，把完整的 Transformer 块并排组合，用来扩展今天的自回归语言模型，还很少有人系统研究过。

### 核心想法

这篇的想法，一张图就能说清楚。左边是普通 Transformer，块一个接一个。右边是 MBT，前面几层共享，中间分成几条独立的分支同时跑，最后把分支的输出融合起来，再交给后面共享的几层。

前面共享的部分叫 prelude，后面的叫 coda，是沿用了已有工作的叫法。每条分支有自己独立的参数，不是同一组权重跑几遍。

关键在于，每条分支内部仍然保留串行的深度，而后面的共享层，可以同时用上几条不同变换路径的结果。

### 方法

先说融合。在每个 token 的位置上，把各条分支的输出分别做归一化，拼接起来，再用一个线性层投影回原来的宽度。就这么简单。

然后是怎么分配算力。在训练计算量基本相同的前提下，作者设计了三种分法。这张图用一个 6 块的模型举例，第一块和最后一块共享，中间四块改成两条分支。

Wide-Shallow，宽而浅，块的宽度不变，中间的块分到两条分支上，每条两块。Narrow-Deep，窄而深，每条分支保留原来的深度，四块，但每块变窄。Balanced，平衡型，宽度和深度都取中间，每条分支三块。

第二个设计叫 Joint-KV。默认情况下，每条分支的每一块，都有自己的一份 KV cache，分支越多，缓存就越多。

Joint-KV 的做法是，同一深度上的所有分支共用一份 KV cache，这份缓存由各分支的输入拼起来投影得到，而每条分支保留自己的 query。和每条分支各存一份相比，分支区域的缓存变成几分之一，有四条分支就是四分之一，投影的参数量和计算量不变。

Joint-KV 改的是模型本身，训练时就用上了。另外，作者还基于 nano-vLLM 写了一个推理引擎，只改执行方式，不改模型。

引擎有两招。第一招叫 branch grouping，把不同分支里同一深度的块合成一步来算。论文举的例子是，一个四分支模型有 24 个块，一条一条跑要 24 步，合起来只要 12 步，而同规模的普通 Transformer 是 16 步。

第二招叫 head folding。共享的缓存，如果每条分支各读一遍，内存流量其实没省下来。head folding 把各分支的 query 并进同一次分组注意力的调用，共享的缓存只读一次。

### 实验和效果

实验设置。作者训练了四个规模的 MoE 模型，叫 W1 到 W4，普通 Transformer 的总参数从 8 亿到 20.7 亿，每个 token 实际激活的主体参数是 1.39 亿到 3.94 亿。数据都是 FineWeb-Edu，上下文长度 4096。

每个规模下，普通模型和 MBT 的训练 token 数一样，大约是每个激活主体参数 200 个 token。普通模型的学习率在每个规模都仔细调过，MBT 只在一个配置上调，然后沿用到其他配置。

第一组实验，用两分支的宽浅型，改变放进分支的块的比例。四个规模都是一条 U 形曲线，大约三分之一的块放进分支时最好，验证 loss 比普通模型低 0.025 到 0.031。

再往上加，按作者的插值估算，大约百分之79的块都放进分支，模型仍然和普通 Transformer 持平。

按拟合出来的 scaling 曲线，作者称，要达到同样的验证 loss，普通 Transformer 大约需要 1.23 倍的训练计算量。九个零样本下游任务，按四个规模平均，九项都是 MBT 更高，平均准确率从百分之54.23 提到百分之56.03。

第二组实验，把分支数从 2 加到 3 和 4。验证 loss 的变化平均不到 0.006，每一种配置都比普通模型好。而平衡型在每个规模、每种分支数下，都比宽浅型更好。

第三组是最终方案，平衡型加 Joint-KV。12 个 Joint-KV 模型，验证 loss 全部低于普通模型，和不共享缓存的版本基本持平。下游平均准确率，两、三、四分支分别是百分之55.92、55.56 和 55.47，普通模型是百分之54.23。

缓存省了多少，看这张表。最大的 W4 规模，四分支 Joint-KV 的 KV cache 比普通模型小百分之31.8。所有配置里，省得最多的是百分之34 左右。

再看速度。作者在一张 H200 上测 decode 吞吐，也就是不算 prefill，只算生成阶段每秒出多少 token。输入从 4K 到 128K，并发请求从 1 个到最多 64 个。

结果在这张图。四个规模平均下来，四分支 Joint-KV 在所有设置下都比普通模型快，单个 4K 请求时，速度是普通模型的 1.12 倍，16 个 128K 请求并发时是 1.5 倍。两分支的范围是 1.01 到 1.32 倍。

为什么会快。生成一步的时间分两块，一块是依次算完各层的固定开销，分支合并执行，这块就省了。另一块是读 KV cache，缓存变小、又只读一次，这块也省了。所以总体上，输入越长，加速越明显。

### 和以前方法的对比

一个自然的疑问是，好处会不会只是因为模型变宽变浅了。作者做了两种对照，一种把整个模型加宽变浅，另一种把中间两条分支换成一条更宽的单路。

结果在这张图。四个规模下，两分支宽浅型都比这两种加宽的对照好。保留完整深度的窄深型，也在四个规模下都好于普通模型，拟合出的 scaling 指数是几种里最陡的。

KV 共享也做了公平对照。在缓存大小完全相同的条件下，共享一份缓存，比每条分支各存一份，验证 loss 低 0.011 到 0.012，四个规模都是这样，而多出来的计算量不到百分之0.6。

还有两个消融。融合方式上，试了六种带可学习参数的融合，最终 loss 差别不到 0.005，简单的拼接加投影就够了，但去掉投影、直接相加会变差。分支放在哪里，放在中间最好，loss 是 2.445，放在前面是 2.458，放在后面是 2.464。

我的判断是，这篇最有说服力的地方，在于质量和速度一起变好了，四分支 Joint-KV 在每个规模上 loss 都低于普通模型，生成也更快，不是拿质量去换速度。

我的判断是，对照范围偏窄，实验只比了作者自己训练的普通 Transformer 和几种加宽变体，没有和 PaLM 式块内并行、ParScale 这些已有的并行结构直接比。

### 局限

论文没有单独写局限这一节，但从实验设置里，能看出几个边界。

第一是规模。最大的模型总参数 20.7 亿，每个 token 激活的不到 4 亿，数据只有 FineWeb-Edu 一种。到几百亿、上千亿参数时结论还成不成立，论文没有测。

第二是长上下文。模型训练时的上下文只有 4096，测速的时候，每个请求加上生成的 1K token，都超出了训练长度，作者明确说，这些测试只衡量速度，不衡量质量。

第三，加速是有条件的。它只算生成阶段，而且两分支在最小的 W1 规模上，有六个设置反而慢于普通模型，最慢是 0.93 倍，作者的解释是，只少了一层，省下的时间不够抵消融合这些额外操作。

### 未来方向

作者提了三个方向。一是把分支数、分支深度和宽度一起纳入 scaling law，再把推理成本也算进去，找最合适的配置。

二是和 YOCO、跨层注意力这些方法结合，缓存既在分支之间共享，也在层之间共享。三是用到多模态上，让不同分支分别处理文字、图像和音频，各自用不同的宽度和深度。

我的判断是，这篇的价值在于，把块怎么组织，变成了和宽度、深度并列的一个设计维度，而且代码放在了 GitHub 上，Hugging Face 上也有对应的合集，方便别人复现。

我的判断是，真正的考验在更大的规模。二十亿参数这一级，loss 降了 0.03 左右，相当于普通模型要多花两成多的训练算力，到了前沿规模还能剩多少，要看后续有没有人在更大的模型上验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is a collaboration between the University of Illinois Urbana-Champaign, Rice, Michigan and Cornell, and it's called Multi-Branch Transformer.

It asks a very basic question. When you scale a language model, you can make it wider or deeper. Is there a third option? The authors' answer is to branch: run several stacks of full Transformer blocks side by side.

### Problem

Some background first. Today's language models are essentially a chain of Transformer blocks. To make the model stronger, you normally do one of two things: make each block wider, or stack more of them.

Attention, mixture-of-experts and residual connections have all improved a lot in recent years, but the full blocks still run one after another. And when the model generates a token, those serial blocks have to run in order, so more blocks means a bigger fixed cost per step.

The other pressure point is the KV cache, the keys and values the model stores for every token it has already read. The longer the context and the more requests in flight, the more of each step goes into reading that cache.

### Prior work

People have parallelized Transformers before. PaLM, for example, runs the attention and feed-forward layers inside one block side by side. That's parallelism within a block.

Another line of work trades depth for width, using wider, shallower stacks. And methods like ParScale run the same shared backbone several times in parallel, each pass with its own learned prefix.

Multi-branch designs have a long history in computer vision, think Inception or ResNeXt. But the authors argue that composing full Transformer blocks in parallel, as a way to scale modern autoregressive language models, is still underexplored.

### Key idea

The idea fits in one figure. On the left, a standard Transformer, one block after another. On the right, the multi-branch version: a few shared layers up front, then several independent branches running at the same time, then a fusion step, and a few more shared layers at the end.

The shared front part is called the prelude, and the shared back part the coda, terms borrowed from earlier work. Each branch has its own parameters. It's not the same weights run several times.

The key point is that each branch keeps its own sequential depth, while the shared layers after it get to build on the results of several different transformation paths.

### Method

Start with fusion. At each token position, the outputs of the branches are normalized, concatenated, and projected back to the original width with one linear layer. That's all.

Next, how to spend the compute. Holding training compute roughly fixed, the authors define three ways to split a model. This figure uses a six-block example: the first and last blocks are shared, and the middle four become two branches.

Wide-Shallow keeps the original block width and splits the middle blocks across the branches, two per branch. Narrow-Deep keeps the original depth in each branch, four blocks, but makes each block narrower. Balanced sits in between, with three blocks per branch at an intermediate width.

The second piece is called Joint-KV. By default, every block in every branch keeps its own KV cache, so more branches means more cache.

With Joint-KV, all branches at the same depth share one KV cache, built by projecting their concatenated inputs, while each branch keeps its own queries. Compared with one cache per branch, the branch region's cache shrinks by the number of branches, a quarter of the size with four branches, at the same projection parameters and compute.

Note that Joint-KV changes the model itself, so it's used in training. Separately, the authors built an inference engine on top of nano-vLLM that changes only how the model is executed, not the model.

The engine has two tricks. The first is branch grouping: blocks at the same depth across branches run as one step. The paper's example is a four-branch model with twenty-four blocks, which would take twenty-four steps one by one, but only twelve when grouped, against sixteen for a standard Transformer of the same scale.

The second is head folding. If every branch reads the shared cache separately, you haven't actually saved memory traffic. Head folding puts the queries of all branches into a single grouped-query attention call, so the shared cache is read once.

### Results

The setup. The authors train mixture-of-experts models at four scales, W1 to W4, with the standard models ranging from point eight to about two point one billion total parameters, and one hundred thirty-nine to three hundred ninety-four million active body parameters per token. All are trained on FineWeb-Edu with a context length of four thousand ninety-six.

At each scale, the standard model and the multi-branch models see the same number of training tokens, about two hundred per active body parameter. The standard model's learning rate was tuned carefully at every scale, while the multi-branch models were tuned on one configuration and reused that setting elsewhere.

First experiment: two Wide-Shallow branches, varying what share of the blocks goes into the branches. This figure shows a U-shaped curve at all four scales. The best point is around a third of the blocks, which lowers validation loss by zero point zero two five to zero point zero three one compared with the standard model.

Push further, and by the authors' interpolation, you can put about seventy-nine percent of the blocks into branches and still match the standard Transformer.

From the fitted scaling curves, the authors estimate that a standard Transformer needs about one point two three times the training compute to reach the same validation loss. On nine zero-shot downstream tasks, averaged over the four scales, the multi-branch model wins on all nine, and mean accuracy goes from fifty-four point two three to fifty-six point zero three percent.

Second experiment: going from two branches to three and four. Validation loss changes by less than zero point zero zero six on average, and every configuration beats the standard model. Balanced models beat Wide-Shallow ones at every scale and branch count.

Third is the final recipe, Balanced plus Joint-KV. All twelve Joint-KV models have lower validation loss than the standard model, and they closely track the versions without cache sharing. Mean downstream accuracy is fifty-five point nine two, fifty-five point five six and fifty-five point four seven percent for two, three and four branches, against fifty-four point two three for the standard model.

To see how much cache that saves, look at this table. At the largest scale, W4, the four-branch Joint-KV model's KV cache is thirty-one point eight percent smaller than the standard model's. Across all configurations, the biggest saving is about thirty-four percent.

Now speed. The authors measure decode throughput on a single H200, meaning generated tokens per second, excluding prefill. Inputs range from four thousand to one hundred twenty-eight thousand tokens, with anywhere from one to sixty-four requests in flight.

Here are the results. Averaged over the four scales, the four-branch Joint-KV model is faster than the standard model in every setting, from one point one two times with a single four-thousand-token request to one point five times with sixteen concurrent requests at one hundred twenty-eight thousand tokens. Two branches range from one point zero one to one point three two times.

Why is it faster? Each decode step has two costs. One is the fixed cost of running the serial blocks in order, and grouping the branches means fewer serial steps. The other is reading the KV cache, which dominates as the context grows, and a smaller cache that's read once cuts that too. So overall, the longer the input, the bigger the speedup.

### Comparison

A natural question is whether the gain just comes from making the model wider and shallower. The authors test that directly with two controls: widening the whole model and making it shallower, or replacing the two middle branches with a single wider path.

Here's the result. At all four scales, the two-branch Wide-Shallow model beats both widening controls. And the Narrow-Deep model, which keeps the full depth, also beats the standard model at every scale, with the steepest fitted scaling exponent of the group.

Cache sharing gets a fair test too. At exactly the same cache size, one shared cache beats a private cache per branch by zero point zero one one to zero point zero one two in validation loss at all four scales, while adding less than zero point six percent compute.

Two more ablations. Six learned fusion operators land within zero point zero zero five of each other in final loss, so plain concatenate-and-project is enough, though dropping the projection and simply adding the branches makes things worse. As for placement, branching in the middle works best, at two point four four five, versus two point four five eight early and two point four six four late.

My read is that the most convincing part is that quality and speed improve together. The four-branch Joint-KV model has lower loss than the standard model at every scale and also decodes faster, so it isn't trading quality for speed.

My read is also that the comparison set is narrow. The experiments pit the method against the authors' own standard Transformers and widened variants, with no direct experimental comparison to existing parallel designs like PaLM-style within-block parallelism or ParScale.

### Limitations

The paper has no separate limitations section, but the experimental setup marks out a few boundaries.

First, scale. The largest model has about two point one billion total parameters, under four hundred million active per token, trained on a single dataset, FineWeb-Edu. Whether the results hold at tens or hundreds of billions of parameters isn't tested.

Second, long context. The models were trained with a context of four thousand ninety-six tokens, and in the speed tests every request, plus its thousand generated tokens, runs past that length. The authors state plainly that these runs measure speed only, not quality.

Third, the speedup comes with conditions. It covers the generation phase only, and two branches are sometimes slower at small scale: in six settings at W1 they fall below the standard model, down to zero point nine three times. The authors explain that removing a single block saves less time than the extra operations like fusion cost.

### Future work

The authors suggest three directions. First, scaling laws that jointly cover branch count, branch depth and width, with inference cost folded in, to find the best configuration.

Second, combining branches with cross-layer methods like YOCO and cross-layer attention, so the cache is shared across both branches and layers. Third, multimodal models, where different branches could handle text, images and audio, each with its own width and depth.

My read is that the paper's real contribution is making block organization a design axis alongside width and depth, and the code is on GitHub with a matching Hugging Face collection, which makes it easy to check.

My read is that the real test is scale. At around two billion parameters, a loss drop of roughly zero point zero three, which a standard model needs about one point two times the compute to match, is a solid result. How much of that survives at frontier scale depends on someone reproducing it on much larger models.

That's today's Daily Frontier Papers. The paper is linked in the description.

