# VLA-Rolling: Fewer Denoising Steps, Better Control via Adaptive Action Reuse

**机器人扔掉的半截动作有用**

- 论文 Paper: [arXiv 2610.25043](https://arxiv.org/abs/2610.25043)
- 单位 Affiliation: 浙江大学
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-10-23
- 视频 Videos: [YouTube 中文](https://youtu.be/G3c7FSWZeCw) · [YouTube English](https://youtu.be/49-WhgrfOZw) · [B站](https://www.bilibili.com/video/BV1Uent6LEzM)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇浙江大学的论文，名字叫 VLA-Rolling，标题的意思是，去噪步数更少，控制反而更好，靠的是自适应地复用动作。

它盯住的是机器人大模型推理里一个很常见、但一直被浪费掉的东西，就是每一轮没执行完、直接扔掉的那半截动作。

### 问题

先说背景。现在很多机器人大模型叫 VLA，也就是视觉、语言、动作模型，看一眼画面、读一句指令，就输出接下来的一串动作，这一串叫 action chunk，动作块。

像 π0.5、SmolVLA 这类模型，动作是用 flow matching 生成的。简单说，就是从一团随机噪声出发，一步步去噪，最后变成一串动作，做这件事的模块叫 action expert。

但整串动作只基于一次观察，环境一变，后面的动作就会不准。所以常见做法是滚动执行，英文叫 receding horizon，只执行前面一小段，然后重新看、重新规划。

代价是，没执行的那一截，论文叫 action tail，动作尾巴，会被直接丢掉，下一轮又从全新的噪声开始。以论文的设置为例，π0.5 每轮生成 50 步动作，只执行 10 步，剩下的 40 步全扔了。

作者的问题是，能不能利用这截尾巴，更快地生成新动作，同时不牺牲动作的准确度。

### 以前的方法

以前也有工作复用上一轮的动作，让动作更连贯。但作者认为，它们太依赖动作块本身的质量，质量差的动作也照样被拿来当基础，结果就会生成错误的动作。

另一类工作专门做 VLA 推理加速，比如把模型做小，或者改动作解码的方式。作者的说法是，这些方法通常要拿动作的准确度去换速度。

### 核心想法

作者的出发点是一个观察。动作尾巴已经去过噪，而且在时间上，正好和下一轮要用的动作重叠，所以它可能是比纯噪声更好的起点。

环境也确实变化不大。作者统计，在常见 VLA 的执行长度下，将近 98% 的相邻步，画面和机器人自身的状态都高度相似。

他们先做了一个诊断实验，就是这张图。在 LIBERO 和 MetaWorld 上，从尾巴出发，π0.5 只要 4 步去噪，SmolVLA 只要 6 步，动作误差就能追平、甚至好于从噪声出发完整去噪 10 步的结果。

但没有一个固定步数适合所有情况。每个样本的最优步数从 1 到 10 都有，π0.5 平均是 7.07 步。而且步数也不是越多越好，π0.5 在 LIBERO 上，尾巴去噪到 8 步，误差反而比 6 步更大。

所以真正的问题，不是要不要复用尾巴，而是在每一个新的观察下，这截尾巴该信几分。

### 方法

VLA-Rolling 分三步，而且原来的 VLA 完全冻结，一点都不改。整体结构是这张图。

第一步，继承尾巴。把上一轮没执行的动作，在时间上对齐到这一轮，同时记下上一轮的画面和机器人状态，算出这一轮相比上一轮变了多少。

第二步，估计置信度。作者训练了一个大约 3000 万参数的小 Transformer，叫动作置信度估计器。它看当前的画面和状态、它们的变化量、上一个执行的动作，还有那截尾巴，给尾巴里的每一个动作打一个 0 到 1 之间的置信度。

它怎么训练呢。用离线的示范数据，取前后相隔一段的两个时刻，看继承来的动作和示范里真实的动作差多少，差得越小，置信度的标签就越高。

这个误差先取对数，再按分位数归一化，让不同模型、不同环境的尺度统一。损失函数还故意不对称，把置信度估高了，罚得是估低的两倍。

第三步，按置信度来精修，置信度同时管两件事。

一是重新加噪。每个动作按置信度和噪声混合，置信度高，保留的原动作就多，置信度低，就更接近从纯噪声重新生成。尾巴之外、新补上的那一段，直接用噪声。

二是决定去噪步数。取整截尾巴里最低的那个置信度，置信度越低，步数越多，上限就是原模型默认的去噪步数。

这样一来，尾巴不再被扔掉，而是一轮一轮往前滚，一个动作真正被执行之前，已经在好几轮里被反复打磨过，这就是名字里 Rolling 的意思。

### 实验和效果

实验在四个仿真基准上做，LIBERO、MetaWorld、LIBERO-Plus 和双臂操作的 RoboTwin 2.0，套在四个不同的 VLA 上，π0.5、X-VLA、SmolVLA 和 MolmoAct2，一共 16 组组合。

结果在表 1。16 组里，每一组的单个动作块推理时间都下降了，降幅在 5% 到 32.52% 之间，最多相当于提速 1.48 倍。成功率在 16 组里有 14 组提高。

主力模型 π0.5 在 LIBERO 上，成功率从 96.38% 提到 97.83%，每个动作块的推理时间从 168.09 毫秒降到 129.6 毫秒，少了 22.9%。

提升最大的是 SmolVLA。在 LIBERO-Plus，也就是加了光照、背景这类干扰的版本上，成功率从 20.74% 提到 35.82%，涨了 15.08 个百分点。最大的提速出现在 MolmoAct2 上，在 LIBERO 上推理时间少了 32.52%。

也有没变好的。X-VLA 在 LIBERO-Plus 和 RoboTwin 上，成功率分别小降了 0.13 和 0.28 个百分点，不过时间也都省下来了。

还要说明一下，这里的时间只算模型推理，包括视觉语言骨干、action expert 去噪，和置信度估计器本身。省下来的部分，只来自去噪步数的减少。估计器本身每次只要大约 1.6 毫秒。

真机实验也做了。用 π0.5 控制一台 6 自由度的机械臂，把土豆放进盘子，土豆快到盘子上方的时候，把蓝盘子撤掉，换成棕色盘子。20 次里，原始模型成功 14 次，加上 VLA-Rolling 成功 15 次，推理时间从 227.28 毫秒降到 176.5 毫秒。

这张图更直观。环境一变，置信度骤降，去噪步数一下子跳上去，模型暂时退回到原来的推理方式，然后转去把土豆放进棕色盘子。

### 和以前方法的对比

这篇论文的对比，主要是和不加插件的原模型比，再加上自己的消融实验，消融在表 2。

只继承尾巴，然后照常完整去噪，π0.5 在 LIBERO-Plus 上成功率反而掉了 4.89 个百分点。只加自适应步数、不重新加噪，也掉了 4.29 个百分点。在 LIBERO 和 LIBERO-Plus 上，成功率的提升都只出现在带重新加噪的版本里。

作者的解释是，继承来的动作有时落在模型训练分布之外，光靠 VLA 自己很难修回来，加一点噪声，能降低这种风险。

还有一个检验。把预测出来的步数故意多一步或者少一步，成功率都会下降，在 LIBERO 上分别掉了 1.08 和 0.83 个百分点，作者认为，这说明预测的步数确实贴合每截尾巴的实际需要。

我的判断是，这个方法最大的好处是即插即用。原模型完全不动，只加一个 3000 万参数的小插件，每个模型训练 4000 步，就在四个不同的 VLA 上都省下了时间。

我的判断是，它的短板也很明显。论文没有和其他加速方法正面比，比如直接固定用更少的去噪步数，或者别的动作复用方法，所以我们不知道它比这些简单基线强多少。

### 局限

作者自己写的局限，主要有两条。第一，跨环境的泛化探索得不多。第二，真机实验还可以扩展到更复杂的任务，和更多种类的机器人上。

再补充两点。论文表 3 显示，执行长度越长、环境变化越大，置信度和环境变化之间的相关性，到 K 等于 40 时明显变弱，作者也承认，估计器有一定鲁棒性，但仍然有局限。另外，真机实验一共只有 20 次，两边成功次数只差 1 次。

### 未来方向

未来方向，作者提到两点。一是在真机上，探索除了环境突变以外，还有哪些因素会影响置信度判断。二是和其他推理加速方法结合，因为它不改原模型，有可能和别的方法叠加使用。

我的判断是，这篇的价值在于换了个角度看效率。大家都在想怎么让模型每一步算得更快，它指出，每一轮扔掉的那截动作，本身就是现成的中间结果，关键是复用之前，先判断它还靠不靠谱。

我的判断是，也要看清楚量级。提速最多 1.48 倍，主力模型 π0.5 上是 5% 到 22.9%，省的只是去噪那一部分，视觉语言骨干的开销没有变。对追求高控制频率的部署来说，它更像一个可以叠加的增量，而不是替代方案。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Zhejiang University. It's called VLA-Rolling, and the title makes a bold promise: fewer denoising steps, better control, by reusing actions adaptively.

What it targets is something robot foundation models throw away on every single cycle: the part of each action plan that never gets executed.

### Problem

Some background first. Many robot models today are VLAs, vision-language-action models. They look at the camera image, read an instruction, and output a whole sequence of upcoming actions, called an action chunk.

In models like pi zero point five and SmolVLA, that chunk comes from flow matching. Put simply, you start from random noise and denoise it step by step until it becomes a sequence of actions, and the module that does this is called the action expert.

But the whole chunk is predicted from one observation, so if the scene changes, the later actions go stale. The standard fix is receding-horizon execution: run only the first few actions, then look again and replan.

The price is that the unexecuted part, which the paper calls the action tail, gets thrown away, and the next round starts again from fresh noise. In the paper's setup, pi zero point five generates fifty actions per round, executes ten, and discards the other forty.

So the authors ask: can that tail be used to generate the next actions faster, without giving up accuracy?

### Prior work

Earlier work has reused previous actions to keep motion smooth and consistent. But the authors argue those methods lean too hard on the quality of the old chunk, so bad actions get reused as a foundation too, and the new actions come out wrong.

A separate line of work speeds up VLA inference directly, with smaller models or changes to how actions are decoded. In the authors' view, those methods usually trade away action accuracy to get the speed.

### Key idea

The starting observation is this. The action tail has already been denoised, and it overlaps in time with the actions needed next round, so it may be a better starting point than pure noise.

And the world usually hasn't changed much. The authors measure that, at the execution horizons common VLAs use, the camera view and the robot's own joint state are highly similar in nearly ninety-eight percent of adjacent steps.

They ran a diagnostic first, shown in this figure. On LIBERO and MetaWorld, starting from the tail, pi zero point five needs only four denoising steps and SmolVLA only six to match or beat the action error of a full ten-step run from noise.

But no single fixed budget works for every sample. The best step count ranges all the way from one to ten, with a mean of seven point zero seven for pi zero point five. And more steps aren't always better: for pi zero point five on LIBERO, denoising the tail for eight steps actually gives a larger error than six.

So the real question isn't whether to reuse the tail. It's how much to trust it, given each new observation.

### Method

VLA-Rolling has three stages, and the original VLA stays completely frozen. This figure shows the whole pipeline.

Stage one is tail inheritance. The unexecuted actions from last round are aligned in time with the current round, and the previous image and robot state are kept, so the system can compute how much has changed since then.

Stage two is confidence estimation. The authors train a small transformer, about thirty million parameters, called the action confidence estimator. It takes the current image and state, how much each has changed, the last executed action, and the tail itself, and gives every action in the tail a confidence between zero and one.

How is it trained? On offline demonstrations, take two moments a fixed gap apart, and measure how far the inherited action is from the action the demonstrator actually took. The smaller the error, the higher the confidence label.

That error is log-transformed and then quantile-normalized, so the scale is consistent across models and environments. The loss is also deliberately lopsided: overestimating confidence costs twice as much as underestimating it.

Stage three is refinement guided by that confidence, and the confidence controls two things.

First, renoising. Each inherited action is blended with noise according to its confidence. High confidence keeps more of the old action, while low confidence moves it closer to regenerating from scratch. The new positions beyond the tail simply start from noise.

Second, the denoising budget. Take the lowest confidence anywhere in the tail. The lower it is, the more steps the chunk gets, capped at the frozen model's default step count.

The effect is that the tail is never thrown away. It rolls forward round after round, so by the time an action is actually executed, it has been refined across several rounds. That's the rolling in the name.

### Results

Experiments cover four simulation benchmarks, LIBERO, MetaWorld, LIBERO-Plus, and the two-arm RoboTwin 2.0, across four different VLAs, pi zero point five, X-VLA, SmolVLA and MolmoAct2. That's sixteen model and benchmark pairs.

The results are in Table 1. Inference time per action chunk drops in all sixteen pairs, by between five and thirty-two point five two percent, which is up to a one point four eight times speedup. Success rate improves in fourteen of the sixteen.

For the main model, pi zero point five on LIBERO, success goes from ninety-six point three eight to ninety-seven point eight three percent, and time per chunk falls from one hundred sixty-eight milliseconds to about one hundred thirty, a twenty-two point nine percent cut.

The biggest gain is SmolVLA. On LIBERO-Plus, the version with perturbations like lighting and background changes, its success rises from twenty point seven four to thirty-five point eight two percent, up fifteen point zero eight points. The biggest speedup is MolmoAct2 on LIBERO, where inference time drops by thirty-two point five two percent.

Not everything improved. X-VLA loses zero point one three points of success on LIBERO-Plus and zero point two eight on RoboTwin, though it still saves time on both.

One caveat on what's being timed: it's model inference only, covering the vision-language backbone, the action expert's denoising, and the confidence estimator itself. All of the savings come from running fewer denoising steps. The estimator itself costs only about one point six milliseconds per call.

There's a real-robot test too. Pi zero point five drives a six-degree-of-freedom arm to put a potato on a plate, and as the potato nears the plate, the blue plate is swapped for a brown one. Over twenty trials, the base model succeeds fourteen times and VLA-Rolling fifteen, with inference time falling from about two hundred twenty-seven to about one hundred seventy-six milliseconds.

This plot makes it concrete. When the scene changes, confidence drops, the denoising step count spikes, the model temporarily falls back to its original inference, and it redirects the potato to the brown plate.

### Comparison

The comparisons here are mainly against the same models without the plug-in, plus the paper's own ablations, which are in Table 2.

Inherit the tail but still denoise for the full budget, and pi zero point five's success on LIBERO-Plus actually drops by four point eight nine points. Add adaptive step counts without renoising, and it still drops by four point two nine. On LIBERO and LIBERO-Plus, every gain in success shows up only in variants that include renoising.

The authors' explanation is that inherited actions sometimes sit outside the model's training distribution, where the VLA alone struggles to fix them, and a little added noise lowers that risk.

There's one more check. Shift the predicted step count by one in either direction and success falls, by one point zero eight points with one extra step and zero point eight three with one fewer on LIBERO, which the authors read as evidence that the predicted budget fits what each tail actually needs.

My read is that the big draw here is that it's plug-and-play. The base model isn't touched, you add a thirty-million-parameter plug-in trained for four thousand steps per model, and it saved time on four different VLAs.

My read is also that the weak spot is clear. The paper never goes head-to-head with other acceleration methods, such as simply using a fixed smaller step count, or other action-reuse schemes, so we don't know how much better it is than those simpler baselines.

### Limitations

The authors list two main limitations. First, there's limited exploration of generalization across environments. Second, the real-robot experiments could be extended to harder tasks and more kinds of robots.

Two more points worth adding. Table 3 shows that as the execution horizon grows and the scene changes more, the link between confidence and scene change gets clearly weaker by a horizon of forty, and the authors acknowledge the estimator is somewhat robust but still limited. And the real-robot test is just twenty trials, with the two sides one success apart.

### Future work

For future work, the authors point in two directions. One is studying, on real robots, what besides sudden scene changes affects the confidence estimate. The other is combining this with other inference speedups, since it doesn't modify the base model and could stack with them.

My read is that the value here is a different angle on efficiency. Most work asks how to make each step cheaper. This paper points out that the tail you throw away every round is a ready-made intermediate result, and the key is checking whether it's still trustworthy before you reuse it.

My read is also that the scale matters. The speedup tops out at one point four eight times, and for pi zero point five it's five to twenty-two point nine percent, because only the denoising part gets cheaper while the vision-language backbone costs the same. For deployments chasing high control rates, this looks like an add-on that stacks, not a replacement.

That's today's Daily Frontier Papers. The paper is linked in the description.

