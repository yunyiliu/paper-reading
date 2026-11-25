# Embo-Harness: Towards Understanding Frontier-Model-as-Policy Harness

**通用大模型控机器人赢在看得准**

- 论文 Paper: [arXiv 2611.31201](https://arxiv.org/abs/2611.31201)
- 单位 Affiliation: 南洋理工大学 S-Lab · 商汤研究院 · 厦门大学 等
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-25
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1ZNy6zTEgm)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇南洋理工大学 S-Lab 牵头的论文，合作方包括商汤研究院和厦门大学等，题目叫 Embo-Harness。

它研究的是一种新做法，直接让最强的通用多模态大模型当机器人的控制策略。论文问了两个问题，围绕模型的那套软件该怎么搭，以及它为什么能行。

### 问题

先解释一个词，harness。论文的定义是，把观察和指令，连接到规划、工具调用、动作生成、执行和反馈的那一整套软件循环，也就是包在模型外面、让它真正能干活的那一层。

最近已经有工作显示，像 GPT-6-Astra 这样的前沿模型，在机器人操作基准上能超过好几个专门训练的机器人模型。但作者指出，这些工作大多只是展示或改进某一个系统，没有系统地研究，到底是 harness 里哪个部分带来了提升，也没有回答，通用模型凭什么能赢过专门做控制的策略。

### 以前的方法

论文把机器人 harness 分成三类。第一类叫工具或代码即策略，大模型只做高层规划，真正的动作交给预先写好的工具，比如物体检测、抓取检测、运动规划，或者让模型写代码去调这些工具。

第二类叫模型即策略，大模型负责规划和反馈，底层动作交给专门训练的策略模型，比如 VLA，也就是把视觉、语言和动作放在一起训练的机器人模型。

第三类就是这篇研究的，前沿模型即策略，前沿模型自己看图、自己规划、自己直接输出机器人动作，全部在一个闭环里。这张图把三类放在一起，区别主要就在于，动作是谁生成的。

### 核心想法

作者没有一上来就设计一个完整系统，而是做对照实验。从一个共同的基线出发，每次只改一个设计因素，把有效的选择保留下来，一步一步往上叠。

研究覆盖 Agent 循环的四个维度，提示、规划、动作执行和工具使用，最后总结出六条发现。核心原则可以用一句话说，顺着前沿模型本来就擅长的能力去设计接口，而不是硬塞人工写好的计划、技能或者动作先验。

### 方法

基线照搬 π0.5 这个 VLA 的实验设置。动作用关节的绝对角度，一次输出 15 步，输入只有任务指令和两个视角的图像，分辨率 224 乘 126。模型是 GPT-6-Astra，中等推理强度。所有对照实验都在 RoboLab 基准里挑出的 20 个任务上做，每个任务跑 3 次。

先看动作执行，结果都在表 1。动作可以用关节角度表示，也可以用末端执行器，也就是夹爪在空间里的位置表示，后者能通过逆运动学，简称 IK，换算成关节角度。四种格式的成功率差不多。区别在于，让模型输出关节角度时，它经常自己写代码调 IK 去算，直接生成的只有大约三成，执行时间也更长。

再看动作块，也就是一次输出几步动作。作者发现，不管要求输出几步，模型预测的目标位置都差不多。块太大，动作变慢，可能把步数预算用完，块太小，又走不到目标。结果 8 步最好，成功率 58.3%，15 步是 43.3%，30 步只有 21.7%。

异步执行，就是当前这块动作还没做完，就开始算下一块，这在机器人策略里很常用。但前沿模型很难预判这块动作做完以后机器人在哪，成功率从 58.3% 掉到 51.7%，所以没有采用。

接下来是规划。论文称，更强的模型和更高的推理强度总体更好，GPT-6-Astra 明显好过 GPT-6.1 的另外两个版本，最高推理强度 xhigh 达到 65%。有个反常的地方，中等推理的 58.3%，反而比高推理的 51.7% 好。另外，额外加一步显式规划，先拆成子任务再执行，成功率是 61.7%，没有超过 65%，还更慢。

上下文方面，去掉历史画面，成功率差不多，但 KV cache 命中率从 95% 以上掉到大约 25%，没命中缓存的 token 消耗涨了 7 到 8 倍，所以作者保留了完整的上下文。

提示方面，作者从失败案例里总结出 12 条和具体任务无关的操作提醒，加进提示词，结果成功率从 65% 降到 60%，执行时间也变长了。

视觉输入的影响很大。去掉手腕相机，只留第三人称视角，成功率从 65% 掉到 30%。分辨率从 224 乘 126 提到 640 乘 360，成功率从 65% 提到 83.3%。

然后是工具。作者加了一个工具，让模型一次连着输出几个动作，中间不看画面，常见组合之一是接近、抓取、撤回。成功率从 83.3% 到 90%，时间从 14.6 分钟降到 10.5 分钟。这种多动作序列大约三成情况会用到，平均每次三个动作。

最后是怎么和 VLA 配合。如果让前沿模型每一步自己决定，是自己出动作还是交给 VLA，它判断不准 VLA 的能力，大多都交了出去，成功率掉到 58.3%。反过来，让 VLA 先提一个动作，前沿模型修改以后再执行，成功率达到 96.7%，是表 1 里最高的。

最终的配方就是这张图。高分辨率的多视角图像输入，GPT-6-Astra 的 xhigh 推理，输出末端执行器的绝对位置，每块 8 步，再加上多动作序列和 VLA 修正这两个工具。

### 实验和效果

完整评测在 RoboLab-120 上做，这是一个基于 NVIDIA Isaac Lab 的高保真仿真基准，有 120 个桌面操作任务，用的是 7 自由度的 Franka Panda 机械臂。所有方法都不做针对任务的微调。

结果在表 2。Embo-Harness 成功率 83.3%，任务完成分 89.2。最强的对比方法 SenseNova-RoboRSI 同样用 GPT-6-Astra，成功率 57.1%，Embo-Harness 高出 26.2 个百分点。专门的机器人策略模型里，最好的 FLUX 3 Action 是 42.9%，π0.5 是 28%。

真实机器人上，四个任务各试 5 次，全部零样本，没有任务相关的记忆或提示。叠积木和装水果都是 5 次全成，洗碗要打开一个玩具水槽的水龙头，成功 2 次，插花成功 3 次，一共 20 次成功 15 次。作为对比，基线配置在后两个任务上都是 0 次。

作者说，失败主要来自精细的空间定位，比如找准一个很小的水龙头开关，或者插花时要对准花瓶口，而手腕相机一倾斜，就很难看清瓶口。

### 和以前方法的对比

论文后半部分问了一个更有意思的问题，通用模型凭什么能和专门的 VLA 比，甚至超过它们。VLA 一般靠模仿学习，学的是逐帧的轨迹，有的还学关节层面的动作模式。前沿模型是不是也靠这个。

作者的实验说，主要不是。看表 4，禁用 IK、让模型直接输出关节角度，成功率从 43.3% 掉到 20%。让模型逐帧生成轨迹，而不是给一个目标位置再插值，8 步块从 58.3% 掉到 40%。也就是说，这些底层动作它会做，但让它做反而更差。

反过来，看表 5，如果只让模型说出最终要到的位置，成功率是 50%，比分段输出末端位置的 43.3% 和 45% 还高一点。

作者又直接测了模型估计目标物体三维位置的误差。推理强度从中等提到 xhigh，平均误差从 14.1 厘米降到 9.07 厘米，提高分辨率也是类似的趋势，和操作成功率的变化方向一致。而在失败的抓取里，夹爪离目标的平均误差是 11.04 厘米，和位置估计的误差差不多大。

所以作者的结论是，前沿模型能当机器人策略，主要靠的是准确的空间感知，而不是底层动作生成。不过作者也说明，这并不代表模型没有隐含的动作知识，也不代表底层动作数据没用。

我的判断是，这对 VLA 路线是个提醒。与其让模型模仿更多的关节和轨迹，不如先把空间定位做准，这和作者呼吁更重视空间落地的模型、数据和基准，是同一个方向。

我的判断是，也要看清几个弱点。配方是在 20 个任务、每个 3 次，也就是 60 次试验上一步步选出来的，差一次成功就是 1.7 个百分点左右，像中等推理赢过高推理这种差别，可能有不少是噪声。这 20 个任务也取自同一个 RoboLab 基准。而且全程只测了 GPT 一个模型家族，真实机器人也只有 20 次试验。

### 局限

作者写的局限比较短。第一，这些结论依赖当前前沿模型的能力，模型进步以后可能会变。

第二，空间感知也许足够做通用的夹爪操作，但灵巧手操作、移动和触觉感知，还需要额外的底层和动态能力。

另外正文提到，同期工作 AMP1 也报告了很强的结果，但它用了外部的高精度深度估计器，设置不同，所以没有直接对比。

### 未来方向

VLA 选择这条路也没有被否定。作者认为，如果让模型看到 VLA 过去的成功和失败，或者给出 VLA 的置信度，委派可能会更可靠，这留到以后做。

更大的方向，作者认为这些发现指向一种超越传统 VLA 和世界动作模型的机器人策略，不太依赖底层动作模仿，也可能做到效果好、执行快。

我的判断是，这篇对做 Agent 的人也有参考价值。harness 应该把模型擅长的事交给它，比如看图、找目标、纠错，把它不擅长的，比如逆运动学和插值，交给确定的程序。人工写的提醒和计划，在强模型面前帮助有限。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by S-Lab at Nanyang Technological University, with partners including SenseTime Research and Xiamen University. It's called Embo-Harness.

It looks at a new way to control robots: take a general-purpose frontier multimodal model and use it directly as the robot's policy. The paper asks two things. How should you build the software around that model, and why does it work at all?

### Problem

First, a word: harness. The paper defines it as the software loop that connects observations and instructions to planning, tool use, action generation, execution and feedback. It's the layer around the model that lets it actually do work.

Recent work has shown that frontier models like GPT-6-Astra can beat several specialized robot models on manipulation benchmarks. But the authors point out that those studies mostly demonstrate or improve one particular system. They haven't systematically tested which parts of the harness drive the gains, or why a general-purpose model can beat policies built specifically for robot control.

### Prior work

The paper sorts robot harnesses into three families. The first is tool-as-policy or code-as-policy. The model only plans at a high level, and the actual motion comes from predefined tools, like object detection, grasp detection and motion planning, or from code the model writes to call them.

The second is model-as-policy. The language model plans and handles feedback, while low-level actions come from a specialized learned policy, such as a VLA, a vision-language-action model trained on robot data.

The third, and the subject of this paper, is frontier-model-as-policy. The frontier model looks, plans, and outputs robot actions itself, all in one closed loop. This figure puts the three side by side. The main difference is who generates the action.

### Key idea

Rather than designing a full system up front, the authors run a controlled study. They start from one common baseline, change a single design factor at a time, keep whatever works, and stack those choices step by step.

The study covers four parts of the agent loop: prompting, planning, action execution and tool use, and it ends with six findings. The principle behind them fits in one line. Design the interface around what the frontier model is already good at, instead of forcing in handcrafted plans, skills or action priors.

### Method

The baseline copies the setup used for pi zero point five, a VLA. Actions are absolute joint angles, fifteen steps per chunk, and the only inputs are the instruction and images from two cameras at two hundred twenty-four by one hundred twenty-six pixels. The model is GPT-6-Astra at medium reasoning effort. Every ablation runs on twenty tasks picked from the RoboLab benchmark, three trials each.

Start with action execution; it's all in Table 1. Actions can be joint angles, or the position of the end effector, meaning the gripper's position in space, which inverse kinematics, or IK, converts into joint angles. All four formats score about the same. The difference is that when asked for joint angles, the model often writes code to call IK, generating them directly only about thirty percent of the time, and execution takes longer.

Next, chunk size: how many steps the model outputs at once. The authors found the model predicts roughly the same target no matter how many steps it's told to cover. Big chunks move slowly and can burn through the step budget; small chunks don't get there. Eight steps wins at fifty-eight point three percent, fifteen steps gets forty-three point three, and thirty steps just twenty-one point seven.

Asynchronous execution means computing the next chunk while the current one is still running, a common trick in robot policies. But the frontier model struggles to predict where the robot will be once the chunk finishes, and success drops from fifty-eight point three to fifty-one point seven. So it's left out.

Then planning. The paper says stronger models and more reasoning generally help. GPT-6-Astra clearly beats two GPT-6.1 variants, and the top effort level, xhigh, reaches sixty-five percent. One oddity: medium effort, at fifty-eight point three, beats high effort, at fifty-one point seven. Adding an explicit planning step, splitting the task into subtasks first, scores sixty-one point seven, no better than sixty-five, and it's slower.

On context, dropping the image history keeps success about the same, but the KV cache hit rate falls from over ninety-five percent to about twenty-five, and uncached token use goes up seven to eight times. So the authors keep the full context.

On prompting, the authors distilled twelve task-agnostic reminders from failure cases and added them to the prompt. Success fell from sixty-five to sixty percent, and runs got slower.

Visual input matters a lot. Remove the wrist camera, keeping only the third-person view, and success falls from sixty-five to thirty percent. Raise resolution from two twenty-four by one twenty-six to six forty by three sixty, and success climbs from sixty-five to eighty-three point three.

Next, tools. The authors add one that lets the model output several actions in a row without looking in between, one common pattern being approach, grasp, retreat. Success goes from eighty-three point three to ninety percent, and time drops from fourteen point six to ten point five minutes. The model uses these sequences in about thirty percent of cases, three actions each on average.

Last, working with a VLA. If the frontier model decides at each step whether to act itself or hand off to the VLA, it misjudges what the VLA can do, hands off most of the time, and success falls to fifty-eight point three. Flip it around: the VLA proposes an action, and the frontier model refines it before executing. That reaches ninety-six point seven percent, the best result in Table 1.

The final recipe is this figure. High-resolution, multi-view images in, GPT-6-Astra at xhigh effort, absolute end-effector positions out, eight-step chunks, plus two tools: multi-action sequences and VLA refinement.

### Results

The full evaluation is RoboLab-120, a high-fidelity simulation benchmark built on NVIDIA Isaac Lab, with one hundred twenty tabletop manipulation tasks and a seven-degree-of-freedom Franka Panda arm. No method gets task-specific fine-tuning.

Table 2 has the numbers. Embo-Harness reaches eighty-three point three percent success and a task completion score of eighty-nine point two. The strongest baseline, SenseNova-RoboRSI, also uses GPT-6-Astra and scores fifty-seven point one, so Embo-Harness is twenty-six point two points ahead. Among dedicated robot policies, the best, FLUX 3 Action, gets forty-two point nine, and pi zero point five gets twenty-eight.

On a real robot, four tasks, five tries each, all zero-shot with no task-specific memory or hints. Block stacking and fruit packing go five for five. Dish washing, which means turning on the faucet of a toy sink, succeeds twice, and flower arranging three times. That's fifteen out of twenty. The baseline setup scored zero on those last two tasks.

The authors say the failures come mostly from fine spatial localization: finding a small faucet switch, or lining a flower up with the vase opening, which is hard to see from the wrist camera once it tilts.

### Comparison

The second half of the paper asks the more interesting question. How does a general model keep up with, and even beat, specialized VLAs? VLAs mostly learn by imitation, picking up frame-by-frame trajectories, and some also joint-level motion patterns. Is the frontier model doing the same thing?

The experiments suggest mostly not. In Table 4, disabling IK and asking for joint angles directly drops success from forty-three point three to twenty percent. Asking for a frame-by-frame trajectory, instead of one target pose plus interpolation, drops the eight-step setup from fifty-eight point three to forty. It can produce those low-level actions, but making it do so hurts.

The flip side is Table 5. If the model just names the final destination, success is fifty percent, a little above the forty-three point three and forty-five from step-by-step end-effector control.

The authors then measured how well the model estimates a target object's 3D position. Going from medium to xhigh effort cuts the average error from fourteen point one to nine point oh seven centimeters, higher resolution shows a similar trend, and both line up with the success rates. And in failed grasps, the gripper ends up eleven point oh four centimeters from the target on average, about the same size as the estimation error.

So the authors conclude that the model's robot skill comes mainly from accurate spatial perception, not low-level action generation. They're careful to add that this doesn't mean the model lacks implicit action knowledge, or that low-level action data is useless.

My read is that this is a nudge for the VLA camp. Before imitating more joint and trajectory data, get spatial grounding right. That matches the authors' own call for more spatially grounded models, datasets and benchmarks.

My read is also that the evidence has soft spots. The recipe was chosen step by step on twenty tasks times three runs, sixty trials, so one success is worth about one point seven points, and gaps like medium beating high effort may be largely noise. Those twenty tasks also come from the same RoboLab benchmark. Only one model family, GPT, is tested, and the real robot gets just twenty trials.

### Limitations

The authors' limitations are brief. First, the findings depend on today's frontier models and may change as models improve.

Second, spatial perception may be enough for general gripper manipulation, but dexterous hands, locomotion and touch need extra low-level and dynamic capabilities.

The paper also notes that the concurrent work AMP1 reports strong results too, but relies on an external high-accuracy depth estimator, so the settings differ and there's no direct comparison.

### Future work

Letting the model choose when to delegate to a VLA isn't ruled out. The authors suggest that showing the model the VLA's past successes and failures, or giving it a confidence estimate, might make delegation more reliable. That's left for later.

More broadly, the authors see these results pointing toward robot policies beyond conventional VLAs and world action models, ones that lean less on low-level imitation and could still be strong and fast.

My read is that agent builders outside robotics can use this too. A good harness hands the model what it's good at, like seeing, locating and correcting, and hands deterministic programs what it isn't, like inverse kinematics and interpolation. Handwritten reminders and plans add little on top of a strong model.

That's today's Daily Frontier Papers. The paper is linked in the description.

