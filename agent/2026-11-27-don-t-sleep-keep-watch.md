# Don't Sleep, Keep Watch: Proactive Perception for Computer-Use Agents

**让小模型替Agent盯屏幕**

- 论文 Paper: [arXiv 2611.28947](https://arxiv.org/abs/2611.28947)
- 单位 Affiliation: 新加坡国立大学 · A*STAR CFAR · 商汤 · 香港浸会大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-27
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1Zm1xzNEtL)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇新加坡国立大学、新加坡科技研究局前沿人工智能研究中心、商汤和香港浸会大学合作的论文，题目叫 Don't Sleep, Keep Watch，别睡，盯着看。

它讲的是电脑操作 Agent，就是让大模型看着屏幕截图，替你点鼠标、敲键盘的那类 Agent。论文提出了一个叫 NightWatch 的框架，用一个只有 0.8B 参数的小模型，替大模型盯着屏幕。

### 问题

先说问题。现在几乎所有电脑操作 Agent，都是同一个循环，截一张图，想一想，做一个动作，然后固定睡一会儿，再截下一张。

可屏幕不会等它。页面在加载，弹窗会冒出来，通知闪一下就没了。论文引用的数据是，在 OSWorld 这个桌面基准上，从截图到动作真正执行，平均隔了 6.51 秒，这段时间里 Agent 是看不见屏幕的。

另一个问题是该睡多久。OSWorld 官方的各个 Agent 脚本里，动作之后的固定等待，从 0 秒到 10 秒都有。等短了，看到的是没渲染完的画面，等长了，就是白白空等。

作者把这归结为两种时间上的失误。一种是过期动作，照着旧截图想出来，执行时屏幕已经变了。另一种是观察太早或太晚，太早看到半成品，太晚就是空等。

### 以前的方法

已经有工作在两步之间盯屏幕了，但作者认为都是被动的。比如 AOI，在两步之间截帧，但只是附在下一次按时的观察里。

还有 PUSV，在点击之前再检查一次屏幕，变了就让 Agent 重新决定。但这时候 Agent 已经想完了，这一轮推理就白费了。

作者的总结是，这些方法都没有让屏幕来决定 Agent 什么时候看，也没法打断已经被屏幕变化作废的推理。论文第一张图把固定睡眠、被动观察和他们的主动感知放在了一起。

### 核心想法

核心想法就叫主动感知，proactive perception。不是 Agent 按自己的钟点看屏幕，而是由感知这一层决定什么时候该看，以及正在进行的推理和准备执行的动作，还算不算数。

具体是放一个很小的模型当守望者，持续看着屏幕，回答一组固定的问题，再用写死的规则，把答案变成三种干预。第一是打断，屏幕变化让正在想的东西作废了，就取消这次推理。第二是作废动作，要点的目标不在了，就不执行。第三是唤醒，屏幕一稳定就叫 Agent 来看，取代固定的睡眠。

这里有个分工。作者的观察是，小模型读状态读得准，做决定做得差，所以小模型只负责读屏幕，决定交给代码。它从不替 Agent 选动作，Agent 本身的模型、提示词、动作空间也都不动。

### 方法

先看守望者看什么。论文这张总览图，左边是监测部分。每次请求，守望者看四张图，Agent 上一次看到的截图、半秒前的画面、当前画面，还有变化区域的特写，再加上 Agent 准备执行的动作。

它要回答一份固定的题库，16 个问题，分四类。屏幕还在变吗，在加载吗，稳定了吗。有没有弹窗、报错或者新页面。要点的元素还在不在，有没有被挡住。上一个动作有没有看得见的效果。这些问题都不提具体应用和任务，所以题库和阈值定一次，到处通用。

守望者是一个 System-1 决策模型，不生成推理过程，直接给每个问题各个答案的概率。画面只编码一次，就能回答一批问题，而且平时只在画面明显变化时才发请求，静止的加载画面再定期查一下，所以很便宜。

三种干预的规则是这样。打断，是在 Agent 推理期间，屏幕每次明显变化，就问有没有新弹窗、报错或者换了页面，概率超过阈值，就取消生成，用当前屏幕重来。

作废，是在动作发出去之前，问一次目标还在不在、有没有被挡住，超过阈值就丢掉这个动作，同样用当前屏幕重来。

唤醒，是在动作执行之后，先等像素停下一小段时间，再问守望者，画面是真稳定了，还是暂时停住。确认稳定、不在加载，就马上让 Agent 看下一张截图，每一步还有最长等待时间兜底。

再说训练。NightWatch-0.8B 从一个 0.8B 的 System-1 决策模型 d3-lite 出发，用 LoRA 微调。训练用的是 360 个新生成的任务，覆盖 OSWorld 的十类应用，和评测任务太像的都剔除了，不用任何评测任务、文件或轨迹。

标签一部分来自更大的老师模型，一部分来自录下的画面能直接确定的事实，两者冲突时，以像素事实为准。弹窗、报错这类少见事件，再用 15000 个合成界面样本来补。

还有个细节。稳定这个标签不看过去安静了多久，而是看未来 1 秒画面有没有再变，因为安静不等于完成，页面可能在加载中途停一下。作者解释说，如果按已经安静了多久来标，模型永远不可能比单纯等像素静止更早唤醒。

### 实验和效果

实验在两个环境上做。桌面是 OSWorld-Verified，随机抽了 100 个任务。动态网页是 DynaCU-Bench 里不需要声音的 49 个任务，比如实时仪表盘、轮播图和小游戏。每个任务跑三次。

Agent 用了两个，都不改动，大的是 MiMo-V2.6-Flash，小的是开放权重的 Qwen3.8-27B。所有方法都只拿截图，对比官方的固定等待循环，以及 AOI 和 PUSV。守望者训练完就冻结，两个基准用同一套阈值。

先看表 1。在 OSWorld 上，用大模型 Agent，NightWatch 成功率 68.0%，原版循环 66.3%，PUSV 67.3%，基本持平。但推理以外的时间，也就是总时间减去大模型思考的时间，每个任务从 197.4 秒降到 132.7 秒，少了 33%，浪费时间的占比从 16.3% 降到 7.7%。

小模型 Agent 也一样，推理以外的时间从 190.2 秒降到 111.9 秒，少了 41%。论文报告，这两个下降在按任务配对的检验里都是显著的。

时机上的失误也少了。大模型 Agent 在 OSWorld 上，过早观察的步数占比从 9.6% 降到 2.6%，过期动作从 17.4% 降到 8.3%。

到了动态网页，变化体现在成功率上。大模型 Agent 从 39.5% 提高到 44.2%，高了 4.7 个百分点，也高于 AOI 的 37.4%。小模型 Agent 和原版持平，都是 34.7%。

成本看表 2。用大模型 Agent，端到端每个任务从 666.0 秒降到 614.7 秒，API 费用和原版相差不到 3%，而 AOI 因为多塞关键帧，费用涨了 52%。守望者跑在一张 L40S 显卡上，不调用 API。

论文给了具体例子。一个服务器监控页面，Agent 还在想，1.1 秒后又一台服务器变红，推理被打断，Agent 重新看屏幕，及时确认了新告警。

一个颜色匹配小游戏，Agent 花了 5.8 秒才发出点击，它选的青色色块已经转走了，这次点击被丢掉，不然会点到橙色上，Agent 在新画面上重新点，完成了任务。

### 和以前方法的对比

再看表 3，只换做判断的那一层。纯像素规则成功率 30.6%，闭源 API 守望者 36.7%，大 30 多倍的 D3-27B 是 32.7%，每次调用还要 7.6 秒。NightWatch-0.8B 是 44.2%，用时也最短。

表 4 每次去掉一种干预。去掉打断，成功率掉到 30.6%，去掉作废，36.1%，把唤醒换成等像素静止，38.1%。每去掉一种，任务都变慢。

在 OSWorld 上，打断和作废很少触发，收益主要来自唤醒。和只等像素静止的版本比，这组只跑了一次，NightWatch 推理以外的时间少了 37%。作者的结论是，关键在判断这份安静是不是最终状态，而不是等它安静够久。

我的判断是，这篇的价值在于，把什么时候看，从一个手调的参数，变成了一个可以学的感知问题，而且不用动 Agent 本身。

我的判断是，也要看清边界。OSWorld 上成功率的差距在误差范围内，68.0 加减 3.6 对 66.3 加减 4.5，在桌面上它省的是时间，不是成功率。论文主打的成功率提升，是动态网页加大模型 Agent 这一组，而论文只对时间做了显著性检验，没有对成功率做。

### 局限

作者自己写了局限。第一，NightWatch 改变的是 Agent 什么时候看，不是它能做什么。OSWorld 上有 11 个任务，两种方法每次都失败，大多是 Agent 自己带着错误结果结束了任务。

第二，打断和作废只能让 Agent 重新想，不能让它想得更快。在 DynaCU-Bench 上两种方法都全败的任务里，Agent 每步光生成就要 9.1 秒，提示条和游戏状态往往变得比这还快。所以作者说，NightWatch 适合变化比 Agent 思考更慢的界面。

还有，在 OSWorld 上，作者统一用了官方的纯截图 Agent，成绩比厂商公布的低，作者说明，厂商的成绩来自各自没有公开的 Agent 框架和推理设置。

### 未来方向

论文没有单独写未来方向。作者在结论里说，感知和动作之间的时间差，是电脑操作 Agent 的核心瓶颈之一，主动感知是一个实用的办法。代码、数据和模型权重都给出了链接。

我的判断是，下一步值得看手机。论文开头引用的数据是，安卓上从观察到执行能隔 4 到 15 秒多，但实验只做了桌面和网页，守望者的训练数据也只有桌面轨迹和合成网页。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the National University of Singapore, A-STAR's Centre for Frontier AI Research, SenseTime, and Hong Kong Baptist University. The title is a good summary: Don't Sleep, Keep Watch.

It's about computer-use agents, the kind where a large model looks at screenshots and clicks and types its way through a desktop or a website. The paper introduces NightWatch, a framework that hands the job of watching the screen to a small model with just zero point eight billion parameters.

### Problem

Here's the problem. Nearly every computer-use agent runs the same loop. Take a screenshot, think, act, sleep for a fixed time, and repeat.

But the screen doesn't wait. Pages load, dialogs pop up, notifications flash and vanish. The paper cites a measurement on the OSWorld desktop benchmark: from screenshot to executed action takes six point five one seconds on average, and for that whole stretch the agent is blind.

Then there's the question of how long to sleep. Across OSWorld's official agent scripts, the pause after each action ranges from zero to ten seconds. Too short, and you capture a half-rendered screen. Too long, and you just sit there idle.

The authors boil this down to two timing failures. One is the stale action: it was planned on an old screenshot, and by the time it runs, the screen has changed. The other is looking too early or too late. Too early shows you a transition, too late wastes time.

### Prior work

Some recent systems do watch the screen between steps, but the authors call that watching passive. AOI, for example, samples frames between steps, but only attaches them to the next scheduled observation.

PUSV checks the screen once, right before a click, and asks the agent to decide again if something changed. But by then the agent has already finished thinking, so that whole round of reasoning is thrown away.

The authors' summary: none of these lets the screen decide when the agent looks, and none can stop reasoning that the screen has already made obsolete. The first figure puts fixed sleeping, passive watching, and their proactive approach side by side.

### Key idea

The key idea is proactive perception. Instead of the agent looking on its own clock, the perception layer decides when to look, and whether the reasoning in progress and the action about to run are still valid.

Concretely, a tiny model acts as a watcher. It keeps looking at the screen, answers a fixed set of questions, and hand-written rules turn those answers into three interventions. Interrupt: if a screen change makes the current reasoning obsolete, cancel it. Void: if the target of the pending action is gone, don't execute it. Wake: as soon as the screen settles, let the agent look, instead of sleeping a fixed time.

There's a deliberate division of labor here. The authors observe that small models read states reliably but decide poorly, so the watcher only reads the screen, and code makes the decisions. It never chooses actions for the agent, and the agent's model, prompt, and action space stay untouched.

### Method

First, what the watcher sees. In this overview figure, the left side is the monitoring part. Each request shows the watcher four images: the agent's last screenshot, the frame from half a second ago, the current frame, and a close-up of the region that changed, plus the text of the action the agent is about to take.

It answers a fixed bank of sixteen questions in four groups. Is the screen still changing, still loading, or settled? Has a dialog, an error, or a new page appeared? Is the target element still there and unobstructed? Did the last action have a visible effect? No question names an app or a task, so the bank and its thresholds are set once and reused everywhere.

The watcher is a System-1 decision model. It doesn't write out reasoning; it returns a probability for each answer. The image is encoded once for a whole batch of questions, and requests go out mainly when the frame changes noticeably, plus a periodic check of a static loading screen, so it's cheap.

Now the three rules. Interrupt: while the agent is reasoning, every significant screen change triggers the event questions, a new dialog, an error, a page change. If the probability crosses a threshold, generation is cancelled and the step restarts from the current screen.

Void: just before dispatch, the watcher checks whether the target is still present and unobstructed. If that crosses its threshold, the action is discarded, and again the step restarts from the current screen.

Wake: after the action runs, the harness first waits for pixels to stop changing for a moment, then asks the watcher whether that stillness is final or just a pause. If the screen is settled and not loading, the agent gets its next screenshot right away, and a maximum wait bounds every step.

Training. NightWatch zero point eight B starts from d3-lite, a zero point eight billion parameter System-1 decision model, fine-tuned with LoRA. The training set is three hundred and sixty newly generated tasks across OSWorld's ten application domains, with anything too close to an evaluation task removed. No benchmark task, file, or trajectory is used.

Labels come partly from a larger teacher model and partly from facts the recorded frames settle directly, and when a pixel fact disagrees with the teacher, the fact wins. Rare events like dialogs and errors are filled in with fifteen thousand examples from synthetic interfaces.

One more detail. The settledness label looks forward, not back: a frame counts as settled if nothing changes over the next second, because quiet doesn't mean done, and a page can pause mid-load. As the authors put it, labeling by how long the screen has already been quiet could never wake the agent earlier than a simple pixel-stillness rule.

### Results

Two test environments. On the desktop, a random sample of one hundred OSWorld-Verified tasks. On the dynamic web, the forty-nine DynaCU-Bench tasks that don't need audio: live dashboards, carousels, games. Every task is run three times.

Two agents, both unmodified: the large MiMo-V2.6-Flash and the smaller open-weight Qwen3.8-27B. Every method gets screenshots only, and the comparison is against the official fixed-wait loop, plus AOI and PUSV. The watcher is frozen after training and uses the same thresholds on both benchmarks.

Table 1. On OSWorld with the large agent, NightWatch succeeds sixty-eight point zero percent of the time, against sixty-six point three for the plain loop and sixty-seven point three for PUSV. Basically on par. But time outside reasoning, meaning wall-clock time minus the large model's thinking time, drops from one hundred ninety-seven point four seconds per task to one hundred thirty-two point seven. That's thirty-three percent less. The share of wasted time falls from sixteen point three percent to seven point seven.

The small agent shows the same pattern: time outside reasoning falls from one hundred ninety point two seconds to one hundred eleven point nine, forty-one percent less. The paper reports both drops as significant in a paired test over tasks.

Timing errors drop too. With the large agent on OSWorld, premature observations go from nine point six percent of steps to two point six, and stale actions from seventeen point four to eight point three.

On the dynamic web, the gain shows up in success. The large agent goes from thirty-nine point five percent to forty-four point two, up four point seven points, and also above AOI's thirty-seven point four. The small agent stays level with the plain loop, at thirty-four point seven.

Costs are in Table 2. With the large agent, the full task time drops from six hundred sixty-six seconds to six hundred fourteen point seven, and API cost stays within three percent of the plain loop, while AOI, which stuffs extra keyframes into every call, costs fifty-two percent more. The watcher runs on a single L40S GPU and makes no API calls.

The paper walks through examples. On a server dashboard, the agent is still reasoning when, one point one seconds later, another server turns red. The reasoning is interrupted, the agent looks again, and it acknowledges the new alarm in time.

In a color-matching game, the agent takes five point eight seconds to emit its click, and the cyan swatch it picked has already rotated away. The click is discarded, so it doesn't hit the orange one, and the agent clicks again on the fresh screen and solves the task.

### Comparison

Table 3 swaps out only the decision layer. Pixel rules reach thirty point six percent success, a closed API watcher thirty-six point seven, and D3-27B, more than thirty times larger, thirty-two point seven, while taking seven point six seconds per call. NightWatch zero point eight B reaches forty-four point two, and it's also the fastest.

Table 4 removes one intervention at a time. Without interrupt, success falls to thirty point six. Without void, thirty-six point one. Replacing wake with waiting for pixel stillness, thirty-eight point one. Every removal makes tasks slower.

On OSWorld, interrupts and voids rarely fire, so the gain comes mostly from waking. Against a version that wakes on pixel stillness alone, in a single run, NightWatch spends thirty-seven percent less time outside reasoning. The authors' conclusion: what matters is judging whether the stillness is final, not waiting for it to last.

My read is that the value here is turning when to look from a hand-tuned constant into a perception problem a model can learn, without touching the agent itself.

My read is also that the limits matter. On OSWorld, the success difference sits inside the error bars, sixty-eight point zero plus or minus three point six versus sixty-six point three plus or minus four point five. On the desktop it saves time, not success. The headline success gain is in one setting, the dynamic web with the large agent, and the paper tests significance only for time, not for success.

### Limitations

The authors list the limits themselves. First, NightWatch changes when the agent looks, not what it can do. Eleven OSWorld tasks fail in every run for both methods, and mostly the agent ends the task itself with a wrong result.

Second, interrupting and voiding make the agent rethink, not think faster. On the DynaCU-Bench tasks where both methods always fail, the agent spends nine point one seconds of generation per step, and toasts and game states often change faster than that. So the authors say NightWatch suits interfaces that change more slowly than the agent reasons.

Also, on OSWorld every method runs on the official screenshot-only agent, which scores below what model vendors report. The authors note that vendor scores come from each vendor's own agent harness and reasoning settings, which aren't public.

### Future work

There's no separate future-work section. In the conclusion, the authors call the gap between perception and action a central bottleneck for computer-use agents, and proactive perception a practical remedy. Links to the code, data and weights are in the paper.

My read is that phones are the next thing to watch. The paper opens with a cited figure of four to over fifteen seconds from observation to execution on Android, but the experiments cover only desktop and web, and the watcher's training data is desktop trajectories plus synthetic web pages.

That's today's Daily Frontier Papers. The paper is linked in the description.

