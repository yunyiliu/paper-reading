# RoboArchitect: Automating the Design of Robotic Agents with Persistent Memory

**不改权重成功率提到4.5倍**

- 论文 Paper: [arXiv 2610.42141](https://arxiv.org/abs/2610.42141)
- 单位 Affiliation: 斯坦福大学
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-04
- 视频 Videos: [YouTube 中文](https://youtu.be/J_YnAYdjxFI) · [YouTube English](https://youtu.be/nCDAELUsQ98) · [B站](https://www.bilibili.com/video/BV1xAxQ6pE9h)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇斯坦福大学的论文，叫 RoboArchitect，意思是机器人的架构师。它问的问题是，套在机器人模型外面的那层控制代码，能不能让 AI 自己来写。

先说结论。不改模型的任何一个权重，只让编码模型一轮一轮地改外面那层代码，在没见过的场景里，平均成功率从 9.7% 提到 43.3%，在没见过的任务上，从 12.7% 提到 40.2%。

### 问题

先解释两个词。VLA，是视觉、语言、动作模型，它看摄像头画面，听一句指令，直接输出机械臂的动作。harness，可以理解成套在模型外面的一层控制代码，决定什么时候问模型，什么时候调用视觉工具，什么时候直接指挥机械臂。

一个现成的 VLA，换到新的机器人、新的任务上，往往表现很差。常规办法是收集新的示范数据，再微调模型权重，成本很高。

最近的一些工作发现，把冻结的模型套进一个好的 harness 里，效果会好很多。问题是，这些 harness 都是人手写的，每换一个任务、一种机器人，就得花大量工程重新设计。

### 以前的方法

人写 harness，是一个熟悉的试错循环，让机器人跑，看哪里失败，找原因，改控制代码，再跑一遍。这很慢，而且机器人的录像很长，工程师很难一帧一帧看完。

相关工作大致有几类。一类像 VoLo 和 Harness VLA，用人写好的 harness 包住冻结的策略。另一类是 code-as-policy，用程序直接控制机器人，但有研究发现，它很依赖人事先写好的基础动作。

还有一类，用大模型自动做优化，比如自动写奖励函数，但它们最后还是拿来训练权重。RoboArchitect 不一样，它不碰权重，自动设计的是围着冻结策略的那层代码。

### 核心想法

核心想法就一句话，把工程师的那个试错循环，从看失败记录、诊断原因，到改代码，整个交给一个编码模型来做。

作者的主张是，优化 harness 可以替代微调，把一个通用策略，变成某个具体机器人场景里的专才。

### 方法

这张图是整个系统。左边是 harness，一开始它只是个直通，把观测交给 VLA，再把 VLA 的动作原样输出，也就是单独用策略的效果。

harness 每一步拿到摄像头画面、深度图、夹爪的位姿和开合状态，还有一句任务指令，输出底层的动作向量，交给机器人执行。

除了 VLA，它还能调用语言模型、视觉语言模型、物体检测、分割、指点和抓取位姿生成这些辅助工具，以及把像素和深度换算成三维坐标的几何工具。

优化循环是这样的。每次评测都记录完整轨迹，包括每一步的画面、机器人读数和所有工具的输出。编码模型看这些记录，诊断失败原因，写出新一版 harness，它可以跟着策略走，可以调用工具，也可以直接用程序指挥机械臂。

新版本在同一批训练回合上重新评测，成绩更好才接受，所以训练集上只升不降，最多迭代 5 轮。

这里的训练只用来优化 harness，不更新任何权重，最后的成绩都在单独留出的回合上测。

还有一个设计，叫持久记忆。harness 可以在回合之间、任务之间保存信息。作者没有规定存什么、怎么存、怎么取，全部由优化过程自己决定。

留出评测有两种。一种叫没见过的场景，任务相同，但物体摆放和布局不同。另一种叫没见过的任务，换成不同的任务，但需要类似的操作技能。

实验用了五个仿真基准，比如 LIBERO-Pro-10、VLABench 和 RoboCasa-365，还有七个 VLA 策略，比如 OpenVLA-7B 和 X-VLA。编码模型用的是 Claude Opus 5，通过 Claude Code 以最高推理强度调用。

成本方面，论文报告，一次完整的 5 轮优化，平均不到 6 个小时的编码模型时间，花费不到 150 美元，不需要收集示范数据。

### 实验和效果

总体结果在表 1。在没见过的场景上，平均成功率从单独用策略的 9.7%，提到 43.3%，是 4.5 倍。在没见过的任务上，从 12.7% 提到 40.2%，是 3.2 倍。

提升最大的，是策略离自己训练数据最远的情况。只看跨基准和域外的设置，没见过的场景平均从 7.3% 提到 58.4%。

最有意思的是和微调的对比。在 VLABench 上，通用的 X-VLA 单独跑只有 7.3%，套上进化出来的 harness 到了 52.6%，而在这个基准上专门微调过的 X-VLA 是 35.2%。没见过的任务上，harness 是 72.5%，微调版是 32.1%。要注意，VLABench 用的是官方进度分，完成一部分步骤也会给分。

在 SimplerEnv WidowX 上，没见过的场景，harness 是 87.5%，微调版是 70.8%。但没见过的任务上，harness 是 66.7%，不如微调版的 87.5%，这是对比里唯一输给微调的地方。

也有没提升的。RoboCasa-365 上的 RLDX-1，本来就是在这个基准上训练的，没见过的场景从 27.9% 到 27.8%，基本没变，它只在优化用的回合上有进步。

再看 harness 和策略怎么分工。通用的 X-VLA 在 WidowX 上，每一步都让夹爪完全张开，等于没用。harness 就学会了完全不理它，自己检测物体，用深度估计位置，算抓取方向，变成了一个 code-as-policy 智能体。

反过来，如果策略是微调过的专才，harness 就让它主导，只在它失败时接管。比如回合后期夹爪还没靠近目标，harness 就用指点模型接手，策略快到了，就把控制权还给它。作者强调，他们没告诉编码模型该怎么分工。

这张图是几个进化出来的控制套路。第一行，策略推着机械臂一直往果汁盒上撞，直到把它推倒，新的 harness 会发现手臂卡住了，先把手臂抬开。

第二行，从书架上取书，原来的 harness 总是从上往下够，撞到隔板，新的 harness 用深度图发现上方有遮挡，转动手腕，从侧面伸进去。

第三行，关烤箱门，策略推得太轻，门关不上，harness 保持方向，放大动作幅度，把门关紧。

第四行，策略记住了原来训练数据里摩卡壶的位置，伸手去抓那里的煎锅，harness 改成先找到指令里说的摩卡壶。

持久记忆也是自己长出来的。比如在 WidowX 上，记忆存的是，在物体高度的多少比例处合拢手指，能把东西抬起来，18% 抬不起胡萝卜，45% 胡萝卜和茄子都能抬起来。

还有的记忆，存每种物体用哪种抓法管用，存哪些指令需要握住东西，甚至发现 RoboCerebra 每 150 步重置一次场景。表 3 显示，留出回合里每个回合会读 3.4 到 13 次记忆，超过 90% 的读取能找到存好的条目。

迭代也很关键。只做一轮设计，也就是 one-shot，没见过的场景平均是 21.4%，迭代到 5 轮是 43.3%。有些一轮的设计甚至不如单独用策略，迭代之后，几乎在每个留出集上都超过了。

最后是一个跨机器人的统一 harness，在程序生成的回合上优化，在基准的标准回合上测试，覆盖 Franka、WidowX 和谷歌的移动机械臂。包住通用 X-VLA 时，留出成绩从 1.9% 提到 28.8%。包住微调过的专才时，从 45.9% 到 47%，基本保住了原来的水平。

这个统一 harness 还学会了按机器人调节。原来每个动作按固定时间结束，在只有 3 赫兹的谷歌机器人上，时间到了手臂还没到位，后来改成到位才结束。包含这项改动的那一轮修改，把这个机器人在训练池上的成功率从 11% 提到了 30.2%。

### 和以前方法的对比

和手写 harness 的工作比，RoboArchitect 的区别在于，harness 的结构、策略和代码怎么分工、记忆长什么样，全部由优化过程决定，而不是事先定死。

和微调比，它不需要示范数据，也不改权重，而且每次修改都是能读懂的代码。

我的判断是，这篇最有价值的发现，是 VLA 和 code-as-policy 不必二选一。策略好用就用，不好用就由代码接管，这条边界可以让优化过程自己去找。

我的判断是，也要看到短板。论文的表里，没有和 VoLo、Harness VLA 这类手写 harness 直接比较，而且每个任务的试验次数不多，LIBERO-Pro-10 和 VLABench 每个任务只跑 10 次，单个数字可能有不小的波动。

### 局限

作者自己列了三条局限。第一，最多只迭代 5 轮，再多几轮会不会更好，什么时候饱和，还没有研究。

第二，还没扩展到接触更复杂的任务，比如灵巧手的手内操作和使用工具，这类任务可能出错的方式比抓取放置多得多。

第三，也是最重要的一条，所有评测都在仿真里。作者认为 harness 只读摄像头、深度和机器人自身读数这些标准输入，接口应该能搬到真机上，但真机测试还是未来的工作。

### 未来方向

未来方向，作者提到两件事，一是在更复杂的操作任务上，看自动进化能发现什么样的控制套路和记忆结构，二是在真机部署时，还需要多少轮改进。

我的判断是，这篇说清楚了一件事，机器人操作的不少提升空间，在模型外面那层代码里，而且这层代码可以自动写。如果真机上也成立，给新机器人做适配，可能会从先采数据再微调，变成先让编码模型改几轮 harness。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Stanford University, and it's called RoboArchitect. The question behind it: can an AI write the control code that wraps around a robot model, instead of an engineer?

Here's the headline. Without changing a single model weight, just by letting a coding model revise that outer code round after round, average success on unseen scenes goes from nine point seven percent to forty-three point three. On unseen tasks, it goes from twelve point seven to forty point two.

### Problem

Two terms first. A VLA, or vision-language-action model, looks at camera images, reads an instruction, and outputs robot arm actions directly. A harness is the layer of control code around the model. It decides when to ask the model, when to call a vision tool, and when to drive the arm itself.

Take an off-the-shelf VLA to a new robot or a new task, and it often does badly. The usual fix is to collect new demonstrations and fine-tune the weights, which is expensive.

Recent work found that a frozen model inside a good harness does much better. The problem is that those harnesses are written by hand, and every new task or robot needs substantial engineering to redesign them.

### Prior work

Building a harness by hand is a familiar trial-and-error loop. Run the robot, see where it fails, figure out why, change the control code, run it again. It's slow, and robot recordings are long, so they're hard for an engineer to inspect frame by frame.

Related work falls into a few camps. Systems like VoLo and Harness VLA wrap a frozen policy in a hand-written harness. Code-as-policy controls the robot with programs, but one study found it depends heavily on the hand-written primitives it's given.

Another camp uses language models to automate optimization, for example writing reward functions, but the end product still trains weights. RoboArchitect is different. It never touches the weights. What it designs automatically is the code around the frozen policy.

### Key idea

The key idea fits in one line: hand the engineer's whole trial-and-error loop, from reading failure traces to diagnosing them to rewriting the code, to a coding model.

The authors' claim is that harness optimization can stand in for fine-tuning, turning a general policy into a specialist for one particular robot setup.

### Method

This figure shows the system. On the left is the harness. It starts as a pass-through: observations go to the VLA, and the VLA's actions go straight out. That's the bare policy.

At every step the harness gets camera images, depth maps, the gripper's pose and status, and the task instruction, and it outputs a low-level action vector for the robot to execute.

Besides the VLA, it can call helper tools: language and vision-language models, detection, segmentation, pointing and grasp-pose generation, plus geometry utilities that turn a pixel and its depth into 3D coordinates.

The optimization loop works like this. Every evaluation logs the full trajectory: camera frames at each step, the robot's readings, and every model and tool output. The coding model reads these traces, diagnoses what went wrong, and proposes a revised harness. The revision can follow the policy, call tools, or command the arm directly in code.

The candidate is re-run on the same training episodes and accepted only if it beats the current harness, so training performance never goes down. There are at most five iterations.

Training here means optimizing the harness and nothing else. No weights are updated, and all the reported scores come from separate, held-out episodes.

One more piece: persistent memory. The harness can store information across episodes and tasks. The authors don't specify what to store, in what form, or how to retrieve it. The optimization decides all of that.

There are two held-out tests. Unseen scenes keeps the tasks the same but changes object placement and layout. Unseen tasks switches to different tasks that need similar manipulation skills.

The experiments use five simulated benchmarks, among them LIBERO-Pro-10, VLABench and RoboCasa-365, and seven VLA policies, among them OpenVLA-7B and X-VLA. The coding model is Claude Opus 5, run through Claude Code at maximum effort.

On cost, the paper reports that a full five-iteration run takes under six hours of coding-model time and under one hundred fifty dollars on average, with no demonstrations collected.

### Results

The overall numbers are in Table 1. On unseen scenes, average success goes from nine point seven percent for the bare policy to forty-three point three, about four and a half times. On unseen tasks, from twelve point seven to forty point two, about three point two times.

The biggest gains come when the policy is furthest from its training data. Looking only at the cross-benchmark and out-of-domain setups, unseen-scene success goes from seven point three to fifty-eight point four.

The most interesting comparison is against fine-tuning. On VLABench, the general X-VLA alone scores seven point three. Inside the evolved harness it reaches fifty-two point six, while X-VLA fine-tuned on that benchmark gets thirty-five point two. On unseen tasks, it's seventy-two point five for the harness versus thirty-two point one fine-tuned. One caveat: VLABench uses its official progress score, which gives partial credit.

On SimplerEnv WidowX unseen scenes, the harness scores eighty-seven point five against seventy point eight for the fine-tuned model. On unseen tasks, though, the harness gets sixty-six point seven and the fine-tuned model eighty-seven point five. That's the one place in these comparisons where fine-tuning wins.

Not everything improves. RLDX-1 on RoboCasa-365 was already trained on that benchmark, and on unseen scenes it goes from twenty-seven point nine to twenty-seven point eight, essentially flat. It improved only on the episodes used for optimization.

Now, how do the harness and the policy split the work? On WidowX, the general X-VLA commands a fully open gripper at every step, so it's useless. The harness learns to ignore it entirely. It detects objects, estimates their position from depth, computes a grasp, and becomes a code-as-policy agent.

The opposite happens with a fine-tuned specialist. The harness lets the policy lead and steps in only when it fails. If, late in an episode, the gripper still hasn't approached the target, the harness takes over using the pointing model, and if the policy is already closing in, it hands control back. The authors stress that they never told the coding model how to divide control.

This figure shows some of the control routines that evolved. In the first row, the policy keeps driving the arm into a juice carton until it tips over. The new harness detects the jam and lifts the arm clear.

In the second row, the task is to take a book off a shelf. The earlier harness always comes in from above and hits the shelf board. The evolved one spots the obstruction in the depth map, turns the wrist and reaches in from the side.

In the third row, closing a toaster oven door, the policy pushes too gently and the door stays open. The harness keeps the direction but scales up the action, and the door latches shut.

In the fourth row, the policy has memorized where the moka pot sat in its original training data, so it grabs the frying pan that's there now. The harness instead finds the moka pot named in the instruction.

Persistent memory evolved on its own too. On WidowX, for example, the memory records at what fraction of an object's height the fingers should close to lift it. Closing at eighteen percent lifts nothing on the carrot. Closing at forty-five percent lifts both the carrot and the eggplant.

Other memories record which grip style works for each object, which instructions require holding something, and even that RoboCerebra resets its scene every one hundred fifty steps. Table 3 shows these memories are read between three point four and thirteen times per held-out episode, and over ninety percent of reads find a stored entry.

Iteration matters. A single design attempt, the one-shot version, averages twenty-one point four percent on unseen scenes, against forty-three point three after up to five rounds. Some one-shot designs even fall below the bare policy. After iteration, the harness beats the bare policy on nearly every held-out set.

Finally, a single unified harness across robots. It's optimized on procedurally generated episodes and tested on the benchmarks' standard ones, across a Franka arm, a WidowX and Google's mobile manipulator. Wrapped around the general X-VLA, held-out success goes from one point nine to twenty-eight point eight. Wrapped around fine-tuned specialists, it goes from forty-five point nine to forty-seven, essentially preserving what they could already do.

This unified harness also learns to adapt to each robot. Each move used to end after a fixed time, which on the slower three-hertz Google robot ran out before the arm arrived. The harness learned to end a move on arrival instead. The revision that included this change raised that robot's success on the training pool from eleven percent to thirty point two.

### Comparison

Compared with hand-built harnesses, the difference is that the structure, the split between policy and code, and the shape of the memory are all decided by the optimization, not fixed in advance.

Compared with fine-tuning, it needs no demonstrations and changes no weights, and every revision is code you can read.

My read is that the most valuable finding here is that VLAs and code-as-policy don't have to be an either-or. Use the policy where it works, let code take over where it doesn't, and let the optimization find that boundary.

My read is also that there's a gap. The paper's tables don't compare directly against hand-written harnesses like VoLo or Harness VLA, and trial counts are small: ten per task on LIBERO-Pro-10 and VLABench, so individual numbers could swing quite a bit.

### Limitations

The authors list three limitations. First, they cap the loop at five iterations. Whether more would help, and where returns flatten out, is still open.

Second, they haven't tried tasks with richer contact, like dexterous in-hand manipulation or tool use, where there are many more ways to fail than in pick-and-place.

Third, and most important, every evaluation is in simulation. The authors argue the harness reads only standard inputs, camera, depth and the robot's own readings, so the interface should carry over to real robots. But real-robot testing is future work.

### Future work

Looking ahead, the authors name two things: seeing what control routines and memory structures automated evolution finds on harder manipulation tasks, and how many more improvement rounds a real deployment would need.

My read is that this paper makes one point very clearly. A lot of the headroom in robot manipulation sits in the code around the model, and that code can be written automatically. If that holds on real robots, adapting to a new robot might shift from collecting data and fine-tuning to letting a coding model revise the harness a few times.

That's today's Daily Frontier Papers. The paper is linked in the description.

