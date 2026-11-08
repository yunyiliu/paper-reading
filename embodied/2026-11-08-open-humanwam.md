# Open-HumanWAM: An Open Recipe for Scaling Human and Robot Experience for Embodied Foundation Models

**人类视频越多机器人越能干**

- 论文 Paper: [arXiv 2611.06817](https://arxiv.org/abs/2611.06817)
- 单位 Affiliation: 北京大学 · 新加坡国立大学 · MIT · UCSB · 英伟达
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-08
- 视频 Videos: [YouTube 中文](https://youtu.be/B7SdOAqsS8M) · [YouTube English](https://youtu.be/CN9OcehX8vo) · [B站](https://www.bilibili.com/video/BV1YRWJ66EXZ)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇北京大学、新加坡国立大学、MIT、加州大学圣塔芭芭拉分校和英伟达合作的论文，叫 Open-HumanWAM，讲的是怎么用人类和机器人的经验，扩展训练具身基础模型。

它要回答的问题是，拿大量人戴头戴相机拍的第一视角视频做预训练，机器人到底能变强多少，又该怎么训。

### 问题

先说背景。机器人基础模型要学会通用的操作技能，需要大量经验，但这些经验主要靠人遥控机器人一条一条地采，成本很高。

人的第一视角视频，配上手部动作和文字标注，量大又便宜。问题在于，什么样的预训练配方，能让机器人随着数据增长持续变强，论文说这件事还没研究清楚。

### 以前的方法

已经有一些工作。EgoScale 表明，人手动作预训练能提升灵巧操作。HumanScale 发现，后训练之后的动作预测误差，随预训练小时数呈对数线性下降。Dyna-2 把人类视频预训练扩到了一百万小时。

不过，EgoScale 和 HumanScale 量的是验证集上的损失，也就是预测误差，不是机器人真去做任务的成功率。论文也提到，在另一些设置下，扩数据的泛化收益有限，甚至不一致。

### 核心想法

这篇的思路，是把规律从损失搬到成功率上。同一个模型、同一套后训练和仿真闭环评测，把纯人类视频预训练从 100 小时扩到 5 万小时，看机器人成功率怎么变。

发现有三层。第一，陌生场景里的成功率，随预训练小时数呈对数线性增长。第二，任务和场景的多样性跟着时长一起扩，曲线更陡。第三，人类数据和机器人数据混着训，模型不微调，也能直接做一部分没见过的任务。

这张图是全文概览，上面一行是纯人类预训练，下面一行加入了机器人数据。

### 方法

先说数据。作者从五个公开数据集拼出 5 万小时第一视角视频，大头是 HumanNet 里约 4 万小时的网络视频，一共大约 3590 万个片段，每段都带手部动作和文字描述。

清洗有三道关。手要在画面里，手和相机的轨迹不能乱跳，长时间不动的部分剪掉。原始的 10 万多小时，筛完剩下 5 万小时。

关键一步是动作对齐。各个数据集记录手部动作的坐标系不一样，作者统一成双手末端执行器的 16 维动作，每只手是三维位置、四维朝向和一维张合程度，并对齐了相机轴、手腕轴和手指张开的尺度。末端执行器，就是机械臂最前面抓东西的那一截。

模型叫 HumanWAM-Vanilla。WAM 是 world action model，世界动作模型，它不只输出动作，还同时预测接下来的画面。它在 Wan2.2 视频生成模型上，加了一个和视频主干一样深的动作专家。

评测是这样的。预训练之后，在双臂仿真基准 RoboTwin 2.0 上，每个任务用 50 条干净场景的演示做后训练，再在三种条件下测，和训练一样的干净场景，换了背景和桌面纹理的场景，以及背景、杂物、光照、桌高全部随机的场景。

第二个模型叫 HumanWAM-Goal，把人类数据和 1.46 万小时机器人数据混着训练。除了语言指令，还能给它一张目标图片，说明场面最后该是什么样。动作空间扩到 49 维，加上了关节和底盘，能控制带轮子的机器人。

### 实验和效果

先看扩展规律。默认设置下，从 100 小时扩到 1 万小时，干净场景成功率从 71.2% 到 80.0%，换场景从 25.2% 到 58.1%，全随机从 8.4% 到 28.7%。不做人类预训练的话，后两项只有 16.2% 和 2.1%。

作者用对数直线拟合，R 方在 0.88 到 0.97 之间。按拟合，数据每多十倍，换场景成功率大约多 15.9 个百分点，全随机多 10.2 个百分点。

换一种注意力设计，让动作能看到预测出的未来画面，一直扩到 5 万小时。干净场景始终在 87% 到 89% 之间，几乎不动，换场景却从 49.1% 涨到 73.2%，全随机从 21.5% 涨到 42.9%。也就是说，人类数据主要提升的是泛化。

再看多样性。同样的时长，一组随着时长扩任务和场景，另一组任务和场景固定，只加轨迹。前者的拟合斜率在三种条件下都更陡，换场景和全随机分别高出 4.73 和 4.50。

样本效率也变好了。每个任务只给 10 条机器人演示时，干净场景成功率从没有人类预训练的 34.5%，涨到 500 小时的 44.2% 和 5000 小时的 50.1%。换场景条件下，5000 小时的模型只用 5 条演示，就超过了没有人类预训练、用 50 条演示的模型。

还有两组对照。做了动作对齐，换场景成功率从 43.2% 提到 52.2%。预训练时给大约一半的视频加上人手动作目标，训练到 2 万步时，换场景从只预测视频的 13.0% 提到 34.8%。

和同样时长的机器人数据比，在 500 到 5000 小时之间，人类数据在三种条件下都更好，1000 小时人类数据训出的模型，在三种条件下都超过了 5000 小时机器人数据训出的模型。

最终的 HumanWAM-Vanilla 用全部 5 万小时人类数据预训练。在 RoboTwin 2.0 上后训练之后，干净场景 92.08%，是表里最高的，随机场景 68.88%，平均 80.48%。在另一个基准 RoboDojo 上，平均成功率 14.17%。

真机上，一台双臂机器人做叠纸杯、叠毛巾、把积木收进抽屉、整理桌面四个任务，每个 10 次。换了桌布和灯光以后，没有人类预训练的模型 40 次只成功 1 次，HumanWAM-Vanilla 平均 67.5%。

HumanWAM-Goal 的真机零样本测试，用的是带轮子的 Astribot S1，20 个任务各 10 次。只用机器人数据，平均成功率 20.0%，加 1.59 万小时人类数据到 26.0%，加 4.59 万小时到 30.5%，至少成功过一次的任务，从 10 个增加到 16 个。

作者还展示了一种具身的上下文学习。参数不动，只换目标图片，机器人就会把东西放到不同的位置。后训练过的模型，目标图里是人手而不是机械臂，也能照着摆。这部分是案例展示，没有统计成功率。

### 和以前方法的对比

先看 HumanWAM-Vanilla。在 RoboTwin 2.0 的对比表里，它平均 80.48%，超过了所有 VLA 基线，比如 π0.5 是 58.35%。但在随机场景上，它落后于最强的世界动作模型 GO-3 Nano Preview，68.88% 对 81.48%，作者认为部分原因是它用的是比较基础的架构。

在 RoboDojo 上，它 14.17% 的平均成功率超过了表里所有世界动作模型，接近 Galaxea G0.5 的 14.60%，但低于 DM0.5 的 17.96%，作者也承认它不如最强的模型。

零样本对比差距更明显。不做任何微调，直接在 RoboTwin 上测，π0.5 平均 1.06%，Pelican-VLA 0.5 是 4.70%，HumanWAM-Goal 只用机器人数据是 11.48%，加上人类数据是 13.94%。表里很多现有模型，在这个设置下根本跑不起来。

后训练时再加上数据增强和配方改进，HumanWAM-Goal 在 RoboTwin 2.0 上达到干净场景 92.70%、随机场景 84.60%，平均 88.65%，比排行榜上之前领先的 GO-3 Nano Preview 高 3.64 个百分点。要注意，基线数字取自排行榜，作者的结果是自己评测的。

我的判断是，这篇最有价值的不是 88.65% 这个榜单成绩，而是在同一套协议下，用闭环成功率把人类视频的价值量清楚了，作者也称数据、代码和模型都已开源，别人可以复现。

### 局限

论文没有单独的局限一节，下面几条来自它自己报告的实验。

第一，大部分扩展规律是在仿真里、一种机器人形态上测的，主要是 RoboTwin 的 50 个任务，真机实验每个任务只做 10 次。

第二，真机零样本的绝对成功率还不高，加了人类数据平均也只有 30.5%。按数据比例采样时，难任务停在 19% 左右，把每批数据里机器人和人类样本改成一比一，难任务才升到 27%。不过这样一来，20 个任务的总平均是 29.0%，比 30.5% 略低。

第三，作者自己说，纯人类预训练的模型没接触过机器人的身体，不能直接部署，每个任务都要拿演示来微调。

第四，RoboTwin 后训练时，八成的目标图片是用 GPT Image 2.5 生成的，审核只保证画面和指令一致，不保证物理上可行。

### 未来方向

作者把这套开放的数据、代码和模型，定位成继续研究人类视频扩展规律的基础。按他们的拟合，作者认为继续加人类数据，泛化还会继续提升。

他们也把带目标图片的模型，定位成多模态 Agent 的动作执行器，上游的规划器或者用户给一张目标图，模型负责把它变成动作。

我的判断是，对数线性也意味着，数据每多十倍，才多涨固定的几个点。论文的拟合只到 5 万小时，再往上是否还成立、划不划算，还要更大规模的实验来回答。

我的判断是，做机器人的人最该记住两点，只堆时长不如连多样性一起扩，混合训练时想做好难任务，机器人数据在每批里的比例不能被人类数据淹没。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is a collaboration between Peking University, the National University of Singapore, MIT, UC Santa Barbara and NVIDIA. It's called Open-HumanWAM, an open recipe for scaling human and robot experience for embodied foundation models.

The question is simple. If you pretrain a robot model on huge amounts of first-person video of people using their hands, how much better does the robot actually get, and what recipe makes more data keep paying off?

### Problem

Some background. Robot foundation models need a lot of manipulation experience to learn skills that carry over across tasks and environments. Today that experience mostly comes from people teleoperating robots, one demonstration at a time, and that's expensive.

First-person human video, paired with hand-motion and language annotations, is a cheap and abundant record of people handling objects in all kinds of settings. The open question, as the paper puts it, is which pretraining recipe makes robot performance keep improving as that data grows.

### Prior work

There's prior work here. EgoScale showed that large-scale human-motion pretraining helps dexterous manipulation. HumanScale found that, after robot post-training, action-prediction loss on held-out real-robot trajectories falls log-linearly with pretraining hours. And Dyna-2 pushed human-video pretraining to one million hours.

But the EgoScale and HumanScale trends are measured in validation loss, a prediction error, not in how often a robot actually completes a task in closed loop. The paper also notes that other setups have reported limited or inconsistent generalization gains from scaling.

### Key idea

The idea here is to move the scaling law from loss to success rate. Keep one model, one robot post-training protocol and one closed-loop simulation benchmark, scale pure human-video pretraining from one hundred to fifty thousand hours, and watch the robot's success rate.

There are three findings. First, success in unseen scenes grows log-linearly with pretraining hours. Second, hours aren't the only lever: growing task and scene diversity along with hours gives a steeper curve. Third, mixing human and robot data lets a model do some unseen tasks without any fine-tuning.

This figure is the overview. The top row is the fifty-thousand-hour human corpus and pure human pretraining. The bottom row adds fourteen point six thousand hours of robot data. On the right are zero-shot results on a real robot.

### Method

Start with the data. The authors combine five public datasets into fifty thousand hours of first-person video. The biggest piece is about forty thousand hours of internet video from HumanNet. In total that's about thirty-six million clips, each with hand motion and a text description.

Curation has three filters. The hands have to be in view, hand and camera trajectories can't jump around, and long idle stretches are cut. About one hundred and four thousand raw hours become fifty thousand.

The key step is action alignment. Each dataset records hand motion in its own coordinate conventions. The authors map everything into one sixteen-dimensional action for two end effectors: per hand, a three-D position, a four-number orientation and one value for how open the hand is. They also align camera axes, wrist axes and the scale of hand opening. An end effector is just the business end of a robot arm, the part that grasps.

The model is HumanWAM-Vanilla. WAM stands for world action model: it doesn't just output actions, it also predicts the upcoming video frames. It starts from the Wan two point two video generation model and adds an action expert as deep as the video backbone, with the two exchanging information through attention.

Here's the evaluation protocol. After pretraining, each model is post-trained on RoboTwin two point zero, a two-arm simulation benchmark, using fifty clean-scene demonstrations per task. It's then tested three ways: the same clean scenes as training, scenes with new backgrounds and table textures, and fully randomized scenes with clutter, lighting and table-height changes.

The second model is HumanWAM-Goal. It's trained on human data mixed with fourteen point six thousand hours of robot data, and besides a language instruction it can take a goal image showing what the scene should look like at the end. Its action space grows to forty-nine dimensions, adding joints and a mobile base, so it can drive wheeled robots too.

### Results

First, the scaling trend. With the default attention setup, going from one hundred to ten thousand hours lifts clean-scene success from seventy-one point two to eighty percent, new-scene success from twenty-five point two to fifty-eight point one, and fully random from eight point four to twenty-eight point seven. With no human pretraining, those last two are just sixteen point two and two point one.

The authors fit a straight line against log hours, with R-squared between zero point eight eight and zero point nine seven. By that fit, every tenfold increase in data adds roughly fifteen point nine percentage points in new scenes and ten point two in fully random ones.

Then they switch to a different attention design, where actions can see the predicted future frames, and scale all the way to fifty thousand hours. Clean-scene success stays between eighty-seven and eighty-nine percent, barely moving. But new-scene success climbs from forty-nine point one to seventy-three point two, and fully random from twenty-one point five to forty-two point nine. In other words, human data mainly buys generalization.

Next, diversity. At equal hours, one series grows its tasks and scenes along with its duration; the other keeps tasks and scenes fixed and only adds trajectories. The first has a steeper fitted slope in all three settings, by four point seven three points in new scenes and four point five in fully random.

Sample efficiency improves too. With only ten robot demonstrations per task, clean-scene success goes from thirty-four point five with no human pretraining, to forty-four point two with five hundred hours, and fifty point one with five thousand. In new scenes, the five-thousand-hour model with just five demonstrations beats the no-pretraining model with fifty.

Two more controlled comparisons. Aligning actions across sources raises new-scene success from forty-three point two to fifty-two point two. And adding hand-motion targets to about half the video during pretraining raises new-scene success, at twenty thousand steps, from thirteen percent for video prediction alone to thirty-four point eight.

What about human data versus the same number of hours of robot data? From five hundred to five thousand hours, human data wins in all three settings. A model pretrained on one thousand hours of human video beats one pretrained on five thousand hours of robot data, in all three.

The final HumanWAM-Vanilla model is pretrained on all fifty thousand hours of human data. After post-training on RoboTwin two point zero it scores ninety-two point zero eight percent in clean scenes, the highest in the table, sixty-eight point eight eight in randomized scenes, and eighty point four eight on average. On a second benchmark, RoboDojo, its mean success rate is fourteen point one seven percent.

On a real two-arm robot, they test four tasks: stacking paper cups, folding a towel, putting a block in a drawer, and tidying a desk, ten trials each. With a new tablecloth and lighting, the model without human pretraining succeeds once in forty trials, two point five percent on average. HumanWAM-Vanilla averages sixty-seven point five.

Now HumanWAM-Goal, zero-shot on real hardware. On an Astribot S1, a wheeled robot, across twenty tasks with ten trials each, the robot-data-only model averages twenty percent success. Adding fifteen point nine thousand hours of human data gives twenty-six, and forty-five point nine thousand hours gives thirty point five. The number of tasks with at least one success goes from ten to sixteen.

There's one more behavior the authors call embodied in-context learning. Same model, weights frozen, and just by changing the goal image the robot puts objects in different places. After post-training, it even follows goal images that show human hands instead of a robot gripper. This part is shown through examples; the paper doesn't report success rates for it.

### Comparison

Start with HumanWAM-Vanilla. In the RoboTwin two point zero table, its eighty point four eight average beats every VLA baseline; pi zero point five, for example, is at fifty-eight point three five. But in randomized scenes it trails the strongest world action model: GO-3 Nano Preview hits eighty-one point four eight, versus its sixty-eight point eight eight. The authors attribute part of that to its deliberately basic architecture.

On RoboDojo, its fourteen point one seven mean beats every world action model in the table and comes close to Galaxea G zero point five at fourteen point six, but it's below DM zero point five at seventeen point nine six. The authors acknowledge it still trails the strongest models.

The zero-shot comparison is more striking. With no fine-tuning at all on RoboTwin, pi zero point five averages one point zero six percent, Pelican-VLA zero point five gets four point seven, HumanWAM-Goal with robot data alone gets eleven point four eight, and with human data added, thirteen point nine four. Many existing models in the table can't even be run in this setting.

After post-training, with added data augmentation and a refined recipe, HumanWAM-Goal reaches ninety-two point seven in clean scenes and eighty-four point six in randomized ones, an eighty-eight point six five average. That's three point six four points above the previous leader on the leaderboard, GO-3 Nano Preview. One caveat: the baseline numbers come from the leaderboard, while the authors' own numbers come from their own evaluation.

My read is that the leaderboard number isn't the most valuable part. What matters is that the value of human video is measured in closed-loop success rates under one fixed protocol, and the authors say the data, code and models are all released, so others can reproduce it.

### Limitations

The paper has no dedicated limitations section, so these points come from its own reported setup and results.

First, most of the scaling results are in simulation, on a single robot embodiment, mainly the fifty RoboTwin tasks. The real-robot experiments use ten trials per task.

Second, absolute zero-shot success on real robots is still low: thirty point five percent on average even with human data. Under data-proportional sampling, hard tasks stall around nineteen percent. Only when the authors force a one-to-one ratio of robot and human samples in every batch do hard tasks rise to twenty-seven. With that balancing, though, the average over all twenty tasks is twenty-nine percent, slightly below thirty point five.

Third, the authors themselves say a model pretrained only on human data has never seen a robot's body or control, so it can't be deployed directly and needs demonstrations for every target task.

Fourth, during RoboTwin post-training, eighty percent of the goal images are generated with GPT Image two point five. The automated review checks that images match the instruction, not that they're physically feasible.

### Future work

In their conclusion, the authors position the open data, code and models as a foundation for further research on human-video scaling. Based on their fits, they expect generalization to keep improving as human data grows.

They also pitch the goal-image model as an action executor for multimodal agents: an upstream planner or a user supplies a goal image, and the model turns it into motion.

My read is that a log-linear law also means each tenfold increase in data buys only a fixed number of points. The fits stop at fifty thousand hours. Whether the trend holds beyond that, and whether it's worth the cost, needs larger experiments.

My read is that robotics practitioners should take away two things: growing diversity along with hours beats piling on hours alone, and when mixing data for hard tasks, the share of robot data in each batch shouldn't get drowned out by human data.

That's today's Daily Frontier Papers. The paper is linked in the description.

