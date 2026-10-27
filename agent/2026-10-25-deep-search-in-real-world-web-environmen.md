# Deep Search in Real-World Web Environments

**最强模型也只答对39%**

- 论文 Paper: [arXiv 2610.30181](https://arxiv.org/abs/2610.30181)
- 单位 Affiliation: 香港科技大学（广州）· 复旦 · 斯坦福 · 微软研究院等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-25
- 视频 Videos: [YouTube 中文](https://youtu.be/-QGuPiew7bM) · [YouTube English](https://youtu.be/MHW7qMdrXGM) · [B站](https://www.bilibili.com/video/BV1Kbnd6WE9e)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇香港科技大学（广州）为主，联合复旦、斯坦福、微软研究院等机构的论文，题目是 Deep Search in Real-World Web Environments，在真实的网页环境里做深度搜索。

他们做了一个新的 benchmark，叫 OpenDS-World。结果很有意思，12 个前沿模型里最强的 GPT-6 Astra，只答对了 39%，而人类独立去做，每次尝试的正确率只有 21.1%。

### 问题

先说背景。Deep Search，也就是深度搜索 Agent，要自己上网查资料、点网页、读文件，把好几处的证据串起来，最后回答一个问题。论文认为，它们在真实开放网页里做完整研究的能力，现在评测得还不够。

作者认为真实网页难在三点。第一是模态多，证据可能在文字里，也可能在图片、音频、视频里。第二是格式杂，HTML、PDF、XML、Parquet 数据文件，读法都不一样。第三是要交互，有些信息不是搜一下就能看到，得点击、填参数、操作界面，甚至把文件下载下来再处理。

这张图用 Our World in Data 的一个页面举例。同一个网页里有文字和图表，有 CSV、JSON、ZIP 几种下载格式，还要选地区、选年份、调数据接口，才能拿到想要的数字。

### 以前的方法

以前的评测，有的测持续查找事实和多跳关系，比如 BrowseComp，有的测多模态浏览，有的测网页导航和跨站操作。

论文认为，这些工作各自覆盖了一部分，但模态、格式、交互这三件事是连在一起的，现有的 benchmark 没有把它们放在一起充分地测。

### 核心想法

核心想法是，每道题都从一个真实视频出发。Agent 先要看懂视频的画面和声音，认出里面的一个东西，然后自己到开放网页上一步步找证据，最后给出一个唯一的、可以核对的答案。

看这个例子。视频里介绍了一种远古生物，Agent 要先认出它，再到博物馆官方的虚拟导览里找到对应位置，在附近的化石展柜里，挑出只有一件标本、尺寸最小的那种生物。然后找到 2024 年 2 月的一期播客，读它引用论文里的一张地层对比图，顺着线索找到 2018 年的一个公园视频，最后到联合国教科文组织的 Parquet 数据集里，查出这个地质公园的编号。

一条证据链，横跨视频、网页、图片、音频、PDF 和结构化数据，而证据在哪、怎么访问，题目都不会提前告诉你。

### 方法

题目是怎么造的，分三步。第一步，专家一共选了 500 个候选视频，起点的那个实体必须靠看画面、听声音才能认出来，不能直接从视频页面的文字里拿到，而且要能对应一个唯一的 Wikidata 条目。

第二步，标注员从当前节点出发，用 Google 搜索，挑一个关系明确、能核实的来源，抽出下一个实体，一步步往外扩展。每个节点都要有多个独立网页印证，直到走到一个唯一、稳定的终点，作为标准答案。

第三步叫问题反转。把整条证据链倒过来写成题目，中间的实体都不直接点名，只用间接描述。单看任何一条线索都找不到答案，必须把整条链还原出来。

然后是人工核验。十位独立审核员把每条证据链重走一遍，用其他独立来源交叉验证，有事实错误、证据不足、答案不唯一的题，直接删掉。最后留下 200 道题，覆盖文化、日常生活、科学、经济、历史、自然、社会、科技八个领域。

统计一下，这 200 道题的证据链上，一共标注了 787 个网页和在线资源，平均每题 3.94 个，覆盖 4 种模态、42 种文件格式、7 种访问方式。95.5% 的题，需要两种以上的访问方式。

这 7 种访问方式，包括直接看、点链接跳转、填参数改变页面状态、在音视频里按时间定位、操作图形界面、写结构化查询，还有把文件传过来再处理。

评测环境是统一的。作者搭了一个 Agent 框架，给模型三类工具，搜索文字、图片和视频的工具，在浏览器里点击、输入、滚动和截图的 GUI 工具，还有读视频帧和字幕的工具。

每道题限时一小时，所有工具加起来最多调用 100 次，到了上限就停，模型只能根据已经找到的证据给答案。答案对不对，交给 Qwen3.8-27B 当裁判，和标准答案比对语义是否一致。

### 实验和效果

先看人。作者把 200 道题分成 10 组，每组 20 道，每组由 5 个人独立做，一共 50 人。可以用普通浏览器和搜索，但不能用任何大模型工具，也不统一限时。

结果是，1000 次尝试只答对 211 次，正确率 21.1%。200 道题里有 85 道，5 个人全都没做出来。每次尝试平均花了 38.8 分钟，43% 的尝试被打了最高难度分。

作者按人的表现给题目分了难度。至少两个人做对的算低难度，54 道，只有一个人做对的算中等，61 道，没有人做对的算高难度，85 道。

再看模型，结果在表 1。GPT-6 Astra 最高，39%，Claude Opus 5.5 是 38%，Gemini 3.8 Flash 是 36.5%，Claude Fable 5.1 是 33.5%。GPT-5.6 Sol 是 28%，Qwen3.8-Max 是 25.5%，其余模型都在 25% 以下，有四个低于 20%。

也就是说，最强的模型 200 道题里也错了 122 道。难度越高掉得越厉害，12 个模型平均下来，低难度 39.66%，中等 23.5%，高难度只有 16.76%。有意思的是，高难度这一档最好的是 Gemini 3.8 Flash，28.24%。

第二个发现，花得多不等于做得好。GPT-6 Astra 和 GPT-5.6 Sol 用的 token 几乎一样多，正确率却差了 11 个百分点。Gemini 3.8 Flash 用的 token 大约是 GPT-6 Astra 的 5.22 倍，正确率反而更低。

这张图是成本。GPT-6 Astra 平均每题调用大约 51 次工具，用 18.66 分钟。Qwen3.8-Flash 平均每题调用将近 90 次，正确率只有 12%。

第三，工具用法差别很大。GPT-5.6 Sol 最偏爱搜索，74.5% 的调用是搜索。Claude Opus 5.5 和 Qwen3.8-Flash 有一半左右的调用是 GUI 操作，但正确率一个 38%，一个 12%。所以光看用了哪些工具，解释不了谁做得好。

第四是失败和预算。失败的工具调用，只占全部调用的 2.83% 到 9.42%，却吃掉了 19.27% 到 61.9% 的工具执行时间。Qwen3.8-Flash 有 80% 的题，是因为调用次数用完才停下的。

反过来，GPT-6 Astra 和 Claude Opus 5.5 碰到预算上限的比例相对低，Claude Opus 5.5 只有 16.5% 的题碰到上限，但正确率还是不到 40%。作者认为，问题不只是效率，还在于选对信息源、整合证据，以及判断什么时候该停。

还有一个细节。平均每道题的标准证据链只有 3.94 个来源，但各个模型平均每条轨迹要调用 42.8 到 89.9 次工具。答案的路径不长，找到它的过程却很长。

### 和以前方法的对比

和以前的 benchmark 相比，论文强调的是把三件事放在了一起。一是起点是视频，必须看懂画面和声音。二是证据分散在 42 种格式里，不只是网页文字。三是把交互访问算进了证据链，比如虚拟导览、数据接口和下载文件。

要说明的是，这篇没有提出新的 Agent 方法，也没有和以前的 benchmark 做数字对比，贡献在任务定义、数据和行为分析。

我的判断是，模型平均正确率比人高，不能直接说成 AI 比人更会查资料。人不能用 AI 工具，答案人工核对，难度分档也是按人的成绩定的，模型则是统一工具、一小时预算加模型裁判，条件不一样。这个对比更多说明的是，这些题确实难。

我的判断是，这篇最有用的不是排行榜，而是那几条行为分析。工具失败吃掉时间、预算用完还没有答案、该停的时候不停，这些都是做 Deep Search 产品的人可以直接拿来改的地方。

### 局限

论文没有单独写局限这一节，下面几点是从它的设置里能看出来的。

第一，网页是活的。为了可复现，作者把起点视频存档，随题目一起发布，但后面的网页、数据集和播客都在真实网络上，作者自己也说真实网页是动态的，过一段时间再测，条件可能已经变了。

第二，判分用的是 Qwen3.8-27B 当裁判，论文没有报告它和人工判分的一致率。

第三，题量不大，200 道，拆到细分类别就更少，比如需要结构化查询的题只有 13 道，这一类的数字波动会很大。

第四，所有模型都在作者的同一个框架、同一套提示词和预算下测，测的是这个设置下的模型能力，不代表各家自己的深度研究产品。

### 未来方向

作者在讨论部分说，OpenDS-World 可以当作研究递归自我改进的试验场，让 Agent 从自己失败的轨迹里总结规律，改进搜索策略和出错后的恢复，再到新题上检验。

论文首页附了代码和数据集的入口。

我的判断是，Deep Search 接下来要比的，可能不是谁搜得多，而是谁更会选来源、更会从失败里恢复、更知道什么时候该停。这篇论文给了一套能量化这些行为的工具。

我的判断是，也要看到，最好的模型也只有 39%，高难度题最好也不到三成，离真正靠得住还很远。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by HKUST in Guangzhou, with co-authors from Fudan, Stanford, Microsoft Research and others. It's called Deep Search in Real-World Web Environments.

They built a new benchmark called OpenDS-World, and the headline result is striking. The best of twelve frontier models, GPT-6 Astra, gets thirty-nine percent right. People working on their own get twenty-one point one percent per attempt.

### Problem

Some background. A deep search agent goes out on the web by itself, searches, clicks through pages, reads files, and stitches evidence from several places into one answer. The paper's argument is that we haven't properly measured how well these agents do that end to end on the real, open web.

The authors say the real web is hard in three ways. First, modalities: the evidence might be in text, but also in images, audio or video. Second, formats: HTML, PDF, XML, Parquet data files, each read differently. Third, interaction: some information doesn't show up in a search result at all. You have to click, set parameters, work an interface, or download a file and process it.

This figure shows all three on a single Our World in Data page. There's text and a chart, downloads in CSV, JSON and ZIP, and to get the number you want you may have to pick a region, find a year, or query the data API.

### Prior work

Earlier benchmarks each cover a piece of this. Some test persistent fact finding and multi-hop relations, like BrowseComp. Others test multimodal browsing, or web navigation and cross-site actions.

The paper's point is that modality, format and interaction are tangled together on the real web, and existing benchmarks haven't covered them together.

### Key idea

The key idea is that every task starts from a real video. The agent first has to understand what it sees and hears and identify something in it, then search the open web step by step, and end with one unique answer that can be checked.

Here's an example. The video introduces a prehistoric animal. The agent has to recognize it, use it as a clue to find the right spot in a museum's official virtual tour, and in a nearby arthropod fossil case, pick the organism that has only one specimen and the smallest one. Then it finds a podcast episode from February twenty twenty-four, reads a stratigraphic chart in the paper that episode cites, follows the clue to a twenty eighteen video about a park, and finally looks up that geopark's identifier in UNESCO's Parquet dataset.

One chain of evidence runs through video, web pages, images, audio, a PDF and structured data, and the task never tells you where the evidence is or how to get to it.

### Method

How were the tasks built? In three stages. First, experts select five hundred candidate videos. The starting entity has to be identifiable only by watching and listening, not from the text on the video's page, and it has to map to one unique Wikidata entry.

Second, annotators start from the current node, search with Google, pick a related, verifiable source, and pull out the next entity, expanding the chain one step at a time. Every node has to be corroborated by multiple independent web sources, until the chain reaches a unique, stable endpoint, which becomes the answer.

Third, what they call question inversion. The whole chain is turned around into a question. The intermediate entities are never named, only described indirectly, so no single clue gives away the answer. You have to rebuild the chain.

Then human verification. Ten independent reviewers walk every chain again and cross-check it against other sources. Any task with a factual error, thin evidence or more than one possible answer is dropped. What's left is two hundred tasks across eight domains: culture, daily life, science, economy, history, nature, society and technology.

Some statistics. Across those two hundred tasks, the evidence chains contain seven hundred eighty-seven annotated web pages and other online resources, three point nine four per task on average, covering four modalities, forty-two file formats and seven access types. Ninety-five point five percent of tasks need at least two kinds of access.

The seven access types are: just looking, following links, setting parameters to change a page, seeking to a time in audio or video, manipulating a graphical interface, writing a structured query, and transferring a file somewhere it can be processed.

The test harness is the same for everyone. The authors built one agent framework with three kinds of tools: search, for text, images and video; GUI tools to navigate, click, type, scroll and take screenshots in a browser; and tools to read video frames and subtitles.

Each task gets one hour, and at most one hundred tool calls in total. Hit either limit and the search stops, and the model has to answer from the evidence it already has. Answers are graded by Qwen3.8-27B as a judge, checking semantic agreement with the reference answer.

### Results

Humans first. The two hundred tasks were split into ten sets of twenty, and each set was done independently by five people, fifty in all. They could use an ordinary browser and search, but no language-model tools, and there was no fixed time limit.

Out of one thousand attempts, two hundred eleven were correct. That's twenty-one point one percent. Eighty-five of the two hundred tasks were solved by none of their five solvers. The average attempt took thirty-eight point eight minutes, and forty-three percent of attempts got the top difficulty rating.

The authors used those human results to set difficulty. Low means at least two people solved it, fifty-four tasks. Medium means exactly one, sixty-one tasks. High means nobody did, eighty-five tasks.

Now the models, in Table 1. GPT-6 Astra leads at thirty-nine percent, then Claude Opus 5.5 at thirty-eight, Gemini 3.8 Flash at thirty-six point five, and Claude Fable 5.1 at thirty-three point five. GPT-5.6 Sol gets twenty-eight and Qwen3.8-Max twenty-five point five. Everything else is under twenty-five percent, and four models are under twenty.

So even the best model gets a hundred twenty-two of the two hundred tasks wrong. And accuracy falls with difficulty. Averaged over the twelve models, it's thirty-nine point six six percent on Low, twenty-three point five on Medium, and sixteen point seven six on High. Interestingly, the best model on the High tier is Gemini 3.8 Flash, at twenty-eight point two four.

Second finding: spending more doesn't mean doing better. GPT-6 Astra and GPT-5.6 Sol use almost the same number of tokens, yet their accuracy differs by eleven percentage points. Gemini 3.8 Flash uses about five point two times as many tokens as GPT-6 Astra and scores lower.

This figure shows the cost. GPT-6 Astra averages about fifty-one tool calls and eighteen point seven minutes per task. Qwen3.8-Flash averages almost ninety calls per task, at twelve percent accuracy.

Third, tool use differs a lot. GPT-5.6 Sol leans hardest on search: seventy-four point five percent of its calls. Claude Opus 5.5 and Qwen3.8-Flash spend about half their calls on GUI actions, yet one scores thirty-eight percent and the other twelve. The tool mix alone doesn't explain who does well.

Fourth, failures and budgets. Failed tool calls are only two point eight to nine point four percent of all calls, but they eat nineteen to sixty-two percent of tool execution time. And Qwen3.8-Flash stops because it ran out of calls on eighty percent of tasks.

On the other side, GPT-6 Astra and Claude Opus 5.5 hit a budget limit relatively seldom, and Claude Opus 5.5 hits one on just sixteen point five percent of tasks, yet it still stays under forty percent accuracy. The authors conclude that the problem isn't only efficiency, but choosing the right sources, putting evidence together, and knowing when to stop.

One more detail. The reference evidence chain averages just three point nine four sources per task, but the models average between forty-two point eight and eighty-nine point nine tool calls per trajectory. The path to the answer is short. Finding it is long.

### Comparison

Compared with earlier benchmarks, the paper's point is that it puts three things together. The starting point is a video, so the agent has to understand images and sound. The evidence is spread across forty-two formats, not just web text. And interactive access, like virtual tours, data APIs and downloads, is part of the evidence chain.

To be clear, this paper doesn't propose a new agent method, and it doesn't put older benchmarks side by side in a results table. Its contribution is the task definition, the data, and the behavioral analysis.

My read is that the models beating the human average doesn't mean AI is better than people at research. The humans had no AI tools and their answers were checked by hand, the difficulty tiers come from their own results, and the models ran with fixed tools, a one-hour budget and a model judge. The conditions differ. What the comparison mostly tells you is that these tasks are genuinely hard.

My read is that the most useful part isn't the leaderboard, it's the behavior analysis. Failed tool calls eating the clock, budgets running out before an answer, not knowing when to stop: those are things anyone building a deep search product can go fix.

### Limitations

The paper has no separate limitations section, so these are points that follow from its setup.

First, the web is live. For reproducibility the authors archive the starting videos and ship them with the tasks, but the pages, datasets and podcasts further down the chain are on the real web, and the authors themselves describe it as dynamic. Run the same task later and the conditions may have changed.

Second, grading relies on Qwen3.8-27B as the judge, and the paper doesn't report how often it agrees with human graders.

Third, two hundred tasks is not many, and sub-categories get small. Only thirteen tasks need a structured query, so numbers in that category can swing a lot.

Fourth, every model runs in the authors' single harness, with one prompt and one budget. This measures the models in that setup, not each company's own deep research product.

### Future work

In the discussion, the authors pitch OpenDS-World as a testbed for recursive self-improvement: agents that mine their own failed trajectories for patterns, improve their search strategy, tool choice and error recovery, and then check those changes on new tasks.

The first page of the paper links to the code and the dataset.

My read is that the next contest in deep search may not be who searches the most, but who picks sources better, recovers from failure better, and knows when to stop. This paper gives a way to measure exactly those behaviors.

My read is also that the ceiling is low. The best model is at thirty-nine percent, and nobody reaches thirty percent on the hardest tier. Reliable is still a long way off.

That's today's Daily Frontier Papers. The paper is linked in the description.

