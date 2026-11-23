# Clover-35B-A3B-Preview Technical Report

**扩散模型也能写代码修 bug**

- 论文 Paper: [arXiv 2611.24636](https://arxiv.org/abs/2611.24636)
- 单位 Affiliation: DiffuSpace
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-23
- 视频 Videos: [YouTube 中文](https://youtu.be/wIWc72U8bAo) · [B站](https://www.bilibili.com/video/BV1bryDzpEoB)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 DiffuSpace 团队的技术报告，主角是一个扩散语言模型，叫 Clover。

它是混合专家结构，总参数 350 亿，每个 token 只激活 30 亿，支持思考模式。报告说，权重、微调代码和推理引擎都已公开。

### 问题

先说什么是扩散语言模型。平时用的大模型是自回归的，一个 token 接一个往后写。扩散模型是先放一串被遮住的位置，一轮一轮去噪，每轮同时猜出好几个 token。

它的吸引力在速度。但到了 Agent 场景，要调工具、改代码、在终端里一步步做任务，开源的扩散模型一直不够强。

报告在同一套评测下做了复现。LLaDA2.2 flash 和 DiffusionGemma 这两个公开的扩散模型，在 Terminal-Bench 2.1 和 SWE-bench Verified 上，都落后于评测的 Qwen 自回归模型。比如 SWE-bench Verified，DiffusionGemma 是 23.8，Qwen3.6 是 65.8。

所以这篇要回答的是，扩散模型能不能把 Agent 能力做上去，同时保住并行生成的速度。

### 以前的方法

一条相关的路是块扩散，把输出切成一块一块，块之间按顺序，块内部并行去噪。Clover 也用这个框架。

另一条是加速，Fast-dLLM 这类方法，靠缓存，加上按置信度一次确定多个 token。

还有投机解码，先用便宜的办法起草几个 token，再让大模型一次验证，通过的直接收下。DFlash 就是用扩散的方式，给自回归模型打草稿。

再进一步是自投机解码，起草和验证在同一个模型里，扩散起草，自回归验证。英伟达的 Nemotron-Labs-Diffusion 就支持这几种解码方式。

### 核心想法

Clover 不是从零训练的。它拿 Qwen3.6-35B-A3B 的文本主干，改造成块扩散模型。这个主干本身是混合结构，一部分层是 Gated DeltaNet，一种线性的循环层，其余是全注意力。

Gated DeltaNet 把前面所有内容压进一个固定大小的记忆矩阵，不用像注意力那样，每个 token 存一份缓存。但它是单向的，只看左边。

作者的思路是分工。写完的历史，交给循环记忆压缩，正在去噪的这一块，块里的 token 用双向注意力互相看。

报告说，他们优先保证当前块内部的双向交互，因为这正是扩散模型的长处。

### 方法

整体结构是 40 层，每 3 层 BD-GDN，也就是块扩散版的 Gated DeltaNet，接 1 层全注意力，重复 10 次，每层后面接一个混合专家模块。

BD-GDN 怎么同时用历史和当前块。先把这一块之前的记忆读出来，变成一个历史向量，这份记忆在整块去噪过程中保持不变。

然后，这个历史向量和块里每个 token 一起，放进同一个 softmax 里抢注意力权重。每个位置自己决定，多看压缩的历史，还是多看块里的 token。

有个细节，历史只占一个位置，块越大，历史越容易被挤掉。所以作者加了一个可学习的块大小校准项，让历史的权重不会因为块变大而被稀释。

第二个改动在卷积。原来有一个只看左边的短卷积，作者保留它，再加一个只看右边 3 个位置的卷积核，初始值全是 0。

这样训练刚开始，这个卷积和原来完全一样，右边的信息慢慢学进来。这些改动一共只新增了大约 74 万个参数。

第三个改动在混合专家，每个模块有 256 个路由专家。一整块并行预测时，token 可能分散到很多专家上，所以先给整块挑一个专家池，覆盖路由概率的百分之九十，每个 token 再从池子里选 8 个专家。

训练时，每条样本放两份，加噪的那份学去噪，干净的那份学普通的下一个 token 预测。块大小从 4、8、16、32 里随机选。

生成时用自投机解码。扩散这一路一次起草一整块，再用单向的那一路逐个验证，遇到第一个不通过的就停，只收下前面的部分。

验证门槛按不确定性调整，模型越确定，门槛越严。另外，扩散解码偶尔会重复，所以还加了重复惩罚。

### 实验和效果

先看能力，表 1。对比对象是三个扩散模型，LLaDA2.2 flash、DiffusionGemma、Mercury 2.5，加上同为 35B-A3B 的自回归模型 Qwen3.6 和 Qwen3.5，全部在作者环境里按同一流程复现。

七项 Agent 指标，Clover 全部第一。SWE-bench Verified 是 73.6，它的底座 Qwen3.6 是 65.8。Terminal-Bench 2.1 是 51.7，Qwen3.6 是 43.1。

但不是每项都领先。高难度的知识题 HLE，Clover 是 12.8，Qwen3.6 是 21.2。AIME 2026 数学竞赛，Clover 是 83.3，两个 Qwen 都是 90.4。

再看速度，表 2。单张 H200，批大小为 1，对手是 Qwen3.6 加 DFlash 2。Agent 回放任务上，Clover 每轮验证平均收下 8.2 个 token，对手是 4.2 个，每秒解码 556.6 个 token，对手是 396.7。

日常聊天上 Clover 也更快，每秒 356.7 对 286.5。但在 MATH-500 上反而慢，465.2 对 604.3。作者的解释是，Clover 起草的成本更高，抵消了优势。

摘要里的每秒 680 个 token，来自图 4 的端到端测试。64 条 Agent 回放，单张 H200 每秒 515 个，GB300 是 680 个。要注意，回放只测生成速度，不执行命令，也不看任务完成没有。

消费级显卡上，一张 16GB 的 RTX 5070 Ti，专家权重量化到 INT4，按路由统计把常用专家放 GPU，其余放 CPU，MATH-500 准确率百分之 97.6，每秒 97.52 个 token。表里的 Qwen3.6 用的是另一套设置，百分之 93.8，每秒 47.04 个。

最后是一个变体，Clover 决策模型，计划之后发布。给它一份材料和最多 8 个相关问题，一次去噪同时填出所有答案。

在内部的 500 组题上，一起答比分开答，整组全对的比例从百分之 65.4 提到百分之 77。4 乘 4 数独上，整盘全对从百分之 22 提到百分之 76。8 个问题一起答，平均延迟 164 毫秒，依次单独调用要 1243 毫秒。

### 和以前方法的对比

和以前的扩散模型比，差距很明显。SWE-bench Verified 上，LLaDA2.2 flash 是 49.3，商用的 Mercury 2.5 是 57.8，Clover 是 73.6。

和 DFlash 这种单独训练的草稿模型比，作者认为，Clover 的起草能力是跟主模型一起训练的，而单独的小草稿模型容量有限，训练提示也是挑过的，不一定覆盖所有任务。

作者也提醒，这些速度对比是系统级的，模型、验证方式、解码策略都不同，没法单独归功于起草算法。而且不加惩罚时 Clover 更快，但有时会输出重复内容。

我的判断是，这篇最有价值的，是给出了一个数据点，扩散模型在同一套评测下，Agent 编码任务超过了自己的自回归底座，而且权重公开，可以复现。

我的判断是，它也留下了问题。相对 Qwen3.6 的提升，来自扩散架构还是来自后续训练，报告没有做分开的消融，训练数据的规模和来源，正文也没交代。

### 局限

作者自己提到几条局限。第一，难的知识题、竞赛数学和单独写代码，还需要改进。

第二，表 3 和其他端侧模型对比时，只有 Clover 和 MiniCPM-V 4.7 是作者自己测的，其他分数引自已发布的报告，评测设置不同，不能直接横比。

第三，决策模型的内部题集，开发中反复用过，只能当能力检查，不算盲测。

第四，一次去噪同时出答案，并不保证所有答案都满足约束。

### 未来方向

作者的下一步，是继续降低延迟，在对齐的条件下讲清楚质量和速度的取舍，加强量化部署，还要扩展多模态和上下文长度。

我的判断是，Clover 说明扩散语言模型可以从强的自回归底座改造出来，不一定从头训练，这条路如果站得住，别的团队也能拿现有开源模型来试。

我的判断是，接下来要看第三方复测，尤其是速度，报告里的数字对设置很敏感，换个任务，开不开惩罚，结果可能差很多。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is a technical report from the DiffuSpace team, about a diffusion language model called Clover.

It's a mixture of experts: thirty-five billion parameters in total, three billion active per token, with a thinking mode. The report says the weights, fine-tuning code and inference engine are all released.

### Problem

First, what's a diffusion language model? Most models we use are autoregressive: one token at a time, left to right. A diffusion model starts with a run of masked positions and cleans them up over several passes, guessing several tokens at once.

The appeal is speed. The problem is agents: calling tools, editing code, working step by step in a terminal. That's where open diffusion models have lagged.

The report reruns the comparison under one protocol. Two public diffusion models, LLaDA2.2 flash and DiffusionGemma, both trail the evaluated Qwen autoregressive models on Terminal-Bench 2.1 and SWE-bench Verified. On SWE-bench Verified, DiffusionGemma scores twenty-three point eight, and Qwen3.6 scores sixty-five point eight.

So the question is whether a diffusion model can get good at agent work and still keep the speed of parallel generation.

### Prior work

One related line is block diffusion: split the output into blocks, go through the blocks in order, and denoise inside each block in parallel. Clover uses this setup.

Another is acceleration. Methods like Fast-dLLM use caching, and commit several tokens at once when the model is confident.

Then there's speculative decoding. Something cheap drafts a few tokens, the big model checks them in one pass, and whatever passes is kept. DFlash drafts for an autoregressive model using diffusion-style block prediction.

One step further is self-speculative decoding, where one model does both: diffusion to draft, autoregression to verify. NVIDIA's Nemotron-Labs-Diffusion supports these decoding modes in a single model.

### Key idea

Clover isn't trained from scratch. It takes the text backbone of Qwen3.6-35B-A3B and converts it into a block diffusion model. That backbone is already a hybrid: some layers are Gated DeltaNet, a linear recurrent layer, and the rest are full attention.

Gated DeltaNet squeezes everything so far into a fixed-size memory matrix, instead of caching every token the way attention does. But it only looks left.

The authors' idea is a division of labor. Finished history goes into the recurrent memory. Inside the block being denoised, tokens see each other in both directions through attention.

The report says they prioritized two-way interaction inside the current block, because that's what diffusion is good at.

### Method

The layout is forty layers: three BD-GDN layers, the block-diffusion version of Gated DeltaNet, then one full-attention layer, repeated ten times, each followed by a mixture-of-experts module.

So how does BD-GDN combine history with the current block? The memory from before the block is read out as one history vector, and that memory stays fixed while the block is denoised.

Then the history vector competes, alongside every token in the block, inside a single softmax. Each position decides how much to lean on compressed history and how much on the block's tokens.

There's a catch: history takes one slot, so in a big block it gets crowded out. A learnable block-size calibration keeps history's share from being diluted as blocks grow.

The second change is in the convolution. The original layer has a short convolution that only looks left. They keep it, and add a separate kernel that looks three positions to the right, initialized to zero.

So the convolution starts exactly where the original was, and learns right-hand context gradually. All of these changes add only about seven hundred and forty thousand parameters.

The third change is in the experts, two hundred fifty-six routed experts per module. When a whole block is predicted at once, its tokens can scatter across many experts. So Clover first picks a pool for the block, covering ninety percent of its routing probability, and each token then picks eight experts from that pool.

In training, every example appears twice. The noised copy learns denoising, and the clean copy learns ordinary next-token prediction. Block size is drawn from four, eight, sixteen or thirty-two.

Generation uses self-speculative decoding. The diffusion path drafts a whole block in one step, the one-directional path checks the tokens in order, and it stops at the first rejection, keeping only what came before.

The acceptance bar adapts to uncertainty: the more confident the check, the stricter the bar. And because diffusion decoding sometimes repeats itself, they add repetition penalties.

### Results

Capability first, in Table 1. The comparison set is three diffusion models, LLaDA2.2 flash, DiffusionGemma and Mercury 2.5, plus the autoregressive Qwen3.6 and Qwen3.5, both also 35B-A3B. Everything is rerun in-house under one protocol.

On all seven agent metrics, Clover comes first. SWE-bench Verified: seventy-three point six, against sixty-five point eight for its base, Qwen3.6. Terminal-Bench 2.1: fifty-one point seven, against forty-three point one.

It doesn't lead everywhere, though. On HLE, a very hard knowledge test, Clover gets twelve point eight and Qwen3.6 gets twenty-one point two. On AIME 2026 math, Clover has eighty-three point three, and both Qwen models have ninety point four.

Now speed, in Table 2. One H200, batch size one, against Qwen3.6 with DFlash 2. On agent replay, Clover keeps eight point two tokens per verification round on average, versus four point two, and decodes about five hundred fifty-seven tokens per second, versus about three hundred ninety-seven.

On daily chat, Clover is faster too, about three hundred fifty-seven against two hundred eighty-seven. But on MATH-500 it's slower, about four hundred sixty-five against six hundred four. The authors say Clover's drafting simply costs more, which eats the gain.

The six hundred eighty tokens per second in the abstract comes from Figure 4, an end-to-end test. On sixty-four agent replays, one H200 gives five hundred fifteen tokens per second, and one GB300 gives six hundred eighty. Keep in mind that replay only measures generation speed: no commands run, and task success isn't measured.

On a consumer GPU, a sixteen-gigabyte RTX 5070 Ti, with expert weights in INT4 and the most-used experts placed on the GPU by routing statistics, Clover scores ninety-seven point six percent on MATH-500 at about ninety-seven and a half tokens per second. Qwen3.6, with a different setup in the same table, gets ninety-three point eight percent at about forty-seven.

Finally, a variant called the Clover decision model, planned for a later release. Give it shared material and up to eight related questions, and it fills in all the answers in one denoising pass.

On an internal set of five hundred groups, answering jointly lifts the share of groups with every answer right from sixty-five point four to seventy-seven percent. On four-by-four Sudoku, fully solved boards go from twenty-two to seventy-six percent. With eight questions, joint answering averages one hundred sixty-four milliseconds, against twelve hundred forty-three for separate calls one after another.

### Comparison

Against earlier diffusion models, the gap is clear. On SWE-bench Verified, LLaDA2.2 flash scores forty-nine point three, the commercial Mercury 2.5 fifty-seven point eight, and Clover seventy-three point six.

Against separate drafters like DFlash, the authors argue Clover's drafting is trained along with the model itself, while a small separate drafter has limited capacity and is trained on chosen prompts, so it may not cover every workload.

The authors also caution that these are system-level speed comparisons: models, verification and decoding policies all differ, so the gain can't be pinned on the drafting algorithm alone. And without penalties Clover is faster, but it sometimes produces repetitive output.

My read is that the most valuable thing here is a data point: under one shared evaluation, a diffusion model beats its own autoregressive base on agentic coding, with open weights so others can check it.

My read is that it also leaves a question open. Does the gain over Qwen3.6 come from the diffusion architecture or from the extra training? There's no ablation separating the two, and the main text doesn't say how much training data was used or where it came from.

### Limitations

The authors name several limitations. First, hard knowledge questions, competition math and standalone coding still need work.

Second, in Table 3, which compares on-device models, only Clover and MiniCPM-V 4.7 were run in-house. The other scores are taken from published reports, with different settings, so they aren't directly comparable.

Third, the decision model's internal test sets were used repeatedly during development, so they're capability checks, not blind tests.

Fourth, answering everything in one pass doesn't guarantee the answers satisfy every constraint.

### Future work

Next, the authors want lower latency, a clear quality-versus-latency trade-off under matched conditions, stronger quantized deployment, plus multimodal capabilities and longer context.

My read is that Clover shows a diffusion language model can be converted from a strong autoregressive base rather than trained from scratch. If that holds up, other teams can try it with open models they already have.

My read is that the real test is third parties rerunning it, speed especially. The numbers are sensitive to settings, and a different workload, or penalties on or off, can change them a lot.

That's today's Daily Frontier Papers. The paper is linked in the description.

