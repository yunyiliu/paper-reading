# Human-to-Robot Transfer via Bridging Representation and Action Space in VLAs

**机器人跟着人学新任务**

- 论文 Paper: [arXiv 2611.04391](https://arxiv.org/abs/2611.04391)
- 单位 Affiliation: KAIST、RLWRLD
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-06
- 视频 Videos: [YouTube English](https://youtu.be/CyXmq7srlbo) · [B站](https://www.bilibili.com/video/BV1ko4z6DESV)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇韩国科学技术院 KAIST 和 RLWRLD 的论文，方法叫 H2RB，意思是人到机器人的桥。

它想解决的问题是，能不能让人自己动手做，拍下来，就把一个新任务教给预训练好的机器人模型，而不用在这台机器人上重新采集演示。

### 问题

先说背景。这里说的模型叫 VLA，视觉语言动作模型，它看摄像头画面，听一句指令，直接输出机械臂和手的动作。预训练好的 VLA 是个不错的起点，但要部署到具体的机器人和环境里，通常还得在这台机器人上采集演示，要遥操作，要反复复位场景，又贵又慢，很难规模化。

人的演示就便宜多了，一个轻量的相机就能拍。问题在于，人手和机械手差别很大，而 VLA 主要是用机器人数据预训练的，人类数据很难直接拿来用。

### 以前的方法

以前的做法，主要是把人和机器人的动作放进同一个动作空间。一种是从视频前后帧的变化里，推断一个潜在动作，另一种是重建人手在三维空间里的运动轨迹。

作者认为，这只解决了一半。动作格式统一了，但模型看到的画面还是不一样，一边是人手，一边是机械手。而且就算是同一个动作，人和机器人做起来的节奏和轨迹也不同。

### 核心想法

H2RB 的思路是，两头都搭桥。感知这一头，用手的几何形状，也就是手腕和五个指尖的位置，把人和机器人的画面特征锚到一起。动作这一头，让人和机器人共享一套潜在动作，但各自保留一个解码器。

而且整个过程，不需要人和机器人一一配对的演示，也不需要用图像合成，把视频里的人手换成机械手。

### 方法

第一步是统一动作格式。人类视频先用一个叫 HaWoR 的工具，重建手腕运动和手指姿态，再转换到机器人的坐标系，映射到机械手上。人和机器人的动作都写成同一种形式，手腕位置、手腕旋转，加上手指关节角度。真正部署的时候，再用逆运动学把手腕位姿换算成机械臂的动作。

第二步是一个动作分词器，看这张图的左半边。一个共享的编码器，把人或者机器人的一段动作，压缩成一串潜在动作，然后人有人的解码器，机器人有机器人的解码器。

为什么要两个解码器。因为格式统一之后，人和机器人的动作风格还是不一样，硬塞进同一个映射，两种动作风格会被混成一种。为了让两边的潜在动作落在相近的区域，训练时会拉近两边潜在表示的平均值，再加上方差和协方差的约束，防止表示塌缩。分词器训完就冻结，VLA 要学的，是预测这串潜在动作。

第三步是几何锚定，看图的右半边。在 VLM 输出的特征上，加一个小的残差适配器，再接一个只在训练时用的读出头，让它从特征里预测出当前的手腕位姿和五个指尖位置。人类数据的几何信息来自重建，机器人的来自它自己的关节状态。

因为每一帧画面都自带这样一个目标，所以不需要人和机器人配对，也不需要时间上对齐。最终的训练目标，是预测潜在动作的 flow matching 损失，加上 0.2 倍的几何损失。

部署时，模型根据机器人看到的画面，生成潜在动作，再用人类那一侧的解码器，解码成机器人的指令。附录里作者对比过，用人类解码器，物体迁移得分 51.8，用机器人解码器只有 32.7。

### 实验和效果

实验全部在真机上做。平台是 OpenArm 双臂人形机器人，配 Inspire 灵巧手，迁移实验只用右臂右手。所有策略都从同一个预训练模型 RLDX-1-IMG 出发，对比的基线用一样的数据和训练预算，包括直接把人和机器人数据混在一起训，以及基于 HaWoR 和基于 Ego-Pi 的共享动作空间方法。

测了两种迁移。一种是物体迁移，机器人原本会抓放瓶子、杯子、玩偶和零食，新的盒子、球和毛巾，只给人类演示，每样 100 条。另一种是技能迁移，打开和关上空气炸锅，这次用的机器人数据里没有这个动作，人类演示各 50 条。打分是 0 到 100 的进度分，够到物体算 0.3，抓稳算 0.6，拿起来移动算 0.8，完成才算满分。

先看物体迁移，表 1。只用机器人数据，新物体平均 12.0 分，人和机器人数据直接混训，反而掉到 8.0，共享动作空间基线里最好的 Ego-Pi 是 27.2，H2RB 是 51.8。机器人原本会的玩偶抓放，H2RB 是 49.0，比只用机器人数据的 55.0 低一些。

把物体摆到训练里没出现过的位置，H2RB 是 45.3，Ego-Pi 是 21.3。

技能迁移更能说明问题，看表 2。直接混训，还有两个共享动作空间的基线，打开和关上两项全是 0 分，一点都没学会。H2RB 打开是 50.7，关上是 74.7，平均 62.7，原来的抓盒子任务也保持在 81.0。

作者还用一万四千多条人和机器人的混合数据，先做了一轮大规模异构训练，再做同样的微调。这样物体迁移从 51.8 提到 57.2，技能迁移从 62.7 提到 81.7。

换一台机器人 RB-Y1，做包裹翻面再推上传送带的任务，下游只用机器人数据微调，测 50 次。前期只用机器人数据训练，成功率 64%，加入人类数据是 72%，再加上几何锚定是 78%。

另外，如果把语言指令在动作模块的每一层都重新注入，技能迁移还能从 62.7 提到 83.3。

### 和以前方法的对比

那为什么共享动作空间不够。论文的失败案例很直观。Ego-Pi 基线会摆出根本抓不住东西的手型，还会在开炸锅的任务里，做出机器人熟悉的抓放动作，等于退回了老本行。

消融实验在表 3 左边。没有动作分词器，物体迁移 34.7，技能迁移 0。加上分词器，变成 43.3 和 49.7。再加上拉近两边潜在表示的约束，变成 51.8 和 62.7，两边潜在表示的差距从 3.81 降到 1.53。

几何锚定单独的作用，在附录表 12。在 Ego-Pi 基线上只加几何锚定，物体迁移从 27.2 提到 34.7，再加上分词器，才到 51.8。

还有一个有意思的发现，在表 3 右边。作者另外采了 100 条机器人遥操作数据，训了一个纯机器人策略做对比。从人类演示迁移过来的策略，手腕平均速度是每秒 7.36 厘米，纯机器人策略是 2.52 厘米，快了将近 3 倍，任务得分相近，62.7 对 65.3。作者的解释是，人类那一侧有自己的解码器，人做事更快的节奏被保留了下来。

我的判断是，这篇最有价值的地方，是指出了统一动作格式不等于能迁移。技能迁移里，几个共享动作基线全是 0，而 H2RB 能学会，这个对比很有说服力。

### 局限

论文没有专门写局限，下面这些，是从实验数据和设置里能看到的。

首先，进度分不等于完成率。看附录表 11，物体迁移里，真正把任务做完的只有 12.2%，技能迁移是 30%。加了大规模训练之后，分别是 21.1% 和 56.7%。

其次，评测规模不大。每个条件只训练了一个策略，物体迁移测 90 次，技能迁移测 30 次。RB-Y1 上从 72% 到 78%，在 50 次里只差 3 次。

再有，人和机器人的数据是在同一个环境里采的，相机视角也大致对齐了人类演示，迁移实验只在一个平台上做。作者也写明，潜在空间里这些对齐和防塌缩的约束，并不能保证每一个人类动作都对应到正确的机器人动作。

### 未来方向

作者的期望是，让人类演示成为一种可规模化的监督来源，减少对目标机器人数据的依赖，让预训练 VLA 靠人的演示学会新行为。

我的判断是，几何锚定只需要手腕和指尖位置，这让人类视频这种便宜的数据，更容易接到机器人模型上，值得关注它在更多机器人和更杂乱场景下是否还成立。

我的判断是，离拍段视频就教会机器人还有距离。物体迁移的完成率只有一成多，而且人类演示和机器人在同一个场景里，下一步要看换了场景、换了人，效果还剩多少。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from KAIST and RLWRLD, and the method is called H2RB, short for Human-To-Robot Bridge.

The question behind it: can a person simply do a task on camera, and have a pretrained robot model pick it up, without collecting new demonstrations on that robot?

### Problem

Some background. The models here are VLAs, vision-language-action models. They look at a camera image, read an instruction, and output arm and hand actions directly. A pretrained VLA is a good starting point, but deploying it on a specific robot in a specific place usually still means recording demonstrations on that robot. That takes teleoperation and constant scene resets. It's expensive, slow, and hard to scale.

Human demonstrations are much cheaper. A lightweight camera is enough. The catch is that a human hand and a robot hand are very different, and a VLA that was pretrained mostly on robot data has a hard time using human data directly.

### Prior work

The usual fix has been to put human and robot actions into one shared action space. Either you infer a latent action from how video frames change, or you reconstruct the human hand's motion in three-dimensional space.

The authors argue that only solves half the problem. The action format matches, but the images still don't: one shows a human hand, the other a robot hand. And even for the same action, humans and robots move with different timing and trajectories.

### Key idea

H2RB builds a bridge at both ends. On the perception side, it uses hand geometry, the wrist and the five fingertip positions, to anchor human and robot image features to the same reference. On the action side, humans and robots share one set of latent actions, but each keeps its own decoder.

And none of this needs paired human and robot demonstrations, or image synthesis that paints a robot hand over the human one.

### Method

Step one is a common action format. Human videos go through a hand reconstruction tool called HaWoR, which recovers wrist motion and finger pose. That gets converted into the robot's coordinate frame and retargeted to the robot hand. Both humans and robots are then described the same way: wrist position, wrist rotation, and finger joint angles. At deployment, inverse kinematics turns the wrist pose into arm motion.

Step two is an action tokenizer, the left half of this figure. A shared encoder compresses a chunk of human or robot motion into a sequence of latent actions. Then humans get their own decoder, and robots get theirs.

Why two decoders? Because even in a shared format, humans and robots move differently, and forcing both through one mapping would blend two motion styles into one. To keep the two sides' latents in the same neighborhood, training pulls their averages together and adds variance and covariance terms to stop the representation from collapsing. Then the tokenizer is frozen, and the VLA learns to predict those latent actions.

Step three is geometry anchoring, the right half of the figure. A small residual adapter sits on the vision-language features, and a readout head, used only during training, has to predict the current wrist pose and the five fingertip positions from them. For human data the geometry comes from the reconstruction; for the robot, from its own joint state.

Since every frame carries its own target, you don't need pairs or time alignment between human and robot data. The full training objective is a flow matching loss on the latent actions, plus zero point two times the geometry loss.

At deployment, the model generates latent actions from the robot's camera view, and the human-side decoder turns them into robot commands. In the appendix, the authors check this choice: the human decoder scores fifty-one point eight on object transfer, the robot decoder only thirty-two point seven.

### Results

Everything runs on a real robot: an OpenArm bimanual humanoid with Inspire dexterous hands, using only the right arm and hand for the transfer tests. Every policy starts from the same pretrained model, RLDX-1-IMG. Baselines get the same data and training budget. They include plain co-training on mixed human and robot data, and two shared-action-space methods built on HaWoR and on Ego-Pi.

There are two kinds of transfer. Object transfer: the robot already knows how to pick and place a bottle, a cup, a doll and a snack, and the new objects, a box, a ball and a towel, come only from human demonstrations, a hundred each. Skill transfer: opening and closing an air fryer, a motion absent from the robot demonstrations used here, with fifty human demonstrations for each. Scoring is a progress score from zero to a hundred. Reaching the object counts zero point three, a solid grasp zero point six, lifting and moving it zero point eight, and only finishing gets full marks.

Object transfer first, Table 1. Trained on robot data only, the new objects average twelve. Plain co-training actually drops to eight. The best shared-action baseline, Ego-Pi, gets twenty-seven point two, and H2RB gets fifty-one point eight. On the doll task the robot already knew, H2RB scores forty-nine, a bit below the fifty-five of the robot-only policy.

With objects placed where training never put them, H2RB scores forty-five point three, against twenty-one point three for Ego-Pi.

Skill transfer is the more telling result, Table 2. Plain co-training and both shared-action baselines score zero on opening and on closing. They learn nothing. H2RB scores fifty point seven on opening and seventy-four point seven on closing, sixty-two point seven on average, and keeps eighty-one on the box task it already knew.

The authors also ran a large heterogeneous training stage first, on about fourteen and a half thousand human and robot episodes, then the same fine-tuning. Object transfer rises from fifty-one point eight to fifty-seven point two, and skill transfer from sixty-two point seven to eighty-one point seven.

On a different robot, the RB-Y1, the task is to flip parcels label-up and push them onto a conveyor, with robot-only fine-tuning and fifty trials. Pre-training on robot data alone gives sixty-four percent success. Adding human data gives seventy-two. Adding geometry anchoring on top gives seventy-eight.

And if the language instruction is re-injected into every layer of the action module, skill transfer climbs further, from sixty-two point seven to eighty-three point three.

### Comparison

So why isn't a shared action space enough? The failure cases make it concrete. The Ego-Pi baseline forms hand poses that can't grasp anything. And on the air-fryer task, it falls back on the pick-and-place motion it knows from robot data.

The ablation is on the left of Table 3. With no action tokenizer, object transfer is thirty-four point seven and skill transfer zero. Adding the tokenizer gives forty-three point three and forty-nine point seven. Adding the term that pulls the two latent distributions together gives fifty-one point eight and sixty-two point seven, and the gap between them shrinks from three point eight one to one point five three.

Geometry anchoring on its own is in appendix Table 12. Adding just the geometry anchoring to the Ego-Pi baseline lifts object transfer from twenty-seven point two to thirty-four point seven. It takes the tokenizer as well to reach fifty-one point eight.

There's one more interesting result, on the right of Table 3. The authors recorded a hundred extra robot teleoperation episodes and trained a robot-only policy for comparison. The policy transferred from human demonstrations moves its wrist at seven point three six centimeters per second, versus two point five two for the robot-only one. That's almost three times faster, with similar scores, sixty-two point seven against sixty-five point three. The authors' explanation is that the human side keeps its own decoder, so the faster human pace survives the transfer.

My read is that the main contribution is showing that a shared action format is not the same as transfer. In skill transfer, every shared-action baseline scores zero while H2RB learns the task, and that contrast is convincing.

### Limitations

The paper has no limitations section, so these come from its own numbers and setup.

First, a progress score is not a completion rate. Appendix Table 11 shows that in object transfer, only twelve point two percent of trials actually finish the task, and thirty percent in skill transfer. With the large-scale training stage, that becomes twenty-one point one and fifty-six point seven percent.

Second, the evaluation is small. Each condition is one trained policy, tested ninety times on object transfer and thirty times on skill transfer. On the RB-Y1, going from seventy-two to seventy-eight percent is a difference of three trials out of fifty.

Third, the human and robot data were collected in the same environment, with the camera roughly matching the human viewpoint, and the transfer experiments use a single platform. The authors also state that these latent alignment and anti-collapse terms do not guarantee that each human action maps to the right robot action.

### Future work

The authors' hope is that human demonstrations become a scalable source of supervision, cutting the need for target-robot data and letting pretrained VLAs learn new behaviors from people.

My read is that geometry anchoring is attractive because it only needs wrist and fingertip positions, which makes cheap human video easier to plug into a robot model. The thing to watch is whether it holds on more robots and in messier scenes.

My read is also that we're still some way from teaching a robot with a quick video. Object transfer finishes the task in barely more than one trial in ten, and the human demos were recorded in the robot's own scene. The next test is how much survives a new scene and a new demonstrator.

That's today's Daily Frontier Papers. The paper is linked in the description.

