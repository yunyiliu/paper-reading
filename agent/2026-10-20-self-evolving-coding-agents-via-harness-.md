# Self-Evolving Coding Agents via Harness-Policy Optimization

**模型自己改框架再训练自己**

- 论文 Paper: [arXiv 2610.22791](https://arxiv.org/abs/2610.22791)
- 单位 Affiliation: 微软研究院 · 弗吉尼亚大学 · 卡内基梅隆大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-20
- 视频 Videos: [YouTube 中文](https://youtu.be/TiGLFe_9tIY) · [YouTube English](https://youtu.be/BnT7L9GtEUM) · B站（审核中）

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇微软研究院的论文，合作方还有弗吉尼亚大学和卡内基梅隆大学，名字叫 Self-HPO，意思是让编程 Agent 通过 harness 和策略的交替优化，自己进化。

简单说，它让同一个模型轮流做两件事，先给自己改工作框架，再在新框架下用强化学习训练自己，如此循环。

### 问题

先解释一个词，harness。一个编程 Agent 由两部分组成，一部分是模型本身，论文叫 policy，也就是策略模型，另一部分是包在模型外面的那层框架，叫 harness。

harness 决定模型能看到什么，它的动作怎么执行，收到什么反馈，什么时候结束。所以同一个模型，换一个 harness，同一道题可能做得出，也可能做不出。

论文要解决的问题是，这两部分怎么一起改进，而且不依赖外部更强的模型，让 Agent 自己来。

### 以前的方法

以前主要有两条路。一条是 harness 优化，模型不动，去搜索更好的提示词、工作流或者 Agent 代码。另一条是策略优化，harness 不动，用成功的轨迹或者可验证的奖励去训练模型。

作者认为这两条路各有天花板。只改 harness，模型本身的能力上不去。只训练模型，它能学到的，只限于这个固定 harness 能引出来的轨迹。

最近也有工作开始两边一起优化，但它们依赖一个外部模型来提 harness 的修改方案，而且模型一边训练一边在变，它的失败方式也在变，优化目标是移动的。

### 核心想法

Self-HPO 的出发点是，harness 和策略互相成就。为旧版本模型调好的 harness，未必对准新模型的弱点，而模型能学到什么，又受 harness 的限制。

所以干脆交替来做，而且让正在进化的模型自己当 harness 的设计者。这张图左边就是整个循环，H-step 改 harness，P-step 训练策略，一轮一轮往下走。

### 方法

先看 H-step，也就是 harness 优化。这一步模型参数冻结，它同时扮演两个角色，一个是做题的，一个是 harness 的提议者。

具体是，当前的模型加当前的 harness，先在一个搜索集上把题跑一遍，然后由模型自己去看失败的轨迹，提出针对性的修改，可以改提示词，也可以改运行时的代码逻辑。

每个 H-step 会提出 6 个候选 harness，每个候选都在同一个搜索集上评测，留下得分最高的那个。

再看 P-step，也就是策略优化。把选中的 harness 固定住，在训练集上用强化学习更新模型，奖励来自一个可执行的验证器，补丁通过就是 1，不通过就是 0。优化用的是 GRPO，再加上 Orchard 框架里的一种平衡采样方法，减少全对或者全错、没有学习信号的样本组。

这里有一个关键设计，叫进化的搜索集。第一轮的搜索集，是从训练集里随机抽的 300 道题。之后每一轮，搜索集换成上一轮强化学习里一次都没做出来的难题。

这样，下一轮 harness 优化盯的，始终是当前这个模型最新的弱点，而不是反复对着最初那批题。训练好的新模型，也会成为下一轮的 harness 提议者。

整个循环在难题集合稳定下来的时候停止，实际实验里跑了两轮。

### 实验和效果

实验设置。基础模型是 Qwen3.5-35B-A3B，总参数 350 亿，激活参数 30 亿，起点 harness 是 mini-SWE-agent。训练集有 2484 道题，每个 P-step 做 50 次强化学习更新。

评测用三个基准。主基准是 SWE-Bench Verified，500 个经过人工核验的 GitHub issue。另外两个是分布外测试，SWE-Bench Multilingual 有 300 题，SWE-Bench Pro 的公开集有 642 题。指标是 resolve rate，就是生成的补丁能通过验证的比例，结果取三次评测的平均。

主要结果在表 1。在不借助外部模型的设置下，Self-HPO 在 SWE-Bench Verified 上，从基础模型的 61.0 提到 69.5。

对比两个单边基线，只训练模型的 RLVR 是 67.5，只改 harness 的 Self-Harness 是 65.2，Self-HPO 分别高出 2.0 和 4.3 个点。

三个基准的平均分，从 57.8 提到 64.8，比更强的单边基线 RLVR 高 2.9 个点。

泛化也有看点。harness 搜索和训练只用了 Python 项目的题，但在覆盖九种非 Python 语言的 Multilingual 上，Self-HPO 比基础模型高 5.5 个点，而 Self-Harness 只高了 0.1 个点。

如果换成 Opus 4.8 这样更强的外部模型来提 harness 修改，同样的交替框架叫 HPO，SWE-Bench Verified 能到 71.6，三项平均 66.3。

还有个对比。用 Opus 4.8 的成功轨迹直接做监督微调，也就是 SFT，Verified 到 65.0，但在两个分布外基准上反而比基础模型还低。让 Opus 4.8 去改 harness 的 Meta-Harness，平均分比 SFT 高 8.9 个点。

再看表 3，这是交叉评测。只把最终的 harness 给基础模型用，从 61.0 到 64.5。只用最终的模型配原始 harness，到 64.4。两者合在一起，到 69.5，一共提高 8.5 个点，比两项单独提升之和 6.9 还多 1.6 个点。

作者据此认为，两部分的改进是互补的。

还有 pass@k 的分析，每道题跑 8 次。8 次里至少成功一次的比例，从 80.0 提到 84.8。按难度分，中等题和简单题的单次成功率分别提高 17.2 和 12.9 个点，基础模型 8 次都做不出的难题，提高 6.1 个点。

那 harness 到底改了什么。这张图左边是原始 harness，右边是用 Opus 4.8 提议的 HPO，在第二轮得到的 harness。调用模型、执行命令的主循环没动，只加了四个检查。

第一，步数或时间快用完的时候，提醒模型赶紧交一个尽力而为的补丁，而不是空手结束。第二，提交了空补丁，就拦一次，提醒新建的文件要用 git add -N 加进来。第三，第一次正式提交时先拦一下，让模型对照仓库里已有的测试，核对函数名和签名。第四，补丁打不上的时候，改用沙箱里的 git diff 作为提交。

作者说，这些改动针对的是仓库里的代码已经改了、却没能变成有效提交的几种情况，也说明 harness 优化改的不只是提示词，还有执行和结束的逻辑。

### 和以前方法的对比

消融实验在表 2，以 HPO 的 71.6 为参照。P-step 把强化学习换成 SFT，掉到 67.2。换成用模型自己成功轨迹做的拒绝采样微调，到 70.5。搜索集固定、不进化，到 70.1。把外部提议者换成模型自己，也就是 Self-HPO，是 69.5。

和同类工作比，作者总结了两点区别。一是训练后的模型直接当下一轮的 harness 提议者，比如 WHALE 用的是冻结的外部提议者，Evo2 是单独训练一个提议者。二是搜索集来自强化学习里没解出的难题，而它们用的是固定任务集。

我的判断是，这篇最有价值的不是某个分数，而是它把 harness 当成和模型权重同等的优化对象，并且用交叉评测说清楚了，两边的收益能叠加。

我的判断是，也要看到差距。附录表 4 列了排行榜，同样用 mini-SWE-agent，Gemini 3 Flash 和 MiniMax-M2.5 是 75.8，Claude Opus 4.6 是 75.6，Self-HPO 的 69.5 还差 6 个多点。而且 Self-HPO 用的是优化过的 harness，这个比较本身并不完全对等。

### 局限

论文没有单独写局限，下面几点，来自它自己的数据。

第一，提升不是均匀的。基础模型本来每次都能做对的那组题，Self-HPO 的单次成功率反而降了 2.2 个点。

第二，实验只用了一个基础模型，只在 SWE-Bench 这一类修 issue 的任务上测，训练题也全部来自 Python 项目。换别的模型，或者别的 Agent 任务，论文没有验证。

第三，模型自己当提议者，比用 Opus 4.8 低 2.1 个点，作者的结论是外部模型有帮助，但不是必需的。附录的案例里，进化出来的 harness 也出现了误报，它提醒的需求关键词，其实来自 issue 里不相关的部分。

### 未来方向

论文没有专门写未来方向，首页给出了 GitHub 和 Hugging Face 的地址。

我的判断是，接下来值得看两件事。一是这个循环多跑几轮还能不能继续涨，论文只跑了两轮。二是能不能推广到写代码以外的 Agent，比如网页操作和电脑操作，这些任务同样离不开 harness。

我的判断是，对做 Agent 的团队，这篇的启发很直接，模型训练和外面那层工程框架不该分开调，失败案例应该同时喂给两边。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Microsoft Research, with the University of Virginia and Carnegie Mellon. It's called Self-HPO, and it's about coding agents that improve themselves by alternately optimizing their harness and their policy.

In plain terms, one model takes turns at two jobs. First it rewrites the scaffold it works inside. Then it trains itself with reinforcement learning inside that new scaffold. Then it rewrites the scaffold again, and trains again.

### Problem

First, a word you'll hear a lot: harness. A coding agent has two parts. One is the model itself, which the paper calls the policy. The other is the scaffold wrapped around it, the harness.

The harness decides what the model sees, how its actions get executed, what feedback comes back, and when the episode ends. So the same model, in a different harness, can solve a problem or fail it.

The question here is how to improve both parts together, and to do it without leaning on a stronger outside model. The agent should do it itself.

### Prior work

There have been two main approaches. Harness optimization keeps the model fixed and searches for better prompts, workflows, or agent code. Policy optimization keeps the harness fixed and trains the model on successful trajectories or verifiable rewards.

The authors argue each one hits a ceiling. Tune only the harness, and the model underneath never gets better. Train only the model, and it can only learn from whatever trajectories that fixed harness happens to draw out.

Some recent work does optimize both. But it relies on an outside model to propose harness changes, and it faces a moving target. As the policy trains, its failure modes shift too.

### Key idea

Self-HPO starts from the view that harness and policy reinforce each other. A harness tuned for an older checkpoint may miss the weaknesses of the new one. And what the model can learn is bounded by its harness.

So alternate between them, and let the evolving model design its own harness. The left side of this figure is the whole loop. The H-step updates the harness, the P-step trains the policy, round after round.

### Method

Start with the H-step, harness optimization. The model's weights are frozen, and it plays two roles at once: it solves the tasks, and it proposes the harness.

Concretely, the current model and current harness run a search set of tasks. Then the model reads its own failed trajectories and proposes targeted fixes. Those can be prompt changes, or changes to the runtime code itself.

Each H-step proposes six candidate harnesses. Every candidate is scored on the same search set, and the best one is kept.

Then the P-step, policy optimization. The chosen harness is frozen, and the model is trained with reinforcement learning on the training set. The reward comes from an executable verifier: one if the patch passes, zero if it doesn't. The optimizer is GRPO, plus a balanced sampling method from the Orchard framework that cuts down on groups where every attempt succeeds or every attempt fails, which carry no learning signal.

One key design choice is the evolving search set. In the first round, the search set is three hundred tasks sampled at random from the training data. After that, each round's search set is the tasks the model never solved, not even once, during the previous round of reinforcement learning.

That keeps the next harness search pointed at the current model's newest weaknesses, instead of grinding on the original tasks. And the newly trained model becomes the next round's harness proposer.

The loop stops when that set of hard tasks stabilizes. In the experiments, it ran for two rounds.

### Results

The setup. The base model is Qwen3.5-35B-A3B, thirty-five billion parameters in total and three billion active. The starting harness is mini-SWE-agent. There are two thousand four hundred eighty-four training tasks, and fifty reinforcement learning updates per P-step.

There are three benchmarks. The main one is SWE-Bench Verified, five hundred human-validated GitHub issues. The other two are out of distribution: SWE-Bench Multilingual with three hundred tasks, and the public set of SWE-Bench Pro with six hundred forty-two. The metric is resolve rate, the share of generated patches that pass the verifier, averaged over three evaluation runs.

The main results are in Table 1. With no outside model, Self-HPO takes SWE-Bench Verified from the base model's sixty-one point zero to sixty-nine point five.

Against the one-sided baselines, RLVR, which only trains the model, scores sixty-seven point five. Self-Harness, which only changes the harness, scores sixty-five point two. Self-HPO beats them by two point zero and four point three points.

Averaged across the three benchmarks, it goes from fifty-seven point eight to sixty-four point eight, two point nine points above the stronger baseline, RLVR.

The generalization result is worth a look. Harness search and training only used Python projects. Yet on Multilingual, which covers nine other languages, Self-HPO gains five point five points over the base model, while Self-Harness gains just zero point one.

Swap in a stronger outside model, Opus 4.8, to propose the harness changes, and the same alternating loop, called HPO, reaches seventy-one point six on Verified and sixty-six point three on average.

That setting has another comparison. Fine-tuning directly on Opus 4.8's successful trajectories, supervised fine-tuning or SFT, gets sixty-five point zero on Verified, but it lands below the base model on both out-of-distribution benchmarks. Meta-Harness, which has Opus 4.8 edit the harness instead, beats SFT by eight point nine points on average.

Now Table 3, a cross-evaluation. Give only the final harness to the base model, and it goes from sixty-one point zero to sixty-four point five. Use only the final model with the original harness, and it reaches sixty-four point four. Put them together and you get sixty-nine point five, a gain of eight point five points. That's one point six more than the six point nine you'd get by adding the two separate gains.

The authors' reading is that the two improvements are complementary. Each holds up on its own, and together they give the largest gain.

There's also a pass-at-k analysis, with eight attempts per task. The share of tasks solved in at least one of eight attempts rises from eighty point zero to eighty-four point eight. Broken down by difficulty, single-attempt success rises by seventeen point two points on medium tasks and twelve point nine on easy ones, and by six point one points on hard tasks the base model never solved in eight tries.

So what does the harness actually change? On the left of this figure is the original harness. On the right is the one HPO, the Opus 4.8 version, produced in its second round. The core loop, query the model and run its command, is untouched. Four checks are added.

One: when the step or time budget is nearly used up, it tells the model to submit a best-effort patch instead of ending empty-handed. Two: if the submitted patch is empty, it holds it once and reminds the model to stage new files with git add -N. Three: it holds the first real submission and asks the model to check its function names and signatures against the repo's existing tests. Four: if the patch doesn't apply, it submits the sandbox's own git diff instead.

The authors say these target the different ways that real edits in the repository fail to become a valid submission. They also show that harness optimization can change execution and termination logic, not just prompts.

### Comparison

The ablations are in Table 2, using HPO's seventy-one point six as the reference. Replace reinforcement learning in the P-step with SFT, and it drops to sixty-seven point two. Use rejection-sampling fine-tuning on the model's own successful runs, and it gets seventy point five. Freeze the search set instead of evolving it, seventy point one. Replace the outside proposer with the model itself, which is Self-HPO, sixty-nine point five.

Against related work, the authors point to two differences. First, the trained model itself proposes the next harness. WHALE, for example, uses a frozen outside proposer, and Evo2 trains a separate proposer. Second, the search set comes from tasks left unsolved during reinforcement learning, while those methods search on a fixed task set.

My read is that the real contribution isn't any single score. It's treating the harness as an optimization target on par with the weights, and using a clean cross-evaluation to show the two sets of gains stack.

My read is also that the gap is real. The leaderboard in appendix Table 4, with the same mini-SWE-agent harness, has Gemini 3 Flash and MiniMax-M2.5 at seventy-five point eight and Claude Opus 4.6 at seventy-five point six. Self-HPO's sixty-nine point five is more than six points behind. And Self-HPO runs on an optimized harness, so that comparison isn't fully like for like.

### Limitations

The paper has no dedicated limitations section. These points come from its own data.

First, the gains aren't uniform. On the tasks the base model already solved every time, Self-HPO's single-attempt success actually drops by two point two points.

Second, the experiments use a single base model, and only SWE-Bench style issue-fixing tasks, with all training tasks drawn from Python projects. Whether it holds for other models or other kinds of agent tasks isn't tested.

Third, using the model as its own proposer scores two point one points below using Opus 4.8. The authors conclude an outside model helps but isn't required. And in the appendix case study, the evolved harness raises a false alarm: its reminder about the issue's key terms pulls them from unrelated parts of the issue text.

### Future work

The paper doesn't have a future work section. Its first page gives GitHub and Hugging Face addresses.

My read is that two things are worth watching. Does the loop keep improving past two rounds, since the paper stops at two? And does it carry over to agents beyond coding, like web and computer use, where the harness matters just as much?

My read for teams building agents is simple. Don't tune model training and the engineering scaffold around it separately. Feed the failures to both.

That's today's Daily Frontier Papers. The paper is linked in the description.

