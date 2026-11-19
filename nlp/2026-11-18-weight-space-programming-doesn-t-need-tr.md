# Weight-Space Programming Doesn't Need Transformers

**让AI亲手写权重它不选Transformer**

- 论文 Paper: [arXiv 2611.22055](https://arxiv.org/abs/2611.22055)
- 单位 Affiliation: 浦项科技大学（POSTECH）
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-18
- 视频 Videos: [YouTube 中文](https://youtu.be/H7AySc7e528) · [B站](https://www.bilibili.com/video/BV1Fyst65E9H)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇韩国浦项科技大学的论文，标题有点挑衅，叫 Weight-Space Programming Doesn't Need Transformers，意思是，权重空间编程不需要 Transformer。

先说什么是权重空间编程。正常的模型，权重是靠梯度下降，从数据里一点点调出来的。权重空间编程不训练，也不拟合数据，而是让一个大模型直接动手，把另一个模型的权重写出来。

论文问的是，既然权重是大模型手写的，还有必要用 Transformer 吗。作者发现，让大模型自己选结构，三个大模型都没有选 Transformer。

### 问题

按作者的说法，Transformer 能成为主流，很大程度上是因为它适合梯度下降，在 GPU 上算得快。

但如果权重是大模型写出来的，这个理由就不一定成立了。这张表把两种方式放在一起。梯度下降里，是梯度决定一条知识放进哪些权重，想改一条，得重新训练。权重空间编程里，是写的人决定，改一条知识，改一行就行。

### 以前的方法

以前相关的工作，一类是先固定结构，再研究能往里面写什么，比如把程序编译成 Transformer 的权重，或者手工构造存事实的 MLP。

另一类是编辑训练好的模型，比如定位并修改 GPT 里的某条事实，这类方法还要用梯度优化出来的目标向量。

这篇论文的不同在于，结构不固定，交给大模型自己设计，再和同一个大模型写的 Transformer 比。

### 核心想法

实验是这样设计的。三个前沿大模型当写手，Claude Opus 5.5、GPT-6.1 Sol 和 GPT-6 Astra。每个写手只能看到 10 道带答案的 ARC 题，ARC 是一套四选一的科学选择题。

写手在断网的沙盒里干活，看不到任何测试分数，可以自己编练习题试，但不能用数据去拟合权重。3530 道测试题，在沙盒外面打分。

对比两种设定。一种是结构固定，写手往 15 个公开架构里写权重，包括 Transformer、状态空间模型和 RNN，还有一个作者给的两层小 Transformer。另一种是结构放开，写手自己设计模型。

结果，三个写手不约而同，做出了同一种东西，一张稀疏的事实表，加一个固定的查表流程。没有注意力，没有 MLP，也没有稠密的词向量。

### 方法

这张图，左边是给定的两层 Transformer，每一块都是稠密矩阵。中间是写手设计的模型。右边是作者后来自己设计的模型。

以 Claude Opus 5.5 的第一代模型为例。它先用大白话写下科学事实，再写脚本，把每句话编译成表里的一行。36 万多个词被归并成 3368 个概念，越少见的概念权重越高。事实表一共 4288 行。

答题时，把题目和每个选项都映射成概念，跟每一行比对，保留匹配最好的几行，再做第二轮扩展，最后用几条规则处理否定词、绝对化的词和语序。

看一个例子。测试题问，哪种适应性特征帮助鸟飞行。正确选项是中空的骨头，它匹配到写手写的一句话，中空的骨头让鸟变轻，所以能飞，支持度 70，三个错误选项最好的只有 13 到 22。

为什么选这种结构。论文的解释是，不用梯度写权重，要能调试，就需要两个性质。一是答案可追溯，出错了能找到是哪一行。二是修改是局部的，改一行，只影响依赖它的那几道题。

Claude Opus 5.5 在运行中也写下了类似的理由，大意是，不训练，把知识放进模型的唯一办法就是写进去，而稀疏的检索模型，能让每个数字都追溯到它写的某句话。

作者用联想记忆来解释。Transformer 也可以看成一叠联想记忆，但它的键是稠密的，彼此重叠，取的时候会混进别的记忆。写手的表里，每个键大约只有 7 个非零概念，很少重叠，再只取前几名，剩下的重叠也去掉了。

但这种表每道题都要把所有事实打一遍分，事实越多越慢。所以作者设计了一个联想记忆模型，前向计算由作者固定，权重仍然由大模型一行一行写。

它有两张表。概念表 A，记录一个概念单独能联想到什么。配对表，记录一对概念一起能联想到什么。每对概念的行号用哈希公式直接算出来，不用搜索。

每道题只取最强的 16 个概念，两两组成 120 对，读配对表的 120 行，再和概念表做一次乘法，就得到选项分数。不管写了多少事实，每道题的计算量都固定。

同一道鸟的题，适应和飞这一对概念，直接指到配对表的一行，联想到中空和骨头。正确选项得 52.1 分，第二名只有 15.8 分。

### 实验和效果

主要结果在表 3。先看 Claude Opus 5.5。写进 15 个公开架构，平均准确率 43.1%。写给定的两层 Transformer，36.1%。自己设计的模型，第一代 50.8%，迭代到第五代 54.4%。作者的联想记忆模型，第一代 49.6%，第五代 52.4%。

另外两个写手，自己设计的模型也都明显高于它们手写的两层 Transformer。GPT-6.1 Sol 自己设计的模型迭代到第七代是 47.4%，两层 Transformer 只有 36.3%。GPT-6 Astra 是 46.2% 对 32.6%。

参数量差得更远。7B，17.2 亿参数，测试准确率 48.7%。它自己设计的第一代模型只有 94 万参数，准确率 50.8%，参数少了大约 1800 倍。

跟梯度下降比，同样从随机初始化开始，用全部 3358 道训练题，最好的公开架构 Mamba-2 只有 24.6%，跟瞎猜差不多。同规模的两层 Transformer 是 29.8%，同规模最好的小网络是一个 MLP，32.7%。

但上限要看清。预训练过、再在 ARC 上微调的 Qwen3.5-0.8B 是 82.1%。作者也说，预训练模型仍然准确得多，他们的结论只关于手写权重时该搭什么结构。

再看速度，表 7。在一张 RTX 3090 上答完 3530 道题，写手设计的第一代模型要 0.566 秒，比同样大小的 Transformer 慢 11 倍，第五代慢 90 倍。作者的联想记忆模型只要 0.018 秒，比同样大小的 Transformer 快 7 倍到 17 倍，准确率也更高。

论文还验证了前面那两个性质。只保留每个选项最匹配的一条事实，准确率从 50.8% 只掉到 49.1%。随机删掉一行再重建，表格模型平均只改变 0.5 到 1.8 道题的答案。而 Claude 手写的第一个 Transformer，删一行平均改变 4.8 道题。

还有一个关键对照。准确率会不会只来自写下的知识，跟结构无关。作者拿写手写的原文，直接跑普通的关键词检索 BM25 来比。Claude 手写的两层 Transformer，平均比在自己原文上做 BM25 还低 8 个百分点，表格模型则都比 BM25 高。作者认为，这和稠密的键互相干扰是一致的。

### 和以前方法的对比

作者也查了别的解释。写下的文字会不会抄了测试题。论文报告，它和测试集的重合程度，跟和没见过的同类题差不多，删掉重合的行，准确率几乎不变。

还有一个解释作者排除不了。大模型本来就熟悉检索，有一个写手还把 IDF 检索称为 ARC 的经典基线。所以选表格，可能是因为适合手写，也可能只是因为熟悉。

我的判断是，这篇最有意思的不是表格模型多强，而是换了一个问题。架构是被训练方法和硬件一起选出来的，训练方法换了，最合适的结构也可能跟着换。

我的判断是，这个对比也有边界。表格模型赢的是同一个写手手写的 Transformer，不是训练好的 Transformer。跟预训练再微调的小模型 Qwen3.5-0.8B 比，54.4% 对 82.1%，差距还很大。速度也只在一张 RTX 3090 上测过。

### 局限

第一条局限，这些模型本质上是在回忆写手写下的事实，不会把几条事实串起来推理。ARC 难的 Challenge 部分，准确率一直在 38% 左右，简单的 Easy 部分能到 63%。为 ARC 写的模型拿去做 MMLU，就是瞎猜水平。

第二，换题目。在同水平的科学题 SciQ 上，第五代是 50.1%，在 OpenBookQA 上只有 30% 到 36%。图像上，直接从原始像素写权重，CIFAR-10 只有 21% 到 28%，作者的解释是，一个像素上的权重，写手说不出它是什么意思。

第三，速度优势有边界。作者的模型时间不随事实数量增长，但会随概念数量增长。概念表稠密存储时，概念数到大约 1.5 万到 3.2 万，就会比同样大小的 Transformer 慢。另外，不需要 Transformer 这个结论，只适用于从随机初始化、不跑数据来写权重的情况。

### 未来方向

作者提了三个方向。把计算和比较也写进权重，用手写的模型做起点再做梯度下降，以及加入语序，让模型能生成文本。

还有一个结果挺实用。在 AG News 新闻分类上，用手写的网络初始化，再用 100 篇带标签的文章训练，准确率最高 84.7%，从随机初始化最多 63.6%。作者认为，标签少的场景可以考虑这种做法。

我的判断是，值得继续看的是，这种能读懂、能逐行改的手写模型，能不能接上梯度训练和表示学习。如果能，它可能变成一种可以逐行审计的小模型。如果不能，它可能就停在一个设计得很巧的检索系统。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from POSTECH, the Pohang University of Science and Technology in Korea, and the title is a provocation: Weight-Space Programming Doesn't Need Transformers.

First, what weight-space programming means. Normally a model gets its weights from gradient descent, nudged into place by data. Weight-space programming skips that. There's no training and no fitting to data. Instead, a large language model sits down and writes the weights of another model directly.

The question the paper asks is simple. If an LLM is writing the weights by hand, should it still be building a Transformer? What the authors found is that when three frontier LLMs were free to choose, none of them built one.

### Problem

The authors' argument starts with why Transformers won in the first place. In their telling, a big part of it is that the architecture suits gradient-based training and runs fast on GPUs.

But if the weights come from an LLM instead of a gradient, that reason may no longer apply. This table puts the two side by side. With gradient descent, the gradient decides which weights hold a fact, and changing one fact means training again. With weight-space programming, the writer decides, and changing one fact means editing one line.

### Prior work

Related work comes in a couple of flavors. One fixes an architecture first and asks what can be written into it, for example compiling programs into Transformer weights, or hand-constructing MLPs that store facts.

The other edits a model that's already trained, like locating and changing a specific fact inside GPT. Those methods start from a trained model and rely on target vectors optimized with gradients.

What's new here is that the structure isn't fixed. The LLM designs it, and the result is compared against Transformers written by that same LLM.

### Key idea

Here's the setup. Three frontier LLMs act as writers: Claude Opus 5.5, GPT-6.1 Sol, and GPT-6 Astra. Each writer sees just ten labeled questions from ARC, a benchmark of four-option science questions.

The writer works in a sandbox with no network access and never sees a test score. It can make up practice questions and run its model on them, but it isn't allowed to fit weights to data. The 3,530 test questions are scored outside the sandbox.

There are two regimes. In the first, the architecture is fixed. The writer edits the weights of fifteen released architectures, including Transformers, state-space models and RNNs, starting from random initialization, plus a small two-layer Transformer from the authors where the writer sets every weight. In the second regime, the writer designs the model itself.

And all three writers landed on the same kind of thing: a sparse table of facts, read by a fixed lookup procedure. No attention, no MLPs, no dense embeddings.

### Method

In this figure, the left side is the given two-layer Transformer, where every block is a dense matrix. In the middle is the LLM-designed model. On the right is a model the authors built later, which we'll get to.

Take Claude Opus 5.5's first-generation model. It writes science facts as plain sentences, then writes a build script that compiles each sentence into one row of a table. Some three hundred sixty thousand words are grouped into three thousand three hundred sixty-eight concepts, one concept per meaning, with rarer concepts weighted higher. The fact table has four thousand two hundred eighty-eight rows.

To answer a question, the model maps the question and each option to concepts, matches them against every row, keeps the best-matching rows, runs a second pass with expanded concepts, and then applies a few rules for negation, absolute words, and word order.

Here's an example. The test question asks which adaptation helps a bird fly. Option D is hollow bones, and it matches a sentence the writer wrote: hollow bones make birds light so they can fly. That row reaches a support of seventy. The best rows for the three wrong options only get thirteen to twenty-two. So the answer traces straight back to that one sentence.

Why would a writer choose this structure? The paper's explanation is that debugging weights without gradients needs two properties. First, traceable answers: each prediction comes from a few identifiable lines, so when it's wrong, you can find the line responsible. Second, local edits: change one line, and only the questions that depend on it change.

Claude Opus 5.5 actually wrote down something like this during its runs. Roughly: without training, the only way to get knowledge into the model is to write it, and a sparse retrieval model makes every number traceable to a sentence it wrote.

The authors frame this in terms of associative memory. A Transformer can also be read as a stack of associative memories: a query recalls the values whose keys it resembles. The difference is that Transformer keys are dense and overlap, so recall mixes in other memories. In the written table, each key has only about seven non-zero concepts, so keys rarely overlap, and a hard top-k selection throws away whatever overlap is left.

But the table has a cost. It scores every fact for every question, so it slows down as facts are added, and it launches lots of GPU kernels. So the authors designed their own associative-memory model. They fix the forward pass, and the LLM still writes the weights, line by line.

It has two tables. Concept table A stores which concepts a single concept recalls. A pair table stores what a pair of concepts recalls together. Each pair's row number is computed directly with a hash formula, so there's no search.

For each question, the model takes the sixteen strongest concepts, which form a hundred and twenty pairs, reads those hundred and twenty rows of the pair table, and does one product with the concept table to score each option. However many facts are written, the work per question stays the same.

On the same bird question, the pair adaptation and fly points straight to one row of the pair table, and that row recalls hollow and bones. Option D ends up at fifty-two point one, against fifteen point eight for the runner-up.

### Results

The main results are in Table 3. Start with Claude Opus 5.5. Writing into the fifteen released architectures averages forty-three point one percent. Writing the given two-layer Transformer gets thirty-six point one. Its own designed model gets fifty point eight in the first generation and fifty-four point four by the fifth. The authors' associative-memory model gets forty-nine point six, then fifty-two point four.

For the other two writers, the designed models also clearly beat the two-layer Transformers they wrote. GPT-6.1 Sol's designed model reaches forty-seven point four by its seventh generation, against thirty-six point three for its two-layer Transformer. GPT-6 Astra: forty-six point two against thirty-two point six.

The size gap is even larger. Among the released architectures Claude wrote, the best by validation accuracy was Qwen3-1.7B, with one point seven two billion parameters and forty-eight point seven percent test accuracy. Claude's first-generation designed model has zero point nine four million parameters and scores fifty point eight. That's about eighteen hundred times fewer parameters.

What about gradient descent? Starting from the same random initializations and using all three thousand three hundred fifty-eight training questions, the best released architecture, Mamba-2, gets twenty-four point six percent, right around the twenty-five percent you'd get by guessing. A two-layer Transformer of the designed model's size gets twenty-nine point eight, and the best small network of that size, an MLP, gets thirty-two point seven.

But keep the ceiling in view. The best pretrained model fine-tuned on ARC does far better: Qwen3.5-0.8B scores eighty-two point one. The authors say so themselves: pretrained models remain more accurate, and their claim is only about what structure to build when you're writing weights.

Now speed, in Table 7. Claude's first-generation designed model takes zero point five six six seconds to answer all three thousand five hundred thirty questions on one RTX 3090. That's eleven times slower than a Transformer of the same size, and the fifth generation is ninety times slower. The authors' associative-memory model takes eighteen milliseconds. At the first-generation size it's seven times faster than a same-sized Transformer, and at the fifth-generation size, seventeen times faster. Meanwhile, Transformers Claude wrote at those widths only score thirty-seven point four and forty point nine.

The paper also tests the two properties directly. Keep only the single best-matching fact for each option, and accuracy goes from fifty point eight to forty-nine point one. Delete one written line and rebuild, and the table models change between zero point five and one point eight answers on average. In the first Transformer Claude wrote, one deleted line changes four point eight answers.

Then there's a control that matters a lot. Maybe the accuracy comes from the facts the LLM wrote, and the structure doesn't matter. To check, the authors ran a plain keyword search, BM25, over each writer's own text. Claude's written two-layer Transformers score eight points below BM25 on their own text, on average, while every table model scores above it, by one point five to five point seven points. The same knowledge, packed into a dense Transformer, does worse than just searching the text, which the authors say is consistent with interference between dense keys.

### Comparison

The authors check other explanations too. Did the written text copy test questions? According to the paper, it overlaps the test set about as often as it overlaps similar questions the writer never saw, and deleting the overlapping lines barely changes test accuracy. Forbidding the writer from making up practice questions costs about one point.

One explanation they say they can't rule out. LLMs already know retrieval, and one writer even called IDF retrieval the classic ARC baseline. So the table might be chosen because it suits hand-writing, or simply because it's familiar.

My read is that the most interesting part isn't how strong the table models are. It's the question itself. Architectures aren't optimal in the abstract. They get selected by the training method and the hardware together, and if the training method changes, the best structure may change with it.

My read is also that the comparison has clear limits. The table models beat Transformers written by the same LLM, not trained Transformers. Against a small pretrained model fine-tuned on ARC, the gap is still wide: fifty-four point four against eighty-two point one. And speed was measured on a single RTX 3090.

### Limitations

The first limitation the authors list: these written models are recalling facts the writer stated. They don't chain facts across steps or compose long answers. On the harder ARC-Challenge split, accuracy stays near thirty-eight percent, while ARC-Easy reaches sixty-three. Run unchanged on MMLU, models written for ARC are at chance.

Second, other tasks. On SciQ, science questions at the same level, the models get forty-seven point seven in the first generation and fifty point one in the fifth, but only thirty to thirty-six percent on OpenBookQA. On images, writing weights over raw pixels gets just twenty-one to twenty-eight percent on CIFAR-10, which the authors explain this way: a weight on a pixel has no meaning a writer can name.

Third, the speed advantage has a boundary. The authors' model doesn't slow down as facts grow, but it does as concepts grow. With the concept table stored densely, it becomes slower than a same-sized Transformer somewhere between about fifteen thousand and thirty-two thousand concepts. And the claim that you don't need a Transformer applies to today's LLMs writing weights from random initialization without running on data.

### Future work

The authors name three directions: writing calculation and comparison into weights, not just recall; starting gradient descent from a written associative-memory model; and adding word order so the model can generate text.

One result looks practical. On AG News text classification, a written network used as the starting point and then trained on a hundred labeled articles reaches up to eighty-four point seven percent, while training from random initialization on the same labels reaches at most sixty-three point six. The authors see this as a reason to try written networks where labels are scarce, though they haven't run experiments in domains like medicine.

My read is that the thing to watch is whether these readable, line-by-line editable models can be connected to gradient training and learned representations. If they can, this could become a kind of small model you can audit line by line. If not, it may stay a cleverly designed retrieval system.

That's today's Daily Frontier Papers. The paper is linked in the description.

