# Embodied AI Fails the Mirror Test: How a Simple Mirror Breaks Robot Policies

**一面镜子让机器人失灵**

- 论文 Paper: [arXiv 2611.33010](https://arxiv.org/abs/2611.33010)
- 单位 Affiliation: 北京大学
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-27
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1f11xziEhk)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇北京大学的论文，标题很直接，具身智能没通过镜子测试，一面普通的镜子，就能让机器人策略失灵。

作者在仿真里，往机器人干活的桌子旁边放镜子，测了五个不同路线的机器人策略。结果每一种镜子设置，都让它们的成功率下降，三种设置平均下来，掉了 10.8 到 56.7 个百分点。

### 问题

灵感来自动物认知研究里的镜子测试，看动物能不能借助镜子里的影像，对自己的身体做出动作。这篇论文换了个问法，当机器人看到的画面里，同时有物体和它的倒影，它还能不能准确地操作真实的物体。

镜子有个很特别的地方。倒影长得和真东西一模一样，还会跟着真东西动，但倒影所在的位置，什么也抓不到。倒影还可能在画面里和真物体连成一个整体，或者让被挡住的东西露出来。

### 以前的方法

以前的鲁棒性测试，已经发现视角、光照、背景这些变化，会让机器人策略的表现大幅下降。但镜子不在这些测试里。它不是改变外观，而是把和任务有关的东西，在另一个位置原样复制了一份。

关于镜子也有不少研究，比如在图里找出镜子在哪，或者测试 Agent 能不能在镜子里认出自己。这篇关心的是，通用机器人策略在有镜子的场景里，操作外部的物体。

### 核心想法

思路很简单。任务不变，评分标准不变，模型权重也不变，只在场景里加镜子，然后每个策略都和它自己没有镜子时的成绩做配对比较。

但镜子同时也是一块实实在在的板子，机器人可能直接撞上去。所以作者加了一个关键对照，把镜子换成形状和位置完全一样、但不反光的板子。如果镜子造成的失败比这块板子多得多，问题就不只是撞到了东西。

### 方法

评测平台叫 RoboDojo，是一个通用机器人操作的评测框架。作者从里面选了八个任务，包括抓取和摆放、叠方块、叠碗、插管子、叠衣服，还有按数字按按钮。

场景有四种。没有镜子，桌边立一面镜子，相邻两边各立一面、互相垂直的双镜，还有把桌面本身换成镜面。立着的镜子往外挪了 5 厘米，多留点空间，但不保证机器人碰不到。

被测的五个策略覆盖三条路线，VLA，也就是视觉语言动作模型，世界动作模型，它同时建模动作和未来的画面，还有反复观察、再调用机器人工具的 Agent 式策略。具体是 WALL-OSS、VPP2、Awomo-0.5、Rex-M1 Preview，以及 OpenAI 的 GPT-6-Astra，下面简称 Astra。

每个任务用 5 种初始布局，前四个策略每种布局跑 3 个随机种子，Astra 每种布局跑一次。同一个布局和种子，在四种场景下配对比较。

除了成功率，作者还记录机器人有没有碰到镜子、碰了多久，并把镜子场景里的失败归成三类。一是碰撞，机器人撞上镜子。二是幻影目标，机器人冲着镜子里的倒影去操作。三是空间错位，机器人把真实物体的位置判断错了。

### 实验和效果

先看总成绩，在表 1。五个策略没有镜子时，成功率在百分之60.8 到百分之72.5 之间。加了镜子以后，每一个策略，在每一种镜子设置下，成功率都下降了。

最严重的是镜面桌面。Rex-M1 Preview 从百分之66.7 掉到百分之4.2，VPP2 从百分之72.5 掉到百分之27.5。

三种镜子设置平均下来，WALL-OSS 掉了 46.9 个百分点，VPP2 掉了 40.6，Awomo-0.5 掉了 50，Rex-M1 Preview 掉了 56.7。掉得最少的是 Astra，10.8 个百分点。

有意思的是，Astra 碰镜子最频繁，双镜下百分之85 的回合都碰到了镜子，成功率却掉得最少。作者据此认为，光看碰没碰到镜子，解释不了成绩下降。作者也提醒，各个策略的执行预算不同，这张表不能当作策略之间的排名。

镜子和不反光板的对照在表 2，只对 WALL-OSS 和 Awomo-0.5 做了，每种设置 120 对配对回合。双镜下，WALL-OSS 只在镜子下失败的有 25 对，只在板子下失败的只有 2 对。Awomo-0.5 是 42 对比 3 对。

更关键的是，那些只因为镜子才失败的回合里，有百分之50 到百分之87.5，在镜子和板子两种条件下，机器人都没有碰到板子。也就是说，很多失败不是撞出来的，作者认为这和空间错位的表现一致。

再看只因镜子失败、而且两种条件下都碰到了板子的回合。双镜下，接触时间的占比，WALL-OSS 是不反光板的 1.85 倍，Awomo-0.5 是 6.83 倍。作者的解释是，机器人可能一直往倒影的位置够，被镜子挡住，这符合幻影目标。不过这两组只有 6 个和 7 个回合。

接下来是三个针对性的诊断。第一个，在镜面桌面上抓东西，结果在表 3。作者检查夹爪合上、抬起来的时候，里面到底有没有东西。在三个任务上，VPP2 的成功率从百分之97.8 掉到百分之4.4，而空抓率，也就是夹爪合上抬起、感应区域里却没有目标，从 0 升到了百分之95.6。几乎每一个回合，都抓了个空。

这张图就是一个例子。夹爪在离真管子有一段距离的地方合上，然后空着手抬了起来。

第二个诊断是推东西。一个 T 形物体紧贴镜子，和倒影在画面里连成十字形、I 形，或者横过来的门字形，机器人要把它推开。VPP2 在门字形下，物体几乎没动，平均却碰了 11.8 次镜子。Astra 则在全部 15 个回合里，都把物体和倒影稳定地分开了。

不过附录里有一个很诚实的对照。如果在仿真里关掉机器人和镜子之间的碰撞，VPP2 在十字形和门字形下，都从 5 次全失败变成 5 次全成功。作者认为，在这个推的任务里，实体镜子挡住了 VPP2 的有效推动动作。

第三个诊断，最像镜子测试。用一块板子挡住主摄像头，让它只能看到物体在镜子里的倒影，手腕上的摄像头还能直接看到一部分物体。任务是用杯子把方块盖住，再按指定顺序揭开。

结果，WALL-OSS 和 Awomo-0.5 只在百分之0 到百分之26.7 的回合里碰到了真杯子，一次都没完成。在指令前面加一句，你看到的是镜子里的虚像，也没有带来稳定的改善。Astra 每次都碰到了杯子，但五次里最多只成功一次。

最后是真机实验，在表 6。作者在 Franka 机械臂上，测了 Pi 0.5 和 FastWAM 两个策略，四个任务，每个任务每个条件 10 次。Pi 0.5 的总成功率，从没有镜子的百分之77.5，降到立镜的百分之52.5，镜面桌面的百分之27.5。FastWAM 从百分之60，降到百分之30 和百分之10。

叠方块这个任务最明显，两个策略没有镜子时都是百分之80，换成镜面桌面，都变成了 0。

### 和以前方法的对比

和以前的鲁棒性评测比，这篇的不同在两点。一是扰动的类型，不是把画面弄模糊、换光照，而是在画面里多出一份看起来完全真实、却抓不到的东西。二是用不反光板做对照，尽量把撞到实体和被倒影误导这两件事分开。

我的判断是，最值得注意的，是除了 Astra 之外的四个策略，掉得非常一致，平均都在 40.6 到 56.7 个百分点之间。标准评测上六七成的成功率，碰到镜面桌面，都只剩不到三成。这说明标准 benchmark 上的高分，不代表策略真的分得清，画面里哪个才是能抓的实物。

我的判断是，Astra 掉得最少，但不能简单下结论说大模型 Agent 更懂镜子。它每个任务每个条件只有 5 个回合，预算也和别人不同，而且在挡住主摄像头的测试里，它同样基本失败。

### 局限

论文没有单独的局限章节，但有几处保留。不反光板的对照只做了两个策略，镜面桌面没有对照。失败的归因，作者只说和幻影目标、空间错位一致，是从轨迹和接触数据推断的，不是直接测出来的。

样本量也不大。Astra 每个任务每个条件 5 个回合，真机每个任务每个条件 10 次。推东西的诊断里，不同形状对应的摆放位置也不同，作者承认，差异可能既来自画面，也来自物理上好不好推。

另外，前四个策略都用官方发布的权重，没有做针对镜子的训练或适配。这篇只回答了现在的策略会不会被镜子骗，没有测试怎么修。

### 未来方向

作者在结论里说，这些结果暴露出，机器人把视觉观察转化成有效物理动作还有困难，并提出把对反光的鲁棒性，作为评测和改进机器人策略的一个具体目标。论文给出了代码仓库的地址。

我的判断是，镜子只是一个代表。玻璃门、不锈钢台面、反光的地板，家里和工厂到处都是。机器人真正走进这些地方之前，这类测试应该成为常规项目，训练数据里也值得有意加入反光场景。

我的判断是，还有一点值得关注，在指令里直接告诉策略这是镜子里的虚像，并没有稳定的帮助。至少在这个小样本测试里，对这几个策略来说，靠改提示词绕不过去，问题更可能出在从画面到动作的这一步。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Peking University, and the title doesn't hedge: Embodied AI fails the mirror test. One ordinary mirror is enough to break today's robot policies.

The authors put mirrors next to a robot's workspace in simulation and tested five robot policies built in very different ways. Every mirror setup lowered every policy's success rate. Averaged over the three setups, the drops ranged from ten point eight to fifty-six point seven percentage points.

### Problem

The inspiration is the mirror test from animal cognition, which asks whether an animal can use its reflection to guide actions toward its own body. This paper asks a robot version: when the camera image contains both an object and its reflection, can a policy still act on the real object?

Mirrors are a special kind of trouble. A reflection looks exactly like the real thing and moves with it, but there is nothing to grab where it appears. A reflection can also merge with the real object into one shape in the image, or reveal something that is otherwise hidden.

### Prior work

Earlier robustness benchmarks already showed that changing viewpoints, lighting or backgrounds can hurt robot policies badly. Mirrors aren't covered by those tests. A mirror doesn't change how things look. It duplicates the task-relevant objects somewhere else in the scene.

There is also plenty of mirror research, from detecting where mirrors are in an image, to testing whether an agent can recognize itself in one. This paper looks at something different: general-purpose robot policies manipulating ordinary objects in a scene that contains mirrors.

### Key idea

The design is simple. Keep the tasks, the success criteria and the model weights fixed. Add only the mirrors. Then compare each policy against its own mirror-free score, layout by layout.

But a mirror is also a physical panel, and a robot can just bump into it. So the key control swaps the mirror for a non-reflective panel with the same shape and position. If mirrors cause far more failures than that panel, collisions alone can't be the explanation.

### Method

The testbed is RoboDojo, an evaluation framework for general-purpose robot manipulation. The authors picked eight of its tasks, covering pick-and-place, stacking blocks and bowls, inserting tubes, folding clothes, and pressing buttons by number.

There are four scenes. No mirror. One upright mirror at the table edge. Two upright mirrors on adjacent sides, at right angles. And a mirrored tabletop, where the table surface itself is reflective. The upright mirrors were pushed back five centimeters for clearance, which still doesn't guarantee the robot won't touch them.

The five policies cover three approaches: vision-language-action models, which map images and instructions straight to actions; world action models, which model actions together with future observations; and agentic policies, which observe, reason and call robot tools. Concretely, WALL-OSS, VPP2, Awomo-zero-point-five, Rex-M1 Preview, and OpenAI's GPT-6-Astra, which I'll call Astra.

Each task uses five starting layouts. The first four policies get three random seeds per layout, and Astra gets one run per layout. Each layout and seed is paired across all four scenes.

Beyond success, the authors log whether and how long the robot touches the mirror, and they sort failures in mirror scenes into three kinds. Collision, where the robot hits the mirror. Phantom targeting, where it acts on a reflection. And spatial mislocalization, where it misjudges where the real object is.

### Results

Start with Table 1. Without mirrors, the five policies succeed between sixty point eight and seventy-two point five percent of the time. With mirrors, every policy drops in every mirror setup.

The mirrored tabletop is the harshest. Rex-M1 Preview falls from sixty-six point seven percent to four point two. VPP2 falls from seventy-two point five to twenty-seven point five.

Averaged over the three mirror setups, WALL-OSS loses forty-six point nine points, VPP2 forty point six, Awomo fifty, and Rex-M1 Preview fifty-six point seven. Astra loses the least, ten point eight.

Here's the twist. Astra touches the mirrors most often, in eighty-five percent of episodes with two mirrors, yet its success drops the least. The authors take that as a sign that contact alone doesn't explain the losses. They also warn that execution budgets differ across policies, so this table is not a ranking.

The mirror versus panel control is in Table 2, run for WALL-OSS and Awomo only, with one hundred and twenty matched pairs per setting. With two mirrors, WALL-OSS has twenty-five pairs that fail only with the mirror, and just two that fail only with the panel. For Awomo it's forty-two versus three.

More telling: among the failures caused only by the mirror, fifty to eighty-seven point five percent happen with no recorded contact with the panel, in either the mirror or the panel run. Many failures aren't collisions at all, which the authors read as consistent with spatial mislocalization.

Then take mirror-only failures where the robot touched the panel in both runs. With two mirrors, the share of time spent in contact is one point eight five times the panel's for WALL-OSS, and six point eight three times for Awomo. The authors' reading is that the robot may keep reaching for where the reflection appears and get blocked by the mirror, which fits phantom targeting. Those two groups are only six and seven episodes.

Next, three targeted diagnostics. The first is grasping on a mirrored tabletop, in Table 3. The authors check whether a gripper that closes and lifts actually holds anything. Across three tasks, VPP2's success falls from ninety-seven point eight percent to four point four, while its empty-grasp rate, closing and lifting with no target in the gripper's sensing region, goes from zero to ninety-five point six. Almost every episode includes an empty grasp.

This figure shows one. The gripper closes some distance away from the real tube and lifts up empty-handed.

The second diagnostic is pushing. A T-shaped object sits against the mirror and merges with its reflection into a cross, an I shape, or a sideways Pi shape, and the robot has to push it away. In the Pi shape, VPP2 barely moves the object while touching the mirror eleven point eight times per episode on average. Astra separates the object from its reflection in all fifteen of its episodes.

But the appendix has an honest control. Turn off collisions between the robot and the mirror in simulation, and VPP2 goes from zero out of five to five out of five on both the cross and the Pi shapes. The authors conclude that in this pushing task, the physical mirror was blocking VPP2's useful pushing motions.

The third diagnostic is closest to the real mirror test. A panel blocks the main camera's direct view, so it sees the objects only in the mirror, while the wrist cameras still see some of them directly. The task is to cover blocks with cups, then uncover them in a set order.

WALL-OSS and Awomo touch a real cup in only zero to twenty-six point seven percent of episodes and never finish the task. Adding a line to the instruction saying that what it sees are mirror images gives no consistent improvement. Astra reaches the cups every time, but succeeds at most once in five tries.

Finally, real robots, in Table 6. On a Franka arm, the authors test pi zero point five and FastWAM on four tasks, ten trials per task and condition. Pi zero point five drops from seventy-seven point five percent with no mirror to fifty-two point five with an upright mirror, and twenty-seven point five on a mirrored tabletop. FastWAM drops from sixty to thirty and ten.

Block stacking is the starkest. Both policies succeed eighty percent of the time without mirrors, and zero percent on a mirrored tabletop.

### Comparison

Compared with earlier robustness benchmarks, two things are new here. The perturbation isn't blur or lighting. It's an extra copy of the scene that looks completely real but can't be grasped. And the non-reflective panel control helps separate hitting an obstacle from being fooled by a reflection.

My read is that the striking result is how consistently the four policies other than Astra collapse, losing between forty point six and fifty-six point seven points on average. Policies that succeed sixty to seventy percent of the time on the standard tasks fall below thirty percent on a mirrored tabletop. A high benchmark score doesn't mean the policy can tell which thing in the image is actually graspable.

My read on Astra is that its small drop doesn't prove large-model agents understand mirrors. It ran only five episodes per task and condition, under a different budget, and it mostly failed the occluded-camera test too.

### Limitations

There's no separate limitations section, but the paper flags several caveats. The panel control covers only two policies, and there's no panel control for the mirrored tabletop. The failure types are described as consistent with phantom targeting and mislocalization, inferred from trajectories and contact data rather than measured directly.

Samples are small. Astra has five episodes per task and condition, and the real robot has ten trials per task and condition. In the pushing test, the object's placement differs between shapes, and the authors note the differences may reflect both appearance and how physically reachable the object is.

Also, the four pretrained policies are used with their released weights, with no mirror-specific training or adaptation. So the paper answers whether today's policies get fooled by mirrors, not how to fix them.

### Future work

In their conclusion, the authors say these results expose difficulties in turning visual observations into effective physical actions, and they propose reflection robustness as a concrete target for evaluating and improving robot policies. The paper links its code repository.

My read is that mirrors are just the clearest case. Glass doors, stainless steel counters and shiny floors are everywhere in homes and factories. Before robots work in those places, tests like this should be routine, and reflective scenes deserve a deliberate place in training data.

My read on one more point: telling the policy outright that it's looking at mirror images didn't reliably help. At least in this small test, for these policies, you can't prompt your way around it. The problem more likely sits in the step from pixels to actions.

That's today's Daily Frontier Papers. The paper is linked in the description.

