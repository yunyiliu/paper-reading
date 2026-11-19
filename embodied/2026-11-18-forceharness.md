# ForceHarness: Turning Force into Action and Physics for Frozen Frontier Models

**不训练不给示范大模型用上力觉**

- 论文 Paper: [arXiv 2611.21099](https://arxiv.org/abs/2611.21099)
- 单位 Affiliation: 新加坡国立大学 Show Lab
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-18
- 视频 Videos: [YouTube 中文](https://youtu.be/oljBKCAo6Ck) · [B站](https://www.bilibili.com/video/BV1F8sh6nE11)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇新加坡国立大学 Show Lab 的论文，叫 ForceHarness，意思是给冻结的前沿大模型，配一套能用上力觉的外壳。

先说结论。在一台手腕上没有力传感器的真实机械臂上，它让 GPT-6 Astra 零样本的成功率，从 3.3% 提到了 90%。

### 问题

先说背景。现在的前沿多模态大模型，已经可以不经过专门训练，直接看图片控制机器人。论文里的例子是 GPT-6 Astra，它只通过 API 调用，权重是冻结的。

但很多操作，靠的是图片看不到的接触力。比如拧螺丝，要一边往下压一边转，直到螺纹咬合。再比如几个长得一样的瓶子，要按重量排序，光看是看不出来的。

把力交给一个只能通过 API 调用的模型，有两个难点。第一，接触变化得太快，模型想一次要好几秒，来不及，所以力的闭环，必须在模型下面完成。

第二，冻结的模型读不懂原始的力信号，一串传感器数字，它没法直接拿来做决定，所以力得换成模型能推理的形式，再交给它。

### 以前的方法

以前把力用进机器人，主要靠学习。比如在视觉运动策略，或者 VLA，也就是视觉语言动作模型里，把手腕上的力作为一路输入，加进网络。

问题是，这要为每个任务收集示范数据，还要更新模型权重，所以没法用在一个只能调用 API 的冻结模型上。

另一类，是给冻结模型做外壳的工作，这种外壳叫 harness。它们大多帮模型看和动，碰到接触时，一般是把原始的力和触觉数值直接给模型，或者让模型自己写力控代码。

这样一来，模型还是得自己解析原始的传感器数字，或者自己写底层控制，拿不到校准过的物理反馈。

### 核心想法

这篇的思路是分工。模型决定做什么、压多重，底层的技能决定什么时候停，下面还有一道固定的安全护栏，设定模型改不了的上限。

具体来说，ForceHarness 在模型和机器人之间放了三层。最底层估计并校准手上受的力，中间是一组会自己停下的力控技能，最上层把力的曲线翻译成物理量，作为工具结果还给模型。

### 方法

先说最底层。很多机械臂手腕上没有力传感器，这一层可以用关节力矩反推手上受的力，做法是用最小二乘求解，再减掉零点。

这种估计噪声小，但零点会随姿态慢慢漂。作者的办法不是更好的滤波，而是控制什么时候读数，每次空手动作前，在静止时重新归零，动作完先让手臂稳住再读，而且只报告相对本回合参照的变化。

中间这层是力控技能，论文叫自终止力控原语，一共 12 个，分成五组，分别负责接触、保持力、找孔和插入、带负载旋转，以及测量。另外还有 10 个工具，负责移动、抓取、感知和结束回合。

大多数机械臂只接受位置指令，不接受力指令。所以原语的做法，是让目标位置稍微领先实际位置，控制器像一根弹簧，把这段差距变成力，差距有上限，力也就有上限。

要在某个方向上保持一个力，就把力的误差换成目标位置的移动速度，这叫导纳控制。所有原语都跑在同一个循环里，区别只是这条控制律不一样。

关键是自终止。原语不是按时间结束，而是一碰到接触、卡住、阻力突然下降这类事件就停，力超过上限或者超时也会停。停下以后，手保持不动，等模型想好下一步。

举个例子，找孔用的 search。模型只大概知道孔在哪，它决定压多重、搜多大范围、要插多深。原语先碰到表面，然后保持压力，沿着一条螺旋线往外走。

一旦零件经过孔的上方，阻力下降，螺旋暂停，同一条控制律把零件压进去。最后报告有没有掉进孔里，前进了多少，末端的侧向力是多少。这张图就是真机插 USB 的一次记录，压力 1 牛，插入前进了 7.8 毫米。

最上层是物理翻译层。一次原语会记录几百个力的采样，模型很难直接用。这一层把它们换成几个物理量，比如负载质量、摩擦系数、接触高度，每个都带误差范围和警告标志。

这些估计故意做得很简单，就是平均和最小二乘拟合。而且这一层没有任何任务阈值，它只说东西有多重、插头进了多深，至于够不够，由模型自己判断。

整个流程是这样的。模型先在图片里指出目标，用深度把像素换成三维点，把手移过去，然后切换到力控原语，按看到和读到的信息设参数，再根据返回的物理量决定下一步，直到结束。

### 实验和效果

实验分三块，真机、公开的 ForceBench，和作者自建的 HiddenPhys。每一块都拿同一个模型，在有力和没力两种条件下对比，没力的对照保留外壳和安全限制，只去掉力控原语和力的读数。

真机是一台 Franka FR3，手腕上没有力传感器，力完全从关节力矩估出来。六个任务，擦白板、拧螺丝、削茄子皮、插 USB、按重量给三个瓶子排序，还有分辨软硬，每个条件跑 10 次。

结果在表 3。GPT-6 Astra 没有力的时候，60 次只成功 2 次，加上 ForceHarness 是 54 次。换成 GPT-6.1 Sol，用同样的提示词，是 47 次。

排瓶子和分辨软硬这两个任务，RGB-D 图像里看不出重量和软硬，没力的对照都是 0 次。用了 ForceHarness，Astra 在这两个任务上都是 10 次全对。

论文还记录了模型怎么根据力来调整。擦白板时，8 牛擦完还有残留，Astra 就把压力提到 10 牛再擦。拧螺丝时侧向力上升，它就在卡死之前原地松手。

第二块是 ForceBench，一个公开的 Isaac Lab 仿真套件，有八个接触任务，作者跑了前 10 个种子。Astra 从没力的 70% 提到 87.5%，Sol 也是 87.5%，Claude Fable 5.2 是 75%。论文列出的最好的训练策略是 35.31%。

不过作者自己说明，这个差距不全是力带来的，因为没有力的时候，Astra 和 Sol 已经有 70% 和 65%。力带来的提升，主要在拉抽屉这个任务上，从 10 次全失败变成 10 次全成功。

第三块是 HiddenPhys，七个 MuJoCo 任务，每个任务藏着一个决定成败的物理变量，比如方块的质量、桌面的高度、孔的位置，指令只给范围，不给具体值。

结果在表 5。Astra 和 Fable 加上 ForceHarness，140 次都成功了 135 次，把手损坏是零，和手写的专家力控制器的 136 次基本持平。

用每个任务大约 200 条示范训练的策略，包括 Diffusion Policy、π0.5 和 ForceVLA，只成功 65 到 83 次，而且经常损坏把手。给它们加上手腕力输入，也没有稳定的提升。作者的解释是，被动地把力当输入，既学不会主动去试探，也不会限制力度。

在同一个外壳里，加上力之后，Astra 从 87 次提到 135 次，Fable 从 96 次提到 135 次，Astra 每个回合的成本还降了 45%。

### 和以前方法的对比

收益从哪来，看表 6 的消融。只保留原语，只报告接触事件和事件发生时的力的大小，不给力的曲线，也不给物理量，Sol 的成功率从 0.63 升到 0.77，更小的 GPT-5.x 模型从 0.18 升到 0.54，小模型在 Lock 任务上损坏把手，也从 25 次里 22 次降到 3 次。

力怎么呈现给模型，也有影响。对 Sol 来说，给力的曲线图、原始数字还是算好的物理量，成功率都在 0.87 到 0.91 之间。但对小模型，曲线图只有 0.56，带误差的物理量是 0.70，判断对错的准确率，原始数字是 0.69，物理量是 0.85。

还有一组，是和经典的 Code as Policies 比，也就是让模型一次性写完整个程序。用上这套力接口，一次性程序从 23% 提到 77%，而一步一步调用、看了结果再决定，能到 91%。

我的判断是，这篇最有价值的地方，是把力控里快的部分和慢的部分拆开了，快的闭环留给原语，慢的判断交给大模型，而且全程不需要示范数据，也不改模型权重。

我的判断是，和训练策略的比较要谨慎看。ForceBench 上训练策略的数字是从原论文摘的，没有复现，作者的工具和阈值，又是在十个种子里的八个上调的。HiddenPhys 是作者自己建的，大模型读的是带噪声的物体位置文本，不是图像。

### 局限

作者列了几条局限。第一，每次查询远端模型要多花几秒，适合准静态的接触任务，做不了动态运动里的快速反应。

第二，原语菜单和物理估计器都是手工设计的，每个任务的指令也是手写的。第三，评测只在单臂、桌面、平行夹爪上，模型来自两家 API 提供商，各个组件的消融只在仿真里做了，没有在真机上重复。

### 未来方向

未来方向，作者提到要自动生成新的原语，以及更开放的任务说明，不再依赖手写指令。代码和配置，论文说会在发表时公开。

我的判断是，这条路线如果走通，做机器人不一定要为每个接触任务收数据训练，把力控技能和物理量接口做好，就能接上更新的大模型。不过真机只有六个任务，每个 10 次，离可靠的实际部署还有距离。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Show Lab at the National University of Singapore. It's called ForceHarness, and it gives a frozen frontier model a way to use the sense of force.

Here's the headline result. On a real robot arm with no force sensor in the wrist, it lifts GPT-6 Astra's zero-shot success rate from three point three percent to ninety percent.

### Problem

Some context first. Frontier multimodal models can now drive a robot straight from camera images, with no task-specific training. The paper's example is GPT-6 Astra, which sits behind an API with frozen weights.

But a lot of manipulation depends on contact forces that a camera simply can't see. Driving a screw means pressing down while you turn until the thread catches. Sorting identical-looking bottles by weight is impossible by eye.

Giving force to an API-only model runs into two problems. The first is speed. Contact changes far faster than a model that takes seconds per decision can react, so the force loop has to close underneath the model.

The second is understanding. A frozen model can't make sense of a raw stream of sensor numbers, so force has to come back in a form the model can actually reason about.

### Prior work

Until now, force has mostly entered robotics through learning. Visuomotor policies and vision-language-action models, or VLAs, take wrist force as one more input to the network.

The catch is that this needs demonstrations for every task and changes the model's weights. So it can't be applied to a frozen model you can only reach through an API.

The other line of work builds harnesses, wrappers around a frozen model. Most of them help the model see and move. When contact comes up, they usually hand the model raw force and tactile values, or ask it to write force-control code itself.

Either way, the model is still parsing raw numbers or writing low-level control, with no calibrated physical feedback to go on.

### Key idea

The core idea is a division of labor. The model decides what to do and how hard to press. A low-level skill decides when to stop. And a fixed guard underneath sets limits the model can't change.

Concretely, ForceHarness puts three layers between the model and the robot. The bottom layer estimates and calibrates the force on the hand. The middle layer is a set of force skills that stop on their own. The top layer turns each force trace into physical quantities and returns them to the model as the tool result.

### Method

Start at the bottom. Many arms have no wrist force sensor, so this layer can recover hand force from the joint torques, solving a least-squares problem and then subtracting a zero point.

That estimate is low-noise, but its zero point drifts slowly as the arm changes pose. Rather than filtering harder, the authors control when they read. They re-zero at rest before each empty-hand action, let the arm settle before reading, and report only changes relative to a reference from the same episode.

The middle layer is what the paper calls self-terminating force primitives. There are twelve, in five groups: making contact, holding force, finding and seating a part, turning under load, and measuring. Ten more tools handle moving, grasping, perceiving, and ending the episode.

Most arms take position targets, not force commands. So a primitive lets the target pose run slightly ahead of the actual pose, and the controller acts like a spring that turns that gap into force. The gap is capped, so the force is capped too.

To hold a force along a direction, the force error is turned into a velocity for the target. That's admittance control. Every primitive runs in the same loop and differs only in this small control law.

The key part is self-termination. A primitive doesn't stop on a timer. It stops on an event, like contact, a stall, or a sudden drop in resistance, with a force cap and a time limit as backstops. Then the hand holds still while the model thinks.

Take search, the primitive for finding a hole. The model only roughly knows where the hole is, and it picks how hard to press, how wide to search, and how deep the part must sink. The primitive touches the surface, keeps pressing, and moves outward along a spiral.

When the part passes over the hole, resistance drops, the spiral pauses, and the same control law pushes the part down. It reports whether the part dropped in, how far it advanced, and the side force at the end. This figure is a logged USB insertion on the real robot, pressing at one newton, with the plug advancing seven point eight millimeters.

The top layer is the physics translation layer. A primitive logs hundreds of force samples, which give a model little to work with. This layer turns them into a few physical quantities, like payload mass, friction, and contact height, each with an uncertainty and warning flags.

The estimators are deliberately simple: averages and least-squares fits. And the layer has no task thresholds. It says how heavy a load is or how far a plug went in, but never whether that's enough. That call is left to the model.

So the loop goes like this. The model points at a target in the image, depth turns that pixel into a 3D point, and the hand moves close. Then the model switches to force primitives, sets their parameters from what it has seen and read, and uses the returned quantities to pick the next call until it ends the episode.

### Results

There are three test settings: a real robot, the public ForceBench suite, and the authors' own HiddenPhys benchmark. In each, the same model runs with force and without. The no-force control keeps the harness and its safety limits but drops the force primitives and force readings.

The real robot is a Franka FR3 with no wrist force sensor, so every force reading comes from joint torques. There are six tasks: wiping a whiteboard, driving a screw, peeling an eggplant, inserting a USB plug, sorting three bottles by weight, and telling soft from hard. Each condition gets ten episodes per task.

The results are in Table three. Without force, GPT-6 Astra succeeds in two of sixty episodes. With ForceHarness, fifty-four. GPT-6.1 Sol, with the same prompts, gets forty-seven.

On bottle sorting and soft versus hard, the RGB-D images carry no clue about mass or stiffness, so the no-force control scores zero on both. With ForceHarness, Astra goes ten for ten on each.

The paper also logs how the model adapts. After an eight newton wipe leaves ink behind, Astra retries at ten newtons. When side load rises while driving a screw, it releases in place before the screw binds.

Second, ForceBench, a public Isaac Lab suite of eight contact-rich tasks, run on its first ten seeds. Astra goes from seventy percent without force to eighty-seven point five. Sol also reaches eighty-seven point five, and Claude Fable five point two reaches seventy-five. The best trained policy the paper lists scores thirty-five point three one.

The authors are upfront that this gap isn't all from force, since Astra and Sol already score seventy and sixty-five percent without it. The force gain shows up mainly on Drawer Pull, which goes from zero out of ten to ten out of ten.

Third, HiddenPhys: seven MuJoCo tasks, each hiding one physical variable that decides success, like a cube's mass, the table height, or where the hole is. The instruction gives the range, never the value.

The results are in Table five. With ForceHarness, both Astra and Fable succeed in a hundred thirty-five of a hundred forty episodes with zero handle damage, essentially matching the hand-written expert force controller at a hundred thirty-six.

Policies trained on about two hundred demonstrations per task, Diffusion Policy, pi zero point five, and ForceVLA, manage only sixty-five to eighty-three, with frequent damage. Adding wrist force as an input brings no consistent gain. The authors' explanation is that passively conditioning on force neither teaches active probing nor caps the force.

Inside the same harness, adding force takes Astra from eighty-seven to a hundred thirty-five, and Fable from ninety-six to a hundred thirty-five. It also cuts Astra's cost per episode by forty-five percent.

### Comparison

Where do the gains come from? Table six has the ablations. Keep the primitives but report only contact events, with the force magnitude at each event and no force traces or quantities, and Sol's success rises from point six three to point seven seven. Smaller GPT-5.x models go from point one eight to point five four, and their damaged Lock handles drop from twenty-two of twenty-five episodes to three.

How force is shown to the model matters too. For Sol, force plots, raw numbers, and computed quantities all land between point eight seven and point nine one. Smaller models are different. Plots get only point five six, quantities with uncertainty get point seven zero, and decision accuracy rises from point six nine with raw numbers to point eight five with quantities.

There's also a comparison with classic Code as Policies, where the model writes the whole program in one shot. With this force interface, one-shot programs go from twenty-three to seventy-seven percent, while calling step by step and reading each result reaches ninety-one.

My read is that the real contribution is splitting force control into a fast part and a slow part. The fast loop lives in the primitives, the slow judgment goes to the model, and none of it needs demonstrations or weight updates.

My read is also that the comparison with trained policies deserves care. The ForceBench trained numbers are copied from the original paper, not reproduced, and the authors tuned their tools and thresholds on eight of the ten seeds. HiddenPhys is the authors' own benchmark, and there the model reads noisy object poses as text, not images.

### Limitations

The authors list several limitations. First, each query to a remote model adds seconds, which suits slow, quasi-static contact tasks but rules out fast reactions during dynamic motion.

Second, the primitive menu and the physics estimators are designed by hand, and so is each task's instruction. Third, testing covers only single-arm tabletop work with a parallel gripper and models from two API providers, and the component ablations were run in simulation, not repeated on hardware.

### Future work

Looking ahead, the authors point to synthesizing new primitives automatically and to more open-ended task specification, without hand-written instructions. They say code and configurations will be released on publication.

My read is that if this approach holds up, you may not need to collect data and train for every contact task. Build good force skills and a physical-quantity interface, and you can plug in newer models as they arrive. But the real-robot test is six tasks at ten episodes each, which is still a long way from dependable deployment.

That's today's Daily Frontier Papers. The paper is linked in the description.

