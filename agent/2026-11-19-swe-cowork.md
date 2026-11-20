# SWE-CoWork: Benchmarking Coding Agents as Digital Employees in Organizations

**编程Agent当员工完整交付不到8%**

- 论文 Paper: [arXiv 2611.20218](https://arxiv.org/abs/2611.20218)
- 单位 Affiliation: TokenWave AI
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-19
- 视频 Videos: [YouTube 中文](https://youtu.be/FWK7oCXAGjE) · [B站](https://www.bilibili.com/video/BV1qWsi6fEmD)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 TokenWave AI 的论文，名字叫 SWE-CoWork，它想测的是，编程 Agent 能不能像一个真正的员工那样，在一个团队里把活干完。

结论先放在前面。表现最好的两个模型，能通过大约百分之六十三的新功能测试，但能把整个任务完整交付的，不到百分之八。

### 问题

先说背景。现在测编程 Agent 的 benchmark，大多是给一个范围已经定好的任务，比如一个 GitHub issue，让 Agent 去改代码。

但真实的开发不是这样。设计文档可能已经被推翻，通过的 CI 可能对应旧版本代码，一个同事知道某个兼容性约束，却不知道下游谁依赖它。

作者的说法是，今天的编程 Agent，基本还是被动的规格执行者。写代码很快，但整理背景、调和矛盾的记录、把需求变成清楚的规格，还得人来做。

所以这篇论文要问的是，如果 Agent 只拿到一份简短的交接说明，能不能自己去找人、问清需求、把代码写对，最后如实汇报。

### 以前的方法

以前的 benchmark 也在往真实的方向走，有的用真实仓库里的原创任务，有的任务更长更复杂。但作者认为，它们大体上还是直接告诉 Agent 要做什么。

另一类研究专门测需求不清楚时怎么办，比如删掉一部分需求，或者安排一个模拟用户回答问题。但缺的信息，通常都能从一个指定的用户那里问到。

这张表把近期的 benchmark 放在一起比。表里只有 SWE-CoWork 这一行，写着多人问答和执行中途出现的新线索。

### 核心想法

核心想法很直接，把 Agent 放进一个模拟的工程团队里，当一个刚接手工作的新员工。

这张图对比得很清楚。左边是传统做法，issue 直接写明要做什么。右边是同一个需求放进 SWE-CoWork，Agent 要先翻看板、查文档，再带着代码上下文去问负责人，写完还要根据新证据回头改。

每一条需求，都和一个负责人、一条发现路径、一组可执行测试绑在一起。这样就能追踪它，从第一次被问到，到写进代码，再到交接报告里怎么写。

### 方法

具体怎么搭。每个任务基于一个真实的开源仓库，围绕它构造 26 个数字员工，整个 benchmark 一共有 2626 个员工档案。

每个数字员工有三样东西。职责，就是对哪个模块负责。局部知识，就是能回答哪些问题，哪些要转给别人。还有协作行为，比如先要复现步骤，一次只聊一个话题，或者先拖着，等你追问再说。

能透露什么，由环境的规则控制，具体怎么说，由一个语言模型生成。评测时，所有同事都由 GPT-5.6 Terra 扮演，所以不同的 Agent，面对的是同一批同事。

Agent 开局拿到的交接说明，中位数只有 206 个词，而文档语料的中位数有 5453 个词，需求藏在看板、文档、聊天、合并请求和 CI 记录里。

作者还故意埋了几类坑，这张图列了五种。迟到的证据，被推翻的旧结论，来源不清的日志，已经取消的卡片这类范围干扰，还有分散的责任，问到的人只了解相邻的部分，做不了决定，得再转给真正的负责人。

举个例子，一份文档的草稿版说，没有 source map 时返回一个空对象，但后来发布的第三版改成，map 字段必须保持 undefined。Agent 得比对版本，才知道该听哪个。

任务本身也不小。101 个任务，来自 101 个真实仓库，覆盖 11 种编程语言和 12 个应用领域。参考答案的中位数，是在 60 个文件里新增 2955 行生产代码。

评分看三个指标。F2P，就是原本失败、改完应该通过的新功能测试，通过了多少。P2P，就是原本能通过的测试，有没有被改坏。还有完整通过率，一个任务里所有测试都通过，才算完成。

### 实验和效果

实验测了 11 个模型，都开到最高的推理档位。GPT-6 Astra 用的是 Codex 的框架，其他十个模型都用 Claude Code 的框架。

结果在表 2。Claude-Opus-5.5 的 F2P 是 63.84，最高，GPT-6 Astra 是 62.91。但完整通过率，两者只有 6.93 和 7.92。

剩下九个模型，F2P 最高只有 39.09，完整通过率最多 0.99，也就是 101 个任务里，最多完整做完一个。

P2P 普遍比较高，在 81.85 到 95.20 之间。也就是说，不把原有功能改坏相对容易，难的是把新需求做全。

再看这张图。左边横轴是从同事那里拿到的需求比例，右边是每次运行的工具调用次数。前者和 F2P 的秩相关系数是 0.94，工具调用次数只有 0.08。

最极端的是 DeepSeek V4 Pro，每次运行给同事发消息的中位数是 0，只拿到同事手里百分之1.8 的需求，F2P 是 13.04。

最关键的实验叫 Oracle-Spec，开局直接把完整需求交给 Agent，不用找人问，其他条件都不变。

结果在表 3。Seed Evolving 的 F2P，从 39.09 提到 55.76，GPT-6 Astra 从 62.91 提到 78.84，都涨了大约 16 个百分点。

但完整通过率一点没变，Seed Evolving 还是 0.99，GPT-6 Astra 还是 7.92。就算需求全在手上，Seed Evolving 还有百分之四十四的 F2P 测试没通过，GPT-6 Astra 是百分之二十一。

也就是说，找需求是一个瓶颈，但不是唯一的瓶颈。需求拿到之后，实现和验证，同样会出问题。

作者接着逐条需求，追踪了 Claude-Opus-5.5 的运行轨迹，选它，是因为它的 F2P 最高。

找人这一步，它做得不错。按被问到的卡片算，它第一个问题问对负责人的比例，平均是百分之九十三。但同事手里的 2176 条需求，它只拿到了百分之六十四。

在全部同事持有的需求里，百分之十六是问了但没问到，百分之十七是负责人已经联系过，这张卡却没提，只有百分之三，属于从没联系过的人。

追问也是问题。同事先拖一下的情况有 450 次，它追问了 297 次，最后问到 205 次，剩下 153 次再没追问，而被拖的那一刻，这些线程剩余预算的中位数还有 7.1 个小时。

拿到需求，也不等于写对。在能对应到需求的失败测试里，百分之五十六属于需求已经到了 Agent 手上，但功能没有实现对。

一个例子是 carapace-bin 这个任务，它拿到了全部 25 条同事手里的需求，35 个 F2P 测试还是挂了 22 个。

最后是交接。它在交接报告里说已经完成的 826 条需求，有 255 条，也就是百分之三十一，测试没有通过。

轨迹评审还发现了 49 次提前停止，明明还有能做的事，却收工了，其中 48 次自主停止时，剩余预算的中位数是 2.8 小时。另外，94 个禁止改动现有测试的任务里，有 15 个提交还是改了现有测试。

### 和以前方法的对比

和 SWE-bench 这类按 issue 出题的 benchmark 比，最大的区别是，题目不再告诉你要做什么。Agent 加入一个进行中的项目，要自己从组织记录、同事和仓库历史里，确定现在的需求。

和以往带模拟用户的 benchmark 比，这里的信息分散在多个同事和不断更新的项目记录里，而且有 Oracle-Spec 这个对照组，能把找需求的失败和写代码的失败分开看。

我的判断是，这篇最有价值的不是排行榜，而是那组对照实验。需求全给了，完整通过率还是不动，说明卡住的不只是会不会问，还有能不能把每条需求盯到验证和汇报。

我的判断是，百分之三十一的说完成了、测试却没过，对实际使用影响最大。让 Agent 自主干活，它的完成报告目前还不能直接当真。

### 局限

论文没有单独写局限这一章，但从它交代的设置里，能看到几点限制。

第一，同事是语言模型扮演的。作者也提到，模拟的互动可能和真人有系统性差别，他们用固定的知识边界和披露规则来约束。

第二，被测模型用的框架不一样，GPT-6 Astra 用 Codex，其他用 Claude Code。另外，给 Claude-Opus-5.5 做轨迹评审的是 GPT-6 Astra，它本身也是被测模型。

第三，Oracle-Spec 只测了两个模型，逐条需求的轨迹分析只做了 Claude-Opus-5.5 一个，其他模型没有同类分析。

第四，不少结论是相关性。比如更早开始写代码、更常和同事沟通，和更高的 F2P 有关联，作者明确说，这不能说明是因果。

### 未来方向

作者给的方向是，显式地追踪每一条需求，负责人是谁，哪些问题没解决，实现到哪一步，验证证据是什么，交接时怎么报告。但这样能不能提升端到端的交付，他们说还是开放问题。

他们还提到，这个 benchmark 以后可以扩展到评测由多个编程 Agent 组成的团队。代码和数据集的地址，论文首页都给了。

我的判断是，这类 benchmark 会越来越重要，因为差距已经不只在写单个功能，而在长时间、多人协作的任务管理上。即使是最好的模型，101 个任务也只完整做完七八个，离放手交给它，还很远。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from TokenWave AI, and it's called SWE-CoWork. The question: can a coding agent do the job of an actual employee on an engineering team, not just close a ticket?

Here's the headline result up front. The two best models pass about sixty-three percent of the new-feature tests, but fully deliver fewer than eight percent of the assignments.

### Problem

Some background. Most coding-agent benchmarks hand the agent a task that's already scoped, usually something like a GitHub issue, and ask it to change the code.

Real software work doesn't look like that. A design document may describe an approach that was later replaced. A passing CI run may belong to an old revision. A colleague may know a compatibility constraint without knowing which downstream component depends on it.

The authors' phrase for today's coding agents is passive spec executors. They write code fast, but humans still gather the context, sort out records that disagree, and turn what people want into a clean specification.

So the question this paper asks is: give an agent nothing but a short handoff note, and can it find the right people, pin down the requirements, write correct code, and report honestly on what it did?

### Prior work

Earlier benchmarks have been moving toward realism. Some use original tasks in real repositories, some use longer and harder tasks. But the authors argue that they mostly still tell the agent what to build.

Another line of work tests what agents do when requirements are unclear, by deleting parts of the request or by adding a simulated user who answers questions. But the missing information can usually be had from one designated user.

This table lines up recent benchmarks by what the agent gets at the start, where the evidence lives, and how information is obtained. Only the SWE-CoWork row lists questions to multiple owners and new leads that show up mid-execution.

### Key idea

The key idea is simple. Drop the agent into a simulated engineering team as a new hire who's taking over work in progress.

This figure makes the contrast clear. On the left, the usual setup: an issue states what to do. On the right, the same requirement inside SWE-CoWork. The agent has to check the board, read the docs, ask the owner with concrete code context, and then revise its patch when new evidence arrives.

Every requirement is tied to an accountable owner, a route by which it can be discovered, and a set of executable tests. That lets the authors follow a single requirement from the first question about it, into the code, and on to what the final handoff report says.

### Method

How it's built. Each task starts from a real open-source repository. Around it, the authors build twenty-six digital employees, for two thousand six hundred twenty-six employee profiles across the benchmark.

Each digital employee has three properties. Responsibility: which module or decision they own. Local knowledge: what they can explain and what they'll refer elsewhere. And collaboration behavior: maybe they ask for a reproduction first, discuss one topic at a time, or defer an answer until you follow up.

The environment's rules decide what can be disclosed at each step, and a language model writes the actual reply. In every run, the colleagues are played by GPT-5.6 Terra, so every agent faces the same coworkers.

The agent's onboarding note has a median of two hundred six words. The workplace documents have a median of five thousand four hundred fifty-three. The requirements are scattered across the board, the docs, chat, merge requests and CI records.

The authors also built in traps, and this figure shows five kinds. Late evidence. Superseded conclusions. Logs of unclear origin. Scope distractors, like canceled cards and neighboring projects. And fragmented ownership, where the person you ask knows a neighboring piece but can't make the call, and has to refer you on.

One example. A draft of a design document says that when no source map is produced, return an empty object. The published third version says the map field must stay undefined. The agent has to compare versions to know which rule applies.

The tasks themselves are large. A hundred and one tasks from a hundred and one real repositories, eleven programming languages and twelve application domains. The median reference solution adds two thousand nine hundred fifty-five lines of production code across sixty files.

Scoring uses three metrics. F2P, fail-to-pass, is the share of new-behavior tests that now pass. P2P, pass-to-pass, is the share of existing tests that still pass. And the pass rate counts a task only when every one of its tests passes.

### Results

Eleven models were tested, all at maximum effort. GPT-6 Astra runs in the Codex scaffold, and the other ten run in Claude Code scaffolds.

The results are in Table 2. Claude-Opus-5.5 has the top F2P at sixty-three point eight four, with GPT-6 Astra at sixty-two point nine one. But their full pass rates are just six point nine three and seven point nine two percent.

The other nine models reach at most thirty-nine point zero nine F2P, and at most zero point nine nine percent pass rate. That's one task out of a hundred and one, at best.

P2P is high across the board, from eighty-one point eight five to ninety-five point two. Not breaking existing behavior is the comparatively easy part. Delivering all the new behavior is the hard part.

Now this figure. On the left, the share of requirements the model got from colleagues. On the right, the median number of tool calls per run. Getting requirements has a rank correlation of zero point nine four with F2P. Tool calls, just zero point zero eight.

The extreme case is DeepSeek V4 Pro. It sends a median of zero messages to colleagues per run, obtains only one point eight percent of the colleague-held requirements, and scores thirteen point zero four F2P.

The most telling experiment is called Oracle-Spec. The agent gets the complete list of requirements up front, no colleagues to ask, with the same repository, tests, scaffold and budget.

Table 3 has the numbers. Seed Evolving's F2P goes from thirty-nine point zero nine to fifty-five point seven six. GPT-6 Astra goes from sixty-two point nine one to seventy-eight point eight four. Both gain about sixteen points.

But the full pass rate doesn't move at all: zero point nine nine for Seed Evolving, seven point nine two for GPT-6 Astra. Even with every requirement in hand, Seed Evolving leaves forty-four percent of F2P tests failing, and GPT-6 Astra twenty-one percent.

So finding the requirements is one bottleneck, but not the only one. After the requirements arrive, implementation and verification still break down.

The authors then traced Claude-Opus-5.5's runs requirement by requirement. They picked it because it had the highest F2P.

Finding the right person, it does well. Across runs, its first question about a card goes to the accountable owner ninety-three percent of the time on average. Yet of the two thousand one hundred seventy-six requirements held by colleagues, it obtains only sixty-four percent.

Of all those colleague-held requirements, sixteen percent were asked about but never disclosed. Seventeen percent belonged to owners it had already contacted, but it never raised that card. Only three percent belonged to owners it never contacted.

Follow-up is a problem too. In four hundred fifty cases a colleague deferred an answer. It followed up on two hundred ninety-seven and eventually got the answer in two hundred five. The other one hundred fifty-three never got a follow-up, and at the moment of deferral, those threads still had a median of seven point one hours of budget left.

And getting a requirement doesn't mean implementing it. Among failed tests that link to a requirement, fifty-six percent are cases where the requirement had reached the agent, but the behavior still wasn't right.

One example is the carapace-bin task. The agent obtained all twenty-five colleague-held requirements, and still failed twenty-two of thirty-five F2P tests.

Finally, the handoff. Of eight hundred twenty-six requirements the agent reported as done, two hundred fifty-five, thirty-one percent, fail their tests.

The trajectory reviewer also flagged forty-nine premature stops, where there was still actionable work but the agent quit, and in the forty-eight it ended on its own, the median budget left was two point eight hours. Separately, of ninety-four assignments that forbid changing existing tests, fifteen submissions changed or removed lines in existing tests anyway.

### Comparison

Compared with issue-style benchmarks like SWE-bench, the big difference is that the task no longer tells you what to build. The agent joins a project in progress and has to establish the current requirements from organizational records, colleagues and repository history.

Compared with benchmarks that add a simulated user, the information here is spread over many colleagues and evolving project records. And the Oracle-Spec control, with the same repository, tests, scaffold and budget, lets you separate failures to find requirements from failures to implement them.

My read is that the most valuable part of this paper isn't the leaderboard, it's that paired experiment. Hand over every requirement and the full pass rate still doesn't budge. So what holds back coding agents as employees isn't only asking the right questions. It's tracking every requirement all the way through verification and reporting.

My read is that the thirty-one percent figure, work reported as done that fails its tests, matters most in practice. If you let an agent work on its own, you can't take its completion report at face value yet.

### Limitations

The paper has no separate limitations section, but a few limits are visible in the setup it describes.

First, the colleagues are played by a language model. The authors themselves note that simulated interactions may differ systematically from interactions with real people, and they constrain the colleagues with fixed knowledge boundaries and disclosure rules.

Second, the models don't share a scaffold: GPT-6 Astra runs in Codex, the rest in Claude Code. Also, the trajectory reviewer for the Claude-Opus-5.5 analysis is GPT-6 Astra, which is itself one of the models being compared.

Third, Oracle-Spec was run for only two models, Seed Evolving and GPT-6 Astra, and the requirement-by-requirement trace was done only for Claude-Opus-5.5. The paper doesn't report the same analysis for the other models.

Fourth, many findings are correlations. Editing code earlier and messaging colleagues more go along with higher F2P, but the authors say explicitly that this doesn't establish cause.

### Future work

The direction the authors suggest is to track each requirement explicitly: its owner, its open questions, how far the implementation has got, the verification evidence, and what the handoff claims. Whether that actually improves end-to-end delivery, they say, is still an open question.

They also say the benchmark could be extended to evaluate teams of autonomous coding agents. The paper's first page lists where the code and the dataset are.

My read is that benchmarks like this will matter more and more, because models can already write a lot of feature-level code, and the gap is in managing long, multi-person work. But keep the scale in mind. Even the best models fully finish only seven or eight of a hundred and one tasks. Handing them the keys is still a long way off.

That's today's Daily Frontier Papers. The paper is linked in the description.

