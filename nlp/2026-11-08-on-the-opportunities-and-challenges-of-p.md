# On the Opportunities and Challenges of Prefill-to-Autoregressive Cross-Model Transfer

**大模型帮了忙还是只是微调**

- 论文 Paper: [arXiv 2611.06976](https://arxiv.org/abs/2611.06976)
- 单位 Affiliation: 伦敦国王学院 · 腾讯优图实验室
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-08
- 视频 Videos: [YouTube 中文](https://youtu.be/caOT0nBOMsU) · [YouTube English](https://youtu.be/qxusKR1pTyI) · [B站](https://www.bilibili.com/video/BV1iDWJ6VEpn)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇伦敦国王学院和腾讯优图实验室的论文，主题是大模型和小模型之间的隐式通信，大模型读题，小模型答题。

有意思的是，作者先做出了一个很强的方法，又自己动手检查，这些提升到底是不是来自大模型。答案是，有时候是，有时候不是。

### 问题

先说背景。大模型能力强，但逐个 token 生成又慢又贵。小模型便宜，但遇到复杂任务和长指令，常常做不好。

有一类思路叫 PACT，意思是从预填充到自回归的跨模型传递。大模型只负责读题，也就是预填充阶段，把对题目的内部表示传给小模型，后面的生成全部交给小模型。

这样大模型只出场一次，最费时间的解码由小模型完成。作者提到，这和预填充、解码分开部署的推理架构也是兼容的。

这条路已经有不错的结果。闭源方法 Mostik 报告说，用 753B 的发送方配 4B 的接收方，在数学和代码上补上了两者百分之四十七的差距。

但有个问题没人认真回答。小模型变强，是真的用上了大模型的信息，还是中间那座可训练的桥，本身就把小模型调得更好了？

### 以前的方法

以前的方法大多在 KV cache 上做文章，KV cache 可以理解成模型读题时的中间笔记。比如 C2C，把大模型的 cache 投影后融合进小模型的 cache，用监督微调训练。

KV-Lingo 先用线性回归初始化映射，直接用投影过来的 cache 替换小模型自己的，再用 KL 散度目标训练。

作者指出两个缺口。Mostik 闭源，没法复现。开源方法没有回答，生成很长时还有没有用，大小模型差距很大时行不行，混合注意力架构能不能用。

### 核心想法

核心想法分两半。第一半，是一个简单、开源、能扩展的 PACT 框架，叫 PLatour。

它把大模型的隐藏状态看成一种潜在的工具，不让小模型模仿大模型的输出，而是用强化学习按任务得分训练中间的桥，让小模型自己学会用这个工具。

第二半，是一套检查方法，叫 PACT-Check。出发点是，分数提高，不等于信息真的从大模型传了过去，得先排除别的解释。

### 方法

先看结构，就是这张图。大小两个模型都冻结，唯一训练的，是每层一个的线性投影。

读题时，大模型某一层的隐藏状态经过投影，直接加到小模型对应层的残差流上，残差流就是层与层之间传信息的主干道。只在读题阶段加，生成完全由小模型自己完成。

层按相对深度配对，小模型的中间层接大模型的中间层。对 Qwen3.5 这种混合注意力模型，小模型的全注意力层，只接大模型的全注意力层。

投影初始化为零，训练刚开始时，小模型的行为和原来完全一样。消融里，零初始化好过随机初始化，单层线性也好过两层 MLP。

传的是隐藏状态，不是 KV cache，这样才能支持有循环状态的混合架构。在作者试过的组合里，传隐藏状态都比传 KV cache 好。

训练用强化学习算法 GRPO，直接最大化任务得分，不做监督微调或蒸馏预热。在 MATH500 上，同样用 30B 教 4B，GRPO 是 88.05，监督微调 83.55，在线蒸馏 83.20，而 4B 小模型自己是 84.60。

再看 PACT-Check，一共四个对照。前两个只在推理时改动，不重新训练。

第一个是假传递，拿掉和题目相关的大模型特征，只留提示模板部分的固定平均特征。第二个是无关传递，小模型读原题，大模型读的却是另一道无关的题。

后两个要单独训练。第三个是自我通信，发送方和接收方都是小模型自己，桥就退化成一个普通的适配器。第四个是训练接收方，不要大模型，直接用 LoRA 或全参数微调小模型。

指标叫 PACT-Score，就是系统准确率减去四个对照里最强的那个。大于零，才算有证据说明大模型的信息起了作用。

### 实验和效果

实验分两套。第一套用 Qwen3，接收方 1.7B，发送方 8B 或 235B，有选择题，也有要写推理过程的生成题。第二套用 Qwen3.5，接收方 4B，发送方 35B 或 397B，测更难的数学和代码。

看表 1 的生成任务，235B 传给 1.7B。化学题从小模型单独的 41.43 提到 69.29，GSM8K 从 80.86 到 89.35，MATH500 从 73.40 到 78.2。

另外两个开源方法 C2C 和 KV-Lingo，在这三个生成任务上，两种发送方下全都不如小模型单独做。235B 发送方下，KV-Lingo 的 GSM8K 只有 17.06。

再看表 2，和闭源 Mostik 比。PLatour 用 35B 发送方，AIME 的 135 道题上是 74.63，Mostik 报告的是 70.74。HumanEval 94.97 对 86.13，MBPP 89.93 对 82.00，LiveCodeBench 基本持平。

作者也说，这不是严格控制的比较。Mostik 用的是 753B 的 GLM-5.2，训练数据没公开，而 PLatour 数学和代码各训了一座桥。

有意思的是，397B 发送方反而不如 35B。作者说训练中出现了一次明显的裁剪尖峰，之后性能大幅下降，恢复很有限。

速度上，读题要跑两个模型，首个 token 会慢一些。但输出越长，这部分固定成本越被摊薄，总时间接近小模型单独跑。

接下来是最关键的部分，PACT-Check 的结果，看这张图。每一行里，小模型单独记为 0，PACT 系统记为百分之百，柱子表示对照复现了多少提升。

化学生成题上，PLatour 提升了 27.86 个百分点，比最强的对照，也就是全参数 GRPO 微调小模型，还高 3.34 个点。假传递只剩 52.44，这是大模型的信息确实起了作用的证据。

但在 GSM8K 上，用无关题目的特征去传，分数是 89.12，和正常的 89.35 几乎一样，PACT-Score 只有 0.23。也就是说，大模型读的是哪道题，基本不影响结果。

MATH500 更明显。PLatour 提升了 4.80 个点，但不用大模型，直接用 LoRA 加 GRPO 训练小模型能到 82.25，反而更高，PACT-Score 是负 4.05。

Qwen3.5 这一套，数学平均从 48.90 提到 63.32，代码从 62.87 提到 79.73。但假传递和无关传递几乎复现了全部提升，全参数微调小模型还略高，PACT-Score 数学负 1.19，代码负 0.99。

所以，超过 Mostik 的那组结果，按作者自己的检查，没法单独归功于大模型传来的信息。

### 和以前方法的对比

用同样的标准看 C2C 和 KV-Lingo，在表 3。在 ARC-C 和 MMLU-Redux 两个选择题基准上，它们的 PACT-Score 明显大于零，在这两个基准上跨模型传递确实有用。

但在三个生成任务上，两者都是大幅负分。PLatour 的监督微调版本，在化学选择题上提升了 39.52 个点，比大模型自己的提升还多，却仍没超过最强对照。作者的结论是，连超过发送方，都不足以证明信息传了过去。

作者给了一个统一解释。大模型特征换成和题目无关的固定信号，桥就相当于 prefix tuning，在输入前加一段可学习的向量。自我通信时，它就是一个 adapter。所以可训练的桥，本身就可能起到微调小模型的作用，哪怕两边都冻结。

我的判断是，这篇最大的价值不在 PLatour，而在 PACT-Check。以后再有人说大小模型隐式通信有效，至少该和假传递、无关传递、直接微调小模型比一比。

### 局限

论文没有单独的局限章节，但讨论部分很坦白。用奖励训练桥，本身分不清提升来自传来的信息，还是桥自己学到的东西。小模型能自己变强时，大模型可能只是一个昂贵的噪声源。

另外，实验都在同一个模型家族内部，跨家族行不行没有测。发送方变大时训练也不稳定，397B 就是例子。

评测本身也难。作者说，需要小模型能用上信息、又有提升空间、大模型确实知道更多的任务，还要控制测试题泄漏。比如他们的 AIME 有 55 道和准备的训练数据池重合，所以单独报告了去掉重合后的 80 道。

还有一个落差。Qwen3.5 这一套里，PLatour 数学平均 63.32，35B 发送方自己是 68.37，离大模型还有距离。

### 未来方向

作者也提了几个机会。一是能力互补的模型之间，甚至弱模型给强模型传信息，也可能有用。

二是把读懂题和找答案分开扩展，大模型读题，小模型花更多时间找答案。

三是缓存长系统提示。比如 Agent 框架里几万 token 的固定指令，大模型的表示可以提前算好，每次请求直接注入。效果还在不在，作者说要验证。

代码已经在 GitHub 开源。

我的判断是，这篇对做推理加速和多模型协作的人是个提醒，只要中间有可训练参数，提升就可能来自微调，而不是通信。从这篇的结果看，在长的数学和代码推理上，还没有证据表明大模型的信息真正帮上了忙。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from King's College London and Tencent Youtu Lab, and it's about a big model and a small model talking in latent space: the big one reads the question, the small one writes the answer.

What makes it worth your time is that the authors build a strong method, and then turn around and test whether its gains really come from the big model. Sometimes they do. Sometimes they don't.

### Problem

Some background. Large models are capable, but generating token by token is slow and expensive. Small models are cheap, but they struggle with hard tasks and long instructions.

One line of work is called PACT, short for prefill-to-autoregressive cross-model transfer. The split is simple. The large model only reads the prompt, the prefill stage, and hands its internal representations to the small model. Every token after that is generated by the small model alone.

So the big model shows up once, and the expensive decoding stage runs on the small one. The authors also note that this fits serving setups that already run prefill and decode on separate machines.

There are promising results already. A closed-source method called Mostik reports closing forty-seven percent of the gap between a seven-hundred-fifty-three-billion-parameter sender and a four-billion-parameter receiver on math and coding.

But one question hasn't been answered properly. When the small model gets better, is it actually using what the big model sent? Or is the trainable bridge in the middle simply tuning the small model?

### Prior work

Most earlier methods work on the KV cache, which you can think of as the notes a model keeps while reading the prompt. C2C, for example, projects the big model's KV cache and fuses it into the small model's own cache, trained with supervised fine-tuning.

KV-Lingo initializes a mapping with linear regression, replaces the small model's cache with the projected one, and then trains with a KL-divergence objective.

The authors point to two gaps. Mostik is closed, so nobody can reproduce or study it. And the open methods leave key questions open: whether the benefit holds for long generations, for large size gaps between the models, and for hybrid-attention architectures.

### Key idea

The paper has two halves. The first is a simple, open, scalable PACT framework called PLatour.

It treats the big model's hidden states as latent tools. Instead of asking the small model to imitate the big model's outputs, it trains the bridge directly with reinforcement learning on task reward, so the small model learns how to use those tools.

The second half is a set of checks called PACT-Check. The starting point: a higher score doesn't prove that information actually crossed over from the big model. To claim that, you have to rule out the other explanations.

### Method

Here's the architecture in this figure. Both models are frozen. The only thing trained is a linear projection, one per layer.

While the prompt is being read, a hidden state from a sender layer is projected and added straight into the matching receiver layer's residual stream, the main path that carries information from layer to layer. This happens only during prefill. Generation is entirely the small model's job.

Layers are paired by relative depth, so the receiver's middle layers connect to the sender's middle layers. For hybrid-attention models like Qwen three point five, there's one more rule: a full-attention receiver layer can only connect to a full-attention sender layer.

The projections start at zero, so at the beginning of training the small model behaves exactly as before. In the ablations, zero initialization beats random initialization, and a single linear layer beats a two-layer MLP.

Another choice: the bridge carries hidden states, not KV caches. That's what lets it work with hybrid architectures that have linear attention and recurrent state. In every combination the authors tried, hidden-state transfer beat KV-cache transfer.

Training uses GRPO, a reinforcement learning method, to maximize the small model's task reward directly, with no supervised or distillation warm-up. On MATH five hundred, with a thirty-billion sender and a four-billion receiver, GRPO scores eighty-eight point zero five, supervised fine-tuning eighty-three point five five, and on-policy distillation eighty-three point two. The four-billion model alone scores eighty-four point six.

Now PACT-Check. It has four controls. The first two only change things at inference time, with no retraining.

Dummy transfer strips out the question-specific sender features and keeps only fixed, averaged features for the prompt template. Irrelevant transfer lets the small model read the real question while the big model reads a different, unrelated one.

The other two need their own training runs. Self-communication uses the small model as both sender and receiver, which turns the bridge into an ordinary adapter. Trained receiver drops the big model entirely and just fine-tunes the small model, with LoRA or full parameters.

Then there's one number, the PACT-Score: the system's accuracy minus the best of those four controls. Only a positive score counts as evidence that the big model's information helped.

### Results

There are two setups. One uses the Qwen three family, with a one-point-seven-billion receiver and an eight-billion or two-hundred-thirty-five-billion sender, on both multiple-choice and chain-of-thought generation tasks. The other uses Qwen three point five, with a four-billion receiver and a thirty-five-billion or three-hundred-ninety-seven-billion sender, on harder math and coding.

Table one, generation tasks, two-thirty-five-B into one-point-seven-B. On chemistry, the small model alone scores forty-one point four. With PLatour it reaches sixty-nine point three. GSM8K goes from eighty point nine to eighty-nine point four, and MATH five hundred from seventy-three point four to seventy-eight point two.

The two open baselines, C2C and KV-Lingo, do worse than the small model alone on all three generation tasks, with both senders. With the big sender, KV-Lingo drops to seventeen percent on GSM8K.

Table two compares against the closed Mostik. With a thirty-five-billion Qwen three point five sender, PLatour scores seventy-four point six on the hundred-thirty-five AIME problems, against Mostik's reported seventy point seven. HumanEval is ninety-five to eighty-six, MBPP ninety to eighty-two. LiveCodeBench is roughly a tie, fifty-four point three to fifty-four point one.

The authors say plainly this isn't a controlled comparison. Mostik uses a seven-hundred-fifty-three-billion GLM five point two sender, its training data isn't disclosed, and PLatour trains separate bridges for math and for code.

Oddly, the three-hundred-ninety-seven-billion sender does worse than the thirty-five-billion one. The authors trace it to a single sharp clipping spike during training, followed by a large, lasting drop with little recovery.

On speed, prefill has to run both models, so time to first token goes up. But that cost is fixed, and as outputs get longer, total time ends up close to the small model running alone.

Now the most important part of the paper, the PACT-Check results in this figure. In each row, the small model alone is zero, the PACT system is one hundred percent, and the bars show how much of that gain a control reproduces.

On chemistry generation, PLatour gains twenty-seven point nine points. The strongest control, full-parameter GRPO on the small model, still trails it by three point three. Dummy transfer falls to fifty-two point four. Here, there's evidence the big model's information really is doing work.

On GSM8K, though, feeding in features from an unrelated question scores eighty-nine point one, almost the same as the real eighty-nine point four. The PACT-Score is just zero point two three. In other words, which question the big model read barely matters.

MATH five hundred is starker. PLatour gains four point eight points, but training the small model alone with LoRA and GRPO reaches eighty-two point three, higher than PLatour. The PACT-Score is minus four point zero five.

In the Qwen three point five setup, math average climbs from forty-eight point nine to sixty-three point three, and code from sixty-two point nine to seventy-nine point seven. That looks big. But dummy and irrelevant transfer reproduce almost all of it, and full fine-tuning of the small model does slightly better. The PACT-Scores are minus one point two for math and minus one point zero for code.

So the results that beat Mostik can't, by the authors' own test, be credited specifically to information from the big model.

### Comparison

Applying the same yardstick to C2C and KV-Lingo, in table three: on the multiple-choice benchmarks ARC-C and MMLU-Redux, their PACT-Scores are clearly positive. On those two, cross-model transfer does help.

On the three generation tasks, both methods score well below zero. And PLatour's supervised variant gains thirty-nine point five points on multiple-choice chemistry, more than the big model's own gain, yet still doesn't beat the strongest control. The authors' conclusion: even outperforming the sender doesn't prove that information was transferred.

They offer a unified explanation. If the sender's features are swapped for a fixed signal unrelated to the question, the bridge is essentially prefix tuning, a learned vector placed in front of the input. In self-communication, it's an adapter. So a trainable bridge can end up fine-tuning the small model, even with both models frozen.

My read is that the lasting contribution here isn't PLatour, it's PACT-Check. Anyone claiming latent communication between big and small models works should, at minimum, compare against dummy transfer, irrelevant transfer, and simply fine-tuning the small model.

### Limitations

There's no separate limitations section, but the discussion is candid. The central problem is that reward-driven training can't tell whether a gain comes from the transmitted information or from what the bridge learned on its own. When the small model can improve by itself, the big model may end up as an expensive source of noise.

Also, every experiment stays inside one model family, Qwen three to Qwen three, Qwen three point five to Qwen three point five. Cross-family transfer is untested. Training also gets unstable as the sender grows, as the three-hundred-ninety-seven-billion run shows.

Evaluation is hard too. The authors say you need tasks where the small model can use the information but still has room to improve, where the big model really knows more, and where test contamination is controlled. Fifty-five of their AIME problems overlap with their prepared training data pool, so they also report the eighty that don't.

And there's still a gap. In the Qwen three point five setup, PLatour's math average is sixty-three point three, while the thirty-five-billion sender alone scores sixty-eight point four.

### Future work

The authors see openings too. Transfer might help not just big-to-small, but between models with complementary skills, even from a weaker model to a stronger one.

Another is scaling problem understanding and solution search separately: a bigger model to read the problem, a smaller one to search longer for the answer.

A third is caching long system prompts. An agent harness can carry tens of thousands of tokens of fixed instructions. The big model's representation of that prefix could be computed once, cached, and injected into every request. Whether that keeps the benefit is, they say, still an open question.

The code is open source on GitHub.

My read is that this paper is a warning for anyone working on inference speedups or multi-model systems. Whenever there are trainable parameters in the middle, a gain may come from fine-tuning rather than communication. PACT is promising, but on the evidence here, there's no sign yet that the big model's information is what helps on long math and coding reasoning.

That's today's Daily Frontier Papers. The paper is linked in the description.

