# Beyond Self-Consistency: Recursive Self-Improvement at Test Time

**不靠多数投票让模型自己改对**

- 论文 Paper: [arXiv 2611.30965](https://arxiv.org/abs/2611.30965)
- 单位 Affiliation: 上海人工智能实验室、上海交通大学等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-28
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1YA1MzGEfX)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇上海人工智能实验室、上海交通大学等单位的论文，题目叫 Beyond Self-Consistency，超越自洽性，副标题是，测试时的递归自我改进。

它想回答的问题是，一个模型部署以后，面对没有标准答案的新题，能不能只靠自己，越做越好。

### 问题

先说背景。模型上线以后来了一批新题，没有标签，没人告诉它答案对不对。这时候有两类做法，一类叫测试时扩展，在每道题上多花推理算力，另一类叫测试时训练，直接拿这些没标签的题去更新模型权重。

理想情况下，这两件事能形成一个循环，答案更好，就能训练出更好的模型，更好的模型又给出更好的答案，全程不需要外部监督。

### 以前的方法

现在的主流做法，推理和训练都靠同一个东西，叫 self-consistency，也就是自洽性。简单说，同一道题独立采样很多次，然后多数投票，票数最多的答案，要么直接当最终答案，要么当训练目标。

作者指出，这有两个问题。第一，推理效率低。每次尝试都从零开始，前面推过的东西全扔了，多花的算力只换来更多独立样本，而不是更好的推理。

第二，监督不可靠。多数投票奖励的，是模型本来就最常给出的答案，如果多数本身是错的，训练就会把这个错误越练越牢。

这张图就是对比，左边是自洽性，并行采样，然后投票，右边是这篇论文的方法，TT-RSI。

### 核心想法

作者的出发点很朴素，人解难题不是这样做的。人会在上一次尝试的基础上接着改，保留有用的推理，修正错误。

所以整个框架建立在一个改进步上，就是把上一次的解答放进上下文，让模型把这道题重新解一遍。推理的时候反复用这一步，训练的时候，让模型学习自己改出来的结果，而不是学多数投票。

关键在于，就算大多数尝试都给出同一个错答案，某一次修改仍然可能把它改对，所以训练信号有机会超出模型原来的多数。

### 方法

框架叫 TT-RSI，测试时递归自我改进，分成两部分。总览图上半部分是 In-Context RSI，管推理，下半部分是 In-Weight RSI，管训练。

先说 In-Context RSI。模型先正常解一遍，得到第一个解答。然后把题目，加上上一次的完整解答，交给模型再解一遍。每次只带最新的一个解答，历史不会越堆越长。

这里有两个设计。第一，给的是完整解答，包括中间推导和最终答案，不只是答案。这样即使最终答案错了，中间对的步骤也能被复用。

第二，提示词是中性的。只说这是之前的一次尝试，不说它对不对，也不要求纠错，而是让模型独立解题，自己决定是沿用、重做，还是换个思路。

为什么要中性。作者引用以前的研究，太依赖上一次解答，模型会重复自己，探索变少，而直接要求纠错，又可能把本来对的答案改错。

论文用一个简单模型描述这种平衡。每一轮修改，有一定概率把错的改对，叫修复率，也有一定概率把对的改错，叫破坏率。反复修改最终收敛到的准确率，叫平衡点，由这两个比率决定。所以好的设计，要提高修复率，同时压低破坏率。

再说 In-Weight RSI。In-Context RSI 的改进只对当前这道题有效，In-Weight RSI 要把改进写进模型权重里。

它用的工具叫在线自蒸馏，on-policy self-distillation。做法是让冻结的初始模型当老师，老师额外看到一份参考解答，学生看不到，然后在学生自己生成的内容上，逐个 token 给出指导。

每道题只需要两份参考，第一个解答 a1，和它修改一次后的 a2。第一项叫答案蒸馏，看过 a1 的老师，教只看到题目的学生，相当于把一步改进压缩进第一次作答。

第二项叫修改蒸馏，学生看着 a1 做修改，老师看的是更好的 a2，这样修改这一步本身也变强了。这两项合起来，叫递归蒸馏。

还有第三项，叫加速蒸馏，直接让看过 a2 的老师来教第一次作答。而且按 token 加权，老师看到 a1 和看到 a2 时预测差别越大的位置，权重越高，也就是重点学修改真正改动的地方。

为什么两份参考就够。因为每次更新后模型变强了，再生成的 a1、a2，就相当于原来修改链里更靠后的尝试。作者在理想化的假设下证明，这样反复蒸馏，第一次作答会逐步逼近初始模型无限次修改的极限，加速蒸馏让这个过程更快。

两部分就这样形成闭环。修改的结果训练出更好的模型，更好的模型修改得更好，又产生更好的参考。

### 实验和效果

实验设置。模型是 Qwen3 的 1.7B、4B 和 8B，评测用五个数学竞赛基准，2025 年和 2026 年的 AIME，2025 年和 2026 年 2 月的 HMMT，还有 2025 年的 BRUMO。每个方法分别在每个基准的无标签题目上做适应，标签只用来评分。

对比的基线是五个无标签测试时训练方法，TTRL、TTPO、OPSD-TTT、CANON 和 U-OPSD，大多依赖多数投票或者多数一致。另外还有一个关键对照，TTPO 加自洽性投票，用和 TT-RSI 一样的推理预算。

实现上，推理时一条修改链是 11 次尝试，也就是一次初始解答加十次修改。训练是 100 次更新。

主要结果在表 1。先看只训练、不做推理扩展的第一次作答。In-Weight RSI 在三个尺寸上，都是无标签训练方法里最好的，平均比原始模型高 5.79 个百分点，15 个设置里，有 13 个超过全部五个基线，另有 1 个并列最好。

完整的 TT-RSI，也就是训练后的模型再跑修改链，平均比原始模型高 13.89 个百分点。比如 4B 模型，五个基准的平均分从 55.76 提到 69.64。

和同等预算的 TTPO 加自洽性投票比，TT-RSI 平均高 4.79 个百分点，15 个设置里赢 11 个，平 2 个。8B 模型在 2026 年 AIME 上，是 86.67 对 73.33。

作者的解释是，自洽性训练可能强化错误的多数，再多采样往往也只是重复同样的错。在 8B 模型、2026 年 AIME 这一组里，自洽性只把 TTPO 提高了 2.5 个百分点，而 In-Context RSI 把原始模型提高了 8.89 个百分点。

还有一个有意思的点。1.7B 模型训练后，第一次作答就有 41.99，已经超过没训练的模型改十次之后的 39.73。作者认为，这说明后面的改进被压缩进了第一次作答。

推理效率看这张图，横轴是累计生成的 token，纵轴是准确率。In-Context RSI 在第五次尝试时，准确率就超过了十个样本的自洽性投票，生成的 token 少了百分之 68.6。

训练成本也更低。在四张 H20 GPU 上，4B 模型每次更新分别是 8.53 分钟和 10.06 分钟，比基线快 2.29 到 5.36 倍，每次更新用的 token 也最少。

论文还举了一个例子，一道 2026 年 HMMT 的概率题。11 个独立样本里，3 个给出同一个错答案，只有 2 个答对，投票选了错的。原始模型用修改链，到第 11 次尝试改对了，训练后的模型，第 2 次就改对了。

### 和以前方法的对比

再看最关键的问题，能不能突破错误的多数。作者在 2026 年 AIME 上，挑出原始 4B 模型多数投票答错的 6 道题，每题采样 64 次。

在这 6 道题上，好几个依赖多数投票的基线，反而比原始模型还差。In-Weight RSI 把平均准确率从 5.73 提到 12.76，多数投票也改对了其中 2 道，对比方法里只有 OPSD-TTT 改对了 1 道。

设计上的消融也支持作者的选择。在 4B 模型上，完整的 In-Context RSI 改十次后，平均准确率从 53.84 提到 63.59。只给上一次的最终答案、不给推导，准确率低 4.28 个百分点，破坏率差不多翻倍。明确要求模型检查纠错，准确率低 3.16 个百分点，修复率也下降。换成一道无关题目的解答，结果比原始模型还差。

训练这边，答案蒸馏和修改蒸馏一起用，比单用任何一项都好，再加加速蒸馏，还有小幅提升。用随训练不断更新的模型生成参考，比一直用初始模型生成参考，改十次后在 2026 年 AIME 上高 8.89 个百分点。

迁移方面，在一个基准上训练的 4B 模型，拿到其他基准上测，六组组合里，In-Weight RSI 都是最高的，也都超过原始模型。

我的判断是，这篇最有价值的地方，是用同一个改进步，把测试时扩展和测试时训练串了起来，而且训练信号来自修改，不是投票，正好绕开了多数投票强化错误的老问题。

我的判断是，也要看到两点。错误多数那组实验只有 6 道题，证据还偏少。另外 TT-RSI 的提升，有很大一部分来自推理时的修改链本身，训练带来的额外增益，是在原始模型加修改链之上再高 7.33 个百分点。

### 局限

作者自己写的局限很短，评测只覆盖数学推理，模型只用了 Qwen3 系列，最大到 8B。

还要注意，主实验是在每个基准自己的无标签题目上做适应，再在同一批题上评测。4B 模型换到没训练过的基准上，平均增益小一些，分别是 2.37 和 3.05 个百分点。

推理时每道题要跑一次初始解答加十次修改，每个回答最多可以生成 38912 个 token。和 TTPO 加自洽性的主对比，是在相同的测试时扩展预算下做的。

### 未来方向

作者提到的方向，是把 TT-RSI 扩展到开放式任务和会用工具的 Agent，在那里改进发生在多轮交互里，还要能跨任务保留下来。

我的判断是，接下来最值得看两点。一是没有明确答案的任务里，中性的修改提示还能不能保持低破坏率。二是模型变大以后，增益还剩多少，论文里完整框架相对原始模型的提升，从 1.7B 的 15.64，到 4B 的 13.88，再到 8B 的 12.14，是在缩小的。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the Shanghai Artificial Intelligence Laboratory, Shanghai Jiao Tong University and several other groups. It's called Beyond Self-Consistency: Recursive Self-Improvement at Test Time.

The question is easy to state. Once a model is deployed and faces new questions with no answer key, can it keep getting better using nothing but itself?

### Problem

Some context first. After deployment, the model sees a batch of new questions, with no labels and nobody telling it whether its answers are right. There are two ways to use that moment. Test-time scaling spends more inference compute on each question. Test-time training updates the model's weights on those unlabeled questions.

Ideally, the two feed each other. Better answers train a better model, the better model produces better answers, and the gains keep stacking up without any outside supervision.

### Prior work

Today, both sides mostly lean on one trick: self-consistency. Sample the same question many times independently, take a majority vote, and use the winning answer either as the final output or as the training target.

The authors point to two problems. First, inference is inefficient. Every attempt starts from scratch and throws away the earlier reasoning, so extra compute buys more independent samples, not better reasoning.

Second, the supervision is unreliable. A majority vote rewards whatever the model already says most often. When that majority is wrong, training locks the mistake in.

This figure puts the two side by side. On the left, self-consistency: parallel attempts and a vote. On the right, this paper's method, TT-RSI.

### Key idea

The starting point is simple. That's not how people solve hard problems. People build on their last attempt, keep the reasoning that worked, and fix what didn't.

So the whole framework rests on one improvement step: put the previous solution in the context and have the model solve the question again. At inference, that step is repeated. In training, the model learns from its own refinements instead of from majority votes.

The key point is that even when most attempts agree on the same wrong answer, a refinement can still fix it. So the training signal can move the model past its original majority.

### Method

The framework is called TT-RSI, test-time recursive self-improvement, and it has two parts. In the overview figure, the top half is In-Context RSI, which handles inference, and the bottom half is In-Weight RSI, which handles training.

Start with In-Context RSI. The model first solves the question normally. Then it gets the question again, along with its full previous solution, and solves it once more. Only the latest solution is carried forward, so the context doesn't keep growing.

Two design choices matter here. First, the reference is the full solution, derivation and final answer, not just the answer. So even a solution with a wrong final answer can contribute correct steps.

Second, the prompt is neutral. It calls the reference a previous attempt, says nothing about whether it's right, doesn't ask for corrections, and tells the model to solve the problem independently. The model decides for itself whether to reuse it, rework it, or try a new approach.

Why neutral? The authors cite earlier work showing that leaning too hard on a previous solution makes a model repeat itself and explore less, while explicitly asking for corrections can make it change answers that were already right.

The paper captures this balance with a simple model. Each refinement round fixes a wrong answer with some probability, the fix rate, and breaks a correct one with some probability, the break rate. The accuracy that repeated refinement settles at, which the authors call the balance point, depends on those two rates. So a good design raises the fix rate and keeps the break rate low.

Now In-Weight RSI. The gains from In-Context RSI only last for the current question. In-Weight RSI writes them into the model's weights.

Its tool is on-policy self-distillation. A frozen copy of the initial model acts as the teacher. The teacher gets to see a reference solution that the student doesn't, and it gives token-level guidance on text the student generated itself.

Each question needs just two references: the first solution, a one, and its first refinement, a two. The first loss is answer distillation. A teacher that has seen a one guides a student that only sees the question, which folds one improvement step into the first attempt.

The second is refinement distillation. The student refines a one, while the teacher sees the better a two, so the refinement step itself gets stronger. Together, these two terms are called Recursive Distillation.

There's a third term, Accelerated Distillation, where the teacher that saw a two supervises the first attempt directly. It's weighted token by token: wherever the teacher's prediction shifts most between seeing a one and seeing a two, that token counts more. In other words, the model focuses on what the refinement actually changed.

Why are two references enough? Because after each update the model is stronger, so its new a one and a two correspond to later attempts in the original chain. Under idealized assumptions, the authors prove that repeating this moves the first attempt toward the initial model's limit after infinitely many refinements, and Accelerated Distillation speeds that up.

That closes the loop. Refinements train a better model, the better model refines better, and that produces better references for the next round.

### Results

The setup. The models are Qwen3 at 1.7, 4 and 8 billion parameters. Evaluation uses five math competition benchmarks: AIME 2025 and 2026, the February 2025 and 2026 HMMT, and BRUMO 2025. Each method adapts separately to the unlabeled questions of each benchmark, and labels are only used for scoring.

There are five label-free test-time training baselines: TTRL, TTPO, OPSD-TTT, CANON and U-OPSD, most of which rely on majority voting or majority agreement. The key extra comparison is TTPO plus self-consistency voting, given the same inference budget as TT-RSI.

For implementation, each refinement chain at inference has eleven attempts: one initial solution and ten refinements. Training runs for one hundred updates.

The main results are in Table 1. First, the first attempt alone, after training but without any test-time scaling. In-Weight RSI is the best label-free training method at all three model sizes. On average it's five point seven nine percentage points above the base model, and it beats all five baselines in thirteen of the fifteen settings, tying for best in one more.

The full TT-RSI, the trained model plus the refinement chain, averages thirteen point eight nine points above the base model. For the 4B model, for instance, the five-benchmark average goes from fifty-five point seven six to sixty-nine point six four.

Against TTPO plus self-consistency at the same budget, TT-RSI is four point seven nine points higher on average, winning eleven of fifteen settings and tying two. On AIME 2026 with the 8B model, it's eighty-six point six seven versus seventy-three point three three.

The authors' explanation: training on self-consistency can reinforce a wrong majority, so more sampling tends to reproduce the same error. In that same 8B, AIME 2026 setting, self-consistency lifts TTPO by only two point five points, while In-Context RSI lifts the base model by eight point eight nine.

One more interesting point. At 1.7B, the trained model's first attempt scores forty-one point nine nine, already above the untrained model after ten refinements, at thirty-nine point seven three. The authors read this as later improvements being compressed into the first attempt.

For inference efficiency, look at this plot: cumulative generated tokens across, accuracy up. In-Context RSI passes ten-sample self-consistency by its fifth attempt, while generating sixty-eight point six percent fewer tokens.

Training is cheaper too. On four H20 GPUs, an update for the 4B model takes eight point five three minutes on one benchmark and ten point zero six on the other, two point two nine to five point three six times faster than the baselines, with the fewest tokens per update.

The paper also walks through an example, an expected-value problem from HMMT 2026. Of eleven independent samples, three agree on the same wrong answer and only two get it right, so the vote picks the wrong one. The base model's refinement chain gets it right on attempt eleven. After training, it gets it right on attempt two.

### Comparison

Now the question that matters most: can the method get past a wrong majority? On AIME 2026, the authors take the six questions where the base 4B model's majority vote is wrong, and sample each one sixty-four times.

On those six questions, several majority-based baselines actually do worse than the base model. In-Weight RSI raises average accuracy from five point seven three to twelve point seven six percent, and its majority vote becomes correct on two of the six. Among the other methods, only OPSD-TTT gets one.

The design ablations back up the choices. On the 4B model, full In-Context RSI lifts average accuracy from fifty-three point eight four to sixty-three point five nine after ten refinements. Passing only the previous final answer, without the derivation, costs four point two eight points and roughly doubles the break rate. Explicitly telling the model to check and correct costs three point one six points and lowers the fix rate. Swapping in a solution to an unrelated question does worse than the base model.

On the training side, answer and refinement distillation together beat either one alone, and Accelerated Distillation adds a little more. Generating references with the evolving model, rather than the fixed initial model, is eight point eight nine points better on AIME 2026 after ten refinements.

For transfer, a 4B model trained on one benchmark and tested on others comes out highest in all six source and target pairs, and above the base model in each.

My read is that the most valuable thing here is using a single improvement step to connect test-time scaling and test-time training, with a training signal that comes from refinement rather than voting. That sidesteps the old problem of majority votes reinforcing mistakes.

My read is also that two caveats apply. The wrong-majority experiment covers only six questions, which is thin evidence. And much of TT-RSI's gain comes from the refinement chain itself: what training adds on top of the base model with the chain is seven point three three points.

### Limitations

The authors' own limitations section is short. The evaluation covers only mathematical reasoning, and only the Qwen3 family, up to 8 billion parameters.

Also note that the main experiments adapt on each benchmark's own unlabeled questions and then evaluate on those same questions. For the 4B model on benchmarks it wasn't adapted to, the average gains are smaller: two point three seven and three point zero five points.

At inference, each question runs one initial solution plus ten refinements, and each response can generate up to thirty-eight thousand nine hundred and twelve tokens. The main comparison with TTPO plus self-consistency is made at the same test-time scaling budget.

### Future work

The authors point toward extending TT-RSI to open-ended tasks and tool-using agents, where improvement happens over multi-turn interactions and has to carry across tasks.

My read is that two things are worth watching next. One, whether the neutral refinement prompt keeps the break rate low on tasks without a clear right answer. Two, how much of the gain survives at larger scale. In this paper, the full framework's lead over the base model shrinks from fifteen point six four points at 1.7B, to thirteen point eight eight at 4B, to twelve point one four at 8B.

That's today's Daily Frontier Papers. The paper is linked in the description.

