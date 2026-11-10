# Learning Heterogeneous Tactile Representation for General Dexterous Manipulation

**换个传感器不用重新训练**

- 论文 Paper: [arXiv 2611.12806](https://arxiv.org/abs/2611.12806)
- 单位 Affiliation: 蚂蚁集团 Robbyant
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-10
- 视频 Videos: —

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇蚂蚁集团 Robbyant 团队的论文，题目是学习异构触觉表征，用于通用的灵巧操作，他们做的模型叫 LingBot-Touch。

一句话概括，它想做一个触觉的基础模型，不管机器人手上装的是哪种触觉传感器，都用同一个编码器来读懂。

### 问题

现在的机器人做操作，几乎全靠眼睛看。可视觉回答不了，手按得有多重，抓着的东西是不是在打滑，被手指挡住的地方发生了什么。

触觉传感器其实不缺，压阻式、磁性、视触觉，都能买到。视触觉，就是在一层凝胶后面放摄像头，拍接触时的形变。

但作者说，触觉缺少让视觉和语言做大的三样东西。没有统一的输入格式，每种传感器的通道、排布、分辨率、采样率都不一样。没有大规模的数据集。也没有公认的训练方法。

作者认为，这三个问题是同一个根，触觉数据绑死在产生它的传感器物理特性上。但接触本身不是。手指按在一条棱上，不管是磁力计、电阻网格，还是凝胶后面的摄像头记录下来，都是同一件事。

### 以前的方法

以前也有人做触觉表征学习，比如 Sparsh、T3、AnyTouch，其中很多主要针对光学触觉传感器。

也有方法专门处理不同种类的传感器，最接近的是 FTP-1，它把不同传感器映射到同一套 token 空间，但适配新硬件，要靠为每种传感器单独训练的编码器。

按作者的说法，这类方法每换一种传感器型号，都需要一个专门的适配器，或者专门的 tokenizer。

### 核心想法

LingBot-Touch 的思路是，把传感器自身的物理特性从输入里去掉，只留下接触本身。论文说靠的是三个设计。

第一，把每个传感器的触点，映射到一个统一的指尖坐标空间。模型看到的是手指上的位置，而不是某个传感器数组里的编号。

第二，用 JEPA 的方式训练，也就是在特征空间里，预测被遮住的那部分，而不是还原原始信号，这适合稀疏又没有标签的触觉数据。

第三，收集了一个 5000 小时的触觉数据集，作者称，这是第一个大到可以做表征学习的触觉语料。

### 方法

先看数据。准确说是 4983 小时，46 万多个片段，大约 13.3 亿帧，覆盖压阻、磁性和视触觉三类传感器。

大约三分之二，来自人戴着触觉手套做日常操作，剩下三分之一，来自遥操作的双臂机器人，手是 Sharpa Wave，指尖是视触觉传感器。片段分成 24 类任务，但没有力、接触位置、接触时间的标注。

再看怎么统一不同的传感器。每个指尖先对齐到标准朝向，再缩放、平移到一个半径为 1 的参考形状里。

每个 token 有一个五维地址，时间窗口、哪根手指，加上三维位置。这个地址通过旋转位置编码，也就是 RoPE，进入模型，所以不需要固定的 token 数量，新传感器的排布也能直接处理。

传感器专属的参数，只放在网络的最外层。三类传感器各有一个两层的小网络，把一个触点连续两帧的读数，一共 6 个数，变成一个 token，后面接一个所有传感器共享的 Transformer，叫 Haptic Transformer。

这张图是训练框架。上下文编码器只看没被遮住的 token，目标编码器看完整片段，参数是上下文编码器的指数滑动平均。预测器要根据可见部分，猜出目标编码器在每个位置的特征。

为什么不直接还原信号。因为触觉信号大部分是接近零的背景，如果还原信号，损失主要被背景占住，在特征空间里预测，就没有这个问题。

遮挡也有讲究。随机选一些触点，在整个片段里都遮住，最终遮掉 80%。但一开始就遮这么多，很多片段的接触会被全部遮掉，作者观察到了梯度爆炸，所以前 300 步，把遮挡比例从 40% 线性升到 80%。

还有一个关键的数据增强。预训练只见过三种触点排布，模型可能会死记这三套坐标。所以训练时，随机改变指尖的采样密度，再把每个触点的位置，在邻域里随机挪一点。

模型有两个尺寸，Tiny 是 546 万参数，Small 是 2176 万参数。

### 实验和效果

实验分两部分。第一部分冻结编码器，只训练一个很轻的读出头，看表征里有没有接触的物理信息。

测试用两只手。Sharpa Wave 的传感器，预训练时见过。XHand1 用的是磁性传感器，但这个型号预训练里完全没有，也没有为它更新任何参数。

五个任务分别是，估计指尖受力，定位接触中心，估计接触面积，预测未来四帧的力，以及只凭触觉，找出同一时刻的视频画面。

在 Sharpa Wave 上，受力误差 0.81 牛，接触中心误差 0.12 毫米，对比的 FTP-1 是 2.31 牛和 2.19 毫米。视频检索的前五召回率，从 16.5% 提到 72.6%。

在没见过的 XHand1 上，受力误差 1.32 牛，接触中心误差 0.16 毫米，检索召回 47.1%。FTP-1 即使专门训练了适配器，也只有 3.72 牛、5.89 毫米和 6%。

数据量方面，Small 模型从 1000 小时到 5000 小时，每加 1000 小时，每个任务都变好或者持平。Tiny 到 4000 小时左右就平了，作者的结论是，模型容量要跟着数据一起涨。

更有意思的是跨传感器。固定某种传感器自己的数据，再加入其他传感器的数据，8 组传感器和任务的组合里，有 7 组变好。磁性那边的受力误差，从 4.34 牛降到 2.52 牛。

但在总共 1000 小时的固定预算下，只用自己传感器数据的模型，在 8 组里赢了 7 组，空间定位上差距最大。作者的解读是，力和动态能跨传感器迁移，空间精度要靠见过自己的排布。

他们也比较了训练目标。在同样的 1000 小时数据上，JEPA 的接触中心误差是 0.90 毫米，两种还原信号的方法，VAE 和 MAE，都在 3 毫米以上。VAE 加了数据，五个任务里反而有四个变差。

第二部分是真机。冻结的 Tiny 编码器输出触觉 token，作为额外的条件输入接进策略模型，给三个 VLA 模型增加的参数量，只有百分之 0.13 到 0.16。

测了四种策略，扩散策略 Diffusion Policy，和三个 VLA 模型，π0.5、LingBot-VLA、T-Rex。每种比三个版本，不加触觉，加原始触觉信号，加 LingBot-Touch 的 token。每个任务 200 条示范，双手任务 500 条，每个任务测 20 次。

26 个真机任务平均，Diffusion Policy 的成功率从 21.6% 到 35.6%，π0.5 从 48.1% 到 69.9%，LingBot-VLA 从 48.3% 到 73.1%，T-Rex 从 45.4% 到 68.6%。

关键的对照是原始触觉信号。直接喂原始信号，四种策略只提升 6.6 到 11.2 个百分点，而且在 104 个任务和策略的组合里，有 22 个反而变差。

在没见过的 XHand1 上更明显。原始信号几乎没用，LingBot-Touch 的 token，给四种策略加了 13.8 到 24.5 个百分点。

涨得最多的，是视觉看不清的那一步，比如拉链有没有咬合，喷壶扳机扣到位没有。LingBot-VLA 的拉链任务，从 11% 涨到 55%。

仿真里的强化学习也是一样。12 个灵巧手任务平均，成功率 69%，和策略一起训练的 CNN 特征是 45.6%，不用触觉是 21.4%，不同随机种子之间的波动也小得多。

### 和以前方法的对比

作者还把 FTP-1 的触觉编码器，放进同一个 LingBot-VLA 策略里，五个任务平均成功率 67.5%，LingBot-Touch 是 82.2%，不加触觉是 55.9%。

而且 FTP-1 在其中两个任务上让成功率下降了，LingBot-Touch 在五个任务上都不低于原策略。

不过它也不是处处更好。在 104 个组合里，有 9 个不如加原始信号，有 5 个不如不加触觉。

我的判断是，这篇最有价值的是那组对照，同样的触觉信号，直接输入经常帮倒忙，换成预训练的表征就稳定有用。触觉进不了主流策略，问题不只在传感器，更在表征。

我的判断是，弱点也要说清楚。真机结果都在作者用的两款手、自己挑的任务上，附录里的评分是按步骤给分的，FTP-1 的适配器也是作者训练的，还没有独立复现。

### 局限

作者自己列了三条局限。第一，需要在更多种类的机器人手上，做真机和仿真实验。

第二，电容式传感器没有进预训练，因为他们手上的电容数据，质量达不到要求。

第三，现在只覆盖指尖，手掌和手指其他部位的接触，覆盖得很少。

### 未来方向

作者说，下一步要把覆盖范围从指尖扩展到整只手。他们还计划公开推理代码、少量样本数据，以及 Tiny 和 Small 两个模型的权重。

作者还有一个观点。既然数据能跨传感器迁移，扩大数据时，不必用最强的传感器，而该用最容易大规模采集的那个，这也是为什么三分之二的数据来自人戴的手套。

我的判断是，如果这个思路成立，触觉数据的瓶颈，会从买什么传感器，变成怎么组织人去采集。值得看的是，权重公开以后，别的实验室能不能在自己的手上复现这些提升。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the Robbyant team at Ant Group. It's called Learning Heterogeneous Tactile Representation for General Dexterous Manipulation, and the model is LingBot-Touch.

The pitch in one line: a foundation model for touch, where one encoder can read whatever tactile sensor happens to be on the robot's hand.

### Problem

Some context first. Robots today manipulate things almost entirely by sight. But sight can't tell a robot how hard it's pressing, whether a grasp is slipping, or what's happening under a finger that blocks the view.

It's not that touch sensors don't exist. Piezoresistive, magnetic and visuotactile sensors are all on the market. A visuotactile sensor is a camera behind a gel pad, filming how the gel deforms on contact.

The authors argue touch is missing the three things that let vision and language scale. There's no common input format, since every sensor has its own channels, layout, resolution and sampling rate. There's no large dataset. And there's no established training recipe.

They trace all three gaps to one root: tactile data are tied to the physics of the sensor that recorded them. Contact itself is not. A fingertip pressing on an edge is the same event whether a magnetometer, a resistive grid, or a camera behind a gel records it.

### Prior work

There is earlier work on tactile representation learning, like Sparsh, T3 and AnyTouch, and much of it focuses on optical tactile sensors.

Some methods do target mixed sensor types. The closest is FTP-1, which maps different sensors into a shared token space, but adapts to new hardware through encoders trained for each specific sensor.

In the authors' framing, approaches like that need a dedicated adapter or tokenizer for every new sensor model.

### Key idea

LingBot-Touch tries to strip the sensor's own physics out of the input and keep only the contact. The paper credits three design decisions.

First, every sensor's taxels, its individual sensing points, are mapped into one canonical fingertip space. The model sees positions on a finger, not indices in some sensor-specific array.

Second, it trains JEPA-style, predicting the hidden parts in feature space rather than reconstructing the raw signal, which suits tactile data that are sparse and unlabeled.

Third, they collected a five-thousand-hour tactile dataset, which the authors call the first at a scale where representation learning for touch becomes possible.

### Method

Start with the data. Precisely, it's four thousand nine hundred and eighty-three hours, over four hundred and sixty thousand episodes, and about one point three three billion frames, across piezoresistive, magnetic and visuotactile sensors.

About two thirds come from people wearing tactile gloves while doing everyday manipulation. The remaining third comes from a teleoperated dual-arm robot with Sharpa Wave hands, whose fingertips carry visuotactile sensors. Every episode belongs to one of twenty-four task categories, but nothing is labeled with force, contact location or contact timing.

Next, how do you unify the sensors? Each fingertip is first aligned to a canonical orientation, then scaled and shifted into a reference shape with a radius of one.

Every token gets a five-dimensional address: its time window, which finger, and a 3D position. That address enters the model through rotary position embeddings, or RoPE, so there's no fixed token count, and a new sensor layout can be processed directly.

Sensor-specific parameters live only at the edges of the network. Each of the three sensor types has a small two-layer network that turns two consecutive frames of one taxel, six numbers in all, into a token. Those tokens go into a Transformer shared by every sensor, called the Haptic Transformer.

This figure shows the training setup. A context encoder sees only the unmasked tokens. A target encoder sees the whole clip, and its weights are an exponential moving average of the context encoder. A predictor has to guess the target encoder's features at every position from the visible part.

Why not just reconstruct the signal? Because most of a tactile clip is near-zero background, so a reconstruction loss is dominated by that background. Predicting in feature space avoids the problem.

Masking needs care too. Random taxels are hidden across the whole clip, eventually eighty percent of them. But mask that much from the start, and many clips lose every bit of contact, and the authors saw gradient explosions. So over the first three hundred steps, the mask ratio ramps linearly from forty to eighty percent.

There's also a key augmentation. Pretraining only sees three taxel layouts, so the model could simply memorize three sets of coordinates. So during training they vary how densely each fingertip is sampled, and nudge each taxel's position a little within its neighborhood.

There are two model sizes: Tiny at five point four six million parameters, and Small at twenty-one point seven six million.

### Results

The experiments come in two parts. Part one freezes the encoder and trains only a light readout head, to test whether the representation carries the physics of contact.

Two hands are used. Sharpa Wave's sensors were seen in pretraining. XHand1 uses a magnetic sensor model that's completely absent from pretraining, and no parameters are updated for it.

The five tasks are: estimating fingertip force, locating the contact center, estimating contact area, predicting force over the next four frames, and, from touch alone, retrieving the video frame recorded at the same moment.

On Sharpa Wave, force error is zero point eight one newtons and contact-center error is zero point one two millimeters, against two point three one newtons and two point one nine millimeters for FTP-1. Top-five video retrieval recall jumps from sixteen point five to seventy-two point six percent.

On the unseen XHand1, force error is one point three two newtons, contact-center error zero point one six millimeters, and retrieval recall forty-seven point one percent. FTP-1, even with an adapter trained specifically for this hand, gets three point seven two newtons, five point eight nine millimeters, and six percent.

On data scale, the Small model improves or holds on every task with each extra thousand hours, from one thousand to five thousand. Tiny flattens out around four thousand hours, and the authors conclude that model capacity has to grow with the data.

The cross-sensor result is more interesting. Hold a sensor's own data fixed and add data from other sensors, and seven of eight sensor-task pairs improve. On the magnetic side, force error falls from four point three four to two point five two newtons.

But at a fixed budget of one thousand hours in total, a model trained only on its own sensor wins seven of eight pairs, with the biggest margins on spatial precision. The authors' reading: force and dynamics transfer across sensors, while spatial precision depends on seeing your own layout.

They also compared training objectives. On the same one thousand hours, JEPA's contact-center error is zero point nine zero millimeters, while two reconstruction methods, a VAE and an MAE, are both above three. And the VAE actually gets worse on four of five tasks when given more data.

Part two is real robots. The frozen Tiny encoder's tactile tokens are fed into a policy as an extra conditioning input, adding only zero point one three to zero point one six percent to the parameters of the three VLA models.

They test four policies: Diffusion Policy, plus three vision-language-action models, pi zero point five, LingBot-VLA, and T-Rex. Each comes in three versions: no touch, raw tactile signal, and LingBot-Touch tokens. Every task gets two hundred demonstrations, five hundred for two-handed tasks, and twenty test trials.

Averaged over twenty-six real tasks, Diffusion Policy goes from twenty-one point six to thirty-five point six percent, pi zero point five from forty-eight point one to sixty-nine point nine, LingBot-VLA from forty-eight point three to seventy-three point one, and T-Rex from forty-five point four to sixty-eight point six.

The key control is the raw tactile signal. Fed raw signals, the four policies gain only six point six to eleven point two points, and in twenty-two of the one hundred and four task-policy pairs, raw touch makes things worse.

On the unseen XHand1 it's starker. Raw signals barely help, while LingBot-Touch tokens add thirteen point eight to twenty-four point five points across the four policies.

The biggest gains land on the step vision can't resolve, like whether a zipper slider has engaged, or whether a spray trigger has been pulled far enough. LingBot-VLA on zipper closing goes from eleven to fifty-five percent.

Reinforcement learning in simulation tells the same story. Across twelve dexterous-hand tasks, average success is sixty-nine percent, against forty-five point six for CNN features trained with the policy and twenty-one point four with no touch, and results vary far less across random seeds.

### Comparison

The probing tasks already compared against FTP-1. The authors also dropped FTP-1's tactile encoder into the same LingBot-VLA policy. Across five tasks, it averages sixty-seven point five percent, versus eighty-two point two for LingBot-Touch and fifty-five point nine with no touch.

And FTP-1 lowered success on two of those tasks, while LingBot-Touch was at or above the original policy on all five.

It isn't better everywhere, though. In nine of the one hundred and four pairs it trails the raw signal, and in five it trails the policy with no touch at all.

My read is that the most valuable result here isn't any single score, it's that control. The same tactile signal often hurts when fed in raw, and helps reliably once it's turned into a pretrained representation. That suggests what's kept touch out of mainstream policies isn't just the sensors, it's the representation.

My read on the weaknesses: every real-robot result is on the two hands the authors used and on tasks they chose, the appendix scores those tasks step by step, and the FTP-1 adapters were trained by the authors too. There's no independent replication yet.

### Limitations

The authors list three limitations. First, real-robot and simulation experiments need a wider variety of robot hands.

Second, capacitive sensors are missing from pretraining, because the capacitive recordings available to them didn't meet their quality bar.

Third, coverage is focused on the fingertips, so contact on the palm and along the rest of the fingers is underrepresented.

### Future work

The authors want to extend coverage from the fingertips to the whole hand. They also plan to release inference code, a small set of sample recordings, and weights for the Tiny and Small models.

They make one more argument. Since data transfer across sensors, the sensor to scale with isn't the most capable one, but the one that's easiest to record with at scale, which is why two thirds of their data comes from gloves a person can wear.

My read is that if this holds, the bottleneck for touch data shifts from which sensor to buy to how to organize people to collect it. The thing to watch is whether, once the weights are out, other labs can reproduce these gains on their own hands.

That's today's Daily Frontier Papers. The paper is linked in the description.

