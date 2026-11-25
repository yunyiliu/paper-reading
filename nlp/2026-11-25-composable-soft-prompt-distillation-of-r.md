# Composable Soft-Prompt Distillation of RL Experts: Toward Continual Post-Training

**模型能力像插件一样装**

- 论文 Paper: [arXiv 2611.27376](https://arxiv.org/abs/2611.27376)
- 单位 Affiliation: 加州大学圣塔芭芭拉分校 · 卡内基梅隆大学 · 英伟达 · 苹果 等
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-25
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1BNy6zTEMA)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇加州大学圣塔芭芭拉分校等机构的论文，合作方还有卡内基梅隆大学、英伟达、苹果等。论文叫 Composable Soft-Prompt Distillation of RL Experts，简称 CSPD，可以理解成，可组合的软提示蒸馏。

它要回答的问题是，大模型做完强化学习，得到一堆各有所长的专家，能不能不动模型本身，把这些专家像插件一样，随时装上，随时卸下。

### 问题

先说背景。现在大模型的强化学习后训练，也就是 RL post-training，通常是一个领域一个领域地做，因为每个领域有自己的数据，自己的判分器，自己的环境。

常见做法是，从同一个基础模型出发，分别训出数学专家、代码专家、科学专家，最后再把它们合成一个模型。

麻烦在于，新领域和更强的专家会不断出现，所以合并不是做一次就完，而是一个持续的过程。作者理想中的样子是，装上新专家，模型就获得它的本事，原来会的不受影响，更新成本也只和这个新专家有关。

### 以前的方法

以前的方法大致分两类。第一类，是把所有专家合进同一套权重。比如多任务联合强化学习，每加一个领域，所有领域都要重新跑一遍。顺序强化学习只训新领域，但前面学到的东西，可能被后面的训练破坏。

权重合并，是直接对参数做加减，不同专家的更新会互相干扰。多老师在线蒸馏，简称 MOPD，每加一个专家，都要让学生对着所有老师重训。而且这些方法都会改写基础模型的权重，可能让目标领域之外的能力变差。

第二类，是专家分开放，每个请求挑一个来用。路由器很难训，同一个请求里没法组合几个专家，每个专家还要部署一整个模型。按请求合并 LoRA 专家能避开这些问题，但每一步解码，每一层，都要多算一遍 adapter。

### 核心想法

作者注意到，这两类方法都把专家放在模型参数里。CSPD 换了个位置，把每个专家变成模型输入的一部分。

具体的载体叫软提示，soft prompt。它不是人写的文字，而是一串可以训练的向量，直接放在模型的输入里，这篇论文里，每个专家用 256 个向量。

这张图是整个流程。每个专家单独蒸馏成一个软提示，攒成一个提示库，用的时候把选中的几个直接拼起来。不同请求带不同的提示，也能放在同一个批次里一起服务。

### 方法

训练一个专家的软提示，用的是序列级蒸馏。先让强化学习之后的模型，也就是老师，回答这个领域的题，只留下判分器认为正确的回答。

然后把强化学习之前的模型当学生，完全冻结，只训练插在系统消息里的那 256 个向量，让学生带着这段软提示时，尽量生成老师那样的回答。损失函数就是这个式子，对回答里每个 token 求平均的负对数似然。

组合的时候更简单，把几个专家的软提示首尾拼接，放进同一个模板，不做任何联合训练。加专家，就训一个新提示放进去，删专家，就把它的提示拿掉，基础模型从头到尾不变。

还有一个细节。每个提示是单独训练的，但用的时候，前面可能接着别的提示。所以在一半的训练样本里，软提示前面会塞一段 128 到 256 个 token 的前缀，取自同领域的其他题目，不算进损失，推理时也不用。作者把这叫做前缀鲁棒训练。

### 实验和效果

先看单个专家能不能装进一个软提示。作者测了 23 个设置，用的都是公开的强化学习前后模型对，覆盖数学、代码，以及 SWE-bench Verified 和 Terminal-Bench 2.1 上的多步 Agent 任务，模型从 7B 到 35B 都有。

这里看的指标叫恢复率，就是老师比学生多出来的那部分分数，加了软提示的学生找回了多少。论文报告，23 个设置里有 22 个恢复了至少百分之九十，有 15 个追平或超过了老师。

举个例子，DeepSWE 这一组，学生在 SWE-bench Verified 上从 24.90 提到 45.65，老师是 44.05。最低的一组是 MiMo-7B 的代码，恢复率 88.08，因为它的强化学习数据没有公开，只能换别的代码题来蒸馏。

再看多个专家的组合。作者沿用 M2RL 的设置，学生是在 Qwen3-4B-Base 上做过监督微调的模型，老师是从它出发，分别训出的数学、代码、科学、指令遵循四个专家，对应的评测是 AIME24、LiveCodeBench、GPQA-Diamond 和 IFBench。

结果在表 2。四个软提示拼在一起，四项平均分从学生的 50.41 提到 67.83，和 MOPD 的 67.79 基本持平，也高于两种权重合并方法，以及用同样数据蒸馏出来、再合并的 LoRA。

它还高于一个理想化的参照，就是每个基准都交给对应领域的专家来做，平均是 66.61。不过分项看，CSPD 在 GPQA 和 IFBench 上比 MOPD 好，在 AIME24 和 LiveCodeBench 上不如 MOPD。

如果去掉前缀鲁棒训练，同样四个提示拼起来，平均分只有 64.50，作者说，这个小设计让组合更稳定。

接着模拟持续后训练。专家按指令遵循、科学、代码、数学的顺序，一个一个加进来，平均分依次是 60.99、61.67、63.67、67.83，每加一个都在涨，之前的提示一个都没有重训。

但单个领域不总是变好。加代码的时候，IFBench 从 66.71 掉到 64.58，再加上数学，又升到 68.42。反过来每次拿掉一个提示，拿掉数学时平均分掉得最多，4.16 分，其中 GPQA 掉了 6.54 分，说明数学专家也帮到了科学题。

再看领域外。作者在六个训练之外的基准上测了这套组合，从拒答、真实性，到工具调用和金融推理。

表 3 里，CSPD 的六项平均是 56.6，学生是 50.3，MOPD 是 53.5。工具调用 BFCL 上，CSPD 是 55.6，MOPD 是 49.4。MOPD 在 TruthfulQA 上比学生掉了 3.6 分，CSPD 只掉了 0.4 分。

然后是部署成本。作者用一张 B200 跑 GPT-OSS-20B，给每个请求配一个不同的适配器，模拟每个用户都有专属专家的极端情况。

软提示的吞吐量，在七档并发下，都和不带适配器的模型相差不到百分之一点五。32 个请求同时跑时，软提示的输出吞吐是秩为 4 的 LoRA 的 2.40 倍，首 token 延迟的 P95，也从 LoRA 的 39.21 秒降到 17.26 秒。

论文还分析了软提示到底学到了什么。在 GPT-OSS 数学专家上，软提示把学生和老师输出分布之间的差距，也就是 KL 差距，缩小了百分之八十六点八，主要办法是让学生更常检查中间结果，被拉高最多的 token 是问号、But 和 Check。

换成用文字写的指令，让模型检查、验证自己的答案，最多只缩小百分之二点一，调高推理强度完全没有缩小。所以作者认为，软提示学到的是一种推理习惯，而不是一句能用文字说出来的话。

### 和以前方法的对比

放在一起看。和 MOPD 这类合进权重的方法比，在这组实验里，领域内分数打平，领域外更好，而且加专家不用重训整个模型。

和渐进式训练比，也就是每个新提示都在前面的提示已经就位时训练，独立训练在每个阶段平均分都更高，加完四个是 67.83 对 65.17。

和多个 LoRA 合并比，用同样的蒸馏数据，合并后的 LoRA 平均分是 65.50 和 66.19，都低于软提示的 67.83。

我的判断是，这篇最大的价值不在分数，而在工程形态。专家变成可插拔的模块，不同团队可以各自训练，随时增删，同一批请求还能用不同的组合。

我的判断是，也要看到它的边界。多专家实验只用了一个 4B 的骨干和四个领域，和 MOPD 只差零点零四分，算是打平。而且软提示是从现成的强化学习专家蒸馏出来的，它省掉的是合并这一步，不是强化学习本身。

### 局限

作者自己列了几条局限。第一，组合实验只有一个骨干模型、四个领域，专家越来越多时还能不能继续涨，领域之间的关系有什么影响，都还不知道。

第二，有些数字是在评测基准上挑出来的，比如单专家的检查点和权重合并的系数。老师比学生提升很小时，恢复率也很不稳定，所以论文同时给出了绝对分数。

第三，每个提示会在上下文里多占 256 个位置，组合的专家越多，注意力和 KV cache 的开销就越大。第四，提示和骨干模型是绑定的，换一个骨干就要重新蒸馏，而且必须先有强化学习训出来的老师。

作者还提醒，冻结骨干并不保证安全性不变，装上一个提示，或者几个提示的组合，安全表现都要单独评估。

### 未来方向

作者提了四个方向。一是扩大规模，更多骨干，更大的提示库。二是自动判断每个请求需要哪几个提示，而不是人工指定。三是搞清机制，为什么提示能跨领域迁移。四是成本，更短的提示，以及和 MOPD、LoRA 完整的训练成本对比。

我的判断是，如果提示选择能自动化，这套方法就有点像后训练的插件库，骨干模型固定，能力按需安装。下一步最值得看的，是专家数量到十几个、几十个之后，平均分还能不能一直往上走。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from UC Santa Barbara and more than a dozen other institutions, including Carnegie Mellon, NVIDIA and Apple. It's called Composable Soft-Prompt Distillation of RL Experts, or CSPD for short.

The question behind it: after reinforcement learning gives you a set of specialist models, can you plug those specialists into one base model like add-ons, and unplug them again, without ever touching the base model's weights?

### Problem

Some context first. Reinforcement learning post-training is usually done one domain at a time, because each domain brings its own data, its own verifier, and its own environment.

So the common recipe is to start from one shared base model, train a math expert, a code expert, a science expert, each separately, and then fold them all into a single model.

The trouble is that new domains and stronger experts keep arriving, so folding them in isn't a one-off job. It's continual. The authors' ideal: a new expert gets installed like a module, the model gains what it learned, nothing it could already do gets damaged, and the cost of the update depends only on the new expert, not on how many came before.

### Prior work

Existing approaches fall into two camps. The first camp puts every expert into one set of weights. Joint multi-task RL has to rerun RL on every domain whenever one is added. Sequential RL trains only on the new domain, but later stages can wreck earlier gains.

Weight merging does arithmetic on the parameters, and the experts' updates can interfere. Multi-teacher on-policy distillation, MOPD, has to retrain the student against all the teachers every time. And the authors point out that all of these overwrite the base weights, which can hurt the model outside the target domains.

The second camp keeps experts separate and picks one per request. But a router is hard to train, it can't combine experts inside one request, and you have to host a full model per expert. Merging LoRA experts per request avoids that, but it adds adapter compute to every layer at every decoding step.

### Key idea

Here's the observation. Both camps keep the experts inside the model's parameters. CSPD moves them somewhere else: into the model's input.

The vehicle is a soft prompt. It isn't text anyone wrote. It's a sequence of trainable vectors that sits in the input alongside the real tokens, and in this paper each expert gets two hundred fifty-six of them.

This figure shows the whole pipeline. Each expert is distilled into its own prompt, the prompts form a library, and at inference you simply concatenate the ones you want. Requests carrying different prompts can even share one batch on the same model.

### Method

To train one expert's prompt, they use sequence-level distillation. The post-RL expert, the teacher, answers questions from its domain, and only the answers that pass the verifier are kept.

Then the pre-RL model plays student. It's completely frozen, and the only thing trained is those two hundred fifty-six vectors inserted into the system message, so that with the prompt attached, the student produces the teacher's answers. The loss is this equation, a negative log-likelihood averaged over the response tokens.

Composing is even simpler. You concatenate the experts' prompts and drop them into the same template, with no joint training at all. Adding an expert means training one new prompt, removing one means dropping its prompt, and the backbone never changes.

One detail matters here. Each prompt is trained alone, but at inference other prompts may sit in front of it. So in half the training examples, they put a prefix of one hundred twenty-eight to two hundred fifty-six tokens before the soft prompt, taken from other questions in the same domain, left out of the loss and never used at inference. The authors call this prefix-robust training.

### Results

First question: can a single expert fit into a single prompt? They test twenty-three settings, all built from public pre-RL and post-RL model pairs, covering math, code, and multi-step agent tasks on SWE-bench Verified and Terminal-Bench two point one, with models from seven billion to thirty-five billion parameters.

The metric is recovery: of the gain the teacher has over the student, how much does the prompted student get back? The paper reports at least ninety percent recovery in twenty-two of the twenty-three settings, and in fifteen the prompted student matches or beats the teacher.

For example, in the DeepSWE pair, the student goes from twenty-four point nine to forty-five point six five on SWE-bench Verified, against forty-four point oh five for the teacher. The weakest result, eighty-eight point oh eight percent recovery, is MiMo seven B on code, where the RL training data isn't public, so they distilled on other code questions.

Next, combining experts. They reuse the M2RL setup: the student is a supervised fine-tuned version of Qwen3 4B Base, and the teachers are four RL experts trained from it, for math, code, science and instruction following, evaluated on AIME twenty-four, LiveCodeBench, GPQA Diamond, and IFBench.

The results are in Table two. Concatenating all four prompts lifts the four-task average from the student's fifty point four one to sixty-seven point eight three, level with MOPD at sixty-seven point seven nine, and above both weight-merging methods and above LoRA adapters distilled on the same data and then merged.

It even edges past an idealized reference, where each benchmark goes to its own domain expert, which averages sixty-six point six one. Per task, though, CSPD beats MOPD on GPQA and IFBench, and trails it on AIME and LiveCodeBench.

Take away prefix-robust training and the same four prompts, concatenated, average only sixty-four point five. The authors say this small trick makes composition more stable.

Then they simulate continual post-training. Experts arrive one at a time, instruction following, then science, code, and math, and the average goes sixty point nine nine, sixty-one point six seven, sixty-three point six seven, sixty-seven point eight three. It rises with every addition, and no earlier prompt is retrained.

Individual tasks don't always improve, though. Adding code drops IFBench from sixty-six point seven one to sixty-four point five eight, and adding math then lifts it to sixty-eight point four two. Going the other way, removing the math prompt costs the most on average, four point one six points, including six point five four on GPQA, so the math expert is helping with science.

Now outside the training domains. They test the composition on six held-out benchmarks: refusals, truthfulness, tool calling, trivia knowledge, very hard reasoning, and financial reasoning.

In Table three, CSPD averages fifty-six point six, against fifty point three for the student and fifty-three point five for MOPD. On BFCL tool calling, it's fifty-five point six versus forty-nine point four for MOPD. MOPD loses three point six points on TruthfulQA relative to the student, while CSPD loses only zero point four.

Then serving cost. They run GPT-OSS twenty B on a single B200 and give every request a different adapter, the extreme case of per-user personalization.

Soft-prompt throughput stays within one point five percent of the plain backbone at all seven concurrency levels. With thirty-two requests in flight, soft prompts deliver two point four times the output throughput of rank-four LoRA, and the ninety-fifth percentile time to first token falls from about thirty-nine seconds with LoRA to about seventeen.

The paper also asks what a soft prompt actually learns. On the GPT-OSS math expert, the prompt closes eighty-six point eight percent of the KL gap between student and teacher, mostly by making the student check its intermediate results. The tokens it boosts most are a question mark, "But", and "Check".

Written instructions asking the model to check and verify its work close at most two point one percent of that gap, and turning up reasoning effort closes none. So the authors conclude the prompt learns a reasoning habit, not a message you could put into words.

### Comparison

Putting it side by side. Against MOPD-style consolidation into the weights, in this setup CSPD ties in-domain, does better out of domain, and adding an expert never means retraining the whole model.

Against progressive training, where each new prompt is trained with the earlier prompts already in place, independent training scores higher at every stage, sixty-seven point eight three versus sixty-five point one seven after all four.

Against merged LoRA adapters trained on the same distillation data, the merged adapters average sixty-five point five and sixty-six point one nine, both below the soft prompts' sixty-seven point eight three.

My read is that the real value here isn't the small score edge, it's the engineering shape. Experts become plug-in modules that different teams can train on their own, add and remove at will, and mix differently for each request in the same batch.

My read is that the limits matter too. The multi-expert study uses one four-billion-parameter backbone and four domains, and the margin over MOPD is four hundredths of a point, which is a tie. And the prompts are distilled from RL experts that already exist, so what CSPD saves is the merging step, not the reinforcement learning itself.

### Limitations

The authors list several limits. First, the composition study covers one backbone and four domains, so whether it keeps improving as experts pile up, and how that depends on how related the domains are, is still open.

Second, some reported numbers were selected on the evaluation benchmarks, such as single-expert checkpoints and merging coefficients. And when the teacher's gain is small, recovery ratios get noisy, which is why absolute scores are reported throughout.

Third, each prompt adds two hundred fifty-six positions to the context, so attention and KV-cache costs grow with every expert you compose. Fourth, prompts are tied to their backbone: a new backbone means distilling again, and you still need RL-trained teachers in the first place.

They also caution that freezing the backbone doesn't guarantee safety is preserved. Any attached prompt, or combination of prompts, needs its own safety evaluation.

### Future work

They name four directions. Scale, with more backbones and bigger prompt libraries. Prompt selection, learning which prompts each request needs instead of picking them by hand. Mechanism, understanding why prompts transfer across domains. And cost, with shorter prompts and a full training-cost comparison against MOPD and LoRA.

My read is that if prompt selection gets automated, this starts to look like a plug-in library for post-training, with a fixed backbone and capabilities installed on demand. The thing I'd watch next is whether the average keeps climbing once there are dozens of experts instead of four.

That's today's Daily Frontier Papers. The paper is linked in the description.

