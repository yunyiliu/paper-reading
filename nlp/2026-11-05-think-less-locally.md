# Think Less, Locally: Local Preference Learning for Reasoning Compression

**推理模型少写四成准确率基本不变**

- 论文 Paper: [arXiv 2611.04814](https://arxiv.org/abs/2611.04814)
- 单位 Affiliation: 西安交通大学 · 腾讯微信 AI
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-05
- 视频 Videos: [YouTube 中文](https://youtu.be/mEEVkC1C90w) · [YouTube English](https://youtu.be/FIoihUgbzu4) · [B站](https://www.bilibili.com/video/BV1TF4v65EL7)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇西安交通大学和腾讯微信 AI 合作的论文，名字叫 Think Less, Locally，意思是，少想一点，而且是在局部少想。

它要解决的是推理模型想太多的问题。作者的方法叫 LoCo，在五个数学基准上，把输出长度砍掉了四成多，准确率基本不变，有的还涨了。

### 问题

先说背景。像 DeepSeek-R1 这样的推理模型，回答之前会先写一大段思考过程，也就是 chain-of-thought，思维链，动不动就是几千个 token。

这很贵，延迟和显存占用都会上去。更关键的是，想得长不一定答得对。模型常常已经算出答案了，还在继续想，这叫 overthinking，过度思考。

论文引用的研究认为，在模型本来就能做对的题上，过度思考更多是偏好问题，不是能力问题。问题不在它能不能答对，而在它什么时候选择停下来。

### 以前的方法

以前压缩推理长度，主要有两类做法。第一类看整条轨迹，比如让模型对同一道题生成长短不同的正确答案，用偏好学习让它偏向短的，或者在强化学习里给短的正确答案更高的奖励。

问题在于，整条轨迹的比较，只告诉模型短的更好，没告诉它到底是哪一段想多了。两个答案可能走了完全不同的路，很难看出哪一部分能安全删掉。

第二类是细粒度的做法，比如删掉多余的 token 或句子，把某一步压短，或者从同一段前缀出发，比较不同的续写。

但作者认为，这些方法的着眼点还是轨迹本身，问的是哪些内容该删，而不是在当前这个状态下，还有没有必要继续想。

### 核心想法

这就是这篇论文的核心想法。把推理过程切成一步一步，每一步结束的地方，都是一个决策点，模型要么接着想，要么收尾给答案。

同一个模型，在这一步可能需要继续想，到了另一步就该停了。所以压缩不该一刀切，而应该在具体的状态上，教模型判断是继续还是结束。

LoCo 是 Local Compression 的缩写，就是局部压缩。它从模型自己的推理里找出想多了的位置，在同一个位置上，对比不同的续写，让模型学会在这里收尾。

### 方法

这张图是整个流程，一共三步。先检测过度思考的位置，再从这个位置重新生成更短的正确续写，最后组成偏好对来训练。

第一步是检测。把整段思维链喂回模型自己，看它在每个位置对下一个 token 有多确定，在一步之内求平均，就是这一步的置信度。

再算波动，也就是相邻两步的置信度变了多少。如果一步的置信度低于第 25 百分位，同时波动高于第 75 百分位，这一步就被标为候选。

为什么要两个条件同时满足？作者的解释是，置信度低，可能只是这一小步本身难，波动大，可能只是换了个思路。两个一起出现，才说明模型既不确定，又不稳定。

光看置信度还是不够可靠，所以还有第二道关，交给一个更强的判断模型，Qwen3-30B。它读当前的推理状态和下一步，判断下一步是在重复已经确定的东西，是在正常推进，还是在给出最终答案。只有确认是重复的，才留下来。

为什么要单独排除最终答案？因为推理模型收尾时，常会把答案再写一遍，看着重复，但删了就提取不到答案了。

第二步，在这个位置把前面的推理冻结住，让模型自己从这里重新往下写，只保留答案正确的，再挑最短的一个。只有它比原来的续写更短，这个位置才保留下来。

第三步，组成偏好对。同一个前缀后面，短的正确续写当作好的，原来那段长的当作差的，用 DPO 训练。DPO 是直接偏好优化，让模型更倾向于好的那个回答。

因为两个续写接的是完全相同的前缀，模型学到的就不是笼统的越短越好，而是，走到这一步的时候，该收尾了。

论文附录里有个例子。题目是，已知 2030 年 7 月 3 日是星期三，问 7 月 14 日是星期几。到第 28 步的时候，模型已经用算术、逐日列表、取模三种方法确认了，是星期天。

原来的续写又换着方法算了好几遍，用了 986 个 token。重新生成的短续写直接收尾，只用了 212 个 token。

### 实验和效果

实验设置。被训练的模型是 DeepSeek-R1-Distill-Qwen-7B，训练数据全部来自它自己在一个 14027 道数学题的训练集上生成的推理过程。

每道题采样三次，取最短的正确答案当原始轨迹。每个检测到的位置重新生成 8 个续写，至少省下 300 个 token 的才算数，每道题最多留 3 对。训练用的是 LoRA，只训练一小部分额外参数，跑两个 epoch。

评测有 5 个数学基准，从较简单的 MATH-500，到最难的 AIME 2024 和 AIME 2025，另外还有 3 个代码基准。每道题都生成 8 次。

对比的基线有两个，DAST 和 TokenSqueeze，都是直接用它们公开的完整权重来评测。

结果在表 1。五个数学基准上，LoCo 的输出长度都减少了百分之 41.7 到百分之 45.4。

准确率呢？在 MATH-500 上基本持平，原始模型 91.88，LoCo 是 91.80，另外四个基准上都有提升。

最难的 AIME 2024 上，准确率从 50.83 提到 54.17，平均输出从 13493 个 token 降到 7364 个，差不多砍掉一半。

两个基线就没这么好。DAST 压缩得很少，在难题上输出反而变长了，AIME 2024 上长了百分之 6.9。TokenSqueeze 压得狠，但难题上准确率下降，AIME 2024 上掉了 2.08 个点。

代码的结果在表 2。注意，训练只用了数学题。在 MBPP 上，准确率不变，输出少了百分之 36.3。更难的 LiveCodeBench 和 CodeContests，输出少了两成多，准确率还分别涨了 0.74 和 2.13 个点。

作者认为，既然训练的是数学、测试的是代码，这说明 LoCo 改变的是模型对推理长度的偏好，而不是教了它新的领域知识。

这张图换了个角度，横轴是允许的最大输出长度，纵轴是准确率。在 AIME 2024 和 CodeContests 上，大部分预算下，LoCo 的曲线都在 TokenSqueeze 上面。

### 和以前方法的对比

消融实验在表 3，回答的是，LoCo 的效果到底来自哪里。第一个对比叫 Global，题目和训练设置都一样，但不做局部，直接拿同一道题最短和最长的正确答案组成偏好对。

结果，在一些基准上准确率差不多，压缩幅度却差很多。Global 在 AMC23、AIME 2024 和 AIME 2025 上，只省了百分之 5.2、11.1 和 15.8，LoCo 是 41.7、45.4 和 45.2。

第二个对比，切点不变，只把短续写换成 GPT-5.4 来写。结果模型自己写的反而更好，AIME 2024 上，GPT-5.4 版本压缩了百分之 31.8，LoCo 是 45.4。作者的解释是，自己写的续写更贴近自己的推理风格，更好学。

第三个对比，切点不用检测器，改成随机选，或者等间距选。长度同样能压下去，但准确率掉了，AIME 2024 上随机切点是 48.33，比原始模型还低 2.5 个点，LoCo 是 54.17。

附录还给了一个数据。检测器选的切点偏靠后，大约一半落在最后百分之 40 的步骤里。随机和等间距的切点更靠前，作者说，也就是在模型还没想清楚的时候就压缩了。

我的判断是，这几组消融里最有说服力的是 Global。只是把偏好从整条轨迹挪到具体位置，压缩幅度就从一成左右变成四成多，说明在哪里压，比要不要压更关键。

我的判断是，也要看到，实验里只比了两个基线。前面提到的那些同样在步骤级别做压缩的方法，比如 SSPO 和 ReCUT，论文在相关工作里讨论了，但没有放进实验里正面比较。

### 局限

论文没有单独写局限，下面几点来自实验设置本身。第一，所有实验只训练了一个模型，就是 7B 的 DeepSeek-R1-Distill-Qwen，而且用的是 LoRA。换更大的模型、做全参数训练会怎样，论文没有测。

第二，AIME 每年只有 30 道题，表 1 里 AIME 准确率的正负范围在 3.75 到 6.61 之间，和三个点左右的提升差不多大，甚至更大。相比之下，输出长度的下降要明显得多。

第三，造训练数据要多一个 30B 的判断模型，每个位置还要重新生成 8 次，这部分额外的计算开销，论文没有报告。

第四，训练数据只有数学题。代码上验证了有效，但更开放的任务，比如写作或者多轮 Agent 任务，论文没有测试。

### 未来方向

作者说，代码和模型会在 GitHub 上公开。

我的判断是，这篇最有价值的地方，是把问题从怎么让推理变短，换成了在哪一步该停。接下来值得看的，是这套办法在更大的模型、更长流程的任务里还灵不灵。

我的判断是，对做部署的人来说，同样的准确率，生成的 token 少四成，成本和等待时间都会跟着降，这往往比再提一两个点准确率更实在。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Xi'an Jiaotong University and Tencent's WeChat AI team. It's called Think Less, Locally.

The problem is reasoning models that think too much. The authors' method, LoCo, cuts output length by more than forty percent on five math benchmarks, while accuracy holds steady or goes up.

### Problem

Some background. Reasoning models like DeepSeek-R1 write out a long chain of thought before they answer, often thousands of tokens of it.

That's expensive. Latency and memory both go up. And longer doesn't mean more accurate. Models often reach the right answer early and then keep going. That's called overthinking.

The work the paper cites argues that when a model can solve the problem, overthinking is more about preference than capability. The question isn't whether it can get the answer. It's when it chooses to stop.

### Prior work

Earlier compression methods come in two flavors. The first works on whole trajectories. For example, sample short and long correct solutions to the same problem and use preference learning to favor the short one, or reward shorter correct answers in reinforcement learning.

The catch is that comparing whole solutions tells the model shorter is better, but not where the wasted reasoning was. Two solutions can take completely different paths, so it's hard to tell which part was safe to cut.

The second flavor is fine-grained. Prune redundant tokens or sentences, compress individual steps, or compare different continuations that start from the same prefix.

But the authors argue these methods still treat the trajectory itself as the thing to compress. They ask what to remove, not whether more reasoning is needed from where the model is right now.

### Key idea

That's the core idea. Split the reasoning into steps. The end of every step is a decision point: keep thinking, or wrap up and answer.

The same model might need to keep going at one step and should stop at another. So compression shouldn't be applied uniformly. It should be a decision made state by state.

LoCo stands for Local Compression. It finds the spots in the model's own reasoning where it started overthinking, compares different continuations from that exact spot, and teaches the model to wrap up there.

### Method

Here's the whole pipeline, in three stages. Detect overthinking states, regenerate shorter correct continuations from those states, and turn them into preference pairs for training.

Stage one is detection. The full trace is fed back into the model, and at every position they record how confident it is about the next token. Average that over a step and you get the step's confidence.

Next comes volatility, how much confidence changed from the previous step. A step whose confidence is in the bottom quarter and whose volatility is in the top quarter becomes a candidate.

Why require both? The authors explain that low confidence alone can just mean a hard sub-problem, and high volatility alone can just mean a legitimate change of approach. Together, they flag a model that is both uncertain and unstable.

Confidence still isn't reliable enough on its own, so there's a second check by a stronger judge model, Qwen3-30B. It reads the current state and the next step, and decides whether that step re-deliberates on something already established, makes real progress, or presents the final answer. Only confirmed overthinking survives.

Why single out the final answer? Because reasoning models often finish by restating or boxing the answer. It looks repetitive, but it's needed to extract the answer, so cutting it would lose the answer itself.

Stage two. Freeze the reasoning up to that point, and let the model write fresh continuations from there. Keep only the ones with a correct answer, and pick the shortest. The state is kept only if that continuation is strictly shorter than the original.

Stage three builds the preference pairs. Same prefix, the short correct continuation is preferred, the original long one is rejected, and the model is trained with DPO, direct preference optimization, which pushes it toward the preferred answer.

Because both continuations share the exact same prefix, the model doesn't learn a vague "shorter is better." It learns that once it reaches this state, it's time to conclude.

The appendix has an example. The question: if July third, twenty thirty, is a Wednesday, what day is July fourteenth? By step twenty-eight, the model has already got Sunday three ways, by arithmetic, by listing the days, and by modular arithmetic.

The original continuation goes around again with several more methods and uses nine hundred eighty-six tokens. The regenerated one confirms the answer and wraps up in two hundred twelve.

### Results

The setup. The model being trained is DeepSeek-R1-Distill-Qwen-7B, and all its training data comes from its own reasoning traces on a training set of fourteen thousand and twenty-seven math problems.

Each problem is sampled three times, and the shortest correct solution becomes the original trace. Each detected state gets eight regenerated continuations, a pair has to save at least three hundred tokens, and each problem contributes at most three pairs. Training uses LoRA, a small set of added weights, for two epochs.

Evaluation covers five math benchmarks, from the easier MATH-500 up to the hardest, AIME 2024 and AIME 2025, plus three coding benchmarks. Every problem gets eight samples.

There are two baselines, DAST and TokenSqueeze, both evaluated from their released full weights.

The results are in Table 1. On all five math benchmarks, LoCo cuts output length by between forty-one point seven and forty-five point four percent.

And accuracy? On MATH-500 it's essentially flat, ninety-one point eight eight for the base model versus ninety-one point eight zero. On the other four it goes up.

On the hardest, AIME 2024, accuracy rises from fifty point eight three to fifty-four point one seven, while average output drops from thirteen thousand four hundred ninety-three tokens to seven thousand three hundred sixty-four. That's close to half.

The baselines don't manage both. DAST barely compresses and actually gets longer on hard problems, six point nine percent longer on AIME 2024. TokenSqueeze compresses hard but loses accuracy as difficulty rises, two point zero eight points on AIME 2024.

Code is in Table 2, and remember, training used only math. On MBPP, accuracy is unchanged and output drops thirty-six point three percent. On the harder LiveCodeBench and CodeContests, output drops by more than a fifth, and accuracy rises by zero point seven four and two point one three points.

The authors' reading: since training was math and testing was code, LoCo is changing the model's preference over reasoning length, not teaching it new domain knowledge.

This figure shows another angle. The horizontal axis is the maximum allowed output length, the vertical axis is accuracy. On AIME 2024 and CodeContests, LoCo's curve sits above TokenSqueeze at most budgets.

### Comparison

The ablations in Table 3 ask where the gains come from. The first variant, called Global, keeps the same problems and training setup but drops the local part. It just pairs the shortest and longest correct full solutions for each problem.

Accuracy is similar on some benchmarks, but compression is far weaker. Global saves only five point two, eleven point one, and fifteen point eight percent on AMC23, AIME 2024 and AIME 2025. LoCo saves forty-one point seven, forty-five point four, and forty-five point two.

The second variant keeps the same cut points but has GPT-5.4 write the short continuations. The model's own continuations actually work better. On AIME 2024, the GPT-5.4 version compresses thirty-one point eight percent, versus forty-five point four for LoCo. The authors' explanation is that self-generated text stays closer to the model's own reasoning style, so it's an easier signal to learn from.

The third swaps the detector for random or evenly spaced cut points. Length still drops, but accuracy suffers. On AIME 2024, random cuts score forty-eight point three three, two and a half points below the base model, against fifty-four point one seven for LoCo.

The appendix adds one more number. The detector's cut points sit late in the trace, with about half in the final forty percent of steps. Random and even cuts land earlier, which the authors say means compressing before the model has converged.

My read is that Global is the most convincing ablation. Moving the preference from whole solutions to specific states takes compression from around ten percent to over forty. Where you compress matters more than whether you compress.

My read is also that the comparison is narrow. There are only two baselines. Step-level compression methods like SSPO and ReCUT are discussed in the related work, but not compared head-to-head in the experiments.

### Limitations

The paper has no limitations section, but a few things are visible in the setup. First, everything is trained on one model, the 7B DeepSeek-R1-Distill-Qwen, using LoRA. Larger models and full fine-tuning aren't tested.

Second, AIME has only thirty problems a year. In Table 1, the plus-or-minus ranges on AIME accuracy run from three point seven five to six point six one, about as large as, or larger than, the roughly three-point gains. The drop in output length is far clearer.

Third, building the data takes an extra thirty-billion-parameter judge and eight regenerations per state. The paper doesn't report that extra compute.

Fourth, the training data is math only. It carries over to code, but more open-ended tasks, like writing or multi-turn agent work, aren't tested.

### Future work

The authors say the code and model will be released on GitHub.

My read is that the paper's real contribution is reframing the question, from how to make reasoning shorter to where reasoning should stop. What to watch next is whether this holds up on bigger models and longer, multi-step tasks.

My read for anyone deploying these models: the same accuracy with forty percent fewer generated tokens means lower cost and less waiting, and that's often worth more than another point or two of accuracy.

That's today's Daily Frontier Papers. The paper is linked in the description.

