# Beyond the Happy Path: Generalizing Robot World Models from Simulated Failures

**机器人抓歪了世界模型看不懂**

- 论文 Paper: [arXiv 2611.33925](https://arxiv.org/abs/2611.33925)
- 单位 Affiliation: 浙江大学、上海创智学院
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-28
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1eA1MzGEwh)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇浙江大学的论文，标题叫 Beyond the Happy Path，意思是走出顺利路径，从仿真里的失败中，让机器人世界模型学会泛化。

它关心的问题很具体，机器人抓东西没抓好的时候，世界模型能不能预测出接下来会发生什么。作者给出的方法叫 WorldReforge。

### 问题

先解释一下这里说的世界模型。它叫动作条件的世界模型，给它一张当前画面，再给一串机器人接下来要执行的动作，它生成未来的视频。现在越来越多人用它代替真机，去评估策略、生成数据、做规划。

这种模型有没有用，取决于它能不能分清不同动作带来的不同结果。操作物体恰恰最难。抓取偏了一厘米，或者夹爪合得太早，并不是简单地失败，物体会被推开，被碰歪，或者从手指间滑出去。

可是机器人的训练数据，大多是成功的演示，这一类物理现象很少出现。作者把这叫做顺利路径的偏差，也就是 happy path bias。

作者先做了一个诊断，结果在表 1。他们拿一个最笨的基线当参照，直接把第一帧一直复制下去，画面里什么都不动。三个模型家族的五个预训练模型，物体运动误差最多只比这个静止基线低百分之 10.4，其中两个还不如它。

也就是说，即使真实情况里物体被推走了，或者转了个角度，这些模型预测出来的物体，基本还停在原地。作者提醒，用这样的模型去评估策略，可能会高估策略。

### 以前的方法

最近也有一些工作拿失败数据来微调世界模型，失败来自扰动动作，或者真实执行。有的保留原来的生成目标，有的用任务进度或者奖励信号来监督结果。

作者认为还剩两个问题。第一，接触发生的区域只占画面很小一块，全局的重建损失几乎照顾不到它。第二，结果层面的信号只告诉你失败了没有，不告诉你物体具体是怎么动的。

### 核心想法

作者的思路是，真机日志里缺的东西，仿真里都有。失败可以按控制好的偏移和时机批量制造，安全，规模也大，而且每一条都带着精确的物体位姿、接触和表面几何。

所以 WorldReforge 分两部分。一是造一个专门教失败的数据集，二是把仿真里的真值，变成两个额外的训练信号，而且推理时不增加任何成本。

### 方法

先看数据集，叫 Reforge。作者先在仿真里，用 RoboLab 和 cuRobo 生成成功的抓取，再去扰动它，改末端的位置，改朝向，改夹爪合上的时机。

最后得到六万两千多条仿真轨迹，覆盖 96 个物体，其中一万三千五百多条和失败有关。另外还有一个真实数据集，叫 Reforge-Real，是 300 条真机上的失败轨迹，只用来评测。

每条轨迹都带着时间对齐的标签，什么时候接触，什么时候脱离，物体有没有被推动、有没有倾斜，抓没抓住，抓住之后有没有一直保持住。

第一个训练信号叫 Interaction Alignment，简称 IA，交互对齐。用仿真里物体的真实位置，在画面上圈出物体和接触附近的区域，单独在这块区域上算重建误差，再按区域大小归一化。这样区域再小，也有一份专门的学习信号。

第二个叫 Surface Correspondence，简称 SC，表面对应。在物体表面取一些点，仿真知道这些点在之后每一帧投影到画面的哪个位置。训练时要求模型内部的特征，能在后面的帧里找到同一个物理表面点。

一个管空间上该看哪里，一个管时间上跟着谁动。两个损失都加在原来的生成目标上，共用同一个主干网络，推理时不加参数，也不改结构。

### 实验和效果

实验设置。训练用 Reforge 里的 2.56 万个交互片段，将近一半来自失败轨迹。评测一共 1500 段，来自仿真评测集，公开的真机数据集 DROID，还有 Reforge-Real。

被改造的模型来自三个家族，英伟达的 Cosmos3，DreamDojo，和 LingBot-VA。每个模型根据一张初始画面和 16 步动作，预测未来 16 帧。物体和夹爪怎么动，用点跟踪工具 CoTracker 从生成的视频里提取，再和仿真真值比，误差单位是像素。

主要结果看 Cosmos3 Nano。和预训练版本比，物体的平均轨迹误差从 9.07 降到 6.51 个像素，降了百分之 28.2。全部评测视频的 PSNR，也就是画面重建质量，提高了 3.27 dB。

不过大部分提升来自数据本身。只用 Reforge 数据训练，不加 IA 和 SC，物体误差已经降了百分之 24.9。在同样的数据和训练预算下，加上 IA 和 SC，再降百分之 4.4。

换成训练时没见过的物体，效果更明显，物体误差比预训练版本降了百分之 35.0，这组对比在表 2。

这张图是三个例子，一个转动的酱料瓶，一个倾斜的遥控器，一个被推动的红辣椒。预训练模型的预测基本停在原位，只加数据的版本能抓到大致的运动趋势，WorldReforge 和真实的方向、位移、最终姿态更接近。

论文还有一个测试。同一个初始画面，给两组不同的动作，比如一组让记号笔滑掉，一组成功拿起来，看模型能不能预测出两个结果之间的差别。在 100 对这样的样本上，这个差别的误差，WorldReforge 比预训练模型低百分之 43.3。

真实视频上，虽然训练只用了仿真数据，Cosmos3 Nano 在 DROID 上的 PSNR 提高了 1.04 dB，在真机失败视频上提高了 0.68 dB。

数据配比的实验，得出一个有意思的分工。在成功抓握的数据里，混入夹爪合上了却没抓住的失败数据，这一类情况的物体误差从 12.78 降到 10.48 个像素，降了百分之 18.0。

但如果要让模型预测抓住的东西保持稳定，只用成功数据训练反而最好。作者的结论是，失败接触教的是物体怎么反应，成功抓握教的是抓住的状态怎么保持。

消融实验在表 7。在 6400 个片段的小规模设置下，IA 和 SC 单独加都有帮助，一起用，物体误差和夹爪误差都最低。

附录里作者还试了，用 CoTracker 从视频里估出来的伪标签，代替仿真真值来做 SC，效果不如用真值，用真值的版本，物体误差低了大约百分之 4。

### 和以前方法的对比

需要说明的是，论文的对比，是同一个模型的三个版本，预训练、只加数据、数据加 IA 和 SC，再加上那个静止基线。表格里没有和其他同样用失败数据的方法做正面比较。

不同模型的效果差别也很大。DreamDojo 用 Reforge 数据训练后，物体误差反而从 9.57 升到 11.81 个像素，加上 IA 和 SC 是 11.62，仍然比预训练版本和静止基线都差，不过它在仿真评测集上的夹爪误差和画面质量是改善的。

我的判断是，这篇最有价值的是那个诊断。现成的机器人世界模型，在接触出问题的时候，几乎等于没在预测物体。如果你在用世界模型给策略打分，值得先拿自己的模型测一下这件事。

我的判断是，IA 和 SC 带来的额外提升在几个百分点的量级，真正起主要作用的，是这批富含失败的仿真数据。

### 局限

论文没有单独列局限，下面几点来自它的实验设置。第一，模型只预测 16 帧，评测里按 15 帧每秒算，大约一秒钟，是很短的片段。

第二，在真实视频上，论文只报告了 PSNR 这类画面重建指标，提升在 0.7 到 1 dB 左右，没有在真实视频上测物体运动误差，真实视频也只用来评测。

第三，论文测的是视频预测本身，没有测换成 WorldReforge 之后，策略评估或者规划是不是真的更准。

### 未来方向

作者说，下一步会把这个框架扩展到更长时程的操作，并研究预测动作后果，能怎样帮助闭环的机器人控制。数据、模型和数据管线，作者说会在论文发表时开源。

我的判断是，世界模型要真正替代真机做评估，就必须能预测失败。这篇给了一个可以复用的办法，用仿真批量制造带真值的失败，再把真值变成训练信号。

我的判断是，接下来最值得看的，是这些改进能不能变成更准的策略打分，这一点论文还没有回答。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Zhejiang University, and its title is Beyond the Happy Path: generalizing robot world models from simulated failures.

The question is very concrete. When a robot fumbles a grasp, can a world model predict what happens next? The authors' answer is a method called WorldReforge.

### Problem

First, what kind of world model are we talking about. It's an action-conditioned world model: you give it the current camera image and a sequence of upcoming robot actions, and it generates the future video. These models are increasingly used in place of real robots, to evaluate policies, generate data, and plan.

That only works if the model can tell different actions apart by their outcomes. And manipulation is where that's hardest. A grasp that's a centimeter off, or a gripper that closes too early, doesn't just fail. The object gets pushed, tipped over, or slips out of the fingers.

The problem is that robot training data is mostly successful demonstrations, so this kind of physics is rarely seen. The authors call it happy-path bias.

They start with a diagnosis, in Table 1. The reference point is the dumbest possible baseline: copy the first frame forward, so nothing ever moves. Across five pretrained models from three families, object-motion error is at most ten point four percent lower than that static baseline, and two of the models are actually worse than it.

In other words, even when the recorded object gets pushed or rotated, these models keep it roughly where it started. And the authors point out that if you evaluate policies with a model like that, you risk overrating them.

### Prior work

Some recent work does fine-tune world models on failures, collected by perturbing actions or by running them. Some keep the original generative objective; others supervise outcomes with progress or reward signals.

The authors see two gaps. First, contact happens in a tiny part of each frame, so a global reconstruction loss barely weighs it. Second, an outcome-level signal tells you whether an interaction failed, not how the object actually moved.

### Key idea

The insight is that simulation has exactly what real robot logs lack. You can produce failures at controlled offsets and timings, safely and at scale, and every one comes with exact object pose, contact, and surface geometry.

So WorldReforge has two parts. A dataset built to teach failure, and two extra training signals that turn the simulator's ground truth into supervision, with no added cost at inference.

### Method

Start with the dataset, called Reforge. The authors generate successful grasps in simulation with RoboLab and cuRobo, then perturb them: shift the end-effector position, change its orientation, or change when the gripper closes.

That gives just over sixty-two thousand simulated trajectories across ninety-six objects, about thirteen and a half thousand of them failure-related. There's also Reforge-Real, three hundred failed grasps recorded on a real robot, used only for evaluation.

Every trajectory carries time-aligned labels: when contact starts and ends, whether the object is pushed or tilted, whether a grasp is established, and whether it's held.

The first training signal is Interaction Alignment, or IA. Using the object's true position from the simulator, it marks the object and the area around the contact, computes reconstruction error just inside that region, and normalizes by the region's size. So even a small region gets a learning signal of its own.

The second is Surface Correspondence, or SC. Pick points on the object's surface. The simulator knows exactly where each point lands in every future frame. During training, the model's internal features have to find the same physical surface point in later frames.

One tells the model where to look in space; the other tells it what to follow through time. Both losses are added to the original generative objective and share the same backbone, so there are no new parameters at inference and no architecture change.

### Results

The setup. Training uses twenty-five point six thousand interaction segments from Reforge, nearly half of them from failed trajectories. Evaluation covers fifteen hundred clips, from the simulated test set, the public real-robot dataset DROID, and Reforge-Real.

The models come from three families: NVIDIA's Cosmos3, DreamDojo, and LingBot-VA. Each predicts sixteen future frames from one starting image and sixteen action steps. Object and gripper motion are pulled out of the generated video with a point tracker, CoTracker, and compared with the simulator's ground truth, in pixels.

The headline numbers are for Cosmos3 Nano. Compared with the pretrained model, average object trajectory error falls from nine point zero seven to six point five one pixels, a twenty-eight point two percent drop. PSNR, a measure of reconstruction quality, rises by three point two seven decibels across all evaluation videos.

But most of that gain comes from the data itself. Training on Reforge with the plain objective, no IA or SC, already cuts object error by twenty-four point nine percent. Adding IA and SC, with the same data and training budget, takes off another four point four percent.

On objects never seen in training, the gain is larger: object error drops thirty-five point zero percent versus the pretrained model. That's Table 2.

This figure shows three examples: a rotating sauce bottle, a tilting remote control, and a red pepper being pushed. The pretrained model's predictions barely move, the data-only version gets the rough trend, and WorldReforge comes closest to the recorded direction, displacement, and final pose.

The paper runs one more test. Take one starting image and two different action sequences, say one where a marker slips and one where it's lifted. Can the model predict how the two futures differ? On one hundred such pairs, WorldReforge's error on that difference is forty-three point three percent lower than the pretrained model's.

On real video, even though training used only simulation, Cosmos3 Nano's PSNR improves by one point zero four decibels on DROID and zero point six eight on the real failure videos.

The data-mix experiments turn up a neat division of labor. Add failed contacts, where the gripper closes but doesn't secure the object, to successful grasps, and object error on that kind of contact falls from twelve point seven eight to ten point four eight pixels, an eighteen percent drop.

But for predicting that a held object stays put, training only on successful holds is best. The authors' conclusion: failed contact teaches how objects respond, and successful holding teaches how a grasp persists.

The ablation is Table 7. In a smaller run on sixty-four hundred segments, IA and SC each help on their own, and together they give the lowest object and gripper error.

In the appendix, the authors also try SC with pseudo-labels, point tracks estimated by CoTracker, instead of simulator ground truth. Ground truth wins, with object error about four percent lower.

### Comparison

One thing to be clear about: the comparisons are three versions of the same model, pretrained, data only, and data plus IA and SC, along with the static baseline. The tables don't include a head-to-head against other methods that also train on failures.

Results also vary a lot by model. For DreamDojo, training on Reforge actually raises object error, from nine point five seven to eleven point eight one pixels, and with IA and SC it's eleven point six two, still worse than both its pretrained version and the static baseline, even though its gripper error and visual quality on the simulated test set improve.

My read is that the most valuable part of this paper is the diagnosis. Off-the-shelf robot world models are barely predicting the object at all once contact goes wrong. If you're scoring policies with a world model, it's worth testing your own model for this.

My read is that IA and SC add a few percent on top, and the main driver is the failure-rich simulated data.

### Limitations

The paper has no limitations section, so these come from its setup. First, the models predict only sixteen frames, about one second at the fifteen frames per second used in evaluation. That's a short clip.

Second, on real video the paper reports only reconstruction metrics like PSNR, with gains of roughly zero point seven to one decibel. It doesn't measure object-motion error on real footage, and the real videos are used only for evaluation.

Third, the paper evaluates video prediction itself. It doesn't test whether swapping in WorldReforge actually makes policy evaluation or planning more accurate.

### Future work

The authors plan to extend the framework to longer-horizon manipulation, and to study how predicting the consequences of actions can support closed-loop robot control. They say the data, models, and data pipeline will be open-sourced on publication.

My read is that if world models are going to stand in for real robots in evaluation, they have to predict failure. This paper offers a reusable recipe: mass-produce failures in simulation with ground truth, then turn that ground truth into training signals.

My read is that the thing to watch is whether these gains turn into more accurate policy scores. The paper doesn't answer that yet.

That's today's Daily Frontier Papers. The paper is linked in the description.

