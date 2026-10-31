# WorkFactory: Turning Professional Intents into Executable Work for Scalable Agent Training

**先理清信息再造训练任务**

- 论文 Paper: [arXiv 2610.37161](https://arxiv.org/abs/2610.37161)
- 单位 Affiliation: IQuest Research、人大高瓴、浙大、上交、北大等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-31
- 视频 Videos: [YouTube 中文](https://youtu.be/gGO4eqaVmW4) · [YouTube English](https://youtu.be/xhCzIX_k-c4) · [B站](https://www.bilibili.com/video/BV1BAxC6nEUj)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 IQuest Research 联合人大高瓴人工智能学院、浙大、上交、北大等单位的论文，叫 WorkFactory，意思是把职业上的工作意图，变成真正能执行的任务，用来规模化训练 Agent。

它没有提新的训练算法，研究的是怎么批量造出做得下去、又值得做的职场任务，让强模型示范，再教给小模型。

### 问题

先说背景。论文里叫 working agent，就是在电脑里替人干专业工作的 Agent，要跑流程、调工具，最后交出表格、报告这类成品。

训练这类 Agent，常见做法是监督微调，让强的老师模型把任务做一遍，学生照着老师的操作记录学，这种记录叫轨迹。

问题出在任务本身。论文举了个例子，一个选供应商的任务，需求预测和几家报价分在不同文件里，却没写清楚其中一家的价格含不含运费。

少了这个条件，成本就没法算，也没法比。可如果直接把算好的成本给出来，老师该干的活又被拿掉了一大块。

所以造任务要守一条边界，必要的前提都得给到，中间结果和结论得留给老师自己推。守不住，老师要么自己编假设，要么活干不完，这样的轨迹拿来训练就打了折扣。

### 以前的方法

写代码的 Agent 比较好办，有现成的代码仓库，还有测试能自动判对错，比如 SWE-Gym、SWE-smith。

到了职场任务，近期的框架会检索、复用、改写或者合成材料，再收集老师轨迹，比如 NexForge 和 Synthetic Computers。

同期还有 GraphForge 和 WorkGenesis，一个在真实文件上建证据图，一个按评分细则找失败原因再修改。作者说，WorkFactory 的不同在于，从要交的成品倒着规划，让模型实际做一遍来找问题，而且修改不能改变原本要做的工作。

### 核心想法

核心想法是，先想清楚信息，再去造文件。从要交的东西出发，倒推它依赖哪些计算和判断，每一步需要哪些信息，这些信息该放在哪里。

环境造好后，先派一个探针模型做一遍，看卡在哪里，有问题就修，修完再试。只有过关的任务，才交给独立的老师模型，它的轨迹才进训练集。

这张总览图用的就是刚才那个纸箱采购的例子，上半部分是规划，下半部分是修复。

### 方法

第一步是攒意图，也就是一件有意义、但具体材料还没定的工作。来源有三个，职业分类体系，公开的技能文档，和网上收集的真实职业文档，再由大模型审一遍，看像不像真实的工作。

第二步是信息支撑图。规划器从成品往回推，每个信息节点都标明，它是直接提供的，要从外部查的，要解题者自己推出来的，还是只在构造时内部使用的。

每条信息还要绑定位置。比如需求和报价放在参考 PDF 里，折现规则写在用户请求里，算出来的净现值和推荐结论，对应一份还不存在的成本对比表，留给解题者做。净现值，就是把未来的花费折算成今天的钱。

然后由一个环境构造 Agent 按计划去复用、改写或者合成文件。之后用程序和大模型检查，不过关最多重试三次，还不过就丢掉。

第三步是执行式修复。探针只看得到用户请求和参考文件，看不到信息图和计划，它边做边报告环境问题，还要分清哪些是环境的错，哪些是自己做错，哪些本来就可以自由选择。

只有挡住任务完成的问题才触发修复。比如供应商 C 没写运费，修复模型就在报价里补一条，每箱额外收 0.15 美元运费，净现值能比了，但比较本身还留给解题者。

修改要过构造检查，还要过大模型的工作保留审查，看核心目标、关键操作和交付要求有没有被改掉。过了就换一个全新的探针再做，最多修三轮，还卡住就丢弃。

最后是收集轨迹。老师是 GLM-5.3，在干净的沙箱里独立做题，看到的信息和探针一样。轨迹再过一道轻量过滤，去掉重复推理、重复调用工具、交付物缺失或太单薄的。

最终语料有 112686 个任务，覆盖 612 个细分职业，平均每个任务带 4.15 个参考文件，老师轨迹平均大约 57 轮、63 次工具调用。

### 实验和效果

学生是三个 Qwen 模型，Qwen3.5-4B、Qwen3.6-27B 和 Qwen3.6-35B-A3B，用同一份数据、同一套配置微调，得到 WorkPilot 系列。主要对照，就是各自的原始模型。

评测用四个职场基准，GDPval、JobBench、Workspace-Bench 和 OmegaUse-OfficeVal。GDPval 按 GDPval-AA 协议算 Elo 分，有官方分的用官方分，没有的，作者用 Gemini 3.1 Pro Preview 当裁判在本地估算，再校准到同一尺度。

结果在表 2。三个尺寸，四个基准，全部比原始模型高。

以 35B-A3B 为例，GDPval-AA 从 879 涨到 1565，JobBench 从 28.91 涨到 62.35，OmegaUse-OfficeVal 从 7.14 涨到 21.18。

最好的是 WorkPilot-27B，GDPval-AA 本地估算 1612 分，比 DeepSeek-V4.1-Flash 高 12 分，比老师 GLM-5.3 低 34 分。要注意，1612 是作者本地估算的，不是官方榜单分数。

WorkPilot-4B 的 GDPval-AA 从 661 涨到 1375，在 GDPval 和 OmegaUse-OfficeVal 上，超过了约 300 亿参数级别的 Apodex-1.1-mini。

再看数据规模，图 5 左边。35B-A3B 的前 5000 个任务就贡献了 447 分，从 7.5 万加到 11 万，只多了 13 分。到 3 万个任务时，35B-A3B 和 27B 已经拿到了各自全部提升的百分之八十八和百分之九十四。

图 5 中间有个有意思的现象。35B-A3B 平均每题的轮数，原始模型是 24.87，训练 2000 个任务后涨到 162.88，到 11 万个任务又降回 85.90，而分数一直在涨。也就是说，先出现更长的尝试，数据多了，执行反而变短。

### 和以前方法的对比

先说清楚，论文没有拿 GraphForge、WorkGenesis 这些同类框架直接比，最接近的是一组控制实验，各用 5000 个任务，比较直接按文件规划、加上信息支撑图、再加上执行式修复，这三种造法。

结果在表 3。加上信息支撑图，八类缺陷有六类下降，最明显的是推导断裂，就是给的条件推不出要的结果，从百分之二十四点八六降到百分之六点八六。

但泄题或捷径、自相矛盾这两类变多了，分别多了 1.44 和 1.18 个百分点。作者猜测，依赖写得太明确，构造模型可能把该推导的信息直接写进了请求，约束太密，也更难前后一致。

再加上执行式修复，八类缺陷全部下降。跨文件不一致从百分之二十五点七四降到百分之四点八四。

更关键的是下游效果。同一个模型分别在三种数据上训练，GDPval-AA 从 1245 到 1293，再到 1326，累计多了 81 分。

换老师也做了实验。同样 3 万个任务，GLM-5.3 当老师，学生到 1483 分，换成 DeepSeek-V4-Flash-0731，学生只有 1248 分，都比原始的 879 分高。附录显示，GLM-5.3 做题用的轮数和工具调用都明显更多。

成本方面，三个 WorkPilot 都在图 6 的性价比前沿上，4B 大约 0.012 美元一题，27B 大约 0.396 美元。这些价格是作者按公开报价映射折算的估计，不是实际账单。

我的判断是，这篇最有价值的不是分数，而是那张缺陷表。它把任务质量拆成可以统计的八类问题，并且显示缺陷少了，学生也更好。

我的判断是，短板也在这里。主结果只和原始模型比，没有在同样的学生、同样的数据量下，和别的造数据方法比。控制实验也没给置信区间，修复带来的那 33 分有多稳，还不好说。

### 局限

作者写了两条局限。第一，老师 GLM-5.3 只能输入文字，训练和评测都是纯文本，Qwen 自带的视觉能力没用上，也没研究从图像证据里学习。

第二，训练只用了监督微调，也就是模仿老师。

还有，轨迹只过了轻量过滤，并不判断交付物对不对，只有轨迹最长的那百分之五的任务，才额外用两两比较来挑选。

造任务也不便宜，平均每个尝试构造的任务要用掉约 1166 万个 token。另外，本地估算的 GDPval 分数，作者检验过，和官方分数相比，验证集上平均差 24 分，测试集上差 19 分。

### 未来方向

作者提了两个方向，一是扩展到多模态的职场任务，二是用这些任务和执行反馈做强化学习。

论文说会公开部分任务数据和模型权重，地址在 GitHub 上。

我的判断是，这套思路对做 Agent 数据的人很实用，先画信息依赖，再让模型真做一遍来查漏，换了老师，学生也都比原始模型强，可以搬到别的领域。

我的判断是，下一步最值得看的，是接上强化学习以后任务够不够可验证。现在的修复只保证任务做得下去，并不提供标准答案来打分。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from IQuest Research, with Renmin University's Gaoling School of AI, Zhejiang, Shanghai Jiao Tong, Peking University and others. It's called WorkFactory: turning professional work intents into tasks an agent can actually execute, at training scale.

There's no new training algorithm here. The question is how to mass-produce workplace tasks that are doable and worth doing, so a strong model can demonstrate them for a small one.

### Problem

Some context. The paper calls them working agents: agents that do professional work on a computer, running workflows, calling tools, and handing back spreadsheets and reports.

The usual way to train one is supervised fine-tuning. A strong teacher does the task, and the student imitates the teacher's record of actions, called a trajectory.

The trouble is the tasks. The paper's example is a supplier-selection task where the demand forecast and vendor quotes sit in separate files, but one quote never says whether freight is included.

Without that, you can't compute or compare costs. But hand over the computed costs, and you've removed a big chunk of the teacher's work.

So building a task means holding a line: every necessary premise is available, while the intermediate and final results are left for the teacher. Get it wrong, and the teacher invents assumptions or leaves the job unfinished, which makes weaker training data.

### Prior work

Coding agents are the easy case: real repositories give context, and tests check the answer automatically, as in SWE-Gym and SWE-smith.

For professional work, recent frameworks retrieve, reuse, adapt or synthesize materials and then collect teacher trajectories, like NexForge and Synthetic Computers.

Concurrent work includes GraphForge, which builds evidence graphs over real files, and WorkGenesis, which traces failures against a rubric and revises. The authors say WorkFactory differs by planning backward from the deliverable, finding problems by actually attempting the task, and forbidding repairs that change the intended work.

### Key idea

The key idea: work out the information before you build the files. Start from the deliverable, trace back the calculations and judgments it depends on, what information each step needs, and where it should live.

Once the environment exists, a probe model attempts it to see where it gets stuck. Broken things get fixed and retried. Only tasks that pass go to an independent teacher, and only its trajectory enters the training set.

This overview figure uses that same shipping-carton example. Planning on top, repair below.

### Method

Step one is collecting intents, meaningful pieces of work before specific materials are fixed. They come from an occupational classification, public skill documentation, and real professional documents from the web, and an LLM reviews whether each looks like real workplace activity.

Step two is the information support graph. The planner works backward from the deliverable, labeling every information node as supplied, looked up externally, derived by the solver, or internal to construction.

Each piece is also bound to a location. Demand and quotes go in a reference PDF, the discounting rules in the user request, and the net present values and recommendation belong to a cost workbook that doesn't exist yet, because producing it is the solver's job. Net present value just means future spending converted into today's money.

An environment-building agent then reuses, adapts or synthesizes the files to plan. Programmatic and LLM checks follow, with up to three retries before a failing task is dropped.

Step three is execution-based repair. The probe sees only the request and reference files, never the graph or plan. It reports environment problems as it works, separating the environment's faults from its own mistakes and from legitimately open choices.

Only blocking problems trigger repair. If Supplier C never mentions freight, the repair model adds a line charging an extra fifteen cents per carton, which makes the NPVs comparable but leaves the comparison to the solver.

Each revision must pass the construction checks and an LLM review that the objective, essential steps and deliverable requirements are intact. Then a brand-new probe tries again, for up to three rounds, and anything still blocked is discarded.

The last step is collecting trajectories. The teacher, GLM-5.3, solves each task in a clean sandbox with the same view as the probe. A lightweight filter drops trajectories with repeated reasoning, repeated tool calls, or missing or thin deliverables.

The final corpus has about one hundred twelve thousand seven hundred tasks across six hundred twelve occupations, with about four reference files per task, and teacher trajectories averaging roughly fifty-seven turns and sixty-three tool calls.

### Results

The students are three Qwen models, Qwen three point five four B, Qwen three point six twenty-seven B, and Qwen three point six thirty-five B A three B, all fine-tuned the same way to make the WorkPilot family. The main control is each model's base version.

There are four workplace benchmarks: GDPval, JobBench, Workspace-Bench and OmegaUse-OfficeVal. GDPval is scored in Elo under the GDPval-AA protocol, with official ratings where they exist. Otherwise the authors estimate locally, with Gemini three point one Pro Preview as judge, and calibrate to the same scale.

That's Table two. Three sizes, four benchmarks, every number above the base model.

Take the thirty-five B model. GDPval-AA goes from eight seventy-nine to fifteen sixty-five, JobBench from about twenty-nine to sixty-two, and OmegaUse-OfficeVal from about seven to twenty-one.

The best is WorkPilot twenty-seven B, at a locally estimated sixteen twelve. That's twelve points above DeepSeek V four point one Flash and thirty-four below the teacher, GLM-5.3. But sixteen twelve is the authors' own estimate, not an official leaderboard score.

WorkPilot four B climbs from six sixty-one to thirteen seventy-five, and beats Apodex one point one mini, from the roughly thirty-billion class, on GDPval and OmegaUse-OfficeVal.

Now data scale, the left of Figure five. For the thirty-five B model, the first five thousand tasks are worth four hundred forty-seven points, while going from seventy-five thousand to one hundred ten thousand adds just thirteen. By thirty thousand tasks, the thirty-five B and twenty-seven B models have eighty-eight and ninety-four percent of their full gains.

The middle panel is curious. Average turns per task go from about twenty-five for the base model to about one hundred sixty-three after two thousand tasks, then back to about eighty-six at one hundred ten thousand, while the score keeps rising. Longer attempts come first; with more data, runs get shorter.

### Comparison

To be clear, there's no head-to-head with GraphForge, WorkGenesis or similar frameworks. The closest is a controlled study, five thousand tasks each, comparing plain file-level planning, adding the information graph, and then adding repair.

That's Table three. With the graph, six of eight defect types fall. The biggest drop is broken derivations, where the premises can't produce the requested result, from about twenty-five percent to under seven.

But leakage or shortcuts, and self-inconsistency, rise by one point four and one point two percentage points. The authors guess that explicit dependencies can lead the builder to write solver-derived information into the request, and denser constraints make consistency harder.

Adding repair brings all eight down. Cross-file inconsistency falls from about twenty-six percent to under five.

What matters more is downstream. Training the same model on each version gives GDPval-AA scores of twelve forty-five, twelve ninety-three, then thirteen twenty-six, eighty-one points in total.

They also swapped teachers. On the same thirty thousand tasks, a GLM-5.3 teacher yields a student at fourteen eighty-three, a DeepSeek V four Flash teacher only twelve forty-eight, though both beat the base model's eight seventy-nine. The appendix shows GLM-5.3 using clearly more turns and tool calls.

On cost, all three WorkPilots sit on the cost-performance frontier in Figure six, from about one cent a task for four B to about forty cents for twenty-seven B. Those are the authors' estimates, mapped from public provider prices, not actual bills.

My read is that the most valuable thing here isn't the scores but the defect table. It turns task quality into eight countable problems, and shows fewer defects come with better students.

My read is that the weakness sits in the same place. The main results only compare against base models, not against other data-building methods at the same student and data size. And the controlled study has no confidence intervals, so it's hard to say how solid the thirty-three points from repair are.

### Limitations

The authors list two limitations. First, the teacher, GLM-5.3, only takes text, so everything is text-only, the Qwen models' vision goes unused, and learning from visual evidence isn't studied.

Second, training is supervised fine-tuning only, pure imitation of the teacher.

Also, the lightweight filter doesn't judge whether a deliverable is correct; only the five percent of tasks with the longest trajectories get an extra pairwise comparison.

Building tasks isn't cheap, at about eleven point seven million tokens per attempted task. And the authors checked their local GDPval estimates against official ratings: the average error was about twenty-four points on validation and nineteen on test.

### Future work

The authors name two directions: multimodal workplace tasks, and reinforcement learning using these tasks and their execution feedback.

The paper says a subset of the task data and the model weights will be released on GitHub.

My read is that this recipe is practical for anyone building agent data: map the information dependencies first, then have a model actually attempt the task to find the gaps. With either teacher the students beat their base model, so it should carry over to other domains.

My read is that the thing to watch is whether these tasks are verifiable enough once reinforcement learning is added. Today's repair makes sure a task can be done, but supplies no reference answer to score against.

That's today's Daily Frontier Papers. The paper is linked in the description.

