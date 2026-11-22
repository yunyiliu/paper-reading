# WeVisForensics: Self-Evolving Agent for Omni AI-Generated Video Detection

**鉴别AI视频检查员也在进化**

- 论文 Paper: [arXiv 2611.21373](https://arxiv.org/abs/2611.21373)
- 单位 Affiliation: 南京大学、腾讯微信视觉
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-22
- 视频 Videos: [YouTube 中文](https://youtu.be/OEsYrzqPBFM) · [YouTube English](https://youtu.be/L0sKWk_rCwg) · [B站](https://www.bilibili.com/video/BV1HUyGzPE5c)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇南京大学和腾讯微信视觉团队的论文，叫 WeVisForensics，题目的意思是，用一个会自我进化的 Agent，来检测 AI 生成的视频。

一句话说，它把鉴别 AI 视频，做成了一场调查，一个主 Agent 负责决定查什么，一组专门的检查员负责看，而且两边在训练里一起变强。

### 问题

先说背景。视频生成模型越来越逼真，伪造的视频越来越难和真实录像区分开，论文提到的担忧，包括虚假信息、冒充他人，还有公众信任。

难点在于，露馅的地方往往只在很短的一段时间，或者画面的一小块里，比如标签上的一行字写歪了，或者画面里是女生在说话，听到的却是男声。线索横跨画面、文字和音频。

### 以前的方法

现在主流的做法是用多模态大模型。第一类是一次性的思维链推理，一个模型把整段视频看一遍，写一段理由，给出真假结论。

问题在于，只出现一瞬间、或者只占一小块画面的瑕疵，很容易被漏掉，模型甚至会编一个理由把它解释过去。而且所有判断混在一段文字里，没法单独检查。

第二类是多 Agent 方法，要么跑一条固定的专家流程，要么只训练负责协调的那一个，底下的专家是冻结的。比如 ARGUS，收集四份专家报告，只训练最后的裁判。

作者指出，这样的系统，上限卡在专家身上。如果某个专家本来就看不出一个细微的瑕疵，调度得再好，也补不回来。

### 核心想法

作者的出发点是，把鉴别当成一次调查，而不是看一眼就下结论。人类鉴定专家也是这样，先看整体，形成假设，再盯住某个时刻、某个区域，找合适的专家去查，证据够了再下结论。

所以系统分两层。上面是主 Agent，论文叫 investigator，也就是调查员，下面是一个检查员子 Agent 库，每个检查员只负责一个角度。

更关键的一点是，不光调查员要学，检查员也要从调查员发给它们的任务里学，两层一起进化。

### 方法

先看推理时怎么跑。调查员先拿到整段视频的概览，大约每秒一帧，最多 16 帧，加上时长，和有没有音轨。每一轮，它写一段推理，然后决定调哪个检查员、看哪个时间窗口，或者证据够了，直接给结论。

规则是最多调用 4 次，不能重复调用，没有音轨就不能用音频工具。奖励很简单，格式合法，而且真假判断对了，才得分。

检查员一共 6 个。画面外观、物理一致性、常识，这三个按调查员指定的时间窗口，每秒 2 帧去看。还有一个专看画面里的文字，另外两个是音频工具，一个估计说话人的年龄和性别，一个转写语音，看节奏是否异常。

画面类检查员的每份报告都是结构化的，一个结论，有问题、没问题，或者无法判断，一个高中低的置信度，再加一段引用具体时间点的解释。

文字检查员值得单独说。视频生成模型到现在还是写不好字，字形会粘连、断裂、漂移。它先用 PaddleOCR 找出文字，再判断是不是真的文字，最多挑 6 块区域，放大裁剪，逐块检查笔画。

所有模型共用一个 Qwen3.5-4B 底座，调查员和 4 个视觉检查员，各是一个 LoRA 适配器，也就是一小组额外训练的参数，部署时一个底座加 5 个小适配器就够了。两个音频工具和 OCR 检测器保持固定。

训练数据叫 ForenCue-30K，来自 3 万个视频，包括 117552 条按角色拆开的检查员记录，33896 条调查轨迹，还有一个 7500 行的强化学习数据池。轨迹里的每次调用，都是真实执行过的，不是想象出来的观察。

先做监督微调，然后进入论文的核心算法 DTPO，也就是双时间尺度策略优化。快的一层，每个批次都用 GRPO 强化学习更新调查员，每个视频跑 4 条轨迹，组内互相比较。

光看最终对错，不知道哪一次调用有用。所以作者加了一个叫 CLUE 的信号，让一个冻结的参考模型，在每份报告前后，各给正确答案打一次分，报告让它更接近正确答案就加分，误导了就减分。另外还有一项奖励，看调用的工具和这个视频真正该查的角度是否吻合。

慢的一层，每 20 步才更新一次检查员。系统把调查员发出的、正好落在标注瑕疵区间上的调用攒起来，挑出误报、漏报，还有虽然没错、但含糊其辞没帮上忙的报告，作者叫它 hedge。

然后用一个能看到答案提示的老师模型，去蒸馏只能看到原始画面的学生。每个检查员还有一张技能卡，就是几条该报什么、该忽略什么的规则，由一个冻结的大模型来改，只有在近期样本上更准，而且原来答对的一个都没答错，新规则才会被采用。

为什么要分快慢两层。作者的解释是，调查员决定检查员看到什么，检查员的报告又影响调查员，两边每一步同时更新，就互相成了移动靶。整套联合训练，在他们的集群上大约一天跑完。

### 实验和效果

评测分三类。域内是 GenBuster，域外是 GenBuster++ 和 MintVid，MintVid 按六个生成器分开报。还有作者自建的 AgenticCue，用的是 Veo 3.1、可灵、Seedance、Sora 2 这些较新的商业模型，分五类瑕疵，每类 500 个假视频，配 500 个真视频，和训练集没有重合。

主结果在表 4。四个基准的平均准确率，WeVisForensics 是 95.2%，比最强的已有检测器 VideoVeritas 的 86.2 高 9 个点。作为参照，Gemini-3.1-Pro 的平均是 59.6。

分开看，域内是 97.7，GenBuster++ 是 97.5，MintVid 六个生成器平均 88.0，AgenticCue 五类都在 97.0 到 98.8 之间。

音频这一类差距最大。论文报告，BusterX++ 在音频类只有 49.9，Qwen3.5-397B 在常识类有 94.5，音频只有 54.1，而 WeVisForensics 是 97.0，比 VideoVeritas 高 27.7 个点。

有个细节值得说。Gemini-3.1-Pro 在 GenBuster++ 上，假视频抓到了 99%，但准确率只有 54.3，因为它几乎把所有视频都判成假的。所以只看抓到多少假视频，是会误导人的。

再看它有没有找对检查员。表 5 里，调查员调用物理、文字、常识和音频检查员的精确率，在 94.3 到 99.7 之间。平均每个视频调用 2.19 次，比监督微调版本的 2.50 次少，准确率反而更高。

### 和以前方法的对比

消融实验在表 6。监督微调版本平均 91.3。只训练调查员、检查员冻结，到 93.2，再加上 CLUE，只多了 0.5，作者的解释是，冻结的检查员成了证据瓶颈。

让检查员一起进化，不加 CLUE 是 94.8，完整的 DTPO 是 95.2。表 9 也说明，只进化检查员是 92.0，只训练调查员是 93.2，两层都进化，才到 95.2。

检查员到底有没有变好。物理检查员在 GenBuster 上，回答无法判断的比例从 26.9% 降到 0.3%，F1 从 66.5 升到 97.8。人工盲评 200 对报告，进化后的版本在 65 对里更好，原版只赢 20 对，有缺陷的报告从 30.5% 降到 8.0%。

更新节奏也很重要。表 7 里，如果每个批次都刷新检查员，在没见过的六个生成器上，平均准确率反而低了 2 个点，海螺掉得最多，4.8 个点。

举个例子。一段视频里，一个女生在介绍几个产品盒，外观检查员先说她的脸过于光滑，像渲染出来的，置信度中等。调查员接着在第 4 秒调用文字检查，放大发现几个盒子上重复的字母 A，结构不一致，最后判为假。

我的判断是，这篇真正的新意，是让被调度的专家，也从调度里学习。以前的 Agent 检测器主要训练指挥，这篇的实验显示，把检查员也放进训练循环，收益比单独训练任何一层都大。

### 局限

这篇论文没有单独的局限章节，下面几条，是我从它的结果里读出来的。

第一，没见过的生成器上，还有明显差距。域内接近 98，但 MintVid 上，即梦是 82.9，Sora 2 是 82.4，Wan 2.5 是 85.6，还低于 BusterX++ 的 87.3。

第二，AgenticCue 是作者自己建的，标注主要靠视觉语言模型，再加人工复核，表 4 里的二值检测器，也是在作者的数据上重新训练的。独立的第三方评测，还需要等。

第三，每个视频要跑多轮调用，最多 4 次，论文报告了调用次数，但我没有在论文里看到单个视频要花多少时间和算力。

### 未来方向

论文首页给出了代码和模型权重的链接，结论部分强调的，是调查员和检查员一起进化的价值。

我的判断是，这套快慢两层的思路，不只适用于鉴伪。任何一个主 Agent 调用一组可训练专家的系统，都可以借鉴，让专家从主 Agent 的真实调用里学习。

我的判断是，鉴伪是一场持续的军备竞赛，部分没见过的生成器上八十出头的准确率说明，检查员能不能跟着新模型持续进化，才是这类系统真正要证明的。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Nanjing University and Tencent's WeChat Vision team. It's called WeVisForensics, and it uses a self-evolving agent to detect AI-generated video.

The short version: it turns spotting a fake video into an investigation. One lead agent decides what to check, a team of specialist inspectors does the looking, and both sides get better together during training.

### Problem

Some background first. Video generators keep getting more realistic, and fabricated clips are getting harder to tell apart from real recordings. The paper points to misinformation, impersonation and public trust as the concerns.

The hard part is that the giveaway is often tiny. It might last a fraction of the clip, or sit in one corner of the frame. A line of text on a label warps, or the person on screen is a woman while the voice you hear is a man's. The clues span pictures, text and sound.

### Prior work

The current approaches use multimodal language models. The first family does one pass of chain-of-thought reasoning: a single model watches the whole clip, writes a rationale, and gives a verdict.

The catch is that brief or localized artifacts are easy to miss, and the model may even invent a reason to explain them away. And because every claim is tangled into one block of text, none of them can be checked on its own.

The second family uses multiple agents. Either they run a fixed pipeline of experts, or they train only the coordinator while the experts stay frozen. ARGUS, for example, collects four expert reports and trains only the final judge.

The authors' point is that such systems are capped by their experts. If an expert can't see a subtle artifact, no amount of better routing will recover it.

### Key idea

The starting idea is to treat detection as an investigation, not a single glance. A human analyst works the same way: form a hypothesis from the whole clip, zoom in on a moment and a region, consult the right specialist, and keep going until the conclusion is supported.

So the system has two levels. On top is the main agent, which the paper calls the investigator. Below it is a library of inspector sub-agents, each covering one forensic angle.

And here's the key part. The investigator learns, but the inspectors also learn, from the very calls the investigator sends them. The two levels evolve together.

### Method

Here's how it runs at test time. The investigator first gets an overview of the clip, about one frame per second, sixteen frames at most, plus the duration and whether there's an audio track. Each turn, it writes its reasoning and then picks an inspector and a time window, or decides it has enough evidence and answers.

The rules: at most four calls, no repeated calls, and no audio tools on silent videos. The reward is simple. The trajectory has to be well formed and the real-or-fake verdict has to be right.

There are six inspectors. Visual appearance, physical consistency and common sense each look at the requested window at two frames per second. One more inspector reads on-screen text. And two audio tools: one estimates the speaker's age and gender, the other transcribes the speech and looks for unnatural timing.

Every visual report has the same structure: a verdict, which is issue, no issue, or inconclusive, a confidence of high, medium or low, and an explanation that cites timestamps.

The text inspector deserves a word. Video generators still struggle with text, and letters fuse, break or drift as the clip plays. The inspector runs PaddleOCR to find text, checks whether it's real text, picks up to six regions, crops them at higher resolution, and inspects the strokes one crop at a time.

All the models share one Qwen3.5-4B backbone. The investigator and the four visual inspectors are each a LoRA adapter, a small set of extra trained weights, so serving the whole system takes one backbone plus five small adapters. The two audio tools and the OCR detector stay fixed.

The training data is called ForenCue-30K. It's built from thirty thousand videos, with one hundred seventeen thousand five hundred fifty-two inspector records split by role, thirty-three thousand eight hundred ninety-six investigation trajectories, and a seven-thousand-five-hundred-row pool for reinforcement learning. Every call in those trajectories was actually executed, so the investigator never learns from imagined observations.

Training starts with supervised fine-tuning, then moves to the paper's core algorithm, DTPO, short for dual-timescale policy optimization. The fast level updates the investigator on every batch with GRPO, running four trajectories per video and comparing them within the group.

But a final right-or-wrong doesn't say which call helped. So the authors add a signal called CLUE. A frozen reference model scores the correct answer before and after each report. If the report moved it toward the right answer, the call gets credit, and if it misled, it gets penalized. There's also a bonus for calling the tools that match the angles this video actually needs.

The slow level updates the inspectors only every twenty steps. The system collects the investigator's calls that land on annotated artifact intervals, and picks out false alarms, misses, and reports that were technically correct but too vague to help, which the authors call hedges.

Then a teacher that gets a hint about the answer is distilled into a student that sees only the raw frames. Each inspector also carries a skill card, a short list of rules about what to report and what to ignore. A frozen language model edits it, and a new card is adopted only if it gets more verdicts right on recent calls without getting a single previously correct one wrong.

Why two speeds? The authors explain that the investigator decides what the inspectors see, and the inspectors' reports shape the investigator. Update both every step and each becomes a moving target for the other. The full joint schedule takes about a day on their cluster.

### Results

Evaluation comes in three groups. In-domain is GenBuster. Out-of-domain is GenBuster++ and MintVid, which is reported per generator, six of them. And there's AgenticCue, which the authors built from newer commercial generators like Veo 3.1, Kling, Seedance and Sora 2. It has five artifact categories, each with five hundred fake clips paired with five hundred real ones, and none of it overlaps the training set.

The main results are in Table 4. Averaged over the four benchmarks, WeVisForensics scores ninety-five point two percent, nine points above the strongest prior detector, VideoVeritas, at eighty-six point two. For reference, Gemini 3.1 Pro averages fifty-nine point six.

Broken down: ninety-seven point seven in-domain, ninety-seven point five on GenBuster++, eighty-eight point zero averaged over the six MintVid generators, and between ninety-seven point zero and ninety-eight point eight on all five AgenticCue categories.

Audio shows the widest gap. The paper reports BusterX++ at forty-nine point nine on the audio category, and Qwen3.5-397B at ninety-four point five on common sense but only fifty-four point one on audio. WeVisForensics scores ninety-seven point zero, twenty-seven point seven points above VideoVeritas.

One detail is worth pausing on. On GenBuster++, Gemini 3.1 Pro catches ninety-nine percent of the fakes, but its accuracy is only fifty-four point three, because it calls almost everything fake. Counting caught fakes alone can mislead you.

Does it call the right inspector? In Table 5, precision for the physical, text, common-sense and audio inspectors runs from ninety-four point three to ninety-nine point seven. It makes two point one nine calls per video on average, fewer than the two point five of the fine-tuned starting point, and its accuracy is still higher.

### Comparison

The ablations are in Table 6. The fine-tuned starting point averages ninety-one point three. Train only the investigator with the inspectors frozen, and you get ninety-three point two. Add CLUE, and it gains just half a point. The authors' reading is that frozen inspectors become an evidence bottleneck.

Let the inspectors evolve too, and you reach ninety-four point eight without CLUE, and ninety-five point two with the full DTPO. Table 9 tells the same story: evolving only the inspectors gives ninety-two point zero, training only the investigator gives ninety-three point two, and evolving both gives ninety-five point two.

Did the inspectors actually get better? On GenBuster, the physical inspector's share of inconclusive answers drops from twenty-six point nine percent to zero point three, and its F1 rises from sixty-six point five to ninety-seven point eight. In a blind human review of two hundred report pairs, the evolved version was preferred in sixty-five and the original in twenty, and flawed reports fell from thirty point five percent to eight.

The update rhythm matters too. In Table 7, refreshing the inspectors on every batch lowers the average on the six unseen generators by two points, with Hailuo dropping the most, four point eight.

Here's an example. A woman presents a few product boxes. The appearance inspector reports that her face looks unusually smooth, almost rendered, with medium confidence. The investigator then calls the text inspector at the four-second mark, which zooms in and finds the repeated letter A on the boxes built inconsistently. Verdict: fake.

My read is that the real novelty here is letting the experts learn from being dispatched. Earlier agent detectors mostly trained the coordinator. These experiments show that putting the inspectors inside the training loop gains more than training either level alone.

### Limitations

The paper has no dedicated limitations section, so the points that follow are my reading of its results.

First, there's still a clear gap on unseen generators. In-domain it's close to ninety-eight, but on MintVid it scores eighty-two point nine on Jimeng, eighty-two point four on Sora 2, and eighty-five point six on Wan 2.5, which is below BusterX++ at eighty-seven point three.

Second, AgenticCue was built by the authors, its labels come mainly from vision-language models with human review, and the binary detectors in Table 4 were retrained on the authors' data. Independent evaluation is still to come.

Third, each video takes several rounds of calls, up to four. The paper reports how many calls are made, but I didn't find a figure for how much time or compute one video costs.

### Future work

The paper's first page links to code and model weights, and the conclusion stresses the value of evolving the investigator together with its inspectors.

My read is that the fast-and-slow idea isn't limited to forensics. Any system where a lead agent calls a set of trainable specialists could let those specialists learn from the lead agent's real calls.

My read is also that detection is an arms race. Scores in the low eighties on some unseen generators show that what this kind of system really has to prove is whether its inspectors can keep evolving as new models arrive.

That's today's Daily Frontier Papers. The paper is linked in the description.

