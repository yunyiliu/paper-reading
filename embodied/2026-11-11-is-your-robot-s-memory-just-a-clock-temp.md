# Is Your Robot's Memory Just a Clock? Temporal Shortcuts in Memory-Augmented VLA Policies

**机器人发个呆就以为干完了**

- 论文 Paper: [arXiv 2611.14488](https://arxiv.org/abs/2611.14488)
- 单位 Affiliation: 南洋理工大学
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-11
- 视频 Videos: [YouTube 中文](https://youtu.be/GLAYTCb3Tt8) · [YouTube English](https://youtu.be/pfAPjr2Zh4s) · [B站](https://www.bilibili.com/video/BV1fuWy6EEFq)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇南洋理工大学的论文，标题是一个问句，你的机器人的记忆，是不是只是一个时钟？

它研究带记忆的 VLA，也就是视觉、语言、动作一体的机器人模型。作者发现，机器人停下来几秒，什么都没干，它的记忆却把这段时间当成了干活，于是提前收工。

### 问题

先说机器人为什么需要记忆。让它把一个方块拿起放下三次，再按一下按钮。第二次拿之前和第三次拿之前，桌面几乎一样，光看眼前的画面，它不知道自己做了几次，必须记住。

一种简单又常用的做法，是按时间填记忆。从整段任务里均匀抽一些过去的画面，每一帧打上时间戳。在专门测机器人记忆的 benchmark RoboMME 上，公开的最强策略就是这么做的。

问题是，什么都没发生的时候，时间也在走。真实的机器人经常要等，等规划器，等操作员，等安全监控放行，也会伸手去拿东西又空手回来。作者把这种花了时间、但任务相关的东西都没变的片段，叫做空事件，null event。

所以论文问的就是一句话，按时间填的记忆，能不能分清是在等待，还是在干活？

### 以前的方法

带记忆的机器人策略已经不少，有看历史的 Transformer，有记忆库，有挑关键帧的，也有循环式的记忆。RoboMME 这类 benchmark 把它们放在一起，比的是哪种分数最高。

也有工作在问，记忆到底起了什么作用，比如历史被编码了却几乎没被用上，或者很长的历史带来虚假相关。它们的解法，一般是挑选留下哪些帧。

这篇的不同在于，它点名了一个具体的因素，就是经过的时间，并用干预实验证明策略真的在读它。作者还说，据他们所知，之前还没有人在测试时给带记忆的策略插入空事件。

### 核心想法

核心想法就在标题里。示范数据里，机器人干活的节奏很稳定，只看过了多久，就能猜出做到第几次了。模仿学习会利用任何能预测下一个动作的特征，于是策略学会了看钟，而不是数事件。作者把这叫做时间捷径。

而且作者认为，这个捷径藏在记忆的索引方式里，时间戳只是载体之一。记忆是跟着时间往前走，还是跟着事件往前走，决定了它会不会被空事件骗。

### 方法

这是一篇诊断类的论文，方法就是一套实验设计。被测的是 RoboMME 公开的 MME-VLA 策略，底座是 π0.5。它的记忆叫 FrameSamp，从整段任务里均匀抽 32 帧，每一帧带一个绝对时间步的编码。默认测其中最强的版本。

测试任务是两个数数的任务。PickXtimes，把方块拿起放下 K 次，再按按钮。SwingXtimes，把方块在两个目标之间来回搬 K 次。

然后在任务中途插入三种空事件。暂停，机器人原地不动 5 秒。手腕晃动，手腕来回转，持续一样长。还有空抓，在静止的物体上方抓一下空气，再回到原来的姿态。这三种都不改变任务依赖的东西。

最关键的设计，是可见和隐藏两种条件。可见，就是空事件的画面照常进入记忆。隐藏，就是机器人照样暂停或者晃动，但这段画面不写进记忆，时间戳也不往前走。一对比，就能分清是事件本身干扰了机器人，还是记忆出了问题。

为了单独检验时间编码，作者还从 π0.5 的底座出发，按公开的配方从头训了两个模型，一个保留时间编码，叫 CLOCK，一个去掉，叫 NOCLOCK，其他完全一样。

每个条件跑 50 个测试回合，不同条件用同一批回合配对比较。作者也提醒，评测不完全确定，同一个条件跑两遍能差 8 个回合，所以 10 个回合以内的差别要谨慎看。

### 实验和效果

先看第一张图。任务要求做三次再按按钮。没有暂停时，50 个回合成功 47 个。插入两次 5 秒暂停，而且暂停画面进了记忆，机器人只做了两次就按了按钮，时间点和没暂停时几乎一样，成功只剩 16 个。把暂停藏起来，成功回到 45 个。

总的来看，两次 5 秒暂停，让 PickXtimes 从 47 降到 16，SwingXtimes 从 44 降到 10。手腕晃动伤害一样大，空抓也让 SwingXtimes 的成功减半。论文报告，暂停只要 1.5 秒，就已经造成大部分损失。

失败的方式也很说明问题。几乎所有失败都是提前停，次数没做够就按了按钮，作者把这叫做幻觉进度，phantom progress。按 K 拆开看，K 等于 1 的大多还能成功，因为做完第一次就按本来就对。K 大于等于 2 的，在暂停或晃动下，一个都没成功。

而在隐藏条件下，每一格都回到了没有事件的水平。机器人身体上做的事一模一样，变的只是记忆，所以问题出在记忆。

接下来是为什么。作者只用经过的时间，去猜示范里已经做完了几次，在 PickXtimes 上能猜对 89%，SwingXtimes 上 80%，大约是总猜最常见次数的两倍。在两个真实遥操作数据集的三次重复示范上，也一样准。示范里又几乎没有发呆的片段，训练从来不会惩罚看钟的策略。

然后是一个很干净的干预实验。不插空事件，画面也不动，只把记忆里的时间戳乘一个系数。在 PickXtimes 上，时间快 1.5 倍，成功从 47 掉到 18，失败全是提前按。时间慢一半，策略反而多做了。时钟往哪边偏，失败就往哪边偏。

那去掉时间编码，是不是就好了？并没有。完全没有时间编码的 NOCLOCK，平时数得和 CLOCK 一样好，也不理会缩放的时间戳，但两次暂停还是把它的成功打到 21 和 14，藏起暂停又能恢复。

原因是第二条路。按时间均匀抽帧，暂停的画面会占掉 32 个格子里的一部分，图里这个例子占了 10 格。把暂停画面全换成暂停前的最后一帧，伤害不变。不暂停，只插入越来越多相同的帧，NOCLOCK 就越来越差。伤害来自这些什么都没发生的格子本身。

诊断完，作者试了两种修法。第一种是改数据，训练时往示范里插暂停。结果预测落空了。PickXtimes 上暂停不再扣分，只是因为没有暂停时的成功已经先从 32 掉到 24，SwingXtimes 上，有暂停时也不比不加增强好。

第二种是改索引。同一个 CLOCK 模型，不重新训练，只在测试时改记忆。如果机器人的姿态回到了之前某个保留帧的样子，场景没变，中间也没有接触，就把这一段压成一帧，长时间静止也一样。这样记忆只在发生事件时才往前走。

效果是，三种空事件下，成功率都至少翻倍，包括机器人在动的两种。空抓下，SwingXtimes 从 13 提到 43。不过 SwingXtimes 有暂停时，还没回到无事件的水平。

要注意，这个检测器读的是仿真器里的信号，判断标准又和作者构造空事件的方式一致。作者自己说，它检验的是把空帧移出索引够不够，不是一个能直接部署的检测器。

### 和以前方法的对比

再放到其他记忆设计上看。三个按时间抽帧的版本全部崩掉。按像素变化挑 token 的 TokenDrop，暂停伤得少，因为静止画面很少进记忆，但晃动伤得多，因为晃动改变了像素，却没改变世界。按子目标记录进度的两种记忆，两种空事件都不受影响。

两个独立开发、用其他底座的策略就没这么整齐。它们都会把晃动当成进度，但同样的暂停，SimpleMemVLA 在 PickXtimes 上几乎不受影响，另一个叫 LTM-π 的策略却崩了。

我的判断是，这篇最有价值的不是新模型，而是一套便宜的检查方法，可见和隐藏对比，加上缩放时间戳，任何带记忆的策略都能拿来测一测，它到底是在记事件，还是在看钟。

我的判断是，这对记忆类 benchmark 也是个提醒。如果示范节奏稳定，策略靠看钟也能把数数任务做对，分数高，不一定代表真的记住了发生过什么。

### 局限

局限作者写得很坦白。因果分析只覆盖了一个策略家族，也就是 MME-VLA，和用同样结构重训的模型，另外两个独立策略只测了，没有拆开分析。

所有实验都在仿真里，空事件都是脚本插入的，不知道真实部署中这种情况有多常见，事件检测器也用了仿真器信号。除了 CLOCK 和 NOCLOCK，其他模型都只训了一个种子。

事件索引本身也有盲区。数数任务里，一次真正的重复也会让世界回到之前的状态，可能被当成什么都没发生，在公开的策略上，这在 PickXtimes 上造成了明显的损失。而如果时间本身就是任务的一部分，把暂停压掉本来就是错的。

### 未来方向

作者提出的原则是，记忆应该在世界发生变化时才往前走，比如物体被移动了，或者产生了接触。时间如果是任务的一部分，应该作为输入交给策略。

他们认为，从真实传感器里检测这种变化，比在仿真里难，是这类记忆最主要的开放问题。在另一个 benchmark MIKASA-Robo 的探索性实验里，策略似乎也是靠提示持续了多久，来推断灯闪了几次。

我的判断是，下一步最值得看的是真机。论文开头列的等规划器、等操作员，都是真机会遇到的情况，但论文没有测它们在部署中有多常见。事件索引能不能只靠真实传感器做出来，又不误伤真正的重复，决定了这个结论能走多远。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Nanyang Technological University, and its title is a question: is your robot's memory just a clock?

It looks at vision-language-action models with memory, and finds a quiet failure. The robot stops for a few seconds and does nothing, but its memory counts that time as work, so it quits early.

### Problem

First, why a robot needs memory at all. Ask it to pick up a cube and put it down three times, then press a button. Before the second pick and before the third, the table looks almost the same. The current camera frame can't tell it how many picks it has done. It has to remember.

A simple and popular way to build that memory is by time. Sample past frames evenly across the whole episode, and stamp each one with its time step. On RoboMME, a benchmark built to test robot memory, the strongest released policy does exactly this.

The trouble is that time passes even when nothing happens. Real robots wait: for a slow planner, for an operator who paused them, for a safety monitor to clear. They hesitate, or reach for something and come back empty-handed. The authors call an interval like that, one that takes time but changes nothing the task depends on, a null event.

So the paper asks one question. Can a memory filled by time tell waiting from working?

### Prior work

There's no shortage of robot policies with memory: history-aware transformers, perceptual memory banks, keyframe selection, recurrent memories. Benchmarks like RoboMME line them up and ask which memory scores highest.

A smaller set of papers asks what memory actually does. Some find history gets encoded but barely used; others find long histories invite spurious correlations. They usually treat it as a shift in the distribution of histories, and fix it by choosing which frames to keep.

This paper's difference is that it names one specific culprit, elapsed time, and shows by intervention that the policy reads it. And on the robustness side, past work perturbs objects, textures, lighting, cameras and instructions. To the authors' knowledge, nobody has injected null events into memory policies at test time.

### Key idea

The key idea is right there in the title. In demonstrations, robots work at a steady pace, so how much time has passed tells you how far along the task is. Imitation learning will happily use any feature that predicts the next action, so the policy learns to read the clock instead of counting events. The authors call this a temporal shortcut.

And they argue the shortcut lives in how memory is indexed. Time stamps are just one of its carriers. Put differently, whether memory advances with time or with events decides whether a null event can fool it.

### Method

This is a diagnosis paper, so the method is an experimental design. The subject is RoboMME's released MME-VLA policy, built on pi zero point five. Its memory, called FrameSamp, keeps thirty-two frames sampled evenly over the episode, each tagged with an encoding of its absolute time step. By default they test the strongest released variant.

The tasks are RoboMME's two counting tasks. In PickXtimes, the robot picks and places a cube K times, then presses a button. In SwingXtimes, it carries a cube back and forth between two targets K times.

Then, mid-task, they inject one of three null events. A pause, where the robot holds still for five seconds. A wrist fidget, where the wrist rolls back and forth for as long. And an air-grasp, where the arm grasps the air above a resting object and returns to its starting pose. None of them changes anything the task depends on.

The crucial design is visible versus hidden. Visible means the event's frames enter memory like any others. Hidden means the robot physically does the same thing, but those frames are kept out of memory and the time stamps don't advance. Compare the two, and you can tell whether the event itself disturbed the robot, or the memory of it did.

To isolate the time encoding, they also trained two models from scratch, starting from the pi zero point five base and following the released recipe. One keeps the time encoding and is called CLOCK. The other drops it and is called NOCLOCK. Everything else is identical.

Every condition runs fifty test episodes, paired across conditions. The authors also warn that evaluation isn't fully deterministic. Two runs of one condition differed by eight episodes, so gaps under about ten deserve caution.

### Results

Start with the first figure. The task is three repetitions, then press. With no pause, the robot does all three and presses, and succeeds in forty-seven of fifty episodes. Add two five-second pauses that enter memory, and it presses after only two repetitions, at almost the same moment as without pauses. Success drops to sixteen. Hide the pauses from memory, and it's back to forty-five.

Across the board, two five-second pauses take PickXtimes from forty-seven to sixteen, and SwingXtimes from forty-four to ten. Wrist fidgets do as much damage, and air-grasps still halve success on SwingXtimes. The paper reports that pauses of just one and a half seconds already cause most of the loss.

How it fails matters too. Nearly every failure is an early stop: the robot presses the button before it has done enough repetitions. The authors call this phantom progress. Split by K, most episodes with K equal to one still succeed, because pressing right after the first repetition is correct anyway. With K of two or more, not a single episode succeeds under pauses or fidgets.

And in the hidden condition, every cell returns to its no-event level. The robot's body does exactly the same thing; only its memory changes. So the problem is the memory.

Now the why. The authors look at the demonstrations and try to guess how many repetitions are done from elapsed time alone. That guess is right eighty-nine percent of the time on PickXtimes and eighty percent on SwingXtimes, roughly twice as often as always guessing the most common count. On three-repetition demonstrations from two real teleoperation datasets, it's just as accurate. And demonstrations contain almost no idle stretches, so training never punishes a policy for reading the clock.

Then comes a very clean intervention. No null events, frames untouched, just multiply the time stamps in memory by a factor. On PickXtimes, make the clock run one and a half times faster, and success falls from forty-seven to eighteen, with every failure an early press. Slow it to half speed, and the policy does extra repetitions instead. The failure follows the direction of the clock.

So does dropping the time encoding fix it? No. NOCLOCK, which never sees a time stamp, counts as well as CLOCK normally and ignores rescaled stamps. Yet two pauses still cut its success to twenty-one and fourteen, and hiding them restores it.

The reason is the second route. With frames sampled evenly in time, a pause takes up a share of the thirty-two memory slots, ten slots in the example shown. Controls show the harm comes from those uneventful slots themselves. Replace every pause frame with the last frame before the pause, and the damage is unchanged. With no pause at all, insert growing blocks of identical frames, and NOCLOCK gets steadily worse.

With the diagnosis in hand, they try two fixes. The first changes the data: train with pauses injected into the demonstrations. That prediction failed. On PickXtimes, pauses no longer cost anything, but only because success without any pause had already dropped, from thirty-two to twenty-four. On SwingXtimes, it does no better under pauses than training without augmentation. The authors' post hoc reading is that the policy's counting leaned on time, and lost that support.

The second fix changes the index. Same CLOCK model, no retraining, a change to memory at test time only. If the robot's configuration returns to that of a kept frame, the scene is unchanged, and no contact was made in between, the whole stretch collapses to one frame, as does any long stillness. Time positions are then recomputed over the kept frames, so memory only advances with events.

The result: success at least doubles under all three null events, including the two where the robot moves. Under air-grasps, SwingXtimes goes from thirteen to forty-three. But under pauses on SwingXtimes it still falls short of the no-event level, so part of the damage survives.

One caveat. The detector reads simulator signals, the robot's silhouette, the gripper width and the external force on the arm, and its criterion mirrors how the null events were built. The authors say plainly that this tests whether taking null frames out of the index is enough. It's not a deployable detector.

### Comparison

Now across memory designs. The same null events are applied to every released RoboMME memory policy. All three time-sampled variants collapse. TokenDrop, which keeps tokens by pixel change, is hurt less by pauses, since still frames rarely get in, but more by fidgets, which change pixels without changing the world. The two memories that record progress by sub-goals are unaffected by both.

Two independently built policies on other backbones fit less neatly. Both add to memory at every step, and both read fidgets as progress. But the same pauses barely touch SimpleMemVLA on PickXtimes, while they collapse LTM-pi.

My read is that the most valuable thing here isn't a new model. It's a cheap test: visible versus hidden events, plus rescaled time stamps. You can run it on any memory policy to see whether it tracks events or just reads the clock.

My read is that it's also a warning for memory benchmarks. Counting tasks are meant to test memory, but if demonstrations move at a steady pace, a policy can pass by reading the clock. A high score doesn't prove it remembers what happened.

### Limitations

The authors are candid about limits. The causal analysis covers one policy family, MME-VLA, plus models retrained with its architecture. The two independent policies were probed, not dissected.

Everything is in simulation, and every null event is scripted, so we don't know how often such events happen in real deployment. The event detector uses simulator signals too. And apart from a second seed for CLOCK and NOCLOCK, every model was trained with one seed.

The event index has its own blind spot. On a counting task, a real repetition also returns the world to an earlier state, so it can be mistaken for nothing, which cost significant success on PickXtimes for the released policy. It recovers little of the damage from brief, randomly timed micro-stops. And where time is the task, as in TimedTransfer, collapsing a pause is wrong by construction.

### Future work

The principle the authors propose: memory should advance when the world changes, like an object moving or a contact being made. When duration matters, time should go in as an input to the policy, but what memory keeps should still follow events.

They see detecting those changes from real sensors, which is harder than in simulation, as the main open problem. And in an exploratory study on another benchmark, MIKASA-Robo, a policy appears to infer how many times a light blinked from how long the cue lasted.

My read is that the next thing to watch is real robots. Waiting for a planner or an operator, the cases the paper opens with, are things real robots run into, but the paper doesn't measure how often they happen in deployment. Whether an event index can be built from real sensors, without swallowing genuine repetitions, decides how far this result travels.

That's today's Daily Frontier Papers. The paper is linked in the description.

