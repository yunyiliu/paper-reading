# Learning to Think and Explore in 3D World

**只给一张图让模型自己去看**

- 论文 Paper: [arXiv 2610.21783](https://arxiv.org/abs/2610.21783)
- 单位 Affiliation: 加州大学圣地亚哥分校、佐治亚理工学院
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-19
- 视频 Videos: [YouTube 中文](https://youtu.be/JYerM0_V_To) · [YouTube English](https://youtu.be/IKpnq17sqa8) · [B站](https://www.bilibili.com/video/BV1sAJd67E98)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇加州大学圣地亚哥分校和佐治亚理工学院的论文，题目是 Learning to Think and Explore in 3D World，学会在 3D 世界里思考和探索，他们训练出来的模型叫 Act3D。

简单说，以前让模型回答关于一个房间的问题，是把整个房间的扫描、视频或者多张照片，一次性交给它。这篇反过来，模型一开始只看到一张图，看不清，就自己移动相机，换个地方再看，看够了再回答。

### 问题

先说要解决的问题。你站在房间里某个位置，很多东西是看不到的，可能被挡住了，也可能在你身后，这叫部分可观测。

这时候，模型不光要会根据眼前的证据推理，还得决定怎么动，才能拿到缺的那部分证据。作者想回答的就是，视觉语言模型，能不能把主动找证据，学成推理过程的一部分。

### 以前的方法

以前的 3D 视觉语言模型，输入大致有三类。第一类直接给 3D 表示，比如点云和网格。第二类给视频，再加上深度、相机位姿这类信息。第三类只给视频或者多视角图片。

它们有一个共同前提，回答问题需要的视觉证据，已经提前准备好交给模型了，模型只是被动地看。

也有工作开始让模型主动选视角，比如 Chain-of-View，不用训练，直接通过离散的相机动作选视角。还有 Think3D，给模型配上 3D 重建工具，再用强化学习优化工具的使用。

### 核心想法

这篇的想法是，把 3D 场景从模型的输入里拿出来，变成一个可以交互的环境。模型每次只看一张第一人称的图，就像人一样。

这张图下面就是一个例子。问题是，走廊尽头是什么。模型先右转两次，再往右走 1.2 米，发现离墙太近，又转了一下，再后退 1.5 米，最后看到走廊尽头是一扇黑色的门，回答 door。

更关键的是训练方式。强化学习阶段，奖励主要看最后答案对不对，不告诉模型该去哪个视角，该做什么动作。怎么探索，它得自己从答对答错里学出来。

### 方法

具体怎么做。作者把它写成一个部分可观测的马尔可夫决策过程，简单说，场景是固定的，对模型隐藏，模型每一步能控制的，只有相机的位置和朝向。

动作都是相对的，前后、左右、上下平移，再加左转右转、抬头低头。默认一步，前后是 1 米，其他平移是 0.6 米，转向是 30 度，也可以自己指定距离，一次最多走 8 步。

有个细节，相机不做碰撞检测，可以穿过墙和家具。每做一个动作，环境就渲染出新视角的图片，模型先在 think 里推理，然后要么再做一个动作，要么给出答案。

训练分两步。第一步是监督微调，用 GPT-5.4 在环境里生成推理、动作、观察交替的轨迹，只保留答对的、操作有效的，最后得到 5712 条轨迹，覆盖 830 个场景。

作者强调，这些轨迹只是教会模型怎么跟环境打交道，不是规定一条标准的探索路线。

第二步是强化学习，用的是 GRPO。同一个问题、同一个起点，模型跑 8 次，因为动作不同，每次看到的东西也不同，再拿每次的得分和这一组的平均分比较。

奖励由几部分组成。最主要的是答案对不对，用一个大模型裁判判断语义是否一致，再加一项，看和标准答案是否完全一致。输出格式出错，会扣分。

还有一项交互奖励，设计得比较巧。只有答对了才给。如果模型确实把问题问的那个物体看进了画面，就直接拿满这一项，否则按交互次数累加，上限一样。作者的说法是，奖励的是证据，而不是多看几眼。

具体权重是，裁判那一项 0.3，完全一致那一项 0.2，交互奖励每次 0.05，最多算 12 次，也就是最多 0.6 分。

### 实验和效果

实验设置。底座模型是 Qwen3.5-9B。混合数据的版本，监督微调用 5712 条轨迹，强化学习用了 51543 个问题，来自 ScanQA、MP3D，以及在 ScanNet 和 ARKitScenes 上生成的问答。

评测时，所有模型都从一张 640 乘 480 的渲染图出发，最多交互 12 次，主动交互的对照模型，用的是同一套接口和次数上限。一共测了六个 3D 基准。

结果在表 1。Act3D 在六个基准上的平均准确率是 53.2%，比底座 Qwen3.5-9B 的 31.2% 高 22 个百分点，也比同样交互设定下的 Gemini 3.6 Flash 高 1 个百分点，Gemini 是 52.2%。

和只做了监督微调的版本比，强化学习把平均分从 48.6% 提到 53.2%，六个基准里五个提高。提升最大的是 ARKit，多了 14.1 个百分点，SPBench-MV 多了 8.1 个百分点。

泛化方面，作者又单独只用 ScanQA 训练了一个版本。ScanQA 本身提高 6.1 个百分点，另外五个基准的平均分，从 49.5% 到 51.6%，提高 2.1 个百分点，其中 MP3D 提高 6 个点，SQA3D 提高 3.8 个点。

模型的行为也变了。混合数据训练的过程中，平均每个回合的交互次数，从大约 3.3 次涨到 8.3 次。

这张图最有意思。作者在 ScanQA 验证集上，根据答案物体离起点多远、在哪个方向，估计每个问题需要几步才能走近它。只要一步的问题，监督微调加强化学习，一共只比底座高 0.6 个点。需要 7 步以上的，高 16.3 个点，其中 10.9 个点来自强化学习。

论文的解读是，越需要去别处找证据的问题，强化学习的收益越明显。

论文还展示了几种自己学出来的探索方式。目标被挡住时，会主动换视角去找。目标已经在眼前，就简单确认一下。要比较几个物体的位置时，会从不同角度多看几眼。

### 和以前方法的对比

消融实验里有几个对比值得一提。初始画面只给一张图，ScanQA 准确率是 41.0%，给鸟瞰图加四张图是 41.4%，几乎一样，鸟瞰图加一张图，反而降到 36.7%。

观察方式上，用渲染出来的图，比去数据集里找最近的真实照片，在 13 个检查点上平均高 10.6 个百分点。原因是真实照片的位置和角度，和模型要求的视角对不上。

动作表示上，相对动作，比如向左走、向右转，比直接预测绝对坐标要好，41.4% 对 36.7%。

不做监督微调，直接强化学习也能学，训练到 470 步，ScanQA 达到 41.4%，但平均每回合只交互 2 次。先做监督微调的版本，从 39.5% 提到 42.7%，交互 8.7 次。作者也提醒，这两组的底座和训练量并不一样。他们的看法是，监督微调帮模型更快适应，但不一定是必须的。

再看交互奖励。加上这一项，交互次数从大约 5 到 6 次涨到大约 9 到 10 次，三个没训练过的基准都提高了，SPBench-MV 多了 6.4 个点，但三个训练过的基准反而略降，降了 0.3 到 2.3 个点。

和别的模型比，有一点要看清。Act3D 赢的是六个基准的平均分，单看每个基准，它并不都是第一。比如 ScanQA 和 SQA3D 上，被动看完整输入的 Ross3D 是 50.7 和 62.6，比 Act3D 的 42.1 和 49.0 高，SPBench-SI 上，Gemini 3.6 Flash 是 44.4，Act3D 是 37.4。另外，被动方法用的是各自的输入设定，不是同一套交互接口。

我的判断是，53.2 对 52.2，比 Gemini 高的这 1 个点，差距很小，表 1 也没有给出误差范围，不能说明明显强过 Gemini。真正有说服力的，是小模型从 31.2 到 53.2 的提升，以及探索需求越大、收益越大这条规律。

### 局限

论文没有单独写局限这一节，下面几条是从论文的设定和实验里整理出来的。

第一，环境是重建出来的室内 3D 场景，相机不做碰撞检测，可以穿过墙和家具，论文里也没有真实机器人上的实验。

第二，渲染图的画质有限，作者自己也说，渲染图的视觉保真度不如真实照片，只是动作和画面对得上更重要。

第三，论文的主要结果都基于同一个底座模型，就是 Qwen3.5-9B，评测时答案对不对，也是靠另一个大模型 Qwen3-30B-A3B 来判。

第四，交互越多不等于越准。加了交互奖励之后，训练领域的分数略有下降，作者也承认，探索得更多，不一定准确率更高。

### 未来方向

论文的结论是，基于结果的强化学习，不仅能教模型怎么根据证据推理，也能教它主动去拿需要的证据。论文没有专门写后续计划。

我的判断是，这套思路最自然的下一步，是接到真正会动的机器人上。那时就得面对碰撞、噪声和真实相机，在渲染环境里学到的探索习惯能不能迁移过去，还是未知数。

我的判断是，这篇真正的价值在于换了问题的设定。以前是先给模型看全，再问它，现在是只给一眼，让它自己决定去看哪里。对做 3D 理解和具身智能的人来说，这个设定更接近真实世界。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from UC San Diego and Georgia Tech. It's called Learning to Think and Explore in 3D World, and the model it trains is called Act3D.

Here's the short version. The usual way to ask a model about a room is to hand it everything up front: a scan, a video, a stack of photos. This paper flips that around. The model starts with a single image, and if it can't see enough, it moves the camera, looks again, and only answers once it has what it needs.

### Problem

Start with the problem. Stand anywhere in a room and a lot of it is hidden from you, behind furniture or behind your back. That's partial observability.

So a model has to do two things. It has to reason about what's in front of it, and it has to decide how to move to get the evidence it's missing. The authors' question is whether a vision-language model can learn that second skill as part of reasoning itself.

### Prior work

Earlier 3D vision-language models take roughly three kinds of input. Some get a 3D representation directly, like a point cloud or a mesh. Some get video plus extra signals such as depth and camera pose. And some get only video or multi-view images.

They share one assumption. The visual evidence needed to answer the question has already been collected and handed over, and the model just looks at it.

A few recent methods do let the model choose views. Chain-of-View picks viewpoints through discrete camera actions with no training at all. Think3D gives the model 3D reconstruction tools and uses reinforcement learning to improve how it uses them.

### Key idea

The idea here is to take the 3D scene out of the model's input and turn it into an environment the model can act in. At each step the model sees one first-person image, the way a person would.

The bottom half of this figure shows an episode. The question is, what's at the end of the hallway? The model turns right twice, moves right one point two meters, finds it's too close to the wall, turns a bit, backs up one point five meters, and finally sees a black door at the end of the hallway. It answers, door.

The training is the more important part. During reinforcement learning, the reward depends mainly on whether the final answer is right. Nobody tells the model which viewpoints to visit or which moves to make. It has to work out how to explore from whether its answers come out right or wrong.

### Method

Formally, the authors set this up as a partially observable Markov decision process. In plain terms, the scene is fixed and hidden from the model, and the only thing the model controls at each step is where the camera is and which way it points.

All actions are relative: move forward or back, left or right, up or down, plus turn left or right and tilt up or down. A default step is one meter forward or back, zero point six meters for the other moves, and thirty degrees for turns. The model can ask for a bigger step, up to eight units at once.

One detail matters later: the camera has no collision checking, so it can pass straight through walls and furniture. After every action the environment renders the new view, and the model thinks, then either moves again or answers.

Training has two stages. First comes supervised fine-tuning on trajectories generated by GPT-5.4 inside this environment, alternating reasoning, actions and observations. Only trajectories with correct answers and valid moves were kept, which left five thousand seven hundred twelve trajectories across eight hundred thirty scenes.

The authors stress that these demonstrations only teach the model how to interact with the environment. They are not meant to prescribe one correct way to explore.

The second stage is reinforcement learning with GRPO. For the same question and the same starting view, the model runs eight episodes. Different moves mean different things get seen, and each episode's score is compared with the group average.

The reward has a few parts. The main one is answer correctness, judged by a language model that checks whether the answer means the same thing as the ground truth, plus a bonus for an exact match. Malformed output is penalized.

Then there's an interaction reward, and it's cleverly designed. It's only paid when the answer is correct. If the model actually brought the asked-about object into view, it gets the full amount at once. Otherwise it earns a little per move, up to the same cap. In the authors' words, the point is to pay for evidence, not for looking.

The weights are zero point three for the judge, zero point two for exact match, and zero point zero five per interaction, counted up to twelve moves, so at most zero point six.

### Results

The setup. The base model is Qwen3.5-9B. In the mixed-data version, supervised fine-tuning uses those five thousand seven hundred twelve trajectories, and reinforcement learning uses fifty-one thousand five hundred forty-three questions from ScanQA, MP3D, and generated question-answer pairs over ScanNet and ARKitScenes.

At test time, every model starts from a single six hundred forty by four hundred eighty render and may interact up to twelve times. The active baselines use the same interface and the same budget. There are six 3D benchmarks.

The headline is in Table 1. Act3D averages fifty-three point two percent across the six benchmarks. That's twenty-two points above its base model, Qwen3.5-9B, at thirty-one point two, and one point above Gemini 3.6 Flash under the same interactive protocol, at fifty-two point two.

Against its own supervised starting point, reinforcement learning lifts the average from forty-eight point six to fifty-three point two, improving five of the six benchmarks. The biggest gains are on ARKit, up fourteen point one points, and SPBench-MV, up eight point one.

For generalization, the authors also trained a version on ScanQA alone. ScanQA itself goes up six point one points, and the average over the other five benchmarks rises from forty-nine point five to fifty-one point six, a gain of two point one. MP3D gains six points and SQA3D three point eight.

The model's behavior changes too. Over mixed-data training, the average number of moves per episode climbs from about three point three to eight point three.

This is the figure I find most interesting. On ScanQA validation questions, the authors estimate, from where the answer object sits relative to the starting view, how many moves it should take to reach it. On questions that need one move, fine-tuning plus reinforcement learning beats the base model by only zero point six points. On questions that need seven or more, the gain is sixteen point three, and ten point nine of that comes from reinforcement learning.

The paper's reading is that the harder it is to find the evidence, the more reinforcement learning helps.

The paper also shows exploration habits that emerged on their own. When the target is hidden, the model changes viewpoint to find it. When the target is already visible, it does a quick check. When it has to compare several objects, it gathers views from different angles.

### Comparison

A few ablations are worth knowing. Starting from a single image gives forty-one point zero percent on ScanQA. A bird's-eye view plus four images gives forty-one point four, about the same, and a bird's-eye view plus one image drops to thirty-six point seven.

On how observations are produced, rendering the requested view beats retrieving the nearest real photo from the dataset by ten point six points on average across thirteen checkpoints. The real photos are taken from positions and angles that don't match what the model asked for.

On the action format, relative moves like step left or turn right beat predicting absolute coordinates, forty-one point four to thirty-six point seven.

Skipping supervised fine-tuning and going straight to reinforcement learning still works. After four hundred seventy steps, ScanQA reaches forty-one point four, but with only about two moves per episode. The fine-tuned version goes from thirty-nine point five to forty-two point seven with eight point seven moves. The authors point out that the two setups differ in backbone and training budget. Their reading is that fine-tuning speeds up adaptation but may not be strictly required.

Then the interaction reward. With it, moves per episode go from about five or six to about nine or ten. All three held-out benchmarks improve, SPBench-MV by six point four points, but the three training domains slip by zero point three to two point three points.

Now the comparison with other models, read carefully. Act3D wins on the six-benchmark average, not on every benchmark. On ScanQA and SQA3D, Ross3D, which passively sees full inputs, scores fifty point seven and sixty-two point six, above Act3D's forty-two point one and forty-nine point zero. On SPBench-SI, Gemini 3.6 Flash scores forty-four point four against Act3D's thirty-seven point four. And the passive methods use their own input settings, not the shared interactive interface.

My read is that a one-point lead over Gemini, fifty-three point two to fifty-two point two, is small, and Table 1 reports no error bars, so it doesn't show Act3D is clearly ahead. What's convincing is the jump from thirty-one point two to fifty-three point two for a small model, and the pattern that the gains grow with how much exploration a question needs.

### Limitations

The paper has no limitations section, so these come from its setup and experiments.

First, the environments are reconstructed indoor 3D scenes, the camera has no collision checking and can pass through walls and furniture, and there are no experiments on a real robot.

Second, rendered images have limited visual quality. The authors say so themselves, and argue that keeping actions and images consistent matters more.

Third, the main results all rest on a single base model, Qwen3.5-9B, and at test time correctness is decided by another language model, Qwen3-30B-A3B.

Fourth, more moves doesn't mean more accuracy. With the interaction reward, scores on the training domains dropped slightly, and the authors acknowledge that more exploration doesn't necessarily give higher accuracy.

### Future work

The paper's conclusion is that outcome-based reinforcement learning can teach a model not only how to reason over visual evidence but also how to go and get it. The paper doesn't lay out specific next steps.

My read is that the obvious next step is a real robot that actually moves. That means collisions, sensor noise and real cameras, and whether exploration habits learned in rendered scenes carry over is still an open question.

My read is that the real contribution is the change of setting. Instead of showing the model everything and then asking, you give it one glance and let it decide where to look. For people working on 3D understanding and embodied AI, that's much closer to the real world.

That's today's Daily Frontier Papers. The paper is linked in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
