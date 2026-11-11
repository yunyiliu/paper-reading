# PCBRouteAgent: How Close Are LLM Agents to Becoming PCB Routing Engineers?

**大模型画电路板还差多远**

- 论文 Paper: [arXiv 2611.08175](https://arxiv.org/abs/2611.08175)
- 单位 Affiliation: 德州大学达拉斯分校 等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-10
- 视频 Videos: [YouTube 中文](https://youtu.be/dbPW0hUyPQ8) · [YouTube English](https://youtu.be/oys0M7vOUp8) · [B站](https://www.bilibili.com/video/BV1KRW36BE7H)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇德州大学达拉斯分校等单位的论文，名字叫 PCBRouteAgent，标题是一个问题，大模型 Agent 离成为 PCB 布线工程师，还有多远。

它做的事情是，让一个多模态大模型 Agent 给电路板布线，而且最后的板子，要由 KiCad 自己的设计规则检查来判对错。

### 问题

先说什么是布线。元件已经摆好，原理图规定了哪些焊盘要连通，一组要连通的焊盘叫一个 net，也就是网络。布线，就是用铜线把每个网络连起来，必要时打过孔，从正面换到背面。

这张图就是一个例子。红色是正面的铜线，蓝色是背面的，过孔是连通两面的金属化小孔。铜线要绕开螺丝孔和禁止布线的区域，线宽和间距都要满足规则。

难点在于，这些决定互相牵连。一条线局部走得通，可能正好堵住后面另一条线的路。反复失败的连接，可能要换走法或换层，而不是原地再试。

### 以前的方法

传统做法是规则式的自动布线器，用启发式搜索、网络排序和拥塞代价来找路径。论文说，碰到复杂的板子，它们仍然需要仔细调参和人工介入。

最直接的前作叫 PCBWorld。它把 KiCad 原生的布线操作，开放给强化学习策略和会用工具的大模型 Agent，Agent 每走一步，都能看到板子变成了什么样。

但在 PCBWorld 里，大模型 Agent 在既要连通、又要满足设计规则这件事上，仍然落后于规则式布线器，在复杂的真实板子上尤其明显。

### 核心想法

思路来自人类布线工程师。他们一边画线，一边想全局，先连哪些，给谁留哪块区域，什么时候换层，局部走不通，再回头改这些决定。

所以 PCBRouteAgent 分成两层。上面是全局布线规划器，管全局策略，初始方案用简单的算法生成，之后由多模态大模型来修改，下面是细节布线执行器，调用 KiCad 原生的推挤式布线引擎，去画真正的铜线。两层之间，靠结构化的失败反馈连成一个循环。

芯片设计里，全局布线加详细布线，也是这样分工的。这张图是整个框架的总览。

### 方法

规划器拿到两种输入。一种是图像，有整块板的布局、正反两面的铜线，和还没连上的需求，另一种是结构化文本，写着焊盘坐标、层、网络和设计规则。图像给空间感，文本给精确的数字。

第一步，决定怎么连。一个网络里已经连在一起的焊盘算一簇，规划器在簇之间求最小生成树，边的权重是两簇焊盘之间的最短距离，层不兼容再加罚分。这样不用穷举所有连法。

布线顺序也有一个简单的初始规则，按每个网络生成树的估计总长度排，短的先布。连法和顺序，后面都还能改。

第二步，是空间和层的规划。规划器不画具体的线，只给一个走廊，就是一块矩形区域。两个端点在同一层，就取它们的包围盒，四周各扩 1.5 毫米。

如果需要换层，就在连线的中点把它分成两段走廊，各在一层，中点周围再划一块过孔候选区。规划器不指定过孔的精确位置，交给执行器在这块区域里找。

执行器按规划的顺序逐条去做，在相邻走廊的重叠区里挑中间点，交给 KiCad 画线，换层时在候选区里试过孔。最后检查两簇是不是真的连通了，而不只是某一步操作成功。

失败了，不会马上找大模型。执行器先做有限的局部恢复，比如绕行，推开挡路的线，用 A 星搜索找绕路，换个起点，每种尝试都有次数上限。

局部恢复用完还不行，执行器就把失败打包成结构化反馈，写清楚试过什么，停在哪，可能被什么挡住，偏离规划多少。这条连接回滚到检查点，其他已接受的连接保留。

这时大模型规划器才上场。它看着图像、结构化信息、当前规划和这批反馈，输出一个补丁，只改出问题的走廊、过孔区、端点或顺序，其余不动。补丁先过一致性检查，不合格，就把错误信息交给它，让它改正。

每块板子最多重新规划 12 次，每条连接最多 4 次，一直循环到全部连通，或者没有任务可做，或者预算用完。

### 实验和效果

评测用的是 PCBWorld 的基准，一共 247 块板子。D2 是 128 块合成板，D3-A 是 99 块小型开源真实板，D3-B 是 10 块中型板，D3-C 是 10 块超过 100 个焊盘的大板。

主要指标叫 clean pass，干净通过率。一块板，只有所有连接都完成，并且 KiCad 报告零个错误级的设计规则违规，才算通过。结果由 KiCad 来评，不是 Agent 自己说了算。

规划器分别用了 Claude Opus 5.5 和 Claude Haiku 5.5。对比对象有 Freerouting 等三个规则式布线器，PCBWorld 里用 GPT-5.4 等模型的交互式 Agent，还有强化学习策略。基线数字大部分取自 PCBWorld 论文。

先看合成板 D2，在表 1。两个版本都是百分之百干净通过，和 Freerouting、KiCadRoutingTools 打平。和 Freerouting 比，Opus 版本的线长少百分之 4.2，过孔少百分之 19.7。

真实小板 D3-A，在表 2。Opus 版本干净通过率百分之 94.9，最强的基线是 PCBWorld 里的一个 PPO 强化学习策略，百分之 81，高出 13.9 个百分点。

中型板 D3-B，在表 3。Opus 版本 10 块板全部干净通过，规则式的 Freerouting 是百分之 64。要注意，这一组包含了模型回答失败后的重试和走线角度修复，论文说不算严格的一次通过。

大板 D3-C 最难，在表 4。Opus 版本干净通过率百分之 40，Freerouting 百分之 30，强化学习基线都是 0。平均每块板的设计规则违规，从 Freerouting 的 32 个降到 6.3 个。但十块板里，还有六块没有完成。

还有一个更公平的对比，同一个 Haiku 5.5，分别放进 PCBWorld 的交互式 Agent 和 PCBRouteAgent。四组的干净通过率，PCBWorld 是百分之 63.3、34.3、0 和 0，PCBRouteAgent 是百分之 100、92.9、80 和 10。

成本也低很多。247 块板加起来，模型调用从 22529 次降到 172 次，API 费用从 20.72 美元降到 4.77 美元，累计运行时间从 36.4 小时降到 2.6 小时。论文的解释是，PCBWorld 每一步动作都调一次模型，而这里模型只做规划，具体画线交给布线引擎。

附录里有一块大板的直接对比，同样用 Haiku。PCBWorld 的 Agent 用满了 403 步，86 条连接只连上 10 条，PCBRouteAgent 只调用了 5 次模型，连上了 84 条。

### 和以前方法的对比

那大模型规划到底贡献了多少。表 5 做了消融，保留同样的执行器和脚本化的局部恢复，只把大模型的重新规划关掉。

关掉之后，四组的干净通过率是百分之 96.1、53.5、0 和 0，加上 Opus 之后，是百分之 100、94.9、100 和 40。在 D3-B 和 D3-C 上，没有大模型时，平均连通率能到百分之 89.1 和 83.1，但一块板都没有干净通过。

代价是线更长、过孔更多。比如在 D3-A 上，Opus 让平均线长从 150.8 毫米增加到 165.5 毫米。在 D3-B 上，Opus 版本的线长比 Freerouting 多了百分之 10.9。

作者也说清楚了这个消融的边界。它只证明比关掉大模型要好，没有和专门的规则式重规划器，或者用同样工具的通用 Agent 比过。

我的判断是，这篇最有意思的不是分数，而是分工。大模型只做全局判断，几何细节交给成熟的布线引擎，比让大模型一步步亲手画线更好，也便宜得多。这个思路，对很多专业软件里的 Agent 都有参考价值。

我的判断是，也要注意比较的口径。大部分基线数字取自 PCBWorld 论文，PCBWorld 的 Agent 也没有换成 Opus 做完整评测，作者说是因为它每一步都调用模型，API 成本要高得多。另外，D3-B 和 D3-C 各只有 10 块板，每一块板，就是 10 个百分点。

### 局限

作者用案例讲了两条局限。

第一，前面的决定会堵住后面的路。在一块大板上，Opus 版本先布了传感器的电源线，没给 SCL 时钟线和 SDA 数据线留出焊盘入口和过孔空间，83 条连接只完成 70 条，原设计则全部连通。

第二，连通不等于电气上能用。这张图里，USB 的 D+ 和 D- 是一对差分信号，原设计让它们并排走，都是 30.25 毫米。PCBRouteAgent 把这块板的 55 条连接全部连通，但这两条线分开走，长度是 34.03 和 36.49 毫米，因为规划的输入里，没有描述这对信号的关系。

作者在结论里也承认，大板的完成率和走线的紧凑程度，仍然是重要的难题。

### 未来方向

未来方向，作者提到更好的空间和层规划，自适应的局部恢复预算，以及在保持连通的前提下，优化走线，降低物理成本。

从失败案例出发，作者还建议排序和修复时看得更远，提前给焊盘留出入口，优先处理选择少的连接，必要时回头改已经布好的线。代码已经在 GitHub 公开。

我的判断是，标题提的问题，论文答得很诚实。在连通加基本规则检查上，中小板已经接近做完，但真正的工程师还要管差分对这类和信号角色有关的约束，现在的任务定义和检查器都还没覆盖。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from a team led by the University of Texas at Dallas, and its title is a question: how close are LLM agents to becoming PCB routing engineers?

What they built is a multimodal language-model agent that routes circuit boards, with the finished board judged not by the agent, but by KiCad's own design-rule checks.

### Problem

First, what routing means. The components are already placed on the board, and the schematic says which pads must be connected. Each group of pads that must be joined is called a net. Routing means drawing copper traces that connect every net, and when needed, dropping a via to hop from the front of the board to the back.

This figure shows an example. Red is copper on the front, blue is copper on the back, and a via is a plated hole that joins the two sides. Traces have to avoid screw holes and keep-out areas, and meet the rules for width and spacing.

The hard part is that these decisions are coupled. A trace that works locally can block the path another net needs later. And a connection that keeps failing may need a different route or a different layer, not one more attempt at the same thing.

### Prior work

The classic approach is a rule-based autorouter, using heuristic search, net ordering and congestion costs. The paper notes that complex boards can still need careful tuning and manual work. Others have applied reinforcement learning to pieces of the problem, such as global routing and net ordering.

The most direct predecessor is PCBWorld. It exposes KiCad's native routing operations to reinforcement-learning policies and tool-using language-model agents, so the agent sees how the board changes after every action.

But in PCBWorld, the language-model agents still trailed rule-based routers at finishing connections while respecting design rules, especially on complex real-world boards.

### Key idea

The authors take their cue from human routing engineers. An engineer draws individual traces while keeping the whole board in mind: what to connect first, which region to save for whom, when to switch layers. When something gets stuck locally, they go back and revise those choices.

So PCBRouteAgent has two layers. On top, a global routing planner that owns the board-level strategy: its first plan comes from simple heuristics, and a multimodal language model revises it. Underneath, a detailed routing executor that drives KiCad's native push-and-shove router to lay real copper. Structured failure reports tie the two into a loop.

Chip design splits routing the same way, into global and detailed stages. This figure is the overview of the whole framework.

### Method

Start with what the planner sees. One part is images: the board layout, the copper on the front and back, and the connections still missing. The other part is structured text: pad coordinates, layers, net assignments and design rules. The images give spatial sense; the text gives exact IDs and numbers.

Step one decides how each net gets connected. Pads already joined form a cluster, and the planner builds a minimum spanning tree over the clusters. Edge weights are the shortest pad-to-pad distance, plus a penalty when layers don't match. That gives a compact starting plan without searching every possible topology.

The routing order starts from a simple rule too. Nets are sorted by the estimated total length of their spanning tree, shortest first. Both the topology and the order can change later.

Step two is spatial and layer planning. The planner doesn't draw traces. It hands out corridors, which are just rectangles. If both endpoints are on the same layer, the corridor is their bounding box, grown by one and a half millimeters on every side.

If a layer change is needed, the connection is split at its midpoint into two corridors, one on each layer, with a candidate via region around the midpoint. The planner doesn't pick the exact via spot. The executor searches that region for one.

On the executor side, connections are worked through in the planned order. It picks waypoints in the overlap between consecutive corridors, has KiCad's push-and-shove router draw the trace, and tries via positions inside the candidate region when changing layers. At the end it checks that the two clusters are actually electrically connected, not just that some routing step succeeded.

A failure doesn't go straight back to the language model. The executor first tries bounded local recovery: walking around obstacles, shoving traces aside, trying other waypoints, an A-star detour search, backing up a stage, or starting from a different anchor. Each kind of attempt has a cap.

If local recovery runs out, the executor packages the failure as structured feedback: what it tried, where it stopped, what might be blocking it, and how far it strayed from the plan. That connection is rolled back to a consistent checkpoint, and every other accepted connection is kept.

Only now does the language-model planner step in. It looks at the original images, the structured data, the current plan and this batch of feedback, and outputs a patch that changes only the corridors, via regions, endpoints or ordering that failed, leaving the rest alone. The patch is validated first, and if it fails, the model can be handed the errors and asked to correct it.

Each board gets at most twelve replans, and each connection at most four. The loop runs until everything is connected, nothing is left to schedule, or the budget runs out.

### Results

The benchmark is PCBWorld's, two hundred forty-seven boards in all. D2 is one hundred twenty-eight synthetic boards. D3-A is ninety-nine small open-source real boards, D3-B is ten medium boards, and D3-C is ten large boards with more than one hundred pads.

The main metric is clean pass. A board passes only if every connection is complete and KiCad reports zero error-level design-rule violations. KiCad does the grading, not the agent.

The planner runs on two models, Claude Opus 5.5 and Claude Haiku 5.5. The baselines include three rule-based routers such as Freerouting, PCBWorld's interactive agents on models like GPT-5.4, and reinforcement-learning policies trained with PPO and GRPO. Most of the baseline numbers are taken straight from PCBWorld's published results.

On the synthetic D2 set, in Table 1, both versions pass every board cleanly, tying Freerouting and KiCadRoutingTools. Against Freerouting, the Opus version uses four point two percent less wire and nineteen point seven percent fewer vias.

On the small real boards of D3-A, in Table 2, Opus reaches a ninety-four point nine percent clean pass rate. The strongest baseline, a PPO policy from PCBWorld, scores eighty-one percent, so that's a gain of thirteen point nine points.

On the medium D3-B boards, in Table 3, Opus passes all ten cleanly, against sixty-four percent for Freerouting. One caveat: this result includes retries after failed model responses, plus a trace-angle repair pass, so the paper says it isn't a strict first-attempt score.

The large D3-C boards are the hardest, in Table 4. Opus passes forty percent, Freerouting thirty, and the reinforcement-learning baselines zero. Average design-rule violations per board drop from Freerouting's thirty-two to six point three. But six of the ten boards are still unfinished.

A fairer comparison puts the same Haiku model inside both PCBWorld's interactive agent and PCBRouteAgent. Across the four sets, PCBWorld's clean pass rates are sixty-three point three, thirty-four point three, zero and zero percent. PCBRouteAgent's are one hundred, ninety-two point nine, eighty and ten.

It's much cheaper, too. Over all two hundred forty-seven boards, model calls fall from twenty-two thousand five hundred twenty-nine to one hundred seventy-two, API cost from twenty dollars seventy-two to four dollars seventy-seven, and cumulative runtime from thirty-six point four hours to two point six. The paper's explanation is that PCBWorld calls the model for every single action, while here the model only plans and the routing engine draws the traces.

The appendix has a head-to-head on one large board, again with Haiku. PCBWorld's agent uses all four hundred three of its steps and connects ten of eighty-six connections, while PCBRouteAgent calls the model five times and connects eighty-four.

### Comparison

So how much does the language-model planner actually contribute? Table 5 is the ablation: same executor, same scripted local recovery, with only the model's replanning switched off.

With it off, clean pass rates on the four sets are ninety-six point one, fifty-three point five, zero and zero percent. With Opus, they're one hundred, ninety-four point nine, one hundred and forty. On D3-B and D3-C, the version without the model still connects eighty-nine point one and eighty-three point one percent of the connections on average, yet not a single board passes cleanly.

The price is longer traces and more vias. On D3-A, for example, Opus raises average wirelength from one hundred fifty point eight to one hundred sixty-five point five millimeters. And on D3-B, the Opus version uses ten point nine percent more wire than Freerouting.

The authors are clear about what this ablation shows. It beats turning the model off, but it isn't compared with a dedicated rule-based replanner, or with a general-purpose agent using the same tools.

My read is that the interesting part here isn't the score, it's the division of labor. The model makes the board-level calls, a mature routing engine handles the geometry, and that beats having the model draw every segment itself, at a fraction of the cost. That pattern should carry over to agents working inside plenty of other professional tools.

My read is also that the comparisons deserve some care. Most baseline numbers come from the PCBWorld paper, and PCBWorld's agent wasn't fully evaluated with Opus, which the authors say was because its per-step model calls make the API cost much higher. And D3-B and D3-C have only ten boards each, so every single board is worth ten percentage points.

### Limitations

The authors use case studies to show two limitations.

First, early decisions can box in later ones. On one large board, the Opus version routed the sensor's power line first, without leaving pad access or room for vias for the SCL clock and SDA data lines. It finished seventy of eighty-three connections, while the original design connects all of them.

Second, connected isn't the same as electrically sound. In this figure, USB D-plus and D-minus are a differential pair, and the original design runs them side by side, each thirty point two five millimeters long. PCBRouteAgent connects all fifty-five connections on that board, but routes those two lines separately, at thirty-four point zero three and thirty-six point four nine millimeters, because nothing in the planner's input describes the pair relationship.

In their conclusion, the authors also acknowledge that finishing large boards, and keeping the routing compact, remain major challenges.

### Future work

Looking ahead, the authors plan better spatial and layer planning, recovery budgets that adapt, and routing refinement that cuts physical cost while keeping every connection intact.

From the failure cases, they also suggest looking further ahead when ordering and repairing: reserve pad access, handle connections with few alternatives first, and go back to revise earlier routes when needed. The code is already public on GitHub.

My read is that the title asks how close agents are to PCB routing engineers, and the paper answers honestly. On connectivity plus basic rule checks, small and medium boards are nearly solved. But real engineers also handle constraints tied to what a signal is, like differential pairs, and today's task definition and checker don't cover that yet.

That's today's Daily Frontier Papers. The link to the paper is in the description.

