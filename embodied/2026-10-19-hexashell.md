# HexaShell: A Thin Harness for Exploring the Limits of VLM Agents as Robot Coding Agents

**编程Agent直接操作机器人**

- 论文 Paper: [arXiv 2610.23479](https://arxiv.org/abs/2610.23479)
- 单位 Affiliation: HexaFuture
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-10-19
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1MRJR6XEfv)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 HexaFuture 团队的论文，叫 HexaShell，把写代码的 Agent 直接接到机器人上。

它问的问题很直接，如果编程 Agent 操作的不是电脑，而是一台双臂机器人，它能走多远，又会卡在哪里。

### 问题

先说背景。机器人操作的排行榜，一直是学习型策略领先，比如 VLA，也就是视觉语言动作模型，从大量示范里学习，直接输出动作。现在，基于视觉语言模型的 Agent 也开始追上来了。

难点在于把像素变成位置。在代码仓库里，Agent 面对的全是文字。到了机器人上，物体只是像素，但每个动作命令都需要世界坐标，抓取要精确到一厘米左右。视觉语言模型本来就不擅长判断距离，标准观测里也没有深度图。

### 以前的方法

让语言模型给机器人写代码，并不新鲜，比如 Code as Policies。论文指出，这些工作里，程序是每条指令生成一次，用完就丢，底层的基本动作由人来写。

另一条路，是直接把前沿大模型当策略用。在 RoboDojo 这个基准上，同一个模型 GPT-6-Astra，不带自己的工具，通过基准作者写的固定接口来选夹爪目标，成功率是 22.5%。

深度信息呢，可以用单目深度网络从一张图来估计。但论文测得，在 RoboDojo 的场景里，Depth Anything V2 即使用桌面校正了尺度，头部相机的中位误差也有 41 毫米，腕部相机大约 100 毫米。

### 核心想法

灵感来自编程 Agent 的日常。在代码仓库里，Agent 读代码，运行，看报错，再修，修好的代码会留下来。

HexaShell 把这个循环搬到机器人上。每次失败后，Agent 读日志，找原因，写或者修一些叫做工具的小程序，学到的东西以代码的形式留下来。作者说，这把以往的分工倒了过来，基本动作由 Agent 自己写，跨回合保留，被优化的正是这些工具。

另一个关键选择，是用三角测量代替深度网络。两台相机看到同一个点，这个点就在两条视线的交会处。如果点在已知平面上，比如桌面，它就在视线和这个平面的交点。

### 方法

先看接口，论文叫 harness。执行的 Agent 是没改过的 Codex 命令行，跑在一个没有模拟器、没有 API 密钥的容器里。它只拿到一个命令 robo，加上头部和两只手腕上三台相机的 RGB 图和标定，没有深度图。

基础命令就是平移、旋转、开合夹爪这些。服务器只做翻译，检查命令，插值成直线路径，解逆运动学，安排时间，再发给模拟器。论文的原则是，这一层只翻译，从不替 Agent 做决定。

还有预算。RoboDojo 给每个任务限定动作步数，从 200 到 1900 步不等。看图、测量和思考都不花步数，只有执行的动作才计步。

再看怎么测量。一个像素加上标定，就是空间里的一条射线，还需要再加一个约束。第一个约束是桌面，一个共享工具扫描候选高度，把一台相机的图像经过这个高度的平面投到另一台相机上，看哪个高度最吻合。论文报告，三对相机的结果相差不到 1 毫米。

第二个约束，是其他已知高度的平面，比如积木顶面、碗沿。第三个约束是第二个视角，头部相机看不清时，机器人就移动手腕相机去看，这叫主动感知。但移动要消耗步数，这是不用深度图的代价。

规则很严格，关于物体的每个数字，都必须在这一回合里测出来，不能取自基准的文件或日志。工具也不能调用模拟器，不能读物体的真实位姿。

最后是进化循环。每个任务用第 0 号种子的前十个布局，一个一个来。执行 Agent 用现有工具去做，失败了，同一个模型扮演的优化 Agent 就读命令日志、相机画面和工具代码，写下诊断，新增或修改一个工具和它的单元测试。

接着是交付检查，确认工具能加载，接口说明里不出现任务名和物体名，不通过就回滚。然后在同一个布局上再试，每个布局最多改五次。十个布局做完，工具冻结，拿到十个新布局上重新测试。

两个 Agent 都是 Codex 加 GPT-6-Astra，中等推理强度。每个任务一块 GPU，开发预算 12 个小时。

### 实验和效果

测试平台是 RoboDojo，在 Isaac Sim 里模拟的 42 个双臂任务，分泛化、记忆、长时序、精细操作和开放五类。泛化类任务还各有一个加了 13 到 18 个干扰物的杂乱版本。

主结果在图 2。按排行榜的算法，HexaShell 成功率 39.7%，进度分 48.0，作者称这是 RoboDojo 上新的最好成绩。排行榜上最高的学习型策略 VPP2 是 32.3%，PhysicalRSI 是 31.4%。

不过要说清楚，HexaShell 的成绩是每个任务在十个新布局上各跑一次，都来自第 0 号种子。排行榜上的条目，是三个种子各跑 50 回合。

分类别看，HexaShell 在要看懂和记住场景的任务上领先，记忆类 65.0%，开放类 50.0%，其他系统最高分别是 55.6% 和 31.0%。学习型策略在精细接触上领先，VPP2 精细操作类 35.2%，长时序类 37.0%，HexaShell 是 31.3% 和 23.8%。

杂乱场景很有意思。在开发布局上，加了干扰物，HexaShell 保留了 95% 的成功率，30 个学习型策略的中位数只保留 18%，VPP2 从 46.4% 掉到 12.6%。作者的解释是，Agent 在目标上点一个像素，工具只测这一个物体。

三角测量比深度网络好多少，看表 1。同样的接口、循环、模型和布局，三角测量在排行榜分数上高 8.7 个点。按布局配对，赢 57 个，输 23 个。

代价在表 2。用三角测量，3.6% 的执行步数花在移动相机去看上，深度网络只有 0.7%。但值得，在做过这种移动的 180 个布局上，三角测量成功 39 次，深度网络 22 次。

作为参考，直接用模拟器渲染的真实深度，分数还要再高 10.6 个点，差距主要在紧配合的插入任务上。

最重要的是表 3 这张能力地图。按任务需要的接触类型分，推、滑、按到了 92.5%，刚体的抓放和堆叠 50.0%，运动中的物体 40.0%，可形变物体和液体 33.3%，铰接物体和使用工具 7.5%，紧配合插入只有 8.3%，12 个插入变体里 7 个是零。

失败的方式也很说明问题。开发布局上 420 个回合，失败 268 个。除去 5 个工具报错，57% 是用光了步数，30% 是 Agent 宣布完成、检查器不认，13% 是还剩不少预算就放弃了。

在可形变、铰接和刚体任务上，大多数失败是用光步数。Agent 看得出结果不对，一直重试到步数耗尽，用作者的话说，它知道自己失败了，却没法让动作做成。而在记忆任务上，一半的失败是 Agent 以为做对了，检查器不认。

循环的成本，论文也给了。一共 769 轮优化，失败后的一轮中位数 6.3 分钟，最后留下 327 个任务命令，大约 7.8 万行 Python。

举个案例，推 T 形积木。工具要比较两台相机看到的积木面积，但头部相机像素粗，面积算小了，高度测量一直被拒。Agent 只好用桌面高度，差了 15 毫米，积木最后偏了 13.6 毫米，没过。优化 Agent 从报错里找到原因，改成按每个像素在平面上的实际面积来算。下一轮就成功了，冻结后这个任务十个开发布局全过。

### 和以前方法的对比

和以前让语言模型写机器人代码的工作比，区别在于工具不是一次性的，它们跨回合保留，失败一次修一次。按作者的说法，不需要机器人数据，不需要训练，也不改模型。

和学习型策略比，两边的长处几乎互补。VPP2 在推类只有 24.7%，刚体 30.0%，但在插入、可形变和铰接物体上，分别是 28.1%、49.4% 和 21.6%，都比 HexaShell 高。

我的判断是，这篇最有价值的不是榜单第一，而是这张能力地图。它说明，对现在的视觉语言 Agent，看懂场景、记住状态、算出几何已经做得不错，真正卡住的是接触控制。

我的判断是，排名要打个折扣。测试规模比排行榜小得多，而且重测时 Agent 的提示里有任务描述和得分表，25 个任务的优化 Agent 还读过作者写的检查条件说明，条件和学习型策略并不完全一样。

### 局限

论文没有单独的局限一节，下面几条来自正文和附录。第一，全部实验都在模拟器里，没有真机结果。

第二，慢。一个重测回合中位 739 秒，机器人实际动作只有 26 秒，几乎全是模型延迟，RoboDojo 不计这部分时间。

第三，作者事后审计，发现 7 个任务的工具里写死了物体的几何尺寸。换成回合内实测后重测，论文只报告重测后的结果，在开发布局上，这七个任务全是十个里成功零个。

第四，在叠衣服上，工具反而不如一步步边看边改的无工具 Agent，50% 对 72%。作者的解释是，工具提前固定了一种理解和一套动作。

### 未来方向

作者认为，下一步是做出在夹爪拿着零件时还能继续测量、在接触中修正动作的工具，而不是提前把动作规划死。目前演化出来的工具已经在 GitHub 开源。

我的判断是，让一个通用编程 Agent 不经训练就在机器人上一点点积累能力，这个思路很有吸引力。但要真正用起来，还得过真机、延迟和接触控制这三关。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the HexaFuture team. It's called HexaShell, and it plugs an ordinary coding agent straight into a robot.

The question is simple. If a coding agent controls a two-armed robot instead of a computer, how far can it get, and where does it stop?

### Problem

Some background first. Robot manipulation leaderboards have mostly been led by learned policies, such as vision-language-action models, which learn from large sets of demonstrations and output robot actions directly. Agents built on vision-language models have now started to compete.

The hard part is turning pixels into positions. In a code repository, everything the agent touches is text it can read exactly. On a robot, objects are just pixels, but every motion command needs a position in the world, accurate to about a centimetre for a grasp. Vision-language models are bad at judging distances, and the standard observation has no depth image.

### Prior work

Language models writing robot code isn't new. Code as Policies and a long line of follow-ups do it. But as the paper points out, in that work the program is generated per instruction and thrown away, and people write the primitives underneath.

The other route is to use a frontier model directly as the policy. On the RoboDojo benchmark, the same model this paper uses, GPT-6-Astra, reaches twenty-two point five percent success with no tools of its own, choosing gripper targets through a fixed harness written by the benchmark's authors.

And where would depth come from? One option is a monocular depth network, which guesses depth from a single image. But on RoboDojo scenes, the paper measures Depth Anything V2 as off by a median of forty-one millimetres at the head camera, and about a hundred at the wrists, even after fixing its scale with the table.

### Key idea

The inspiration is how coding agents already work. In a repository, the agent reads the code, runs it, reads the error, fixes the code, and the fix stays.

HexaShell runs that same loop on a robot. After each failure, the agent reads the log, finds the cause, and writes or repairs small programs called tools, so what it learns is kept as code. The authors describe this as inverting the old division of labour: the agent writes the primitives, they persist across episodes, and they are the thing being optimized.

The second key choice is triangulation instead of a depth network. If two cameras see the same point, the point sits where their two rays meet. If a point lies on a known plane, like the table or the top of a block, it sits where its ray hits that plane.

### Method

Start with the harness. The acting agent is a stock Codex command-line agent running in a container with no simulator and no API key. It gets one command, called robo, three six-forty by four-eighty RGB images from a head camera and two wrist cameras, and the camera calibration. No depth image, and a manual of about sixty lines.

The base commands are things like move, rotate, choose a gripper orientation, open or close the gripper, and go home. The server only translates. It validates each command, interpolates a straight path, solves inverse kinematics, times the motion within the robot's limits, and sends it to the simulator. The rule is that this layer translates and never decides.

There's also a budget. RoboDojo caps each task at a number of action steps, from two hundred to nineteen hundred. Looking, measuring and model thinking are free. Only executed motion costs steps.

Now the measuring. A pixel plus the calibration gives a ray in space, but a ray isn't a position, so you need one more constraint. The first is the table. A shared tool called table plane sweeps candidate heights, warps one camera's image into another through a plane at each height, and keeps the height where they agree best. The paper reports the three camera pairs agree to within one millimetre.

The second constraint is any other plane with a known height, like the top of a block or the rim of a bowl. The third is a second view. When the head camera is too far, at a bad angle, or blocked, the robot moves a wrist camera to look. That's active perception, and it costs action steps. That's the price of measuring without depth.

The rules are strict. Every number about an object must be measured during the episode, never taken from the benchmark's assets, layouts, configuration or logs. Tools can't import the simulator or read object poses.

Finally, the evolution loop. For each task, it uses the first ten layouts of seed zero, one at a time. The acting agent attempts the layout with the current tools. On a failure, an optimizer agent of the same model reads the command log, the camera frames for every command, and the tool code, writes a diagnosis, and adds or fixes one tool along with its unit tests.

A delivery check then confirms the files are complete, the tools load, and the interface text doesn't name the task or its objects. If it fails, the edit is reverted. The simulator restarts and the same layout runs again, with up to five edits per layout. After ten layouts, the tools are frozen and tested on ten new layouts.

Both agents are Codex with GPT-6-Astra at medium reasoning effort. Each task gets one GPU and a twelve-hour development budget.

### Results

The testbed is RoboDojo: forty-two simulated dual-arm tasks in Isaac Sim, in five groups, generalization, memory, long-horizon, precision and open. The twelve generalization tasks each have a cluttered variant with thirteen to eighteen extra objects.

The headline is Figure 2. On the leaderboard's metric, HexaShell reaches thirty-nine point seven percent success and a progress score of forty-eight, which the authors call a new state of the art on RoboDojo. The top learned policies, VPP2 and PhysicalRSI, sit at thirty-two point three and thirty-one point four.

One thing to be clear on is the test size. HexaShell's numbers come from one episode on each of ten new layouts per task, all from seed zero. The leaderboard entries report fifty episodes on each of three seeds.

By task type, HexaShell leads where the job is reading and remembering the scene. Memory tasks reach sixty-five percent, against at most fifty-five point six for others, and open tasks fifty percent, against at most thirty-one. Learned policies lead where precise contact decides. VPP2 gets thirty-five point two on precision and thirty-seven on long-horizon, against HexaShell's thirty-one point three and twenty-three point eight.

Clutter is the interesting part. On the development layouts, HexaShell keeps ninety-five percent of its success when distractors are added. The thirty learned policies keep a median of eighteen percent, and VPP2 drops from forty-six point four to twelve point six. The authors' explanation: the agent picks a pixel on the target, and the tool measures that one object and nothing else.

So how much does triangulation beat a depth network? That's Table 1. With the same harness, loop, model and layouts, triangulation scores eight point seven points higher on the leaderboard metric. Paired layout by layout, it wins fifty-seven and loses twenty-three.

The cost is in Table 2. With triangulation, three point six percent of executed steps go to moving a camera to look, against zero point seven with the depth network. But it pays off. On the hundred and eighty layouts with such a viewing move, triangulation succeeds thirty-nine times, the depth network twenty-two.

For reference, using the simulator's rendered depth would score another ten point six points higher, with most of that gap in tight insertion.

The most important result is Table 3, the capability map. Grouped by the contact a task needs, pushing, sliding and pressing reach ninety-two point five percent, rigid pick, place and stack fifty, moving objects forty, deformables and fluids thirty-three point three, articulated objects and tool use seven point five, and tight insertion just eight point three, with seven of its twelve variants at zero.

How the episodes fail is telling too. Of four hundred and twenty episodes on the development layouts, two hundred and sixty-eight failed. Leaving out five tool errors, fifty-seven percent ran out of steps, thirty percent ended with the agent declaring success and the checker disagreeing, and thirteen percent were the agent giving up with plenty of budget left.

On deformable, articulated and rigid tasks, most failures exhaust the budget. The agent can see the result is wrong and keeps retrying until the steps run out. In the authors' words, it knows it has failed and can't make the motion work. On memory tasks, half the failures are declared successes that the checker rejects.

The paper also reports what the loop cost. Seven hundred and sixty-nine optimizer rounds in total, with a median of six point three minutes for a round after a failure. What's left is three hundred and twenty-seven task commands in about seventy-eight thousand lines of Python.

One case study: push T. A tool compared the block's area as seen by two cameras, but the head camera's coarse pixels made the area look smaller, so the height measurement kept getting refused. The agent fell back to the table height, fifteen millimetres off, and the block ended thirteen point six millimetres from target, a fail. The optimizer found the cause in the tool's error messages and replaced counting grid cells with summing each pixel's actual footprint on the plane. The next attempt succeeded, and the frozen tools passed all ten development layouts.

### Comparison

Compared with earlier work where language models write robot code, the difference is that the tools aren't disposable. They persist across episodes and get repaired after each failure. According to the authors, no robot data, no training and no change to the model are needed.

Compared with learned policies, the strengths are almost mirror images. VPP2 manages only twenty-four point seven on pushing and thirty on rigid tasks, but on insertion, deformables and articulated objects it scores twenty-eight point one, forty-nine point four and twenty-one point six, all above HexaShell.

My read is that the most valuable thing here isn't the leaderboard spot, it's the capability map. It says that for today's vision-language agents, reading the scene, remembering state and computing geometry already work reasonably well. What actually blocks them is contact control.

My read is also that the ranking deserves a discount. The test is ten layouts per task, one episode each, far smaller than the leaderboard's. And in the retests the agent's prompt includes the task description and score table, while for twenty-five tasks the optimizer read notes the authors wrote on what the checker requires. That's not quite the same footing as the learned policies.

### Limitations

The paper has no separate limitations section, so these come from the main text and appendix. First, every experiment is in simulation. There are no real-robot results.

Second, it's slow. A retest episode takes a median of seven hundred and thirty-nine seconds for twenty-six seconds of robot time, almost all of it model latency, which RoboDojo doesn't charge for.

Third, an audit. The authors found that the tools for seven tasks had fixed object geometry hard-coded, mostly in tools a colleague had written with another coding agent. They replaced those constants with measurements made in the episode and report only the retests, where on the development layouts all seven tasks score zero out of ten.

Fourth, on cloth, the tools actually do worse than an agent without tools correcting step by step: fold clothes reaches fifty percent against seventy-two. The authors' explanation is that a tool commits in advance to one interpretation and one motion.

### Future work

The authors see the next step as tools that keep measuring while the gripper holds the part and correct the motion during contact, instead of planning it all in advance. The tools evolved so far are open source on GitHub.

My read is that this is an appealing recipe: a general coding agent, with no training at all, steadily building up skill on a robot. But to be useful in practice, it still has to clear three hurdles: real hardware, latency and contact control.

That's today's Daily Frontier Papers. The paper is linked in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
