# Deep Fusion Memory: A Chunk-Level Parametric Memory for Large Language Models

**不检索也能用上知识库**

- 论文 Paper: [arXiv 2610.25100](https://arxiv.org/abs/2610.25100)
- 单位 Affiliation: 上海交通大学 LUMIA Lab · 上海人工智能实验室
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-25
- 视频 Videos: [YouTube 中文](https://youtu.be/FnpN1f1gTMk) · [YouTube English](https://youtu.be/WuHtHRsUZc0) · [B站](https://www.bilibili.com/video/BV1jkH564EsN)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇上海交通大学 LUMIA 实验室和上海人工智能实验室的论文，名字叫 Deep Fusion Memory，简称 DFM，可以叫它深度融合记忆。

它要解决的问题是，怎么让大模型学会一个新语料库里的知识，既不改动模型本身，推理的时候也不用再去检索。

### 问题

先说背景。大模型回答问题，靠的是预训练时记下的知识。可有些知识它没见过，有些记得不牢，有些已经过时了。

所以要把外部语料里的知识加进去，这叫知识注入。难点在于，既要加得有效，又不能破坏模型原有的能力，学习和推理的成本也得可控。

### 以前的方法

以前大致有三条路。第一条是继续预训练，或者用 LoRA 这种只训练少量参数的微调。推理时没有额外步骤，但模型可能会忘掉原来会的东西，这叫灾难性遗忘。

第二条是检索增强，也就是 RAG，回答之前先去资料库里搜相关段落，再交给模型。不用改模型，但每次推理都要搜索、处理检索结果，可能明显增加延迟。

第三条是参数化记忆，代表方法是 MLP Memory 和 Memory Decoder。思路是离线先把检索结果学进一个额外的记忆模块，推理时就不用再检索了。

但它有两个问题。一是训练数据很贵，要在语料里每一个 token 的位置都做一次检索，来构造训练目标。二是融合太浅，记忆模块只在最后输出概率那一步，和原模型的结果加权混合，没有进入模型内部的计算。

### 核心想法

这篇的想法可以拆成两半。第一半，监督不按 token 给，而是按块给，同一块里的 token 共享同一份记忆。

第二半，记忆不在输出端混合，而是直接注入到模型中间层的隐藏状态里，这样后面的层还能接着加工这份记忆，和上下文结合起来。

整个过程中，原来的大模型参数一直冻结，不动。

这张图是整体结构。左边是记忆怎么预测，中间是原来的模型，右边是记忆怎么融合进去。

### 方法

先看训练数据怎么造。把每条训练文本切成 32 个 token 一块。对每一块，拿它前面那一块去语料库里检索，找出最相近的 16 个文本块。

每个检索到的文本块，用一个冻结的文本编码器变成一个向量，16 个向量按排名排好，就是这一块的目标记忆。这一步离线做好，训练时不再变。

然后是记忆预测器。它是一个因果 Transformer，输入是前面每一块的平均词向量，输出 16 个记忆向量，目标就是逼近刚才检索出来的那 16 个向量。

换句话说，预测器学的是，只看前文，就直接生成检索本来会返回的那些向量，推理时不用真的去搜。

记忆怎么用。预测出的记忆先经过一个共享的投影层，变到模型隐藏层的维度。然后在选定的几层里，加一个记忆交叉注意力，让每个 token 去这 16 个记忆槽里挑自己需要的信息。

每个注意力头还配一个门控，是零到一之间的系数，决定这个头从记忆里拿多少信息。记忆分支和原来的自注意力并行，结果一起加回到残差里。

记忆注入在编号 9、16、24、31 的四层。每层有自己的注意力和门控参数，预测器和投影层是共享的。

训练目标有两部分。一个是正常的下一个 token 预测损失，一个是预测记忆和目标记忆之间的均方误差，两者权重相同，一起端到端训练。

推理时，就不需要任何检索了。没有资料库搜索，没有检索文本，也不需要外部编码器，预测器直接从输入算出记忆。问答时，记忆根据问题算一次，生成答案时保持不变。

加上去的模块，在两个问答底座上是 0.75B 参数。训练只跑一个 epoch，用 16 张 A800 显卡。

### 实验和效果

实验设置。问答部分用 2018 年 12 月的英文维基百科训练，大约 40 亿 token，底座模型是 Mistral-7B-v0.3 和 Qwen3-8B-Base。

评测有六个问答数据集，两个单跳，NQ 和 WebQA，四个多跳，HotpotQA、2Wiki、MuSiQue 和 Bamboogle。多跳的意思是，要把好几条信息串起来，才能得到答案。

对比的基线有六个。三个非参数方法，RAG、xRAG 和 RETRO，推理时都要检索。三个参数方法，继续预训练、LoRA 和 MLP Memory。

结果在表 1。在 Mistral 上，六项平均分从最强基线 xRAG 的 24.14，提到 27.21。在 Qwen3 上，从最强基线 MLP Memory 的 23.80，提到 26.46。

摘要里说的百分之 12.71 和百分之 11.19，是相对提升。换成绝对分数，是 3.07 分和 2.66 分。

和原始模型比，Mistral 平均涨了 13.59 分，六项全部提升。比如多跳题 Bamboogle，从 3.91 涨到 22.65。

再看继续预训练和 LoRA。它们在维基上训练之后，Qwen3 的平均分反而比原模型低了 7.58 分和 8.04 分，论文没有单独分析原因。DFM 冻结底座，在 Qwen3 上六项里五项上涨。

附录里还补了一个 OLMo-3 的 7B 模型，DFM 把平均分从 17.99 提到 25.49，比最强基线 RETRO 高 1.86 分。

第二组实验是领域适配。在代码、数学和法律三个领域分别训练，底座是 Qwen3 的 4B 和 8B，结果在表 2。

DFM 在两个尺寸上，四个评测全部比原模型高，平均分从 58.18 提到 64.69，从 63.14 提到 67.52，在比较的方法里平均分最高。

法律上变化最大。LawBench 在 4B 上从 29.96 提到 40.41，在 8B 上从 38.45 提到 46.61。而 RAG 在法律上反而比原模型低，4B 是 24.35，8B 是 26.89。

第三组是成本。和 MLP Memory 比，DFM 加的参数少一半，0.75B 对 1.5B。训练时间，同样 16 张 A800，从大约 64 小时降到 24 小时。

构造训练数据的检索量降得更多。对 40 亿 token 的语料，按 token 检索要 40 亿次查询，每次取 1024 个邻居。按 32 个 token 一块，只要 1.25 亿次查询，每次取 16 个，检索返回的条目总数少了 2048 倍。

### 和以前方法的对比

但 DFM 不是每一项都赢。在 Qwen3 上，WebQA 从 41.78 掉到 39.47，而 MLP Memory 是 43.26。作者自己也说，记忆适配并不是在所有底座上都有好处。

多跳题上，显式检索有时还是更强。Mistral 上的 2Wiki，RAG 是 31.02，DFM 是 25.00。Qwen3 上的 HotpotQA，xRAG 是 27.74，DFM 是 26.46。

领域适配里也一样。4B 的法律，继续预训练比 DFM 高 1.25 分。8B 的 HumanEval+ 和 GSM8K，LoRA 更强。

还有一个细节。对比用的 MLP Memory，插值权重是在每个评测集上挑的最好值，这个设定对基线是有利的。

消融实验里，影响最大的是记忆注入的位置。只在最后一层注入，平均分只有 21.11。放在最后连续四层，是 22.07。分散在中间和靠后的四层，是 27.21。

块大小和记忆槽数量反而没那么敏感。块大小 16、32、64，平均分在 26.51 到 27.21 之间。记忆槽从 2 个到 16 个，平均分只差 0.33 分。

门控也有用。去掉门控，平均分从 27.21 降到 26.83。有意思的是，门控偏置初始化成 0 的时候，平均分是 27.90，比默认设置还高，作者保留了更保守的默认值。

我的判断是，DFM 的价值在于两件事一起做，一是把监督从 token 粒度放大到块粒度，构造数据要做的检索大幅减少，二是把记忆放进模型中间层。消融显示，注入的位置比块大小、记忆槽数量重要得多。

我的判断是，它的定位介于 RAG 和微调之间，不碰原模型，推理不用检索，对想把领域知识加进开源模型的团队会有吸引力。

### 局限

论文没有单列局限，但从实验细节能看出几点。第一，所有消融都是单次运行，很多差距只有零点几分，单次运行看不出这些差距稳不稳定。

第二，检索量减少的倍数，算的是查询次数和返回条目数，不是实际耗时。论文明确说，这个数字不计算浮点运算量，也不计算存储。

第三，规模和数据有限。问答只在 7B、8B 左右的模型上做，知识来源是 2018 年的维基百科。领域适配里，代码和数学各用 1 亿 token，法律约 1.2 亿 token。

第四，论文的出发点之一是 RAG 推理延迟高，但没有报告 DFM 自己推理时的延迟。它加上去的是 0.75B 参数，其中预测器有 28 层。

另外，代码暂时还没有公开，论文说会尽快发布。

### 未来方向

论文没有专门写未来方向。作者的结论是，共享监督加上模型内部的融合，可以让知识获取既高效又有效。

我的判断是，接下来最值得看三件事。一是在更大的模型、更大的语料上，效果还能不能保持。二是推理时多出来的计算到底有多少，和 RAG 比划不划算。三是知识更新的时候，能不能只重训记忆模块，就跟上新知识。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the LUMIA Lab at Shanghai Jiao Tong University and the Shanghai Artificial Intelligence Laboratory. It's called Deep Fusion Memory, or DFM.

The question it asks: how do you get a language model to absorb the knowledge in a new corpus, without touching the model's own weights, and without running a search every time it answers?

### Problem

Some background first. A language model answers from what it picked up in pretraining. But some facts it never saw, some it barely retained, and some are simply out of date.

Adding knowledge from an outside corpus is called knowledge injection. The hard part is doing it well while keeping the model's existing skills intact, and keeping both learning and inference affordable.

### Prior work

There are roughly three options. The first is continued pretraining, or a lightweight fine-tune like LoRA. Inference stays simple, but the model can lose things it used to know. That's catastrophic forgetting.

The second is retrieval-augmented generation, or RAG. Before answering, you search a datastore for relevant passages and hand them to the model. No weight changes, but every query pays for the search and for processing what comes back, which can add a lot of latency.

The third is parametric memory, with methods like MLP Memory and Memory Decoder. The idea is to learn the retrieval results offline into an extra memory module, so that at inference time there's no search at all.

That approach has two problems. First, the training data is expensive: you run a retrieval at every single token position in the corpus to build the targets. Second, the fusion is shallow. The memory only gets mixed in at the very end, by blending output probabilities, and never enters the model's internal computation.

### Key idea

This paper's idea has two halves. Half one: supervise by chunk, not by token, so every token in a chunk shares the same memory.

Half two: instead of mixing memory in at the output, inject it into the hidden states of intermediate layers, so the later layers can keep working on that memory alongside the context.

And throughout, the original model's weights stay frozen.

Here's the overall architecture. On the left, how memory is predicted. In the middle, the original model. On the right, how the memory gets fused in.

### Method

Start with the training data. Each training sequence is cut into chunks of thirty-two tokens. For each chunk, the chunk before it is used as a query to search the corpus for its sixteen nearest chunks.

Each retrieved chunk is turned into a vector by a frozen text encoder. Stack the sixteen vectors in rank order and you have that chunk's target memory. This is all done offline and stays fixed during training.

Next, the memory predictor. It's a causal Transformer. Its inputs are the averaged token embeddings of each earlier chunk, and it outputs sixteen memory vectors, trained to match the sixteen retrieved ones.

In other words, the predictor learns to read the preceding text and directly produce the vectors a retrieval would have returned, so at inference time nothing actually gets searched.

How the memory is used. The predicted memory first goes through a shared projector into the model's hidden size. Then, in a few selected layers, a memory cross-attention lets each token pick what it needs from those sixteen memory slots.

Each attention head also gets a gate, a number between zero and one that decides how much that head takes from memory. The memory branch runs in parallel with the usual self-attention, and both feed into the same residual update.

Memory goes into four layers, numbered nine, sixteen, twenty-four and thirty-one. Each of those layers has its own attention and gate parameters, while the predictor and projector are shared.

Training has two losses. One is the usual next-token prediction loss. The other is the mean squared error between predicted and target memory. They're weighted equally and trained end to end.

At inference, retrieval is gone. No datastore search, no retrieved text, no external encoder. The predictor computes memory straight from the input. For question answering, memory is computed once from the prompt and held fixed while the answer is generated.

The added modules come to zero point seven five billion parameters on the two question-answering backbones, trained for one epoch on sixteen A800 GPUs.

### Results

The setup. For question answering, training uses English Wikipedia from December twenty eighteen, about four billion tokens. The backbones are Mistral seven B version zero point three, and Qwen3 eight B Base.

Evaluation covers six question-answering sets. Two single-hop, NQ and WebQA, and four multi-hop, HotpotQA, 2Wiki, MuSiQue and Bamboogle. Multi-hop means the answer requires chaining several pieces of information together.

There are six baselines. Three non-parametric ones that retrieve at inference: RAG, xRAG and RETRO. And three parametric ones: continued pretraining, LoRA, and MLP Memory.

Results are in Table 1. On Mistral, the six-task average goes from twenty-four point one four for the strongest baseline, xRAG, to twenty-seven point two one. On Qwen3, from twenty-three point eight for MLP Memory to twenty-six point four six.

The abstract's twelve point seven one and eleven point one nine percent are relative improvements. In absolute terms, that's three point zero seven and two point six six points.

Against the untouched model, Mistral gains thirteen point five nine points on average, and improves on all six tasks. On the multi-hop set Bamboogle, for example, it goes from three point nine one to twenty-two point six five.

Now look at continued pretraining and LoRA. After training on Wikipedia, their Qwen3 averages actually fall below the original model, by seven point five eight and eight point zero four points. The paper doesn't analyze why. DFM, with the backbone frozen, improves five of six tasks on Qwen3.

An appendix adds a third backbone, OLMo-3 seven B. There DFM lifts the average from seventeen point nine nine to twenty-five point four nine, one point eight six points above the strongest baseline, RETRO.

The second experiment is domain adaptation. Separate models are trained on code, math and law, on Qwen3 four B and eight B. Results are in Table 2.

At both sizes, DFM beats the base model on all four benchmarks. The average rises from fifty-eight point one eight to sixty-four point six nine, and from sixty-three point one four to sixty-seven point five two, the highest average among the methods compared.

Law moves the most. LawBench goes from twenty-nine point nine six to forty point four one at four B, and from thirty-eight point four five to forty-six point six one at eight B. RAG, meanwhile, scores below the base model on law: twenty-four point three five and twenty-six point eight nine.

The third experiment is cost. Compared with MLP Memory, DFM adds half the parameters, zero point seven five billion versus one point five. Training time, on the same sixteen A800s, drops from about sixty-four hours to twenty-four.

The retrieval needed to build the training data drops even more. For a four-billion-token corpus, token-level targets take four billion queries, with one thousand and twenty-four neighbors each. With thirty-two-token chunks, it's one hundred twenty-five million queries with sixteen neighbors each, so the total number of retrieved items falls by a factor of two thousand and forty-eight.

### Comparison

But DFM doesn't win everywhere. On Qwen3, WebQA drops from forty-one point seven eight to thirty-nine point four seven, while MLP Memory scores forty-three point two six. The authors say themselves that memory adaptation isn't uniformly beneficial across backbones.

On multi-hop questions, explicit retrieval sometimes still wins. On Mistral's 2Wiki, RAG scores thirty-one point zero two to DFM's twenty-five. On Qwen3's HotpotQA, xRAG scores twenty-seven point seven four to DFM's twenty-six point four six.

Same story in domain adaptation. On law at four B, continued pretraining beats DFM by one point two five points. At eight B, LoRA is stronger on HumanEval-plus and GSM8K.

One more detail. For the MLP Memory baseline, the interpolation weight was picked as the best value on each evaluation set, a setup that favors the baseline.

In the ablations, what matters most is where the memory goes in. Fusing at the last layer only gives an average of twenty-one point one one. Putting all four fusion layers at the very end gives twenty-two point zero seven. Spreading them across middle and later layers gives twenty-seven point two one.

Chunk size and the number of memory slots matter much less. With chunks of sixteen, thirty-two or sixty-four tokens, the average stays between twenty-six point five one and twenty-seven point two one. From two to sixteen memory slots, it moves by only zero point three three.

The gate helps too. Removing it drops the average from twenty-seven point two one to twenty-six point eight three. Interestingly, initializing the gate bias at zero scores twenty-seven point nine, higher than the default, but the authors kept the more conservative default.

My read is that DFM's value comes from doing two things together. It moves supervision from tokens to chunks, which slashes the retrieval needed to build the data, and it puts memory inside the model's middle layers. The ablations say placement matters far more than chunk size or slot count.

My read is that it sits between RAG and fine-tuning. It leaves the base model alone and needs no retrieval at inference, which should appeal to teams that want to add domain knowledge to an open model.

### Limitations

The paper has no separate limitations section, but a few things stand out. First, every ablation is a single run, and many of the gaps are fractions of a point. A single run can't tell you whether those gaps are stable.

Second, the retrieval savings count queries and returned items, not wall-clock time. The paper says plainly that the count excludes compute and storage.

Third, scale and data are limited. Question answering is tested only on models around seven to eight billion parameters, with twenty eighteen Wikipedia as the knowledge source. Domain adaptation uses one hundred million tokens each for code and math, and about one hundred twenty million for law.

Fourth, one motivation is RAG's inference latency, yet the paper doesn't report DFM's own inference latency. It adds zero point seven five billion parameters, including a twenty-eight-layer predictor.

Also, the code isn't out yet. The paper says it will be released soon.

### Future work

The paper doesn't lay out future work. The authors' conclusion is that shared supervision plus fusion inside the model can make knowledge acquisition both efficient and effective.

My read is there are three things to watch. Whether the gains hold on bigger models and bigger corpora. How much extra compute inference really costs, and whether that beats RAG. And whether, when knowledge changes, you can retrain just the memory module to keep up.

That's today's Daily Frontier Papers. The paper is linked in the description.

