# Actor, Judge, or Teacher? Understanding Decision Models in Long-Horizon Agents

**自己做不好也能当好老师**

- 论文 Paper: [arXiv 2611.17167](https://arxiv.org/abs/2611.17167)
- 单位 Affiliation: 浙江大学 · 百度
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-15
- 视频 Videos: [YouTube 中文](https://youtu.be/BsDZaM7KvaA) · [YouTube English](https://youtu.be/xp6uygI9JNQ) · [B站](https://www.bilibili.com/video/BV18QsJ6iE7L)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇浙江大学和百度的论文，题目是，Actor, Judge, or Teacher，行动者、裁判，还是老师，研究的是决策模型在长流程 Agent 里到底该怎么用。

先把结论放在前面，一个自己做任务做不好的模型，照样可以当一个好老师，前提是你问对了问题，也给它看对了东西。

### 问题

先解释什么是决策模型。它不生成一段文字，而是对一个问题给出固定格式的答案，比如在几个选项里选一个，并给每个选项一个概率，特点是快，而且便宜。

这样一来，同一个决策模型就有三种用法。当行动者，替 Agent 选下一步动作，当裁判，给已经做过的动作打分，当老师，在强化学习训练的时候，给每一步打分，去教一个小的策略模型。

为什么需要给每一步打分。常见的强化学习只看最后成没成功，一条轨迹里每个动作拿到的功劳都一样，有用的动作和白费的动作，分不出来。

### 以前的方法

想拿到每一步的信号，以前要么给每个环境标注步骤级的标签，要么在每个动作上都调用一次大模型当裁判，两条路都很贵。

而决策模型过去主要被当成行动者来评估，在这个角色上，它们经常失败，会被提到任务物品的指令吸引，跳过必须先做的子目标，或者把子目标的顺序做反。

所以作者问的是，一个当行动者很差的模型，为什么还能当好老师，又是什么样的提问方式，让这件事成立。

### 核心想法

核心是一个观察。行动的时候，模型必须提前预判一个动作的后果，而当裁判的时候，它可以等这一局结束，看到这个动作后来带来了什么。

于是作者把裁判拆成两个可以调的选择。第一是看什么，只看到这个动作为止的前半段，叫 Prefix，还是看完整条轨迹，叫 Hindsight，也就是事后回看。第二是问什么，问这一步之后任务会不会成功，叫 Value，还是问这一步有没有帮助完成目标，叫 Progress。

### 方法

实验用了六个模型。三个是专门的决策模型，JEV、Decider v1.2 和 GPT Decisions，另外三个是通用大模型，用提示词，或者直接读第一个 token 的概率，当决策模型来用，用的正好是 Decider 和 GPT Decisions 的两个底座模型。

环境是两个纯文字的任务环境。ALFWorld 是家务场景，比如把加热过的杯子放到桌上，WebShop 是网购，要搜索商品、选尺码颜色、再下单。两个环境里，都有一些要求必须按顺序完成。

当老师的做法是这样。训练小策略模型用的是 GRPO，就是同一个任务采样一组轨迹，按结果好坏互相比较。每一局结束后，决策模型给轨迹里的每个动作，打一个 0 到 1 之间的分。

这些分数在一组轨迹里减去平均值，乘上 0.5，再加到原来按结果算的优势上。比平均分高的动作，训练信号就大一点，低的就小一点。老师模型在训练中固定不动，测试的时候也不用它。

### 实验和效果

先看行动者。表 2 里，六个模型在有先后依赖的任务上都明显变差。JEV 在两个物品可以任意顺序放的 Pick2 任务上，成功率是 70.8%，而在必须先清洗、加热或冷却的任务上，最高也只有 18.8%。

问题不在于它不知道该做什么。作者收集了 116 个状态，模型已经拿着还没处理的物品站在目的地，比如拿着没加热的杯子站在桌边。目标完整时，JEV 把 63.8% 的概率给了直接放下，只有 14.6% 给了先去清洗、加热或冷却，把放下这个子目标从目标里拿掉，先做这一步的概率就升到了 77.6%。

提醒也没什么用。在提示里加一句，请考虑任务要求之间的依赖，六个模型的成功率只变化了负 2.8 到正 5.2 个百分点。再加一个加热杯子的例子，有的模型提升明显，但 Decider v1.2 反而从 89.7% 降到了 85.3%。

再看裁判。图 5 是 JEV 在 462 条轨迹上的打分。问 Value 的时候，完成子目标的里程碑动作、普通动作，和什么都没改变的空操作，分数几乎一样，分别是 0.632、0.604 和 0.577。换成问 Progress，分数拉开成 0.659、0.440 和 0.298。

在同一条轨迹里面，Progress 让里程碑动作比空操作平均高 0.36，Value 只高 0.05。道理不难理解，问任务会不会成功，同一条轨迹里每一步的答案，其实都是那个最终结果。

看到完整轨迹也很关键。图 7 是一个例子，任务是把加热过的杯子放进柜子。只看前半段时，JEV 给过早把杯子放进柜子这一步打了 0.79，给打开微波炉只打了 0.20，看完整条轨迹之后，打开微波炉升到了 0.76。

最后是老师。表 3 里，用完整轨迹加 Progress 的打分，去教一个 Qwen2.5 1.5B 的策略模型，ALFWorld 成功率从只用结果奖励的 72.8%，提到了 82.9%，WebShop 从 56.8% 提到了 71.4%。

四种组合里，这一种在两个环境上都是最好的。只看前半段时，两种问法差不多，而看完整轨迹却问 Value，ALFWorld 反而只有 69.3%，比只用结果奖励还低。

收益最大的是有先后依赖的任务，达到 86.8%，比其他三种设置高 6.9 到 16.2 个百分点。而在这类任务上，JEV 自己当行动者，只完成了 14.7%。

换成 7B 的策略模型也有提升，ALFWorld 从 77.6% 到 83.8%，WebShop 从 66.1% 到 73.6%。而且用老师教出来的 1.5B，在两个环境上，都超过了只用结果奖励训练的 7B。

成本方面，老师是通过 API 调用的 JEV，表 5 里四次训练的打分费用，每次大约 22 到 30 美元。

### 和以前方法的对比

和直接让决策模型当行动者比，差距很大。同样的 ALFWorld 评测任务，JEV 自己做只完成 42.9%，它教出来的 1.5B 小模型完成了 82.9%。

和用通用大模型当裁判比，作者在附录里报告，两个公开了底座的决策模型，都比用提示词提问的底座模型漏判更少，每次打分还便宜大约 28% 到 43%。

需要说明的是，作为基线的 GRPO 数字，是直接引用 GiGPO 那篇论文报告的结果，不是作者自己重新跑的。

我的判断是，这篇最有用的结论，不是又一个打分技巧，而是挑老师的标准变了，一个模型适不适合当监督者，要看它事后评判轨迹的能力，而不只是看它自己会不会做。

我的判断是，这里也缺一个关键对照，表里没有和 GiGPO 这类同样做步骤级功劳分配的方法正面比较，所以还不能说它比现有的细粒度方法更好。

### 局限

作者自己写了三条局限。第一，实验只用了两个文字环境，每局最多 30 个动作，软件工程、工具调用这类更长的任务里，结论是否还成立，还不知道。

第二，裁判都是在一局结束之后才打分。如果要在线使用，比如实时过滤动作，就只能看到前半段，事后回看的优势就用不上了。

第三，JEV 和 GPT Decisions 都是闭源模型，训练方法没有公开，所以没法知道，是哪些训练选择让一个决策模型成为好裁判。

另外，当老师的实验只用了 JEV 一个模型，其余五个模型，只做了行动者和裁判的分析。

### 未来方向

作者提出的方向是，研究能随着后续步骤到来，延迟或者修正打分的方法，把事后回看的优势带到在线场景，也可以用 Decider v1.2 这样的开放权重模型，直接训练它去评判单个动作的贡献。

我的判断是，这个思路对做 Agent 训练的团队很实际，不用逐步标注，也不用额外训练一个 critic，训练时调一个便宜的决策模型，就能拿到步骤级的信号，接下来值得看的，是它在更长、更开放的任务上能不能站得住。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Zhejiang University and Baidu. It's called Actor, Judge, or Teacher, and it asks how decision models should actually be used inside long-horizon agents.

Here's the punchline up front. A model that's bad at doing a task can still be a good teacher for it, as long as you ask it the right question and show it the right evidence.

### Problem

First, what's a decision model? Instead of writing out text, it answers a question with a typed output, like picking one of several options and giving each option a probability. It's fast, and it's cheap.

That means one decision model can play three roles. As an actor, it picks the agent's next move. As a judge, it scores moves that were already made. And as a teacher, it scores every step during reinforcement learning, to train a small policy model.

Why score every step? Standard reinforcement learning only looks at whether the episode succeeded, so every action in a trajectory gets the same credit. Useful moves and wasted moves look identical.

### Prior work

Getting step-level signal used to be expensive. You either labeled steps for each environment, or you called a language model judge on every single action.

Meanwhile, decision models have mostly been evaluated as actors, and in that role they often fail. They get pulled toward commands that mention the task's objects, they skip prerequisite subgoals, and they do subgoals in the wrong order.

So the authors ask why a model that acts poorly might still teach well, and which ways of querying it make that possible.

### Key idea

The core observation is simple. When acting, a model has to anticipate an action's consequences in advance. When judging, it can wait until the episode is over and see what that action actually led to.

So the authors split the judge into two knobs. The first is what it sees: just the trajectory up to the action, called Prefix, or the whole trajectory, called Hindsight. The second is what it's asked: will the task succeed after this action, called Value, or did this action help achieve the goal, called Progress.

### Method

Six models are tested. Three are dedicated decision models: JEV, Decider v1.2, and GPT Decisions. The other three are general language models turned into decision models, either by prompting or by reading off first-token probabilities, and they're built on the two base models behind Decider and GPT Decisions.

The environments are two text-based benchmarks. ALFWorld is household tasks, like putting a heated mug on a table. WebShop is online shopping: search, pick size and color, then buy. In both, some requirements have to be met in a fixed order.

Here's how teaching works. The small policy is trained with GRPO, which samples a group of attempts at the same task and compares them by outcome. After each episode, the decision model scores every action in the trajectory between zero and one.

Those scores are centered within the group, multiplied by zero point five, and added to the usual outcome-based advantage. Actions scored above average get a stronger push, actions below average a weaker one. The teacher stays frozen during training and isn't used at test time.

### Results

Start with acting. In Table 2, all six models get much worse on tasks with an ordering dependency. JEV succeeds seventy point eight percent of the time on Pick2, where two objects can be placed in either order, but at most eighteen point eight percent on tasks that require cleaning, heating or cooling first.

And it's not that the model doesn't know the right step. The authors collected one hundred sixteen states where the agent is already standing at the destination holding an object that still needs work, like an unheated mug at the table. With the full goal, JEV puts sixty-three point eight percent of its probability on just placing it, and only fourteen point six percent on the cleaning, heating or cooling it still needs. Remove the placement subgoal from the goal, and that required step jumps to seventy-seven point six percent.

Reminders don't fix it. Adding a hint to consider dependencies between requirements changes success by between minus two point eight and plus five point two percentage points across the six models. Adding a heated-mug example helps some models a lot, but Decider v1.2 actually drops from eighty-nine point seven to eighty-five point three percent.

Now judging. Figure 5 shows JEV scoring four hundred sixty-two trajectories. Under the Value question, milestone actions that complete a subgoal, other actions, and no-op actions that change nothing all get nearly the same score: zero point six three two, zero point six zero four, and zero point five seven seven. Under Progress, they spread out to zero point six five nine, zero point four four zero, and zero point two nine eight.

Within the same trajectory, Progress scores milestones zero point three six above no-ops. Value scores them just zero point zero five higher. That makes sense: if you ask whether the task will succeed, every step in a trajectory has the same answer, the final outcome.

Seeing the whole trajectory matters too. Figure 7 walks through a task where a heated mug has to go in a cabinet. With only the prefix, JEV gives the premature placement in the cabinet zero point seven nine, and opening the microwave just zero point two zero. With the full trajectory, opening the microwave rises to zero point seven six.

Finally, teaching. In Table 3, Hindsight plus Progress scores train a Qwen2.5 1.5B policy from seventy-two point eight to eighty-two point nine percent success on ALFWorld, compared with outcome rewards alone, and from fifty-six point eight to seventy-one point four on WebShop.

Of the four combinations, this one is best on both environments. With only the prefix, the two questions perform about the same. But Hindsight with the Value question gets just sixty-nine point three on ALFWorld, below outcome rewards alone.

The biggest gains are on tasks with ordering dependencies, where the policy reaches eighty-six point eight percent, six point nine to sixteen point two points above the other settings. On those same tasks, JEV acting on its own completes only fourteen point seven percent.

It holds for a 7B policy too: seventy-seven point six to eighty-three point eight on ALFWorld, and sixty-six point one to seventy-three point six on WebShop. And the 1.5B policy trained with the teacher beats the 7B policy trained on outcome rewards alone, in both environments.

On cost, the teacher is JEV called through an API, and Table 5 puts the scoring bill for each of the four training runs at roughly twenty-two to thirty dollars.

### Comparison

Compared with just using the decision model as the actor, the gap is large. On the same ALFWorld evaluation tasks, JEV completes forty-two point nine percent by itself, while the 1.5B model it taught completes eighty-two point nine.

Compared with general models as judges, the appendix reports that the two decision models with a disclosed base model miss fewer key actions than their prompted base models, at roughly twenty-eight to forty-three percent lower cost per score.

One caveat on the baseline: the GRPO numbers are taken from the results reported in the GiGPO paper, not rerun by the authors.

My read is that the most useful takeaway isn't a new scoring trick. It's a new way to pick a teacher: judge a model's fitness as a supervisor by how well it evaluates recorded trajectories, not only by how well it performs the task itself.

My read is also that a key comparison is missing. The table doesn't go head to head with GiGPO or other methods that already assign step-level credit, so we can't yet say this beats existing fine-grained approaches.

### Limitations

The authors list three limitations. First, everything runs in two text environments with at most thirty actions per episode. Whether it holds for longer tasks like software engineering or tool use is still open.

Second, the judges only score after an episode ends. Used online, say to filter actions in real time, they'd only see the prefix, and the hindsight advantage goes away.

Third, JEV and GPT Decisions are closed models with undisclosed training, so there's no way to tell which training choices make a decision model a good judge.

On top of that, only JEV was tested as a teacher. The other five models appear only in the actor and judge analyses.

### Future work

The authors point to methods that delay or revise scores as later steps arrive, to bring hindsight into online settings, and to open-weight decision models like Decider v1.2, which could be trained directly to judge each action's contribution.

My read is that this is practical for teams training agents. No step labels, no extra learned critic, just a cheap decision model queried during training to get step-level signal. The thing to watch is whether it holds up on longer, more open-ended tasks.

That's today's Daily Frontier Papers. The paper is linked in the description.

