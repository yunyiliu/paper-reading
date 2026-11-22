# Game2Player: Scaling Gameplay Experience for Visual Agents through Environment–Player Co-Development

**AI自己改造游戏玩出训练数据**

- 论文 Paper: [arXiv 2611.24154](https://arxiv.org/abs/2611.24154)
- 单位 Affiliation: UIUC 等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-21
- 视频 Videos: —

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇伊利诺伊大学厄巴纳-香槟分校牵头的论文，合作方还有佐治亚理工、IBM 等，名字叫 Game2Player。

它想解决的问题是，让 AI 自己把网上现成的小游戏，改造成能自动打分的训练场，再自己玩出大量数据，教会一个只看屏幕的视觉 Agent 玩游戏。

### 问题

为什么用游戏。每个游戏都有自己的画面、规则和操作，Agent 只能从屏幕上推断，还得掐准时机按键。所以作者认为，游戏经验是训练通用视觉 Agent 的好数据。

难点在于，能用来训练的轨迹，要同时有动作，有明确的目标，还要有可靠的成功信号。

真人录制很贵。网上的游戏视频没有按键记录，动作要靠模型反推，目标和结果也是事后补标的。

可交互的游戏环境倒是天然提供这三样，但每接一个新游戏，都要专门做一遍工程，所以覆盖的游戏很少。

### 以前的方法

表 1 整理了现有资源。比如 GameWorld 基准，有 34 个浏览器游戏，按游戏内部状态打分，但环境不是自动搭的。PLAYER-1 数据集有 172 个游戏，用的是人类操作。

另一条路，是让编程 Agent 自动搭环境。但作者指出，这样搭出来的环境，可能通过了它自己的测试，实际上任务完成不了，或者进度不动，或者成功判定提前触发。这种毛病，往往要真有人去玩才会暴露。

### 核心想法

所以核心想法是，环境和玩家一起开发。玩家在玩的过程中发现问题，反馈回去修，修好了再接着玩。

而且玩本身就是数据。环境一旦验收，玩家就从测试转为采集，每一局都自带任务说明，和基于游戏状态的进度信号。

### 方法

来看总览图。Scout 负责去 itch.io、Newgrounds 这些网站，找能单人离线玩的网页游戏。Builder 是一个编程 Agent，负责把游戏改造成环境。另外还有两个玩家。

Builder 拿到游戏和源码，加上针对 Unity、Flash、Godot、PICO-8 这些引擎的技能说明，先提出任务清单。清单定下就冻结，难的任务不能删，也不能改简单。

每个任务包括一句指令，一段把游戏放到起始局面的程序，一个 0 到 1 的进度函数，和一个成功判定。进度和成功都只读游戏内部变量，所以打分不需要人，也不需要大模型当裁判。

比如图里这个任务，要买下一个钻头升级，进度就读运回去的铁矿数量，买到了才算成功。

还有个细节。打分读的内部变量，玩家程序是读不到的。玩家程序只能读人从屏幕上看得到、或者推断得出的信息，比如扣着的牌就看不到。

两个玩家看的东西不一样。VLM 玩家是一个视觉语言模型，只看截图，每一步游戏都暂停，它给出一个动作，再附一句理由。

Coding 玩家是一个编程 Agent，给每个游戏写一个 Python 策略程序，实时玩，不调用任何模型。

这张图是两个玩家玩同一个任务。代码策略每一帧都在按键，63 秒完成。VLM 玩家每步暂停思考，100 步之后，进度是 0.37。

玩家遇到问题，会提交报告，附上录像。一种是缺陷报告，比如菜单跑出屏幕、按键没反应、进度不动。另一种是观察请求，只有 Coding 玩家会提，就是程序需要某个屏幕上看得到、却读不到的量。不允许要隐藏信息，要捷径，或者要求把任务改简单。

Builder 不会直接改。玩家可能把自己的失误当成环境的毛病，所以 Builder 先回放录像，对照状态记录核实，被证据否定的判为驳回，其余的才修。所有报告处理完，再过一道脚本检查，环境才算验收。

最后得到 568 个游戏环境，3436 个任务，覆盖 25 种以上的类型和 7 种主要引擎。其中 50 个游戏、287 个任务单独拿出来，做成测试基准 PlayerBench，剩下 518 个游戏用于训练。

两个玩家一共录了 1.57 万条轨迹，307 小时。代码策略的录像被切成和 VLM 玩家一样的格式，最终得到 156 万条训练样本，百分之 87.7 来自代码策略。

训练方法叫进度加权 SFT。一局游戏里，大量时间在等待、乱走、从失误中恢复，真正推进任务的只有少数动作，比如一次时机刚好的跳跃。

所以，进度上涨之前的最后几个动作，权重调高。失败和超时那几局最后的动作，以及没带来进展的重复动作，权重调低或去掉。样本特别多的游戏也整体降权。

训练用的是 Qwen3.5 的 4B 和 9B，视觉编码器冻结，得到 Game2Player-4B 和 9B。

### 实验和效果

先看一起开发有没有用。作者用同样的 20 个游戏、145 个任务比了三种设置，Builder 用 GPT-6.1 Terra，玩家用 GPT-6.1 Sol。

看这张图。同一个模型，不用论文的技能和工具，当通用编程 Agent 去搭，120 个环境是错的，只解决了 1 个任务。用论文的 Builder，但不让玩家提报告，错误 51 个，解决 34 个。完整流程，错误降到 40 个，解决提高到 46 个。

比如 There Is No Game，Coding 玩家报告进度卡在 0.9，修复后九个任务全部正确、全部解决，没有报告的那一版九个全错。代价是每个游戏的模型成本增加百分之二十。

采集成本差别更大。用 GPT-6.1 Sol 当 VLM 玩家，玩一小时要 54.7 美元。代码策略写一次 10.8 美元，之后每小时 0.13 美元。每小时录下的步数，代码策略是 7775 步，VLM 玩家只有 354 步。

再看主结果，表 3。同时测了两个基准的模型，除了一个以外，在 PlayerBench 上都更低，进度平均低 12.8 个百分点。15 位第一次玩这些游戏的真人，进度是百分之 69.4。

Game2Player-9B 开启思考，在 PlayerBench 上，进度从百分之 14.3 提到 31.2，翻了一倍多。在 GameWorld 上，从百分之 30.0 提到 53.1，而 GameWorld 和他们的数据没有共用的游戏和任务。

4B 模型也差不多，PlayerBench 从 15.9 到 31.1，GameWorld 从 26.9 到 49.5。作者在关掉思考时跑了四个种子，标准差在 2.5 个点以内，远小于提升。

还有一个发现，训练让思考变得有用。训练之前，开不开思考影响很小，有正有负。训练之后，开启思考在 GameWorld 上，4B 和 9B 分别多出 14.6 和 19.2 个点。

### 和以前方法的对比

和其他模型比。在 GameWorld 上，9B 的 53.1，超过了同一套测试框架下的 Qwen3.8-Flash、GLM-5.3-Flash、GPT-6 Luna 和 Claude Haiku 5.5，接近 Claude Sonnet 5.5 的 54.7。它也高于专门做游戏的 PLAYER-1，不过 PLAYER-1 这个分数，是按它自己的测法，400 步，每个任务三次。

消融在表 4，都是关掉思考测的。VLM 玩家的数据效率高得多，19 万条样本，就把 4B 模型在 PlayerBench 上从 12.7 提到 25.1，137 万条代码策略样本只到 17.3。原因是它的格式和测试一样，还带着理由和思考。

但在没见过的 GameWorld 上，只用代码策略数据反而最好。作者认为，代码策略按键的时机和时长很精确，这种能力能迁移到别的游戏。

进度加权也有用。样本数一样时，9B 模型在 PlayerBench 上，从均匀采样的 23.9 提到 28.6。

最直观的案例是时机控制。平台跳跃游戏 Give Up，方向键按多久，基本决定了跑多远、跳多远。100 步内，基座 4B 模型进度是百分之 12.3，训练后是 92.5。训练后的模型，百分之 47 的按键会明确设定按住多久，基座只有百分之 1。

我的判断是，这篇最有价值的，不是又多一个基准，而是让玩家去验收环境。环境搭错了，Builder 自己的测试查不出来，得有人真去玩。这个思路放到搭网页、搭软件环境训练 Agent，应该也成立。

我的判断是，差距也要看清。在 PlayerBench 上，9B 的进度是 31.2，最强的闭源模型在 60 左右，第一次玩的真人是 69.4。开源小模型还差得很远。

### 局限

作者列了几条局限。第一，通过验收的环境不保证正确，没有全量人工审核。在那 20 个游戏的审核里，Terra 做 Builder，百分之 72 的任务正确，Sol 做 Builder 是百分之 94。不过 PlayerBench 的 287 个任务，人工逐个检查过，没发现错误。

第二，只覆盖浏览器里离线运行的单人网页游戏，联网、多人和大型商业游戏不在范围内。

第三，VLM 玩家每步都暂停游戏，分数衡量不了实时反应。另外 Builder、玩家和很多对比模型都是闭源模型，版本会变，换模型重建，结果可能不一样。

### 未来方向

作者的结论是，视觉 Agent 可以靠搭建、验证、再从现有游戏里学习，来扩大游戏经验。从训练曲线看，到最后一个检查点，PlayerBench 的进度还在涨。论文首页列出了项目网站、代码和模型的链接。

我的判断是，目前只用了 SFT，也就是模仿学习。每个任务都有不需要裁判的进度分数，这天然就是强化学习的奖励信号，下一步很可能往这个方向走。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by the University of Illinois Urbana-Champaign, with co-authors from Georgia Tech, IBM and others. It's called Game2Player.

The pitch: let AI agents turn ordinary web games into environments that score themselves, play them to generate training data at scale, and use that data to teach a screen-only visual agent to play.

### Problem

Why games? Every game has its own visuals, rules and controls, which an agent has to work out from the screen, while acting with precise timing. That variety is exactly why the authors see gameplay as valuable training data for general visual agents.

The hard part is the data. A useful trajectory needs three things at once: the actions, an explicit goal, and a reliable signal of success.

Existing sources fall short. Human recordings are expensive. Gameplay videos online have no actions, so those have to be inferred by a model, and goals and outcomes get labeled after the fact.

Interactive game environments give you all three for free, but every new game needs its own engineering, so coverage stays small.

### Prior work

Table 1 lays out existing resources. GameWorld, for example, has thirty-four browser games scored from internal game state, but its environments aren't built automatically. PLAYER-1 covers one hundred seventy-two games using human actions.

The other route is to let a coding agent build environments automatically. The catch, the authors point out, is that such an environment can pass the coding agent's own tests while a task is actually impossible, progress never updates, or the success check fires too early. Those bugs often only show up when someone actually plays.

### Key idea

So the key idea is to develop the environments and the players together. Players find problems while playing, report them, the environment gets fixed, and they play again.

And the playing is itself the data. Once an environment is accepted, the players switch from testing to collecting, and every episode comes with its task instruction and a dense progress signal read from game state.

### Method

Here's the overview. A Scout searches sites like itch dot io and Newgrounds for web games one person can play offline. A Builder, which is a coding agent, turns each game into an environment. And then there are two players.

The Builder gets the game, its source code, and skill files for engines like Unity, Flash, Godot and PICO-8, and first proposes a task plan. That plan is then frozen: hard tasks can't be dropped or made easier.

Each task has a natural-language instruction, a routine that sets up the starting situation, a progress function between zero and one, and a success check. Both read only internal game variables, so scoring needs no human and no model acting as judge.

In the example from the figure, the task is to buy a drill upgrade. Progress reads how much iron has been delivered, and success means the drill is bought.

There's a subtle point here. The internal variables used for scoring are off-limits to the players' code. A player program can only read what a person could see or infer from the screen, so, for instance, face-down cards stay hidden.

The two players see different things. The VLM Player is a vision-language model that sees only screenshots. The game pauses at every step, and the model returns one action plus a one-sentence reason.

The Coding Player is a coding agent that writes one Python policy per game, which plays in real time without calling any model.

This figure shows both players on the same task. The code policy presses keys every frame and finishes in sixty-three seconds. The VLM Player pauses to think at each step, and after one hundred steps its progress is zero point three seven.

When a player hits a problem, it files a report with recordings attached. A defect report covers things like a menu that runs off the screen, a key that does nothing, or progress that never moves. An observation request, which only the Coding Player files, asks for a quantity that's visible on screen but that the program can't read. Reports may not ask for hidden information, shortcuts, or easier tasks.

The Builder doesn't just take a report at face value. A player can mistake its own error for a bug, so the Builder first replays the recording and checks the state trace, marks the report disproved if the evidence contradicts it, and repairs the environment otherwise. Once every report is resolved and a scripted gate passes, the environment is accepted.

The result is five hundred sixty-eight game environments with three thousand four hundred thirty-six tasks, across more than twenty-five genres and seven major engines. Fifty games, with two hundred eighty-seven tasks, are held out as a test benchmark called PlayerBench, and the other five hundred eighteen are used for training.

Together the players recorded fifteen point seven thousand trajectories, or three hundred seven hours of play. Code policy recordings are cut into the same step format as the VLM Player's, giving one point five six million training examples, eighty-seven point seven percent of them from code policies.

The training recipe is called progress-weighted SFT. Most of an episode is waiting, wandering, or recovering from mistakes, and only a few actions, like a well-timed jump, actually move the task forward.

So the last few actions before progress rises get more weight. The final actions of failed or timed-out episodes, and repeated actions that don't help, get less weight or none. Games with very many examples are also scaled down overall.

The models are Qwen3.5 4B and 9B, fine-tuned with the vision encoder frozen, which gives Game2Player-4B and Game2Player-9B.

### Results

First, does co-development itself help? The authors rebuilt the same twenty games and one hundred forty-five tasks in three settings, with GPT-6.1 Terra as the Builder and GPT-6.1 Sol as the players.

Here's the result. The same model as a generic coding agent, without the paper's skills and tools, leaves one hundred twenty environments incorrect and solves one task. The paper's Builder without player reports leaves fifty-one incorrect and solves thirty-four. The full loop gets incorrect down to forty and solved up to forty-six.

In one game, There Is No Game, the Coding Player reported progress stuck at zero point nine, and after the repair all nine tasks were correct and solved, against nine out of nine wrong without reports. Reports add twenty percent to model spending per game.

Data collection costs differ even more. GPT-6.1 Sol as a VLM Player costs fifty-four dollars seventy per hour of gameplay. A code policy costs ten dollars eighty to write, then thirteen cents an hour. Per hour of wall-clock time, a code policy records seven thousand seven hundred seventy-five steps, against three hundred fifty-four for the VLM Player.

Now the main results, in Table 3. Every model tested on both benchmarks except one scores lower on PlayerBench, by twelve point eight progress points on average. Fifteen people playing these games for the first time reach sixty-nine point four percent progress.

With thinking on, Game2Player-9B more than doubles its PlayerBench progress, from fourteen point three to thirty-one point two percent. On GameWorld it goes from thirty to fifty-three point one, even though GameWorld shares no games or tasks with their data.

The 4B model looks similar: fifteen point nine to thirty-one point one on PlayerBench, and twenty-six point nine to forty-nine point five on GameWorld. Across four seeds with thinking off, the standard deviation stays within two point five points, well below these gains.

Another finding: training makes thinking useful. Before training, turning thinking on barely matters, and the effect goes both ways. After training, it adds fourteen point six and nineteen point two points on GameWorld for the 4B and 9B models.

### Comparison

How does it stack up? On GameWorld, the 9B model's fifty-three point one beats Qwen3.8-Flash, GLM-5.3-Flash, GPT-6 Luna and Claude Haiku 5.5 under the same harness, and comes close to Claude Sonnet 5.5 at fifty-four point seven. It's also above the game-specific PLAYER-1, though PLAYER-1's GameWorld numbers come from its own protocol, with four hundred steps and three trials per task.

The ablations are in Table 4, all with thinking off. VLM Player data is far more sample-efficient: one hundred ninety thousand examples lift the 4B model from twelve point seven to twenty-five point one on PlayerBench, while one point three seven million code policy examples only reach seventeen point three. It matches the test format and carries the model's reasons and thinking.

But on the unseen GameWorld games, code policy data alone does best. The authors' explanation is that the precisely timed key presses and holds transfer to other games.

Progress weighting helps too. At the same number of examples, the 9B model goes from twenty-three point nine with uniform sampling to twenty-eight point six on PlayerBench.

The clearest case study is timing. In the platformer Give Up, how long you hold a direction key largely decides how far you run and jump. Within one hundred steps, the base 4B model reaches twelve point three percent progress, and the trained model ninety-two point five. The trained model sets an explicit hold duration on forty-seven percent of its key presses, against one percent for the base model.

My read is that the most valuable part isn't another benchmark, it's making players sign off on the environment. A Builder's own tests miss bugs that only show up in play. That idea should carry over to building web and software environments for agent training too.

My read is also that the gap is still large. On PlayerBench, the 9B model reaches thirty-one point two, the strongest closed models are around sixty, and first-time human players sixty-nine point four. Small open models have a long way to go.

### Limitations

The authors list several limitations. First, accepted environments aren't guaranteed correct, and there was no full human audit. In the twenty-game audit, seventy-two percent of tasks were correct with Terra as the Builder, and ninety-four percent with Sol. That said, all two hundred eighty-seven PlayerBench tasks were checked by hand, and no errors were found.

Second, it only covers single-player web games that run offline in a browser. Online, multiplayer and large commercial games are out of scope.

Third, the VLM Player pauses the game at every step, so the scores don't measure real-time play. And the Builder, the players and many baselines are proprietary models whose versions change, so a rebuild with other models could come out differently.

### Future work

The authors' conclusion is that visual agents can scale up their gameplay experience by building, verifying and learning from existing game worlds. Their training curves show PlayerBench progress still rising at the final checkpoint. The paper's first page links to the project website, code and models.

My read is that training here is SFT only, which is imitation. Every task already has a judge-free progress score, which is a natural reward for reinforcement learning, and that looks like the obvious next step.

That's today's Daily Frontier Papers. The paper is linked in the description.

