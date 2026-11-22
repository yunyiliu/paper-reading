# Language Models Can Rewrite Their Own Weights

**不训练不算梯度模型自己改权重**

- 论文 Paper: [arXiv 2611.21885](https://arxiv.org/abs/2611.21885)
- 单位 Affiliation: UIUC 等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-22
- 视频 Videos: [YouTube 中文](https://youtu.be/J4NRzMos9_8) · [YouTube English](https://youtu.be/I5TduhH0MKM) · [B站](https://www.bilibili.com/video/BV1HUyGzPETD)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇以伊利诺伊大学厄巴纳香槟分校为主，联合卡内基梅隆、普林斯顿、斯坦福等学校的论文，题目是，语言模型可以改写自己的权重。

它问的问题很大胆，一个模型能不能不训练，不算梯度，只靠自己推理，直接改自己的参数，让自己变得更好。论文的回答是，在他们测的这个 27B 开源模型上，可以，但前提和限制不少，我们一条条看。

### 问题

先说背景。今天的大模型训练完就冻结了，想让它变好，要么再跑一轮训练，这需要懂训练的人、大集群和整条流水线，要么改提示词、套一层外部框架、推理时多花算力，可这些在任务结束后就没了。

作者想找第三条路，让改进直接写进权重，部署以后，每次回答不多花一分钱，也不需要任何训练设施。

### 以前的方法

以前的自我改进，大致三类。一类是改提示词或者 Agent 的代码，权重不动。一类是让模型用自己的输出做训练数据，但真正改权重的还是梯度下降。还有一类是推理时自我反思，每来一个新问题，都要再付一次成本。

最接近的工作是一篇博客，一个前沿大模型去读一个小推理模型的权重，找到让它陷入循环的方向，把它删掉，没有训练。但那是强模型改弱模型，自己改自己，被当成了未来工作。这篇论文接着做的就是这件事。

### 核心想法

核心想法一句话，让模型当自己的优化器。把它自己的权重、它在练习题上的判分结果、以及运行自己的能力都交给它，但禁止任何形式的训练，每一个改动的数值，都得是它自己推理出来写进去的。

作者还设计了第二个任务。给模型一个自己的损坏副本，其中一块权重被噪声覆盖，几乎一道题都做不对，只告诉它有些数值坏了，不说在哪、怎么坏的，看它能不能自己修好。

### 方法

具体流程看这张图。一次会话里，模型每一轮先写几句理由，再做一个动作。动作有三种，在 CPU 上跑 Python，直接读写权重矩阵，让自己做题，带编辑和不带编辑用同样的随机种子各答一遍，作者叫配对测试，或者读取自己在指定文本上的隐藏状态。

预算是固定的，一次会话最多 20 次前向运行、200 次代码运行。Python 环境里梯度是关掉的，算梯度、跑优化器的代码都会被拒绝。输入嵌入、最后的归一化和输出层不许改，把整个矩阵乘一个常数也不行，因为那相当于调采样温度。

任务要求是，在没用来挑选编辑的题目上，明显减少因为某种失败丢掉的答案，同时准确率不能显著下降。两条都做不到，就什么都不交，这就是撤回。

实验用的是 Qwen3.5 的 2B、4B、9B 和 27B 四个尺寸，这一代把线性注意力 Gated DeltaNet 和全注意力混在一起。练习题是 DeepScaleR 里的 3000 道数学题，去掉了和 MATH-500、AIME 这些评测重叠的题，另外留出 1000 道，任何会话都看不到。

### 实验和效果

先看 27B 模型怎么改自己。在 4096 token 的预算下，它读起始诊断，发现 80.5% 的答案没在上限内写完，但写完的答案 96% 是对的。它自己的结论是，推理没问题，问题是收不住。

它对比写完的答案和没写完的答案在隐藏状态上的差别，先改后面的层，测试说变差了，就放弃。再往前看，发现差别随深度变大，于是改到第 8 层，把这个方向从这一层的两个输出矩阵里削掉 5%。三次配对测试都支持这个编辑，它就提交了。

这个改动很小，两个矩阵的范数只变了 0.6% 和 0.1%。在事先固定好的评测里，490 道留出题，每题回答 8 次，准确率从 57.2% 到 58.3%，提高 1.1 个点，统计上显著，但幅度不大。

更大的提升来自第二套流程。每次会话都从原始权重出发，模型提交之前，框架先用一批它读不到、也用不了的预留练习题替它检查一次，再由它决定提交还是撤回。

前八次会话全部报告，五次撤回，三次提交。第 8 次的编辑，把准确率从 57.2% 提到 63.8%，高 6.6 个点。第 2 次到 61.3%，高 4.0 个点，两个都显著。第 1 次只高 0.5 个点，不显著。

有意思的是，6.6 个点这次，改法非常简单，把全部 48 个线性注意力层的时间步偏置统一加 0.5。按模型自己的解释，这让这些层衰减得更快，长推理不容易卡住。跑满 token 上限的答案，从 33.6% 降到 26.1%。

再看修复损坏副本。作者把 27B 模型第 30 层 MLP 输出矩阵里连续的 8 行，换成标准差放大 200 倍的噪声。这个副本在留出题上一道都做不对。

模型的排查很像工程师调试。它先看到隐藏状态随层数暴涨，猜是前几层坏了，几次尝试都失败。然后它把每个输出矩阵的最大值和中位数之比算了一遍，第 30 层那个矩阵是 1397 倍，其他矩阵都不超过 162.3 倍。

它的修法是，把超过中位数 15 倍的权重截断，用在这个矩阵和第 0 到第 10 层的另外 19 个矩阵上，只动了其中 0.013% 的权重。留出题准确率从 0 回到 33%，原模型是 58.5%。

消融显示，只改它点名的那一个矩阵，准确率能到 55%，单独改另外 19 个，几乎没用。而且在坏掉的那几行里，它写进去的是自己的截断值，不是把原始权重抄回来。

更让人意外的是，这个修复只在文字数学题上找到，却在六个它从没见过的图像基准上也起作用，准确率从 7.3% 回到 70.2%，是原模型的 83%。在界面元素定位 ScreenSpot-v2 上，从 0 回到 89%，原模型是 93%。

最后是规模。同样的任务和工具，2B 模型不把自己的权重当成能改的东西，4B 和 9B 往权重里加随机噪声，把自己改坏了，9B 提交的编辑让留出题准确率从 47.5% 掉到 0。只有 27B 是先测量，再修改，再按测试结果取舍。

### 和以前方法的对比

和以前的路线比，在作者自己的对比表里，自我改写是唯一五项都打勾的，改权重，不训练，由模型自己选，看自己内部，每次回答不加成本。

作者也承认，27B 的一些编辑，形式上就是研究者早就在用的方向消除，区别在于，这里每一个选择都是模型自己做的。

我的判断是，这篇最有说服力的是实验纪律，事先固定的评测，配对测试，八次会话连撤回的都报告了。但提升基本都来自同一个症状，答案太长、收不住，最大的那次就是一个统一的偏置调整，更像是模型找到了一个好用的旋钮，而不是学会了新的数学能力。

我的判断是，论文也没有报告更简单的对照，比如研究者手动调同一个偏置，或者随机选方向做同样幅度的改动，能拿到多少提升。所以模型自己推理的贡献到底多大，还不好说。

### 局限

再说局限。所有结果都在 Qwen3.5 一个模型家族上，任务只有数学题，规模的结论，来自每个尺寸在自己权重上一次完整的会话。

损坏实验是人为放进去的单一故障。作者试过三种噪声强度，只有最强的 200 倍才能把模型弄坏。修复后的 33%，离原模型的 58.5% 也还有不小距离。

还有一点，生成本身不能完全复现，同样的权重和种子跑两次，答案也会分叉。所以单个例子说明不了什么，要看整体的统计检验。

### 未来方向

作者提到的方向有，给每次会话更多的测量预算，让改过的模型在自己身上继续跑下一轮，把损坏的位置、类型和强度做成分级，用来衡量修复能力怎么随规模变化，以及走出数学，只要有判断成功与否的反馈就能用。

他们也强调，这条通往递归自我改进的路，要有人在环里。模型改出来的每一个新版本，都要做安全和任务评测，并且由人确认之后才能使用。

我的判断是，这篇的意义不在 6.6 个点，而在于把训练和推理之间那堵墙推开了一条缝。每次改动都是模型写的代码，可以读，可以比较，也可以像代码提交一样回滚。下一步最值得看的，是换别的模型家族、别的任务，还能不能复现。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes mostly from the University of Illinois Urbana-Champaign, with co-authors at Carnegie Mellon, Princeton, Stanford and others, and its title is a claim: language models can rewrite their own weights.

The question is a bold one. Can a model, with no training and no gradients, just by reasoning, edit its own parameters and come out better? The paper's answer, for the twenty-seven billion parameter open model they test, is yes, with quite a few conditions. Let's go through them.

### Problem

Some background first. A language model is trained once and then frozen. To make it better, you either run another round of training, which takes people who know training, big clusters and a whole pipeline, or you change the prompt, wrap it in a harness, or spend more compute at inference, and all of that is gone once the task ends.

The authors want a third route: an improvement that lives in the weights, so the deployed model costs nothing extra per answer and needs no training infrastructure at all.

### Prior work

Earlier self-improvement falls roughly into three groups. Models rewrite their prompts or their agent code, and the weights stay put. Or they train on their own outputs, but it's still gradient descent that changes the weights. Or they critique themselves at inference time, which costs you again on every new input.

The closest prior work is a blog post in which a frontier model reads the weights of a small reasoning model, finds a direction behind its looping, and removes it, with no training. But that's a strong model editing a weaker one, and a model editing itself was left as future work. This paper picks up exactly that.

### Key idea

The key idea fits in one line: the model is its own optimizer. It gets its own weights, its graded answers on practice problems, and the ability to run itself, but no training of any kind. Every value it changes has to be something it reasoned out and wrote in.

The authors add a second task. The model gets a damaged copy of itself, where one block of weights has been overwritten with noise, so the copy gets almost nothing right. It's told only that some values were corrupted, not where or how, and has to fix it.

### Method

Here's the setup in this figure. In a session, the model writes a short rationale each turn, then takes one action. There are three kinds: run Python on a CPU to read and write weight matrices directly, generate answers with and without its edit on the same random seeds, which the authors call a paired test, or read its own hidden states on texts it picks.

The budget is fixed: at most twenty forward passes and two hundred code runs per session. Gradients are switched off in the Python session, and anything that computes them or runs an optimizer is refused. The input embedding, final normalization and output layer are off limits, and so is multiplying a whole tensor by one number, since that just acts like changing the sampling temperature.

The task asks for an edit that, on problems not used to choose it, clearly cuts the answers lost to a failure without significantly lowering accuracy. If no edit does both, it should submit nothing. That's how the model withdraws.

The models are Qwen3.5 at four sizes, two, four, nine and twenty-seven billion parameters, which mix Gated DeltaNet linear attention with full attention. Practice problems are three thousand math problems from DeepScaleR, cleaned of overlap with evaluations like MATH-500 and AIME, and another thousand are held out where no session can see them.

### Results

Start with the twenty-seven B model editing itself. With a four thousand ninety-six token budget, its starting diagnosis shows eighty point five percent of answers running out of tokens, but ninety-six percent of the finished answers are correct. Its own conclusion: the reasoning is fine, it just doesn't know when to stop.

It compares its hidden states on finished and unfinished answers, edits a late layer first, sees its tests say that hurts, and drops it. Looking earlier, it sees the difference grow with depth, so it moves to layer eight and removes five percent of that direction from the layer's two output matrices. All three paired tests favor the edit, and it submits.

It's a tiny change, zero point six and zero point one percent of the two matrices' norms. In the evaluation fixed in advance, four hundred ninety held-out problems with eight answers each, accuracy goes from fifty-seven point two to fifty-eight point three percent. That's one point one points, statistically significant, but small.

The bigger gains come from a second protocol. Every session starts from the original weights, and before the model submits, the framework checks the edit once on reserved practice problems the session can't read or use. Then the model decides to submit or withdraw.

All of the first eight sessions are reported. Five withdraw, three submit. Session eight's edit takes accuracy from fifty-seven point two to sixty-three point eight percent, up six point six points. Session two reaches sixty-one point three, up four points. Both are significant. Session one gains only half a point, which isn't.

Here's the interesting part. The six point six gain comes from a very simple edit: raise the time-step bias of all forty-eight linear-attention layers by zero point five. In the model's own account, that makes those layers decay faster so long reasoning gets stuck less. The share of answers that hit the token limit falls from thirty-three point six to twenty-six point one percent.

Now the repair task. The authors overwrite eight consecutive rows of the MLP output matrix in layer thirty of the twenty-seven B model with noise two hundred times the matrix's normal spread. The damaged copy solves none of the held-out problems.

The model debugs like an engineer. It sees hidden states blow up with depth, guesses the early layers are broken, and its first attempts fail. Then it computes, for every output matrix, the largest weight over the median one. Layer thirty's is one thousand three hundred ninety-seven times; no other matrix exceeds one hundred sixty-two point three.

Its fix clips every weight above fifteen times the median magnitude, in that matrix and nineteen other output matrices of layers zero to ten, touching zero point zero one three percent of their weights. Held-out accuracy goes from zero to thirty-three percent. The undamaged model scores fifty-eight point five.

The ablation shows that changing only the matrix it named gets the copy to fifty-five percent, while its changes to the other nineteen alone barely help. And in the damaged rows, it writes its own clipped values; it doesn't copy the original weights back.

The surprising part: the repair was found on text math problems only, yet on six image benchmarks it never saw, accuracy comes back from seven point three to seventy point two percent, eighty-three percent of the original model's. On ScreenSpot-v2, a screen-element grounding task, it goes from zero to eighty-nine percent, against ninety-three for the original.

Finally, scale. With the same task and tools, the two B model doesn't treat its weights as something it can change. The four B and nine B models add random noise to their weights and break themselves, and the nine B model's submitted edit drops its held-out accuracy from forty-seven point five percent to zero. Only the twenty-seven B model measures, edits, and then keeps or drops the edit based on its tests.

### Comparison

Against earlier routes, in the authors' own comparison table, self-rewriting is the only one that ticks all five boxes: it changes weights, needs no training, is chosen by the model, looks inside itself, and adds no cost per answer.

The authors also acknowledge that several of the twenty-seven B model's edits are, in form, direction ablation, a technique researchers already use. The difference is that here the model makes every choice itself.

My read is that the strongest thing here is the experimental discipline: an evaluation fixed in advance, paired tests, and all eight sessions reported, withdrawals included. But nearly all the gain comes from one symptom, answers that run too long, and the biggest edit is a single uniform bias shift. That looks more like the model finding a useful knob than learning new math.

My read is also that the paper doesn't report simpler controls, like a researcher tuning that same bias by hand, or random directions edited at the same strength. So how much of the gain is due to the model's own reasoning is still hard to say.

### Limitations

On limitations. Every result is on a single model family, Qwen3.5, the task is only math, and the scaling claim rests on one complete session per size on its own weights.

The damage is a single fault the authors planted. They tried three noise strengths, and only the strongest, two hundred times, broke the model. And thirty-three percent after the repair is still well short of the original fifty-eight point five.

One more thing: generation isn't exactly reproducible. Run the same weights with the same seeds twice and answers still diverge. So a single example proves nothing; what counts is the statistical test over many problems.

### Future work

The authors list several directions: more measurement budget per session, letting the edited model run the next session on itself, grading the planted damage by place, kind and strength to measure how repair scales, and moving beyond math, since the session only needs a check of success.

They also stress that this path toward recursive self-improvement should keep people in the loop. Every model a model makes of itself should be evaluated, for safety as well as for the task, and confirmed by a person before it's used.

My read is that the point of this paper isn't the six point six points. It's that it opens a crack in the wall between training and inference. Each change is code the model wrote, which can be read, compared and rolled back like a commit. What I'd watch next is whether this replicates on other model families and other tasks.

That's today's Daily Frontier Papers. The paper is linked in the description.

