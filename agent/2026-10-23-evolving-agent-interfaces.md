# Evolving Agent Interfaces

**接口不再靠人Agent自己进化**

- 论文 Paper: [arXiv 2610.28601](https://arxiv.org/abs/2610.28601)
- 单位 Affiliation: Simular · 加州大学圣塔芭芭拉分校、康奈尔、卡内基梅隆等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-23
- 视频 Videos: [YouTube 中文](https://youtu.be/JH6e1_2HcJw) · [YouTube English](https://youtu.be/r_aEEoDJJYE) · [B站](https://www.bilibili.com/video/BV14bnd6WEBU)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 Simular 的论文，作者来自加州大学圣塔芭芭拉分校、康奈尔和卡内基梅隆等，名字叫 Evolving Agent Interfaces，系统叫 Agent S4。

它的主张一句话就能说清楚，给 Agent 用的接口，不该全靠人来设计，接口设计本身，应该成为 Agent 的一项能力。

### 问题

先说什么是接口。电脑操作 Agent，就是替你用电脑的 Agent，它能看到什么、能做哪些操作，都由接口决定。最常见的，是看截图，再点鼠标、敲键盘。

论文开头举了两个例子。Agent 可能一格一格地往表格里填数，其实一行代码就能搞定，或者为了找一个函数，把两千行的文件整个打印出来。

作者认为，固定接口有三个问题。第一，进步全靠人工，每来一类新任务、新模型，都要研究员读轨迹、手写新接口。第二，接口和任务常常不匹配，只能靠多采样、多花算力硬补。第三，把经验总结成技能，也只能重组已有的操作，扩展不了接口本身。

### 以前的方法

以前的工作大致有两类。一类是人来设计接口，SWE-agent 说明了接口会影响 Agent 的表现，Agent S 和 S2 用人工设计的模块组织规划和动作，还有系统在点击之外加上 API 或代码执行。论文的评价是，这些接口要么是人设计的，要么是单个任务里临时写的。

另一类是积累经验，比如 Reflexion 把反馈存成文字，SkillWeaver 让网页 Agent 自己练出 API。自动设计 Agent 这条线上，还有 ADAS、Darwin Gödel Machine 和 AlphaEvolve，用模型改代码，再配上进化搜索。Agent S4 的区别是，它进化的是一整个仓库，接口代码和使用说明一起改，一起被挑选。

### 核心想法

核心想法是，接口本身就是代码。既然是代码，一个会读轨迹、会写代码的 Agent，就可以重新设计它，决定合并哪些操作、返回什么信息、结果怎么检查。

论文举了改表格的例子。用截图改，要先找格子，输入数值，再看屏幕确认。换成结构化接口，可以直接拿到相关的行和公式，一次批量修改，再返回改完的值。目标一样，但成本和可靠性不一样。

所以 Agent S4 产出的不是新模型，而是一个代码仓库，里面有可执行的接口，也有什么时候、怎么用它们的知识。做任务的模型和 harness 全程固定。harness 就是模型外面那层壳，负责调用模型、执行代码、管权限和预算。

### 方法

看这张总览图。系统里有两个角色，任务 Agent 拿当前的仓库去做题，元 Agent，也就是 meta-agent，读任务 Agent 的轨迹，找出问题，然后改仓库。

改的方式借用了进化算法。所有仓库组成一个种群，每一步挑出父代，做变异或者交叉。变异是改一个仓库，增加、修改或者删除接口和说明。交叉是从两个仓库里取长处，合成一个新仓库。

挑父代用的是二元锦标赛，随机抽两个，留下验证分数高的那个。新仓库不管分数高低都加入种群，父代也保留，早期的设计以后还能被拿回来重新组合。

这里有一个关键设计，作者叫非对称评估。元 Agent 只能看合成训练题的轨迹，而且这些轨迹没有对错标签。验证题的题目、轨迹和分数，它一律看不到，验证分数只用来决定谁当父代。

这样做，是为了防止仓库只对见过的题有用。元 Agent 在构建过程中，没见过任何一道基准题。

训练题是合成的，OSWorld V2 用了 30 道，ScienceBoard 用了 48 道，另外各有 15 道官方题做验证。出题由 GPT-6 Astra 协调，GPT-5.6-Sol 出题和审题，审题时会和官方题目比对，避免重合，全程不产生标准答案。

实验里，任务 Agent 和元 Agent 都用 Qwen3.8-27B，跑在一个改过的 Prime Agent harness 里，它在一个持续运行的 IPython 环境里执行代码。仓库从一个大约 550 行的种子开始，只有 15 个基础接口，比如截图、点击、读无障碍树、列出 Chrome 标签页。

每一轮进化构建 30 个新仓库，每构建一个，元 Agent 在 OSWorld V2 上有两个小时，在 ScienceBoard 上有一个小时。

### 实验和效果

主要结果在表 2。OSWorld V2 有 108 个长流程任务，都在真实的桌面和网页应用里完成。用进化出来的仓库，Qwen3.8-27B 的完整完成率是 21.30%，每个任务成本 1.10 美元，作者称这是开源电脑操作 Agent 的新最好成绩。

同一个模型，用 Qwen CUA 那套固定接口，完整完成率只有 5.56%。换成进化仓库，部分得分也从 20.69% 提到 46.90%，成本从 1.39 美元降到 1.10 美元。21.30 除以 5.56，就是摘要里说的 3.8 倍。

再看闭源模型。表里 GPT-5.5 是 13.00%，每个任务 25.50 美元，Claude Opus 4.8 是 20.60%，每个任务 72.40 美元。作者称，Agent S4 的成本大约是它们的 4.3% 和 1.5%。

但要注意，闭源模型的数字，是从 OSWorld V2 的原论文里引用的，用的是基准自带的截图式 harness，上限 500 步，Agent S4 用的是能直接跑代码的 Prime Agent，上限 300 步，所以这不是同一套设置下的对比。而且部分得分上，Opus 4.8 的 54.80% 仍然更高。

再换个领域。ScienceBoard 有 169 个任务，覆盖天文、分子可视化、地理信息、代数、定理证明和排版六个科学软件。重新进化一轮之后，成功率是 58.58%，Qwen CUA 是 41.4%，种子仓库是 55.03%。

进化出来的仓库，从种子的 15 个函数涨到 64 个，新增了 ChimeraX 的命令接口、GRASS GIS 的模块，和一次能算多个表达式的 KAlgebra 计算器。KAlgebra 上提升最大，成功率从 48% 到 68%。

这张图是一个报销任务，要收集收据和对应的信用卡扣款。固定接口的 Agent，一封一封点开邮件，处理了五次下载警告，两次在下载完成之前就去打开发票，为了找一笔 500 美元的 NeurIPS 扣款，滚动了八次，点了四次加载更多。

用进化接口的 Agent，通过浏览器接口，在已经登录的网页里直接调用网站后台，把邮件和附件链接作为文本拿回来，用 curl 下载、pdftotext 读 PDF，再用同一个接口查账单核对。

### 和以前方法的对比

先拆开看提升从哪里来，看表 4。同样是 Prime Agent，完全不给仓库，完整完成率已经有 15.74%。加上种子仓库，到 19.44%，进化之后，到 21.30%。

换算成题数，108 道题里，分别是 17 道、21 道和 23 道。也就是说，进化本身，比种子仓库多做对了 2 道题，部分得分从 44.90% 提到 46.90%。

进化策略的对比也在这张表。scratch VM，就是给元 Agent 一台能试跑代码的虚拟机。有它，部分得分最高，51.06%，但完整完成率只有 19.44%，不给虚拟机，完整完成率反而最高。

还有一个细节。在 ScienceBoard 上，作者每五步把前三名重新评估。有两个仓库第一次验证拿到 73.33%，再测一次，平均掉到 60% 和 66.67%。单次验证分数，确实会高估。

我的判断是，这篇最有价值的地方，是把接口从人定好的配置，变成可以被搜索、被挑选的代码，而且用元 Agent 看不到的验证集来控制过拟合，这比只在训练题上自我反思更稳。

我的判断是，也要看清楚，相对 Qwen CUA 的 3.8 倍，很大一部分来自换成能直接跑代码的 harness。进化相对种子只多了 2 道题，论文自己也说，长流程任务每次运行的波动很大，而 OSWorld V2 的完整评测没有给出重复运行的结果。

### 局限

论文没有单独写局限，下面几点来自论文自己的设置和数据。

第一，前面说过，和闭源模型的比较，harness 和步数上限都不同，数字也是引用来的。

第二，验证集只有 15 道题，一道题就是 6.7 个百分点，靠这么小的集合挑仓库，噪声不小。ScienceBoard 上加了重复评估，OSWorld V2 因为时间和成本太高，没有做。

第三，效率上的好处没有那么普遍。附录表 6 显示，108 道题平均下来，进化仓库和种子仓库的工具调用次数几乎一样。在两边都拿满分的 11 道题上，进化仓库反而从 87.8 次增加到 103.2 次。

第四，成本只算了做题时的模型调用，不包括进化仓库本身的花费，也不包括运行环境。

### 未来方向

作者的结论是，从合成经验里学接口，是提升电脑操作能力的一条路，和改进模型、改进 harness 是互补的。代码已经在 GitHub 开源。

我的判断是，接下来最值得看的，是同一个进化出来的仓库，换一个模型、换一个 harness 还管不管用。论文说仓库可以给兼容的 Agent 复用，但这篇里任务 Agent 一直是同一个模型。

我的判断是，也别忘了绝对水平，最好的结果，108 道题也只完整做完了 23 道，电脑操作 Agent 离真正可靠，还有很长的路。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Simular, with authors from UC Santa Barbara, Cornell, Carnegie Mellon and others. It's called Evolving Agent Interfaces, and the system is Agent S4.

The pitch fits in one line. The interfaces agents use shouldn't all be designed by people. Designing them should itself be something the agent can do.

### Problem

First, what's an interface here? A computer-use agent operates a computer for you, and its interface decides what it can see and which operations it can ask for. The usual one is screenshots in, mouse clicks and keystrokes out.

The paper opens with two examples. An agent may type a spreadsheet cell by cell when one line of code would do, or print all two thousand lines of a file just to find one function.

The authors see three problems with fixed interfaces. One, progress depends on manual labor: every new family of tasks or new model means a researcher reading trajectories, diagnosing failures, and writing new primitives by hand. Two, the interface often doesn't fit the task, and today's fix is more sampling and more test-time compute. Three, methods that turn experience into skills can only recombine the operations the interface already has. They can't extend it.

### Prior work

Earlier work falls into two camps. In the first, people design the interface. SWE-agent showed that the agent-computer interface shapes performance, Agent S and S2 organize planning, perception and action through engineered components, and other systems add APIs, code execution or tool calls next to clicking. The paper's point is that these interfaces are either designed by people or written within a single task.

In the second camp, agents accumulate experience: Reflexion stores textual feedback, Agent Workflow Memory induces reusable procedures, and SkillWeaver has web agents practice their way to APIs. On the automated agent design side, there's ADAS, the Darwin Gödel Machine and AlphaEvolve, which pair model-written code changes with evolutionary search. What Agent S4 evolves is a whole repository, where interface code and usage notes are changed and selected together.

### Key idea

The key idea is that interfaces are code. And if they're code, an agent that can read trajectories and write code can redesign them: which operations to combine, what to return, and how to check the result.

The paper's example is a spreadsheet. Through screenshots, the agent finds the visible cells, types values, and looks at the display to check. A structured interface can expose the relevant rows and formulas, take one bulk edit, and return the changed values. Same goal, but different decisions, different feedback, and different cost and reliability.

So Agent S4's output isn't a new model. It's a code repository of executable interfaces, plus knowledge about when and how to use them. The task agent's model and harness stay fixed throughout. The harness is the layer around the model that invokes it, runs code, and manages permissions and budgets.

### Method

Here's the overview figure. There are two roles. A task agent attempts tasks using the current repository. A meta-agent reads the task agent's trajectories, diagnoses what went wrong, and rewrites the repository.

The rewriting borrows from evolutionary algorithms. Repositories form a population, and each step picks parents and applies mutation or crossover. Mutation revises one repository, adding, changing or removing interfaces and notes. Crossover builds one new repository from the strengths of two, which doesn't have to be the union of both.

Parents are picked by binary tournament: draw two at random, keep the one with the higher validation score. Every new repository joins the population whatever its score, and parents stay too, so early designs can be revisited and recombined later.

Then there's the key design choice, which the authors call asymmetric evaluation. The meta-agent only sees trajectories from synthetic training tasks, and those have no success labels. The validation tasks, their trajectories, diagnostics and scores are all hidden from it. Validation scores only decide who becomes a parent.

The point is to keep the repository from overfitting to the tasks it learned from. The final repository is then run on the full benchmark, and the meta-agent never saw a single benchmark task while building it.

The training tasks are synthetic: thirty for OSWorld V2 and forty-eight for ScienceBoard, plus fifteen official tasks each for validation. GPT-6 Astra coordinated task creation, and GPT-5.6-Sol wrote and reviewed the tasks, checking them against the official ones to avoid overlap. No step produced reference solutions or graded the agent's attempts.

In the experiments, both the task agent and the meta-agent are Qwen3.8-27B, running in a modified Prime Agent harness that executes code in a persistent IPython kernel. The repository starts from a seed of about five hundred and fifty lines, with fifteen basic interfaces, like taking a screenshot, clicking, reading the accessibility tree, and listing Chrome tabs.

Each evolution run builds thirty new repositories. For each one, the meta-agent gets two hours on OSWorld V2 and one hour on ScienceBoard.

### Results

The main results are in Table 2. OSWorld V2 has one hundred and eight long-horizon tasks in real desktop and web applications. With the evolved repository, Qwen3.8-27B fully completes twenty-one point three percent of them, at one dollar ten per task. The authors call this a new state of the art for open-source computer-use agents.

The same model with the fixed interfaces of Qwen CUA completes only five point five six percent. With the evolved repository, partial score also rises from twenty point seven to forty-six point nine percent, and cost drops from one thirty-nine to one ten per task. Twenty-one point three over five point five six is the three point eight times in the abstract.

Now the closed models. In the table, GPT-5.5 scores thirteen percent at twenty-five fifty per task, and Claude Opus 4.8 scores twenty point six percent at seventy-two forty per task. The authors put Agent S4's cost at about four point three and one point five percent of theirs.

But note that the closed-model numbers are quoted from the OSWorld V2 paper. They used the benchmark's own screenshot-based harness with a five-hundred-step budget, while Agent S4 runs in Prime Agent, which executes code directly, with a three-hundred-step budget. So this isn't a comparison under one setup. And on partial score, Opus 4.8's fifty-four point eight is still higher.

Then a different domain. ScienceBoard has one hundred and sixty-nine tasks across six scientific applications, covering astronomy, molecular visualization, geographic analysis, algebra, theorem proving and typesetting. After a fresh evolution run, success reaches fifty-eight point six percent, against forty-one point four for Qwen CUA and fifty-five point zero for the seed repository.

The evolved repository grows from fifteen functions in the seed to sixty-four, adding a ChimeraX command interface, GRASS GIS modules that run in the live session, and a KAlgebra calculator that evaluates many expressions in one call. KAlgebra gains the most, with success rising from forty-eight to sixty-eight percent.

This figure walks through one task: an expense claim that needs receipts and the matching card charges. The fixed-interface agent opens the emails one at a time, deals with five browser download warnings, twice tries to open an invoice before it has finished downloading, and needs eight scrolls and four clicks on Load more to find a five-hundred-dollar NeurIPS charge.

The agent with evolved interfaces uses its browser interface to call the site's own backend from inside the logged-in tab, getting the mail and attachment links back as text. It downloads the PDFs with curl, reads them with pdftotext, and uses the same interface to pull the bank records and cross-check the charges.

### Comparison

Now let's take the gain apart, using Table 4. In the same Prime Agent harness with no repository at all, full completion is already fifteen point seven four percent. The seed repository brings it to nineteen point four four, and evolution to twenty-one point three.

In tasks out of one hundred and eight, that's seventeen, twenty-one and twenty-three. So evolution itself solves two more tasks than the seed, and lifts partial score from forty-four point nine to forty-six point nine percent.

The same table compares evolution strategies. A scratch VM is a virtual machine where the meta-agent can open applications and try out its code. With it, partial score is highest, at fifty-one point one percent, but full completion is nineteen point four four. Without it, full completion is the highest of all. Linear evolution, which only ever revises the latest version, gets nineteen point four four full completion and forty-seven point two partial.

One more detail. On ScienceBoard, the authors re-evaluate the top three repositories every five steps. Two repositories scored seventy-three point three percent on their first validation run, but their means fell to sixty and sixty-six point seven after a second one. The repository finally selected averaged seventy-one point seven over four runs. A single validation score really does overrate.

My read is that the real contribution is turning the interface from a configuration people set into code that can be searched and selected, with overfitting held in check by validation the meta-agent can't see. That's sturdier than an agent reflecting on its own training tasks.

My read is also that you should look closely at that three point eight times over Qwen CUA. Much of it comes from switching to a harness that runs code directly: with no repository at all, it's already at fifteen point seven four. Evolution adds two tasks over the seed, and the paper itself says long-horizon tasks vary a lot from run to run, yet it reports no repeated runs of the full OSWorld V2 evaluation.

### Limitations

The paper has no limitations section, so these points come from its own setup and data.

First, as we saw, the closed-model comparison uses a different harness and a different step budget, and those numbers are quoted from another paper.

Second, there are only fifteen validation tasks, so one task is six point seven percentage points, and choosing repositories on a set that small is noisy. ScienceBoard adds repeated evaluation to deal with that, but OSWorld V2 doesn't, because of the time and cost.

Third, the efficiency gains aren't uniform. Table 6 in the appendix shows that across all one hundred and eight tasks, the evolved and seed repositories use almost the same number of tool calls, one sixty-two point one versus one sixty-three point two. On the eleven tasks both solve fully, the evolved repository actually goes from eighty-seven point eight calls to one hundred and three point two.

Fourth, the cost figures only count model calls while solving tasks. They leave out the cost of evolving the repository and the runtime infrastructure.

### Future work

The authors conclude that learning interfaces from synthetic experience is one way to improve computer use, complementary to better models and better harnesses. The code is open source on GitHub.

My read is that the next thing to watch is whether an evolved repository still helps with a different model or a different harness. The paper says repositories can be reused by compatible agents, but here the task agent is always the same model.

My read is also: keep the absolute level in view. The best result fully completes twenty-three of one hundred and eight tasks. Computer-use agents still have a long way to go before they're reliable.

That's today's Daily Frontier Papers. The paper is linked in the description.

