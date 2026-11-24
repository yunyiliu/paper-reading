# A Glimpse of Training VLMs as Foundation Models for Robotic Manipulation

**9B大模型零样本操控机械臂**

- 论文 Paper: [arXiv 2611.29338](https://arxiv.org/abs/2611.29338)
- 单位 Affiliation: 清华大学 · 腾讯混元
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-24
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1NsyfzTEbz)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇清华大学和腾讯混元的论文，标题翻过来是，初探把视觉语言模型训练成机器人操作的基础模型。

背景是这样的，GPT-6 Astra 在机器人控制上表现出很强的泛化能力，但它是怎么训练出来的，公开的信息很少。这篇论文想回答，这种能力能不能用一套讲得清楚、可以复现的方法做出来，他们的模型叫 nanoAstra。

### 问题

机器人操作的基础模型，目标是从大量经验里学到通用的本事，再用到没见过的物体、指令和环境上。

现在主流有两类。一类叫 VLA，视觉语言动作模型，把预训练的视觉和语言知识迁移到机器人控制上。另一类叫 WAM，世界动作模型，借助对未来画面的预测来学动作。

问题是，遇到没见过的任务配置，或者要把几个技能重新组合，现有模型还很吃力。作者的猜测是，原因之一在于机器人演示数据被组织成了什么样的监督信号。

### 以前的方法

看论文第一张图。左边两种，VLA 和 WAM，预测的都是动作块，也就是一串高频采样的控制步，每一步是一个向量，写着这一瞬间手臂怎么动，夹爪开还是关。

这种目标在物理上很明确，但它没有告诉模型，哪几步合起来是一个完整的动作，在任务里起什么作用。同一个动作，控制频率不同、执行速度不同、中途修正一下，写出来的数字序列就不一样。

作者认为，这样学，模型得从零碎的轨迹采样里自己拼出可复用的行为，容易依赖某段演示特有的执行方式。

### 核心想法

这篇论文的核心是两个设计。第一个叫语义动作建模，把密集的控制步，改写成一个个语义步骤。第二个叫 CP-SFT，能力保持的监督微调，让模型学会操作，同时尽量不丢掉原来的本事。

语义步骤长什么样，看图里右边。每一步由可复用、带参数的动作原语组成，比如沿 x 轴移动 5 厘米，或者夹爪张开，上面还配一句进度描述，比如，把杯子移到托盘上。

按作者的说法，进度描述告诉机器人要做成什么，动作原语告诉它具体怎么做。而且模型直接用自己原本的文字生成接口输出这些动作，不额外加一个动作预测头。

### 方法

先说数据。作者用了四个来源，ABC-130K、AgiBot、HiFi-UMI 和 HY-UMI，既有真机器人的演示，也有人拿着手持夹爪录下来的数据。

这些数据格式各不相同，处理流程分五步，过滤，统一坐标约定和相机，切分子任务并写描述，构造语义步骤，最后和通用视觉问答数据混在一起。

过滤这一步，会丢掉时间戳有断档、视频和状态对不上、数值在物理上不合理的演示，还会把超过半秒的空闲停顿遮掉。

对齐这一步，统一坐标系，位置用厘米，角度用度，夹爪开合映射到 0 到 1。有的数据集没公开相机外参，或者外参不准，作者就把机器人模型投影到画面里，重新标定相机，这张图左边是标定前，投影和手臂对不上，右边是标定后。

切分子任务时，作者用夹爪开合等操作事件找出一段段操作，再用 Qwen3.5-122B-A10B 给每段视频写描述，说清楚动作、物体和目标，这就是进度描述的来源。

子任务还是太粗，比如抓一个东西，其实包括先伸过去，再合上夹爪。所以作者在夹爪开合、停顿、急转弯和手腕大幅旋转的地方再切开，每一段的起点就是一个决策点，模型只在这些点上观察和预测。

每个语义步骤里，左右手臂各有一张命令列表，两只手臂同时执行，不动的那只就写 wait。

最后留下来的，是 5310.5 小时的操作数据，大约 81.8 万段演示，1490 万个训练样本。再混入 155 万条通用视觉问答样本，一共约 36.5 亿个训练 token，机器人数据占百分之 62.7，视觉问答占百分之 37.3。

再说 CP-SFT。训练机器人样本时，会加一段机器人专用的系统提示，写明是什么机器人、坐标怎么定义、能用哪些动作、回答用什么格式。

这张图解释了用意。有这段提示，模型就按固定格式输出进度和动作。没有这段提示，比如问它两个杯子哪个离镜头更近，它就像原来一样用自然语言回答。作者希望这样把改动尽量限制在机器人场景里，所有权重其实都会更新，所以这是一种软约束。

另外，动作里有大量数字。作者在交叉熵之外，加了一个专门针对数字的损失，用来提高动作数值的精度。它按位加权，越高位的数字权重越大，还单独监督正负号，这一项的系数设为 0.3。

训练从 Qwen3.5-9B 出发，全参数微调，在 56 张 H20 上训练一个 epoch。

### 实验和效果

主要评测在 RoboTwin 2.0 的随机化设置上，50 个任务，每个任务跑 10 次取平均。

关键是零样本。nanoAstra 只用真实世界的数据训练，没见过仿真轨迹，没见过 RoboTwin 的演示，也没有针对这个基准微调。模型闭环运行，每一轮看当前画面，输出当前进度和接下来要执行的几个步骤。

结果在表 1。nanoAstra-9B 成功率百分之 17.8，是表里最高的。最强的 VLA，LingBot-VLA-2.0 是 10.8，最强的通用 VLM 基线，开思考模式的 Qwen3.8-27B 也是 10.8，nanoAstra 比它们高 7 个百分点。

π0.5 只有 1.4，两个世界动作模型最多 0.2。作者说，世界动作模型这么低，说明在这个设置下迁移很难。

还有一个细节。表里的 Qwen 模型打开思考模式都会变好，从 3.2 到 4.0，提高到 8.2 到 10.8，但思考要多生成很多 token，动作就来得更慢。nanoAstra 不思考，直接输出动作，比不思考的 Qwen3.5-9B 高 14.6 个百分点，比它开思考也高 9.6 个百分点。

速度看表 2。在一张 RTX 4090 上，nanoAstra 平均 3.89 秒生成 3.5 条命令，这些命令要执行 2.87 秒。换到一张 H20 上，推理降到 1.69 秒，比对应动作的执行时间 2.66 秒还短。开思考的 Qwen3.5-9B 要 60.7 秒。当然，VLA 的绝对延迟低得多，π0.5 只要 101 毫秒。

推理比动作快，就可以边动边想。作者用了流式执行，机器人一边执行这一轮剩下的步骤，模型一边算下一轮，这张图画的就是这个过程。

效果在表 3。这里仿真器在推理时不暂停，像真机器人一样一直在跑，所以成功率比表 1 低。机器人干等的时间占比，从百分之 35 降到百分之 6.9，成功率从 11.2 到 12.9，没有下降。

作者还在另一个基准 RoboDojo Open 上测了，8 个任务，每个模型 400 次。nanoAstra 成功 19 次，也就是百分之 4.75，没训练的 Qwen3.5-9B 不思考只有百分之 0.25，表里其他模型最多百分之 0.75。

那原来的能力保住了多少。训练后，MMMU 从 58.6 降到 53.1，ERQA 从 47.3 降到 41.0，CV-Bench 二维从 82.2 到 81.6，三维从 92.6 到 87.6。作者的说法是大部分保住了，但确实有下降。

### 和以前方法的对比

和以前的方法比，区别主要在三处。VLA 和 WAM 预测的是密集控制步，nanoAstra 预测的是带进度说明的语义步骤。它不加动作头，直接用 VLM 的文字输出。它还刻意混入视觉问答数据，想保住原有能力。

数据流程上，论文提到 Qwen-RobotManip 也做了信号过滤、一致性检查和坐标对齐，但那是为训练 VLA 服务的。这篇的不同，是把轨迹组织成能执行的语义步骤，给 VLM 当监督。

我的判断是，这篇最有价值的地方，是把训练配方讲得很具体。一个 9B 的模型，只用真实数据训练，在仿真基准上零样本超过了表里所有的 VLA 和世界动作模型。

我的判断是，空缺也很明显。论文没有做消融，没有分开验证语义步骤和 CP-SFT 各自贡献多少，没有和 GPT-6 Astra 本身直接比较，也没有说明哪项结果是在真机器人上跑的。17.8 这个成功率，绝对值也还很低。

### 局限

论文没有单独的局限章节，但失败分析里给了两个典型例子，两次都在轮数用完之前做出了不少进展。

第一个是开微波炉。进度描述基本是对的，抓把手、拉开、松手，但每次拉门都是走直线，门只开了一部分，没达到成功标准。

第二个是把罐子放进篮子，再把篮子提起来。模型第二轮就把罐子放进去了，可后来进度又退回到已经完成的子任务，最后还转向一只当干扰物的鞋，篮子始终没提起来。

作者的总结是，动作参数和任务状态的估计误差，会在闭环执行里不断累积。至于通用能力的下降，作者认为更多样的视觉问答数据可能会缓解。

### 未来方向

作者提出的方向，是给微波炉门这类铰接物体，设计以物体为中心的动作原语，以及显式记录哪些子任务已经完成。

我的判断是，这条路值得跟。把机器人动作写成文字步骤，好处是能直接用上大模型的语言和视觉能力，代价是精细的连续动作不好表达，开微波炉那个失败，就是这个代价的一个例子。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Tsinghua University and Tencent Hunyuan, and it's called A Glimpse of Training VLMs as Foundation Models for Robotic Manipulation.

Here's the backdrop. GPT-6 Astra has shown strong generalization in robot control, but very little has been disclosed about how it was trained. This paper asks whether that kind of capability can be reproduced with a recipe anyone can follow. Their model is called nanoAstra.

### Problem

A robot manipulation foundation model is supposed to learn general skills from lots of experience, and then apply them to objects, instructions and environments it hasn't seen.

Today there are two main families. Vision-language-action models, or VLAs, carry pretrained visual and language knowledge over into robot control. World-action models, or WAMs, use predictions of future video frames to help learn actions.

The trouble is that these models still struggle with unfamiliar task setups and with combining skills in new ways. The authors' hypothesis is that part of the problem is how robot demonstrations are turned into training targets.

### Prior work

Look at the paper's first figure. On the left, VLAs and WAMs both predict action chunks: strings of densely sampled control steps, where each step is a vector saying how the arm moves and whether the gripper is open at that instant.

Those targets are physically precise, but they never tell the model which steps form one coherent motion, or what that motion is for. And the same behavior produces different number sequences depending on control frequency, execution speed, and small corrections along the way.

So, the authors argue, the model has to piece reusable behavior together from fragmented samples, and it may end up leaning on the quirks of particular demonstrations.

### Key idea

The paper's answer has two parts. The first is semantic action modeling: rewrite dense control steps as semantic steps. The second is CP-SFT, capability-preserving supervised fine-tuning, meant to teach manipulation without wiping out what the model already knows.

What does a semantic step look like? On the right of the figure. Each step is built from reusable, parameterized action primitives, like move five centimeters along x, or open the gripper, with a progress description on top, such as move the mug to the tray.

In the authors' framing, the progress description says what the robot should accomplish, and the primitives say how to do it. And the model outputs all of this through its normal text generation, with no separate action head bolted on.

### Method

Start with the data. The authors draw on four sources, ABC-130K, AgiBot, HiFi-UMI and HY-UMI, which mix real robot demonstrations with recordings from hand-held grippers.

Since these come in very different formats, the pipeline has five stages: filtering, aligning conventions and cameras, segmenting subtasks with captions, building semantic steps, and finally mixing in general vision-language data.

Filtering throws out demonstrations with timestamp gaps, video that's out of sync with the robot state, or physically implausible signals. It also masks idle pauses longer than half a second.

Alignment puts everything into common units: centimeters for position, degrees for rotation, and gripper opening from zero to one. Some datasets don't release camera extrinsics, or release inaccurate ones, so the authors project the robot model into the image and recalibrate the camera. In this figure, the left side is before, with the projection off the arms, and the right side is after.

For subtasks, gripper transitions and other manipulation events mark out segments, and Qwen3.5-122B-A10B captions each clip with the action, the object and the target. Those captions become the progress descriptions.

Subtasks are still too coarse. Grasping an object, for example, means reaching toward it and then closing the gripper. So the authors cut again at gripper transitions, pauses, sharp turns and large wrist rotations. The start of each piece is a decision point, and those are the only moments where the model observes and predicts.

Inside each semantic step, the left and right arms each get their own command list. The two arms run at the same time, and an idle arm just gets wait.

What survives is five thousand three hundred and ten point five hours of manipulation data, about eight hundred and eighteen thousand episodes, and fourteen point nine million training samples. Add one point five five million general vision-language examples, and the mix comes to about three point six five billion training tokens, sixty-two point seven percent robot data and thirty-seven point three percent vision-language.

Now CP-SFT. Every robot sample comes with a robot-specific system prompt that spells out the embodiment, the coordinate conventions, the allowed actions and the reply format.

This figure shows the intent. With that prompt, the model answers in a fixed form with progress and actions. Without it, say, which of the two mugs is closer to the camera, it answers in free-form language as before. The hope is to keep the changes confined to robot contexts. All the weights still get updated, so it's a soft constraint.

Actions are full of numbers, so on top of cross-entropy the authors add a loss aimed at numerical precision. It weights digits by decimal place, so the more significant digits count more, and it supervises the sign separately. That term gets a weight of zero point three.

Training starts from Qwen3.5-9B, with full fine-tuning for one epoch on fifty-six H20 GPUs.

### Results

The main test is RoboTwin 2.0's randomized setting: fifty tasks, ten runs per task, averaged.

The key word is zero-shot. nanoAstra is trained only on real-world data. It never sees simulated trajectories, never sees RoboTwin demonstrations, and gets no benchmark-specific fine-tuning. It runs in closed loop: each turn, it looks at the current observation and replies with its progress and the next few steps.

The results are in Table 1. nanoAstra-9B reaches seventeen point eight percent success, the best in the table. The strongest VLA, LingBot-VLA-2.0, gets ten point eight, and so does the strongest general VLM baseline, Qwen3.8-27B with thinking on. That's a seven-point lead.

Pi zero point five scores one point four, and the two world-action models manage at most zero point two. The authors read the world-action numbers as a sign of how hard transfer is under this protocol.

One more detail. Turning on thinking helps every Qwen model in the table, lifting them from the three-to-four percent range to between eight point two and ten point eight. But thinking means extra tokens before any action comes out. nanoAstra doesn't think at all, and still beats Qwen3.5-9B by fourteen point six points without thinking, and by nine point six points with it.

Speed is in Table 2. On a single RTX 4090, nanoAstra takes three point eight nine seconds to produce three and a half commands on average, and those commands take two point eight seven seconds to execute. On a single H20, inference drops to one point six nine seconds, shorter than the two point six six seconds of motion it produces. Qwen3.5-9B with thinking needs sixty point seven seconds. To be fair, VLAs are far faster in absolute terms: pi zero point five takes about a hundred milliseconds.

Once inference is faster than motion, the robot can think while it moves. The authors use streaming execution: the robot keeps running the remaining steps of the current reply while the model works on the next one. That's what this figure shows.

Table 3 has the payoff. Here the simulator keeps running during inference, like a real robot, which is why success is lower than in Table 1. The share of time the robot sits waiting falls from thirty-five percent to six point nine percent, and success goes from eleven point two to twelve point nine, so nothing is lost.

They also test on a second benchmark, RoboDojo Open: eight tasks, four hundred episodes per model. nanoAstra succeeds nineteen times, four point seven five percent. The untuned Qwen3.5-9B without thinking gets zero point two five percent, and nothing else in the table goes above zero point seven five.

So how much general ability survives? After training, MMMU drops from fifty-eight point six to fifty-three point one, ERQA from forty-seven point three to forty-one, CV-Bench 2D from eighty-two point two to eighty-one point six, and CV-Bench 3D from ninety-two point six to eighty-seven point six. The authors call it substantial but incomplete retention.

### Comparison

Compared with earlier approaches, three things differ. VLAs and WAMs predict dense control steps, while nanoAstra predicts semantic steps with progress descriptions. There's no action head; it's just the VLM's text output. And vision-language data is mixed in on purpose, to protect existing abilities.

On the data side, the paper points to Qwen-RobotManip, which also does signal filtering, consistency checks and coordinate alignment, but for training VLAs. The difference here is organizing trajectories into executable semantic steps as supervision for a VLM.

My read is that the real value is how concrete the recipe is. A nine-billion-parameter model, trained only on real data, beats every VLA and world-action model in the table, zero-shot, on a simulation benchmark.

My read is also that the gaps are obvious. There are no ablations separating what semantic steps and CP-SFT each contribute, no direct comparison with GPT-6 Astra itself, and nothing in the paper says which results, if any, come from a real robot. And seventeen point eight percent is still a low success rate in absolute terms.

### Limitations

There's no dedicated limitations section, but the failure analysis shows two typical cases, both of which make real progress before running out of turns.

The first is opening a microwave. The predicted progress is mostly right: grasp the handle, pull it open, let go. But every pull is executed as straight-line motion, so the door only opens partway and never reaches the success threshold.

The second task is to drop a can into a basket and then lift the basket. The model places the can on turn two, but its progress then slips back to the finished subtask, and later drifts to a distractor shoe. The basket is never lifted.

The authors' takeaway is that errors in motion parameters and in tracking the task state pile up during closed-loop execution. As for the drop in general ability, they think broader and more diverse vision-language data could reduce it.

### Future work

Their suggested next steps are object-centric primitives for articulated objects like that microwave door, and explicit tracking of which subtasks are already done.

My read is that this direction is worth following. Writing robot actions as text steps lets the model use its language and vision abilities directly. The cost is that fine continuous motion is hard to express, and the microwave failure is one example of that cost.

That's today's Daily Frontier Papers. The paper is linked in the description.

