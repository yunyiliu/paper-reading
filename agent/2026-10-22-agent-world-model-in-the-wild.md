# Agent World Model in the Wild: Computer-Use Agent Environment Synthesis

**开源模型自己造电脑练Agent**

- 论文 Paper: [arXiv 2610.26828](https://arxiv.org/abs/2610.26828)
- 单位 Affiliation: 北卡罗来纳大学教堂山分校 · 微软研究院等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-22
- 视频 Videos: [YouTube 中文](https://youtu.be/cUNFS8XcRco) · [YouTube English](https://youtu.be/We_2WrK5llQ)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇北卡罗来纳大学教堂山分校和微软研究院等单位的论文，名字叫 Agent World Model in the Wild，简称 WildAWM。

它要回答的是，训练会用电脑的 Agent，练习用的电脑和题目从哪里来。论文的答案是，全部用开源模型自动造出来。

### 问题

现在 Agent 的能力，越来越多靠强化学习，也就是在能真正执行操作的环境里反复尝试，做对了给奖励。这样一来，环境够不够，就成了瓶颈。

直接在真实世界里练，又贵又有风险。状态难重置，线上服务会变，Agent 乱点一下，可能真的造成后果。人工搭环境可控，但成本高，很难规模化。

电脑操作 Agent 更复杂。同一个任务，可以在图形界面上点，可以在命令行里敲，也可以调用 MCP 服务。MCP 是让模型调用外部工具的标准接口。所以选哪个接口、什么时候切换，本身就是一种能力。

造题还有三个难点。题目要做得出来，要足够难，还要配一个可靠的自动判分器。而且不能假设，最强的模型做对了就一定对。

### 以前的方法

论文的表 1 列了几种已有的合成方法。它们大多只覆盖一个应用，接口上也大多只支持一种，比如只做网页，或者只做命令行。

而且它们造环境，用的都是闭源模型，比如 GPT-5、Claude Opus 4.6、Gemini 3 Pro。作者认为，这不利于复现和开放。

### 核心想法

WildAWM 的想法有两层。第一层，不造孤立的小应用，而是给一个虚构的人，造一整台个人电脑，里面的网站、MCP 服务和本地文件，是这个人一天天用出来的，带着历史。

第二层，题目好不好，不靠一个强模型说了算，而是让很多不同的 Agent 真的去做，拿执行结果当证据，再过五道质量关卡。

### 方法

这张总览图，左边造世界，中间造题目，右边用造好的环境、题目和判分器做强化学习。

先看造世界。生成器是一个编程 Agent，用开源模型 GLM-5.2，跑在 OpenCode 里。它先从零写出一个人物，包括职业、联系人、常用网站和正在做的事。

然后按天规划这个人在电脑上做了什么，一天一天落实成数据库记录和文件，每天都接着前一天的状态。为了有真实的网页画面，还接入了 15 个镜像网站，比如 Amazon、Booking、Coursera 和 GitHub。

常见的事件，比如报一门课，会沉淀成可复用的技能，缺的 MCP 服务就现写一个，放进共享库。每天的变化写成确定性的程序，回放两次结果必须完全一样，再由另一个 Agent 检查是否符合计划。

最后得到 36 个人物，640 个世界状态，83 个可复用技能。

再看造题目。生成器在这个环境里探索，提出任务，每个任务都必须同时用到图形界面、MCP 和命令行。同时写好判分器、评分细则和提示。判分器是一组是或否的检查项，全部通过才算成功。

然后找人做题。三个开源模型，GLM-5.2、DeepSeek-V4-Flash 和 MiniMax-M3，搭配六种 Agent 框架，一共 18 种组合。每道题跑 6 次，一半给提示，一半不给。

每次执行，要经过三层判断，程序判分器，大模型评审，还有一个会进环境检查的 Agent 评审。

接下来是五道关卡。第一道，判断一致，三层判断在 6 次执行上都要给出相同结论。第二道，初始不满足，什么都不做的时候，判分器必须判失败。

第三道，可解，至少成功一次，而且成功的那次真的用到了声明的所有接口。第四道，难度，不给提示的 3 次里，最多成功 1 次算难，2 次算中等，3 次全成功就太简单，退回去加难度。

第五道最有意思，叫反事实探测。大模型容易顺着人说话，所以作者故意告诉 Agent 评审一个相反的结论，让它重新调查。它坚持原判，题目才算通过。

没通过的题，带着诊断信息退回去，最多改两轮。这张漏斗图显示，2551 个候选，最后留下 341 个。

论文举了一个例子。一位音视频技术员，要给体育酒吧的显示墙订浪涌保护器。Agent 先在聊天服务里确认客户同意，再写脚本，把库存表、物料清单和 PDF 规格书合起来，给产品排序。

然后去 Amazon 网页加购物车。两个候选参数完全一样，只能看图片，选白色、带显示屏的那个。最后写备注，发消息通知安装师傅。不给提示的 3 次全部失败，所以这道题算难题。

训练用的是 GRPO 强化学习。做成功拿满分，没做成的按通过的检查项给一点部分分，格式出错扣一分，让长任务的信号更密。

341 道题里，100 道做监督微调，208 道做强化学习，33 道留作评测。基座是 Qwen3.6-35B-A3B 和 Qwen3.5-4B，先用 GLM-5.2 的 100 条轨迹热启动，再做最多 64 步强化学习。

### 实验和效果

先看题有多难。造题时，不给提示的平均成功率只有 21.3%，给提示是 45.6%。在留出的评测集上，GPT-6-Astra 单次通过率 59.85%，试 8 次，还有 27.3% 的题一次都没做出来。

判分器靠不靠谱。随机抽 100 次执行，和几个 Agent 评审的多数意见比，一致率 95%，Cohen's kappa 是 0.90，越接近 1，说明一致越不是碰巧。

主结果在表 5。训练只在合成环境里做，然后拿 8 个没见过的基准来测，覆盖网页 Agent、OpenClaw 这类 claw Agent 和电脑操作 Agent。论文自己测的模型，都用同一个 OpenCode 框架。

两个模型在 8 个基准上全部提升。35B 模型的平均分从 42.84 到 53.42，涨了 10.58 分。4B 模型从 21.83 到 30.04，涨了 8.21 分。

35B 模型涨得多的有 ClawEval 涨 16.67，OSWorld-Verified 涨 16.06，WebArena 涨 13.72。训练后的 35B 模型，WebArena 和 GLM-5.2 打平，都是 67.26，Online-Mind2Web 拿到 80.77，超过 GPT-5.5 的 72.22。

在自己的评测集上，4B 模型从 0.38 到 27.65，35B 模型从 5.68 到 35.98，接近 GLM-5.2 的 37.88。

### 和以前方法的对比

和已有方法比，论文主要比的是环境本身。WildAWM 一个环境状态的中位大小是 5.24 万 token，CUA-Gym 只有 1100 个。环境大得多，多样性指标 Vendi score 也在各个规模下都更高。

表 7 是关卡的消融，每次去掉一道关卡造数据，再训练 4B 模型。去掉任何一道，平均分都掉 2.6 到 9.3 分。去掉判断一致，平均分从 31.86 掉到 22.56，评测集从 21.59 掉到 3.79。不过不是每个基准都掉，去掉可解这一关，Online-Mind2Web 反而更高。

强化学习的消融在表 8。去掉监督微调热启动，掉得最多，平均少 5.31 分。去掉部分分，少 2.25 分。

行为上，训练后在不同接口之间切换的比例，从 18.4% 升到 25.5%，接近老师 GLM-5.2。只用自己的成功轨迹微调，或者跳过微调直接强化学习，切换率反而降到 15.8% 和 15.7%。

作者的解读是，跨接口的习惯来自老师的轨迹，强化学习加的是长流程能力。在 40 到 120 步之间，只做监督微调多解出 9.7 个百分点的题，加上强化学习是 17.0。

我的判断是，这篇最有价值的不是分数，而是一套不靠闭源模型、也不靠单个强模型背书的造题验题流程，用多个不同 Agent 的执行结果，证明题目可解、够难、判分可信。

我的判断是，也要看到，它没有在同样设置下，和其他合成方法训练出的模型正面比较。表里 CUA-Gym 和 OpenForgeRL 的数字取自各自的论文，框架不同，作者也说只供参考。

### 局限

论文没有单独的局限章节，下面几条来自它自己的数据。第一，最难的题基本没进步。按前沿模型能解出多少把题分成四档，最难那一档主要是 OSWorld 2.0 和 Agents' Last Exam，所有训练变体都没有明显提升。4B 模型在 OSWorld 2.0 上，只从 0.87 到 1.03。

第二，离前沿模型还有差距。35B 模型平均 53.42，GPT-6-Astra 是 73.11，Claude Opus 5 是 70.83。

第三，作者发现，提升主要集中在老师 GLM-5.2 也够得着的题上，而热启动用的正是它的轨迹。

第四，成本不低，规模也不大。造一道题，生成器平均要读入大约 6980 万个 token，最后一共 341 道题。几个基准只测了子集，比如 WebArena 只测了 812 题里的 226 题。

### 未来方向

作者在附录里说，如果算力允许，会研究更大的模型能不能更好地利用这些数据，攻下最难的那一档。代码已经在 GitHub 公开。

作者认为，这条路能让开源模型自己造世界，在里面学习，最终成为通用的电脑操作 Agent。

我的判断是，Agent 训练的瓶颈，正在从模型转向环境，谁能便宜、安全、大规模地造出可信的练习场，谁就有优势。这篇说明，开源模型已经能造出连 GPT-6-Astra 都只做对六成的题。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from UNC-Chapel Hill and Microsoft Research, with several other groups. It's called Agent World Model in the Wild, or WildAWM.

The question: if you want to train an agent that uses a computer, where do the practice computers and practice tasks come from? The paper's answer is to generate all of it, using only open-source models.

### Problem

Agents increasingly learn through reinforcement learning: they try things over and over in an environment that really executes their actions, and get rewarded when they succeed. So the supply of environments becomes the bottleneck.

The real world is a risky place to train. States are hard to reset, online services change, and an exploring agent can do things with real consequences. Hand-built environments are safer, but costly and hard to scale.

Computer-use agents make it harder. The same task might be done by clicking through a graphical interface, typing in a terminal, or calling an MCP service. MCP is a standard way for models to call outside tools. So choosing an interface, and knowing when to switch, is a skill in itself.

Tasks bring three more problems. They must be solvable, hard enough to teach something, and come with an automatic grader you can trust. And you can't assume the strongest model is always right.

### Prior work

Table 1 lines up existing synthesis methods. Most cover a single application, and almost all support only one interface: web only, or terminal only.

And every one of them builds its environments with a proprietary model, such as GPT-5, Claude Opus 4.6 or Gemini 3 Pro. The authors argue that hurts reproducibility and openness.

### Key idea

WildAWM's idea has two parts. First, instead of small isolated apps, build an entire personal computer for a fictional person, with websites, MCP services and files this person has used day after day, so the machine has a history.

Second, don't let one strong model judge the tasks. Have many different agents actually attempt them, treat the outcomes as evidence, and pass every task through five quality gates.

### Method

The overview has three parts: world synthesis on the left, task synthesis in the middle, and reinforcement learning on the right.

First, the world. The generator is a coding agent running the open-source GLM-5.2 model inside the OpenCode harness. It writes a persona from scratch: a job, contacts, favorite websites, ongoing projects, rewriting any persona too close to an existing one.

Then it plans this person's computer use day by day, and turns each plan into database records and files, each day building on the last. For realistic web pages, it adds fifteen mirrored websites, including Amazon, Booking, Coursera and GitHub.

Recurring events, like enrolling in a course, become reusable skills, and missing MCP services get written and added to a shared library. Each day's changes are a deterministic program that must replay identically twice, and a second agent checks the result against the plan.

The result: thirty-six personas, six hundred forty world states, and eighty-three reusable skills.

Next, the tasks. The generator explores the environment and proposes tasks that must use all three interfaces: the graphical interface, MCP, and the command line. It also writes an executable grader, a rubric and a hint. The grader is a checklist of yes-or-no checks, and the task counts as solved only if all of them pass.

Then come the test-takers. Three open-source models, GLM-5.2, DeepSeek-V4-Flash and MiniMax-M3, paired with six agent harnesses, make eighteen combinations. Each task gets six attempts, half with the hint.

Each attempt is judged three ways: by the grader, by an LLM judge, and by an agent judge that goes into the environment and inspects it.

Then the five gates. Gate one, judge consistency: all three judges must agree on all six attempts. Gate two, initial unsatisfaction: if the agent does nothing, the grader must say fail, which rules out tasks that are done from the start.

Gate three, solvability: at least one attempt succeeds, and it actually uses every interface the task declared. Gate four, difficulty: of the three attempts without hints, at most one success means hard, two means medium, and three means too easy, so it's sent back.

Gate five is the clever one, the counterfactual probe. Models tend to agree with whatever they're told, so the authors tell the agent judge the opposite verdict and ask it to look again. The task passes only if the judge holds its ground.

Failed tasks go back with the evidence, for up to two rounds of revision. This funnel shows two thousand five hundred fifty-one candidates in, three hundred forty-one accepted.

Here's an example. An audio-visual technician needs surge protectors for a sports bar's display wall. The agent confirms in a chat service that the client approved, then writes a script that combines an inventory sheet, a bill of materials and a PDF spec sheet to rank the candidates.

Then it adds one to the cart on the Amazon site. Two candidates have identical specs, so it has to look at the photos and pick the white one with a display. Finally it writes a note and messages the installer. All three unhinted attempts failed, so this one is labeled hard.

Training uses GRPO, a reinforcement learning method. Success earns full reward, failure earns a little partial credit for checks passed, and a formatting error costs a point, which gives a denser signal on long tasks.

Of the three hundred forty-one tasks, one hundred go to fine-tuning, two hundred eight to reinforcement learning, and thirty-three to evaluation. The base models, Qwen3.6-35B-A3B and Qwen3.5-4B, are warm-started on one hundred GLM-5.2 trajectories, then trained for up to sixty-four reinforcement learning steps.

### Results

First, how hard are the tasks? During synthesis, the average success rate was twenty-one point three percent without hints and forty-five point six with them. On the held-out set, GPT-6-Astra passes fifty-nine point eight five percent on one try, and even with eight tries never solves twenty-seven point three percent.

Can the grader be trusted? On one hundred sampled attempts, it agrees with the majority of several agent judges ninety-five percent of the time, a Cohen's kappa of zero point nine, where closer to one means the agreement isn't luck.

The main results are in Table 5. Training is entirely synthetic, and testing uses eight unseen benchmarks covering web agents, claw-style agents like OpenClaw, and computer-use agents. Every model the authors ran themselves uses the same OpenCode harness.

Both models improve on all eight. The 35B model's average goes from forty-two point eight four to fifty-three point four two, up ten point five eight. The 4B model goes from twenty-one point eight three to thirty point zero four, up eight point two one.

For the 35B model, the biggest jumps are ClawEval, up sixteen point six seven, OSWorld-Verified, up sixteen point zero six, and WebArena, up thirteen point seven two. The trained 35B model ties GLM-5.2 on WebArena at sixty-seven point two six, and beats GPT-5.5 on Online-Mind2Web, eighty point seven seven to seventy-two point two two.

On the paper's own held-out set, the 4B model goes from zero point three eight to twenty-seven point six five, and the 35B model from five point six eight to thirty-five point nine eight, close to GLM-5.2 at thirty-seven point eight eight.

### Comparison

Against other synthesis methods, the paper mainly compares environments. A WildAWM environment state has a median size of about fifty-two thousand tokens, versus about eleven hundred for CUA-Gym. And it still scores higher on the Vendi diversity metric at every sample size.

The gate ablations are in Table 7. With the 4B model, the authors drop one gate at a time, rebuild the fine-tuning data, and retrain. Removing any gate costs two point six to nine point three points on average. Without judge consistency, the average falls from thirty-one point eight six to twenty-two point five six, and the held-out score from twenty-one point five nine to three point seven nine. It isn't uniform, though: without the solvability gate, Online-Mind2Web actually goes up.

Table 8 ablates the reinforcement learning. Dropping the fine-tuning warm start hurts most, five point three one points on average. Dropping partial credit costs two point two five.

On behavior, the share of tool calls that switch interfaces rises from eighteen point four percent to twenty-five point five after training, close to the GLM-5.2 teacher. Fine-tuning on the model's own successes, or skipping the warm start, lowers it instead, to fifteen point eight and fifteen point seven.

The authors read this as: cross-interface habits come from the teacher, and reinforcement learning adds long-horizon skill. Between forty and one hundred twenty steps, fine-tuning alone solves nine point seven percentage points more tasks, and with reinforcement learning on top, seventeen.

My read is that the real contribution isn't the scores. It's a pipeline for building and validating tasks that needs no proprietary model and no single model's say-so, using many different agents' outcomes to show a task is solvable, hard, and fairly graded.

My read is also that something is missing. There's no head-to-head comparison with models trained on other synthesis methods' data under the same setup. The CUA-Gym and OpenForgeRL numbers in the table come from their own papers, with different harnesses, and the authors say they're for reference only.

### Limitations

There's no limitations section, so these come from the paper's own numbers. First, the hardest tasks barely move. Split by how many frontier models solve them, the hardest quarter, mostly OSWorld 2.0 and Agents' Last Exam, shows no clear gain for any training variant. On OSWorld 2.0, the 4B model goes from zero point eight seven to just one point zero three.

Second, the frontier is still well ahead. The 35B model averages fifty-three point four two, against seventy-three point one one for GPT-6-Astra and seventy point eight three for Claude Opus 5.

Third, the authors find the gains concentrate on tasks the GLM-5.2 teacher can already reach, and the warm start uses that teacher's trajectories.

Fourth, it's expensive and still small. One task takes the generator about seventy million input tokens on average, for a final set of three hundred forty-one tasks. Several benchmarks are also subsets: WebArena uses two hundred twenty-six of its eight hundred twelve tasks.

### Future work

In the appendix, the authors say that with enough compute, they'll test whether larger models can make better use of this data on the hardest tier. The code is already public on GitHub.

Their broader view is that this lets open-source models build worlds, learn inside them, and eventually become general computer-use agents.

My read is that the bottleneck in agent training is moving from models to environments, and whoever can build trustworthy practice grounds cheaply and safely at scale has the edge. This paper shows open-source models can already build tasks that GPT-6-Astra solves only about sixty percent of the time.

That's today's Daily Frontier Papers. The paper is linked in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
