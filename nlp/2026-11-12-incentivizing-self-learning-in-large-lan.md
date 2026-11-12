# Incentivizing Self-Learning in Large Language Models via Reinforcement Learning

**教大模型边做题边翻书**

- 论文 Paper: [arXiv 2610.39671](https://arxiv.org/abs/2610.39671)
- 单位 Affiliation: 上海人工智能实验室等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-12
- 视频 Videos: [YouTube 中文](https://youtu.be/X40NUEV1KxE) · [YouTube English](https://youtu.be/o6_yCczt0SQ) · [B站](https://www.bilibili.com/video/BV1q3Wd61Enm)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇上海人工智能实验室等单位的论文，题目是，用强化学习激励大语言模型自主学习。

简单说，它让模型像学生一样，一边做题一边翻教科书，然后用强化学习，把会翻书这件事练出来。

论文报告的主要结果是，一个 35B 的模型，在数学证明题上的总体准确率，从百分之 51.5 提到了百分之 79.5。

### 问题

先说问题。现在的大模型回答知识类问题，一靠预训练记进参数里的知识，二靠 RAG，也就是检索增强生成，先搜出资料，再交给模型。

但作者认为，遇到不熟悉的领域，光拿到资料是不够的。模型得真的读懂新的原理，判断它能不能用，再把它用进自己的推导里。作者把这种能力叫做 self-learning，也就是自主学习。

难点在于，学和做是交织在一起的。推到一半，可能才发现缺一个前置知识，读到一个新定理，又可能整个改掉解题思路。

### 以前的方法

以前相关的工作大概有三类。第一类是用强化学习训练的检索 Agent，边搜边推理，最后给出答案。

第二类是上下文学习，研究模型能不能从事先给好的材料里学到新知识。第三类是元强化学习和测试时适应，靠环境的观察和行动的结果来调整。

作者认为，怎么通过研读长篇资料来培养这种能力，研究还很少。在他们的设定里，没有拼好的上下文，也没有试错的反馈，模型面对一本静态的书，要自己决定学什么，学完怎么用。

### 核心想法

作者的出发点，是模仿人怎么学数学。拿到一道题，你会翻开教科书，看目录，找到相关章节，读定义和定理，然后自己写证明。

所以他们设计了这样一个任务。每道证明题配一本结构化的教科书，里面有定义、定理和一般结论的证明，但这道习题的答案被拿掉了，模型必须自己把分散在不同章节的概念连起来。

这张图是整体框架。左边是教科书，中间的模型在学习和推理之间来回切换，最后交出证明，一个评判模型拿着模型看不到的参考答案打分，分数就是强化学习的奖励。

作者的目标不是让模型背下训练用的那几本书，而是练出一种能迁移的学习策略，换一本没见过的书，也能学得会。

### 方法

先看数据。作者把 158 本本科和研究生水平的数学教材，解析成按章节索引的文本，把习题和正文分开，删掉习题答案，一共得到 5183 道证明题。

每本书平均有 56 个章节，大约 20 万 8 千个 token，章节之间的依赖关系很复杂，是一个相当长的学习环境。

模型和书之间只有一个接口，叫 BookNavigator，它提供三个动作。一个看整本书的目录，一个按关键词搜索，返回相关章节和片段，还有一个读某一节的全文。

每个动作返回的结果，都会接到对话历史里，模型可以一边推理一边翻书，根据已经读到的东西决定下一步。在这个设置下，最多交互八轮，整条轨迹不超过 64k 个 token。

接下来是训练。所有强化学习实验都从 Intern-S2-Preview-35B 出发，用 GRPO 训练 50 步。GRPO 的做法是，同一道题采样八条回答，按组内的相对好坏来更新模型。

奖励的核心是证明对不对。评判用的是 GLM-5.2，它对照参考答案独立审八次，八次都判对才算对，否则就算错，这是一种很严格的悲观判定。

但只看最后的对错，有两个问题。一条成功的轨迹里，可能夹着很多没用的翻书，而模型做题越来越强以后，也会越来越不想翻书。所以作者加了两个设计。

第一个是探索奖励。证明做对了，而且至少成功调用过一次工具，就额外加分。这个加分的权重，按余弦曲线从 1 降到 0.25，前期多鼓励翻书，后期还保留一点激励。

第二个是按轮次给工具打分。一个评判模型逐轮去看，这一轮翻书有没有用，标成有用、中性或者有害。它会参考下一轮的推理，看读到的东西有没有用上，但它看不到参考答案，也不知道最后做没做对。

有意思的是，这个评判模型，就是训练开始前那个模型的冻结副本，不需要更强的模型。有害的轮次记 0.5 分，有用和中性的记 1 分。

然后，把整条轨迹的优势值，也就是它比同组平均好多少，按每一轮的分数重新分配。有害的轮次分得少一些，但加权平均不变，所以最终对错仍然是主要信号。

### 实验和效果

实验怎么设。训练集是 132 本书里的 4557 道题。测试分三档，训练用过的书里留出来的题，没见过的书但学科见过，还有训练里完全没有的学科，也就是概率和统计。

对照组有四个，不给书，按关键词检索最多八个章节一次性贴进去，按原书顺序贴满 64k 个 token，以及直接贴上题目出处那一节。它们都只生成一次回答，中间不能翻书。

先看表 1 的核心结果。训练之前，让模型自己翻书，总体准确率只有百分之 51.5，比不给书的百分之 55.7 还低，也低于所有固定检索的对照组。论文的说法是，一开始这种自主学习的方式基本是无效的。

强化学习之后，情况反转了。自主学习的总体准确率到了百分之 79.5，提升了 28 个百分点，在五种做法里，分数最高，涨得也最多。

同样经过强化学习，表现最好的对照组，是检索八个章节的那个，百分之 73.1，自主学习比它高 6.4 个点，比不给书高 11 个点。

换成没见过的书和没见过的学科，自主学习也是最高，分别是百分之 85.2 和 86.7，比最强的对照组高 4.2 和 5.4 个点。

那提升到底从哪来，是模型本身推理变强了，还是背下了书？看表 2。训练后的模型不给书，也从百分之 55.7 涨到了 71.7，说明它自己的解题能力确实变强了。但有书还是比没书高 7.8 个点，说明翻书仍然有用。

换一本同学科的别的书，是百分之 75.2，把题目出处那一节，连同前后各四节一起删掉，是 74.6，都比不给书高。作者认为，这和一种更稳健的学习策略是吻合的，而不是只依赖那本熟悉的书。

再看跨任务。训练只用了数学证明，作者又测了两个完全不同的基准。BrowseComp-Plus 考多轮信息检索，CL-bench 考从给定的上下文里学新知识。而且测的时候不用 BookNavigator，用的是各个基准原本的设置。

结果在表 3。用自主学习训练的模型，BrowseComp-Plus 从 50.2 到 51.9，CL-bench 从 14.8 到 17.1，分别涨了 1.7 和 2.3 个点，两项都是所有模型里最高的。而用固定检索方式训练的模型，在 BrowseComp-Plus 上全都低于训练前。

不过提升不是全面的。CL-bench 里考严格按步骤办事的流程任务执行，所有经过强化学习的模型都比训练前差。作者的一种解释是，证明题训练奖励的是有效推导，和照章办事不是一回事。

消融实验在表 4，这张表很值得看。只用对错当奖励，总体准确率也能到百分之 74.9，但带工具调用的轨迹，从百分之 36.9 掉到了 6.7，平均每条轨迹只调用 0.3 次。模型等于学会了不翻书。

只加工具打分，这个趋势扭转不了，只有百分之 7 的轨迹会用工具，不过在训练用过的书上，准确率涨了 3.3 个点。

加上探索奖励，用工具的轨迹回到了百分之 79.9，在没见过的书和学科上，分别多涨 2.0 和 2.9 个点。两个设计都加上，三个测试集全部最高，总体百分之 79.5，平均每条轨迹调用 3 次工具，反而比只加探索奖励时少。

### 和以前方法的对比

和以前的检索 Agent 比，比如 Search-R1 和 R1-Searcher，它们用最终结果当奖励来学怎么搜索。这篇的不同在于，习题和答案已经从书里删掉了，模型搜不到答案，只能去找定义和定理，自己拿来用。作者说，区别不在检索还是推理，而在于让找知识和推理互相配合。

和 Reflexion、SEAL 这类方法比，它们要么靠多次尝试的反馈，要么在测试时更新权重。这篇是在一道题之内，只靠静态的参考资料来学，不靠多次尝试得到的对错反馈，测试时也不改权重。

我的判断是，这篇最值得记住的，是表 4 那个现象。只按对错训练，模型会越来越不翻书，分数照样往上涨。如果你在训练会用工具的 Agent，这是一个很现实的提醒，最终分数好看，不代表它学会了用工具。

我的判断是，也要看到弱点。跨任务的提升只有 1.7 和 2.3 个点，幅度不大。所有强化学习实验都从同一个基础模型出发，换别的模型结论还成不成立，论文没有测。

### 局限

论文没有单独写局限这一节，下面几点，是从实验设置和结果里能直接看到的。

第一，对错完全靠 GLM-5.2 对照参考答案来判，虽然要求八次都判对，但论文里没有看到这个评判和人工评分的一致性检验。

第二，只用了一个基础模型，也就是 Intern-S2-Preview-35B，强化学习也只训练了 50 步。

第三，训练只有数学证明，跨任务有涨有跌，作者自己也说，迁移的收益是有选择性的。

### 未来方向

作者在结论里说，希望这项工作能打开一条路，把教科书这类长篇语料当成自监督的来源，通过自主学习，提升模型在复杂、长上下文任务上的通用解题能力。

我的判断是，下一步最值得看的，是把这套方法从数学教科书搬到技术文档、代码库、法规这些地方。那里同样是大量的长篇资料，也同样需要模型边查边做。

我的判断是，读这篇的数字要分开看。28 个点的提升里，有一大块来自模型本身解题变强，真正能归功于翻书的，是有书和没书之间那 7.8 个点。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Shanghai AI Laboratory, with co-authors at Shanghai Jiao Tong, CUHK, USTC and Fudan, and it's called Incentivizing Self-Learning in Large Language Models via Reinforcement Learning.

The short version: the model works like a student, solving a problem while flipping through a textbook, and reinforcement learning is used to train the habit of looking things up well.

The headline result, as reported in the paper: a thirty-five billion parameter model goes from fifty-one point five percent to seventy-nine point five percent overall accuracy on math proofs.

### Problem

Start with the problem. Language models answer knowledge-heavy questions with two things: what they absorbed into their weights during pre-training, and retrieval-augmented generation, where relevant text is fetched and handed to the model.

The authors argue that access to information isn't enough in an unfamiliar domain. The model has to understand a new principle, decide whether it applies, and fold it into its own derivation. They call this ability self-learning.

What makes it hard is that studying and solving are tangled together. Halfway through a derivation you may discover a missing prerequisite, and a newly read theorem may change your whole approach.

### Prior work

Related work falls into roughly three groups. First, retrieval agents trained with reinforcement learning, which interleave search and reasoning on the way to an answer.

Second, context-learning studies, which ask whether a model can pick up new knowledge from material someone has already supplied. Third, meta-RL and test-time adaptation, which learn from observations and the outcomes of actions.

According to the authors, how to build this ability through sustained study of long reference material is still underexplored. In their setting there's no pre-assembled context and no feedback from repeated attempts: the model faces a static book and has to decide on its own what to study and how to use it.

### Key idea

The starting point is how people study math. You get a problem, open the textbook, scan the table of contents, find the relevant section, read the definitions and theorems, and then write your own proof.

So they built exactly that task. Each proof problem is paired with a structured textbook containing definitions, theorems and proofs of general results, but the solution to that exercise is removed, so the model has to connect concepts spread across different sections by itself.

This figure is the overall setup. The textbook is on the left, the model in the middle switches between studying and reasoning before handing in a proof, and a judge scores that proof against a reference solution the model never sees, which becomes the reinforcement learning reward.

The goal isn't for the model to memorize the training books. It's to learn a study strategy that transfers, so it can learn from a textbook it has never seen.

### Method

First, the data. The authors parsed one hundred fifty-eight undergraduate and graduate math textbooks into chapter-indexed text, separated the exercises from the exposition, stripped out the solutions, and ended up with five thousand one hundred eighty-three proof problems.

On average each book has fifty-six sections and about two hundred eight thousand tokens, with complicated dependencies between sections, so it's a genuinely long study environment.

The only link between the model and the book is an interface called BookNavigator, with three actions. One returns the full table of contents, one runs a keyword search and returns matching sections with snippets, and one reads an entire section.

Every result is appended to the conversation history, so the model can reason and read in turns, choosing its next step based on what it has already seen. In this setting it gets at most eight rounds of interaction, and the whole trajectory is capped at sixty-four thousand tokens.

Now training. Every reinforcement learning run starts from Intern-S2-Preview-35B and trains with GRPO for fifty rollout steps. GRPO is a reinforcement learning method that samples a group of answers for the same problem, eight here, and updates the model based on how each one compares with the rest of its group.

The core reward is whether the proof is correct. The judge is GLM-5.2, which checks the proof against the reference solution in eight independent reviews, and the proof only counts as correct if all eight say so, a deliberately pessimistic standard.

Rewarding only the final answer has two problems, though. A successful trajectory can contain plenty of pointless lookups, and as the model gets better at solving, it has less and less reason to open the book, so the authors added two pieces.

The first is an exploration bonus. If the proof is correct and the model made at least one successful tool call, it gets extra reward. The weight of that bonus follows a cosine schedule from one down to zero point two five, pushing hard on exploration early and keeping a small incentive later.

The second is round-level tool credit. A judge model looks at each round and labels the lookup as useful, neutral or harmful. It uses the next round's reasoning to see whether what was read actually got used, but it never sees the reference solution or whether the final proof was correct.

Here's the interesting part: that tool judge is just a frozen copy of the starting model, not a stronger one. Harmful rounds score zero point five, and useful or neutral rounds score one.

Then the trajectory's advantage, meaning roughly how much better it did than the group average, is redistributed across rounds according to those scores. Harmful rounds get a smaller share, but the token-weighted average stays the same, so final correctness remains the main signal.

### Results

Here's the setup. Training uses four thousand five hundred fifty-seven problems from one hundred thirty-two books. Testing has three tiers: held-out problems from books seen in training, unseen books in familiar subjects, and a subject never trained on at all, probability and statistics.

There are four comparison setups. One gets no book at all. One retrieves up to eight sections by keyword search and pastes them in up front. One pastes sections in the book's original order, up to sixty-four thousand tokens. And one pastes in the exact section the problem came from. All of these produce a single answer with no lookups along the way.

Table one has the main result. Before training, letting the model use the book gives only fifty-one point five percent overall, below the no-book setup at fifty-five point seven and below every fixed-retrieval setup. In the paper's words, the self-learning protocol starts out ineffective.

After reinforcement learning, that flips. Self-learning reaches seventy-nine point five percent overall, a gain of twenty-eight points, the highest score and the biggest improvement of the five setups.

With the same training, the strongest comparison is the eight-section retrieval setup at seventy-three point one. Self-learning beats it by six point four points, and beats no-book by eleven.

On unseen books and on the unseen subject, self-learning is also on top, at eighty-five point two and eighty-six point seven percent, which is four point two and five point four points above the strongest comparison.

So where does the gain come from: better reasoning on its own, or memorized books? Table two answers that. Without any book, the trained model still rises from fifty-five point seven to seventy-one point seven percent, so its own problem solving clearly improved. But having the book is still worth seven point eight points, so reading still matters.

Swap in a different book on the same subject and it scores seventy-five point two; delete the source section along with the four sections on each side, and it scores seventy-four point six, both above the no-book score. The authors say this fits a more robust study strategy rather than reliance on one familiar book.

Next, other tasks. Training only ever used math proofs, but the authors also tested two very different benchmarks. BrowseComp-Plus measures multi-step information retrieval, and CL-bench measures learning new knowledge from supplied context, and both were run with their own original setups, not with BookNavigator.

Table three has the numbers. The self-learning model goes from fifty point two to fifty-one point nine on BrowseComp-Plus and from fourteen point eight to seventeen point one on CL-bench, gains of one point seven and two point three, the best of all the models on both. Meanwhile every model trained with fixed retrieval ends up below the starting model on BrowseComp-Plus.

The gains aren't across the board, though. One CL-bench category, procedural task execution, tests following prescribed steps and constraints exactly, and every reinforcement-learning-trained model does worse there than the starting model. One explanation the authors offer is that proof training rewards valid derivations, which is a different skill from following procedures faithfully.

The ablation in table four is the one worth slowing down for. Rewarding correctness alone still reaches seventy-four point nine percent overall, but the share of trajectories that use any tool drops from thirty-six point nine percent to six point seven, with only zero point three calls per trajectory. The model effectively learned to stop opening the book.

Adding tool credit alone doesn't reverse that, with only seven percent of trajectories using tools, although accuracy on the training books goes up three point three points.

Adding the exploration bonus brings tool-using trajectories back to seventy-nine point nine percent and adds two point zero and two point nine points on unseen books and the unseen subject. With both pieces together, all three test splits are at their best, seventy-nine point five overall, with three tool calls per trajectory, fewer than with the bonus alone.

### Comparison

Compared with earlier retrieval agents like Search-R1 and R1-Searcher, which learn to search from outcome rewards, the difference is in the material. Here the exercises and their solutions are removed from the book, so the model can't look up the answer and has to find and apply definitions and theorems instead. As the authors put it, the distinction isn't retrieval versus reasoning, it's coordinating the search for knowledge with a reasoning process that's still evolving.

Compared with methods like Reflexion or SEAL, which rely on feedback from repeated attempts or on updating weights at test time, this work learns within a single problem, from static reference material, with no correctness feedback from repeated attempts and no test-time weight updates.

My read is that the finding to remember is that table four pattern. Train on correctness alone and the model stops consulting the book while its score keeps climbing. If you're training tool-using agents, that's a practical warning: a good final score doesn't mean the agent learned to use its tools.

My read is that the weaknesses matter too. The cross-task gains are only one point seven and two point three points. And every reinforcement learning run starts from the same base model, so whether the results hold for other models isn't tested in the paper.

### Limitations

The paper has no separate limitations section, so these are points you can see directly from the setup and results.

First, correctness rests entirely on a model judge. GLM-5.2 grades each proof against the reference solution, and although it uses the strict all-eight-reviews rule, the paper doesn't report a check of that judge against human grading.

Second, there's only one base model, Intern-S2-Preview-35B, and reinforcement learning ran for just fifty steps.

Third, training used math proofs only, transfer to other tasks is mixed, and the authors themselves describe the transfer gains as selective.

### Future work

In their conclusion, the authors say they hope this opens a path to using long-form corpora like textbooks as a source of self-supervision, strengthening general problem solving on complex, long-context tasks through self-learning.

My read is that the next thing to watch is moving this from math textbooks to technical documentation, codebases or regulations. Those are also huge bodies of long text where a model has to look things up while it works.

My read is that the numbers in this paper need separating. A big part of the twenty-eight-point gain comes from the model simply getting better at solving, and the part you can credit to reading the book is the seven point eight point gap between book and no book.

That's today's Daily Frontier Papers. The paper is linked in the description.

