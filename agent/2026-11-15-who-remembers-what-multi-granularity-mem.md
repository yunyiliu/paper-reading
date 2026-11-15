# Who Remembers What? Multi-Granularity Memory Extraction for Multi-User, Multi-Agent Interaction

**多用户共享记忆还要守住隐私**

- 论文 Paper: [arXiv 2611.10439](https://arxiv.org/abs/2611.10439)
- 单位 Affiliation: AWS Agentic AI · 南加州大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-15
- 视频 Videos: [YouTube 中文](https://youtu.be/Da1NITyi6_E) · [YouTube English](https://youtu.be/Vc85SA1KCJU) · [B站](https://www.bilibili.com/video/BV1b9sJ6NE3S)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇亚马逊 AWS Agentic AI 团队和南加州大学的论文，题目可以翻译成，谁该记住什么，面向多用户、多 Agent 的多粒度记忆。

它要回答的问题是，一大群用户，用着各种不同的 Agent，怎么互相分享经验，同时又不把谁的隐私泄露出去。

### 问题

先说背景。自我进化的 Agent，不重新训练底层大模型，而是把经验写成技能、记忆，甚至改自己的外壳代码，也就是 harness。

再往前一步，就是在很多 Agent 之间共享经验。但作者指出，一旦跨用户、跨 Agent 共享，就得回答两个问题，这条经验对谁有用，哪些信息必须保密。

论文第一张图就是例子。某个软件包有隐藏依赖要装，这对所有人都有用。用户说不喜欢破折号，这是个人偏好，应该跟着用户走。某个 Agent 做的 PPT 很潦草，改进的技巧应该跟着这个 Agent 走。

作者把这叫做 cross-learning，交叉学习，并提出三个要求。可迁移，一个人发现的经验，有机会帮到所有人。受众选择，每条经验有自己适用的范围。保密，每条经验有一个披露边界，超出去就会暴露原来那个用户的隐私。

### 以前的方法

作者画了一张演进图。一个 Agent 服务一个用户，各记各的。一个 Agent 团队服务一个用户，共享记忆。一个 Agent 服务很多用户，用户之间要隔离。这篇要做的，是一大群用户和各种 Agent 混在一起共同进化。

已经有系统在用户之间共享经验，但谁能收到，是事先定死的。比如 SkillClaw 把验证过的技能广播给所有人，Collaborative Memory 用访问策略，把私有记忆和共享记忆分开。

作者的看法是，一条经验可以沿两个方向推广，跨用户，和跨 Agent。表 1 把现有系统对到这些范围上，它们都只覆盖一部分，共享规则要么是固定的，要么按策略设定，而 MixUA 是对每一条记忆去学出来的。

### 核心想法

MixUA 的核心想法很简单。两个方向，每个方向要么只留给来源本身，要么开放给所有人，组合起来就是四个范围。

PAIR，只有这一对用户和 Agent 能看。USER，这个用户用任何 Agent 都能看。AGENT，这个 Agent 服务任何用户都能看。GLOBAL，所有人都能看。

每种范围放不同的知识。GLOBAL 放大家都用得上的，比如某个库的缺陷，甚至某个任务的解法。USER 放用户偏好，比如文档格式。AGENT 放某个 Agent 的工具调用怪癖，或者常犯的错。PAIR 放跨 Agent 会冲突的偏好，比如让啰嗦的 Agent 简洁点，让简洁的 Agent 详细点。

隐私怎么保证。四个范围里，只有向其他用户开放这一步会暴露隐私，所以只在这一步把关。用公式说，一条记忆的受众，必须同时落在它的适用范围和披露边界之内。

### 方法

具体流程看这张图。各个用户和 Agent 的组合独立做任务，产生轨迹，也就是做任务的完整记录。

第一个设计叫双流写入。每条轨迹拆成两部分，可复用的流和受保护的流，分别交给两个互相隔离的写入器。这样，看到受保护内容的写入器，永远写不出共享的记忆。

而且，不靠写入器自己给记忆贴标签，而是看它来自哪个流。来自受保护流的记忆，只能进它主人的 USER 或 PAIR 记忆。

每条记忆有四部分，适用的情境，经验本身，范围，还有出处，也就是来自哪些任务、哪一轮、结果如何。每条经验都要绑定情境，检索时才能和任务对上。

写入器有两种。一种是大模型一次返回所有操作，一种是 Agent 式写入器，能查看样本和记忆库，多轮修改。

第二个设计是定期复审。写入器一次只看一条轨迹，范围可能定得太窄或太宽，还会写出重复的经验。所以每一轮之后，带着整个记忆库重新调整范围，再合并同一处的重复记忆。

检索很直接。做任务之前，只从这对组合有权看的记忆里取，按情境和任务的向量相似度排序，每个范围有自己的字数预算。

### 实验和效果

评测用的是作者新做的基准 CrossLearnBench。任务来自七个现有的 Agent 基准，比如 Terminal-Bench 2.0 和 SpreadsheetBench，在 Docker 沙箱里跑，环境和判分器都不改。

在这之上加了两层。第一层是隐私，给每个合成用户造一个人设，把私密信息写成文件，用中性的名字放在项目根目录，比如一份填了一半的贷款申请，叫 form.txt，再看它会不会进到别人能读的记忆里。

第二层是偏好。Agent 做完任务要写总结文档，每个用户有四个偏好，文件名、日期格式、章节顺序，和禁用一种标记，比如破折号。偏好只在第一轮说一次，之后全靠记忆。

实验里，合成用户配七种大模型加 harness 的组合，包括 Claude Code、Codex、Hermes 和 OpenClaw。主实验是五个用户，按排好的日程跑三轮，每轮 171 次尝试，每个条件跑三次。

结果在表 2。不用记忆，第三轮通过率是 72.0%。MixUA 用大模型写入器加复审，是 89.8%，第二、三轮相比不用记忆，提高了 16.3 个百分点。

单一范围的记忆都只能顾一头。只用 GLOBAL，提高 11.5 个百分点，但偏好只保留 11%，和不用记忆一样。只用 PAIR，偏好保留 80%，但只提高 4.3 个百分点。论文报告，MixUA 两头都要到了，提高 13 到 16 个百分点，偏好保留 80% 到 90%。

更关键的是交叉的情况。一个任务，之前只被别的用户、用别的 Agent 做过，这时 MixUA 仍然提高 11 到 14 个百分点，只用 GLOBAL 是 8.9，只用 PAIR 是 1.6。

再看隐私。用 MixUA 时，Agent 在 11% 到 26% 的尝试里打开了埋下的私密文件。但四个变体里有三个，在各约一万八千次探测里，一次也没把私密信息放进禁止的范围。剩下的 Agent 式写入器有 4 次违规，复审后清掉了。

作者还测了随机分配任务，更接近真实部署。第 2 到第 8 轮的 821 次尝试里，有 526 次是之前没有任何组合做过的新任务。在这些新任务上，MixUA 提高 10.1 个百分点，几乎是只用 GLOBAL 的 5.5 的两倍。

偏好也保得住。到第八轮，MixUA 仍保留 76%，只用 PAIR 却从 87% 掉到 44%。作者推测，是它的偏好要和约 1700 条任务记忆抢同一份预算。

### 和以前方法的对比

前面那几个单一范围的记忆，其实代表以前的设计。只用 PAIR 对应 MemGPT 这类助手记忆，只用 USER 对应 MIRIX，只用 AGENT 对应 G-Memory，只用 GLOBAL 对应 SkillClaw 这类技能库。要注意，这些是作者搭的替身，不是原系统。

消融实验在表 4，每次拿掉一个范围。拿掉 GLOBAL，对通过率影响最大，总体提升从 15.3 降到 9.0，交叉情况从 13.0 降到 4.6。拿掉 USER，偏好保留从 90% 降到 81%。

拿掉 AGENT 或 PAIR，基准上变化不大，作者就专门设计了探针。每个 Agent 需要自己的构建方式、预算又紧的时候，没有 AGENT，探针通过率从 89% 掉到 40%。没有 PAIR，只想留给一个 Agent 的工作背景，出现在其他 Agent 记忆里的比例，从 22% 升到 95%。

提升会不会只是因为塞进去的文字更多。看表 3，每个范围限制在一万字符时，MixUA 每个任务的上下文约 2.3 万字符，不到只用 GLOBAL 的 60%，提升却更大，13.1 对 11.5。

我的判断是，这篇最有价值的地方，是把一条记忆该给谁看，变成了一个明确的设计问题，而且同时测效果、个性化和隐私，而不是只看通过率。

### 局限

作者列了几条局限。第一，用户和 Agent 都被当成彼此独立。真实世界有团队、部门、公司这样的层级，团队内部的秘密，既不是个人的，也不是全局的，这套规则表达不了。

第二，受众只有一个和全部两档。只对一部分人有用的经验，比如一个团队，要么给得太少，要么给得太多。

第三是规模。实验只有 5、20、100 个合成用户，七个 Agent，最多 546 个任务、八轮。记忆库无限增长、写入成本越来越高，这些都还没解决。

论文数据里还能看到几点。用户和隐私信息全是合成的，作者也说，探针泄露少，不代表可以不做访问控制和隐私审查就上线。按用户数分组的随机实验里，每组一个种子，MixUA 比只用 GLOBAL 的优势都不显著。成本也更高，表 2 成本一栏，MixUA 加复审是 3.43，只用 GLOBAL 是 1.77，不用记忆是 0.30。

### 未来方向

未来方向，作者想给用户和 Agent 的关系、层级建模，每层有自己的共享和披露边界，加上介于一个和全部之间的群组范围，并扩展到真实用户的开放人群。

作者还提到，真正部署，需要用户同意才能共享，要让用户能查看、删除从自己身上学到的东西，还要防有人故意写误导性的经验。记录出处让查看和删除成为可能，但这些论文都没有评测。

我的判断是，对做多用户 Agent 产品的团队，这篇给了一个可以直接借鉴的框架，四个范围、双流写入、定期复审。但离上线还差两件事，真实用户的验证，和对恶意写入的防御。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the AWS Agentic AI team and the University of Southern California, and it asks a deceptively simple question: who should remember what?

The setting is a large population of users working with many different agents. How do they share what they learn, without leaking anyone's private information?

### Problem

Some background. Self-evolving agents improve without retraining the underlying model. They turn experience into skills, into memory, even into changes to their own harness code.

The next step is to share those lessons across many agents. But the authors point out that once you share across users and across different agents, you have to decide two things: who can benefit from each lesson, and what has to stay private.

The first figure makes this concrete. A package with a hidden dependency that needs installing is a fix everyone can use. A user who says "I don't like em dashes" has a personal preference that should follow that user. An agent that keeps making sloppy slide decks needs a trick that should follow that agent.

The authors call this setting cross-learning, and they set three requirements. Transferability: a new insight found by anyone should have a chance to help everyone. Audience selectivity: every lesson has its own range of validity. And confidentiality: every lesson has a disclosure boundary, and past it, it exposes the user it came from.

### Prior work

Here's how the authors lay out the progression. One agent serving one user, with its own memory. A team of agents serving one user, with shared memory. One agent serving many users, who must be kept apart. And their target: a mixed population of users and agents evolving together.

Systems that already pool experience across users fix the recipients in advance. SkillClaw broadcasts validated skills to everyone. Collaborative Memory uses access policies to split private and shared memory.

The authors' framing is that a lesson can generalize along two axes, across users and across agents. Table 1 maps existing systems onto these scopes. Each covers only some of them, with a sharing rule that's either fixed or set by policy. MixUA learns it, entry by entry.

### Key idea

The core idea is simple. Take the two axes, and on each one, either keep a lesson with its origin or open it to everyone. That gives four scopes.

PAIR is read only by that one user with that one agent. USER is read by that user on any agent. AGENT is read by that agent with any user. And GLOBAL is read by everyone.

Each scope suits a different kind of knowledge. GLOBAL holds what helps everyone, like a library defect, or even the fix for one task. USER holds a user's preferences, like a document format. AGENT holds advice for one agent, like a tool-calling quirk or a mistake it tends to make. PAIR holds preferences that conflict across agents, like asking a verbose agent for terse replies but a terse one for more detail.

What about privacy? Of the four scopes, only opening a lesson to other users can expose anything, so that's the only step gated on privacy. Formally, an entry's audience has to sit inside both its scope of validity and its disclosure boundary.

### Method

Here's the loop. Independent user-agent pairs do their tasks and produce trajectories, the full record of each attempt.

The first design choice is dual-stream writing. Each trajectory is split into a reusable stream and a protected stream, and each goes to its own isolated writer. A writer that has seen protected content never writes a shared entry.

And the system doesn't trust the writer to label its own entries. Each entry is labeled by the stream it came from, so a protected entry can only land in its owner's USER or PAIR memory.

Each entry has four parts: the situation where it applies, the lesson, a scope, and provenance, meaning which tasks, pairs, rounds and outcomes it came from. Every lesson must be tied to a situation, so retrieval can match it to the right tasks.

There are two writers. One is a plain language model call that returns all its edits at once. The other is an agentic writer that can inspect the sample and the store and edit over several turns.

The second design choice is periodic store review. A writer sees one trajectory at a time, so it can scope a lesson too narrowly or too broadly, and different trajectories produce duplicates. So after every round, a review re-scopes each entry with the whole store in view, then merges entries within each destination.

Retrieval is straightforward. Before each attempt, the system only draws from the memory that pair is allowed to read, ranks entries by embedding similarity between their situation and the task, and fills a character budget per scope.

### Results

To evaluate this, the authors built a benchmark called CrossLearnBench. The tasks come from seven existing agent benchmarks, including Terminal-Bench two point oh, SpreadsheetBench and SWT-Bench, run in Docker sandboxes with environments, instructions and verifiers unchanged.

On top of that sit two layers. The first is privacy. Each synthetic user gets a persona, and private facts are planted as files with neutral names at the project root, like a half-filled loan application saved as form dot t x t. Then they check whether those facts end up in memory other users can read.

The second layer is preferences. After each task, the agent writes a summary document, and each user has four document preferences, such as a filename, a date format, an order of sections, and one banned piece of markup, like em dashes. The preferences are stated only in round one. After that, memory has to carry them.

The agents are seven model-and-harness combinations, including Claude Code, Codex, Hermes and OpenClaw. The main experiment has five synthetic users and a fixed schedule over three rounds, one hundred seventy-one attempts per round, with every condition run three times.

The main results are in Table 2. With no memory, the round three pass rate is seventy-two point zero percent. MixUA with the language model writer plus store review reaches eighty-nine point eight, a gain of sixteen point three percentage points over no memory in rounds two and three.

Every single-scope store gets only half the job done. GLOBAL alone gains eleven point five points but retains just eleven percent of preferences, the same as no memory. PAIR alone keeps eighty percent of preferences but gains only four point three points. The paper reports that MixUA gets both: gains of thirteen to sixteen points with eighty to ninety percent retention.

The more telling case is crossover. On tasks that were tried before only by a different user with a different agent, MixUA still gains eleven to fourteen points, against eight point nine for GLOBAL alone and one point six for PAIR alone.

Now privacy. With MixUA, agents open a planted private file in eleven to twenty-six percent of attempts. Yet three of the four variants put no planted fact into a forbidden scope, across about eighteen thousand probes each. The remaining one, the agentic writer, had four violations, and store review removed them.

The authors also ran random task assignment, closer to a real deployment. In rounds two to eight, five hundred twenty-six of eight hundred twenty-one attempts were on tasks no pair had tried before. On those, MixUA gains ten point one points, nearly twice the five point five of GLOBAL alone.

Preferences hold up too. MixUA still retains seventy-six percent in round eight, while PAIR alone drops from eighty-seven to forty-four percent. The authors suggest that's because its preferences compete with about seventeen hundred task entries for one budget.

### Comparison

Those single-scope stores are stand-ins for earlier designs. PAIR alone plays an assistant memory like MemGPT, USER alone a personal memory like MIRIX, AGENT alone a team memory like G-Memory, and GLOBAL alone a broadcast skill library like SkillClaw. Keep in mind these are the authors' own stand-ins, not the original systems.

The ablations are in Table 4, removing one scope at a time. Removing GLOBAL hurts utility most: the overall gain falls from fifteen point three to nine point zero, and the crossover gain from thirteen point zero to four point six. Removing USER hurts preferences most, from ninety to eighty-one percent.

Removing AGENT or PAIR barely moves the benchmark, so the authors built targeted probes. When each agent needs its own build mode and the budget is tight, removing AGENT drops the probe pass rate from eighty-nine to forty percent. Removing PAIR raises the rate at which a work context meant for one agent shows up for the user's other agents, from twenty-two to ninety-five percent.

There's also the obvious question: is the gain just more text in the prompt? Table 3 answers it. At ten thousand characters per scope, MixUA delivers about twenty-three thousand characters per task, under sixty percent of what GLOBAL alone delivers, and still gains more, thirteen point one against eleven point five.

My read is that the paper's main contribution is turning "who gets to read this memory" into an explicit design problem, and measuring utility, personalization and privacy together instead of pass rate alone.

### Limitations

The authors list several limitations. First, users and agents are modeled as independent. Real users sit in teams, departments and organizations, and a secret shared within one team is neither personal nor global. These rules can't express that.

Second, an audience is either one or all. A lesson that's valid for a subset, like one team or the agents built on one model, either reaches too few or too many.

Third, scale. The study has five, twenty or one hundred synthetic users, seven agents, and at most five hundred forty-six tasks over eight rounds. A store that grows without bound, and writing costs that grow with it, are unsolved.

A few more things are visible in the paper's own data. All users and private facts are synthetic, and the authors say low leakage in their probes doesn't justify deploying shared memory without access control and privacy review. In the user-scaling runs, one seed each, MixUA's edge over GLOBAL alone was not significant. And it costs more: in Table 2's cost column, MixUA with review is three point four three, GLOBAL alone one point seven seven, and no memory zero point three.

### Future work

Looking ahead, the authors want to model relations and hierarchies among users and agents, with sharing and disclosure boundaries at each level, add group scopes between one and all, and scale storage, retrieval and review to an open population of real users.

In their ethics section, they add that a real deployment would need user consent before sharing, a way for users to inspect and delete what was learned from them, and defenses against people who write misleading lessons on purpose. Recording provenance makes inspection and deletion possible, but the paper doesn't evaluate any of it.

My read is that teams building multi-user agent products can borrow this framework today: four scopes, dual-stream writing and periodic review. But two things stand between it and production: validation with real users, and defenses against malicious writes.

That's today's Daily Frontier Papers. The paper is linked in the description.

