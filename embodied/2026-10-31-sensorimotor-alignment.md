# Sensorimotor Alignment: A Platonic Proxy for Embodied Policy Success

**不跑机器人先猜成功率**

- 论文 Paper: [arXiv 2610.31670](https://arxiv.org/abs/2610.31670)
- 单位 Affiliation: 莱斯大学、英伟达
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-10-31
- 视频 Videos: [YouTube 中文](https://youtu.be/YEh5-MlIoYw) · [YouTube English](https://youtu.be/hcoYwpV2GWA) · [B站](https://www.bilibili.com/video/BV1EcxC6QEDB)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇莱斯大学和英伟达的论文，题目叫 Sensorimotor Alignment，感觉运动对齐，副标题是，一个预测具身策略能不能成功的柏拉图式代理指标。

它想回答一个很实际的问题，能不能不让机器人真的动起来，就提前判断一个机器人模型好不好用。

### 问题

先说背景。现在的机器人基础模型，大多建在大的视觉语言模型、视频模型，或者世界动作模型之上。比如 VLA，视觉语言动作模型，就是从视觉语言模型微调出来、能直接输出机器人动作的模型，世界动作模型，英文缩写 WAM，则是建在视频生成模型上。

但要评估这些模型，最靠谱的办法还是闭环测试，也就是让模型在仿真里或者真机上一遍遍执行任务，看成功率。作者说，这样做又慢、又有噪声，结论还取决于用的是哪个 benchmark。

所以大家想要一个便宜的离线指标，不用跑机器人，先给候选模型排个序，决定先测哪个。

### 以前的方法

现成的做法，是借用感知这一侧的分数。比如视觉语言模型看视觉问答，也就是 VQA 的成绩，视频模型看生成画面的质量，像 FID、SSIM、PSNR 这些指标。

问题在于，这些分数衡量的是看得懂、画得好，而不是这个表示有没有为动作组织好。两个场景看起来很不一样，可能需要差不多的动作，两个场景看着很像，也可能需要完全不同的轨迹。

论文引用的已有结果也说明了这一点。VLM4VLA 这项工作发现，VQA 分数和后来 VLA 的表现相关性很弱，甚至是负的，另一项研究发现，预测视频的质量和下游表现，在不同任务上并不一致。

### 核心想法

作者的思路来自一个叫柏拉图表示假说的观点，英文是 Platonic Representation Hypothesis。它认为，不同模态的模型越做越强，在各自的表示空间里会越来越趋同，逼近同一个关于现实世界的统计模型，比如图像和文字，就是同一个世界的两种投影。

这篇论文问的是，机器人的运动轨迹，能不能看成同一个世界的又一种投影。如果能，那越强的具身模型，它对图像和指令的表示，结构上就应该越接近真实运动轨迹的结构。

换句话说，在感知上挨得近的两段数据，在动作上也应该挨得近。这张图是整篇论文的概念图，左边是这个思路，右边是在 12 个 VLA 上的结果，后面细讲。

### 方法

具体怎么算。作者把这个指标叫 Sensorimotor Alignment，论文里写作 S 的平方，下面就叫它 S 方。它不需要任何训练，只需要一个机器人数据集，加上要评估的模型。

参考数据集用的是 DROID 和 BridgeDataV2，两个公开的机器人示范数据集。从里面抽出很多段 episode，每段随机取一帧画面，配上这段的任务指令。

感知这一侧，用被评估模型自己的视觉编码器和文本编码器，把这帧图像和这句指令编码，再平均池化成一个向量。运动这一侧，取这一帧之后若干步的末端执行器轨迹，包括位置、姿态和夹爪状态。

然后在两个空间里分别找近邻。感知空间用余弦距离，运动空间用动态时间规整，也就是 DTW，它能把快慢不同、但形状相同的动作对齐了再比。

对每一段数据，找出它在感知空间里最近的 10 个邻居，再找出它在运动空间里最近的 10 个邻居，看两组邻居重合了多少。重合越多，说明看起来像的场景，动作也真的像，对齐就越好。

这张图是个例子。查询的这一段是把罐子放进柜子，绿色的勾是两边都出现的邻居，红色的叉是只在感知空间里出现的邻居。

实际报告的数值，用的是这个重合度的一个中心化、归一化的版本，叫 CKNNA，邻居数默认取 10。

### 实验和效果

实验分四组模型。第一组是 12 个微调好的 VLA，把它们的 S 方和在 SimplerEnv 仿真基准上的成功率放在一起，皮尔逊相关系数是 0.717，作者给的 bootstrap 百分之九十五置信区间，是 0.436 到 0.912。

第二组更实用。拿 8 个还没做任何机器人微调的原始视觉语言模型，先算 S 方，再和它们后来被微调成 VLA 之后的成功率对比，相关系数是 0.681。论文的结论是，在花大成本微调之前，就可以用 S 方先挑底座模型。

第三组是 6 个世界动作模型，在双臂数据集 ALOHA 上算 S 方，和 RoboTwin 2.0 上的成功率对比，相关系数 0.718。不过这一组只有 6 个点，单侧 p 值是 0.054，严格说没有过 0.05 这条显著性线。

第四组是真机。真机成功率受太多因素影响，所以作者直接用了 MolmoBot 论文里的一次统一评测，5 个策略、40 个任务，再拿这些策略公开的权重算 S 方，相关系数是 0.831。

这里有个细节要说清楚。真机这一组用的轨迹长度，也就是往后看多少步，取的是 110 步，论文自己标注这是探索性的设定，如果用事先定好的 7 步，相关系数只有 0.429，并不显著。

作者还做了稳健性检查。参考数据集从 DROID 换成 BridgeDataV2，两个是不同的机械臂，相关系数都在 0.7 左右，轨迹长度从短到长换一遍，相关系数几乎都在 0.5 以上。

还有一个反向对照。给双臂的世界动作模型，故意用单臂数据集 DROID 来算，相关系数从 0.999 掉到 0.246，作者认为，这说明这个指标确实对机械臂的构型敏感，而不是对什么都给高分，不过这组对照只有 3 个模型。

论文后半部分，用 S 方去解释几个以前只在闭环结果里看到的现象。第一个，大家早就发现 VLA 经常不太听指令，主要照着画面行事。作者把视觉特征和文本特征分开算，在 12 个微调后的 VLA 里，视觉特征的相关系数是 0.694，文本特征只有 0.237，不显著。

有意思的是，还没微调的原始视觉语言模型正好反过来，文本特征的相关是 0.501，高于视觉特征的 0.425，不过这两个数单独都不显著。作者据此推断，微调之后，VLA 主要靠看到的东西来产生动作，这解释了它为什么不太听指令。

第二个现象，以前有研究报告，微调 VLA 时把视觉编码器冻住，效果会明显变差。作者拿两个只差这一点的模型对比，冻住之后，S 方下降百分之四十一，SimplerEnv 成功率下降百分之四十，从百分之五十降到百分之三十点二一。

第三和第四个现象在世界动作模型上。世界动作模型一般要一步步去噪，生成未来的视频，以前的研究发现，跳过生成视频、直接解码动作，成功率损失不大，作者测到，S 方从最早的去噪步就已经很高，之后几乎不变，三个模型里最多只变了大约百分之六，正好解释了这一点。另外，在 30 个 block 的视频骨干网络里，位于中间的第 14 个 block，S 方最高，三个模型都是这样。

### 和以前方法的对比

那 S 方比老指标强在哪。同样 8 个底座模型，VQA 平均分和最终成功率的相关系数是负 0.154，基本没关系，而 S 方是 0.681。

在 3 个能生成未来视频的世界动作模型上，SSIM 和 PSNR 跟成功率的相关都不显著，FID 甚至方向是反的，画面质量更差的模型成功率反而更高。S 方的相关系数是 0.999，不过只有 3 个点。

和其他表示研究比，以前也有人问过，什么样的视觉编码器对机器人有用，但这些工作只看视觉编码器，也都没有把机器人实际执行的轨迹放进衡量里。S 方用同一套算法，覆盖了 VLA、原始视觉语言模型、世界动作模型，也包括前人评测过的那一批冻结视觉编码器。

我的判断是，这篇的价值在于给了一个便宜的初筛工具，只要一个公开数据集和模型的前向计算，就能在昂贵的闭环测试之前，先把排在后面的候选往后放。

我的判断是，也要看到它的证据还不算厚。正文的四组，每组只有 5 到 12 个模型，视频质量那组只有 3 个，世界动作模型那组没过显著线，真机那组换回事先定好的轨迹长度，相关就不显著了，附录里去掉 StarVLA 这一类模型之后，剩下 7 个 VLA 的相关也不再显著。

### 局限

作者自己列了三条局限。

第一，S 方和成功率之间是相关，不是因果。作者还没有证明，在训练时直接优化这个对齐，就能提高成功率。

第二，S 方相近的模型，成功率也可能差很多，所以它最适合用来初筛，最终选模型还是要靠闭环测试。

第三，覆盖的模型家族、benchmark 和训练配方都有限，每组模型不多，真机也只有一套评测，能不能推广还要再测。

附录里还有一个值得一提的边界。在冻结视觉编码器的实验里，如果策略靠任务奖励做强化学习，S 方和成功率就没有正相关了，作者的解释是，只有当学习方法直接依赖这个表示的时候，S 方才和成功率相关。

### 未来方向

作者在结论里说，目前 S 方只被当作离线的选模型和诊断工具，但这个相关性提示，可以直接把感觉运动对齐当成目标，来优化具身模型。代码已经在 GitHub 上公开。

我的判断是，最值得看的下一步，是有没有人真的把 S 方放进训练，如果成功率跟着涨，它就从一个相关指标变成了能用的抓手。另外，论文显示参考数据要和目标机器人的机械臂构型匹配，双臂就要用双臂数据，这会限制它在新构型机器人上的使用。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Rice University and NVIDIA, and it's called Sensorimotor Alignment, with the subtitle "A Platonic Proxy for Embodied Policy Success."

The question it asks is a practical one: can you tell whether a robot model is any good without actually running it on a robot?

### Problem

Some context first. Robot foundation models are increasingly built on top of large vision-language models, video models, or world action models. A VLA, a vision-language-action model, is fine-tuned from a vision-language model to output robot actions, while a world action model, or WAM, is built on a video generator instead.

The trouble is evaluation. The most reliable test is still closed-loop rollout, letting the model actually attempt tasks in simulation or on real hardware and counting successes, and the authors point out that this is slow, noisy, and depends on which benchmark you pick.

So what people want is a cheap offline score, something that ranks candidate models before you spend time on rollouts and tells you which to test first.

### Prior work

The obvious option is to borrow perception-side scores: visual question answering, or VQA, results for vision-language models, and pixel-level video quality, like FID, SSIM, and PSNR, for video models.

The catch is that these measure whether a model sees well or renders well, not whether its representation is organized for action. Two scenes can look very different but call for similar motions, and two scenes can look almost identical but need completely different trajectories.

Earlier results that the paper cites back this up. A study called VLM4VLA found weak or even negative correlations between VQA scores and the performance of the resulting VLA, and another found that predicted video quality tracks downstream performance inconsistently across tasks.

### Key idea

The authors start from the Platonic Representation Hypothesis, the idea that as models in different modalities get stronger, their representations converge toward a shared statistical model of reality, with images and text as two projections of the same world.

This paper asks whether robot motion can be treated as one more projection of that world. If it can, then a more capable embodied model should represent images and instructions in a way whose structure looks more like the structure of real motion trajectories.

Put simply, episodes that are close together in what the model perceives should also be close together in how the robot moves. This figure is the paper's overview, with the idea on the left and the result across twelve VLAs on the right, which we'll get to.

### Method

So how is it computed? The metric is called Sensorimotor Alignment, written S-squared in the paper, and it needs no training at all, just a robot dataset and the model you want to score.

The reference datasets are DROID and BridgeDataV2, two public robot demonstration datasets. From many episodes, the authors take one random frame each and pair it with that episode's task instruction.

On the sensory side, the model's own vision and text encoders embed the frame and the instruction, and the output tokens are average-pooled into one vector. On the motor side, they take the end-effector trajectory over the next several steps after that frame: position, orientation, and gripper state.

Then they look for neighbors in each space. Sensory space uses cosine distance, and motor space uses dynamic time warping, or DTW, which lines up motions that have the same shape but run at different speeds.

For every episode, they find its ten nearest neighbors in sensory space and its ten nearest neighbors in motor space, and measure how much the two sets overlap. More overlap means that scenes that look alike really do get similar motions, which is stronger alignment.

Here's an example from the paper. The query episode is placing cans into a cabinet, green checks are neighbors found in both sets, and red crosses are neighbors that show up only in the sensory set.

The number they actually report is a centered, normalized version of that overlap called CKNNA, with ten neighbors by default.

### Results

The experiments cover four groups of models. First, twelve fine-tuned VLAs: their S-squared against their success rate on the SimplerEnv simulation benchmark gives a Pearson correlation of zero point seven one seven, with a bootstrap ninety-five percent confidence interval of zero point four three six to zero point nine one two.

The second group is the more useful one. The authors take eight raw vision-language models, before any robot fine-tuning, compute S-squared, and compare it with the success of the VLAs later fine-tuned from them, getting a correlation of zero point six eight one. Their conclusion is that S-squared can pick promising backbones before you pay for fine-tuning.

Third, six world action models, scored on the two-arm ALOHA dataset and compared with their success on RoboTwin two point zero, give a correlation of zero point seven one eight. With only six points, though, the one-tailed p-value is zero point zero five four, just short of the usual zero point zero five threshold.

Fourth, real robots. Because real-world success depends on so many factors, the authors reuse one shared evaluation reported in the MolmoBot paper, five policies on forty tasks, and compute S-squared on those policies' public checkpoints, getting a correlation of zero point eight three one.

There's a detail worth flagging here. That real-robot result uses a trajectory horizon, meaning how many steps ahead the motion is taken, of one hundred ten steps, which the paper itself labels exploratory, and with the prespecified seven steps the correlation is only zero point four two nine and not significant.

The authors also ran robustness checks. Switching the reference data from DROID to BridgeDataV2, which use different robot arms, keeps the correlation around zero point seven, and across short and long trajectory horizons it stays above zero point five almost everywhere.

There's also a negative control. Scoring the two-arm world action models on the single-arm DROID dataset on purpose drops the correlation from zero point nine nine nine to zero point two four six, which the authors read as the metric responding to arm configuration rather than handing out high scores to everything, though this control has only three models.

The second half of the paper uses S-squared to explain effects that had only been seen in closed-loop results. The first is that VLAs often ignore the instruction and act mostly on what they see. Splitting the features, the twelve fine-tuned VLAs show a correlation of zero point six nine four for vision features but only zero point two three seven, not significant, for text features.

Interestingly, raw vision-language models show the opposite pattern, with text features at zero point five zero one above vision features at zero point four two five, though neither is significant on its own. The authors infer that after fine-tuning, VLAs produce motion mainly from what they see, which would explain their weak instruction following.

The second effect: earlier work reported that freezing the vision encoder during VLA fine-tuning hurts performance. Comparing two models that differ only in that choice, freezing cuts S-squared by forty-one percent and SimplerEnv success by forty percent, from fifty percent down to thirty point two one percent.

The third and fourth effects are about world action models. These models usually denoise step by step to generate future video, earlier work found that skipping that video generation and decoding actions directly costs little success, and the authors find that S-squared is already high at the earliest denoising step and barely moves afterward, by about six percent at most across three models, which fits that finding. And in the thirty-block video backbone, block fourteen, right in the middle, has the highest S-squared in all three models.

### Comparison

So how does it compare with the older scores? On the same eight backbones, average VQA score correlates with final success at minus zero point one five four, essentially nothing, while S-squared reaches zero point six eight one.

On the three world action models that generate future video, SSIM and PSNR show no significant correlation with success, and FID even points the wrong way, with worse video quality going with higher success. S-squared correlates at zero point nine nine nine, but that's from only three points.

Compared with other representation studies, earlier work asked which vision encoders help robots, but looked only at vision encoders and never brought the trajectories the robot executes into the measurement. S-squared uses one method across VLAs, raw vision-language models, world action models, and the frozen vision encoders that earlier work had benchmarked.

My read is that the value here is a cheap first-pass filter: with one public dataset and forward passes through each model, you can push weaker candidates down the list before spending on closed-loop tests.

My read is also that the evidence is still thin. Each of the four main groups has only five to twelve models, the video-quality comparison has three, the world action model result misses significance, the real-robot result loses significance at the prespecified horizon, and in the appendix, dropping the StarVLA family leaves seven VLAs whose correlation is no longer significant.

### Limitations

The authors list three limitations themselves.

First, the link between S-squared and success is a correlation, not causation. They have not shown that optimizing for this alignment during training actually raises success rates.

Second, models with similar S-squared can still have quite different success rates, so it's best used for early screening, with closed-loop testing still the final step.

Third, the study covers a limited set of model families, benchmarks, and training recipes, with few models per comparison and a single real-robot setting, so generalization still has to be tested.

One more boundary from the appendix is worth mentioning. With frozen vision encoders, when the policy is trained by reinforcement learning from task rewards, there's no positive correlation, and the authors' explanation is that S-squared tracks success only when the learning method relies directly on that representation.

### Future work

In the conclusion, the authors say they use S-squared only offline, for model selection and diagnosis, but that the correlation suggests optimizing embodied models directly for sensorimotor alignment. The code is public on GitHub.

My read is that the next thing to watch is whether anyone actually puts S-squared into training, because if success rises with it, it goes from a correlate to something you can steer with. Also, the paper shows the reference data has to match the target robot's arm configuration, two arms for two-arm robots, which may limit its use on robots with new body layouts.

That's today's Daily Frontier Papers. The paper is linked in the description.

