# Video-Coder: Reinforcing Agentic Coding for Video Generation

**先写代码搭场景再让AI拍视频**

- 论文 Paper: [arXiv 2611.09894](https://arxiv.org/abs/2611.09894)
- 单位 Affiliation: 中佛罗里达大学 等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-10
- 视频 Videos: —

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇中佛罗里达大学牵头的论文，合作方有维尔茨堡大学、微软研究院、清华大学等，名字叫 Video-Coder。

一句话概括，它训练了一个会写代码的 Agent，先用代码搭出 3D 场景，渲染成草稿视频，再交给视频生成模型，照着草稿画成真实画面。

### 问题

先说问题。现在的视频生成模型画面很好看，但它们是在像素或隐空间里拟合视频分布，并没有建模背后的 3D 世界。

所以提示词里一旦有精确的空间关系、多个物体互动，或者受物理支配的运动，就容易出错，数量不对，位置不对，遮挡矛盾，轨迹不合理，物体还会变形甚至消失。

而且端到端生成的视频很难编辑，也很难复现，用户只能一遍遍改提示词碰运气。

### 以前的方法

以前的一条路，是给视频模型加控制信号，比如深度、轨迹、相机位姿，或者物理仿真。但这些信号通常要用户提供，或者靠人工搭的流水线生成。

另一条路是让大模型写代码。有的先画 2D 布局，但 2D 表达不了 3D 几何、遮挡和相机运动。也有的直接写 Blender 代码搭 3D 场景，Blender 是一款开源的 3D 建模和渲染软件。

最接近的是 GPT4Motion 和 Video-CoCo，一个用 Blender 脚本写物理运动来引导视频模型，一个把代码当中间的思维链。问题是，它们大多在固定流程里提示一个冻结的、往往是闭源的模型，没有针对最终视频的质量去优化。

### 核心想法

这篇的核心想法，是把视频生成变成一个 Agent 编程任务。代码精确、能执行、能组合，天然适合描述几何、运动和相机轨迹。

论文的例子是，三个球从不同高度同时落下，一小段程序就能写清楚，而像素级的模型很难守住这种约束。

看这张总览图。直接生成，数量和运动容易出错。Video-Coder 先写代码，在沙箱里渲染出草稿视频，再让视频模型照着画。

因为场景由一段能执行的程序决定，生成就变得可控、可解释、可验证。而且是多轮的，Agent 能看到报错和渲染结果，再回头改代码。

### 方法

方法分数据、环境和训练三块。先说数据。这种训练数据天然不存在，每条样本要同时有场景描述、一条完整的多轮写代码轨迹，和一段对应的视频。

所以作者搭了一条四步的数据流水线，就是这张图。第一步，把公开视频数据集里的描述，改写成写清物体数量、布局、运动和相机的场景描述，主要靠精细外观、人脸表情的提示词，直接丢掉。

第二步，让 Gemini 3.8 Flash 在环境里写 Blender 的 Python 代码，执行后看报错或画面，反复修改，直到草稿和描述对得上，或者到了轮数上限，整个过程记成一条轨迹。

第三步，把提示词和这段没有纹理的灰色草稿，也叫白模视频，交给 MiniMax-H3 视频模型，生成结构跟着白模、外观跟着提示词的视频，当参考答案。

第四步是过滤，丢掉渲染失败、几乎静止、主体出画的，再用 Seed2.0 判断白模和描述是否一致。最后分成互不重叠的两份，一份做监督微调，一份做强化学习。

再说环境，叫 Video-Coder Harness。每一轮在隔离沙箱里跑 Blender，统一用灰色素模、固定分辨率和帧率渲染，所以视频里只有结构和运动，外观全留给后面的视频模型。

Agent 有两个工具，静态检查和执行渲染。每次调用后，环境返回三类反馈，执行报错，内容检查，比如画面空了、物体出了镜头，还有抽出的几帧画面，让模型看图调整。

还有个细节。Agent 提交后，环境会在干净目录里从头重跑最终脚本，全部检查通过才算数，保证结果只靠代码就能复现。

再说训练。底座是 Qwen3.5-9B，分两阶段。先监督微调，学会规划场景、写代码、调工具、看反馈再修改。再做强化学习，优化多轮写代码的策略。视频模型全程固定，只训练这个 9B 的写代码模型。

强化学习有两个奖励。执行奖励只有零和一，工具格式全对，程序成功执行并通过所有检查，才给一分。视频奖励用 Qwen3.8-omni-flash 当裁判，对照提示词和参考视频，给草稿生成的最终视频打分，归一化到零到一。

两个奖励在不同阶段作用不同。训练早期程序经常报错，拿不到视频奖励，这时执行奖励最关键。代码能跑通以后，视频奖励才能给出更细的指导。

所以作者加了退火，就是这个式子。执行奖励乘上一个随训练步数指数衰减的系数，视频奖励权重 alpha 固定为 0.5。重心就从代码能不能跑，慢慢转到视频好不好。

优化算法是 GRPO，同一个提示词采样一组轨迹，按组内相对好坏来更新。

### 实验和效果

实验设置。训练用 8 张 H100，另外 16 张 H100 专门部署 MiniMax-H3，在强化学习时生成视频。每组 8 条轨迹，最多 100 轮交互，训练大约 3 天。

评测用两个偏物理的基准。PhyGenBench 分力学、光学、热学和材料四类，用 GPT-4o 当裁判，满分是 1。VBench-2.0 只看力学、热学和材料这几个物理维度，按百分比算。

先看表 1。MiniMax-H3 直接生成，平均 0.61。前面接上 Video-Coder，平均到了 0.67，四个类别都是表里最高，力学 0.68，光学 0.79，热学 0.60，材料 0.61。

作为参照，开源的 Wan2.2-TI2V-5B 是 0.54，商用的 Gen-3 是 0.51，Kling 是 0.49，同样走代码路线的 Video-CoCo 是 0.51。

作者还换了视频模型来测。训练时用的是 MiniMax-H3，测试时接到 OmniWeaving 前面，平均分从 0.48 提到 0.51。作者称，这说明它能迁移到别的生成模型。

再看表 2，VBench-2.0。MiniMax-H3 原本平均 54.58%，接上 Video-Coder 后是 78.19%，提高二十多个百分点，也略高于表里最强基线 CogVideoX-1.5 的 77.04%。

作者特别点出力学，从 67.59% 到 88.67%，也超过了 CogVideoX-1.5 的 80.80%。作者解释，下落、碰撞、滚动，正好是 Blender 程序里用关键帧或物理仿真明确写出来的。

这张图是定性例子。三辆车在路上开，三艘船在海上跑，草稿先定好数量、布局、方向和镜头，生成的视频保留这些，再补上真实外观。

### 和以前方法的对比

消融在表 3，都用 MiniMax-H3，看 PhyGenBench 平均分。原始 0.61，只做监督微调 0.63，加上强化学习 0.67，也就是 0.06 的提升里，0.04 来自强化学习。

两个奖励都有用。只用执行奖励是 0.64，只用视频奖励是 0.63，两个都用但不退火是 0.65，都低于完整版的 0.67。

作者推测，执行奖励一直占大头，会盖住视频奖励里的细微差别，模型就倾向写能跑、但保守的程序。

视频奖励权重 alpha 试了 0.4 到 0.7，0.5 和 0.6 相近且更好，0.4 和 0.7 略降，作者认为不太敏感。

我的判断是，和 GPT4Motion、Video-CoCo 这类提示冻结模型的做法比，真正新的地方，是把写 3D 场景代码做成了可训练的 Agent，用最终视频的质量反过来优化代码，让一个 9B 小模型给强视频模型当导演。

我的判断是，也要注意，表 1 的对手大多是直接生成的模型，Video-CoCo 和 LTX-Video-2B 两行分数完全一样，论文没交代这个基线怎么来的。同条件的对比，主要是它自己的消融。

### 局限

论文没有单独的局限章节。作者承认的一处是材料，在 VBench-2.0 上从 48.94% 提到 75.02%，但仍低于 CogVideoX-1.5 的 83.19%。

作者的解释是，材料现象依赖精细外观，而白模故意只编码结构和运动。换句话说，它管的是东西在哪、怎么动，管不到质感。

第二，数据阶段就筛掉了主要靠精细外观和人脸表情的提示词，这类场景从训练开始就没覆盖。

第三，只有两个物理类基准，PhyGenBench 靠 GPT-4o 当裁判，没有人工评测，也没报告代码执行成功率。

第四，论文有几处数字前后不一致。方法部分说数据集是 6k 和 2k，实验和结论写的是 10k 和 4k。摘要说 PhyGenBench 提高约 7 分，表里是 0.61 到 0.67。OmniWeaving 从 0.48 到 0.51，正文写成提高 16.28 分。这些要等后续版本确认。

### 未来方向

未来方向，论文没列具体计划，只希望成为视频生成里代码 Agent 的开放基础，数据、模型和代码完全开源。

我的判断是，这条路的价值，在于把视频生成拆成两步，代码定结构和运动，视频模型负责好看。前一步可检查、可编辑、可复现，正好补上端到端生成最弱的地方。

我的判断是，也要看到一个闭环。参考视频、强化学习出片、主要结果，用的都是 MiniMax-H3，换成 OmniWeaving 只多了 0.03。它学到的是通用的场景编程，还是只会配合这一个生成器，还要更多验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by the University of Central Florida, with collaborators at the University of Würzburg, Microsoft Research, Tsinghua and others. It's called Video-Coder.

The short version: they train an agent that writes code to build a 3D scene, renders it as a rough draft video, and hands that draft to a video generator to paint the realistic version.

### Problem

Today's video generators look great, but they fit the distribution of videos in pixel or latent space. They never explicitly model the 3D world underneath.

So when a prompt needs precise spatial relationships, several interacting objects, or physics-driven motion, things break. Wrong counts, wrong positions, contradictory occlusions, odd trajectories, and objects that deform or vanish.

And an end-to-end generated video is hard to edit or reproduce, so users rewrite the prompt by trial and error.

### Prior work

One fix is to give the video model explicit control signals, like depth, trajectories, camera poses, or physics simulation. The catch is that those signals usually come from the user or a hand-built pipeline.

Another is to have a language model write code. Some work draws a 2D layout first, but 2D can't capture 3D geometry, occlusion or camera motion. Other work writes code for Blender, the open-source 3D modeling and rendering tool, to build the scene directly.

The closest relatives are GPT4Motion, which scripts physical motion in Blender to guide a video model, and Video-CoCo, which uses code as an intermediate chain of thought. But most of these prompt a frozen, often proprietary model in a fixed pipeline, with nothing optimized for the final video.

### Key idea

The key idea is to treat video generation as an agentic programming task. Code is precise, executable and composable, a natural fit for geometry, motion and camera paths.

The paper's example: three spheres falling at once from different heights. A short program states that exactly, and it's the kind of constraint pixel-level models struggle to keep.

Here's the overview. Direct generation gets counts and motion wrong. Video-Coder writes code, renders a draft in a sandbox, and the video generator follows that draft.

Because an executable program pins down the scene, generation becomes controllable, interpretable and verifiable. And it's multi-turn: the agent sees errors and renders, and goes back to fix its code.

### Method

Three parts: data, environment, training. Data first. This kind of data doesn't exist. Each sample needs a scene prompt, a full multi-turn coding trajectory, and a matching video.

So the authors built a four-stage pipeline, shown here. Stage one rewrites captions from public video datasets into scene-centric prompts that spell out object counts, layout, motion and camera, and drops prompts that depend on fine appearance or facial expressions.

Stage two has Gemini 3.8 Flash write Blender Python code in the environment, run it, read the errors or frames, and revise until the draft matches the prompt or it hits the turn limit. Each episode becomes a trajectory.

Stage three feeds the prompt and that untextured gray draft, which the authors call the white-model video, into the video generator MiniMax-H3. The output follows the draft for structure and motion and the prompt for appearance, and serves as the reference answer.

Stage four filters out failed renders, near-static videos and lost subjects, and uses Seed2.0 to check that each draft matches its prompt. The rest is split into two non-overlapping sets, one for supervised fine-tuning and one for reinforcement learning.

Next, the environment, the Video-Coder Harness. Each episode runs Blender in an isolated sandbox with one shared look: gray clay shading, fixed resolution and frame rate. So the video carries only structure and motion, and appearance is left to the generator.

The agent has two tools: a fast static check, and a command that runs and renders the script. After each call it gets three kinds of feedback: execution errors; content checks, like empty frames, a static video, or objects leaving the shot; and a few evenly spaced frames, so it can look at its own output and adjust.

One more detail. When the agent submits, the harness re-runs the final script from scratch in a clean directory, and accepts it only if every check passes again. Every accepted result is reproducible from the code alone.

Then training. The base model is Qwen3.5-9B, in two stages. Supervised fine-tuning teaches it to plan, write code, call tools and revise. Reinforcement learning then sharpens its multi-turn strategy. The video generator stays frozen; only the nine-billion-parameter coder is trained.

There are two rewards. The execution reward is one only if every tool call is well-formed and the final program runs and passes all checks, otherwise zero. The video reward uses Qwen3.8-omni-flash as a judge on the final video made from the draft, comparing it with the prompt and reference on objects, layout, motion and camera, normalized to between zero and one.

They matter at different times. Early on, programs often crash, so there's no video to score, and the execution reward does the work. Once code reliably runs, the video reward gives finer guidance.

So the authors anneal, with this formula. The execution reward is multiplied by a factor that decays exponentially with the training step, while the video weight, alpha, is fixed at zero point five. The focus shifts from "does it run" to "is the video good".

The optimizer is GRPO: sample a group of trajectories per prompt, and update on how each compares with its group.

### Results

Setup. Training used eight H100s, plus sixteen more serving MiniMax-H3 to generate videos during reinforcement learning. Eight trajectories per group, up to one hundred turns, about three days in total.

Evaluation uses two physics-focused benchmarks. PhyGenBench covers mechanics, optics, thermal and material, judged by GPT-4o, scored up to one. For VBench-2.0 they use only the physics dimensions, mechanics, thermotics and material, as percentages.

Table one. MiniMax-H3 alone averages zero point six one. With Video-Coder in front, it reaches zero point six seven, the top score in the table in all four categories: mechanics zero point six eight, optics zero point seven nine, thermal zero point six zero, material zero point six one.

For reference, the open model Wan2.2-TI2V-5B scores zero point five four, commercial Gen-3 zero point five one, Kling zero point four nine, and Video-CoCo, the other code-based approach, zero point five one.

They also swapped the generator. Training always used MiniMax-H3, but in front of OmniWeaving at test time, the average went from zero point four eight to zero point five one. The authors read this as transfer to other generators.

Table two, VBench-2.0. MiniMax-H3 averages fifty-four point five eight percent alone, and seventy-eight point one nine with Video-Coder, over twenty points higher, and just above the strongest baseline, CogVideoX-1.5, at seventy-seven point zero four.

The authors single out mechanics: from sixty-seven point five nine to eighty-eight point six seven percent, also above CogVideoX-1.5 at eighty point eight. The authors' explanation: falling, collisions and rolling are exactly what the Blender program spells out with keyframes or physics simulation.

This figure shows examples. Three cars on a road, three boats on the ocean. The draft fixes counts, layout, direction and framing, and the final video keeps them while adding realistic appearance.

### Comparison

The ablations are in Table three, all with MiniMax-H3 on PhyGenBench. The plain generator: zero point six one. Fine-tuning only: zero point six three. Adding reinforcement learning: zero point six seven. So four of the six hundredths come from reinforcement learning.

Both rewards earn their place. Execution reward only scores zero point six four; video reward only, zero point six three; both without annealing, zero point six five. All below the full model.

The authors' hypothesis: if the execution reward keeps a big share, it drowns out fine differences in the video reward, and the agent learns to write code that runs but plays it safe.

For alpha they tried zero point four to zero point seven. Zero point five and zero point six are similar and better, the ends dip slightly, and they conclude it isn't very sensitive.

My read is that, compared with GPT4Motion, Video-CoCo and others that prompt a frozen model, what's new is making 3D scene coding a trainable agent, optimized by the quality of the final video. A nine-billion-parameter model ends up directing a strong video generator.

My read is also that Table one mostly compares against direct generators, and the Video-CoCo row is identical to the LTX-Video-2B row, number for number, with no explanation of how that baseline was run. The truly controlled comparisons are the paper's own ablations.

### Limitations

There's no dedicated limitations section. The one the authors acknowledge is material. On VBench-2.0 it goes from forty-eight point nine four to seventy-five point zero two percent, but still trails CogVideoX-1.5 at eighty-three point one nine.

Their explanation is that material depends on fine appearance, while the white-model video deliberately encodes only structure and motion. Put simply, it controls where things are and how they move, not what they look like.

Second, the data pipeline dropped prompts relying mainly on fine appearance or facial expressions, so those scenes were never in training.

Third, there are only two physics benchmarks, PhyGenBench relies on GPT-4o as judge, and there's no human evaluation and no reported code execution success rate.

Fourth, several numbers disagree. The method section describes datasets of six thousand and two thousand samples, while the experiments and conclusion say ten thousand and four thousand. The abstract claims about seven points on PhyGenBench, but the tables go from zero point six one to zero point six seven. And OmniWeaving's move from zero point four eight to zero point five one is described in the text as sixteen point two eight points. A later version will need to settle these.

### Future work

The paper doesn't lay out specific plans. It hopes to be an open foundation for coding agents in video generation, and says data, models and code are fully open-sourced.

My read is that the value here is splitting video generation in two: code decides structure and motion, the video model makes it look good. The first half can be checked, edited and reproduced, exactly where end-to-end generation is weakest.

My read is also that there's a loop to watch. The reference videos come from MiniMax-H3, reinforcement learning renders with it, and the headline results use it. On OmniWeaving the gain is only three hundredths. Whether it learned general scene programming or how to work with this one generator needs testing on more generators.

That's today's Daily Frontier Papers. The paper is linked in the description.

