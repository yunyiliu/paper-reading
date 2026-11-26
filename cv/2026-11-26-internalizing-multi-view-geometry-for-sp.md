# Internalizing Multi-View Geometry for Spatial Intelligence

**大模型脑子里有没有3D房间**

- 论文 Paper: [arXiv 2611.28649](https://arxiv.org/abs/2611.28649)
- 单位 Affiliation: 南洋理工大学、香港科技大学等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-26
- 视频 Videos: [B站](https://www.bilibili.com/video/BV11f17zREwL)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇南洋理工大学、香港科技大学等单位的论文，叫 S-Mind，讲的是怎么让多模态大模型，在自己内部真正搭出一个三维场景。

先看论文开头这个例子。给你同一个房间的两张照片，一张对着窗户，一张对着门。问题是，站在沙发旁边，面朝书架，窗户在你的左边还是右边。人做这道题，会先把两张照片拼成一个房间，再在脑子里转到新的站位。作者说，空间智能靠的就是这两步，多视角融合，和视角变换。

### 问题

多模态大模型，也就是能看图又能对话的大模型，这两年进步很快。但具身智能、自动驾驶这类任务，要求模型在内部构建一个连贯的三维场景，而不只是认出二维图像里的图案。论文认为，这仍然是一个主要瓶颈。

那么，做过空间微调的模型，到底学会了没有。作者设计了一套空间探针，也就是在冻结的模型上接一个很小的读出器，直接看模型内部存着一个什么样的场景。

第一个发现，多视角融合很弱。在 Qwen3.5-4B 上，探针能恢复第一张图自己的 60.62% 的三维点，但其他图片的点，必须把几张图关联起来才能放对，只恢复了 11.23%。做了 SFT 和 GRPO 之后，也只到 14.19% 和 16.02%。

这在答案上也看得出来。MindCube-Tiny 里问，从第一张图到第二张图，相机是怎么移动的。SFT 和 GRPO 模型只答对 53.28% 和 56.57%，论文说，答错的题全部是把第二个相机放到了镜像的那一侧。

第二个发现，视角变换不完整。推理里声明了一个新视角之后，探针在这个视角下读出的场景，平均准确率只有 30.02%，而随机猜是 25%。论文的结论是，场景基本还是钉在相机上，没有跟着推理转到新的视角。

### 以前的方法

以前大致有两条路。第一条是外部几何，在模型外面做三维重建、渲染新视角，或者给模型外挂一个几何编码器。几何是显式的，但作者认为，空间能力并没有内化到模型里，而且推理延迟很高。

第二条是空间微调。有的只监督模型输出的文字，有的进一步让隐藏状态和三维几何对齐。作者的批评是，这些监督目标大多只编码输入视角本身的几何，视角发生变化这件事，没有被监督。

这张图把三种范式放在一起。外部几何，场景在模型外面。空间微调，模型内部有没有场景，不能保证。S-Mind 要做的，是把场景放进模型里面，而且能跟着推理转动。

### 核心想法

核心想法就两句话。第一，用一小组空间 token，装下融合好的三维场景。第二，推理每换一个视角，模型就把这组 token 重新输出一遍，让它们装着从新视角看到的同一个场景。

换句话说，视角变换被简化成，把同一个场景换一个参考坐标系重新表达。模型只要说出我现在站在哪、面朝哪，就等于转动了场景。

### 方法

先看整体流程图。左边是多视角融合，右边是视角变换，最下面一行是三个训练阶段。

第一步，多视角融合。给 N 张图，在图片后面接上空间 token，每张图 2 个场景 token，再加 4 个所有视角共享的相机 token。它们就在模型自己的序列里，后面的推理通过普通的注意力就能看到，不需要改模型结构。

怎么保证这些 token 真的装着场景。作者给每个视觉 token 分配它看到的那个三维点。一个只有一层交叉注意力的读出器，从空间 token 的隐藏状态里，解码出每个视觉 token 的三维点，和每个相机的位姿。读出器的查询只包含格点位置，不带图像内容，所以它只能读出空间 token 里本来就有的几何。

训练目标是让解码出来的点和位姿，回归到真实值。作者还故意做了冗余监督，同时监督第一张图坐标系下的全局点、每个相机自己坐标系下的局部点，和相机位姿。这三样里任意两样能推出第三样，这样就逼着解码结果构成一个前后一致的场景。

第二步，视角变换。推理链每声明一个新视角，模型就重新输出一组空间 token。训练时读出器冻结，从新 token 里解码出每个点，要求它等于原始场景经过刚体变换、换到新视角坐标系之后的位置。原始场景这一侧不回传梯度，所以这一项只监督场景怎么转，不去改感知。

训练分三个阶段。先在 ScanNet 和 MindCube 带位姿的图片组上做几何预训练。然后在推理链上做 SFT，交叉熵之外加上几何和视角两项损失，权重设为 0.1。推理链由 GPT-6-Astra 来写，每个视角写一句，开头先说站在哪、朝哪，空间 token 就插在这个短语后面。最后用 GRPO 强化学习，奖励只看最终答案对不对。

推理时，读出器和这些损失都不用，模型自己学会了在什么时候写出空间 token。

### 实验和效果

实验设置。骨干模型是 Qwen3.5 的 4B 和 9B。评测四个基准，MindCube-Tiny、SPAR-Bench、BLINK 和 ViewSpatial，报告每个的准确率，和四个的平均。

主结果在表 1。4B 版本经过强化学习后，平均 72.83%，基座模型是 50.57%，提高了 22.26 个点。比同样设置下训练的最强方法 ProSR 高 7.65 个点，比已发布的最好空间推理模型 Spatial-Interactor-4B 高 17.91 个点，在空间推理模型和同骨干的方法里，四个基准都排第一。

9B 版本平均 74.03%，比同骨干的 GRPO 高 6.76 个点。再看通用大模型，GPT-6-Astra 是 73.58%，Claude-Opus-5.5 是 74.15%，作者称 9B 版本和它们相当。

速度看表 2，对比对象是 SpatialClaw，一个同骨干、调用外部三维重建和分割工具的 Agent。在 MindCube-Tiny 上，S-Mind 每题 1.38 秒，SpatialClaw 要 111.74 秒，快 81 倍，准确率是 88.00% 对 56.95%。在 ViewSpatial 上快 115 倍。

再看模型内部到底学会没有。这张图是探针结果。融合后的点，S-Mind 恢复了 65.11%，基线最多 16.02%。声明新视角之后的场景，S-Mind 准确率 82.17%，基线在 25.22% 到 37.28% 之间。在需要关联两张图的 274 道题上，准确率从 SFT 和 GRPO 的 53.28% 和 56.57%，提高到 96.35%。

作者还做了因果干预。把最后一次重新输出的空间 token 状态，换成没转过的原始场景，有 9.05% 的答案会改变，大多是从对变错，而同样大小的随机扰动只改变 1.62%。

更有意思的是，推理时把空间 token 全部拿掉，平均只掉 2.37 个点，从 72.83% 到 70.46%，仍然比最强基线高 5.28 个点。作者据此认为，空间能力已经内化到了骨干模型里。

### 和以前方法的对比

和同样把几何训练进模型状态的 3DThinker 比，S-Mind 每张图监督的数值少 1600 倍，监督数据从 192 GB 降到 44 MB，几何模块参数少 19 倍，平均分还高 9.97 个点。

消融实验在图 8。只加空间 token、不加几何监督，准确率变化最多 1.24 个点，说明多几个 token、多一点计算，本身没什么用。加上几何监督提升最大，MindCube-Tiny 从 74.00% 到 81.81%。再加视角监督，BLINK 提到 75.00%。

我的判断是，这篇最值得看的是诊断加干预这一套。它没有只报准确率，而是用探针和状态替换，说明模型内部确实有一个会转动的场景，而且答案依赖它。

我的判断是，也要看清差距。表 1 里腾讯的 Hy4 平均 77.39%，比 S-Mind 9B 高 3.36 个点。而且 GPT-6-Astra 和 Claude-Opus-5.5 是在低思考强度下测的，所以和通用大模型相当这个说法，要打个折扣。

### 局限

论文没有单独的局限一节，下面几点来自文中的设置。第一，训练题目来自 MindCube、SpatialLadder 和 SPAR，评测里的 MindCube-Tiny 和 SPAR-Bench 出自同样的数据项目。4B 版本在 MindCube-Tiny 上从 46.29% 涨到 88.00%，在 BLINK 上从 68.25% 到 78.00%，在 ViewSpatial 上从 41.58% 到 54.27%。

第二，附录里说，每个模型只训练一次，附加分析也都是单个随机种子的单次运行，所以看不到多次训练之间的波动。

第三，作者在结论里自己承认，现在只监督输入视角看到的几何，看不到的区域和动态环境都还没有覆盖。

### 未来方向

作者提到的下一步，是让内部场景超出输入视角，补全看不到的区域，在多次交互中保持下来，跟着动态环境变化，让同一组 token 支持长时间运行的具身 Agent。

我的判断是，这条路线本质上是把三维重建从推理时挪到了训练时。推理时零工具调用、一次生成，对机器人、AR 眼镜这类对延迟敏感的场景很有吸引力，但能不能扩展到户外和动态场景，目前还没有证据。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Nanyang Technological University, HKUST and collaborators. It's called S-Mind, and it asks how a multimodal model can build an actual 3D scene inside itself.

Start with the example on the first page. You get two photos of the same room, one facing the window and one facing the door. Now stand by the sofa and face the shelf. Is the window on your left or your right? A person solves this in two moves: stitch the photos into one room, then mentally turn to the new spot. The authors call these the two cornerstones of spatial intelligence, multi-view fusion and perspective transformation.

### Problem

Multimodal language models, the ones that take images and text, have improved fast. But tasks like embodied AI and autonomous driving need the model to hold a coherent 3D scene, not just recognize 2D patterns. The paper calls this a major bottleneck.

So do spatially fine-tuned models really learn it? To find out, the authors build spatial probes: a tiny readout attached to a frozen model that decodes what scene the model is actually holding inside.

Finding one: fusion is weak. On Qwen3.5 4B, the probe recovers sixty point six two percent of the first image's own 3D points, but only eleven point two three percent of the points from the other images, the ones you can only place by relating the views. Fine-tuning with SFT or GRPO lifts that to just fourteen point one nine and sixteen point zero two percent.

It shows up in the answers too. On MindCube-Tiny questions about how the camera moved between two views, the SFT and GRPO models get only fifty-three point two eight and fifty-six point five seven percent right, and every error puts the second camera on the mirrored side.

Finding two: perspective transformation is incomplete. When the reasoning declares a new viewpoint, the scene read out in that viewpoint's frame is right only thirty point zero two percent of the time on average, against a chance level of twenty-five. The scene stays largely anchored to the camera instead of following the reasoning.

### Prior work

There have been two main approaches. The first is external geometry: reconstruct the scene in 3D outside the model, render new views, or bolt on a geometry encoder. The geometry is explicit, but the authors argue the model itself never learns the skill, and inference gets slow.

The second is spatial fine-tuning. Some methods supervise only the text the model writes; others align its hidden states with 3D geometry. The authors' critique is that these targets mostly encode the geometry of the input views. Changing viewpoint is never supervised.

This figure lines up the three paradigms. External geometry keeps the scene outside the model. Spatial fine-tuning gives no guarantee there's a scene inside at all. S-Mind aims to put the scene inside the model and have it turn as the reasoning moves.

### Key idea

The key idea fits in two sentences. First, a small set of spatial tokens holds the fused 3D scene. Second, every time the reasoning adopts a new viewpoint, the model emits those tokens again, and now they hold the same scene as seen from there.

Put differently, changing perspective becomes re-expressing one scene in a different frame of reference. The model just says where it's standing and which way it faces, and that turns the scene.

### Method

Here's the overall pipeline. Multi-view fusion on the left, perspective transformation on the right, and the three training stages along the bottom.

Step one, fusion. For N images, spatial tokens are appended after the images: two scene tokens per view, plus four camera tokens shared across views. They live in the model's own sequence, so later reasoning sees them through ordinary attention, with no change to the architecture.

How do you make sure they actually hold the scene? Every visual token is assigned the 3D point it observes. A readout with a single cross-attention layer decodes, from the spatial tokens' hidden states, the 3D point for each visual token and the pose of each camera. Its queries carry only grid positions, no image content, so it can only read out geometry the tokens already contain.

Training regresses those decoded points and poses onto the ground truth. And the supervision is deliberately redundant: global points in the first image's frame, local points in each camera's own frame, and the camera poses. Any two determine the third, which pushes the decoded result toward one consistent scene.

Step two, transformation. Each time the reasoning chain declares a viewpoint, the model emits a fresh run of spatial tokens. During training the readout is frozen, and the points decoded from the new tokens must match the original scene after a rigid transform into the new viewpoint's frame. Gradients don't flow into the original scene, so this term supervises only how the scene turns, not how it's perceived.

Training runs in three stages. Geometry pre-training on posed image sets from ScanNet and MindCube. Then SFT on reasoning chains, with cross-entropy plus the geometry and viewpoint losses at a weight of zero point one. The chains are written by GPT-6 Astra, one sentence per viewpoint, each opening with where the viewer stands and faces, and the spatial tokens go right after that phrase. Finally GRPO reinforcement learning, rewarded only on whether the final answer is right.

At inference, neither the readout nor the losses are used. The model has learned on its own when to write the spatial tokens.

### Results

The setup: Qwen3.5 backbones at 4B and 9B, evaluated on four benchmarks, MindCube-Tiny, SPAR-Bench, BLINK and ViewSpatial, with accuracy on each and the average of the four.

Main results are in Table 1. The 4B model after reinforcement learning averages seventy-two point eight three percent, up twenty-two point two six points from the base model's fifty point five seven. That's seven point six five points above ProSR, the strongest method trained in the same setting, and seventeen point nine one above Spatial-Interactor 4B, the best released spatial reasoning model. Among the spatial reasoning models and the methods on its backbone, it ranks first on all four benchmarks.

At 9B it averages seventy-four point zero three, six point seven six points above GRPO on the same backbone. For reference, GPT-6 Astra scores seventy-three point five eight and Claude Opus 5.5 seventy-four point one five, and the authors say the 9B model matches them.

Speed is in Table 2, against SpatialClaw, an agent on the same backbone that calls external 3D reconstruction and segmentation tools. On MindCube-Tiny, S-Mind takes one point three eight seconds per question versus about a hundred and twelve for SpatialClaw: eighty-one times faster, with eighty-eight percent accuracy versus fifty-seven. On ViewSpatial it's a hundred and fifteen times faster.

Did the model really learn it inside? This figure has the probe results. S-Mind recovers sixty-five point one one percent of the fused points, versus at most sixteen point zero two for the baselines. After a viewpoint is declared, it reads the scene correctly eighty-two point one seven percent of the time, versus between twenty-five and thirty-seven for the baselines. And on the two hundred seventy-four questions that require relating two views, accuracy goes from the mid fifties for SFT and GRPO to ninety-six point three five percent.

The authors also intervene causally. Replace the last run of re-emitted spatial tokens with the original, unturned scene, and nine point zero five percent of answers change, mostly from right to wrong. A random perturbation of the same size changes only one point six two percent.

More striking: drop the spatial tokens entirely at inference, and the average falls only two point three seven points, from seventy-two point eight three to seventy point four six, still five point two eight points above the best baseline. The authors read this as the skill being internalized in the backbone itself.

### Comparison

Against 3DThinker, which also trains geometry into the model's states, S-Mind supervises sixteen hundred times fewer values per image, cuts supervision data from a hundred ninety-two gigabytes to forty-four megabytes, uses a nineteen times smaller geometry module, and still scores nine point nine seven points higher on average.

The ablation is in Figure 8. Spatial tokens without geometry supervision move accuracy by at most one point two four points, so extra tokens and compute alone do little. Geometry supervision gives the biggest jump, MindCube-Tiny from seventy-four to eighty-one point eight one. Adding viewpoint supervision lifts BLINK to seventy-five.

My read is that the most valuable part here is the diagnose-then-intervene approach. Instead of only reporting accuracy, the paper uses probes and state swaps to show there's a scene inside the model that turns, and that the answers depend on it.

My read is also that the gap should be stated plainly. In Table 1, Tencent's Hy4 averages seventy-seven point three nine, three point three six points above S-Mind at 9B. And GPT-6 Astra and Claude Opus 5.5 were run at low thinking effort, so "on par with frontier models" deserves a discount.

### Limitations

The paper has no separate limitations section, so these come from its setup. First, training questions come from MindCube, SpatialLadder and SPAR, and two of the test sets, MindCube-Tiny and SPAR-Bench, come from the same data projects. At 4B, MindCube-Tiny jumps from forty-six point two nine to eighty-eight percent, while BLINK goes from sixty-eight point two five to seventy-eight, and ViewSpatial from forty-one point five eight to fifty-four point two seven.

Second, per the appendix, each model is trained once and the extra analyses are single runs with one seed, so no run-to-run variance is reported.

Third, the authors themselves note that they supervise only the geometry the input views observe. Unseen regions and dynamic environments aren't covered yet.

### Future work

The authors' next step is an internal scene that reaches beyond the input views: completing unseen regions, persisting across interactions, and evolving with dynamic environments, so the same tokens can support embodied agents over long horizons.

My read is that this line of work moves 3D reconstruction from inference time into training time. Zero tool calls and a single generation at inference is attractive for latency-sensitive settings like robots or AR glasses, but whether it extends to outdoor or dynamic scenes, there's no evidence yet.

That's today's Daily Frontier Papers. The paper is linked in the description.

