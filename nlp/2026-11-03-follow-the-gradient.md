# Follow the Gradient: Tracing Safety Degradation Back to Training Data

**无害的数据也会教坏模型**

- 论文 Paper: [arXiv 2610.37481](https://arxiv.org/abs/2610.37481)
- 单位 Affiliation: DatologyAI · 伊利诺伊大学香槟分校
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-03
- 视频 Videos: [YouTube 中文](https://youtu.be/1KeLxly4TLM) · [YouTube English](https://youtu.be/CbiSxtaXpBc) · [B站](https://www.bilibili.com/video/BV1h44g6UErr)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 DatologyAI 和伊利诺伊大学香槟分校的论文，题目叫 Follow the Gradient，跟着梯度走，副标题是，把安全性的下降追溯到训练数据。

它要回答的问题很实际。你拿一批看起来完全无害的数据去微调大模型，模型却变得更容易被越狱，那到底是哪些数据在拖后腿。

### 问题

先说背景。微调是让大模型适应新任务的主要手段，但已有研究发现，就算训练数据里没有任何明显有害的内容，微调也会削弱模型的安全对齐。

之前有一项研究发现，这种退化集中在一小部分数据上。从良性数据里挑出 100 条梯度最像有害样本的，拿去微调，攻击成功率能超过百分之七十，随机挑 100 条只有百分之十三。而这些数据里根本没有有害内容，更多是列表、要点和数学题。

所以，保住安全性就变成了一个数据筛选问题，找到这些数据，训练前删掉。难点在于它们看起来是无害的，毒性过滤器、大模型安全裁判，这些看内容的工具，很难把它们挑出来。

### 以前的方法

以前的做法大概有几类。一类也是用梯度打分，比如刚才那项研究，它只看回答前十个 token 的梯度，拿去和有害样本、安全样本对比。也有方法要额外训练一个筛选器。

另一条线叫训练数据归因，问的是，模型的某个行为，是哪些训练样本造成的。经典方法是影响函数，要用到海森矩阵的逆，在大模型上算不动，后来的工作都在想办法压缩每个样本的梯度。

共同的瓶颈是成本。逐个样本算梯度太贵，以前基于梯度的安全研究，只用在比较小的数据集上，没法直接用到上百万条的后训练数据集。

### 核心想法

这篇的出发点是，不看数据长什么样，看模型会从这条数据里学到什么。一条样本的梯度，描述的就是模型如果用它训练，参数会往哪个方向更新。

第一页这张图说的就是这个意思。上半部分是文字表面，不安全的代码和有害言论看起来毫不相干。下半部分是梯度空间，不安全的代码和有害输出挤在了一起，安全的代码留在另一边。

所以作者问的是，哪些训练样本，把模型往和不安全回答一样的方向推。找到它们，删掉。

### 方法

方法叫 SimpleLoRIF，是在一个叫 LoRIF 的归因方法上做减法。先说怎么把一条样本变成一个向量。

完整的梯度太大，存不下。LoRIF 先对每一层的梯度做随机投影，压到 32 乘 32，再只保留秩为 1 的分解，每一层从 1024 个数降到 64 个数。两条样本像不像，直接用这些小向量算，不用还原完整的梯度。

第一处简化，LoRIF 原本还有一项曲率修正，用来更准地估计一条数据的影响有多大。作者说，他们只需要给样本排序，决定删谁，梯度相似度就够了，所以这一项直接去掉。

第二处简化更关键，反向传播只走最后四个 transformer 块。为了找到最省的设置，作者在四个方向上做了消融，投影维度、反传多少层、每条样本读多少 token，还有换更小的模型。

这张图横轴是耗时，纵轴是检测质量 AUC。只算最后两层，AUC 掉到 0.65。最后四层是 0.684，论文报告，相比基础的简化版本提速 91.53%。再加到六层、八层，几乎没有收益。

这样一来，Tulu 3 这个后训练数据集的全部 939343 条样本，用 8 张 H100，一小时零一分钟就算完了。

### 实验和效果

第一个实验，先验证梯度能不能看出问题。用的是一个叫涌现错位的现象，就是只用不安全的代码去微调模型，模型在跟编程无关的问题上也会变坏。

作者用了两个已经出现这种现象的模型，一个是 Qwen 的代码模型，一个是 Mistral Small 24B。训练数据是原研究的 6000 条不安全代码和 6000 条安全代码，两组在表面上非常接近。

然后让模型回答一些和编程无关的问题，只留下明显有害的回答当作目标，再看能不能靠梯度的相似度，把不安全的代码挑出来。

衡量标准是 AUC，就是随机抽一条不安全代码、一条安全代码，前者得分更高的概率。0.5 等于瞎猜，1 是完美区分。

结果在这张图里。微调后的 Qwen，AUC 是 0.604，微调后的 Mistral 是 0.865。而在微调之前，两个模型分别只有 0.552 和 0.431。也就是说，这种关联是模型变坏之后，才在梯度空间里出现的。

第二个实验是真正的应用，叫 SafeTulu。流程是四步，算梯度向量，打分，删掉最像不安全回答的百分之二十，再用剩下的数据训练和评测。

具体设置。算梯度用的是很小的 Qwen 3 0.6B。目标集是 SafeRLHF 数据集里带不安全回答的样本，43469 条。被训练的是 Qwen 3 1.7B、Llama 3.2 1B 和 Olmo 2 1B 三个小模型。

安全评测用了两种攻击。GCG 是白盒攻击，在有害请求后面优化出一串后缀，诱导模型照做。ActorBreaker 是多轮攻击，从看起来无害的问题开始，一步步把对话引到有害目标上。

对比的基线有四个，不过滤，随机删百分之二十，用 Qwen 3 8B 当安全裁判、删掉评分最差的百分之二十，还有刚才说的前人梯度方法。每种设置跑三个随机种子。

结果看这张图。在 Qwen 上，GCG 攻击成功率从不过滤时的 47.7%，降到 37%，安全裁判过滤后是 47.3%，几乎没变。

Llama 从 38% 降到 30%，Olmo 从 48% 降到 41%，安全裁判对应是 39% 和 46%。作者把这组结果概括为，攻击成功率最多降低 22%，Qwen 从 47.7 到 37，少了两成多。

在多轮攻击 ActorBreaker 上，SafeTulu 在所有模型上的攻击成功率也都是最低的。能力方面，通用、代码、数学和医学这些基准，基本和用全量数据训练的模型持平，有几项还更高。

还有一个细节值得说，算梯度的只是一个 0.6B 的 Qwen，筛出来的数据换到 Llama、Olmo 上也有效。附录里还补了一个更大的 Llama 3.1 8B，GCG 攻击成功率从 11.7% 降到 7%，也是几种做法里最低的。

### 和以前方法的对比

和安全裁判的差别，在删除比例的实验里最明显。作者在 Llama 1B 上，把删除比例从百分之五一直调到百分之四十。

安全裁判删到百分之十，攻击成功率最低，是 35%，再多删反而变差，删到百分之四十是 43%，比不过滤还高。梯度过滤一直到百分之二十都在下降，最低到 30.33%，到百分之四十才回升到 39%。

作者的解读是，Tulu 3 里明显有害的内容本来就少，看内容的裁判删到一成左右就找不到问题数据了，梯度过滤能多删大约一倍，说明它找到的问题数据更多。

再看被删掉的是什么。三个数学子集，安全裁判的删除比例几乎一样，都在百分之十五点五左右，梯度过滤分别删了 7.9%、19.4% 和 17.2%，作者认为，这说明它看到的东西不只在文字表面。

有几个规律。带 Python 报错信息的数学解题，更容易被删，论文给的相对比例是 1.71 倍。简单的写一个 Python 函数这类任务，是 1.74 倍。用文字一步步推算数量的叙述式算术，也被删得更多。而真正设变量、列方程、做代数变形的题，更多被留了下来。

作者给的一种解释是，直接、立刻给出答案的数据，会强化一种马上照做的模式。但他们反复强调，这些只是相关性，不能说明这些特征导致了安全下降。

我的判断是，这篇最有价值的地方，是把梯度归因从小规模的分析工具，变成了能过滤百万级数据集的工具，成本是 8 张卡一小时。

### 局限

作者自己写了两条局限。第一，只测了总体的安全下降，没测具体的不良行为，比如谄媚。第二，目标集只用了 SafeRLHF 一个，没试别的。

附录里还有两个结果需要注意。在涌现错位实验里，梯度信号换个模型不一定成立，用 Qwen 去算 Mistral 的有害回答，AUC 掉到 0.247。模型规模也不是越大越好，没有微调过的 Qwen 代码模型里，32B 的 AUC 比 14B 还低。

我的判断是，还要看到几点。训练的都是 1B 到 8B 的小模型，过滤以后，三个 1B 级别模型的 GCG 攻击成功率仍在 30% 到 41%。在 8B 上，几种过滤方法差距不大，7% 对 8.7% 和 9.3%。过度拒答的指标，也是有升有降。

### 未来方向

未来方向，作者提了几种用法。实验室可以在新收集的数据，比如 Agent 工具调用数据，并入训练集之前先筛一遍。也可以换一个目标集，比如谄媚的回答，看哪些数据在助长谄媚，或者反过来，给有助于安全的数据加权。

他们还认为，这个方法可以和数据投毒的检测手段结合，因为这里找到的数据和投毒样本一样，表面上看不出问题，却会改变模型的行为。这一点论文还没有做实验。

作者公开了 SafeTulu 数据集和一个 Hugging Face Space，可以自己选删除比例，或者只过滤某些子集。

我的判断是，后训练数据越来越多来自合成数据和 Agent 轨迹，这种看似无害的数据只会更多。按梯度筛数据，可能会成为数据流水线里的常规一步，但还需要在更大的模型、更多的目标集上验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from DatologyAI and the University of Illinois Urbana-Champaign. It's called Follow the Gradient: Tracing Safety Degradation Back to Training Data.

The question is a practical one. You fine-tune a model on data that looks completely harmless, and it comes out easier to jailbreak. Which examples did that?

### Problem

Some background. Fine-tuning is the main way models get adapted to new tasks, and earlier work showed it can weaken safety alignment even when nothing in the training data is obviously harmful.

A previous study found the damage is concentrated in a small slice of the data. Fine-tune on just one hundred benign examples whose gradients look most like harmful ones, and attack success climbs above seventy percent, against thirteen percent for a random hundred. And those examples contained no harmful content at all; they were mostly lists, bullet points and math questions.

So keeping a model safe becomes a data selection problem: find those examples and drop them before training. The hard part is that they look benign, so toxicity filters and LLM safety judges, which read what the content says, struggle to flag them.

### Prior work

There are a few existing approaches. Some also score data by gradients, like that earlier study, which only looks at the gradient of the first ten response tokens and compares it with harmful and safe anchors. Others train a separate selector, or first pick out the one layer most tied to safety.

A second line of work is training data attribution: which training examples caused a given model behavior? The classic tool is influence functions, which need an inverse Hessian and are intractable at LLM scale, so later work keeps finding ways to compress each example's gradient.

The shared bottleneck is cost. Per-example gradients are expensive, and earlier gradient-based safety work stayed on fairly small datasets, nowhere near a million-example post-training mix.

### Key idea

The starting point here is to ignore what an example looks like and ask what the model would learn from it. An example's gradient is exactly that: the direction the weights would move if you trained on it.

The figure on the first page makes the point. On top is the surface text, where insecure code and harmful statements look nothing alike. Below is gradient space, where insecure code and harmful outputs cluster together, and secure code sits off to the side.

So the authors ask which training examples push the model in the same direction as unsafe responses. Find those, and remove them.

### Method

The method is called SimpleLoRIF, a stripped-down version of an attribution method called LoRIF. First, how an example becomes a vector.

A full gradient is far too big to store. LoRIF randomly projects each layer's gradient down to thirty-two by thirty-two, then keeps only a rank-one factorization, so each layer goes from one thousand and twenty-four numbers to sixty-four. Similarity between two examples is computed straight from these small factors, without rebuilding the full gradients.

Simplification one: LoRIF includes a curvature correction to estimate more precisely how much an example matters. The authors only need to rank examples to decide what to cut, so plain gradient similarity is enough, and the correction goes.

Simplification two matters more: backpropagate only through the last four transformer blocks. To find the cheapest setup, the authors ablated four knobs: projection size, how many blocks to backpropagate through, how many tokens to read per example, and a smaller embedding model.

In this figure, the horizontal axis is time and the vertical axis is detection quality, measured as AUC. With only the last two blocks, AUC drops to zero point six five. With four it's zero point six eight four, and the paper reports a ninety-one point five three percent speedup over the base simplified version. Going to six or eight blocks adds little.

The result: all nine hundred thirty-nine thousand, three hundred forty-three examples of the Tulu 3 post-training dataset embedded in one hour and one minute on eight H100s.

### Results

The first experiment checks whether gradients can see the problem at all. It uses emergent misalignment: fine-tune a model only on insecure code, and it starts misbehaving on questions that have nothing to do with programming.

The authors take two models that already show this, a Qwen coding model and Mistral Small twenty-four B. The training data is the original study's six thousand insecure and six thousand secure code examples, which look very similar on the surface.

They prompt each model with non-coding questions, keep only the clearly misaligned answers as targets, and then ask whether each code example's gradient resembles the gradients of those answers closely enough to pick out the insecure code.

The metric is AUC: draw one insecure and one secure example at random, and it's the chance the insecure one scores higher. Zero point five is a coin flip, one is perfect separation.

Here are the results. For the fine-tuned Qwen, AUC is zero point six zero four, and for the fine-tuned Mistral, zero point eight six five. Before that fine-tuning, the same models scored zero point five five two and zero point four three one. In other words, the link only shows up in gradient space after the model has gone bad.

The second experiment is the real application, called SafeTulu. Four steps: embed every example, score it, drop the twenty percent most aligned with unsafe responses, then train and evaluate on the rest.

The setup. Embeddings come from a tiny model, Qwen 3 zero point six B. The target set is the forty-three thousand, four hundred sixty-nine SafeRLHF examples with unsafe responses. The models being trained are three small ones: Qwen 3 one point seven B, Llama 3.2 one B and Olmo 2 one B.

Safety is measured with two attacks. GCG is a white-box attack that optimizes a suffix tacked onto a harmful request to make the model comply. ActorBreaker is multi-turn, starting from innocent-looking questions and steering the conversation toward the harmful goal.

There are four baselines: no filtering, removing a random twenty percent, using Qwen 3 eight B as a safety judge and dropping its worst-scored twenty percent, and the earlier gradient method. Each setting is run with three random seeds.

Now the results. On Qwen, GCG attack success falls from forty-seven point seven percent with no filtering to thirty-seven percent, while the safety-judge filter lands at forty-seven point three, barely moved.

Llama drops from thirty-eight to thirty percent and Olmo from forty-eight to forty-one, against thirty-nine and forty-six for the judge. The authors sum this up as up to a twenty-two percent reduction in attack success, and Qwen's drop from forty-seven point seven to thirty-seven is a bit more than a fifth.

On the multi-turn ActorBreaker attack, SafeTulu also has the lowest attack success on every model. And capability holds: on general, coding, math and medical benchmarks, the SafeTulu models mostly stay close to models trained on the full data, and a few scores go up.

One detail worth stressing is that the gradients came from a zero point six B Qwen, yet the filtered data also works when training Llama and Olmo. The appendix adds a larger Llama 3.1 eight B, where GCG attack success falls from eleven point seven percent to seven, again the lowest of all the options.

### Comparison

The gap with the safety judge is clearest when you vary how much data gets removed. On Llama one B, the authors swept the removal fraction from five percent up to forty.

The judge does best at ten percent, at thirty-five percent attack success, and then gets worse, reaching forty-three at forty percent removal, which is above no filtering at all. Gradient filtering keeps improving through twenty percent, bottoming out at thirty point three three, and only rises back to thirty-nine at forty percent.

The authors' reading: Tulu 3 has little overtly harmful content, so a content judge runs out of genuinely harmful data at around ten percent. Gradient filtering keeps helping for about twice as much removed data, which suggests it finds a larger pool of safety-degrading examples.

Then look at what gets removed. On three math subsets, the judge removes almost exactly the same share, around fifteen and a half percent each, while gradient filtering removes seven point nine, nineteen point four and seventeen point two percent. The authors take this as a sign that it's picking up on something beyond the surface of the text.

A few patterns emerge. Math solutions containing a Python traceback are more likely to be cut, with a relative rate the paper puts at one point seven one. Simple write-a-Python-function tasks come in at one point seven four. Narrative arithmetic, where quantities are carried through a chain of prose steps, is removed more often too. Problems with real algebra, defining variables and setting up equations, are more often kept.

One explanation the authors offer is that direct, immediately helpful examples reinforce a habit of instant compliance. But they stress repeatedly that these are correlations, not evidence that these features cause safety loss.

My read is that the real contribution is turning gradient attribution from a small-scale analysis tool into a filter you can run over a million-example dataset, for the cost of one hour on eight GPUs.

### Limitations

The authors list two limitations. First, they only test general safety degradation, not specific behaviors like sycophancy. Second, they use only one target set, SafeRLHF.

The appendix has two more results worth knowing. In the emergent misalignment setting, the gradient signal doesn't reliably carry across models: using Qwen to embed Mistral's harmful answers drops AUC to zero point two four seven. And bigger isn't always better: among the un-fine-tuned Qwen coding models, thirty-two B scores lower than fourteen B.

My read is that a few more caveats apply. Every trained model is between one and eight billion parameters, and after filtering, the three one-B-class models still sit between thirty and forty-one percent GCG attack success. At eight B, the filtering methods are close: seven percent for SafeTulu, eight point seven for the judge, nine point three for the earlier gradient method. And over-refusal goes down on three models but up on Olmo.

### Future work

Looking ahead, the authors sketch some uses. A lab could screen newly collected data, say agentic tool-use data, before merging it into a training mix. It could swap in a different target set, like sycophantic responses, to see which data feeds sycophancy, or flip the method around to upweight data that pushes toward safe behavior.

They also suggest pairing it with data-poisoning detectors, because the data found here behaves like clean-label poison: nothing looks wrong on the surface, yet it changes how the model behaves. That part is untested in the paper.

The authors release the SafeTulu dataset and a Hugging Face Space where you can pick your own removal fraction or filter only certain subsets.

My read is that as more post-training data comes from synthetic generation and agent trajectories, there will be more of this harmless-looking, quietly safety-eroding data. Filtering by gradients could well become a routine step in data pipelines, but it still needs testing on larger models and more target sets.

That's today's Daily Frontier Papers. The paper is linked in the description.

