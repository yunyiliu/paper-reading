# Has GPT-6-Astra Achieved Embodied Intelligence?

**大模型直接开机器人离实用还差多远**

- 论文 Paper: [arXiv 2610.22471](https://arxiv.org/abs/2610.22471)
- 单位 Affiliation: SenseNova · Ropedia · 南洋理工大学 · 东京大学等
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-10-21
- 视频 Videos: [YouTube 中文](https://youtu.be/RA5lsdTI1t0) · [YouTube English](https://youtu.be/AF9qy6em0IM) · [B站](https://www.bilibili.com/video/BV1wTnz6eEvk)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 66 页的技术报告，标题是一个问题，GPT-6-Astra 实现具身智能了吗。署名是联合研究团队，首页的 logo 有 SenseNova、Ropedia、南洋理工大学和东京大学等。

GPT-6-Astra 是 OpenAI 的通用大模型。这篇报告把它直接当成机器人的控制器，在 12 个仿真 benchmark 和 8 台真机上测了一遍，然后问一个更大的问题，这对整个领域意味着什么。

### 问题

先说背景。机器人操作和导航，过去基本是专用模型的地盘，比如 VLA，也就是视觉语言动作模型，还有世界动作模型 WAM。通常是为特定任务、特定机器人收集数据，再训练一个专用模型。

作者的说法是，现在一个通用模型也能做这些任务，还表现出零样本泛化、上下文学习和失败恢复的能力。所以问题变成两个，它到底能做到多好，以及剩下的难点在哪里。

### 以前的方法

以前的主流路线，是从示范里学策略。先采集一个任务的示范数据，拟合一个策略，再在同一任务的新回合上评测。作者指出，现在大多数具身 benchmark，也是按这个思路设计的。

另一条路线，是让大模型只做高层规划，比如 SayCan、Code as Policies。大模型决定做什么，怎么做交给预先定义好的技能或者控制器。这篇报告问的是，前沿大模型能不能自己当控制器，直接输出末端执行器的动作，也就是夹爪该到哪个位置、朝哪个方向。

GPT-6-Astra 发布后几周内，已经冒出一批评测，但大多一次只看一个 benchmark、一种能力，或者一种 harness。harness 指的是包在模型外面的那套接口、工具和流程。这些研究的结论也不完全一致，比如有一篇发现，给示范反而降低了它在 RoboDojo 上的成功率，另一篇却报告示范加交互记忆能带来提升。

### 核心想法

这篇报告的做法，是用同一套协议，测全部 12 个 benchmark，覆盖桌面机械臂、人形机器人、移动平台、灵巧手和导航，模型权重全程不变。

协议有两条原则。第一，允许上下文学习，英文叫 in-context learning，简称 ICL。传统策略能用训练集里的示范来训练，那 GPT-6-Astra 也可以看这些示范，只是不更新权重。

第二，把执行预算放大，最多给到官方步数上限的 10 倍。作者发现，标准的步数上限会明显低估它，因为它能发现动作失败，换个策略，再恢复过来，多给步数，就多了纠错的机会。

### 方法

具体流程看这张图。测试之前，GPT-6-Astra 先自己研究训练数据，留下它认为有用的东西，写成笔记和代码，打包成一个知识包。这个包在测试前冻结，只读。

每个测试回合，都从一个全新的会话开始，带着这个知识包，在 1 倍或 10 倍的步数预算里和环境交互。回合之间不传递任何信息，测试回合也不参与准备。

知识包里放什么，是模型自己决定的，但在不同 benchmark 上，都反复出现六类内容，任务说明、示范轨迹、参考图片、辅助代码、完成检查和恢复办法，还有机器人参数这类元数据。

测试时，它先读任务说明，用辅助代码取出候选动作，再对照参考图片检查执行情况。它不是照搬示范，比如在一个双手装箱任务里，它把参考关节重新映射，补偿了工作台 4 到 5 厘米的高度差。

### 实验和效果

先看总体结果，就是第一张图。在标准预算下，它在多数 benchmark 上已经追平或超过已发表的最好结果，差距最大的是 LIBERO-Ultra-Long 和 DexJoCo，成功率接近之前最好成绩的两倍。

具体数字在表 2。LIBERO-Ultra-Long 是长链条的桌面操作，之前最好是 44.0，它在标准预算下是 82.2。DexJoCo 是灵巧手任务，之前最好是 40.3，它是 78.2。

放大到 10 倍预算以后，LIBERO 系列和人形机器人的 SIMPLE，成功率都超过 98，接近饱和。

涨得最多的，是需要反复纠错的任务。RoboDojo 是双臂操作，在 10 倍预算这组运行里，到 1 倍步数时成功率是 27.6，跑满之后是 81.9，大约是原来的三倍。灵巧手的 Bench2Dex，从 25.6 涨到 61.5，翻了一倍多。

但也有没解决的。长程导航 LHPR-VLN，加了预算也只是小幅提升，最终 28.9。家务任务 BEHAVIOR 是 37.0。这两个是全场最低的。

代价也很重。在 LIBERO-Long++ 上，标准预算下成功率已经超过 97%，但每个回合仍然要消耗大约 100 万 token，花将近一个小时。在 RoboDojo 上，10 倍预算下，单个回合平均要三个多小时。

剩下的失败，作者归成三类，见这张图。第一类，判断不了物理上是否真的成功。挂杯子任务里，它在头部相机里看到杯柄和挂钩重叠，就当成已经挂上了，其实杯子掉在了桌上。

第二类，挽回不了的后果。倒液体时杯子早早翻倒，液体洒在桌上，它把杯子扶起来继续倒，但洒出去的收不回来。第三类，长流程里保不住进度。耳机已经挂好了，它回头调整一个耳罩，又把整副耳机拽了下来。

然后是真机。作者在 8 台机器人上做了测试，从单臂、双臂、灵巧手，到轮式移动机器人和 Unitree G1 人形机器人。真机上不给示范，是零样本，只给人写的上下文，比如工具说明、关节限位和安全约束。预算按实际时间算，一般是 60 分钟。

结论是，成功率很看任务。刚性物体的抓取和放置，18 次成功 17 次。叠布在 Franka 3 上，4 次成功 2 次。插插头，在三个平台上一共 11 次，全部失败。

有一个很有意思的发现，叫主动视觉。插花任务里，单臂的腕部相机看不到花瓶口，插花失败了。换成双臂，它会把空着的那只手移过去，用那只手上的相机对准瓶口，然后把花插进去。

真机也暴露了仿真里没有的约束。碰撞会触发保护停机，灵巧手的手指在负载下卡住，温度升到阈值，抓取被迫中断。在 G1 人形机器人上，大约 90% 的时间花在思考上，一次操作要一个多小时。仿真里用更多步数换成功率的办法，在真机上行不通。

### 和以前方法的对比

先说清楚比较的口径。表 2 里的已发表结果，都是标准预算下的，作者在附录里也写明，延长预算的结果，不能和标准预算的基线直接比。而且在导航的 VLN-CE R2R 上，它标准预算的 80.0，还低于另一篇论文里 GPT-6-Astra 智能体方法的 81.3。

再看它和专用 VLA 怎么配合。作者在 RoboDojo 上做了消融，见这张图。单独用微调过的 π0.5，成功率 6.91%。让 GPT-6-Astra 只给子目标，VLA 直接执行，是 5.83%。反过来，让 VLA 提出动作，GPT-6-Astra 审核修改以后再执行，升到 23.61%。

每次审核 4 个动作，token 用量减少 63.5%，成功率反而升到 25.83%。把知识包和闭环审核一起用，成功率最高，到 54.17%，但每个回合要用掉约 1933 万 token，大约是不加知识包的最好配置的 6.9 倍。

动作的表示方式也很关键。让它直接输出关节角度，比输出末端位姿低了 5.84 个百分点。在 BEHAVIOR 上，从关节空间的接口换成末端位姿的接口，成功率从 1% 升到了 37%。

我的判断是，这篇报告最有价值的地方，不是又刷了一组高分，而是把高分拆开了。分数很大一部分来自 benchmark 给的条件，示范可以学，步数可以加，仿真会停下来等它思考，而这些条件到了真实世界都不成立。

### 局限

作者自己列了几条。知识包基本是按任务做的，迁移能力差。RoboDojo 里没有专属示范的那一组，成功率是各组最低的。RoboLab 没有训练集，它在那里虽然领先，绝对成功率也只是中等。

它常常能发现任务还没完成，却说不出是哪个物理条件没满足，所以纠正怎么也收敛不了。一旦出现不可逆的失误，也没法可靠地恢复。

评测本身也有边界。家务原型里的失败案例，作者明说数量少，是定性的。真机部分，单个任务大多只试了 1 到 6 次。

### 未来方向

作者认为，接下来要攻的是五件事，上下文学习的通用性，成本和执行时间，物理状态的验证，从不可逆失败中恢复，以及长流程里保住进度。

他们还提出了下一代具身 benchmark 的三条原则。任务从人的真实需求出发，比如帮我做个早餐，按结果对人有多大用来打分。不给场景内的示范，让机器人在部署中自己积累经验。世界时钟一直在走，思考的时间也算成本。

他们在 Isaac Sim 里搭了一个一居室公寓做原型，用 AgiBot G2 零样本测试，见这张图。冰箱门从第 300 秒开到第 765 秒都没关。水龙头没关，水槽溢了出来。煎鸡蛋时，它的决定是对的，等蛋凝固就关火，但每次决策都要花几秒，火一直开着，等它关火，蛋已经凝固了大约 90 秒，煎糊了。

作者的总结是，GPT-6-Astra 已经基本跨过了具身能力的门槛，但还没跨过实用具身智能的门槛。

我的判断是，对做 VLA 和机器人数据的人来说，这篇的信号很清楚，在现有 benchmark 上刷分的意义在下降，竞争会转向成本、速度、可靠性，以及更接近真实部署的评测。

我的判断是，也要看到这篇的边界。结果依赖作者自己搭的 harness 和大量 token，别人能不能用合理的成本复现，论文没有回答。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is a sixty-six page technical report whose title is a question: has GPT-6-Astra achieved embodied intelligence? It's signed by a joint research team, with SenseNova, Ropedia, Nanyang Technological University and the University of Tokyo among the logos on the front page.

GPT-6-Astra is OpenAI's general-purpose model. The authors use it directly as a robot controller, run it across twelve simulation benchmarks and eight real robots, and then ask the bigger question: what does this mean for the field?

### Problem

Some background first. Robot manipulation and navigation have mostly belonged to specialized models, like VLAs, vision-language-action models, and world action models, or WAMs. The usual recipe is to collect data for a specific task on a specific robot, and train a dedicated model on it.

The authors' claim is that a general-purpose model can now do these tasks too, and shows zero-shot generalization, in-context learning and failure recovery along the way. So the question becomes twofold: how good can it actually get, and what's still hard?

### Prior work

The dominant approach has been learning a policy from demonstrations. You collect demonstrations of a task, fit a policy to them, and evaluate on held-out episodes of that same task. The authors point out that most embodied benchmarks today were designed around exactly that setup.

A second line of work uses large models only for high-level planning, as in SayCan or Code as Policies. The model decides what to do, and a predefined skill set or controller decides how. This report asks whether a frontier model can be the controller itself, emitting end-effector actions directly, meaning where the gripper should go and which way it should point.

Within weeks of GPT-6-Astra's release, a wave of evaluations appeared, but most looked at one benchmark, one capability, or one harness at a time. A harness here means the interfaces, tools and workflow wrapped around the model. Those studies don't always agree either: one found that giving demonstrations actually lowered its success rate on RoboDojo, while another reported gains when demonstrations were combined with interaction memory.

### Key idea

This report runs all twelve benchmarks under one protocol, covering tabletop arms, humanoids, mobile platforms, dexterous hands and navigation, with the model weights fixed throughout.

The protocol rests on two principles. First, in-context learning, or ICL, is allowed. Conventional policies get to train on the benchmark's demonstrations, so GPT-6-Astra gets to study them too, just without any weight updates.

Second, the execution budget is extended, up to ten times the official step limit. The authors found that standard step limits significantly underestimate the model, because it can notice a failed action, change strategy, and recover, so more steps means more chances to correct itself.

### Method

Here's the pipeline in this figure. Before testing, GPT-6-Astra studies the training data on its own, keeps what it judges useful, and writes notes and code into a knowledge package. That package is frozen and read-only before testing begins.

Every test episode starts a fresh session with that package and interacts with the environment under a one-times or ten-times step budget. Nothing carries over between episodes, and the test episodes are never used for preparation.

The model decides what goes into the package, but six kinds of content keep showing up across benchmarks: task guidance, motion references from the demonstrations, visual reference frames, executable helper code, completion checks and recovery procedures, and metadata such as robot specifications.

At test time, it reads the task guidance, retrieves a candidate motion through a helper, and checks the execution against the visual references. It doesn't just replay demonstrations: in one bimanual box-loading task, for example, it remapped the reference joints and compensated for a four to five centimeter difference in workspace height.

### Results

Start with the headline figure. Under the standard budget, it already matches or beats the best published result on most benchmarks, with the biggest margins on LIBERO-Ultra-Long and DexJoCo, where it nearly doubles the previous best success rate.

The numbers are in Table 2. LIBERO-Ultra-Long is long-chain tabletop manipulation; the previous best was forty-four point zero, and it scores eighty-two point two at the standard budget. DexJoCo is dexterous-hand manipulation; the previous best was forty point three, and it scores seventy-eight point two.

With the budget extended ten times, the LIBERO family and SIMPLE, the humanoid benchmark, all land above ninety-eight percent, close to saturation.

The biggest gains come on tasks that need repeated correction. RoboDojo is bimanual manipulation. In the ten-times run, success at the one-times cutoff was twenty-seven point six, and by the end it was eighty-one point nine, roughly triple. On Bench2Dex, a dexterous-hand benchmark, success went from twenty-five point six to sixty-one point five, more than double.

Some things stay unsolved. On long-horizon navigation, LHPR-VLN, the extra budget only helps modestly, ending at twenty-eight point nine. The household benchmark BEHAVIOR sits at thirty-seven. Those are the two lowest scores of all.

And the cost is steep. On LIBERO-Long++, where standard-budget success is already above ninety-seven percent, each episode still burns about a million tokens and takes nearly an hour. On RoboDojo with the extended budget, a single episode averages more than three hours.

The authors group the remaining failures into three types, shown here. First, it can't verify physical success. In a mug-hanging task, it sees the handle overlap the peg in its head camera and takes that as hung, but the mug is actually on the table.

Second, consequences it can't undo. In a pouring task, the cup tips over early and the liquid spills; it rights the cup and keeps pouring, but the spill is permanent. Third, losing progress over long horizons. The headphones are already hung on their stand, it goes back to adjust one earcup, and pulls the whole headset off.

Then come the real robots. The authors tested on eight platforms, from single arms, dual arms and dexterous hands up to wheeled mobile manipulators and a Unitree G1 humanoid. There are no demonstrations here; it's zero-shot, with only human-written context such as tool schemas, joint limits and safety constraints. The budget is wall-clock time, normally sixty minutes.

The result: success depends heavily on the task. Rigid pick-and-place worked in seventeen of eighteen trials. Cloth folding on the Franka 3 succeeded twice in four tries. Plug insertion failed all eleven trials across three platforms.

One finding stands out, which the authors call active vision. In a flower-insertion task with a single arm, the wrist camera can't see the mouth of the vase, and the insertion fails. With two arms, the model moves the free arm over until its wrist camera looks at the vase mouth, and then completes the insertion.

The real world also adds constraints simulation doesn't have. Collisions trigger protective stops, and a dexterous hand's fingers stalled under load and heated up to a temperature limit, cutting the attempt short. On the G1 humanoid, about ninety percent of each run is spent deliberating, and a single manipulation trial takes over an hour. Trading more steps for higher success, which works in simulation, doesn't carry over to real hardware.

### Comparison

First, be clear about what's being compared. The published baselines in Table 2 all use the standard budget, and the appendix states plainly that extended-budget results aren't directly comparable with standard-budget baselines. And on the VLN-CE R2R navigation benchmark, its standard-budget eighty point zero is actually below the eighty-one point three reported for a GPT-6-Astra agent in another paper.

Next, how it works with a specialized VLA. The authors ran ablations on RoboDojo, shown in this figure. The fine-tuned pi zero point five on its own reaches six point nine one percent success. With GPT-6-Astra only setting subgoals and the VLA executing directly, it's five point eight three. Flip it around, so the VLA proposes actions and GPT-6-Astra reviews and edits them before execution, and success rises to twenty-three point six one.

Reviewing four actions per call cuts token use by sixty-three point five percent and actually raises success to twenty-five point eight three. Combining the knowledge package with closed-loop review gives the best result, fifty-four point one seven percent, but at about nineteen point three million tokens per episode, roughly six point nine times the best setup without it.

The action representation matters a lot too. Having the model output joint angles directly scores five point eight four points lower than end-effector poses. On BEHAVIOR, switching from a joint-space interface to an end-effector pose interface took success from one percent to thirty-seven.

My read is that the real value here isn't another set of high scores; it's that the report takes those scores apart. A large share of the performance comes from conditions the benchmarks grant: demonstrations to study, extra steps for free, and a simulator that pauses while the model thinks. None of those hold in the real world.

### Limitations

The authors list several themselves. The knowledge packages are mostly task-specific, so they transfer poorly. The RoboDojo group with no task-specific demonstrations has the lowest success of any group. RoboLab has no training set at all, and while the model leads there, its absolute success is only moderate.

It often notices a task isn't done without identifying which physical condition is unmet, so its corrections never converge. And once something irreversible happens, it can't reliably recover.

The evaluation has limits too. The authors say outright that the household failure episodes are few and qualitative. And on the real robots, most individual tasks were run only one to six times.

### Future work

The authors name five things to work on next: making in-context learning more general, cutting cost and execution time, verifying physical state, recovering from irreversible failures, and preserving progress over long horizons.

They also propose three principles for the next generation of embodied benchmarks. Tasks should come from real human needs, like "make me breakfast," and be scored by how useful the outcome is. No in-domain demonstrations, so the robot has to build experience during deployment. And a world clock that keeps running, so thinking time counts as cost.

They built a one-bedroom apartment in Isaac Sim as a prototype and ran an AgiBot G2 robot zero-shot, as shown here. The fridge door stayed open from second three hundred to second seven hundred sixty-five. A running tap was left on and the sink overflowed. Frying an egg, it made the right call, turn off the burner once the egg sets, but each decision took seconds while the burner stayed on, and by the time it was off, about ninety seconds after the egg had set, the egg was charred.

The authors' conclusion is that GPT-6-Astra has largely crossed the threshold of embodied competence, but not yet that of practical embodied intelligence.

My read is that for people building VLAs and robot datasets, the signal is clear: topping existing benchmarks is worth less and less, and the competition is moving to cost, speed, reliability, and evaluation that looks like real deployment.

My read is also that this report has its own limits. The results depend on the authors' own harness and a lot of tokens, and whether anyone else can reproduce them at a reasonable cost is a question the paper doesn't answer.

That's today's Daily Frontier Papers. The paper is linked in the description.

