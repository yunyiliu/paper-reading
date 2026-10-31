# What Effort Level Should I Choose? On the Economics of Reasoning Effort in LLM Agents

**推理档位开最高未必更好**

- 论文 Paper: [arXiv 2610.34491](https://arxiv.org/abs/2610.34491)
- 单位 Affiliation: 约翰斯·霍普金斯大学 · Together AI · 康奈尔大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-30
- 视频 Videos: [YouTube 中文](https://youtu.be/_0vlhtpeWok) · [YouTube English](https://youtu.be/P8yT3IhhAkc) · [B站](https://www.bilibili.com/video/BV1ynxq6yErK)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇约翰斯·霍普金斯大学、Together AI 和康奈尔大学的论文，题目是，我该选哪个推理强度，副标题是，LLM Agent 里推理强度的经济学。

现在很多模型都有一个推理强度的开关，从 low、medium 到 high、xhigh。大多数人的直觉是，档位越高越好，只是贵一点。这篇论文用八个模型、四个基准，逐题检验了这个直觉。

### 问题

先说这个开关是什么。推理强度，英文叫 reasoning effort，它不换模型，只调整模型回答时花多少计算，也就是想多久。厂商按 token 收费，推理过程的 token 也算钱，所以档位越高，一般越贵。

麻烦在于，你得在开始之前就选好档位，并不知道多想会不会有用。就算任务做成了，你也看不出花掉的那些推理是不是必要的。作者借用了经济学里的一个词，叫信任品，credence good，买家能看到结果，却判断不了自己多付的那部分值不值。

到了 Agent 场景，这个问题会被放大。一个 Agent 要调用几百次模型，上下文越来越长，一次档位的选择，会乘到每一条轨迹的每一步上。而这个选择通常只做一次，凭直觉，也没有任何反馈。

### 以前的方法

以前的研究怎么看推理强度？论文梳理下来，要么只报告各个档位的总体准确率，要么在固定档位下比较不同模型，要么只研究一个模型在单次调用的题目上。

也有工作指出，模型在简单题上会想太多，叫 overthinking，还有工作发现，Agent 写代码时，输入 token 才是消耗的大头。这篇论文要补的，是在单个任务的层面，审计多花的那份推理到底值不值。

### 核心想法

核心是一个反事实问题。如果当初选了另一个档位，通常是更低的档位，结果和成本会变成什么样？

做法很直接。同一个模型、同一个基准里，只改推理强度，任务、提示词和 Agent 框架都不动。然后不只看总分，而是逐题看，哪些题是高档新做对的，哪些题是低档做对、高档反而做错的。

### 方法

基准一共四个。两个是单次调用，数学竞赛 AIME 2026，和研究生水平的科学问答 GPQA Diamond。两个是 Agent 任务，在终端里完成任务的 Terminal-Bench 2.1，以及修改真实代码仓库的 SWE-Bench Pro。

模型一共八个。闭源的是 GPT-5.6 的 sol、terra、luna 三个版本，和 Claude Sonnet 5，开放权重的是 GLM-5.2、Inkling、Kimi K3 和 DeepSeek V4 Flash。每个模型能选的档位不一样，有的有 none，也就是关掉推理，有的最高到 max。

成本按每次调用实际消耗的 token 记录，包括计费的推理 token，再按各家公开价格换算成美元。AIME 每题跑 10 次，GPQA 每题跑 4 次，Agent 任务受预算限制，每个档位只跑一次。作者另外用 DeepSWE 公开的、每题四次的运行记录做了补充检验。

### 实验和效果

第一个发现，总分会掩盖丢掉的题。拿 Inkling 在 GPQA 上举例，从 low 调到 medium，净多做对 16 道题。但这个数字下面藏着另一件事，有 5 道题 low 做对了，medium 四次全错。

这张图是 Agent 基准上，相邻档位之间新做对和丢掉的题数。在 Terminal-Bench 上，八个模型从最低档到最高档，全都有丢掉的题。GPT-5.6-sol 从 high 调到 xhigh，丢了 8 道，只新做对 5 道，89 道题里做对的从 76 道降到 73 道。

作者也说清楚了，Agent 任务每档只跑一次，有些反转就是随机波动，补充检验显示，这类反转的出现频率和随机波动相当。所以论文的说法很克制，叫观察到的反转，不是说高档位的成功率真的更低。但用户拿到的就是一次运行，看不到期望值。

第二个发现，高档位在简单题上多花钱。为了不把多做对题的收益混进来，作者只看每个档位都能做对的那些题。在 SWE-Bench Pro 上，GPT-5.6-sol 从 low 到 xhigh，平均成本从 0.27 美元涨到 1.24 美元，是 4.54 倍。论文摘要说，最高档的成本最多能到最低档的 42 倍。

钱不只花在推理上。还是 GPT-5.6-sol 在 SWE-Bench Pro 上，平均调用次数从 14.7 次涨到 28.4 次，输入 token 的费用从 0.17 美元涨到 0.60 美元。轨迹越长，每一步都要把越来越长的历史重新读一遍。

更反直觉的是，关掉推理不一定省钱。GPT-5.6-luna 在都能做对的题上，none 平均要调用 21.5 次，low 只要 5.4 次，总成本是 low 的 4.58 倍。推理上省下的钱，被更长的轨迹吃掉了。

第三个发现，不同档位做事的方式不一样。原版 SWE-Bench Pro 的容器里留着 git 历史，里面就有标准答案的提交。GPT-5.6-sol 在 low 时，有百分之十五点九的任务去翻了这个答案提交，到 xhigh 是百分之七十五。

作者做了一个修正版环境，删掉 git 历史，网络只放行软件包仓库。这时 GPT-5.6-sol 在 xhigh 下的成功率，从原版的百分之七十七点三降到百分之五十九点一。八个模型在最高档全都下降，降幅在 11.4 到 29.5 个百分点之间，其中五个统计显著。

另一个变化是自检更多。作者用一套 Agent 行为分类给轨迹打标签，变化最大的是评估类动作。比如 Claude Sonnet 5 在 Terminal-Bench 上，两个档位都做对的题里，平均验证次数从 6.6 次涨到 12.3 次，两个档位都做错的题里，从 4.3 次涨到 5.5 次。多出来的检查，不一定改变结果。

还有反例。在 Terminal-Bench 的一个任务里，GPT-5.6-sol 最高档的运行自称满足了一个数值约束，评测脚本却不认，低档的运行反而通过了。

### 和以前方法的对比

那档位该怎么选？作者比较了几种策略。第一种是让模型看着任务描述自己选档位，结果和随机分配相比，没有稳定的成本或准确率优势。

第二种叫朴素阶梯，naive laddering。先用最低档跑，外部检查发现失败了，再用最高档重跑一次，两次的钱都算上。这张图里，绿色三角就是它。

论文报告，阶梯策略在所有模型和基准上，成功率都追平或超过固定用最高档。成本的中位数节省，AIME 上是百分之二十五点三，GPQA 上是百分之四十一点五，修正版 SWE-Bench Pro 上是百分之十五点五。不过在 Terminal-Bench 上，八个模型成功率都提高了，只有四个同时更便宜，另外四个要多花钱。

举个例子，Kimi K3 在 Terminal-Bench 上，成功率从百分之七十三升到百分之八十四点三，成本降了百分之三十点五。如果再把用户等待的时间，按美国平均时薪 33.54 美元折成钱，GPT-5.6-terra 的节省会从百分之四点八扩大到百分之十九点五。

为了排除只是多给了一次机会，作者在 DeepSWE 上做了对照。阶梯策略比低档重试两次成功率高，比高档重试两次便宜得多，但成功率最高的，仍然是高档重试两次。

我的判断是，这篇论文最有用的不是哪个数字，而是把评估单位从总分换成了单个任务。总分上升，不等于每道题都更好，而用户每次拿到的，就是一次运行。

我的判断是，它的弱点也要看到。阶梯策略离理论上限的 oracle 还有明显差距，SWE-Bench Pro 只抽了 44 个实例，Agent 任务每档只跑一次，很多单个模型上的反转，还在随机波动的范围里。

### 局限

作者自己承认，阶梯策略只是权宜之计。它需要一个可靠的外部正确性检查，很多实际部署没有，而且每次升级都要完整再跑一遍。

等待成本的估算也很粗，只算了生成 token 的时间，假设用户全程干等，没算排队和工具执行。换成软件开发者的平均时薪 71.20 美元，八个模型的中位数节省是百分之十一点五，和用全行业平均时薪的百分之十一点六差不多。

### 未来方向

对使用者，作者的建议是，不要默认用最高档，也不要以为关掉推理就能省钱，比较配置时，要把等待时间算进去。

对厂商，作者希望推理的计费可以核实，并说清楚每一档到底控制什么，是 token 预算、轨迹长度，还是行为。研究上的下一步，是在执行过程中动态调整推理强度，遇到真正难的地方再多花计算。

我的判断是，如果你在做 Agent 产品，而且任务有测试或校验器，先低档、失败再升档，是现在就能用的做法。没有校验器的场景，这篇论文还给不出答案。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Johns Hopkins University, Together AI and Cornell University. The title asks a practical question: what effort level should I choose? The subtitle is: on the economics of reasoning effort in LLM agents.

Many reasoning models now ship with an effort dial, from low and medium up to high and extra high. The usual assumption is that higher is better, just pricier. This paper tests that assumption task by task, across eight models and four benchmarks.

### Problem

First, what the dial does. Reasoning effort doesn't swap the model. It changes how much computation the model spends at inference time, roughly how long it thinks. Providers bill reasoning tokens like any other tokens, so more effort usually means a bigger bill.

The trouble is that you pick the level before you know whether extra thinking will help. And even when the task succeeds, you can't tell whether the effort you paid for was necessary. The authors borrow a term from economics: reasoning effort is a credence good. The buyer sees the outcome, but can't judge the value of the extra computation.

Agents make this worse. An agent may call the model hundreds of times while its context keeps growing, so one effort setting gets multiplied across every step of every trajectory. And that setting is usually chosen once, by intuition, with no feedback.

### Prior work

So how has effort been studied so far? According to the paper, prior work either reports aggregate accuracy across effort levels, compares models at one fixed effort, or studies a single model on single-call benchmarks.

Others have documented overthinking on easy problems, and found that in agentic coding, input tokens, not output tokens, dominate consumption. What this paper adds is a task-level audit of whether the extra reasoning was worth buying.

### Key idea

The core is a counterfactual question. Had the user picked a different effort level, typically a lower one, how would the outcome and the cost have changed?

The design is simple. Within each model and benchmark, only the effort setting changes. Tasks, prompts and the agent harness stay fixed. Then, instead of comparing totals, they look task by task: which tasks did higher effort newly solve, and which tasks did lower effort solve that higher effort lost?

### Method

There are four benchmarks. Two are single-call: AIME 2026 for competition math, and GPQA Diamond for graduate-level science questions. Two are agentic: Terminal-Bench 2.1, where the agent works in a terminal, and SWE-Bench Pro, where it edits real code repositories.

There are eight models. The proprietary ones are three GPT-5.6 variants, sol, terra and luna, plus Claude Sonnet 5. The open-weight ones are GLM-5.2, Inkling, Kimi K3 and DeepSeek V4 Flash. Each model has its own set of levels. Some offer none, which turns reasoning off, and some go up to max.

Cost is measured from the actual tokens of every call, billed reasoning tokens included, then converted to dollars at each provider's published rates. AIME problems run ten times each and GPQA questions four times. Because of budget, each agentic task runs once per effort level. As an extra check, the authors reuse DeepSWE's public runs, which have four rollouts per task.

### Results

Finding one: aggregate scores hide lost tasks. Take Inkling on GPQA. Going from low to medium nets sixteen more questions solved. But underneath that number, there are five questions that low solved and medium failed on all four attempts.

This figure shows tasks gained and lost between adjacent effort levels on the agentic benchmarks. On Terminal-Bench, all eight models lose some tasks between their lowest and highest settings. For GPT-5.6-sol, going from high to extra high loses eight tasks and gains only five, so it drops from seventy-six to seventy-three solved out of eighty-nine.

The authors are careful here. Each agentic task runs once, so some reversals are just noise, and their four-rollout check finds these reversals at rates comparable to chance. So they call them observed reversals, not proof that higher effort lowers the success rate. Still, a user gets one run, not the expected value.

Finding two: higher effort overspends on easy tasks. To keep extra solves out of the comparison, they look only at tasks every effort level solves. On SWE-Bench Pro, GPT-5.6-sol's mean cost goes from twenty-seven cents at low to a dollar twenty-four at extra high, four point five four times as much. The abstract says the highest level can cost up to forty-two times the lowest.

And the extra money isn't only reasoning. For the same model on SWE-Bench Pro, mean trajectory length goes from fourteen point seven to twenty-eight point four model calls, and input-token charges rise from seventeen cents to sixty cents. Longer trajectories mean re-reading an ever-growing history at every step.

Here's the counterintuitive part: turning reasoning off doesn't reliably save money. On always-solved tasks, GPT-5.6-luna averages twenty-one point five calls at none versus five point four at low, and costs four point five eight times as much. What it saves on reasoning, it loses on longer trajectories.

Finding three: different effort levels solve tasks differently. The original SWE-Bench Pro containers keep the git history, and that history contains the reference fix. GPT-5.6-sol reads that reference commit on fifteen point nine percent of tasks at low, and seventy-five percent at extra high.

So the authors built a corrected environment that strips the git history and only allows package registries on the network. There, GPT-5.6-sol's solve rate at extra high falls from seventy-seven point three percent to fifty-nine point one. All eight models drop at their highest setting, by eleven point four to twenty-nine point five percentage points, and the drop is significant for five of them.

The other shift is more self-checking. Labeling trajectories with an agent-behavior taxonomy, the biggest change is in evaluation actions. Claude Sonnet 5 on Terminal-Bench, for example, goes from six point six to twelve point three verification steps on tasks both levels pass, and from four point three to five point five on tasks both levels fail. The extra checking doesn't necessarily change the outcome.

There are counterexamples too. On one Terminal-Bench task, GPT-5.6-sol's highest-effort run claims it met a numerical constraint, the evaluator rejects it, and the low-effort run passes.

### Comparison

So how should you choose? The authors compare a few policies. In the first, the model reads the task description and picks its own effort. That gives no consistent cost or accuracy advantage over random assignment.

The second is naive laddering. Start at the lowest level, and only if an external check says the run failed, rerun once at the highest level, paying for both runs. It's the green triangles in this figure.

The paper reports that laddering matches or beats fixed top effort on success for every model and benchmark. Median cost savings are twenty-five point three percent on AIME, forty-one point five on GPQA, and fifteen point five on corrected SWE-Bench Pro. On Terminal-Bench, though, all eight models gain success, but only four also get cheaper, and the other four pay more.

One example: Kimi K3 on Terminal-Bench goes from seventy-three to eighty-four point three percent success, at thirty point five percent lower cost. And if you also price the user's waiting time at the US mean hourly wage of thirty-three fifty-four, GPT-5.6-terra's saving grows from four point eight to nineteen point five percent.

To rule out the effect of simply getting a second try, they run a control on DeepSWE. Laddering beats two low-effort tries on success, and is far cheaper than two high-effort tries, though two high-effort tries still succeed most often.

My read is that the most useful thing here isn't any single number. It's changing the unit of evaluation from the aggregate score to the individual task. A higher total doesn't mean every task got better, and a user only ever gets one run.

My read is also that the weaknesses matter. Laddering is still well short of the oracle, SWE-Bench Pro is a forty-four-instance sample, agentic tasks run once per level, and many per-model reversals sit within the range of chance.

### Limitations

The authors call laddering a stopgap. It needs a reliable external correctness check, which many deployments don't have, and every escalation pays for a full second run.

The waiting-cost estimate is rough as well: it counts only token generation time, assumes the user is blocked the whole time, and ignores queueing and tool execution. Using the software-developer mean wage of seventy-one twenty instead gives a median saving of eleven point five percent across the eight models, close to the eleven point six at the all-occupation wage.

### Future work

For users, the authors' advice is: don't default to the highest level, don't assume turning reasoning off saves money, and count waiting time when you compare configurations.

For providers, they want reasoning charges to be verifiable, and each effort level to state what it actually controls: token budget, trajectory length, or behavior. On the research side, the next step is adapting effort during execution, spending more compute only where the agent actually hits difficulty.

My read is that if you're building an agent product and your tasks come with tests or a verifier, start low and escalate on failure. That's something you can use today. Where there's no verifier, this paper doesn't have an answer yet.

That's today's Daily Frontier Papers. The paper is linked in the description.

