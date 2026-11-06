# PyRUA-DataFarm: LLM Agent as a Cost-Efficient Teleoperator for Real-Ready Robot Data

**让Agent替人采机器人数据**

- 论文 Paper: [arXiv 2611.05320](https://arxiv.org/abs/2611.05320)
- 单位 Affiliation: 北京大学、英伟达
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-05
- 视频 Videos: [YouTube 中文](https://youtu.be/_zbgFkgyBcY) · [YouTube English](https://youtu.be/pcAOU9Oechk) · [B站](https://www.bilibili.com/video/BV1TF4v65EWV)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇北京大学和英伟达的论文，叫 PyRUA-DataFarm，意思是，让大模型 Agent 当一个便宜的遥操作员，生产能直接上真机的机器人数据。

简单说，给 Agent 一段真机录像和一句需求，它自己在仿真里把任务搭出来，再用视频模型把仿真画面变得像真机拍的，训练出的策略直接上真机。

### 问题

先说背景。现在训练机器人策略，主要靠遥操作数据，就是人远程操控机器人，把示范一条条录下来。

这种数据很真实，但很贵。每一条都要占用机器人和操作员，失败了要人复位，换个任务、换个场景，又得从头录。

### 以前的方法

合成数据本来想解决这个问题，但论文认为，现有做法只是把采集成本，换成了另外两种成本。

第一种是人工。不管是把少量示范扩增成很多，还是让语言模型写任务，每个新任务的场景、示范程序和成功判定，还得有人去搭、去调。

第二种是仿真和真实的视觉差距。先把真实场景重建进仿真，能缩小差距，但布料、会亮的显示屏这类东西，看起来还是假的。而直接用视频模型生成机器人视频，画面真了，却给不出精确的动作标签。

### 核心想法

DataFarm 的思路是分工。编码 Agent 接替遥操作员，仿真器负责动作，视频模型负责外观。

动作来自仿真，所以标签是精确的。外观来自真实图像，所以画面像真的。想换桌布、换灯光，只要改一句文字描述。

但让 Agent 生产数据，就有一个新问题，谁来检查。Agent 自己写示范程序，又自己写成功判定，就可能给自己的错误放行。所以 DataFarm 把认证和生产分开，谁生产，谁就不能给自己签字。

这张总览图就是整个流程。一段标定录像进来，Agent 在仿真里重建任务，视频模型渲染真实外观，最后独立审核，合格的才进数据集。

### 方法

输入是一个请求，比如这个任务要 100 条数据，再加一段标定录像，也就是机器人在目标场景里的一小段多视角视频。它只用来认场景和相机，里面的动作一条都不用。

第一步，还原场景。Agent 识别机器人和物体，估计几何形状，拟合相机和尺度。质量、摩擦这些视频里看不出来的量，不去瞎猜，而是标明是测量、拟合、假设还是未知，不确定的就在仿真里随机化。

第二步，检查仿真器。先用小测试确认仿真器能做出任务需要的技能，比如布料仿真。不然示范失败了，会被错怪到示范程序头上。

第三步，写任务和示范程序，这是防作弊的关键。先写一份任务合同，列出步骤、完成条件和禁止的捷径，比如把物体瞬移过去。再由另一个角色把合同变成裁判程序，在写示范程序之前就冻结。

写示范程序的 Agent，可以在开发用的随机种子上反复试，看裁判的结果，但看不到裁判的代码，也跑不了最终验收用的保留种子。只有在保留种子上通过，才算合格。这防的就是 reward hacking，也就是钻评分规则的空子。

第四步，渲染。默认用 MiniMax-H3 视频模型。从仿真视频里提取边缘图，通过 ControlNet 锁住机械臂姿态、物体位置和时间，真实画面只提供外观。

还有一个关键设计，头部和两个手腕相机的画面，拼在一张画布上一起生成，这样几个视角的光照一致，有哪些物体、夹爪拿着什么，也能对得上。长视频分段独立生成，再在相邻两段差别最小的地方拼起来，避免越生成越偏。

最后是审核。自动检查程序会看物体对不对、有没有多出来的东西、事件时间和仿真是否一致、背景是否静止。这个检查程序本身也要先校准，作者故意把合格数据改坏，比如删掉物体、挪动事件时间，每种破坏都必须被抓出来。

自动检查之后，每一条数据还要交给一个独立的审核 Agent，对照真实参考画面和仿真状态。没通过的退回出错的那一步，所有失败尝试都留有记录。

### 实验和效果

实验用一台双臂 PiPER 机器人，五个任务，叠杯子，把苹果放到带显示屏的电子秤上，敲鼓，把方块放进抽屉再关上，还有叠毛巾。

每个任务给 Agent 一段标定录像，要 100 条数据。论文说，跑实验的 Agent 可以是 GPT-6 Astra、GPT-5.6-Sol 或者 Claude Opus 5.5。

对比是这样设计的。都用 π0.5 这个 VLA 模型做后训练，三份训练集各 100 条，配方一样，只是数据来源不同。一份是真人遥操作。一份叫 Sim 加 DR，和 DataFarm 用同样的仿真轨迹，但画面直接用仿真器渲染，加上随机纹理和光照，这是传统做法。还有一份是 DataFarm 的数据，一条真实数据都没有。

每个任务、每种条件在真机上跑 20 次。一种是原始场景，另一种铺上训练集里都没有的桌布，还改变房间灯光的颜色。

结果在这张图。原始场景下，一共 100 次，DataFarm 数据训练的策略成功 93 次，遥操作数据是 84 次。五个任务里四个持平或更好，只有叠杯子少了一次。

换了桌布和灯光，差距拉开了。遥操作数据训练的策略掉到 46 次，DataFarm 还有 87 次，五个任务都领先。

作者的解释是，渲染器能围绕同一条精确的轨迹，不断变换桌面和灯光，策略就没法依赖桌子长什么样。而遥操作数据，只有录制时那一个场景。

传统的 Sim 加 DR，原始场景只成功 26 次，新场景 21 次。它和 DataFarm 动作完全一样，只是画面来源不同，所以 93 比 26 的差距，体现的就是用真实图像渲染的作用。

渲染会不会让动作走样。作者用光流对比渲染视频和仿真视频，H3 渲染平均偏差 4.4 个像素，完全照搬仿真运动的 Sim 加 DR 也有 3.4 个像素，而故意错开半秒去比，偏差升到 8.8 个像素。

画面像不像真的，作者用 KID 衡量和真实画面的距离，越小越像。三个任务平均，H3 渲染是 0.83，仿真器原始画面是 1.93，加了随机纹理反而更远，是 2.50。

这张图更直观，同一个任务阶段，真实录像、仿真器、加随机化的仿真，和 DataFarm 的渲染，放在一起。

那为什么不直接把仿真视频丢给视频模型。作者试了，直接调用时，模型会照抄参考图里的任务状态，运动相关性只有 0.07，DataFarm 的渲染器是 0.74。每个相机分开生成，手腕视角里还会多出机械臂。

图 8 里红框标出的，就是和仿真对不上、或者几个视角互相矛盾的画面。

最后是成本。作者用六种大模型和编码框架的组合，在三个任务上测一次性的适配成本，六种都能跑通，平均每个任务 12 到 124 美元，差别主要来自 token 价格。

但最便宜的组合，有两个任务需要人追加指令，还放进了超过关节速度上限的数据。后面报告的成本，用的是 Claude Opus 5.5 加 SoL-Pi 框架，全程不需要人介入。

按这个配置，适配一个新任务平均花 21 美元，只付一次，渲染合同再加 5 美元。之后每条数据平均 0.8 美元，不需要机器人，也不需要操作员。

折算下来，第一个小时的三视角数据平均 107 美元，之后每小时 81 美元。作为对比，作者在中国拿到的遥操作报价，大约每小时 800 元人民币，也就是 117 美元，北美光人工就估算要 290 美元左右。

### 和以前方法的对比

和以前的工作比，论文这张表列得很清楚。MimicGen 这类方法要人先搭好仿真任务，RoboTwin 2.0 让语言模型写任务，但画面还是仿真渲染。DreamGen 直接生成视频，动作只能靠模型反推。

DataFarm 的不同在于，只要一段标定录像，由 Agent 搭任务，动作来自仿真，画面由以真实图像为条件的视频模型生成，而且每一条多视角数据都经过独立审核。

我的判断是，这篇最有价值的，是把 Agent 放进数据生产时的那套制衡设计。裁判对写程序的 Agent 隐藏，生产和认证分开，每条数据都有审核记录，会犯错的 Agent 产出的数据，也能被查。

我的判断是，也要看到对比的边界。真机上只和遥操作、Sim 加 DR 比，没有和 DreamGen 这类生成式方法正面比较。而且每个任务、每种条件只有 20 次，单个任务差一两次，说明不了太多。

### 局限

作者自己列了几条局限。第一，DataFarm 依赖仿真器能做出任务需要的技能，做不到的话，得先做一个新的仿真器版本。

第二，标定录像看不出质量、摩擦这些动力学参数，只能随机化，对成败取决于精确动力学的任务，可能不够。

第三，生成视频的检查并不完美，自动检查会漏掉问题，所以每一条都要独立审核。第四，评测只有一种双臂机器人和五个任务。

成本这块，作者也说明，中国的数字是全包报价，北美是只算人工的估算，两边不是同口径。而且成本依赖自己部署的渲染器，按 Seedance 的公开价格，渲染一条就要 12 到 63 美元。

作者还提醒，数据便宜了，不代表部署更安全，策略上线前还是要在真机上评测。

### 未来方向

未来方向，作者提到的是，其他机器人形态和任务类型还有待测试。

我的判断是，如果这条路走得通，机器人数据里真机和操作员的投入，会从每条都要，变成每个任务只要一段标定录像，换场景只是改一句描述，这对需要大量多样数据的 VLA 训练很有吸引力。

我的判断是，真正要看的是，它能不能扩展到依赖精确动力学的任务，以及换一种机器人还稳不稳，这两点论文还没有回答。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Peking University and NVIDIA. It's called PyRUA-DataFarm, and the idea is an LLM agent as a cheap teleoperator, producing robot data that's ready for the real robot.

The short version: give an agent one real recording and a one-line request. It builds the task in simulation, a video model makes the footage look real, and the policies trained on it go straight onto the robot.

### Problem

Some background. Robot policies are still trained mostly on teleoperation data, where a person drives the robot remotely and every demonstration gets recorded.

That data is faithful, but it's expensive. Every episode ties up a robot and an operator, someone resets the scene after failures, and a new task or scene means starting over.

### Prior work

Synthetic data was supposed to fix this. The paper argues that existing pipelines just trade the cost of collection for two other costs.

The first is human engineering. Whether you multiply a few demonstrations or have a language model write the task, a person still builds and debugs the scene, the expert controller and the success check for every new task.

The second is the visual gap. Reconstructing the real scene in simulation helps, but things like cloth or a lit display still look synthetic. And video models that generate the motion themselves look real, but can't give you exact action labels.

### Key idea

DataFarm splits the job. A coding agent replaces the teleoperator, the simulator supplies the actions, and a video model supplies the appearance.

Actions from simulation means exact labels. Appearance from real images means realistic frames. And a new tablecloth or new lighting is just a text change.

But letting an agent produce data raises a new question: who checks it? An agent that writes both the expert and its success check can approve its own mistakes. So DataFarm separates certification from production, and nothing signs off on its own work.

This overview shows the pipeline. A calibration recording comes in, the agent rebuilds the task in simulation, a video model renders the real look, and an independent review decides what enters the dataset.

### Method

The input is a request, say one hundred episodes of this task, plus a calibration episode, a short multi-view recording of the robot in the target scene. It identifies the scene and the cameras, and none of its actions are reused.

Step one is grounding. The agent identifies the robot and objects, estimates their geometry, and fits the cameras and scale. Things a video can't reveal, like mass or friction, aren't guessed. Each quantity is labelled measured, fitted, assumed or unknown, and uncertain ones are randomized.

Step two, qualify the simulator. Short probes confirm it can reproduce the skills the task needs, like cloth. Otherwise a simulator failure would be blamed on the expert.

Step three is the task and the expert, where the anti-cheating design lives. A task contract lists the stages, what counts as done, and forbidden shortcuts, like teleporting objects. A separate role turns it into a judge, frozen before the expert is written.

The agent writing the expert can try things on development seeds and see the verdicts, but it can't read the judge or run the held-out seeds used for admission. It passes only if it succeeds on those. That's a guard against reward hacking, where a model games its grader.

Step four is rendering, by default with the MiniMax-H3 video model. Edge maps from the simulator video drive a ControlNet that pins down the arm pose, object positions and timing, while real frames only supply the look.

Another key choice: the head camera and both wrist cameras are tiled on one canvas and generated together, so the views share lighting and agree on which objects exist and what each gripper holds. Long episodes are generated in independent chunks and joined where neighbouring chunks differ least, which avoids drift.

Then admission. An automatic verifier checks for the right objects and no extras, event timing that matches the simulation, and a still background. The verifier is calibrated first: the authors deliberately corrupt a good episode, deleting objects or shifting events, and every corruption must be caught.

After that, every episode goes to an independent reviewer agent, which compares it with the real reference frames and the simulated state. Rejections go back to the stage that caused them, and every failed attempt is logged.

### Results

The robot is a dual-arm PiPER, with five tasks: stacking cups, putting an apple on a scale with a live display, beating a drum, putting a block in a drawer and closing it, and folding a towel.

For each task the agent gets one calibration episode and a request for one hundred episodes. The paper says the agent can be GPT-6 Astra, GPT-5.6-Sol, or Claude Opus 5.5.

The comparison: every policy is pi zero point five, a VLA model, post-trained on one hundred episodes with the same recipe, and only the data source changes. One set is real teleoperation. Another is Sim plus DR, the same simulated trajectories as DataFarm but rendered by the simulator with randomized textures and lighting, the conventional approach. The third is DataFarm data alone, with no real episodes.

Each policy gets twenty real-robot trials per task and condition. One is the original scene. The other adds a tablecloth that's in no training set and changes the colour of the room light.

Here are the results. In the original scene, out of one hundred trials, the DataFarm policy succeeds ninety-three times, against eighty-four for teleoperation. It matches or beats teleop on four of five tasks, and is one short on stacking cups.

With the new tablecloth and lighting, the gap opens. Teleop-trained policies drop to forty-six, while DataFarm keeps eighty-seven, and leads on every task.

The authors' explanation: the renderer varies the table and lighting around the same exact trajectories, so the policy can't rely on how the table looks. A teleop dataset only shows the scene it was recorded in.

Sim plus DR manages only twenty-six successes in the original scene and twenty-one in the new one. It has the same actions as DataFarm and differs only in how images are made, so ninety-three versus twenty-six is the effect of rendering with real images.

Does rendering distort the motion? Comparing optical flow with the simulator video, H3 renders are off by four point four pixels on average. Sim plus DR, which has exactly the simulated motion, is still off by three point four. Shift the comparison by half a second and the error jumps to eight point eight.

Do the renders look real? On KID, a distance to real camera frames where lower is better, averaged over three tasks, H3 scores zero point eight three. The raw simulator is one point nine three, and domain randomization makes it worse, at two point five.

This figure shows it: the same task phase in a real recording, in the simulator, with randomization, and in a DataFarm render.

So why not just hand the simulator video to the video model? The authors tried. A direct call copies the task state from its reference images, with a motion correlation of just zero point zero seven, against zero point seven four for DataFarm's renderer. Generate each camera separately, and the wrist views sprout extra arms.

In Figure 8, the red boxes mark views that contradict the simulator or each other.

Finally, cost. The authors tried six combinations of language models and coding-agent harnesses on three tasks. All six got through setup, at twelve to one hundred twenty-four dollars per task on average, mostly tracking token prices.

But the cheapest run needed follow-up instructions on two tasks and admitted episodes that exceed the robot's joint speed limit. The reported costs use Claude Opus 5.5 in the SoL-Pi harness, which needed no human help.

With that setup, adapting a new task costs twenty-one dollars on average, paid once, plus five for a render contract. Each further episode costs about eighty cents, with no robot and no operator.

That works out to one hundred seven dollars for the first hour of three-view data, and eighty-one for each hour after. For comparison, a teleoperation quote the authors got in China is about eight hundred yuan, or one hundred seventeen dollars, an hour, and they estimate North American labour alone at about two hundred ninety.

### Comparison

The paper's table lays out earlier work. MimicGen-style methods need a person to build the simulation task. RoboTwin 2.0 has a language model write the task, but images are still simulator renders. DreamGen generates video directly, so actions have to be inferred.

DataFarm needs only one calibration episode, an agent builds the task, the simulator provides actions, a video model conditioned on real images provides frames, and every multi-view episode is independently reviewed.

My read is that the most valuable part is the checks and balances around an agent producing data. The judge is hidden from the code writer, production and certification are separate, and every episode has a review record, so data from an agent that makes mistakes can still be audited.

My read is also that the comparison has limits. On the real robot, the only baselines are teleoperation and Sim plus DR, with no head-to-head against generative methods like DreamGen. And with twenty trials per task and condition, one or two successes on a single task doesn't tell you much.

### Limitations

The authors list several limitations. First, DataFarm needs a simulator that can reproduce the task's skills, and if it can't, the task needs a new simulator variant first.

Second, a calibration episode can't reveal dynamics like mass or friction, so they're randomized, which may not be enough for tasks that hinge on precise dynamics.

Third, checking generated video is imperfect, and automatic checks miss things, which is why every episode is reviewed. Fourth, the evaluation covers one dual-arm robot and five tasks.

On cost, the authors note that the China figure is a full-service quote and the North American one a labour-only estimate, so they're not like for like. And the costs rely on a self-hosted renderer: at Seedance's published price, rendering one episode would cost twelve to sixty-three dollars.

They also warn that cheaper data doesn't make deployment safer, and policies still need real-robot evaluation before use.

### Future work

Looking ahead, the authors say other robots and task families remain to be tested.

My read is that if this holds up, robot and operator time shifts from every episode to one calibration episode per task, with a new scene being a one-line description. That's attractive for VLA training, which needs lots of varied data.

My read is that the real tests are whether it extends to tasks that hinge on precise dynamics, and whether it holds up on a different robot. The paper doesn't answer either yet.

That's today's Daily Frontier Papers. The paper is linked in the description.

