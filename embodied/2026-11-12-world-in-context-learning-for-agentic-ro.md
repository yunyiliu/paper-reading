# World In-Context Learning for Agentic Robot Manipulation

**一条示范不改权重学操作**

- 论文 Paper: [arXiv 2611.12885](https://arxiv.org/abs/2611.12885)
- 单位 Affiliation: 华中科技大学、ArcheOne AI
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-12
- 视频 Videos: [YouTube 中文](https://youtu.be/zntJavtcGHg) · [YouTube English](https://youtu.be/g885jplIOOs) · [B站](https://www.bilibili.com/video/BV1q3Wd61EAe)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇华中科技大学和 ArcheOne AI 的论文，题目可以翻成，让机器人在上下文里学会操作。

先说结论。他们的系统叫 RoboScript，在 RoboTwin 2.0 全部 50 个任务上，每个任务只给一条示范，不更新任何权重，平均成功率 83.4%，比最强的训练型策略高出 20 多个点。

### 问题

机器人操作的主流做法，是训练一个动作策略，比如视觉语言动作模型 VLA，或者世界动作模型 WAM，把技能写进权重。

问题在于，学到的东西被绑在训练数据的分布上。论文举的例子是 FastWAM，在 RoboTwin 2.0 的干净场景里成功率 77.8%，一到随机化场景，就掉到 1.9%。

而且每来一个新任务、新分布，都要重新采数据、重新训练。

### 以前的方法

另一条路是上下文学习，in-context learning，权重不动，只看提示里的示范做新任务，最近已经能靠一条示范完成新任务。

但它一直追不上训练出来的策略。示范是在别的场景录的，没法告诉机器人，眼前这张桌子上会发生什么。

Code as Policies 这类方法让模型写代码控制机器人，但没有世界反馈，错误会一路累积。

### 核心想法

作者的灵感来自编程 Agent。编程 Agent 学一个新代码库，靠的是在终端里跑代码、看返回结果，机器人操作恰恰缺这样一个反馈回路。

这张图就是这个类比，左边是编程 Agent 和终端，右边是操作 Agent 和世界。对机器人来说，世界就是它的终端。

关键观察是，Agent 在上下文里学到的是语义，记的是物体扮演什么角色、任务分几步、哪里出过错，而不是场景长什么样。所以一个世界里攒下的经验，可以搬到另一个世界，只要任务语义一致。

于是有了 WICL，世界内上下文学习。权重始终冻结，从一条示范出发，在一个世界里排练，把反馈写进上下文，而不是权重。这个世界可以是模拟器，也可以是学出来的世界模型。

还有一条规矩，排练的场景和评测的场景完全不重叠，所以测的是泛化，不是死记硬背。

### 方法

具体的系统叫 RoboScript，这张图从左到右是四个阶段，造技能，学任务，部署，反思。

第一步造技能。Agent 把示范切成一段段动作，观察、抓取、放置这些基础动作解释不了的部分，交给编程 Agent 写新技能，不许写死任何绝对位置。

每个动作都返回一张回执，写明目标位姿、实际位姿、夹爪开度、手里有没有东西，全部来自本体感知和运动规划器，真机上一样拿得到。

第二步学任务。学出来的叫 Task Memory，任务记忆，是一份结构化的 Markdown 文档，写着物体的角色、每个阶段的流程、抓取和放置的规格、学到的参数和教训。

抓取规格是相对的，比如从左边靠近把手，抓在中点，从不写坐标。还有一个检查器，记忆里只要出现坐标、真值信息或者实例编号，一律拒绝。

学习分两段。第一段是模仿，一个角色读示范写初版记忆，一个角色照着练，一个角色每练一次改一次记忆，直到稳定成功。

第二段是排练，换到新的训练场景，每个场景只算第一次尝试。失败了，复盘角色拿到一份真值诊断，去改记忆的副本，只有改完能让这个场景成功，才合并进正式记忆。

这里有严格的数据隔离。动手的角色只能看到相机的 RGB-D 图像和本体感知，跟真机一样，模拟器的特权信息只用来监督学习。

第三步是部署。记忆冻结，在没见过的场景上跑，不再有世界反馈。动手之前，Agent 先把示范的语义蓝图对到当前场景，看哪个物体对应哪个角色、哪只手够得着。

一个动作失败了，回合不会结束，Agent 退到安全姿态，重新观察，从当前状态接着做。它自己确认完成才收手，但算不算成功，由模拟器的官方检查说了算。

第四步是反思。每评测完一批，反思 Agent 读记录，里面只有部署时能看到的信息和成败，自己找原因，写指导给下一批。

### 实验和效果

实验在 RoboTwin 2.0 上，50 个双臂任务，分干净和随机化两种设置，随机化就是纹理、光照、桌高和干扰物都随机。排练只在干净场景，每个任务每种设置评测 10 个回合，共 1000 个回合。

对比的训练型策略，每个任务用 50 条干净示范训练，结果取自官方排行榜或已发表的报告。

主结果在表 1。用 GPT-6.1-Sol 做底座，干净场景 83.6%，随机化 83.2%，平均 83.4%。最强的训练型策略 4D-WAM，干净 81.5%，随机化掉到 41.8%，平均 61.7%。

两者在干净场景差不多，差距来自随机化。用保持率算，也就是随机化成功率除以干净成功率，RoboScript 是 99.5%，训练型策略里最高的 π0.5 是 65.1%。

换底座模型也成立。DeepSeek-V4.1-Flash 平均 78.1%，GPT-5.6-Luna 是 77.9%，都高于所有训练型策略，代码和提示词一点没改。

为什么随机化不掉分。作者把 GPT-6.1-Sol 失败的 166 个回合按原因分类，知识类失败在两种设置下是 37 和 38，几乎没变，变多的是执行类失败，从 23 到 29。

消融在表 2，用 10 个任务。只拿示范写记忆、不在世界里练，成功率 69.0%，加上和世界的交互是 78.5%。把示范直接放进部署时的上下文，也就是传统的上下文学习，是 67.5%，和只给指令的 68.0% 差不多。

反过来，没有示范、只从指令开始排练，成功率从 86.0% 掉到 56.5%，是全文最大的落差。示范和世界，缺一不可。

还有一个关键检查。把学习阶段的特权信息全部拿掉，成功率 86.5%，完整方法是 86.0%，说明成绩不是靠偷看模拟器真值来的。

反思循环跑了三轮。在 17 个较弱的任务上，每轮救回来的回合都比弄坏的多，净增 18、8、4 个回合，收益减得很快。

成本在表 3。学完 50 个任务再评测 1000 个回合，按 API 标价，GPT-5.6-Luna 花了 308 美元，GPT-6.1-Sol 花了 1911 美元。GPT-6.1-Sol 一个回合平均调用模型 54.7 次，用掉 233 万 token。

最后是真机。作者把在模拟的 ALOHA 机器人上学到的记忆，用一个轻量适配器改写坐标、运动学和夹爪参数，直接搬到真实的双臂 PiPER 机器人上，不重新示范，也不重新排练。

四个任务各测 10 次，按铃成功 10 次，双手抓滚筒 7 次，移开扑克牌 7 次，叠两个碗 6 次。失败主要来自夹爪没对准，和真实传感器的深度误差。

### 和以前方法的对比

和最接近的 HarnessVLA 比，它用编程 Agent 调度一个在 RoboTwin 数据上微调过的 VLA，随机化场景是 58.4%。RoboScript 只在干净场景排练过，随机化场景高出 24.8 个点。

论文还指出，以往用冻结模型做机器人 Agent 的工作，都没有把世界里得到的经验，沉淀成可复用的记忆。

我的判断是，最有说服力的是消融里那一组。同一个底座模型，示范直接塞进上下文只有 67.5%，加上在模拟器里排练到了 78.5%，说明提升确实来自世界反馈，而不只是大模型本身强。

我的判断是，也有几处要打折扣。RoboScript 每个任务每种设置只测了 10 个回合，π0.5 是排行榜上的 100 个回合，基线数字都来自排行榜和已发表的报告。真机实验是先离线规划好整个动作序列再执行，没有在线重规划，也只有四个任务。

### 局限

作者列了几条局限。第一，依赖商业大模型 API，服务质量随时间波动，不同时段成功率能差好几个点。

第二，慢。GPT-6.1-Sol 一个回合平均调用 54.7 次模型，机器人每一步都要等。按附录，一个部署回合的中位时长是 6.4 分钟，大约 80% 的时间在等模型。

第三，反思的收益缩得很快，容易的失败先被修掉，剩下的成功信号稀疏，很难归因。

第四，评测的都是刚体或带关节的物体。布料、线缆这类可变形物体，还有交接这种需要两只手紧密配合的任务，依然很难。

另外，这次的世界就是 RoboTwin 2.0 模拟器本身。用学出来的世界模型来排练，作者放在了未来工作里。

### 未来方向

作者提的方向有三个，技能不够用时 Agent 自己造新技能，多个专门的 Agent 协作完成长任务，没有模拟器时到学出来的世界模型里排练。

他们还认为，专门训练一个会用具身工具、会维护任务记忆的基础模型，能让 WICL 更快更便宜。代码已经在 GitHub 公开。

我的判断是，这篇把编程 Agent 那一套，跑一跑、看结果、改笔记，搬到了机器人上。在公开基准上，一条示范在干净场景追平、在随机化场景大幅超过 50 条示范训练的策略。

我的判断是，它离实用还有距离，一个回合要等好几分钟、用掉两百多万 token，真机也只验证了四个简单任务。它更像是证明了技能不一定要存在权重里，而不是马上能替代 VLA。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Huazhong University of Science and Technology and ArcheOne AI, and it asks whether a robot can learn a manipulation task in its context as well as in its weights.

Here's the headline. Their system, RoboScript, takes one demonstration per task and no weight updates, and on all fifty tasks of RoboTwin 2.0 it reaches eighty-three point four percent mean success. That's more than twenty points above the strongest trained action policy.

### Problem

Some background. The standard way to teach a robot manipulation is to train an action policy, either a vision-language-action model, a VLA, or a world action model, a WAM, and the skill ends up stored in the network's weights.

The catch is that what's learned stays tied to the training distribution. The paper's example is FastWAM: seventy-seven point eight percent success on clean RoboTwin scenes, and one point nine percent once the scenes are randomized.

And every new task or new distribution means new data and new training.

### Prior work

The other route is in-context learning. The weights stay frozen, and the model picks up a new task from what its prompt shows. Recent models can already do a new manipulation task from a single demonstration in context.

But that route hasn't matched trained policies. A demonstration is recorded in some other scene, so it can't tell the robot what will happen on the table in front of it.

Then there's Code as Policies and its successors, where a model writes code against robot APIs. Without feedback from the world, their errors pile up.

### Key idea

The inspiration is coding agents. A coding agent learns an unfamiliar repository by running code in a terminal and reading what comes back. Robot manipulation has no equivalent of that loop.

This figure makes the analogy. On the left, a coding agent and its terminal. On the right, a manipulation agent and the world. For the robot, the world is the terminal.

The key observation is that what an agent learns in context is semantic. It records object roles, task steps, and what went wrong, not what the scene looks like. So experience from one world carries over to another, as long as the task means the same thing in both.

That gives World In-Context Learning, or WICL. The weights stay frozen. The model starts from one demonstration, rehearses the task in a world, and writes the feedback into its context instead of its weights. The world can be a simulator, or a learned world model.

One more rule. The scenes used for rehearsal never overlap with the scenes used for evaluation, so the score measures generalization, not memorization.

### Method

The system is called RoboScript, and this figure runs left to right through its four stages: build skills, learn the task, deploy, and reflect.

Stage one builds skills. There's a set of environment-independent entry points, like observe, grasp and place. The agent cuts each demonstration into action segments, and whatever the existing entries can't explain goes to a coding agent that writes a new skill. Skills may not hard-code any absolute position.

Every action returns a receipt: the commanded pose, the pose actually reached, how far the gripper is open, and whether an object is held. Receipts come from proprioception and the motion planner, not from simulator state, so a real robot can produce them too.

Stage two learns the task. The result is a Task Memory, a structured Markdown document covering the roles of the objects and how to find them, the workflow of each stage, grasp and place specifications, learned parameters, and lessons from past attempts.

Grasp specs are relative, something like approach the handle from the left and grasp at its midpoint, never a coordinate in a particular scene. A linter rejects any memory that contains coordinates, ground-truth terms, or instance identifiers.

Learning has two parts. First, imitation. One role reads the demonstration and writes the first memory, another practices on the demonstration scene, and a reviewing role revises the memory after each attempt until the agent succeeds repeatedly.

Second, rehearsal on new training scenes, where only the first attempt on each scene counts. After a failure, the reviewer gets a ground-truth diagnosis and edits a cached copy of the memory. The edit is merged only if it makes that scene succeed, and the edit budget shrinks with experience, so the memory settles down.

Data isolation matters here. The roles that actually act see only RGB-D camera images and proprioception, the same inputs a real robot has. Privileged simulator information is used only as supervision, and it reaches the actor only through the memory.

Stage three is deployment. The memory is frozen and run on unseen scenes, with no feedback from the world. Before acting, the agent maps the demonstration's semantic blueprint onto the current scene: which object plays which role, and which arm can reach.

A failed action doesn't end the episode. The agent moves to a safe pose, looks again, and continues from where things stand. It declares the task done only when its perception says the scene matches the demonstration's final state, but the simulator's official check decides what counts as success.

Stage four is reflection. After each evaluation block, a reflection agent reads the records, which hold only what a deployed robot would observe, plus whether each episode succeeded. It works out the causes itself and writes guidance for the next block, and blocks never overlap.

### Results

The benchmark is RoboTwin 2.0: fifty dual-arm tasks, each in a clean setting and a randomized one with random textures, lighting, table height and distractors. Each task gets one automatically chosen clean demonstration, and rehearsal happens on clean scenes only. Each task is evaluated on ten episodes per setting, a thousand episodes in all.

The trained policies it's compared with use fifty clean demonstrations per task, and their numbers come from the official leaderboard or published reports.

Main results are in Table 1. With GPT-6.1-Sol as the base model, RoboScript scores eighty-three point six clean, eighty-three point two randomized, eighty-three point four on average. The strongest trained policy, 4D-WAM, scores eighty-one point five clean but forty-one point eight randomized, sixty-one point seven on average.

So the two are close on clean scenes, and the gap comes from randomization. The paper measures retention, randomized success divided by clean success. RoboScript retains ninety-nine point five percent, and the best trained policy, pi zero point five, retains sixty-five point one.

It holds across base models too. DeepSeek-V4.1-Flash averages seventy-eight point one and GPT-5.6-Luna seventy-seven point nine, both above every trained policy, with identical code, prompts and protocol.

Why doesn't randomization hurt? The authors sorted GPT-6.1-Sol's one hundred sixty-six failed episodes by cause. Knowledge failures were thirty-seven clean and thirty-eight randomized, basically unchanged. Execution failures went up, from twenty-three to twenty-nine, which the authors say is consistent with random table heights making motions harder.

The ablations, in Table 2, use ten tasks. Writing the memory from the demonstration alone, with no practice in the world, gives sixty-nine percent. Adding interaction with the world gives seventy-eight point five. Putting the demonstration straight into the deployment context, plain in-context learning, gives sixty-seven point five, about the same as the instruction alone at sixty-eight.

The other way round, rehearsing from the instruction with no demonstration drops the mean from eighty-six to fifty-six point five, the biggest gap in the study. You need both the demonstration and the world.

One important check. Removing all privileged information from learning gives eighty-six point five, against eighty-six for the full method. So the score doesn't come from peeking at simulator ground truth.

The reflection loop ran for three rounds. On seventeen weaker tasks, each round rescued more episodes than it broke, with net gains of eighteen, eight and four, so the returns shrink fast.

Cost is in Table 3. Learning all fifty tasks and running the thousand evaluation episodes cost three hundred eight dollars at list API prices with GPT-5.6-Luna, and one thousand nine hundred eleven with GPT-6.1-Sol. With Sol, one evaluation episode averages fifty-four point seven model calls and two point three three million tokens.

Finally, a real robot. The authors took memories learned on a simulated ALOHA robot and used a lightweight adapter to remap coordinate conventions, arm kinematics and gripper parameters for a physical dual-arm PiPER, with no new demonstrations and no new rehearsal.

Four tasks, ten tries each: ring the bell, ten out of ten. Grab a roller with both arms, seven. Move a playing card away, seven. Stack two bowls, six. Failures came mainly from gripper misalignment and depth errors on real sensors.

### Comparison

The closest agent baseline is HarnessVLA, which has a coding agent orchestrate a VLA fine-tuned on RoboTwin data. Its best randomized score is fifty-eight point four. RoboScript, which only ever rehearsed on clean scenes, beats it there by twenty-four point eight points.

Against earlier frozen-model robot agents, the paper's point is that they put demonstrations, instructions or tool descriptions into context, but none of them distills experience from a world into a reusable memory.

My read is that the most convincing evidence is that ablation group. A demonstration dropped straight into context gets sixty-seven point five, and rehearsing in the simulator lifts it to seventy-eight point five with the same base model. That points to world feedback as the source of the gain, not just a stronger language model.

My read is also that some of this deserves a discount. RoboScript was evaluated on ten episodes per task and setting, while pi zero point five's leaderboard numbers use a hundred, and all baseline numbers come from the leaderboard or published reports. The real-robot test planned each full action sequence offline with no online replanning, and covered only four tasks.

### Limitations

The authors list several limitations. First, it relies on commercial models served through APIs, and service quality drifts enough to move success by several points between sessions.

Second, it's slow. With GPT-6.1-Sol an evaluation episode averages fifty-four point seven model calls, and the robot waits for every one of them. According to the appendix, a deployment episode takes a median of six point four minutes, and about eighty percent of that is waiting on the model.

Third, the gains from reflection shrink quickly, because the easy failures get fixed first and sparse success signals make the rest hard to attribute.

Fourth, every task involves rigid or articulated objects. Deformable things like cloth and cables, and tasks that need tight two-arm contact such as handovers, remain hard.

Also, the world here is the simulator itself, with rehearsal and evaluation both in RoboTwin 2.0. Rehearsing in a learned world model is something the authors leave for future work.

### Future work

The authors name three directions: agents that build new skills when existing ones fail, several specialized agents coordinating on a long task, and rehearsal inside learned world models when no simulator is available.

They also argue that a foundation model trained to use embodied tools, read physical feedback and maintain a task memory could make WICL faster, cheaper and more reliable. The code is already on GitHub.

My read is that this takes the coding-agent loop of run it, read the result, update your notes, and puts it on a robot. On a public benchmark, one demonstration matches fifty-demo policies on clean scenes and beats them by a wide margin on randomized ones.

My read is also that it's a long way from practical. Episodes take minutes and millions of tokens, and the real robot was tested on just four fairly simple tasks. What it shows is that task skill doesn't have to live in weights, not that it's ready to replace VLAs.

That's today's Daily Frontier Papers. The paper is linked in the description.

