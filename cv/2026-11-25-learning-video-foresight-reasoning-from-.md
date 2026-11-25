# Learning Video Foresight Reasoning from Unlabeled Videos via On-Policy Self-Distillation

**用错误的未来教模型预见**

- 论文 Paper: [arXiv 2611.26954](https://arxiv.org/abs/2611.26954)
- 单位 Affiliation: 布法罗大学 · 石溪大学 · 约翰斯·霍普金斯大学 · 杜克大学 · 马里兰大学 等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-25
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1qNy6zTEY4)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇第一作者来自布法罗大学，合作者来自石溪大学、约翰斯·霍普金斯大学、杜克大学和马里兰大学等的论文，方法叫 Future-OPSD。

一句话说，它让视觉语言模型看着视频的前半段，学会猜接下来会发生什么，而且训练时不需要人工标注，监督信号来自视频自己的后半段。

### 问题

先说什么是视频预见推理。给模型看一段正在发生的视频，问它，接下来会发生什么，或者我下一步该做什么。论文说，这是自动驾驶、具身机器人、主动帮人的第一人称助手都需要的能力。

论文第一张图举了个厨房的例子。问题是，根据我现在看到的，我下一步要干什么。未来那段视频里有水龙头、碗架、水槽、海绵，但真正回答问题的，只有一件事，冲洗刀。

作者把水龙头这类信息叫上下文信息，把冲洗刀叫内容信息。这张示意图里画的比例，是八成上下文，两成内容。

### 以前的方法

以前训练预见能力的做法，大致分三类。一类是直接学未来片段的文字描述和推理过程，一类是去预测未来的画面，或者画面的特征，还有一类是强化学习，用答案或者事件当奖励。

作者指出它们有两个共同问题。第一，监督要么来自人工标注，要么来自外部大模型，比如让大模型写推理过程，或者当裁判打分，这很难规模化。

第二，它们把整段未来当成同样有价值的目标，全部去拟合。可就像刚才厨房的例子，大部分未来信息跟问题无关。

### 核心想法

Future-OPSD 对应两个想法。第一个解决标注问题，叫在线自蒸馏，英文是 on-policy self-distillation。做法是把每段视频切成过去和未来两段。

学生模型只看过去，自己写一段推理加答案。老师是学生训练前的一份冻结副本，它额外能看到真实的未来，然后给学生回答里的每个 token 打分。监督来自视频自己的未来，所以既不需要标注，也不需要外部大模型。

第二个想法解决什么都学的问题。光让老师打分还不够，论文引用的相关研究发现，看到额外信息的老师，打分会被和任务无关的东西带偏。

作者的办法是对比。同一段学生回答，让老师再打一次分，但这次给它看的是另一段不相关视频的未来。两次分数一减，什么未来都支持的上下文就抵消了，只有真实未来特别支持的内容留下来。

### 方法

看这张总览图。左边是普通的监督微调，学一段标注好的未来描述。中间是普通的在线自蒸馏，用老师和学生的打分差，当每个 token 的优势。右边是 Future-OPSD，多了一个看错误未来的老师。

理论上，作者先定义了一个预见分数，借鉴了对比学习里的 InfoNCE。它衡量的是，学生的回答在真实未来下，比在随便一段未来下，平均多被老师支持多少。

这个分数要对所有可能的未来求平均，没法直接算。所以作者换成一个能算的替代目标，只拿一段随机的参考未来来对比，而且它能拆到每个 token 上。

但替代目标有个漏洞，它比真实的预见分数多出一个非负的差值。学生可以钻空子，专挑在不同未来下分数差别大的回答，对比分涨了，预见能力却没提高。

所以作者再加一项 KL 约束，不让学生离看到真实未来的老师太远，并证明这样组合出来的目标，是预见分数的一个下界。作者也写明，下界变高，不一定代表预见分数真的变高。

落到每个 token 上，优势是两项的加权。一项是真实未来和错误未来下，老师打分的差，叫对比优势。另一项是老师和学生打分的差，叫蒸馏优势。

权重 λ 等于 0，就退化成普通的自蒸馏。等于 1，就只剩对比，没有老师的约束，论文说这可能让训练崩掉。

最后，这些优势先裁剪，再减去均值，用 PPO 更新学生。只有学生会更新，推理的时候也只用学生，不需要未来视频。

### 实验和效果

实验设置。学生是 Qwen3-VL-8B 和 Qwen3.5-9B。任务有两个，一个是开放式预测，在 Future-L1 的五万个片段上训练，在 TwiFF-Bench 上测。另一个是四选一的下一步规划，在 EgoPlan-IT 上训练，在 SEED-Bench-R1 上测。

训练只跑 100 步，每步 256 个问题，λ 取 0.75，老师最多看三帧未来画面。开放式题目由 Qwen3.6-27B 当裁判打分，规划题就看选对没有。

先看表 1，开放式预测。Qwen3-VL-8B 在 TwiFF-Bench 上的平均分，从 53.6 提到 67.8，涨了 14.2。普通自蒸馏是 61.2，用外部模型生成文字目标的监督微调是 58.5。

表 2 是规划任务。Qwen3.5-9B 在 SEED-Bench-R1 的第三级上，从 37.9 提到 53.8，涨了 15.9，这是论文报告的最大提升。

同一张表里，同样不用标签的强化学习方法 TTRL，在 Qwen3.5-9B 上反而把规划分数拉低了，第三级从 37.9 掉到 26.4。

还有一点，监督微调会伤害通用视频理解。Qwen3-VL-8B 在四个通用视频基准上的平均分，监督微调后从 56.0 掉到 49.8，而 Future-OPSD 是 57.5，没有下降，还略有提升。

换更大的老师，效果更好。表 3 里，2B 的学生配一个冻结的 4B 老师，规划平均分从 33.9 提到 48.2。

数据规模的实验也显示，随着数据增加，它的提升比普通自蒸馏更明显，数据量大的时候，还超过了用答案标签训练的监督微调和 GRPO。

为什么有效，看这个案例。目标是把面团放到烘焙纸上，问下一步做什么。普通自蒸馏给“观察”这种泛泛的词正的优势，却给关键词“烘焙纸”负的优势。Future-OPSD 把“观察”压低，把“烘焙纸”变成了正的。

训练之后，普通自蒸馏直接跳到把面团放上去，选了 B，答错了。Future-OPSD 先说烘焙纸还没用上，下一步是拿起烘焙纸，选 A，答对了。作者说，这说明对比未来能帮模型分清下一步和最终目标。

### 和以前方法的对比

和 GRPO 的正面对比在表 4。GRPO 每个问题跑八次，用学生自己的基础模型当裁判给奖励。在四个模型上，Future-OPSD 在两个基准上都更好，比如 Qwen3-VL-2B 的 TwiFF 平均分，GRPO 是 52.1，Future-OPSD 是 59.2。

作者给的可能原因是，开放式答案没有可以验证的奖励，大模型裁判给不准。另外，只强化学生自己的回答，未必能迁移到别的任务。

成本方面，Future-OPSD 只比普通自蒸馏多一次老师前向，作者测得 FLOPs 多百分之十九到百分之三十三，训练时间多百分之八到百分之十五。不过它比监督微调贵得多，在 EgoPlan-IT 上用 8 张 B200，监督微调训练 0.62 小时，Future-OPSD 要 3.77 小时。

另外两个设计上的结论。给老师看未来的画面，比看大模型写的未来文字描述效果好。λ 在测试的几个值里，0.75 整体最好，取 1 的时候，在 Future-L1 上训练崩了。

我的判断是，这篇真正的新意，是那个对比的设计。用一段错误的未来当参照，把老师打分里只跟场景有关、跟问题无关的部分减掉。这个做法很便宜，只是多算一次老师前向。

### 局限

作者自己写的局限很简短，只说结论限于论文研究的数据集和模型。

从实验本身看，有几点要注意。学生都在 9B 以内，只有 Qwen3-VL 和 Qwen3.5 两个家族。开放式预测的分数，是另一个大模型裁判打的。规划任务的训练数据，来自 EPIC-KITCHENS 的厨房视频。

理论部分，作者也承认，优化的是下界，下界升高不保证预见分数升高。至于为什么画面比文字好，作者给了一个解释，但也说，分数对比本身不能证明这个机制。

通用视频理解虽然整体没掉，但也不是每一项都涨，比如表 1 里，Qwen3.5-9B 在 TempCompass 上掉了 0.6。

### 未来方向

论文没有专门写未来方向。作者在引言里说，预见能力是规划和控制用的世界模型、自动驾驶、具身机器人的基础。代码作者说会公开。

我的判断是，这套思路最值得看的是能不能扩展。监督只来自视频本身，理论上没标注的视频都能拿来训练，可论文的数据规模实验，最多只到五万条。

我的判断是，用错误的未来当参照，去掉额外信息里的干扰，这个做法不一定只适用于视频，老师比学生多看到信息的蒸馏，都可以试试。不过论文只在视频预见上验证过。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper has a first author at the University at Buffalo, with co-authors from Stony Brook, Johns Hopkins, Duke, the University of Maryland and elsewhere. The method is called Future-OPSD.

In one line: it teaches a vision-language model to watch the first part of a video and guess what happens next, with no human labels, because the supervision comes from the rest of the same video.

### Problem

First, what is video foresight reasoning? You show a model a video of something in progress and ask what happens next, or what I should do next. The paper calls this essential for autonomous driving, embodied robotics, and proactive first-person assistants.

The paper's first figure uses a kitchen example. The question is: given what I see now, what will I do next? The future clip shows a tap, a dish rack, a sink and a sponge. But only one thing actually answers the question: rinse the knife.

The authors call things like the tap context information, and rinsing the knife content information. In this schematic, the split is drawn as eighty percent context and twenty percent content.

### Prior work

Earlier training methods for foresight fall roughly into three groups. Some learn text descriptions and rationales of the future clip. Some predict future frames, or their features. And some use reinforcement learning, with answers or events as rewards.

The authors see two shared problems. First, the supervision comes from human annotation or from external large models, writing rationales or acting as judges. That is hard to scale.

Second, they treat the whole future as an equally valuable target and try to fit all of it. But as the kitchen example shows, most of the future has nothing to do with the question.

### Key idea

Future-OPSD answers with two ideas. The first removes the labels, and it's called on-policy self-distillation. Each video is cut into a past clip and a future clip.

The student model sees only the past and writes its own reasoning and answer. The teacher is a frozen copy of the student from before training, which also gets to see the real future, and it scores every token of the student's response. The supervision comes from the video's own future, so there are no labels and no external large model.

The second idea deals with learning everything. Teacher scores alone aren't enough: related work the paper cites finds that a teacher with extra information can be pulled around by things that have nothing to do with the task.

The fix is a contrast. Take the same student response and have the teacher score it again, but this time show it the future of an unrelated video. Subtract the two scores, and context that any future would support cancels out. What's left is content that the true future specifically supports.

### Method

Here's the overview figure. On the left, plain supervised fine-tuning learns a labeled description of the future. In the middle, plain on-policy self-distillation uses the gap between teacher and student scores as each token's advantage. On the right, Future-OPSD adds a second teacher that sees the wrong future.

On the theory side, the authors first define a foresight score, borrowing from InfoNCE in contrastive learning. It measures how much more the teacher supports the student's answer under the true future than under an average random future.

That score averages over every possible future, so you can't compute it directly. Instead, the authors use a surrogate you can compute: contrast against a single randomly drawn reference future. It also breaks down token by token.

The catch is that the surrogate overshoots the real foresight score by a non-negative gap. A student could game it, favoring answers whose scores swing a lot across futures. The contrast goes up, but foresight doesn't improve.

So the authors add a KL penalty that keeps the student close to the teacher who saw the true future, and they prove the combined objective is a lower bound on the foresight score. They also note that a higher bound doesn't have to mean a higher foresight score.

At the token level, the advantage is a weighted sum of two terms. One is the teacher's score under the true future minus its score under the wrong future: the contrastive advantage. The other is teacher minus student: the distillation advantage.

With the weight lambda at zero, this collapses to plain self-distillation. At one, only the contrast is left, with no teacher anchor, and the paper says training may collapse.

Finally, the advantages are clipped, centered, and fed into a PPO update. Only the student is trained, and at inference only the student runs, with no future video needed.

### Results

The setup. The students are Qwen3-VL-8B and Qwen3.5-9B. There are two tasks. Open-ended prediction trains on fifty thousand clips from Future-L1 and tests on TwiFF-Bench. Next-step planning, as four-way multiple choice, trains on EgoPlan-IT and tests on SEED-Bench-R1.

Training runs for just one hundred steps of two hundred fifty-six questions each, with lambda at zero point seven five, and the teacher sees up to three future frames. Open-ended answers are scored by a Qwen3.6-27B judge; planning is plain accuracy.

Table 1 covers open-ended prediction. Qwen3-VL-8B's average on TwiFF-Bench rises from fifty-three point six to sixty-seven point eight, a gain of fourteen point two. Plain self-distillation reaches sixty-one point two, and supervised fine-tuning on text targets from an external model reaches fifty-eight point five.

Table 2 is planning. Qwen3.5-9B on level three of SEED-Bench-R1 goes from thirty-seven point nine to fifty-three point eight, up fifteen point nine, the largest gain the paper reports.

In the same table, TTRL, another label-free reinforcement learning method, actually lowers Qwen3.5-9B's planning scores. On level three it drops from thirty-seven point nine to twenty-six point four.

Another point: supervised fine-tuning hurts general video understanding. Qwen3-VL-8B's average on four general video benchmarks falls from fifty-six point zero to forty-nine point eight after supervised fine-tuning, while Future-OPSD lands at fifty-seven point five, no loss and a slight gain.

A bigger teacher helps more. In Table 3, a 2B student with a frozen 4B teacher raises its planning average from thirty-three point nine to forty-eight point two.

The data-scaling experiment shows it scales better than plain self-distillation as data grows, and at larger data sizes it overtakes supervised fine-tuning and GRPO, both trained with answer labels.

Why does it work? Look at this case. The goal is to put dough on greaseproof paper, and the question is the next step. Plain self-distillation gives a positive advantage to a generic word like "observation" and a negative one to the key word "greaseproof." Future-OPSD pushes "observation" down and turns "greaseproof" positive.

After training, the plain version jumps straight to putting the dough on the paper, picks B, and gets it wrong. Future-OPSD notes that the paper isn't in use yet, says the next step is to pick it up, picks A, and gets it right. The authors read this as the contrast helping the model separate the next step from the overall goal.

### Comparison

The head-to-head with GRPO is in Table 4. GRPO runs eight rollouts per question and uses the student's own base model as the reward judge. Across four models, Future-OPSD is better on both benchmarks. For Qwen3-VL-2B on TwiFF, GRPO scores fifty-two point one and Future-OPSD fifty-nine point two.

The authors offer two possible reasons. Open-ended answers have no verifiable reward, so a language-model judge can't give accurate signals. And reinforcing only the student's own answers may not transfer to other tasks.

On cost, Future-OPSD adds just one extra teacher forward pass over plain self-distillation. The authors measure nineteen to thirty-three percent more compute and eight to fifteen percent more training time. It is much more expensive than supervised fine-tuning, though. On EgoPlan-IT with eight B200 GPUs, fine-tuning takes zero point six two hours and Future-OPSD three point seven seven.

Two more design findings. Showing the teacher future frames works better than showing it text descriptions of the future written by a large model. And among the lambda values tested, zero point seven five is best overall, while lambda of one collapses training on Future-L1.

My read is that the real novelty here is the contrast. Using a wrong future as the reference subtracts the part of the teacher's score that's about the scene rather than the question. And it's cheap: one extra teacher forward pass.

### Limitations

The authors' own limitations note is brief: their conclusions are limited to the datasets and models they studied.

Looking at the experiments, a few things stand out. Every student is nine billion parameters or smaller, from just two families, Qwen3-VL and Qwen3.5. The open-ended scores come from another language-model judge. And the planning training data is kitchen video from EPIC-KITCHENS.

On the theory, the authors acknowledge that they optimize a lower bound, and raising it doesn't guarantee a higher foresight score. As for why frames beat text, they offer an explanation, but say the score comparison alone doesn't establish that mechanism.

General video understanding holds up overall, but not on every benchmark. In Table 1, Qwen3.5-9B drops zero point six on TempCompass.

### Future work

The paper has no dedicated future-work section. In the introduction, the authors say foresight underpins world models for planning and control, autonomous driving, and embodied robotics. They say the code will be released.

My read is that the thing to watch is scale. The supervision comes from the video itself, so in principle any unlabeled video is training data, yet the paper's data-scaling experiment stops at fifty thousand prompts.

My read is also that using a wrong context as the reference, to strip noise out of privileged information, may not be limited to video. Any distillation where the teacher sees more than the student could try it. But the paper has only tested it on video foresight.

That's today's Daily Frontier Papers. The paper is linked in the description.

