# On the Design of Hyper-Connections: Simplification, Stability, and Scalability

**多流残差做减法开销降到三分之一**

- 论文 Paper: [arXiv 2611.05984](https://arxiv.org/abs/2611.05984)
- 单位 Affiliation: 上海交通大学 · 小红书 Dots Studio
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-14
- 视频 Videos: [YouTube 中文](https://youtu.be/27rONL2NjU8) · [YouTube English](https://youtu.be/d7AGluYCELY) · [B站](https://www.bilibili.com/video/BV1owsG6dEVY)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇上海交通大学和小红书 Dots Studio 的论文，题目是，关于 Hyper-Connections 的设计，简化、稳定和可扩展。

它研究的是大模型里很不起眼的部件，残差连接。结论是，流行的多流残差结构可以拿掉一块，效果基本不变，训练更省。新设计叫 SimpleHC。

### 问题

先补个背景。残差连接，就是把每一层的输入直接加到它的输出上，这样信息和梯度可以绕过这一层往下传，现代的大语言模型都靠它来训练。

Hyper-Connections，简称 HC，把这一条残差通路，扩成好几条并行的流，论文里用的是四条。每一层先从这几条流里读出自己的输入，叫读门，再把输出按比例写回各条流，叫写门，流和流之间还可以互相混合。

后来的 mHC，用一种叫 Sinkhorn 的归一化去约束这个混合，目的是让很深的网络训练更稳。论文说，mHC 这类结构已经被前沿语言模型采用，它引用的例子是 DeepSeek-V4 和 GLM-5。

问题是贵。每过一个子层，也就是注意力层或前馈层，都要把四条流的完整状态读一遍、写一遍，这些内存读写不体现在计算量上。作者实测，在高效实现下，mHC 增加的计算量不到百分之一，训练时间却多出百分之 7 到 27。

所以论文要回答的问题是，能不能有一个更简单的 HC 设计，执行成本更低，同时保住模型质量和训练稳定性。

### 以前的方法

之前已经有工作发现，把 mHC 的混合矩阵直接换成单位矩阵，也就是不混合，模型质量基本不掉。这个版本叫 identityHC，腾讯 Hy 团队的 Hy4 preview 也用了它。

但论文指出，这类结构用的人越来越多，读门、写门和混合各自起什么作用，却还没弄清楚，开销也一直在。

### 核心想法

作者在一个 150 亿总参数的 MoE 模型上，把三个部件一个一个拆开做实验，结果在表 2。

第一个发现，不加约束的混合确实有用，但加上 Sinkhorn 约束，好处就基本没了。在 mHC 的设置下，不加约束的混合平均分是 56.2，加 Sinkhorn 是 55.0，完全不混合是 55.4。

第二个发现，三个部件其实分成两组。一组是学出来的读门加写门，不混合也能拿到 55.4，固定住其中任何一个，分数都明显下降。另一组是不加约束的混合，就算读门写门都固定死，也能拿到 56.0。两组单独用，都和 mHC 差不多，合起来更好。

那为什么不选分数最高的混合。作者的理由是，不加约束的混合会在层与层之间反复做矩阵乘法，信号可能越乘越大或越乘越小，有训练不稳的风险。他们也观察到，这种配置的残差路径系数峰值更大。

所以 SimpleHC 的第一步，是保留学出来的读门和写门，去掉混合，作者称之为最小有效结构。

第二个想法在执行上。去掉混合以后，残差的更新就变成了简单的累加，这给优化留出了空间。

### 方法

看这张图的右半边。SimpleHC 把连续四个子层分成一组，也就是两个 Transformer 块。

因为更新只是累加，组里后面某一层的输入，可以拆成两部分。一部分是从组入口的状态直接读出来的，叫基础读取。另一部分，是同组前面几层的输出，各乘一个标量系数加上去，这个系数，就是后一层读门和前一层写门的内积，作者叫它标量修正。

这样一来，只要门已经给定，中间状态不用真正写进那四条宽的残差流，靠每层普通宽度的输出就能还原输入。

但还剩一个依赖。每一层的门，本来要根据这一层当时的状态来算，那还是得每层都去读一遍完整状态。

SimpleHC 的做法是，组里四层的门，全部根据组入口那一个状态一次算出来。每层还是有自己独立的门参数，只是输入共享。论文说得很清楚，相对于逐层计算门，这是唯一的近似。

结果是，四次基础读取合并成一次扫描，所有写回推迟到组的边界，一次写完。作者在附录里证明，在使用同样这组门的前提下，这种执行方式和逐层执行在数学上等价。

表 1 算了一笔账。按每个 token、每个子层统计残差流的读写量，融合实现的 mHC 和 identityHC 都是 18C，C 是隐藏层宽度，SimpleHC 是 9C，正好减半。

最后是工程。作者用 DeepGEMM 和 Triton 写了专门的算子。在测试的最大宽度 6144 上，这些算子把相对普通残差的额外训练时间，从 PyTorch 实现的百分之 22.5，降到百分之 2.4。

### 实验和效果

实验设置。主实验是一个 MoE 模型，总参数 150 亿，每个 token 激活 10 亿。再用总参数 300 亿、激活 20 亿的模型验证规模。训练数据是中英文文本、代码和数学的混合。

先看 150 亿模型上的消融，在表 3。SimpleHC 的验证损失是 1.660，和 identityHC 一样，平均分 55.0，和 mHC 持平，比 identityHC 的 55.4 略低。

两个对照很说明问题。如果组里四层共用同一对门，平均分从 55.0 掉到 53.5。去掉标量修正，后面的层看不到同组前面层的输出，损失从 1.660 升到 1.705，平均分只有 47.9，比普通残差的 52.4 还低。

再看 300 亿模型，表 4，一共十四个基准。SimpleHC 的损失是 1.561，和 mHC、identityHC 一样，普通残差是 1.579。平均分 SimpleHC 是 63.1，mHC 是 62.9，identityHC 是 62.8，普通残差是 59.3。

分项看，SimpleHC 在 MMLU、MATH 和 C3 上更高，MATH 是 41.5，mHC 是 38.4。但在 GSM8K 上落后 mHC，在 HumanEval 上两者都不如，45.1 对 47.6 和 48.2。

效率在图 3。用高效实现，SimpleHC 在所有测试宽度上额外开销都最低。最小宽度 1024 时，SimpleHC 是百分之 6.3，mHC 是 26.8，identityHC 是 10.1。最大宽度 6144 时，两个基线都在百分之 7 左右，SimpleHC 是 2.4，大约砍掉三分之二。

作者还和另一种扩展残差流的方法 xHC 比较，在 6144 宽度上，xHC 的额外开销是百分之 27.7。按作者估算，SimpleHC 为反向传播保存的存储，比 mHC 少大约百分之 31。

稳定性看图 6。正常训练 150 亿模型时，热身之后的前期，SimpleHC 的梯度范数波动更小，后期三者差不多。热身之后，一千步滚动标准差的最大值是 0.032，mHC 是 0.206，identityHC 是 0.248。

作者还做了压力测试，用总参数 2300 亿、激活 60 亿的 MoE 模型，学习率开到参考值的 10 倍。mHC 出现了梯度尖峰，最高大约 190 万，SimpleHC 和 identityHC 的梯度范数一直有界。

### 和以前方法的对比

放在一起看。和 mHC 比，SimpleHC 去掉了混合和 Sinkhorn，在 150 亿和 300 亿两个规模上质量持平，开销更低，压力测试里更稳。

和 identityHC 比，两者都不混合，质量和稳定性接近。SimpleHC 多出来的是执行方式，读写量减半，最大宽度下额外训练时间从大约百分之 7 降到 2.4。

我的判断是，这篇最有价值的不只是新结构，而是拆解本身。它说明 mHC 里那层 Sinkhorn 约束，带来的提升很有限，真正撑起效果的，是学出来的读门和写门。

我的判断是，质量上的差距都很小，300 亿模型上平均分只差零点几分，论文里也没看到重复实验的波动范围，所以说持平比说更好更准确。另外，表 2 里分数最高的其实是不加约束的混合，作者为了稳定放弃了它。

### 局限

论文没有单独的局限一节，但写明了几个条件。第一，效率是在八层的模型上测的，作者认为能代表更深模型的相对开销，但不是全深度实测。

第二，那个百分之 31 的节省，是按代码估算的为反向保存的存储，作者明确说，它不是总激活量，也不是峰值显存。

第三，组内的门都由入口状态算出，本身是个近似，附录的等价证明不覆盖逐层计算门的原设计。

第四，作者承认，专用算子需要大量工程投入，会限制推广。纯 PyTorch 实现下，SimpleHC 的开销也是几种里最低的，但在最大宽度上仍有百分之 22.5。

第五，压力测试里 mHC 为什么不稳，作者做了分析，但也说，到底是数学上的敏感，还是反向计算里的数值问题，还没有定论。另外，配置表里没有给出训练用了多少 token。

### 未来方向

作者在附录里还提出一个推测，压力测试中前馈层输出的异常值，可能会被混合进一步放大，这可能解释了 DeepSeek-V4 为什么要对 SwiGLU 做截断。

作者希望 SimpleHC 能推动基础模型设计更多地追求简单，让性能提升在大规模训练里更容易用上。论文给出了 GitHub 代码地址。

我的判断是，对正在训练大模型的团队，如果已经在用 mHC，可以先试着去掉 Sinkhorn 混合，再用分组执行压开销，代价是维护专用算子。接下来要看的，是它在更大规模、更长的训练里，能不能一直和 mHC 持平。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Shanghai Jiao Tong University and Dots Studio at Xiaohongshu. It's called On the Design of Hyper-Connections: Simplification, Stability, and Scalability.

It's about one of the least glamorous parts of a language model, the residual connection. The finding is that a popular multi-stream residual design can lose a big piece of its machinery, keep its quality, and train faster. The authors call the result SimpleHC.

### Problem

Quick background. A residual connection adds a layer's input straight onto its output, so information and gradients can skip past the layer. Modern language models rely on it for training.

Hyper-Connections, or HC, widen that single path into several parallel streams. The paper uses four. Each layer reads its input from the streams through a read gate, writes its output back through a write gate, and the streams can also be mixed with each other.

A later variant, mHC, constrains that mixing with a normalization step called Sinkhorn, to keep very deep networks stable. The paper says mHC-style designs have been adopted in frontier language models, citing DeepSeek-V4 and GLM-5.

The problem is cost. Every Transformer block has two sublayers, attention and the feed-forward part, and at each one the full state of all four streams gets read and written. That memory traffic doesn't show up in FLOP counts. In the authors' measurements with an efficient implementation, mHC adds less than one percent more compute, but seven to twenty-seven percent more training time.

So the question is whether a simpler Hyper-Connections design can cut that execution cost while keeping model quality and training stability.

### Prior work

Earlier work had already found that if you replace mHC's mixing matrix with the identity, meaning no mixing at all, quality mostly holds. That version is called identityHC, and Tencent's Hy4 preview uses it.

But as the paper points out, even as these designs spread, nobody had cleanly worked out what the read gates, write gates and mixing each contribute, and the training overhead stayed.

### Key idea

So the authors take the three components apart, one at a time, on a mixture-of-experts model with fifteen billion total parameters. The results are in Table 2.

Finding one: unconstrained mixing really helps, but the Sinkhorn constraint mostly cancels the gain. Under mHC's setup, unconstrained mixing averages fifty-six point two, Sinkhorn mixing fifty-five point zero, and no mixing fifty-five point four.

Finding two: the three components actually form two groups. One is the learned read and write gates. With no mixing they reach fifty-five point four, and fixing either gate drops the score clearly. The other is unconstrained mixing, which reaches fifty-six point zero even with both gates frozen. Either group alone is about as good as mHC, and combining them is better.

So why not keep the mixing that scores highest? Because unconstrained mixing multiplies earlier contributions by matrix after matrix as you go deeper, so signals can blow up or fade away, which risks unstable training. The authors also see larger peak residual-path coefficients in that configuration than in identityHC.

That gives SimpleHC its first step: keep the learned read and write gates, drop the mixing. The authors call this the minimal effective structure.

The second idea is about execution. Without mixing, residual updates become plain addition, and that opens room to optimize.

### Method

Look at the right half of this figure. SimpleHC groups four consecutive sublayers together, which is two Transformer blocks.

Because the updates just add up, the input to a later layer in the group splits into two parts. One is read straight from the state at the start of the group, the base read. The other is the outputs of earlier layers in the same group, each multiplied by a single number and added on. That number is the inner product of the later layer's read gate and the earlier layer's write gate. The authors call these scalar corrections.

So, given the gates, the intermediate states never have to be written into the four wide streams. Each input can be rebuilt from the ordinary-width outputs of earlier layers.

One dependency remains. Each layer's gates are normally computed from that layer's current state, which would still mean reading the full state at every layer.

SimpleHC computes the gates for all four layers in the group at once, from the single state at the group's entrance. Each layer still has its own gate parameters; only the input is shared. The paper is explicit that, compared with computing gates layer by layer, this is the only approximation.

The result: four base reads become one pass over memory, and all the write-backs are deferred to the group boundary and done once. In the appendix, the authors prove this is mathematically equivalent to running the layers one by one with those same gates.

Table 1 does the accounting. Counting residual-stream reads and writes per token per sublayer, fused mHC and identityHC both come to eighteen C, where C is the hidden width. SimpleHC comes to nine C. Exactly half.

Finally, engineering. The authors wrote dedicated kernels with DeepGEMM and Triton. At the largest width they tested, six thousand one hundred forty-four, those kernels cut the extra training time over a plain residual from twenty-two point five percent in PyTorch to two point four percent.

### Results

The setup. The main model is a mixture-of-experts with fifteen billion total parameters and one billion active per token. Scaling is checked on a thirty billion total, two billion active model. Training data mixes English and Chinese text, code and math. Four residual streams, four sublayers per group.

First, the fifteen billion ablations in Table 3. SimpleHC's validation loss is one point six six zero, the same as identityHC. Its average score is fifty-five point zero, level with mHC and slightly below identityHC's fifty-five point four.

Two controls are telling. If the four layers in a group share one pair of gates, the average drops from fifty-five point zero to fifty-three point five. Remove the scalar corrections, so later layers can't see earlier outputs in the same group, and loss rises from one point six six zero to one point seven zero five. The average falls to forty-seven point nine, below the plain residual baseline at fifty-two point four.

Now the thirty billion model, Table 4, fourteen benchmarks. SimpleHC's loss is one point five six one, the same as mHC and identityHC, against one point five seven nine for a plain residual. Average scores: SimpleHC sixty-three point one, mHC sixty-two point nine, identityHC sixty-two point eight, plain residual fifty-nine point three.

By benchmark, SimpleHC is higher on MMLU, MATH and C3. On MATH it's forty-one point five against mHC's thirty-eight point four. But it trails mHC on GSM8K, and trails both on HumanEval: forty-five point one, against forty-seven point six for mHC and forty-eight point two for identityHC.

Efficiency is in Figure 3. With efficient implementations, SimpleHC has the lowest overhead at every width tested. At the smallest width, one thousand twenty-four, it's six point three percent, against twenty-six point eight for mHC and ten point one for identityHC. At the largest width, both baselines sit around seven percent and SimpleHC is at two point four, cutting about two-thirds.

The authors also compare against xHC, another way of scaling residual streams. At the largest width, its overhead is twenty-seven point seven percent. And by the authors' estimate, SimpleHC keeps about thirty-one percent less storage for the backward pass than mHC.

Stability is in Figure 6. In normal fifteen billion training, SimpleHC's gradient norm fluctuates less early after warmup, and all three end up in a similar range later. After warmup, the largest thousand-step rolling standard deviation is zero point zero three two for SimpleHC, zero point two zero six for mHC, and zero point two four eight for identityHC.

Then a stress test: a mixture-of-experts model with two hundred thirty billion total and six billion active parameters, trained at ten times the reference learning rate. mHC shows gradient spikes reaching roughly one point nine million. SimpleHC and identityHC keep their gradient norms bounded.

### Comparison

Putting it together. Against mHC, SimpleHC drops the mixing and Sinkhorn, matches quality at both fifteen and thirty billion parameters, costs less to train, and stays stable under stress.

Against identityHC, neither one mixes the streams, and quality and stability are close. What SimpleHC adds is the execution scheme: memory traffic from eighteen C to nine C, and overhead at the largest width from about seven percent to two point four.

My read is that the most valuable part isn't the new design so much as the dissection. It shows that mHC's Sinkhorn constraint buys very little, and that the learned read and write gates are what carry the quality.

My read is also that the quality differences are small. At thirty billion, the averages are a few tenths of a point apart, and I didn't see run-to-run variance reported in the paper, so "on par" is a fairer summary than "better". And in Table 2 the top score actually belongs to unconstrained mixing, which the authors gave up for stability.

### Limitations

There's no dedicated limitations section, but the text and appendix spell out some conditions. First, efficiency was measured on eight-layer models. The authors argue that represents the relative overhead in deeper models, but it isn't a measurement at full depth.

Second, the thirty-one percent saving is a source-level estimate of storage kept for the backward pass. The authors state plainly that it isn't total activation memory or peak device memory.

Third, computing a group's gates from its entry state is itself an approximation. The appendix's equivalence proof covers only these shared-input gates, not the original design that computes gates layer by layer.

Fourth, the authors acknowledge that specialized kernels take substantial engineering effort, which limits adoption. In plain PyTorch, SimpleHC still has the lowest overhead of the variants, but at the largest width that's still twenty-two point five percent.

Fifth, the authors analyze why mHC destabilizes under stress, but they say it's unresolved whether that's mathematical sensitivity or a numerical effect in the backward pass. Also, the configuration table doesn't give the number of training tokens.

### Future work

In the appendix the authors also offer a suggestion: under stress, outliers from the feed-forward layers could be further amplified by mixing, which may explain why DeepSeek-V4 clamps its SwiGLU activations.

The authors hope SimpleHC pushes foundation model design toward simplicity, making these gains easier to use at scale. The paper links its code on GitHub.

My read for teams training large models: if you already run mHC, try dropping the Sinkhorn mixing first, then use grouped execution to squeeze the overhead, at the price of maintaining custom kernels. The thing to watch is whether it keeps pace with mHC at larger scales and over longer training runs.

That's today's Daily Frontier Papers. The paper is linked in the description.

