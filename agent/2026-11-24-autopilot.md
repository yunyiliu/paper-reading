# AutoPILOT: A Visual Agent Harness for Real-Time Gameplay

**AI想太慢？让它写个自动驾驶**

- 论文 Paper: [arXiv 2611.28202](https://arxiv.org/abs/2611.28202)
- 单位 Affiliation: UIUC · 佐治亚理工 · 西北大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-24
- 视频 Videos: [B站](https://www.bilibili.com/video/BV11syfzKEjK)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇伊利诺伊大学厄巴纳-香槟分校，也就是 UIUC 牵头的论文，合作方是佐治亚理工和西北大学，题目叫 AutoPILOT，一个给实时游戏用的视觉 Agent harness。

它想回答一个很具体的问题，大模型玩游戏，游戏等它的时候玩得不错，游戏不等它的时候，为什么一下子就不行了。

### 问题

先解释一个词，harness。它指的是包在模型外面的那一层系统，决定模型看到什么，记住什么，以及模型的输出怎么变成一个个动作。

作者说，以前的工作主要在改进前两件事。第三件通常是默认的，一次模型调用，最多做一个动作，而且假设环境会等模型想完。

世界一直在跑的时候，这个默认就成了瓶颈。论文开头这张图，是射击游戏 Praxis Fighter X，模型想了 4.1 秒，等动作落下去，飞船已经被击毁了。

数字也很直观。在 GameWorld 这个基准上，只要让游戏时钟一直走，VISTA 的成功率就从 84.1% 掉到 42.9%。VISTA 是之前在 GameWorld 和 ARC-AGI-3 上表现最好的视觉 harness。

更好的记忆和视觉工具，也解决不了这个问题。实时条件下，VISTA 和最朴素的 Codex harness 差不多，38.3% 对 38.9%。多花推理预算，也只给 VISTA 加了 4 到 7 个点。

### 以前的方法

以前的路大概有几类。大多数 Agent 沿用 ReAct 的循环，想一步，做一步。很多游戏 Agent 和游戏基准，干脆在模型生成的时候，把游戏暂停。

也有人给慢模型配一个快的执行层，比如单独的反应线程，事先固定好的宏动作，或者提前编好的反射规则。

还有一类，是让语言模型写程序来当策略。但作者指出，这些程序大多读的是结构化的游戏状态，而且环境会等它。

这张表把相关工作放在一起比。作者的概括是，这些系统里的快速通道，大多是另一次模型推理，事先写死的代码，或者某个游戏专用执行器里的规则。

### 核心想法

AutoPILOT 的想法，名字里就有。就像飞行员和自动驾驶仪，模型把日常操控交出去，被叫回来的时候再接手。

具体说，就是让一次模型调用覆盖很多个环境步。模型自己决定这一次调用管多少步，动作也不用再等下一次调用。

### 方法

AutoPILOT 建在 VISTA 之上。模型看到的仍然是游戏画面，时钟和剩余预算，看不到游戏内部状态。两个 harness 除了委托执行这一块，工具是一样的。

每次模型调用，可以从三种执行方式里选一种。第一种是基本动作，直接发一个按键或者鼠标事件。

第二种是开环宏，模型写好一串动作，后台按顺序发出去，中间不看屏幕。比如这个推箱子类的解谜游戏，模型一次发了九个方向键。

第三种是闭环控制器，这是关键。模型在运行时现写一个简短的 Python 程序，它自己读屏幕，自己做判断，自己按键，大约每 0.4 秒一次，不用再调用模型。

控制器在沙盒里跑，最多 120 秒，只有一个 CPU，1 GB 内存，不能联网。

这就是论文里一个真实的控制器。它按颜色找到自己的飞船和附近的子弹，一边开火一边躲，循环结束，就把控制权交回模型。这一段连续打了 103 步，直到通关。

交出去容易，收回来才是安全的关键。程序在跑的时候，模型那边是阻塞的，它不产生 token，就在那等着。

一共有六种唤醒事件，会把控制权交回模型。程序这边，控制器主动请求交接，程序到了停止条件或者出错，宏动作发完。harness 这边，游戏等玩家做选择，画面大变，或者程序跑满 120 秒。

输了重开的时候，游戏不会暂停。正在跑的控制器会接着玩下一局，模型到下一次被唤醒才知道失败了，这时它可以反思，再写一个改进版的控制器。

### 实验和效果

为了测实时表现，作者做了一个基准，叫 AFK-Bench，AFK 是 Away From Keyboard，离开键盘的意思。一共 120 个浏览器游戏。

按 Agent 什么都不做时会发生什么，分成三类，每类 40 个。敌对实时，不动就可能输。良性实时，世界在变，但不动不会输。回合制，只有 Agent 动了，游戏才往前走。

每个游戏都能用两种时钟来玩，实时，或者模型思考时暂停。两者之间的落差，就是 Agent 因为自己太慢，损失了多少。

每局有预算，100 次模型调用，2000 秒游戏时间，2000 个环境步，用完任何一个就结束。输了可以在预算内重开。主要指标是成功率。

对比对象是 VISTA，以及两个模型自带的朴素 harness，Codex 和 Claude Code。主力模型是 GPT-6.1 Sol 和 Claude Opus 5.5，另外还测了四个模型，主实验全部用低推理强度。

主结果在表 2。实时条件下，用 GPT-6.1 Sol，AutoPILOT 成功率 68.6%，VISTA 是 38.3%，Codex 是 38.9%。用 Claude Opus 5.5，AutoPILOT 是 53.3%，VISTA 是 30.0%，Claude Code 是 32.5%。

差距最大的是敌对实时游戏，GPT-6.1 Sol 下，62.5% 对 16.7%。换成另外四个模型，AutoPILOT 也都领先，幅度在 13.3 到 29.2 个点之间。

成本反而更低。GPT-6.1 Sol 下，AutoPILOT 每局 0.89 美元，VISTA 是 1.50 美元，token 用量是 280 万对 530 万。按总花费算，AutoPILOT 只用五分之一的钱，就达到了 VISTA 的最终成功率。

它对时钟也不敏感。从暂停换到实时，GPT-6.1 Sol 下，VISTA 掉了 17.8 个点，AutoPILOT 只掉了 4.5 个点。

再换一个和 AFK-Bench 不重叠的基准 GameWorld，170 个任务。暂停时两者差不多，84.1% 对 82.9%。实时下 VISTA 掉到 42.9%，AutoPILOT 还有 80.0%。

### 和以前方法的对比

那它为什么赢。即使暂停时钟，动作不会过时，AutoPILOT 用 GPT-6.1 Sol 还是比 VISTA 高 17.0 个点，用 Claude Opus 5.5 高 21.7 个点。所以延迟只解释了一部分。

另一部分是预算。VISTA 一次调用只做一个动作，实时条件下，它的 222 次失败里，有 208 次是把 100 次调用用光了，平均每局只走了 78 步。AutoPILOT 平均用 43 次调用发出 681 步，每次调用大约 16 步。

消融在表 4。去掉控制器，敌对实时游戏从 62.5% 掉到 50.8%。宏和控制器都去掉，整体掉到 45.8%，敌对实时只剩 22.5%。

重开也很关键。在敌对实时游戏里，如果不重开，AutoPILOT 只赢 11.7%，VISTA 是 8.3%。允许最多 20 次重开，AutoPILOT 爬到 54.2%，而且还在涨，VISTA 在大约七次之后，就停在 15.8%。

放宽预算，VISTA 也追不上。给 VISTA 三倍的模型调用，它从 37.5% 涨到 55.0%，但每局成本 5.28 美元，是 AutoPILOT 的 6 倍，而 AutoPILOT 在标准预算下就是 69.2%。

推理强度也是这样。用 Claude Opus 5.5 把推理调高，AutoPILOT 从 53.3% 涨到 69.2%，VISTA 只从 30.0% 涨到 36.7%。作者的解释是，想得更久，写出来的程序更好，这个程序又能连续执行很多步。

我的判断是，这篇真正的贡献，是把 harness 设计里一个被默认掉的问题摆上了台面，就是模型的输出怎么随时间变成动作。模型不变，主要改的是这一层，实时成功率接近翻倍，这个信号比具体分数更重要。

我的判断是，也要看清对比的范围。基线都是只看屏幕，不做训练的视觉 harness，读文本状态或者需要训练的方法，作者说不适用这个设定，所以没有比。而且除了 GPT-6.1 Sol，其他模型每个任务只跑了一局。

### 局限

作者自己写了三条局限。第一，AutoPILOT 依赖模型自己写的感知和控制代码，这在 2D 浏览器游戏里比较容易，在杂乱的 3D 场景里会更难。

第二，在敌对实时游戏里，大部分胜利来自重开之后，第一次就打赢一个没见过的实时游戏，还是没解决的问题。

第三，模型很少在交接之后让控制器继续玩，作者试着把这设成默认，也没有多赢。

另外，论文里逐帧回放的几个案例，是作者挑出来的示范，不来自主表的运行。

### 未来方向

作者在结论里说，模型的输出怎么随时间变成动作，在 harness 设计里，应该和模型看到什么，记住什么，受到同样的重视，这不只是游戏 Agent 的事，也关系到操作实时界面和机器人的 Agent。

我的判断是，最值得跟的，是这套做法能不能迁到真实的电脑操作和机器人上。那里的画面复杂得多，靠颜色阈值写出来的控制器不一定站得住，这一点论文还没有验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by the University of Illinois Urbana-Champaign, with Georgia Tech and Northwestern. It's called AutoPILOT, a visual agent harness for real-time games.

The question behind it is very concrete. Frontier models play video games pretty well when the game waits for them. So why do they fall apart when the game keeps running?

### Problem

First, a word on the term harness. It's the system wrapped around the model. It decides what the model sees, what it remembers, and how its outputs turn into actions.

The authors argue that prior work has mostly improved the first two. The third is usually left as a default: one model call produces at most one action, and the environment is assumed to wait while the model thinks.

When the world keeps moving, that default becomes the bottleneck. The opening figure shows the shooter Praxis Fighter X. The model thinks for four point one seconds, and by the time its action lands, the ship is already gone.

The numbers make the point. On the GameWorld benchmark, simply letting the game clock run drops VISTA's success rate from eighty-four point one percent to forty-two point nine. VISTA was the best previous visual harness on GameWorld and ARC-AGI-3.

Better memory and vision tools don't fix it. In real time, VISTA does about as well as the plain Codex harness, thirty-eight point three versus thirty-eight point nine. And spending more on reasoning only adds four to seven points to VISTA.

### Prior work

Previous approaches fall into a few groups. Most agents follow the ReAct loop: think one step, act one step. Many game agents and game benchmarks simply pause the game while the model generates.

Others pair a slow model with a fast layer, such as a separate reactive thread, a fixed set of macro actions, or reflex rules compiled in advance.

And there's a line of work where language models write programs that act as policies. But as the authors point out, those programs mostly read structured game state, in environments that wait for them.

This table lines up the related work. The authors' summary is that in these systems, the fast path is mostly another model inference, code fixed in advance, or rules for a game-specific executor.

### Key idea

The idea is right there in the name. Like a pilot with an autopilot, the model hands off routine control and takes over again when it's called back.

Concretely, one model call can cover many environment steps. The model decides how many steps each call covers, and its actions no longer have to wait for the next call.

### Method

AutoPILOT is built on top of VISTA. The model still sees rendered frames, the game clock, and its remaining budgets, but never the game's internal state. Apart from delegation, the two harnesses expose the same tools.

On each model call, the model picks one of three execution modes. The first is a primitive action: one key press or mouse event, sent straight to the game.

The second is an open-loop macro. The model writes a sequence of actions, and a background worker sends them in order without looking at the screen. In this box-pushing puzzle, the model sends nine arrow keys in one go.

The third is a closed-loop controller, and this is the key part. At run time, the model writes a short Python program that reads the screen, decides with ordinary code, and presses keys about every zero point four seconds, with no further model calls.

Controllers run in a sandbox for up to one hundred and twenty seconds, with one CPU, one gigabyte of memory, and no network access.

Here's a real controller from the paper. It finds its own ship and nearby bullets by color, dodges while firing, and hands control back when its loop ends. This one played one hundred and three steps straight through to the win.

Handing off control is easy. Getting it back is what makes delegation safe. While a program runs, the model's call simply blocks. It produces no tokens and just waits.

There are six wake events that return control to the model. From the program side: the controller asks for a handoff, the program reaches the stopping condition it was given or hits an error, or a macro finishes sending its actions. From the harness side: the game stops at a decision screen, the screen changes substantially, or the program hits the one-hundred-twenty-second limit.

Restarts after a loss don't pause the game. A running controller just keeps playing the next attempt, and the model learns about the failure at its next wake event. At that point it can reflect and write a revised controller.

### Results

To measure real-time play, the authors built a benchmark called AFK-Bench, as in Away From Keyboard. It has one hundred and twenty browser games.

The games are split into three types of forty, based on what happens when the agent does nothing. In hostile real-time games, doing nothing can end the attempt. In benign real-time games, the world moves on, but idling doesn't end the attempt. In turn-based games, nothing happens until the agent acts.

Every game can be played under two clocks: real time, or paused while the model thinks. The drop between the two measures how much an agent loses to its own slowness.

Each episode has a budget of one hundred model calls, two thousand seconds of game time, and two thousand environment steps. It ends when any of these runs out, and after a loss the agent can restart within the same budget. The main metric is success rate.

The baselines are VISTA plus the two models' own plain harnesses, Codex and Claude Code. The main models are GPT-6.1 Sol and Claude Opus 5.5, with four more models tested as well, and all main runs use low reasoning effort.

The main results are in Table 2. In real time with GPT-6.1 Sol, AutoPILOT succeeds sixty-eight point six percent of the time, against thirty-eight point three for VISTA and thirty-eight point nine for Codex. With Claude Opus 5.5, it's fifty-three point three, against thirty point zero for VISTA and thirty-two point five for Claude Code.

The gap is widest in hostile real-time games: sixty-two point five versus sixteen point seven with GPT-6.1 Sol. With each of the four other models, AutoPILOT also leads, by thirteen point three to twenty-nine point two points.

And it's cheaper. With GPT-6.1 Sol, AutoPILOT costs eighty-nine cents per episode against a dollar fifty for VISTA, and uses two point eight million tokens against five point three million. Looking at total spend, AutoPILOT reaches VISTA's final success rate at a fifth of the cost.

It also barely cares about the clock. Going from paused to real time, VISTA loses seventeen point eight points with GPT-6.1 Sol. AutoPILOT loses four point five.

On a second benchmark, GameWorld, with one hundred and seventy tasks that don't overlap with AFK-Bench, the two are close with the clock paused, eighty-four point one versus eighty-two point nine. In real time, VISTA falls to forty-two point nine, while AutoPILOT holds eighty point zero.

### Comparison

So why does it win? Even with the clock paused, where no action is stale, AutoPILOT beats VISTA by seventeen points with GPT-6.1 Sol and twenty-one point seven with Claude Opus 5.5. Latency is only part of the story.

The rest is budget. VISTA spends one model call per action. In real time, two hundred and eight of its two hundred and twenty-two failures come from running out of its one hundred calls, after only seventy-eight steps per episode on average. AutoPILOT sends six hundred and eighty-one steps from forty-three calls, about sixteen steps per call.

The ablations are in Table 4. Remove controllers, and hostile real-time success drops from sixty-two point five to fifty point eight. Remove both controllers and macros, and overall success falls to forty-five point eight, with hostile games down to twenty-two point five.

Restarts matter a lot too. In hostile games with no restarts, AutoPILOT wins eleven point seven percent and VISTA eight point three. Allow up to twenty restarts, and AutoPILOT climbs to fifty-four point two and is still rising, while VISTA stalls at fifteen point eight after about seven.

Giving VISTA more budget doesn't close the gap. With three times the model calls, VISTA goes from thirty-seven point five to fifty-five percent, but at five dollars twenty-eight per episode, six times AutoPILOT's cost. AutoPILOT gets sixty-nine point two on the standard budget.

Same with reasoning effort. With Claude Opus 5.5 at high effort, AutoPILOT rises from fifty-three point three to sixty-nine point two, while VISTA only goes from thirty to thirty-six point seven. The authors' explanation: a longer call produces a better program, and that program then acts for many steps.

My read is that the real contribution here is putting a default up for design: how model outputs turn into actions over time. Same model, mostly that layer changed, and real-time success nearly doubles. That signal matters more than any specific score.

My read is also that the comparison has a clear scope. The baselines are all screen-only, training-free visual harnesses. Methods that read text state or need training were judged not applicable to this setting, so they weren't compared. And apart from GPT-6.1 Sol, every model ran just one episode per task.

### Limitations

The authors list three limitations. First, AutoPILOT relies on perception and control code the model writes itself, which is easier in 2D browser games than in cluttered 3D scenes.

Second, in hostile real-time games, most wins come after a restart. Winning an unseen real-time game on the first attempt is still open.

Third, models rarely let a controller keep playing after a handoff, and making that the default didn't add wins.

One more caveat: the frame-by-frame case studies in the paper are hand-picked illustrations, not taken from the main-table runs.

### Future work

In their conclusion, the authors argue that how outputs become actions over time deserves as much attention in harness design as what the model sees and remembers, and not only for game agents, but for agents running live interfaces or robots.

My read is that the thing to watch is whether this transfers to real computer use and robots. Scenes there are far messier, and a controller built on color thresholds may not hold up. The paper doesn't test that yet.

That's today's Daily Frontier Papers. The paper is linked in the description.

