# Learning from Fleet Experience: A Collective Learning Paradigm for Robots

**不再要人纠正机器人越干越稳**

- 论文 Paper: [arXiv 2610.25736](https://arxiv.org/abs/2610.25736)
- 单位 Affiliation: 清华大学 · AgiBot · 无问芯穹
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-10-20
- 视频 Videos: [YouTube 中文](https://youtu.be/R9onXqQG9ZM) · [YouTube English](https://youtu.be/slUaldJRU3Y) · [B站](https://www.bilibili.com/video/BV1t4nc6KEiN)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇清华大学、AgiBot 和无问芯穹合作的论文，标题是 Learning from Fleet Experience，从机器人集群的经验里学习。

它要回答的问题是，一大批机器人部署出去以后，怎么让它们边干活边变可靠，又不用一直派人守着纠正。

### 问题

先说背景。现在的机器人基础模型，比如 VLA，也就是视觉语言动作模型，能看画面、听指令、输出动作，但刚部署时还不够可靠。物体、接触条件，甚至每台机器人自身的差异，都会让它出错，多步骤任务里，错误还会累积。

另一方面，机器人在真实世界里干活，本身就在产生经验，集群越大，经验越多。问题是，怎么把这些经验变成更可靠的执行。

### 以前的方法

以前比较有效的办法，是让人来教。一种是专家示范，人遥操作机器人做一遍，机器人照着学。另一种叫 HG-DAgger，机器人自己跑，人在旁边盯着，快出错的时候人接管，把人的纠正动作记下来再训练。

也有强化学习的路线，比如论文拿来对比的 RLT 和 RECAP，它们也用机器人自己的经验，但在作者的实验里，这两种方法在训练过程中都还要用到人的纠正动作。

问题在于，只要学习依赖新的专家动作，就得有遥操作员一直守着，看机器人什么时候失败，及时接管。机器人从几台变成几十上百台，人手很难跟上。

### 核心想法

这篇论文提出的思路叫 CEL，集体经验学习。部署出去的机器人，把各自干活的经验汇总起来，一起训练一个共享的策略，更新后的策略再发回整个集群，继续干活，继续产生经验。

关键在于学习用的动作从哪里来。在作者的实现里，经过最初的专家训练之后，用来学习的动作全部来自机器人自己的执行，不再收集新的专家示范或纠正动作。人只做两件事，判断这一次成功还是失败，以及把场景复位。

论文的第一张图把三种经验来源放在一起，从左到右，对专家动作的依赖越来越低。

### 方法

具体实现分成算法和系统两部分。算法叫 FERT，意思是失败引导的经验检索微调。用一句话说，成功的轨迹提供要模仿的动作，失败的轨迹决定学习的重点放在哪里。

这张图是 FERT 的四个步骤。第一步，每台机器人各自异步地跑任务，每条轨迹标上成功或失败。成功的轨迹进入共享的经验池，失败的轨迹只用来做查询，它的动作不会被模仿。

第二步，把每一帧编码成向量。用的是模型自带的视觉编码器，但把它冻结住，再拼上机器人关节的状态，这样既知道场景是什么样，也知道机器人当时的姿势。

第三步是关键，失败引导的检索。一个失败的标签，并不能告诉你到底是哪一刻开始出错的。所以 FERT 取失败结束前最多 3 秒的画面，当作查询。

然后拿这段查询，去成功的轨迹里找最像的那一段。比对用的是一种带约束的动态时间规整，叫 S-DTW，简单说，就是允许两段动作快慢有些差别，但先后顺序必须对得上。每次检索，从不同的成功轨迹里取最像的 10 段，每段前后再各扩展 30 帧。

打个比方，一台机器人插第二朵花的时候失败了，系统就会从成功的记录里，找出成功插第二朵花的那几段。论文附录里就有这样的例子。

第四步，按优先级微调。被检索到的成功片段，采样优先级会提高，训练时被抽到的次数更多。旧的优先级会按比例逐渐衰减，所以反复出现的失败，会让相关的成功动作一直被重点训练。

训练目标本身很朴素，就是模仿这些成功的动作。论文强调，这里不需要学一个价值函数，也不需要估计优势，二元的成败标签就够了。

作者还给了一个理论分析，在一定假设下，用小步长更新一步，只用成功经验就能提高成功率，加上检索加权，提高得更多。实际效果，要看后面的消融实验。

系统部分叫 FATS，容错异步训练系统。这次部署里，96 台机器人和 6 台云服务器，通过一条 30 Gbps 的专线连在一起。机器人在现场执行，推理和训练都在云端。

难点在于机器人会出故障。96 台的那次训练里，平均每 22 分钟就有机器人被打断。如果一台出问题就整体重启，或者每次更新都等所有机器人，其他机器人就只能陪着等。

FATS 的做法是这样。数据面叫 SwarmLink，所有机器人共享云端推理，轨迹数据边执行边上传。控制面叫 HotSwarm，哪台机器人出了故障，就只在原位重启那一台，其他机器人照常干活，出故障那台留下的旧数据，会被识别出来丢掉。

### 实验和效果

实验设置。机器人是 AgiBot G2 双臂机器人，基础模型是 pi zero，一个基于 flow 的 VLA 模型。一共五个真实世界的任务，插花、叠衣服、给手机充电、用螺丝刀组装凳子，还有手机装盒。

手机装盒是多步骤任务，要把说明书、充电线、充电头、手机壳和手机放进盒子，再把盒子盖上，一次要两到三分钟，前面一步出错，可能影响后面的步骤。

训练流程是这样的。先用人类示范做监督微调，再让多台机器人跑 HG-DAgger，收集人的纠正，得到一个起点。FERT 和两个对比方法，都从同一个起点出发。

评测有个重要细节。测试用的是两台没参与训练数据收集的机器人，每个策略每台跑 60 次，一共 120 次，全程没人干预。也就是检验集群学到的东西，能不能迁移到同型号的其他机器人上。

主要结果在这张图。五个任务上，FERT 的成功率在 93.3% 到 99.2% 之间，用的是每个任务里最大的训练集群，最大的是手机装盒的 96 台。

五个任务平均，起点是 81.0%，FERT 提到 97.3%，提高了 16.3 个百分点。换个角度看，平均失败率从 19% 降到 2.67%，相对降低了 86%。

提升最明显的是给手机充电，从 60.0% 提到 97.5%。手机装盒用 96 台机器人训练，从 79.2% 提到 98.3%。

论文还拿人类遥操作做了参照，六个操作员，每人每个任务做 20 次。FERT 在五个任务上都超过了这个人类参照，平均高 16.7 个百分点。

再看规模。每台机器人的收集时间固定，机器人越多，经验就越多。这张图里，FERT 的成功率大体上随集群规模上升。手机装盒从 16 台扩到 96 台，成功率从 86.7% 提到 98.3%。

但规模小的时候不一定有用。在凳子组装和手机装盒上，只用 8 台机器人训练，FERT 反而比起点更差。作者的解释是，小集群收集的成功经验不够丰富，覆盖不了策略遇到的各种失败情况。

作为对照的 HG-DAgger 靠人来纠正。机器人从一台加到几台时有提升，再往上加，在充电和装盒上饱和，在凳子组装上还下降了。作者认为，人盯不过来可能是原因之一。

### 和以前方法的对比

再看正面对比。和单机器人在线强化学习 RLT，以及用集群数据做离线强化学习的 RECAP 相比，FERT 在五个任务上都更高，平均分别高 12.3 和 11.5 个百分点。

差距最大的地方，相对 RLT 是给手机充电，75.8% 对 97.5%，相对 RECAP 是手机装盒，75.0% 对 98.3%。而且这两个对比方法在训练中还用了人的纠正动作，FERT 没有用。

消融实验是在手机装盒、16 台机器人上做的。完整的 FERT 是 86.7%。去掉失败引导的检索，改成从成功经验里均匀抽样，掉到 64.2%，少了 22.5 个百分点。

如果检索到的片段不往前后扩展，会掉到 57.5%。说明匹配片段前后的动作，对学会一段成功的操作也很重要。

系统方面，96 台机器人的那次训练，从上午 10 点 37 分跑到下午 6 点半，中间午休停了一个小时。一共收集了 9834 条轨迹，记录到 22 台机器人、一共 30 次故障。这些故障总共损失了 8.3 个机器人小时，只占 1.3%。

作者还用真实记录做了回放比较。如果改成同步方式，每次更新都等所有机器人跑完一轮，只能收集到实际数量的 33%。如果每次故障都整体重启 10 分钟，只剩 74%。

我的判断是，这篇最有价值的，是把集群规模变成了数据来源。以前机器人越多，需要的遥操作员越多，在这套方法里，多加机器人，主要增加的是经验，而不是人的纠正工作。

我的判断是，这组对比并不对等。FERT 每次大约 7 到 8 小时在线训练，最多用了 96 台机器人，而 RLT 只在一台机器人上跑了大约 30 到 100 条完整轨迹，RECAP 也只做了一轮离线改进。论文没有在相同机器人时间下做对比。

### 局限

作者自己说明的局限是，这次只研究了五个任务、一种机器人平台，而且每个任务训练的是单独的策略。

还有几点要注意。第一，不需要专家动作是有前提的，起点仍然要靠人类示范和纠正训练出来，FERT 阶段，操作员也还要标注成败、复位场景、看着安全。

第二，集群太小的时候，FERT 可能让策略变差，前面说的 8 台机器人就是例子。第三，和人类遥操作的比较要谨慎看，操作员之间差别很大，在手机装盒上，个人成功率从 25% 到 95% 都有。

### 未来方向

作者把更广的任务分布、不同型号的机器人，以及长时间运行中的持续适应，列为未来方向。他们还提到，成败标签可以由人来标，也可以换成学出来的奖励模型，不过这次实验用的是人工标签。

作者的愿景是，机器人集群不只是在执行任务，也在贡献让自己变强的经验，而专家监督不需要跟着集群规模同比例增加。

我的判断是，下一步最值得看的，是成败判断能不能也自动化。现在每条轨迹还要有人判断成败、复位场景，这部分人工仍然跟着机器人数量增长。如果换成奖励模型，被误判成成功的失败，会不会被学进去，需要实验来回答。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is a collaboration between Tsinghua University, AgiBot and Infinigence AI, and it's called Learning from Fleet Experience: A Collective Learning Paradigm for Robots.

The question it asks: once a whole fleet of robots is deployed and working, how do you make them more reliable on the job, without keeping people on standby to correct them?

### Problem

Some context first. Robot foundation models, like VLAs, or vision-language-action models, can look at a scene, take an instruction and output actions. But out of the box, they aren't reliable enough. Differences in objects, in contact, even between individual robots, cause errors, and in multi-stage tasks those errors pile up.

At the same time, robots working in the real world are generating experience all the time, and the bigger the fleet, the more of it there is. The question is how to turn that experience into more reliable execution.

### Prior work

The approaches that have proved useful rely on people. One is expert demonstrations: a human teleoperates the robot, and the robot imitates. Another is called HG-DAgger: the robot runs on its own, a human watches, takes over when things go wrong, and those corrections become training data.

There are reinforcement learning approaches too, like RLT and RECAP, which the paper compares against. They also use the robot's own experience, but in the authors' experiments, both of them still use human corrective actions during training.

The problem is that as long as learning depends on new expert actions, teleoperators have to stay on watch, spot failures and step in. Go from a handful of robots to dozens or a hundred, and the people can't keep up.

### Key idea

The paper's answer is called CEL, collective experience learning. Deployed robots pool their experience to train one shared policy, and the updated policy goes back out to the whole fleet, which keeps working and keeps generating experience.

The key question is where the training actions come from. In the authors' implementation, after the initial expert-supervised training, every action used for learning comes from the robots' own executions. No new demonstrations, no new corrections. Humans do just two things: they label each attempt as a success or a failure, and they reset the scene.

The paper's first figure puts the three sources of experience side by side, and from left to right, the need for expert actions goes down.

### Method

The implementation has two halves, an algorithm and a system. The algorithm is FERT, failure-guided experience retrieval fine-tuning. In one line: successes supply the actions to imitate, and failures decide where learning concentrates.

Here are FERT's four steps. Step one: every robot in the fleet runs tasks on its own schedule, and each trajectory is labeled success or failure. Successes go into a shared experience buffer. Failures are only used as search queries, and their actions are never imitated.

Step two: every frame is turned into an embedding. They use the model's own vision encoder, frozen, and append the robot's joint state, so the embedding captures both what the scene looks like and how the robot is posed.

Step three is the key one: failure-guided retrieval. A failure label doesn't tell you the moment things started going wrong. So FERT takes up to three seconds of observations just before the failed episode ends, and uses that as the query.

That query is matched against the successful trajectories to find the most similar stretch. The matching uses a constrained form of dynamic time warping called S-DTW, which, put simply, lets the two clips run at somewhat different speeds but keeps them in the same order. Each retrieval keeps the ten closest segments, each from a different successful trajectory, and pads each one by thirty frames on either side.

For example, if a robot fails while inserting the second flower into a vase, the system pulls up clips where the second flower went in successfully. The appendix shows exactly this kind of case.

Step four: prioritized fine-tuning. Retrieved success segments get a higher sampling priority, so they show up in training more often. Old priorities decay gradually, so a failure that keeps recurring keeps the related successful actions in focus.

The training objective itself is plain imitation of those successful actions. The paper stresses that there's no learned value function and no advantage estimate. A binary success label is enough.

The authors also give a theoretical analysis. Under certain assumptions, one small update on successful experience alone raises the success rate, and adding retrieval weighting raises it further. Whether that matters in practice is a question for the ablations later.

The system half is FATS, a fault-tolerant asynchronous training system. In this deployment, ninety-six robots and six cloud servers are linked by a single thirty-gigabit-per-second connection. Robots act on site, while inference and training run in the cloud.

The hard part is that robots break. In the ninety-six-robot run, some robot was interrupted every twenty-two minutes on average. If one fault restarts the whole system, or every update waits for every robot to finish, everyone else sits idle.

So FATS splits the work. The data plane, SwarmLink, lets all robots share cloud inference and streams trajectory data while the robots are still running. The control plane, HotSwarm, restarts only the faulty robot, in place, while the rest keep working, and any stale data the failed process left behind is recognized and thrown away.

### Results

The setup. The robots are dual-arm AgiBot G2s, and the base model is pi-zero, a flow-based VLA. There are five real-world tasks: arranging flowers, folding clothes, charging a phone, assembling a stool with a screwdriver, and packing a phone into its box.

Phone packing is the multi-stage one. The robot puts the manual, charging cable, charger, phone case and phone into the box, then closes it. That takes two to three minutes, and a mistake early on can affect the steps after it.

Training goes like this. First, supervised fine-tuning on human demonstrations, then HG-DAgger across multiple robots to collect human corrections. That gives a starting checkpoint, and FERT and both baselines all start from the same one.

One evaluation detail matters a lot. Testing uses two robots that contributed no training data. Each policy gets sixty trials per robot, a hundred and twenty in total, with no human intervention. So this checks whether what the fleet learned transfers to other robots of the same model.

The main results are in this figure. Across the five tasks, FERT reaches between ninety-three point three and ninety-nine point two percent success, using the largest training fleet for each task, which tops out at ninety-six robots on phone packing.

Averaged over the five tasks, the starting point is eighty-one percent, and FERT brings it to ninety-seven point three, a gain of sixteen point three percentage points. Put another way, the average failure rate falls from nineteen percent to two point six seven, a relative cut of eighty-six percent.

The biggest jump is on phone charging, from sixty to ninety-seven point five percent. On phone packing, training with ninety-six robots takes success from seventy-nine point two to ninety-eight point three.

The paper also includes a human reference: six teleoperators, each doing twenty trials per task. FERT beats that human reference on all five tasks, by sixteen point seven percentage points on average.

Now scale. Collection time per robot is held fixed, so more robots means more experience. In this figure, FERT's success rate generally rises with fleet size. On phone packing, going from sixteen to ninety-six robots lifts success from eighty-six point seven to ninety-eight point three percent.

But small fleets don't always help. On stool assembly and phone packing, training FERT with only eight robots actually ends up below the starting point. The authors' explanation is that a small fleet doesn't gather enough varied successes to cover the failures the policy runs into.

The HG-DAgger comparison is interesting too. It relies on human corrections, and it improves going from one robot to several. But add more robots and it plateaus on charging and packing, and actually declines on stool assembly. The authors suggest limited human monitoring capacity may be part of the reason.

### Comparison

Now the head-to-head. Against RLT, which is single-robot online reinforcement learning, and RECAP, which is offline reinforcement learning on fleet data, FERT scores higher on all five tasks, by twelve point three and eleven point five points on average.

The largest gap against RLT is on phone charging, seventy-five point eight versus ninety-seven point five. Against RECAP, it's phone packing, seventy-five versus ninety-eight point three. And both baselines used human corrections during training, which FERT did not.

The ablations were run on phone packing with sixteen robots. Full FERT scores eighty-six point seven. Turn off failure-guided retrieval and sample successes uniformly instead, and it drops to sixty-four point two, a loss of twenty-two point five points.

Don't pad the retrieved segments with their neighboring frames, and it falls to fifty-seven point five. So the actions just before and after a matched moment matter for learning the successful behavior.

On the systems side, the ninety-six-robot run went from ten thirty-seven in the morning to six thirty in the evening, with an hour off for lunch. The fleet collected nine thousand eight hundred and thirty-four episodes, and logged thirty fault incidents on twenty-two robots. In total, those faults cost eight point three robot-hours, just one point three percent.

The authors also replayed the real traces under other designs. With synchronous rounds, where every update waits for every robot, you'd collect only thirty-three percent as many episodes. Restart the whole system for ten minutes after every fault, and you keep seventy-four percent.

My read is that the most valuable thing here isn't any single number. It's that fleet size becomes a data source. Before, more robots meant more teleoperators. Here, adding robots mostly adds experience, not correction work for humans.

My read is also that the comparison isn't evenly matched. Each FERT run involves about seven to eight hours of online training, with up to ninety-six robots, while RLT ran on a single robot for roughly thirty to a hundred complete episodes, and RECAP did only one round of offline improvement. The robot time behind each method differs a lot, and the paper doesn't compare them at equal robot time.

### Limitations

The limitation the authors state themselves is scope: five tasks, one robot platform, and a separate policy trained for each task.

A few more things to keep in mind. First, no expert actions comes with conditions. The starting point is still built from human demonstrations and corrections, and during FERT, operators still label outcomes, reset scenes and watch over safety.

Second, with too small a fleet, FERT can make the policy worse, as the eight-robot runs showed. Third, read the human comparison with care: operators vary a lot, and on phone packing, individual success rates range from twenty-five to ninety-five percent.

### Future work

The authors list broader task distributions, mixed robot platforms and adaptation over longer periods of operation as future work. They also note that outcome labels could come from a learned reward model instead of people, though these experiments used human labels.

Their larger vision is a fleet that contributes not just task execution but the experience that makes it better, without expert supervision having to grow in proportion.

My read is that the thing to watch next is whether the success-or-failure judgment can be automated too. Right now, people still label every outcome and reset every scene, and that work still grows with the number of robots. And if a reward model takes over, will a failure it mislabels get learned as a success? That needs experiments to answer.

That's today's Daily Frontier Papers. The paper is linked in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
