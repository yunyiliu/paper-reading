# Any Time, Any Space: A World-State-Centric Framework for Agentic Video Generation

**AI拍长片记住去过的场景**

- 论文 Paper: [arXiv 2610.20571](https://arxiv.org/abs/2610.20571)
- 单位 Affiliation: 浙江大学 等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-17
- 视频 Videos: [YouTube 中文](https://youtu.be/LFoOfN2lLCc) · [YouTube English](https://youtu.be/jzSLlWC7oTc) · [B站](https://www.bilibili.com/video/BV1jcJe6uEhe)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇浙江大学牵头的论文，合作方还有莫纳什大学、加州大学伯克利分校和阿德莱德大学，名字叫 Any Time, Any Space，简称 ATAS。

它想解决的问题是，用 AI 拍一部多镜头的短片时，怎么让后面的镜头记得前面发生过什么，回到同一个房间，房间还是原来的样子。

### 问题

先说背景。视频生成模型一次只能生成一段比较短的片子，要拍一个有情节的故事，就得有一个 Agent，把剧本拆成很多个镜头，一个个生成，再接起来，这叫 Agent 式视频生成。

难点在于，故事会跳来跳去。比如一个人在房间里做了一件事，镜头切到别的地方，过一会儿又回到这个房间，或者换一个角度，重放刚才那件事。

回来的镜头，房间布局得和之前一样，重放的镜头，动作和先后顺序也得和之前一样。这已经不只是单个片段画质好不好的问题了。

### 以前的方法

以前的 Agent 视频系统，靠什么保持连贯呢。通常是把最近几帧、一段文字摘要，或者检索出来的参考图，传给下一个镜头。

对紧挨着的两个镜头，这基本够用。但这些线索既没有共享的空间坐标，也没有按时间编号的记录，所以隔了好几个镜头之后，再想回到某个场景，或者重放某个事件，就没有依据了。

另一条线是带三维记忆的视频生成，用点云这类显式几何来约束画面。作者认为，这类方法有几何记忆，但没有按事件和时间来查的接口，也没有规定生成出来的内容什么时候可以写回记忆。

### 核心想法

ATAS 的核心想法是，给整部片子维护一个持续存在的世界状态。它分两块，一块是场景记忆，在共享坐标系里存静态的空间结构，另一块是事件记忆，存动态的画面，并且记下它属于哪个场景、哪个事件、哪段时间。

第二个关键点是，生成出来的片段，不直接当成真相，而是当成一份候选观测，先检查，过关了才写进世界状态。这样生成里的错误，就不会一路传到后面的镜头。

还有一个细节，片段能不能放进成片，和它能不能写进记忆，是两个分开的决定。

论文第一张图画的就是这个思路。底下的胶片是成片，背后是同一个共享的场景，不同镜头从不同位置拍它，后面的镜头可以回来复用这些空间和时间信息。

### 方法

具体流程在这张图里，从左到右，是编译、生成和校验、两种记忆。

第一步是编译。编译器把剧本递归地往下拆，按场景、时空关系、事件引用和镜头要求一层层拆，直到每个请求都能用一次短视频生成完成。

拆出来的每个镜头请求，包含六样东西，镜头描述、哪个场景、在场景里的空间位置、目标机位、时间区间，以及一个可选的事件编号。

第二步是查记忆再生成。场景记忆里存的是静态点云，按请求的机位重新渲染出一张参考画面。如果请求提到了之前的某个事件，就按场景、事件和时间，从事件记忆里把对应的动态记录取出来。

这些参考和镜头描述一起交给视频生成模型。论文说明，这个接口在所有对比里保持不变。

第三步是校验和写回。生成的片段先过成片审核，不合格就重试。合格之后，系统把它重建成三维，和场景坐标对齐，对齐成功，再检查来源、场景范围和状态版本，几关都过了，才更新世界状态。

如果对齐失败，这个片段照样可以留在成片里，只是不会被后面的镜头当作依据。

这张图是两个故事的例子，一行是成片，一行是同步攒下来的三维世界，可以看到场景和人物的记录，随着镜头一点点扩展。

### 实验和效果

实验用了四个视频生成模型做底座，MiniMax-H3、Wan 2.7、HappyHorse 1.0 和 Wan 3.0，表里把第一个归为开源，后三个归为闭源。对比的基线有五个，一个是简单的图生视频续写，另外四个是 Agent 系统，VGoT、Mora、MovieAgent 和 ReCA。

评测用的是 VBench、ST-Bench 和 ViStory 这几个公开基准。同一个底座下，所有方法共用同样的生成器、提示词和时长预算。

结果在表 1。在 MiniMax-H3、HappyHorse 1.0 和 Wan 3.0 上，ATAS 每一项汇总指标都是第一。以 Wan 3.0 为例，平均分从最强基线 ReCA 的 0.648，提到 0.734。

唯一的例外是 Wan 2.7。在偏重单个片段质量的 VBench 上，ATAS 略低于 ReCA，0.846 比 0.854，但在 ST-Bench 和两项 ViStory 一致性指标上，仍然领先。

再看两个专门的测试。第一个叫空间回访，就是回到之前的场景，把生成的画面和目标图对比。ATAS 的 PSNR 是 15.968，高于专门做三维引导视角合成的 ViewCrafter 的 15.255，三项指标都是最好。

第二个叫事件重放，换一个机位，重拍之前发生过的事，由一个视觉语言模型对照原始事件打分。五项里，ATAS 在事件识别、时间顺序和视角实现三项领先，人物物体一致性和 ReCA 并列，动作状态略低，0.688 比 0.700。

人工评测也是这个结果。评审看完整个视频，按六项标准打分，满分 5 分，ATAS 六项都是最高。脚本忠实度 4.40，人物一致性 4.22，最强基线 ReCA 对应是 3.35 和 3.67。

消融实验在表 5，固定 Wan 3.0，逐步加记忆。从没有记忆到完整状态，跨镜头一致性指标 C-Top10 从 0.536 升到 0.719，事件重放分从 0.452 升到 0.739，其中事件记忆对事件重放的帮助最大。

另一个消融在表 4，把组件一个个加上去。递归拆解剧本这一步，让事件重放分从 0.508 跳到 0.711，是单步最大的提升。

### 和以前方法的对比

和以前的 Agent 系统比，区别不在用哪个视频模型，而在记忆怎么组织。以前传的是提示词、参考图或者生成历史，ATAS 存的是有坐标的场景和有时间戳的事件，能按地点和时间去查。

代价论文也量了，在图 6。用上完整记忆，每个镜头的重试次数从 1 次降到 0.54 次，算上重试的交付时间，从每个镜头 359 秒降到 339 秒。但每个故事的总耗时，从 51.9 分钟涨到 64.9 分钟，每小时能出的镜头数，从 7.28 降到 6.21。

作者自己的总结是，好处是重试更少，交付更快，而不是总算力更省。

我的判断是，这篇的价值，是把视频 Agent 的记忆，从一堆参考素材，变成了一个可以按地点和时间查询的小世界。

我的判断是，这组对比也要打个折扣。最强的基线 ReCA，从参考文献看，作者和这篇有很多重叠，是同一个团队之前的工作，外部的 Agent 对手只有 VGoT、Mora 和 MovieAgent。

### 局限

论文没有单列局限一节，但作者在正文里写了，事件重放剩下的难点，是把动作和状态精确地再现出来。人工评测里，背景一致性 3.62 和叙事连贯 3.77，也是 ATAS 六项里最低的两项。

第二，记忆会过时。离上一次成功写回记忆隔的镜头越多，质量越差，隔 5 个以上镜头时，C-Top10 从 0.736 降到 0.651。作者也说，还分不清这是场景变化、事件太密，还是对齐误差累积造成的。

第三，事件重放的打分靠视觉语言模型，作者承认，它只能判断和原始事件是否吻合，做不到像素级的比对。

它和人工打分的相关系数在 0.41 到 0.61 之间，低于人和人之间 0.67 到 0.80 的一致程度。

第四，写回检查的松紧要权衡。默认设置接受 67.4% 的候选更新，放得太松或卡得太严，效果都会变差，作者也说，这个默认值不代表普遍最优。

另外，附录里的几部长片例子，作者也说明，展示的是生成上的连贯，不是对没拍到的那段时间做物理模拟。

### 未来方向

作者在结论里说，因为地点和之前的事件在整个故事里都能随时查到，ATAS 可以作为 Agent 视频工具的基础，用在交互式电影制作、连续剧集内容和预演上。

论文首页给了代码仓库的地址。

附录里还提到，身份保持还有提升空间。

状态检查也只是一个控制写入的机制，代替不了更好的重建和检索。

我的判断是，长视频生成的瓶颈，正在从画质转向记忆。把生成的内容当成待验证的观测，而不是直接当成真相，这个思路对做视频 Agent 的人很值得借鉴。

我的判断是，也要看到，回访画面的 PSNR 只有 16 左右，离精确还原还远，而且总耗时变长了，要用在实际制作里，成本还得降下来。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by Zhejiang University, with Monash, UC Berkeley and Adelaide University. It's called Any Time, Any Space, or ATAS.

The question: when AI films a story shot by shot, how do later shots remember what happened earlier? If a character walks back into a room, the room should still look like the same room.

### Problem

Some background. A video generator only produces a short clip at a time. To tell a story, an agent breaks the screenplay into many shots, generates them one by one, and strings them together. That's agentic video generation.

The hard part is that stories jump around. Someone does something in a room, the film cuts elsewhere, and later it comes back to that room, or replays the earlier moment from a different angle.

The return shot has to keep the room's layout. The replay has to keep the same actions in the same order. That goes beyond how good any single clip looks.

### Prior work

So how do existing agent systems keep things consistent? Usually by passing recent frames, a text summary, or retrieved reference images to the next shot.

For back-to-back shots, that mostly works. But none of those cues give you a shared spatial frame or a time-indexed record. Once several shots have passed, there's nothing to anchor a return to a place or a replay of an event.

A second line of work gives video generation an explicit 3D memory, like point clouds. The authors argue these methods offer geometric memory, but no way to look things up by event and time, and no rule for when generated content is allowed back into memory.

### Key idea

ATAS keeps a persistent world state for the whole film, in two parts. A scene memory holds static spatial structure in a shared coordinate frame. An event memory holds dynamic observations, tagged with their scene, their event and their time.

The second idea: a generated clip is not treated as truth. It's a candidate observation. It gets checked, and only if it passes does it update the world state, so mistakes in one clip don't spread to later shots.

And one more detail. Whether a clip goes into the film, and whether it gets written into memory, are two separate decisions.

That's what the first figure shows. The film strip at the bottom is what viewers see. Behind it is one shared scene, filmed from different camera positions, and later shots can come back and reuse that spatial and temporal information.

### Method

Here's the pipeline, left to right: compilation, generation with validation, and the two memories.

Step one is compilation. The compiler recursively splits the screenplay, by scene, by spatial and temporal relations, by event references and by camera needs, until every request fits in a single short-video generation.

Each shot request carries six things: a shot description, the scene, a spatial anchor inside that scene, a target camera, a time interval, and an optional event ID.

Step two is to query memory, then generate. The scene memory stores static point clouds, which are re-rendered from the requested camera as a scene reference. If the request points to an earlier event, the matching dynamic records are pulled from event memory by scene, event and time.

Those references go to the video generator along with the shot description. The paper says this interface is kept fixed across all the comparisons.

Step three is validation and write-back. The clip first goes through film review, with retries if it fails. If it's accepted, it's reconstructed in 3D and registered to the scene's coordinates. If that works, a final check looks at provenance, scene scope and state version. Only then does the world state update.

If registration fails, the clip can still stay in the film. It just won't be used as evidence for later shots.

This figure shows two stories. One row is the film, the other is the 3D world built up alongside it, with the scene and the people recorded as the shots go on.

### Results

The experiments use four video generators as backbones: MiniMax-H3, Wan 2.7, HappyHorse 1.0 and Wan 3.0. The table lists the first as open source and the other three as proprietary. There are five baselines: a plain image-to-video extension, and four agent systems, VGoT, Mora, MovieAgent and ReCA.

Evaluation uses public benchmarks: VBench, ST-Bench and ViStory. Within each backbone, every method shares the same generator, prompts and duration budget.

The results are in Table 1. On MiniMax-H3, HappyHorse 1.0 and Wan 3.0, ATAS leads every aggregate score. On Wan 3.0, the average goes from zero point six four eight for ReCA, the strongest baseline, to zero point seven three four.

The one exception is Wan 2.7. On VBench, which leans toward single-clip quality, ATAS is slightly behind ReCA, zero point eight four six versus zero point eight five four. It still leads on ST-Bench and both ViStory consistency scores.

Then two targeted tests. The first is spatial revisit: go back to an earlier scene and compare the generated view with a target image. ATAS reaches a PSNR of fifteen point nine seven, above ViewCrafter, a 3D-guided view-synthesis method, at fifteen point two six, and it's best on all three metrics.

The second is event recapture: re-film an earlier event from a new camera, scored by a vision-language model against the original event. Of five criteria, ATAS leads three, event identity, temporal order and viewpoint. It ties ReCA on character and object consistency, and is slightly lower on action and state, zero point six eight eight versus zero point seven.

Human raters agree. They watched whole videos and scored six criteria on a five-point scale, and ATAS has the top mean on all six. Script faithfulness is four point four and character consistency four point two two, against three point three five and three point six seven for ReCA.

The memory ablation is Table 5, with Wan 3.0 fixed. Going from no memory to the full state, the cross-shot consistency score C-Top10 rises from zero point five three six to zero point seven one nine, and the event recapture score from zero point four five two to zero point seven three nine. Event memory gives the biggest boost to recapture.

The component ablation is Table 4, adding parts one at a time. Recursive screenplay compilation lifts the event score from zero point five zero eight to zero point seven one one, the largest single step.

### Comparison

Compared with earlier agent systems, the difference isn't which video model you use. It's how memory is organized. They pass along prompts, reference images or generated history. ATAS stores scenes with coordinates and events with timestamps, and looks them up by place and time.

The cost is measured too, in Figure 6. With full memory, retries per shot fall from one to zero point five four, and delivery time including retries drops from three hundred fifty-nine to three hundred thirty-nine seconds per shot. But total runtime per story rises from fifty-one point nine to sixty-four point nine minutes, and throughput falls from seven point two eight to six point two one shots per hour.

The authors' own summary: the benefit is fewer retries and faster delivery, not less total compute.

My read is that the contribution here is turning a video agent's memory from a pile of reference material into a small world you can query by place and time.

My read is also that the comparison deserves a discount. The strongest baseline, ReCA, shares many authors with this paper according to its reference entry, so it's the same group's earlier work. The outside agent baselines are just VGoT, Mora and MovieAgent.

### Limitations

There's no dedicated limitations section, but the authors do say the remaining difficulty in event recapture is reproducing actions and states exactly. And in the human study, background consistency at three point six two and narrative coherence at three point seven seven are ATAS's two lowest scores.

Second, memory goes stale. The more shots since the last accepted update, the worse the quality. After five or more, C-Top10 drops from zero point seven three six to zero point six five one. The authors say they can't yet tell whether that comes from scene changes, event density, or accumulated registration error.

Third, event recapture is scored by a vision-language model, and the authors acknowledge it can only judge agreement with the source event, not pixel-level agreement.

Its correlation with human ratings ranges from zero point four one to zero point six one, below the zero point six seven to zero point eight agreement between humans.

Fourth, the write-back check is a trade-off. The default setting accepts sixty-seven point four percent of candidate updates. Too loose or too strict both hurt, and the authors say the default isn't universally optimal.

And the long films in the appendix, the authors note, show generated continuity, not a physical simulation of what happened off camera.

### Future work

In the conclusion, the authors say that because places and past events stay addressable throughout a story, ATAS could be a foundation for agentic video tools in interactive filmmaking, episodic content and previsualization.

A code repository is linked on the paper's first page.

The appendix also notes there's room to improve identity preservation.

And the state check is only a way to control what gets written. It doesn't replace better reconstruction and retrieval.

My read is that the bottleneck in long video generation is moving from image quality to memory. Treating generated content as an observation to verify, rather than as truth, is an idea worth borrowing for anyone building video agents.

My read is also that a revisit PSNR of around sixteen is still far from exact reconstruction, and total runtime went up. Cost has to come down before this is practical for real production.

That's today's Daily Frontier Papers. The paper is linked in the description.

---

_文字稿即视频讲解稿，发布前已对照论文原文独立核查；由 AI（Claude）撰写。 The text is the video script, checked against the paper before release; written with AI assistance (Claude)._
