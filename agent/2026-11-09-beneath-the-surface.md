# Beneath the Surface: What We Overlook When We Talk About Agent Evolution

**自我进化涨分了也在丢任务**

- 论文 Paper: [arXiv 2611.07624](https://arxiv.org/abs/2611.07624)
- 单位 Affiliation: 中国科学技术大学 · 北京通用人工智能研究院 · 上海交通大学 等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-09
- 视频 Videos: [YouTube 中文](https://youtu.be/Wu_-XshtgTA) · [YouTube English](https://youtu.be/-86OTsw-FCA) · [B站](https://www.bilibili.com/video/BV14WWb6XEyH)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇中国科学技术大学、北京通用人工智能研究院、上海交通大学和加州大学伯克利分校等单位合作的论文，题目叫，水面之下，我们谈论 Agent 进化时忽略了什么。

它问的问题很直接，Agent 自我进化之后，benchmark 分数涨了，这个 Agent 真的更好用了吗。

### 问题

先解释一个词，harness。一个 Agent，等于一个模型，加上包在模型外面的一层系统，比如提示词、技能、工具接口、调度流程，甚至源代码，这一层就叫 harness。

Harness 进化，就是模型权重不动，让 Agent 根据跑任务的经验，一轮一轮地改这层系统。论文把它看作递归自我改进，也就是 recursive self-improvement 的一种具体做法。

最近几个框架，把执行、诊断、修改、评估连成了闭环，都报告了 benchmark 上的提升。

但作者指出，总分是对所有任务、所有尝试取的平均。新解决的任务，可以把丢掉的任务掩盖掉，部署要花多少钱，在没参与进化的任务上表现怎么样，总分里也看不到。

所以他们问了三个问题。提升是不是可靠、划算，提升能不能带到新环境和新任务，以及进化流程里的哪些环节，决定了好的改动能不能产生、能不能留下来。

### 以前的方法

论文研究的是四个已有的进化方法。AHE 和 HyperAgents 偏工程实现，直接改 Agent 的实现，A-Evolve 和 GEPA 偏反思，把执行反馈提炼成可以复用的改动。

它们可以放进同一个流程，就是图 1 左边这个圈。选一个父代 Agent，跑任务收集轨迹，分析问题，提出修改，评估，最后由一道门决定新版本能不能留下。

起点版本叫 Seed，进化结束时选出来的版本叫 Champion。这些方法报告的，通常就是 Champion 在 benchmark 上比 Seed 高了多少分。

### 核心想法

这篇论文的核心想法是，只看一个总分，可能会选中脆弱或者昂贵的版本，所以要把总分拆开看。

先拆成两个指标。一个是可靠性，每个任务跑 8 遍，平均成功率是多少，论文记作 Avg@8。另一个是覆盖，8 遍里至少成功一次的任务占多少，记作 Pass@8。

再加上任务的进出，也就是相对 Seed，新覆盖了哪些任务，又丢了哪些，以及每个任务的部署成本，用美元算。

在这些指标之外，再做两类检验。一类看提升能不能迁移，一类把进化流程拆成环节，看问题出在哪一步。

### 方法

实验是这样搭的。四个进化方法用的模型都是 GPT-5.4，每个方法配两种 harness。一种是 MiniSWE，可以改源代码，另一种是 Codex，只能改外层，比如提示词、技能和配置。

任务有两类。通用任务用 Tau3-Banking，97 个银行业务任务，编程任务用 Terminal-Bench 2，89 个任务。两个都分成训练、验证和测试三部分，测试部分不参与进化。

四个方法，乘两种 harness，乘两类任务，一共 16 条进化线。每个 Champion 和对应的 Seed，都在完整任务集上各跑 8 遍来比较。

### 实验和效果

先看总分，在表 1。16 条线里，可靠性提高的有 11 条，覆盖提高的有 10 条，还有 4 条是两个指标一升一降。

涨得最多的，比如通用任务上 Codex 配 HyperAgents，可靠性从 30.3 提到 36.9，涨了 6.6 个百分点。

关键发现在图 2。作者把每个任务在 Seed 和 Champion 下 8 遍里成功几次，画成格子图。

结果是，16 个 Champion 全都有得有失。新覆盖了一些任务，同时丢掉了一些原来能做的任务。

举个例子。通用任务上 MiniSWE 配 HyperAgents，可靠性涨了 5.2 个百分点，但新覆盖 13 个任务的同时，丢了 10 个原来覆盖的任务。

所以作者说，覆盖是流动的。总分涨了，不代表原来会做的都还会做。

他们还翻了执行记录，看任务为什么由败转成，或者由成转败。常见的原因是，状态有没有保住，参数和日期是不是精确，任务有没有真正做完。

比如一个编程任务，Seed 在检查数据库日志的时候，把原始日志弄丢了，8 遍全失败。进化后的版本先备份再修复，8 遍成功 7 遍。

反过来也有。一个银行任务里，进化后的 Agent 开完支票账户就结束了，没有接着办后面的储蓄和注资请求，成功次数从 8 遍里 5 遍掉到 0 遍。

再看成本。可靠性提高的 11 个 Champion 里，有 8 个每次尝试都比 Seed 更贵，更便宜的 3 个，都是 MiniSWE 在编程任务上。

但贵不一定不划算。图 3 里，MiniSWE 配 AHE 在通用任务上，跑 4 遍就达到百分之 56.7 的覆盖，超过 Seed 跑 8 遍的百分之 54.6，部署成本还更低。

接下来是迁移，在表 3。同一个 benchmark 里，测试集上的可靠性有 10 条线提高，其中工程派的 8 条里有 7 条。也就是说，提升经常能延伸到没参与进化的任务。

换一个 benchmark 就不一样了。通用任务迁移到完整的 Tau2-bench，8 个 Champion 里有 7 个覆盖下降，编程任务迁移到 DeepSWE，8 个里有 7 个覆盖上升。

而且可靠性和覆盖可能反着走。能改源代码的 Champion，迁移到通用任务时可靠性全部提高，迁移到编程任务时可靠性全部下降。

最后一组实验，把进化流程拆成四个环节来看。第一个是选父代。作者称，在他们的四个候选池里，分数最高的 Agent，没有一次带来最大的平均提升。

表 4 更直观。原方法的选择规则，在通用任务上选中最佳父代的比例，是百分之 17.5 和百分之 8.8，比随机选的百分之 25 还低，到了编程任务上，又都超过百分之 55。

第二是分析环节。作者把 Champion 里的改动拆成一个个组件，和泛泛的提醒比较，比如提醒检查工具、核对结果。16 个具体组件里，有 15 个在可靠性上胜过对应的泛泛提醒。

第三是修改环节，好的改动放在一起，效果会变。8 个 Champion 里有 6 个，在可靠性上比单独任何一个组件都强，但也有两个通用任务的例子，只用一部分改动反而更好。

第四是准入环节。作者试了更严格的规则，要求新版本平均成功率更高，而且相对现任版本的平均退步不超过 0.05。结果能留住最佳候选的池子，从 15 个降到 6 个，更严不一定更好。

### 和以前方法的对比

作者没有停在诊断上。他们做了一个叫 EvoAudit 的流程，让 Agent 去审计已经存档的进化记录，跑诊断实验，给出对选择、分析、修改和准入四个环节的修订。

把这四个修订装进 A-Evolve，就得到 AGE，也就是审计引导的进化。

举两个改动。选父代的时候，如果几个候选分数打平，就挑相对 Seed 退步最少的那个。准入的时候，新版本分数和父代打平，但验证成本更低，也可以留下。

结果在表 10。AGE 在四种设置下，可靠性和覆盖都超过了原来的 A-Evolve，丢掉的原有任务，三种设置更少，一种持平。

和全部五个方法放在一起，AGE 在四种设置下可靠性都是最高的，比如通用任务配 Codex，从 Seed 的 30.3 提到 38.1。论文还请了八位专家评审这四个修订，平均有 6.75 位支持把修订送去验证。

我的判断是，这组对比要打个折扣。AGE 的修订，来自对同样两个 benchmark 上进化记录的诊断，再回到同样的 benchmark 上评估，用的也是单个随机种子，论文没有给出在新 benchmark 上的检验。

我的判断是，这篇最有用的，是那张得失表。它提醒做 Agent 的人，看自我进化的结果，别只看总分，要看丢了什么，贵了多少，换个场景还在不在。

### 局限

论文没有单独的局限章节，下面几点来自正文和附录。

第一，每种方法、harness 和任务的组合，只有一条进化线。Champion 会重复评估 8 遍，但进化过程本身没有重复跑。

第二，作者做了稳定性检查，把 8 遍评估分成前 4 遍和后 4 遍。在通用任务上，有些总体趋势在两半里方向不同，说明结果对执行中的随机波动敏感。

第三，范围有限。只用了 GPT-5.4 一个模型，两个 benchmark，任务数都不到 100 个，在覆盖这个指标上，一个任务的变化，就差不多是一个百分点。

第四，作者在伦理声明里也说，benchmark 上的提升和专家的认可，都不能证明可以安全部署。

### 未来方向

作者给的方向是，针对每个决策设计反馈。评估选父代，要看它之后带来的提升，分析和修改，要基于执行轨迹和依赖关系，准入要同时看留下的质量和保住的备选，再用部署成本和外部评测，判断提升能用在多大范围。

论文给出了代码仓库，名字叫 RSIHub。

我的判断是，Agent 自我进化接下来要回答的，不是能不能涨分，而是能不能只涨不丢。谁能把退步控制住，谁的方法才更适合放进生产环境。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from a group including the University of Science and Technology of China, the Beijing Institute for General Artificial Intelligence, Shanghai Jiao Tong University and UC Berkeley. It's called Beneath the Surface: What We Overlook When We Talk About Agent Evolution.

Its question is blunt. When a self-evolving agent posts a higher benchmark score, did it actually get more useful?

### Problem

First, one term: the harness. An agent is a model plus the system wrapped around it: prompts, skills, tool interfaces, orchestration, sometimes source code. That wrapper is the harness.

Harness evolution keeps the model weights frozen and lets the agent rewrite that wrapper, round after round, based on its own experience running tasks. The paper treats it as a concrete form of recursive self-improvement.

Recent frameworks close the loop between running, diagnosing, editing and evaluating, and they all report gains on standard benchmarks.

The catch, the authors argue, is that a benchmark score averages over every task and every attempt. Newly solved tasks can hide the ones that were lost. And the score says nothing about deployment cost, or about tasks that never guided the evolution.

So they ask three questions. Are the gains reliable and affordable? Do they survive in new environments and tasks? And which steps of the evolution loop decide whether good changes get made, and kept?

### Prior work

The study covers four existing methods. AHE and HyperAgents are implementation-centric: they engineer the agent directly. A-Evolve and GEPA are reflection-centric: they distill execution feedback into reusable changes.

All four fit one loop, shown on the left of Figure 1. Select a parent agent, roll it out on tasks, analyze what happened, mutate it into a candidate, evaluate, and let a gate decide whether the new version stays.

The starting version is called the Seed, and the version chosen at the end is the Champion. What these methods usually report is how far the Champion beats the Seed on a benchmark.

### Key idea

The key idea is that relying on one score can favor brittle or expensive agents, so the score has to be taken apart.

Start with two metrics. Reliability: run each task eight times and take the average success rate, written Avg at eight. Coverage: the share of tasks solved at least once in those eight runs, written Pass at eight.

Add task turnover, meaning which tasks were newly covered relative to the Seed and which were lost, and the per-task deployment cost, in dollars.

On top of those metrics come two kinds of tests: one asks whether gains transfer, the other breaks the evolution loop into steps to find where things go wrong.

### Method

Here's the setup. All four methods run on GPT-5.4, each with two harnesses. MiniSWE is source-editable, so evolution can change its code. Codex is outer-layer only: prompts, skills and configuration.

There are two task types. General tasks come from Tau-three Banking, ninety-seven banking tasks. Coding tasks come from Terminal-Bench 2, eighty-nine tasks. Both are split into train, validation and test, and the test split is kept out of evolution.

Four methods, times two harnesses, times two task types, gives sixteen evolution lineages. Each Champion and its Seed are run eight times over the full task set and compared.

### Results

Start with the headline numbers in Table 1. Of the sixteen lineages, eleven improve reliability and ten improve coverage, and four move the two metrics in opposite directions.

The biggest reliability jump: on general tasks, HyperAgents with Codex lifts reliability from thirty point three to thirty-six point nine, up six point six points.

The key finding is in Figure 2. For every task, the authors plot how many of eight runs succeeded under the Seed and under the Champion.

All sixteen Champions both gain and lose. Each one covers some new tasks while dropping some tasks the Seed could already solve.

One example. On general tasks, HyperAgents with MiniSWE raises reliability by five point two points, but while it gains thirteen tasks, it loses ten that the Seed covered.

The authors' phrase for this is that coverage is a flow. A higher score doesn't mean the agent can still do what it used to.

They also read the traces to see why tasks flipped. The recurring reasons: whether state was preserved, whether arguments and dates were exactly right, and whether the task was actually finished.

Take one coding task. The Seed lost the original database log while inspecting it, and failed all eight runs. The evolved version saved a copy before repairing, and succeeded seven times out of eight.

It cuts the other way too. On one banking task, the evolved agent stopped after opening a checking account, without handling the savings and funding requests, and dropped from five successes in eight to zero.

Then cost. Of the eleven Champions with better reliability, eight cost more per attempt than their Seed. The three cheaper ones are all MiniSWE on coding tasks.

But more expensive isn't always worse value. In Figure 3, MiniSWE with AHE on general tasks reaches fifty-six point seven percent coverage in four runs, beating the Seed's fifty-four point six in eight runs, at lower deployment cost.

Next, transfer, in Table 3. Within the same benchmark, test-split reliability improves in ten of sixteen lineages, including seven of the eight implementation-centric ones. So gains often do reach tasks that didn't guide evolution.

Switch benchmarks and the picture changes. Moving general tasks to the full Tau-two bench, coverage falls for seven of eight Champions. Moving coding tasks to DeepSWE, coverage rises for seven of eight.

And reliability can move against coverage. Every source-editable Champion gains reliability when transferred on general tasks, and every one loses reliability when transferred on coding tasks.

The last set of experiments takes the loop apart, step by step. First, selecting the parent. The authors report that in none of their four pools did the highest-scoring agent produce the largest mean improvement.

Table 4 makes it concrete. On general tasks, the methods' own selection rules pick the best parent seventeen point five and eight point eight percent of the time, below the twenty-five percent you'd get at random. On coding tasks, both are above fifty-five percent.

Second, analysis. The authors split the Champions' changes into components and compared them with generic reminders, like "check your tools" or "verify your results". Fifteen of sixteen specific components beat their matching reminder on reliability.

Third, mutation, where good changes can interact. On reliability, six of eight Champions beat either of their components on its own, but in two general-task cases a subset of the changes did better.

Fourth, the gate. The authors tried a stricter rule: a new version needs higher mean success, and its mean regression against the incumbent can't exceed zero point zero five. The number of pools that kept the best candidate fell from fifteen to six. Stricter isn't automatically better.

### Comparison

The authors don't stop at diagnosis. They built a workflow called EvoAudit, where an agent audits archived evolution runs, runs diagnostic experiments, and proposes revisions to the select, analyze, mutate and gate steps.

Install those four revisions in A-Evolve, and you get AGE, Audit-Guided Evolution.

Two of the changes, for example. When candidate parents tie on score, pick the one with the fewest regressions relative to the Seed. And at the gate, a new version that ties its parent's score can still be admitted if its validation run costs less.

The results are in Table 10. In all four settings, AGE beats the original A-Evolve on both reliability and coverage, and it loses fewer of the Seed's tasks in three settings, tying in the fourth.

Against all five methods, AGE has the highest reliability in all four settings; with Codex on general tasks, for instance, it goes from the Seed's thirty point three to thirty-eight point one. Eight experts also reviewed the four revisions, and on average six point seven five of them endorsed sending each one to validation.

My read is that this comparison deserves a discount. AGE's revisions came from diagnosing evolution runs on these same two benchmarks, they were evaluated on those same benchmarks with a single search seed, and the paper doesn't test them on a new benchmark.

My read is that the most useful part of this paper is the gain-and-loss chart. If you build agents, judge a self-evolution result by what it dropped, what it costs, and whether it holds up elsewhere, not just by the score.

### Limitations

The paper has no separate limitations section, so these points come from the main text and appendix.

First, each combination of method, harness and task type is a single evolution lineage. Champions are evaluated eight times, but the evolution itself isn't repeated.

Second, in a stability check, the authors compared the first four runs with the last four. On general tasks, some overall trends point in different directions in the two halves, which shows sensitivity to run-to-run variation.

Third, the scope is narrow. One model, GPT-5.4, two benchmarks, each with fewer than a hundred tasks. On coverage, a single task is worth roughly one percentage point.

Fourth, the authors' own ethics statement says that neither benchmark gains nor expert endorsement establish that an agent is safe to deploy.

### Future work

The direction they propose is feedback matched to each decision. Judge parent selection by the gains that follow, ground analysis and mutation in traces and dependencies, judge the gate by both what it keeps and the alternatives it preserves, and use deployment cost and external benchmarks to decide how broadly a gain applies.

The paper links its code repository, called RSIHub.

My read is that the next question for self-evolving agents isn't whether they can raise a score. It's whether they can improve without losing ground. Whoever controls regressions will have the method that's ready for production.

That's today's Daily Frontier Papers. The paper is linked in the description.

