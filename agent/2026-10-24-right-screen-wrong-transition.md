# Right Screen, Wrong Transition: World Models as Verifiers for GUI Agents

**画面没问题跳转有问题**

- 论文 Paper: [arXiv 2610.11942](https://arxiv.org/abs/2610.11942)
- 单位 Affiliation: 南洋理工大学 · 东京科学大学 · Sony AI
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-10-24
- 视频 Videos: [YouTube 中文](https://youtu.be/xGwfIoP5gkE) · [B站](https://www.bilibili.com/video/BV11bnd6pEfH)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇南洋理工大学、东京科学大学和 Sony AI 合作的论文，标题叫，画面没错，跳转错了，副标题是，让世界模型给 GUI Agent 当验证器。

它要回答的问题是，一个替你点手机、点网页的 Agent，怎么发现自己被人带进了陷阱。

### 问题

先看论文开头的例子。你点了登录，弹出一个要邮箱和密码的页面，很正常。可如果你点的是查看订单，弹出的却是一模一样的登录页，那就是攻击。

GUI Agent，就是能看屏幕、替人点按钮的 AI。它操作的 App、网页、广告都是第三方做的，不一定诚实。这类攻击的特点是，每个画面看起来都正常，只是动作之后出现的东西被换掉了。

所以作者说，安全是跳转的属性，不是画面的属性。只盯画面的监控，攻击者复用一个正常页面就能骗过去。

要判断跳转对不对，得先有预期，就是这个动作之后，本来应该出现什么。

### 以前的方法

能提供预期的，叫世界模型，就是给定当前画面和动作，预测下一步。GUI 领域已经有不少世界模型，但它们都把预测画出来，输出成文字、截图或者代码。

问题在于，这些输出和 Agent 真正看到的截图不是同一种东西，对不对得上，只能再找一个模型当裁判，而裁判本身也得被信任。作者的说法是，需要裁判的验证器，只是把问题挪了个地方。

还有成本。这张图左边是现有做法，光是生成下一屏，就要 93 到 124 秒，之后还要跑裁判。

### 核心想法

作者的想法是，用来做验证的世界模型，不该把未来画出来，而该直接在观测的编码空间里做预测。

简单说，截图经过编码器变成一组向量，模型直接预测下一屏的向量。预测和观测在同一个空间，一个余弦相似度就能判断有没有对上，不需要解码器，也不需要裁判。论文报告，一次判断 17 毫秒。

这个模型叫 LGWM，隐空间 GUI 世界模型。作者也说，网络结构很常规，新的地方不在模块，而在输出拿来干什么。

### 方法

看这张图的左边，是训练。当前屏幕和动作分别编码，一个 Transformer 预测器输出下一屏的表示，目标是真实下一屏的编码。

屏幕编码器是 DINOv2，一张截图被切成 512 个小块，每块一个向量。预测器是 12 层 Transformer，运行时整个模型大约 1.73 亿参数。

第一个设计针对动作。返回、滚动在哪儿都一个意思，但点击离不开坐标。所以点击坐标和屏幕小块用同一套位置编码，点在哪里，就和那里的图像块对上。

第二个设计针对稀疏变化。一次点击通常只改屏幕的一小块，如果损失平均分配，模型的精力都会花在不变的背景上。所以变了的小块，损失权重乘以 8。

另外有两个辅助目标，一个防止表示塌缩，一个要求能从前后两屏还原出是哪个动作。训练后期，还让预测器拿自己的输出接着预测两到三步，习惯自己的误差。

训练数据是 5 个公开来源的 185 万条真实 GUI 跳转，覆盖手机、网页和跨 App 操作，不需要人工标注。

使用时看图的右边。Agent 执行动作后，把真实出现的屏幕也编码，和预测相减，得到残差。从残差上读两样东西。

第一样是大小。完整性分数是 1 减去预测和实际的余弦相似度，越高越说明这次跳转不在预期里。它不用攻击样本，也不用安全标注，作者称之为严格意义上的免训练。

第二样是方向。出乎预期不等于有害，推送通知、加载转圈、页面改版都会出乎预期，但不伤人。有没有害要看偏到了哪个方向，所以作者在残差上接了一个线性分类头，这是整个系统唯一需要监督训练的部分。

### 实验和效果

为了测试，作者做了一个评测集，叫 RSWT-BENCH，构造很巧。劫持样本保留真实的屏幕和动作，把结果换成别处的登录页。而这个登录页，在评测集里也作为另一次正常跳转的结果出现，像素完全一样，标签相反。

这样，任何只看结果画面的检测器，AUC 期望值正好是 0.5，和瞎猜一样，论文给了证明。AUC 可以理解为，随机拿一个坏样本和一个好样本，把坏的排在前面的概率。

测试集有 966 条跳转，围绕 222 对配对样本。三位标注员盲审 72 条，和构造标签的一致率大约 96.3%。

结果在表 1。LGWM 的免训练分数，AUC 是 0.987。222 对里，排对了 219 对。

最强的闭源模型，Gemini 3.7 Flash 是 0.995，GPT-5.6 Luna 是 0.988，Qwen3.7 Plus 是 0.973。配对统计检验显示，LGWM 和这三个没有显著差异。

这些大模型拿到的是同样的前后两屏和动作，再加一个提示词去判断。而 LGWM 在表里只有 2.61 亿参数，算的只是一个余弦距离。

人类标注员在 40 个样本上的平均 AUC 是 0.958，说明这个任务人来看也不简单。

再看成本。一次判断 17.1 毫秒，显存 1.09 GB。和同样条件下的 Qwen3-VL-8B 比，计算量少 27 倍，延迟低 64 倍。作者说，这样 Agent 每一步都能检查，而不是抽查。

然后是判断有没有害。把劫持跳转和无害但意外的跳转放在一起比，叫 Harm AUC。只看偏离大小的完整性分数只有 0.513，提示词方式的大模型在 0.512 到 0.614 之间，都接近瞎猜。读残差方向的线性头，达到 0.953。

反过来，在找错误跳转的 RSWT 上，完整性分数 0.987，比线性头的 0.930 更强。作者的结论是，大小负责发现异常，方向负责解释异常。

作者还排除了几种取巧的解释。第一，是不是在复制当前屏。预测和真实下一屏的相似度是 0.618，和当前屏只有 0.245。

第二，是不是在背训练数据。在大约 100 个候选里找真实下一屏，模型 68.3% 排第一，记忆基线只有 44.6%。

第三，预测会不会跟着动作变。把点击换成滚动，原预测仍更接近真实结果的比例是 0.891。第四，连续往前推 10 步，和真实编码的相似度还有 0.562，画面不变的基线是 0.455。

表 2 的消融在更小的模型上做。去掉多步训练，10 步预测变差。去掉防塌缩的正则，有效秩降了 49%。

### 和以前方法的对比

最直接的对比，是同样有预期、但把预期渲染出来，再交给 Gemini 3.7 Flash 判断的生成式世界模型。gWorld-8B 是 0.885，Code2World 是 0.870，低大约 10 个点，而且每条跳转至少要 92.9 秒，这还没算裁判。

开源视觉语言模型里，80 亿到 90 亿参数的 GLM-4.6V-Flash 和 Qwen3-VL-8B，分别是 0.909 和 0.907。

信号从哪来，作者也做了对照。只看单帧最好是 0.796，能看前后两屏和动作、但看不到预测的分类器是 0.833。把预测换成别的跳转的，线性头从 0.930 掉到 0.635。起作用的是学到的跳转规律。

我的判断是，这篇的价值不在又多一个安全分类器，而在换了问法。只看画面，在这个评测里理论上就是瞎猜，必须看跳转。17 毫秒一次，它可以放在每一步，当一道便宜的闸门。

我的判断是，也要看到差距。三个指标的第一名仍是 Gemini 3.7 Flash。而且评测集是作者自己构造的，劫持方式只有换成登录页这一种，论文没有报告真实攻击下的测试。

### 局限

论文没有单独的局限章节，正文里作者点出了几处。第一，评测集只有 222 对，因为构造条件很严格。作者的理由是，结论靠的是构造的严格，而不是数量。

第二，攻击者想骗过它，得在 Agent 自己的表示空间里伪造结果，作者认为这更难，但有多难还没有量化。

第三，消融只用了一个随机种子，而且是在更小的模型上做的。

另外，判断有害的线性头需要标注数据，用的是和测试集不重叠的 6786 条样本。

### 未来方向

作者的设想是，两类世界模型互补。隐空间闸门每一步都查，只有被标记的少数情况，再交给渲染式世界模型或大模型裁判。

这个分数也不只针对攻击，漏掉的动作、布局变化、意外弹窗都能发现，所以也能当可靠性监控用。

作者的总结是，世界模型过去当模拟器和规划器，对环境不可信的 Agent，它还能当验证器。用他们的话说，渲染是给人看的，预测是给机器用的。

我的判断是，下一步要看它在真实劫持攻击里能不能站住，以及攻击者知道这道闸门之后，会不会专门去骗它的表示空间。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Nanyang Technological University, the Institute of Science Tokyo and Sony AI. It's called Right Screen, Wrong Transition, and the subtitle is the pitch: world models as verifiers for GUI agents.

The question: how can an agent that taps through apps and web pages for you tell that it has just been led into a trap?

### Problem

Start with the paper's opening example. You tap Sign in, and a screen asks for your email and password. Normal. But if you tapped View order and the exact same login screen appears, that's an attack.

A GUI agent is an AI that reads the screen and presses buttons for you. The apps, pages and ads it works through are built by third parties, and they may not be honest. These attacks change what happens after an action, while every single screen still looks plausible.

So the authors argue that safety belongs to the transition, not the screen. A monitor that only looks at screens can be beaten by reusing a legitimate one.

Judging a transition takes an expectation: after this action, what should have come next?

### Prior work

That's what a world model provides: given a screen and an action, predict what comes next. There are already several world models for GUIs, but they all render their predictions, as text, screenshots, or code.

The trouble is that none of those is the same kind of object as the screenshot the agent sees. Comparing them needs a second model as a judge, and that judge has to be trusted too. As the authors put it, a verifier that needs a judge has only moved the problem.

Then there's cost. On the left of this figure, generating the next screen alone takes ninety-three to one hundred twenty-four seconds, before the judge runs.

### Key idea

The idea is that a world model built for verification shouldn't draw the future. It should predict directly in the space where observations are encoded.

Put simply, a screenshot becomes a set of vectors, and the model predicts the next screen's vectors directly. Prediction and observation share a space, so one cosine similarity says whether they match, with no decoder and no judge. The paper reports seventeen milliseconds per decision.

The model is called LGWM, a Latent GUI World Model. The authors say the architecture is conventional. What's new is what the output is for.

### Method

The left of this figure is training. The current screen and the action are encoded, a Transformer predictor outputs the next screen's representation, and the target is the encoding of the real next screen.

The screen encoder is DINOv2, which cuts a screenshot into five hundred twelve patches, each with its own vector. The predictor is a twelve-layer Transformer, and the whole model runs with about one hundred seventy-three million parameters.

The first design choice is about actions. Back and scroll mean the same thing anywhere, but a tap means nothing without its coordinates. So tap coordinates share a positional encoding with the screen patches, lining up a touch with the patch it lands on.

The second is about sparse changes. A tap usually changes a small part of the screen, and an evenly weighted loss would spend its effort on static background. So patches that changed get eight times the weight.

Two auxiliary objectives keep the representations from collapsing and make them recover which action caused a change. Later in training, the predictor also feeds its own output back for two or three steps, to get used to its own errors.

Training uses one point eight five million real GUI transitions from five public sources, across mobile, web and cross-app tasks, with no human labels.

At runtime, look at the right side. After the agent acts, the actual screen is encoded and the prediction is subtracted from it, giving a residual. Two things are read from it.

The first is its size. The integrity score is one minus the cosine similarity between prediction and observation, so higher means less expected. It uses no attack examples and no safety labels; the authors call it training-free in a strict sense.

The second is its direction. A surprise isn't harm: push notifications, loading spinners and layout changes all break expectations without hurting anyone. Harm depends on which way reality departed, so a linear head reads the residual, and it's the only supervised part of the system.

### Results

To test this, the authors built a benchmark, RSWT-BENCH, and the construction is clever. A hijacked example keeps a real screen and action but swaps in a login screen from somewhere else. That same login screen also appears in the set, pixel for pixel, as the outcome of a legitimate transition, with the opposite label.

So any detector that only sees the resulting screen has an expected AUC of exactly zero point five, a coin flip, and the paper proves it. AUC here is the chance of ranking a bad example above a good one.

The test set has nine hundred sixty-six transitions around two hundred twenty-two pairs. A blind audit of seventy-two items by three annotators matched the labels about ninety-six point three percent of the time.

The results are in Table 1. LGWM's training-free score reaches an AUC of zero point nine eight seven. It orders two hundred nineteen of the two hundred twenty-two pairs correctly.

The strongest closed models: Gemini 3.7 Flash at zero point nine nine five, GPT-5.6 Luna at zero point nine eight eight, Qwen3.7 Plus at zero point nine seven three. Paired statistical tests found no significant difference between LGWM and any of the three.

Those models get the same before and after screens and action, plus a prompt. LGWM is listed at two hundred sixty-one million parameters, and computes one cosine distance.

Human annotators, on forty examples, averaged zero point nine five eight, so this isn't easy even for people.

Now cost. One decision takes seventeen point one milliseconds and about one gigabyte of memory. Against Qwen3-VL-8B under the same setup, that's twenty-seven times fewer FLOPs and sixty-four times lower latency. The authors' point: every step of an agent can be checked, not just a sample.

Then harm. Comparing hijacked transitions with harmless but surprising ones gives Harm AUC. The integrity score, which only measures size, gets zero point five one three, and prompted models land between zero point five one two and zero point six one four, all near chance. The linear head on the residual's direction reaches zero point nine five three.

On RSWT, it flips: the integrity score's zero point nine eight seven beats the head's zero point nine three zero. Size detects a violation; direction interprets it.

The authors also rule out cheap explanations. First, is it copying the current screen? The prediction's similarity is zero point six one eight to the true next screen and only zero point two four five to the current one.

Second, is it memorizing training data? Among about one hundred candidates, it ranks the true next screen first sixty-eight point three percent of the time, against forty-four point six percent for a memorization baseline.

Third, does the prediction follow the action? Swap a tap for a scroll, and the original forecast stays closer to the real outcome at a rate of zero point eight nine one. Fourth, ten steps out, agreement with the true encoding is still zero point five six two, against zero point four five five for a no-change baseline.

The ablations in Table 2 use a smaller model. Without multi-step training, ten-step forecasts get worse. Without the anti-collapse regularizer, effective rank falls forty-nine percent.

### Comparison

The most direct comparison is generative world models, which hold the same expectation but render it and hand it to Gemini 3.7 Flash as judge. gWorld-8B scores zero point eight eight five and Code2World zero point eight seven zero, about ten points lower, and each needs at least ninety-two point nine seconds per transition, before the judge.

Among open vision-language models of eight to nine billion parameters, GLM-4.6V-Flash and Qwen3-VL-8B score zero point nine zero nine and zero point nine zero seven.

Controls show where the signal comes from. The best single-frame probe gets zero point seven nine six, and a classifier that sees both screens and the action, but not the prediction, gets zero point eight three three. Pairing the prediction with the wrong transition drops the residual head from zero point nine three zero to zero point six three five. The learned model of how screens change is doing the work.

My read is that the value isn't one more safety classifier, it's the reframing. On this benchmark, screen-only detection is a coin flip by construction, so you must look at transitions. At seventeen milliseconds, this can sit on every step as a cheap gate.

My read is also that the gap is real. Gemini 3.7 Flash still tops all three metrics. And the benchmark is the authors' own, with one kind of hijack, a swapped-in login screen, and the paper reports no tests against real attacks.

### Limitations

There's no limitations section, but the authors flag several things. First, the benchmark has only two hundred twenty-two pairs, because the construction is strict. They argue that strictness, not size, is what supports the conclusion.

Second, to fool this monitor an attacker would have to forge an outcome in the agent's own representation space, which the authors think is harder, but they haven't quantified how much.

Third, the ablations use a single random seed and a smaller model.

Also, the harm head needs labeled data: a separate split of six thousand seven hundred eighty-six examples, disjoint from the test set.

### Future work

The authors see the two kinds of world model as complementary. The latent gate checks every step, and only the flagged few go on to a rendering world model or a language-model judge.

The score also isn't specific to attacks. It flags missed actions, layout changes and unexpected pop-ups, so it can double as a reliability monitor.

Their summary: world models have served as simulators and planners, and for agents in untrusted environments they can also be verifiers. As they put it, rendering is for people, and prediction is for machines.

My read is that the thing to watch is whether this holds up against real hijacking attacks, and whether attackers who know about the gate start targeting its representation space.

That's today's Daily Frontier Papers. The paper is linked in the description.

