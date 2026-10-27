# RynnVLA-Latent: Scaling Robotic Manipulation Foundation Models with 3D-Grounded Latent Actions

**人类视频也能教机器人**

- 论文 Paper: [arXiv 2610.34356](https://arxiv.org/abs/2610.34356)
- 单位 Affiliation: 阿里巴巴达摩院等
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-10-27
- 视频 Videos: [YouTube 中文](https://youtu.be/Q4sBEQkSKLQ) · B站（审核中）

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇阿里巴巴达摩院等单位的论文，叫 RynnVLA-Latent，讲的是怎么用大量视频，来预训练机器人的操作模型。

一句话概括，先从三万多小时的人类、机器人和仿真视频里，自动提取一种叫潜在动作的标签，让模型先学会东西是怎么动的，再换到具体机器人的控制指令上。

### 问题

先说背景。VLA，也就是视觉-语言-动作模型，看画面、听指令，直接输出机器人的动作。想把它做大，最缺的是带动作标签的数据。

机器人演示数据采集很贵，而且每台机器人的控制信号都不一样，维度、坐标系、含义都不同。人类视频和网上的视频，覆盖的交互场景广得多，但里面没有机器人指令。

所以这些数据，没法直接放进同一个训练目标里，需要一个大家共用的接口。

### 以前的方法

以前有两条路。一条是把不同机器人的动作硬对齐，比如统一成一个固定长度的向量，用不上的维度遮掉，或者给每种机器人配一个单独的输入输出投影层。

另一条是潜在动作，latent action。从前后两帧画面的变化里，学出一个代表这次动作的向量，不需要真实的控制信号。LAPA、Moto 这些工作，都是靠重建下一帧来学。

问题在于，画面变化不全是动作造成的。相机自己在动，背景的外观，还有跟操作无关的变化，都会混进去。画面重建得好，不代表对预测底层动作有用。

### 核心想法

这篇的思路是，给潜在动作加上三维几何的监督，并且把相机运动明确地分离出去。

然后把它当作所有视频共用的标签。预训练只学预测潜在动作，真实机器人的动作和关节状态，只在最后适配某一台机器人时才用。

### 方法

方法分两部分。第一部分叫 RynnLAM，就是潜在动作模型，建在深度估计模型 Depth-Anything-3 之上。这张图左边是以前的做法，右边是它的做法。

输入一对画面，它输出三样东西，8 个局部运动 token，一个 64 维的全局动作向量，和一个 32 维的相机向量，拼起来一共 608 维。

训练目标有几个。第一，预测下一帧特征的变化量，而不是整张画面，让模型盯着变化，少记静态的外观。

第二，预测三维场景流，也就是场景里每个点在三维空间里怎么动。标签是用估计出来的深度、相机位姿和光流算出来的，扣掉相机运动之后，静止的地方运动应该接近零。

第三，把相机运动单独拿出来。相机向量要能还原出两帧之间相机怎么动，同时用一个叫梯度反转的技巧，让动作向量故意猜不出相机运动，这样相机的信息就被赶到相机向量里去了。

训练好以后，只用编码器，给整个视频库离线打标签。

数据有多大。25 个来源，大约 3.01 万小时，1663 万段视频，30.57 亿帧，人类视频和真实机器人视频各占四成左右，仿真占两成。每隔 4 帧取一对画面，一共得到大约 7.46 亿个潜在动作。

第二部分是策略模型 RynnVLA-Latent。它用视觉语言模型 RynnBrain 做主干，旁边接一个窄一些的动作专家网络，两路在每一层一起做注意力。

主干有 2B 和 4B 两个版本，算上动作专家，总参数分别是 25.1 亿和 50.4 亿。

预训练阶段，模型看多视角画面和语言指令，用 flow matching，也就是从噪声一步步生成目标的方法，预测接下来 6 个潜在动作。这一步完全不用机器人动作，也不用机器人自身的关节状态。

到了后训练，主干和动作专家都保留，只把最后的输出层换成一个很薄的新投影，直接输出目标机器人的动作，同时加入关节状态。中间不需要再接一个把潜在动作翻译成动作的解码器。

### 实验和效果

先看潜在动作本身好不好。作者用 LARY 这个基准，把表示冻住，只训练一个小网络，看能不能从中预测机械臂末端的运动，误差越低越好。

在预测相对位移的设置下，RynnLAM 的平均误差是 0.38，DreamDojo 是 0.55，Moto 是 0.77，LAPA 是 0.93。不过也不是每项都赢，在 CALVIN 这个数据集上，DreamDojo 更好。

再看完整的策略。在 LIBERO 上，4B 版本平均成功率 98.8%，是表里最高，但只比 Xiaomi-Robotics-0 高 0.1 个百分点，作者自己也说，这么小的差距不能当成显著差异。

更有意思的是 LIBERO-Plus。它用同样的任务，但加入相机角度、光照、背景、语言等扰动。拿同一个模型直接测，不再专门调整，4B 得到 84.7%，略高于 π0.5 的 84.4%。

但在相机视角和机器人初始状态这两类扰动上，π0.5 更好。

在考验语义理解的 VLABench 上，4B 五个赛道的平均成功率是 51.9%，π0.5 是 48.1%。跨类别那一项，是 48.0% 对 38.2%。

双臂操作的 RoboTwin 2.0 上，4B 在干净场景是 88.7%，随机化场景是 88.4%，两项都是表里最高。要注意，对比方法的分数取自各自的论文，而且这个协议训练时就包含随机化场景的数据。

最后是真机。两台双臂机器人，一台带灵巧手，一台是带轮子的移动机器人，一共 6 个任务，每个任务测 35 次。

4B 在六个任务上都是最高。比如收拾桌子，π0.5 是 25.71%，它是 40%，端牛排上桌，π0.5 是 40%，它是 57.14%。

但也看得出，真机任务的绝对成功率还不高，翻牛排只有 42.86%。而且这是用了目标机器人的演示做后训练之后的结果，每个任务 500 到 850 条，一共 3850 条。

### 和以前方法的对比

那潜在动作，到底比把动作硬对齐好在哪。作者在一个大约 1000 小时的子集上做了对照实验，几种做法用同样的数据。

在有扰动的 LIBERO-Plus 上，用潜在动作的 4B 是 84.2%，统一动作向量是 83.5%，单独投影层是 83.6%。但在不加扰动的 LIBERO 上，反而是两种硬对齐的方法更高。

作者的解读是，潜在动作可能更有利于应对分布变化，而直接对齐动作，在训练分布以内依然有竞争力。

同样条件下，把潜在动作换成 LAPA 的潜在动作，LIBERO-Plus 降了 2.1 和 2.9 个百分点。潜在动作模型的消融里，在预测相对位移的设置下，去掉预测变化量这一项，误差从 0.38 涨到 0.57，影响最大。

数据规模方面，用 10% 到 100% 的预训练视频分别训练，两个尺寸都是用全部数据时 LIBERO-Plus 最好，但 4B 从 10% 到 30% 时有一点下降，并不是一路上升。

我的判断是，这篇最有价值的地方，是把学会东西怎么动，和学会这台机器人怎么控制，拆成了两步。这样没有动作标签的人类视频，也能变成预训练数据。

我的判断是，证据还不算强。在对照实验里，潜在动作的优势不到 2 个百分点，而且只在有扰动的测试上领先。LIBERO 已经接近满分，很难再拉开差距。

### 局限

作者自己列了几条局限。深度、相机位姿和光流都是估计出来的伪标签，所以几何监督不能保证得到准确的真实运动，也不能保证相机和操作的信息完全分开。

冻结表示的测试，只能说明从中能预测动作，不能证明学到的就是动作本身。部署到新机器人，仍然需要这台机器人带动作标签的演示。

数据规模的结果，也不能说明存在一般的规模定律。另外，论文没有把推理速度作为评测指标。

还有一点要提醒，论文里的数字前后不完全一致。RoboTwin 的平均分，表 6 写的是 88.6%，引言和结论写的是 88.3%，结论还说 RoboTwin 分数仍低于最强的基线，这和表 6 对不上。

### 未来方向

作者说，下一步会提升几何一致性和视角鲁棒性，并且让模型适配新机器人时，需要更少的演示。

论文首页列出了项目的 GitHub 和 Hugging Face 地址。

我的判断是，接下来值得看的是，能不能几乎不用目标机器人的演示，就把这些视频里学到的东西用起来。现在真机每个任务还要几百条演示，这才是真正的成本所在。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Alibaba's DAMO Academy and partners. It's called RynnVLA-Latent, and it's about pretraining robot manipulation models on huge amounts of video.

The short version: take more than thirty thousand hours of human, robot and simulation video, automatically label it with something called latent actions, let the model learn how things move, and only then teach it the control commands of a specific robot.

### Problem

Some background first. A VLA, a vision-language-action model, looks at camera images, reads an instruction, and outputs robot actions directly. The bottleneck for scaling these models is data with action labels.

Robot demonstrations are expensive to collect, and every robot speaks a different control language, with different dimensions, coordinate frames and meanings. Human and web videos cover a far broader range of interactions, but they contain no robot commands at all.

So you can't just pour all of that into one training objective. You need a shared interface.

### Prior work

There have been two routes. One is to force the action spaces to line up, say by packing every robot into one fixed-length vector and masking the unused slots, or by giving each robot type its own input and output projection.

The other is latent actions. From the change between two video frames, you learn a vector that stands for the action, with no real control signal needed. Methods like LAPA and Moto learn it by reconstructing the next frame.

The catch is that not every visual change comes from the action. The camera moves, the background has its own appearance, and plenty of changes have nothing to do with manipulation. A model that reconstructs images well isn't necessarily good at predicting low-level actions.

### Key idea

This paper's idea is to give latent actions 3D geometric supervision, and to explicitly separate camera motion out.

Then use those latents as a shared label for every video. Pretraining only predicts latent actions. Real robot actions and joint states come in at the very end, when the model is adapted to one particular robot.

### Method

The method has two parts. Part one is RynnLAM, the latent action model, built on top of the depth model Depth-Anything-3. In this figure, the left side is the conventional design and the right side is theirs.

Given a pair of frames, it outputs three things: eight local motion tokens, a sixty-four dimensional global action vector, and a thirty-two dimensional camera vector. Together that's six hundred and eight dimensions.

There are several training objectives. First, predict how the features change in the next frame, rather than the whole frame, so the model attends to change and memorizes less static appearance.

Second, predict 3D scene flow, meaning how each point in the scene moves in three dimensions. The labels are computed from estimated depth, camera poses and optical flow, and once camera motion is subtracted, static regions should show roughly zero motion.

Third, pull camera motion out on its own. The camera vector has to recover how the camera moved between the two frames, while a trick called gradient reversal makes the action vector deliberately bad at predicting camera motion, which pushes that information into the camera vector.

Once it's trained, only the encoder is used, to label the whole video corpus offline.

How big is the corpus? Twenty-five sources, about thirty thousand one hundred hours, sixteen point six million episodes, and just over three billion frames, with human video and real robot video at roughly forty percent each and simulation at the remaining twenty. Taking a frame pair every four frames yields about seven hundred forty-five million latent actions.

Part two is the policy, RynnVLA-Latent. It uses the vision-language model RynnBrain as its backbone, with a narrower action expert alongside, and the two streams attend jointly at every layer.

The backbone comes in 2B and 4B versions. Counting the action expert, total parameters are two point five one billion and five point zero four billion.

In pretraining, the model looks at multi-view images and the instruction, and uses flow matching, a way of generating targets step by step from noise, to predict the next six latent actions. This stage uses no robot actions and no robot joint state at all.

In post-training, the backbone and the action expert are both kept. Only the final output layer is swapped for a thin new projection that outputs the target robot's actions directly, and joint state is added. There's no extra decoder translating latent actions into robot actions.

### Results

First, are the latent actions themselves any good? The authors use the LARY benchmark: freeze the representation, train only a small probe, and see how well it predicts the motion of the robot's end effector. Lower error is better.

When predicting relative displacement, RynnLAM's average error is zero point three eight, against zero point five five for DreamDojo, zero point seven seven for Moto, and zero point nine three for LAPA. It doesn't win everywhere, though. On the CALVIN dataset, DreamDojo is better.

Now the full policy. On LIBERO, the 4B version averages ninety-eight point eight percent success, the best in the table, but only zero point one points above Xiaomi-Robotics-0, and the authors themselves say that gap shouldn't be read as significant.

LIBERO-Plus is more interesting. It uses the same tasks but adds perturbations: camera angle, lighting, background, language and more. Tested directly with the same model, no extra tuning, the 4B scores eighty-four point seven percent, just above pi zero point five at eighty-four point four.

But on camera viewpoint and robot initial-state perturbations, pi zero point five does better.

On VLABench, which tests semantic understanding, the 4B averages fifty-one point nine percent success across five tracks, against forty-eight point one for pi zero point five. On the cross-category track it's forty-eight point zero against thirty-eight point two.

On the bimanual benchmark RoboTwin 2.0, the 4B scores eighty-eight point seven percent in clean scenes and eighty-eight point four in randomized scenes, the top of the table on both. Two caveats: the baseline numbers are copied from their own reports, and this protocol includes randomized scenes in the training data.

Finally, real robots. Two dual-arm platforms, one with dexterous hands and one on a wheeled mobile base, six tasks in total, thirty-five trials per task.

The 4B has the highest success on all six. For table cleaning, pi zero point five gets about twenty-six percent and this model gets forty. For serving a steak, pi zero point five gets forty percent and this model gets about fifty-seven.

But absolute real-world success is still modest. Flipping a steak succeeds only about forty-three percent of the time. And that's after post-training on demonstrations from the target robot, five hundred to eight hundred fifty per task, three thousand eight hundred fifty in total.

### Comparison

So what do latent actions buy you over forcing the actions to line up? The authors ran a controlled comparison on a subset of about one thousand hours, with every approach trained on the same data.

On the perturbed LIBERO-Plus, the latent-action 4B scores eighty-four point two percent, against eighty-three point five for the shared action vector and eighty-three point six for per-robot projections. But on unperturbed LIBERO, the two explicit alignment methods actually score higher.

The authors' reading is that latent actions may help more under distribution shift, while direct action alignment stays competitive inside the training distribution.

Under the same conditions, swapping in LAPA's latent actions drops LIBERO-Plus by two point one and two point nine points. In the latent model's ablations, in the relative-displacement setting, removing the change-prediction objective hurts most, raising the error from zero point three eight to zero point five seven.

On data scale, they pretrained on ten to one hundred percent of the video. Both sizes do best on LIBERO-Plus with all of the data, but the 4B dips slightly from ten to thirty percent, so the curve isn't a straight climb.

My read is that the most valuable thing here is splitting learning how things move from learning how this particular robot is controlled. That's what lets unlabeled human video become pretraining data.

My read is also that the evidence isn't strong yet. In the controlled comparison, the latent-action advantage is under two points, and it only leads on the perturbed test. LIBERO is close to saturated, so it's hard to separate methods there.

### Limitations

The authors list several limitations. Depth, camera poses and flow are all estimated pseudo-labels, so the geometric supervision can't guarantee accurate real-world motion, or a clean split between camera and manipulation information.

Frozen-representation probes only show that actions can be predicted from the latents, not that the latents are the actions. And deploying to a new robot still needs action-labeled demonstrations from that robot.

The data-scaling results don't establish a general scaling law either. Also, inference speed was not one of the evaluation metrics.

One more thing to flag: the paper's numbers aren't fully consistent. For the RoboTwin average, Table 6 says eighty-eight point six percent, while the introduction and conclusion say eighty-eight point three, and the conclusion also says RoboTwin scores remain below the strongest baselines, which doesn't match Table 6.

### Future work

Looking ahead, the authors plan to improve geometric consistency and viewpoint robustness, and to need fewer demonstrations when adapting to a new robot.

The first page lists GitHub and Hugging Face pages for the project.

My read is that the thing to watch is whether the knowledge in all this video can be put to work with almost no demonstrations from the target robot. Right now each real task still needs hundreds of them, and that's where the real cost sits.

That's today's Daily Frontier Papers. The paper is linked in the description.

