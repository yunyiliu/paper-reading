# Few-Step Streaming Video Generation via Condition Shifting

**视频蒸馏少养一个网络**

- 论文 Paper: [arXiv 2611.14033](https://arxiv.org/abs/2611.14033)
- 单位 Affiliation: 清华大学 · 生数科技 · 西湖大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-12
- 视频 Videos: [YouTube 中文](https://youtu.be/6Mm0N0YTW3M) · [YouTube English](https://youtu.be/IpSsjpUyaak) · [B站](https://www.bilibili.com/video/BV1Z3Wd61EEf)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇清华大学、生数科技和西湖大学的论文，题目翻过来是，用条件偏移做少步数的流式视频生成，方法叫 APEX。

它要解决的问题是，把视频模型蒸馏成几步就能出画面的时候，能不能少训练一个网络，让训练本身更省。

### 问题

先说背景。流式视频生成，就是视频一块一块往下生成，前面生成的内容，就是后面的历史。

扩散模型和 flow 模型画质好，但每出一块画面，网络都要跑很多遍，流式生成每一块都要付这个成本，所以把步数降下来特别有价值。蒸馏能减少步数，但蒸馏出来的模型，还得保住画质，保住人物和场景，还要跟得上不断变化的指令。

### 以前的方法

现在常用的一种蒸馏方法叫 DMD，分布匹配蒸馏。它用两个网络来打分，一个是冻结的老师，代表真实数据该是什么样，另一个叫 fake-score 网络，专门学习学生当前生成的东西是什么样，两者的差，告诉学生该往哪改。

问题出在这个额外的 fake 网络。对大型视频模型来说，多维护一个辅助网络，训练时更占显存，也更费算力。而且学生一直在变，这个网络也得跟着学，两边的优化设置和更新节奏都要协调。

这篇的基线叫 S2-Avatar，来自 Vidu S2。它的训练方法叫 Self-Replay Forcing，自我回放，模型先自己生成一长段，再从里面截一段，重新加噪后训练，让模型在自己生成的历史上学习，里面纠正学生用的就是 DMD。

### 核心想法

核心想法是，不要单独的 fake 网络了，让生成器自己兼任，同一套参数，靠输入的文本条件来切换角色。

具体说，用原来的文本条件，模型负责生成，把文本条件取一个负号，模型就切换成 fake 估计器，去学它自己生成的视频是什么分布。这个技巧叫条件偏移，condition shifting，来自之前的 APEX 工作，这篇把它推广到了流式视频。

冻结的老师仍然保留，提供参考方向。生成和 fake 估计两个角色共享模型参数，默认连 AdamW 优化器的状态也共享。

### 方法

看这张图。上面是 DMD，生成器、冻结老师和 fake 估计器各有一套参数。下面是 APEX，fake 估计器和生成器用的是同一套参数，只是输入的文本变成了负的。

训练时两步交替。第一步，把生成出来的视频块截断梯度，重新加噪，再配上取负的文本条件，用标准的 flow matching 损失，去拟合这些生成视频的速度场，这一步只更新 fake 估计这个角色的预测。

第二步，在同一段加噪视频上，分别让冻结老师和 fake 估计器去预测干净的画面，两个预测的差，就是生成器该往哪改的方向。

这个差归一化以后，加到当前输出上，作为一个固定的回归目标，梯度只通过回放出来的输出传回生成器。论文推导了，在它用的这条插值路径下，这个速度差和 DMD 用的分数差只差一个系数。

其余部分都沿用 S2，同样的老师，同样的自我回放流程，同样的感知正则项。训练数据也完全复用 S2-Avatar 的，包括直播、口播、影视、独舞和动画。作者明确说，APEX 只改了模型的训练过程。

推理时，用原来的文本条件，每个视频块默认跑 3 次主干网络，再加一个一步完成的 2 倍空间超分 Refiner。老师和 fake 估计器在推理时都不增加调用。

### 实验和效果

实验设置。生成的视频都是 180 秒长。评测有两部分，一是 StreamAV-Bench 的全部 320 个场景，分渐进和交互两个赛道，各 160 个，二是 70 对视频的人工偏好评测，请的是有艺术或美学背景的专家。

先看表 1，12 个模型的比较。APEX 在视觉美学、视频指令完成度和音频指令完成度上排第一，视觉质量排第二。不过要注意，除了 APEX 和 S2-Avatar，其他模型的分数是从 StreamAV-Bench 和 Vidu S2 的论文里引用的，不是作者自己重跑的。

再看表 2，和基线 S2-Avatar 一对一比。渐进赛道的视频指令完成度，从 3.831 提到 3.856，高了 0.025。视觉美学分在两个赛道都略有提高，合起来从 0.633 到 0.642。

但这不是全面提升。音频质量、音画对齐和主体一致性在两个赛道都下降了，音画同步误差从 1.319 秒升到 1.341 秒，交互赛道的视觉质量也掉了 0.070。作者自己的说法是，指令和美学上有收益，同时在视觉和音画方面有取舍。

人工评测看这张图。提升最明显的是指令遵循，70 对里 APEX 赢 22 次，平 47 次，只输 1 次，净偏好是正百分之 30。一致性和视频质量各是正百分之 4.3，也就是净赢 3 次。

运动质量反而是 S2-Avatar 更好，净偏好负百分之 8.6。音画同步 70 对里有 69 对打平。两边的可用率一样，都是 70 个里 62 个可用，百分之 88.6。

再看训练成本。在相同训练设置下单独测量，每张 GPU 的峰值显存从 102.7 GiB 降到 98.2 GiB，少了百分之 4.4，平均每步迭代时间从 35.9 秒降到 32.6 秒，快了百分之 9.2。

还有一个两步生成的实验。APEX 每块只跑 2 次主干网络，去和跑 3 次的 S2-Avatar 比，指令遵循的净偏好还是正百分之 27.1。但其他几项都是负的，一致性和视频质量负百分之 2.9，同步和运动质量负百分之 7.1。

### 和以前方法的对比

把它和 DMD 放在一起看。两者都靠老师和 fake 估计之差来纠正学生，区别只在 fake 估计从哪来，DMD 单独维护一个网络，APEX 让生成器换个条件兼任。省掉的是一个独立网络，在论文的测量里，换来显存少百分之 4.4，迭代快百分之 9.2。

相关工作里也有类似思路。TwinFlow 在一个模型里，用正的时间和负的时间区分真假两条轨迹，FSF-DMD 用生成器推出来的伪速度替代 fake 网络。APEX 的区别是，在取负的文本条件下，用 flow matching 直接拟合生成样本的速度场。

论文还比较了几种训练变体，在表 4。共享和不共享优化器状态，指令遵循的净偏好分别是正百分之 21.4 和正百分之 14.3，不过这两个是在不同步数下测的，不是严格的同条件对比。

我的判断是，这篇的贡献偏工程，思路干净，但收益不大。冻结老师这个大网络仍然保留，显存只省了百分之 4.4，指标上的提升也多在小数点后第二、三位。

我的判断是，还有两点要注意。一是基线 S2-Avatar 来自 Vidu S2，参考文献显示它和这篇有多位共同作者，缺少和其他团队方法在同一设置下的重跑对比。二是人工评测的波动不小，表 4 里标为默认、同样 3 步的那一行，是另一轮 70 对评测，一致性和视频质量都是负百分之 5.7，和主实验的正百分之 4.3 方向相反。每轮只有 70 对，一次净胜大约就是 1.43 个百分点，几个百分点的差别要谨慎看。

### 局限

论文没有单独的局限章节，但结论里自己写了，运动质量和音画同步仍然是取舍，可用率和基线一样，没有提高。

附录里作者也说明，这些理论推导不能证明长时间生成的稳定性，也不是整段视频轨迹上精确的 KL 梯度。

另外，评测只用了 StreamAV-Bench 和自己的人工评测，作者明确说没有做 StreamEval 的测量。训练数据全部沿用 S2-Avatar，以人物、直播、舞蹈和动画为主，换到其他类型的视频效果如何，论文没有测。

### 未来方向

论文没有专门写未来工作。我的判断是，最值得看的是两步生成。现在 2 步下只有指令遵循是正的，其他几项都输给 3 步的基线，如果能补上，主干网络的计算就能再少三分之一。

我的判断是，这篇说明了，在一个完整的流式视频系统里，独立的 fake 网络可以去掉，大部分指标基本持平，指令遵循还更好。对做视频蒸馏的团队来说，这是一个改动小、可以直接试的省钱办法，但别指望它带来画质上的大跃升。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Tsinghua University, Shengshu Technology and Westlake University. It's called Few-Step Streaming Video Generation via Condition Shifting, and the method is named APEX.

The question is narrow but practical. When you distill a video model so it can draw each piece of video in just a few steps, can you get rid of one whole network and make training cheaper?

### Problem

Some background first. Streaming video generation means the video is produced block by block, and everything generated so far becomes the history for what comes next.

Diffusion and flow models look great, but they run the network many times for every block, and a stream pays that cost again and again. So cutting the number of steps really matters. Distillation does that, but the distilled model still has to keep the picture quality, keep characters and scenes stable, and follow instructions that change mid-stream.

### Prior work

A common distillation method is DMD, distribution matching distillation. It relies on two scoring networks. A frozen teacher represents what real data should look like. A second network, the fake-score network, keeps learning what the student is currently producing. The gap between the two tells the student which way to move.

The trouble is that extra fake network. For a large video model, maintaining an auxiliary network costs training memory and compute. And because the student keeps changing, the fake network has to keep tracking it, so the two need coordinated optimizer settings and update schedules.

The baseline here is S2-Avatar, from Vidu S2. It trains with Self-Replay Forcing: the model generates a long rollout on its own, a segment is cut out and re-noised, and the model trains on that replay of its own history. The correction inside it is DMD.

### Key idea

The key idea: drop the separate fake network and let the generator do that job too. One set of weights, with the text condition deciding which role it plays.

Concretely, with the normal text condition, the model generates. Flip the sign of the text condition, and the same model becomes the fake estimator, learning what its own generated videos look like. This trick is called condition shifting. It comes from earlier APEX work, and this paper extends it to streaming video.

The frozen teacher stays, as the reference. The generator and the fake estimator share the model's parameters, and by default they share the AdamW optimizer state as well.

### Method

Here's the picture. On top is DMD: generator, frozen teacher and fake estimator, each with its own weights. Below is APEX: the fake estimator and the generator are the same weights, and only the text input is negated.

Training alternates between two steps. In the first, the generated video blocks are detached, re-noised, and paired with the negated text. A standard flow-matching loss then fits the velocity field of those generated videos. This step only updates the fake-estimation role.

In the second step, on that same noisy video, both the frozen teacher and the fake estimator predict the clean frames. The difference between their predictions is the direction the generator should move.

That difference is normalized, added to the current output, and held fixed as a regression target, so gradients flow back to the generator only through its replay outputs. The paper derives that, on the interpolation path it uses, this velocity difference equals the score difference DMD uses, up to a scaling factor.

Everything else is inherited from S2: the same teacher, the same self-replay procedure, the same perceptual regularizer. Even the training data is S2-Avatar's, covering livestreams, talking heads, film and TV, solo dance and animation. The authors state plainly that APEX changes only the training procedure.

At inference, the model uses the normal text condition and runs the backbone three times per block by default, followed by a one-step, two-times spatial Refiner. The teacher and the fake estimator add no calls at inference.

### Results

The setup. Every video is one hundred eighty seconds long. There are two evaluations. First, all three hundred twenty scenarios of StreamAV-Bench, split into a Progressive track and an Interactive track of one hundred sixty each. Second, a seventy-pair human preference study, rated by experts with backgrounds in art or aesthetics.

Table 1 compares twelve models. APEX ranks first on visual aesthetics, video instruction fulfillment and audio instruction fulfillment, and second on visual quality. One caveat: apart from APEX and S2-Avatar, the other models' scores are quoted from the StreamAV-Bench and Vidu S2 papers, not re-run by the authors.

Table 2 is the head-to-head with S2-Avatar. On the Progressive track, video instruction fulfillment goes from three point eight three one to three point eight five six, up by zero point zero two five. Visual aesthetics edges up on both tracks, from zero point six three three to zero point six four two overall.

But it isn't a clean sweep. Audio quality, audio-visual alignment and subject consistency all drop on both tracks. Audio-visual sync error rises from one point three one nine to one point three four one seconds, and visual quality on the Interactive track falls by zero point zero seven. The authors themselves describe it as gains in instructions and aesthetics, alongside visual and audio-visual trade-offs.

Now the human study, in this figure. The clearest win is instruction following. Out of seventy pairs, APEX wins twenty-two, ties forty-seven and loses one, a net preference of plus thirty percent. Consistency and video quality are each plus four point three percent, which is three net wins.

Motion quality goes the other way, favoring S2-Avatar at minus eight point six percent. Audio-visual sync is tied in sixty-nine of seventy pairs. Both models are usable in sixty-two of seventy sequences, eighty-eight point six percent.

Then training cost. Measured separately under matched training settings, peak memory per GPU falls from one hundred two point seven to ninety-eight point two gibibytes, four point four percent less. Mean iteration time drops from thirty-five point nine to thirty-two point six seconds, nine point two percent faster.

There's also a two-step experiment. APEX runs the backbone only twice per block and is compared against S2-Avatar at three. Instruction following still comes out at plus twenty-seven point one percent. But everything else is negative: minus two point nine for consistency and video quality, and minus seven point one for sync and motion quality.

### Comparison

Put it next to DMD. Both correct the student using the gap between the teacher and a fake estimate. The only difference is where the fake estimate comes from: DMD maintains a separate network, APEX has the generator switch conditions and do it. What you save is one independent network, which in the paper's measurement means four point four percent less memory and nine point two percent faster iterations.

Related work has similar ideas. TwinFlow uses positive and negative time inside one model to separate real and fake trajectories. FSF-DMD replaces the fake network with a pseudo-velocity derived from the generator. APEX differs by directly fitting the velocity field of generated samples with flow matching, under the negated text condition.

Table 4 also compares training variants. With shared versus separate optimizer states, instruction-following preference is plus twenty-one point four and plus fourteen point three percent. But those two were measured at different step counts, so it isn't a like-for-like comparison.

My read is that this is an engineering contribution: a clean idea with modest payoff. The big frozen teacher is still there, the memory saving is only four point four percent, and the metric gains mostly sit in the second or third decimal place.

My read is that two more things deserve caution. First, the baseline S2-Avatar comes from Vidu S2, which the reference list shows shares several authors with this paper, and there's no re-run against other groups' methods in the same setting. Second, the human ratings are noisy. In Table 4, the row marked default at three steps is a separate seventy-pair round, and there consistency and video quality are both minus five point seven percent, the opposite sign from the main study's plus four point three. With seventy pairs, one net win is worth about one point four three points, so differences of a few points should be read carefully.

### Limitations

There's no dedicated limitations section, but the conclusion says it directly: motion quality and audio-visual sync remain trade-offs, and usability is the same as the baseline, not better.

The appendix also notes that the theory does not establish long-horizon stability, and does not give an exact KL gradient over the whole video trajectory.

And the evaluation uses only StreamAV-Bench plus the authors' own human study; they say explicitly that no StreamEval measurements are included. The training data is entirely S2-Avatar's, centered on people, livestreams, dance and animation. How it does on other kinds of video isn't tested.

### Future work

The paper doesn't list future work. My read is that the two-step result is the one to watch. At two steps, only instruction following is positive and everything else trails the three-step baseline; close that gap, and the backbone compute drops by another third.

My read is that the paper shows that in a full streaming video system, the separate fake network can go, with most metrics roughly level and instruction following better. For teams distilling video models, it's a small change that's cheap to try. Just don't expect a big jump in picture quality from it.

That's today's Daily Frontier Papers. The paper is linked in the description.

