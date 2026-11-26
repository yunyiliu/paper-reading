# Lazy Agents Answer Against Their Own Beliefs: Reinforcement Learning with Verifiable Beliefs Teaches Them to Act on What They Believe

**AI为什么急着瞎猜答案**

- 论文 Paper: [arXiv 2611.32884](https://arxiv.org/abs/2611.32884)
- 单位 Affiliation: 南洋理工大学 DeCLaRe Lab
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-26
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1yf17zREBy)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇新加坡南洋理工大学 DeCLaRe 实验室的论文，标题的意思是，偷懒的 Agent 会违背自己的判断去答题，而可验证信念强化学习，能教它们按自己的判断行动。

这个方法简称 RLVB，要解决的是 Agent 在长任务里，证据还不够就急着下结论。

### 问题

先看论文的例子。桌上四个倒扣的杯子，Agent 要一个个掀开，找红色圆柱。掀了 1 号和 3 号，一个是绿色圆柱，一个是空的，红色圆柱在 2 号或 4 号，各一半概率。

这时该做的，是再掀一个。可基础模型经常直接猜，哪怕它自己刚写下，还有好几个杯子都有可能。论文把这叫做偷懒，laziness。

作者让模型每次行动前，先写一行信念，比如，可能是 2 号和 4 号。结果在证据不够、信念也还列着好几个答案的回合里，基础模型有百分之 3 到 68 的时候照样作答。它不按自己写下的判断行事。

### 以前的方法

现在训练 Agent，常用可验证奖励强化学习，RLVR，就是最后检查答案，对了给奖励。

问题是，长任务里只看结果，分不清哪一步有功。运气猜对的答案，和有证据支撑的答案，奖励一样。在论文的表 1 里，只用结果奖励的 GRPO 和 PPO，平均准确率反而比训练前低了 1.0 和 5.1 个百分点。

另一条路是过程奖励，每一步打分，常见的是请大模型当裁判。但裁判如果看得到正确答案，就可能奖励碰巧猜对，惩罚正确但还不确定的判断。而且还要多训练一个价值网络，每步调用裁判，成本更高。

这张图把三种框架放在一起。GRPO 按整条轨迹给分，PPO 让看得到答案的裁判逐步打分，RLVB 用精确后验逐步打分。

### 核心想法

作者的关键观察是，训练用的模拟器，知道的远不止答案对不对。它知道隐藏的答案，也知道每个动作会揭示什么。

所以它能在每一步之后，算出精确后验，也就是根据已有证据，每个答案还有多大可能。这就是证据真正支持的信念。

RLVR 在最后验证答案，RLVB 在每一步验证信念，名字就是这么来的。

回到杯子。2 号和 4 号都还可能时，再掀一个，奖励接近 0。直接答 4 号，奖励大约负 7，不管圆柱是不是真在 4 号下面。

### 方法

具体做法。有一个后验老师，在精确后验上往前看几步，给每个动作估值。作答看答对的概率，再按时间打折扣，收集证据看这一步之后后验会变成什么样。

每个回合的奖励，主要是 Agent 的动作在老师那里的对数概率。证据还允许好几个答案时，继续收集得分高，只剩一个答案时，作答得分高。

两种错误代价差别很大。证据不够就作答，奖励很低，图 1 里是负 7.42。证据够了多走一步，只损失一步的时间折扣。所以这个奖励主要教的，是别太早作答。

有意思的是，奖励从不读 Agent 写的那行信念。但更新提高的是整个回合的概率，信念那一行也跟着动作一起被训练。论文说，这就是 RLVB 的主要想法。

训练分阶段，一步一步更新。每个阶段，固定当前模型去玩游戏，在每个历史状态上采样 N 个候选回合，逐个打分。

每个候选的权重，是它的奖励减去同一状态下其他候选的平均奖励。很像 GRPO 的组内比较，只是 GRPO 一组是几条完整轨迹，这里一组是同一状态下的几个回合，所以一个决定只和相同证据下的其他选择比。

整个过程不需要裁判，不需要价值网络，也不需要奖励模型，因为后验是精确算出来的。

### 实验和效果

实验用三个开源视觉语言模型，Qwen3-VL 的 4B 和 8B Thinking 版，还有 Qwen3.5-4B，所有方法都只训练 LoRA，最多 50 次更新。

基准有三个。HiddenGames 由扫雷、推箱子、三个桌面任务等七个现有任务组成，测试用训练中没见过的 756 局。

Pro-Worlds 是作者新做的，Agent 通过小窗口看一个大世界，最多移动 35 步，再从候选里选答案，多数游戏是 8 个候选，一共 15 个游戏。

RoboQuest 是十个模拟厨房里的移动操作任务，比如靠称重找出不一样的包裹，没有精确后验。后两个基准训练中都没见过。

总体结果，三个模型乘三个基准，九种设置，RLVB 全部超过基础模型，平均提高 17.1 个百分点。另一个用期望最大化的 EM 版本，平均提高 10.4。

举两个数。Qwen3.5-4B 在 HiddenGames 上，从 26.5 到 48.5。Qwen3-VL-4B 在没见过的 RoboQuest 上，从 28.4 到 61.2。

同样的学习算法，只把结果奖励换成后验过程奖励，九种设置全部更好，平均高 18.1 个百分点。作者认为，收益主要来自后验奖励本身。

和用大模型裁判打过程奖励的 PPO 比，RLVB 九种设置赢了八种，平均高 9.3 个百分点，例外是 Qwen3.5-4B 的 RoboQuest。成本上，论文在 Qwen3.5-4B 上比了一次，这个 PPO 用了 120.1 个 GPU 小时，RLVB 是 74.2，EM 版只要 29.7。

结果奖励为什么帮助不大，看这张图。GRPO 一组轨迹如果没有一条赢，奖励全一样，就没有学习信号。50 次更新后，百分之 18.8 到 53.1 的组是这样。RLVB 没有信号的组，大约只有百分之 2 到 14。

提升到底从哪来。作者按作答时的证据多少给每局分组，证据充分，指最可能答案的后验达到 0.95。训练后的 Agent，在三个基准上分别有百分之 51、12 和 74 的局在证据充分时作答，基础模型只有百分之 8、0.4 和 36。

而不管哪种方法，证据充分时的答案，百分之 66 到 92 是对的。在 HiddenGames 上，18.1 个百分点的提升里，多收集证据贡献了 26.7，同样证据下的准确率反而是负 8.5。也就是说，它更准是因为等到了更多证据，而不是同样的证据读得更好。

它为什么愿意等，因为学会了按自己的信念行事。证据不够、信念还列着好几个答案时，RLVB 只有百分之 0.1 到 2.2 的回合会作答，基础模型是百分之 3 到 68。

但信念本身并没有一致地变准，有的设置更准，有的更差。它每一步带来的证据，也和基础模型差不多。

### 和以前方法的对比

最接近的工作是 Agent-BRACE，也让 Agent 说出信念，但由大模型裁判对照真实答案打分。两个答案一样可能时，裁判会奖励只说出真答案的判断，等于又在奖励运气。RLVB 用精确后验，奖励可验证，不需要裁判。

消融在表 3，用 Qwen3-VL-4B。只给信念行打分、去掉动作那一项，HiddenGames 上和不训练差不多，RoboQuest 上从 34.0 掉到 19.7。收益来自动作那一项，只奖励信念，教不会 Agent 什么时候该停。

那不训练，在框架里加一道检查行不行，第一次作答先不收，让它再确认一次。论文试了，RoboQuest 上提高 6.1 到 11.3 个百分点，HiddenGames 和 Pro-Worlds 上最多变化 1.9。因为框架读不了观察，判断不了证据够不够。

我的判断是，这篇的价值在于，把什么时候该停，从结果奖励里单独拎出来，用模拟器的特权信息给出精确、不靠裁判的信号，而且作者把收益来源拆得很清楚。

我的判断是，它也点出一个普遍问题，很多 Agent 不是不知道，而是知道了不照着做。光让模型写出思考不够，要让决策真正跟着它走。

### 局限

局限方面，第一，RLVB 需要规则已知的任务，环境要知道答案的先验和每个观察的概率，才能算出精确后验。规则未知或开放的任务，后验只能估计。

第二，动作空间很大时，老师往前看的成本会变高。第三，信念行只是一组可能答案，表达不了更细的把握程度。

第四，训练后，证据充分却答错的局，从百分之 2 到 4 升到 10 到 15，主要是桌面任务上看错了观察。它选的下一步，也没比基础模型更有信息量。

在扫雷和 MiniGrid 上，一步只能把正确答案的后验提高百分之 1 左右或更少，RLVB 没有提升。HiddenGames 上它有百分之 19 的局没作答，基础模型是百分之 7。

而且离后验老师还差得很远，比如 Pro-Worlds 上，论文举的是 32.4 对 100。

### 未来方向

作者的方向是，扩展到后验要学出来的任务，改进下一步的选择，以及训练 Agent 在证据够了时正确读懂观察。论文首页给了 GitHub 代码地址。

我的判断是，最值得看的是怎么走出模拟器。真实任务里没有精确后验，如果后验要靠模型估计，这套奖励还靠不靠得住，是关键。

我的判断是，也要看到范围，这是 4B 和 8B 模型在游戏和模拟环境里的结果，论文也说，RLVB 的准确率在不同训练运行之间最多差 4 个百分点。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from the DeCLaRe Lab at Nanyang Technological University in Singapore, and its title says it plainly: lazy agents answer against their own beliefs, and reinforcement learning with verifiable beliefs teaches them to act on what they believe.

The method is called RLVB. The target is agents on long tasks that commit to an answer before they have enough evidence.

### Problem

Take the paper's example. Four upside-down cups, and the agent lifts them one at a time to find a red cylinder. Cup one hides a green cylinder, cup three is empty, so the red one is under cup two or cup four, fifty-fifty.

The right move is to lift another cup. But base models often just guess, even right after writing that several cups are still possible. The paper calls this laziness.

The authors had the models write a belief line before each action, like, possible: cups two and four. When the evidence was insufficient and that line still listed several answers, the base models committed anyway on three to sixty-eight percent of those turns. They don't act on what they themselves wrote.

### Prior work

The standard recipe is reinforcement learning with verifiable rewards, RLVR. Check the final answer, reward it if it's right.

But over a long task, a final reward can't say which steps helped. A lucky guess earns exactly what an evidence-backed answer earns. In the paper's Table 1, GRPO and PPO trained on outcome reward end up below the untrained model on average, by one point zero and five point one percentage points.

The alternative is process rewards, usually from a language model acting as judge. But a judge that sees the right answer can reward a lucky guess and penalize a belief that's correct but still uncertain. You also train an extra value network and call the judge every step, so it costs more.

This figure lines up the three frameworks. GRPO credits whole trajectories, PPO has a judge that sees the answer score each step, and RLVB scores each step with the exact posterior.

### Key idea

The key observation: the training simulator knows much more than whether the answer is right. It knows the hidden answer, and what each action will reveal.

So after every step, it can compute the exact posterior, how likely each answer still is given the evidence so far. That's the belief the evidence actually supports.

RLVR verifies the answer at the end. RLVB verifies the belief at every step. Hence the name.

Back to the cups. With two and four both still possible, lifting another cup earns close to zero. Answering cup four earns about minus seven, whether or not the cylinder is really there.

### Method

Here's the machinery. A posterior teacher looks a few steps ahead on the exact posterior and values every action. An answer is valued by its chance of being right, discounted for time. An evidence action is valued by what the posterior will look like after it.

Each turn's reward is mainly the log probability of the agent's action under that teacher. While several answers remain, gathering evidence scores well. Once only one remains, answering scores well.

The two mistakes are priced very differently. Answering too early gets a very low reward, minus seven point four two in Figure 1. One extra step after the evidence is sufficient costs only a single step's time discount. So the reward mostly teaches the agent not to answer too early.

Here's the interesting part. The reward never reads the agent's belief line. But the update raises the likelihood of the whole turn, so the belief line gets trained along with the action. The paper calls this the main idea of RLVB.

Training runs in phases, updating one step at a time. In each phase, a frozen copy of the model plays episodes, and at every history it samples N candidate turns, each scored with this reward.

Each candidate is weighted by its reward minus the mean reward of the other candidates at that state. It's like GRPO's group comparison, except the group is a set of turns at one state rather than whole trajectories, so a decision is only compared with alternatives under the same evidence.

No judge, no value network, no reward model: the posterior is computed exactly.

### Results

The models are three open vision-language models: Qwen3-VL in its 4B and 8B Thinking versions, and Qwen3.5-4B. Every method trains LoRA adapters, for up to fifty updates.

There are three benchmarks. HiddenGames combines seven existing tasks, like MineSweeper, Sokoban and three tabletop tasks, tested on seven hundred and fifty-six games unseen in training.

Pro-Worlds is new: the agent sees a big world through a small window, gets at most thirty-five moves, then picks an answer, usually from eight candidates. There are fifteen games.

RoboQuest is ten mobile manipulation tasks in simulated kitchens, like finding the odd parcel by weighing, and it has no exact posterior. Neither of these last two is seen in training.

Overall, across nine settings, three models times three benchmarks, RLVB beats the base model every time, by seventeen point one percentage points on average. A second version that uses expectation maximization, RLVB EM, gains ten point four.

Two examples. Qwen3.5-4B on HiddenGames goes from twenty-six point five to forty-eight point five. Qwen3-VL-4B on unseen RoboQuest goes from twenty-eight point four to sixty-one point two.

Keep the same learner and only swap outcome reward for the posterior reward, and all nine settings improve, by eighteen point one points on average. The authors conclude the gain comes mainly from the posterior reward itself.

Against PPO with a judge-based process reward, RLVB wins eight of nine settings, by nine point three points on average. The exception is Qwen3.5-4B on RoboQuest. On cost, measured on Qwen3.5-4B, that PPO run used one hundred and twenty point one GPU-hours, RLVB seventy-four point two, and RLVB EM just twenty-nine point seven.

This figure shows why outcome reward helps so little. If no GRPO trajectory in a group wins, they all get the same reward, and there's no learning signal. After fifty updates, eighteen point eight to fifty-three point one percent of groups are like that. For RLVB it's only about two to fourteen percent.

So where does the gain come from? The authors group episodes by the evidence the agent had when it answered, where sufficient means the top answer's posterior reached zero point nine five. The trained agent answers with sufficient evidence in fifty-one, twelve and seventy-four percent of episodes on the three benchmarks, against eight, zero point four and thirty-six for the base model.

For every method, answers given with sufficient evidence are right sixty-six to ninety-two percent of the time. On HiddenGames, the eighteen point one point gain splits into plus twenty-six point seven from more evidence and minus eight point five at the same evidence. So it's more accurate because it waits for more evidence, not because it reads the same evidence better.

Why does it wait? Because it learned to act on its own belief. When evidence is insufficient and the belief line still lists several answers, RLVB commits on only zero point one to two point two percent of turns, against three to sixty-eight for the base model.

But the belief itself isn't consistently more accurate: better in some settings, worse in others. And each step adds about as much evidence as a base-model step.

### Comparison

The closest prior work is Agent-BRACE, which also has the agent state its belief, but an LLM judge grades it against the true answer. When two answers are equally likely, that judge rewards naming only the true one, which is rewarding luck again. RLVB scores against the exact posterior, so the reward is verifiable and needs no judge.

The ablations are in Table 3, on Qwen3-VL-4B. Score only the belief line and drop the action term, and HiddenGames stays where the untrained model is, while RoboQuest drops from thirty-four point zero to nineteen point seven. The gain comes from the action term; rewarding beliefs alone doesn't teach when to stop.

What about skipping training and adding a check in the harness, refusing the first answer and asking the agent to confirm? The paper tried it. It helps by six point one to eleven point three points on RoboQuest, but on HiddenGames and Pro-Worlds it moves accuracy by at most one point nine. The harness can't read the observations, so it can't tell when the evidence is enough.

My read is that the contribution is separating the question of when to stop from the outcome reward, and using the simulator's privileged information to give an exact signal with no judge. And the authors take their own gain apart carefully.

My read is that it also names a broader problem: many agents don't fail because they don't know, but because they know and don't act on it. Writing out the reasoning isn't enough; the decision has to follow it.

### Limitations

On limitations, first, RLVB needs tasks with known rules: the environment must know the prior and the likelihood of each observation to compute the exact posterior. With unknown or open-ended rules, the posterior has to be estimated.

Second, with very large action spaces, the teacher's lookahead gets expensive. Third, a belief line is just a set of answers, so it can't express finer degrees of certainty.

Fourth, after training, wrong answers given with sufficient evidence rise from two to four percent of episodes to ten to fifteen percent, mostly tabletop tasks where the agent misreads what it saw. And its chosen steps are no more informative than the base model's.

In MineSweeper and MiniGrid, a step moves the correct answer's posterior by about one percent or less, and RLVB shows no gain. On HiddenGames it ends nineteen percent of episodes with no answer, against seven percent for the base model.

And the agents stay far below the posterior teacher. On Pro-Worlds, the paper's example is thirty-two point four against one hundred.

### Future work

The authors want to extend RLVB to tasks where the posterior must be learned, improve the choice of the next step, and train agents to read their observations correctly once the evidence is sufficient. The paper's first page gives a GitHub link to the code.

My read is that the thing to watch is leaving the simulator. Real tasks don't hand you an exact posterior, and if a model has to estimate it, whether this reward stays reliable is the key question.

My read is also that scope matters. These are four- and eight-billion-parameter models in games and simulations, and the paper notes RLVB's accuracy varies by up to four points across training runs.

That's today's Daily Frontier Papers. The paper is linked in the description.

