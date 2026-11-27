# Improving and Expanding Video Editing Capabilities Without Videos

**只用图片教会视频改动作**

- 论文 Paper: [arXiv 2611.34941](https://arxiv.org/abs/2611.34941)
- 单位 Affiliation: 卡内基梅隆大学 · 南洋理工大学 · 悉尼科技大学 · 密歇根大学
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-27
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1Em1xzKEg6)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇卡内基梅隆大学、南洋理工大学、悉尼科技大学和密歇根大学的论文，题目是，不用视频，也能提升和扩展视频编辑能力。

方法叫 I2VT，意思是从图像迁移到视频。它想回答一个问题，没有新的成对视频数据，视频编辑模型能不能变强，甚至学会新本事。

### 问题

先说背景。现在的图像编辑模型进步很快，能按很细的指令改图。

视频编辑模型就差一截。论文说，它们连删掉一个物体这种基础编辑，也只做到中等水平，要改人物的动作、表情、身体姿态，或者改场景的光照，就更吃力。

论文这张图讲的就是这个动机。左边，图像编辑能在单帧上做动作、打光这类复杂改动，作者想要的是，视频编辑也能做到，同时整段视频前后连贯。

有意思的是，图像和视频编辑模型的结构其实很像，用同样的 flow matching 训练目标，类似的扩散 Transformer，很多还共用同一个 VAE 隐空间。VAE 就是把图像压缩成模型内部表示的编码器。

作者认为，差距的一个原因在数据。训练视频编辑需要成对的视频，一段原视频，一段按指令改好的视频，整段都要改对，无关的地方又不能动。这种数据很难造，编辑越复杂越难。

### 以前的方法

以前用图像编辑帮视频编辑，大概有几类做法。一类是先用图像编辑模型改第一帧，再让视频模型把这个改动传播到后面。

论文附录指出，引导不够强的时候，模型会重新生成动作，而不是保留原视频的动作，第一帧看不到的内容要靠猜，数据质量也被这个生成模型卡住。

另一类，直接拿图像编辑的成对数据训练，比如把一帧重复成视频，或者人为做出运动。还有一类，改模型结构，或者加适配器。

最直接的一种叫离线蒸馏，先让图像编辑模型把每张图改好，再让视频模型照着这些改好的图去学。这篇论文把它当作主要的对照组。

### 核心想法

作者的出发点很简单，一张图片，就是只有一帧的视频。所以把图片编辑好，可以是把视频编辑好的基础。

于是他们让视频编辑模型去扮演图像编辑模型，把一张图当成一帧的视频来改，旁边有一个冻结的图像编辑模型当老师。

关键在于怎么教。离线蒸馏是先准备好标准答案，I2VT 是在学生自己一步步去噪的过程中，每一步都由老师告诉它，当前这个中间结果该怎么更新。

### 方法

先补一点背景。扩散模型生成图像，是从纯噪声出发一步步去噪。flow matching 的做法是，模型在每一步预测一个速度，也就是当前状态该往哪个方向走，才能到达干净的结果。

这张图左边是离线蒸馏，中间是 I2VT，右边是推理。训练时，学生从纯噪声出发，按指令编辑一张图，每张图走 10 步去噪。

每走一步，把学生当前的中间状态交给老师，老师给出它认为该走的速度，学生调整自己，让自己预测的速度尽量接近老师的。

能这么做，前提是老师和学生共用一个 VAE 隐空间，老师可以直接根据学生的中间状态给出目标。论文用的老师是 JoyAI-Image-Edit，老师和三个学生都用 Wan 2.1 的 VAE。

指令跟随的力度，靠 classifier-free guidance 控制，简称 CFG，可以理解成让模型更用力地按指令改。默认强度都是 1，在学生较弱的动作和身体动作编辑上，以及 VideoCoF 上，把老师的强度提到 2。

推理时就不要老师了，学生直接编辑整段视频，采样设置和原来的模型一样。虽然训练只见过单帧，它能把学到的能力用到整段视频上。

训练数据叫 I2VT-17K，是从 OpenVE-3M 这个视频数据集里抽出来的单帧。一共 17227 张图，每张图只配两条指令，一条给老师，一条给学生，没有任何改好的目标图或视频。

新任务的数据，先用一个视觉语言模型描述每一帧，再挑合适的编辑。比如要抬手，这只手得在画面里，而且还没抬起来。

训练成本很低。每个模型用 LoRA 微调 200 步，LoRA 是一种只训练少量附加参数的方法。因为只处理单帧，视觉 token 少了很多，1.3B 的模型只要 20 个 A100 GPU 小时，14B 的模型是 20 到 25 个 B300 GPU 小时。

### 实验和效果

实验用了三个现成的视频编辑模型当学生，Bernini-R 的 1.3B 和 14B 两个版本，还有 VideoCoF。

评测有两套。基础编辑用 OpenVE-Bench，新能力用作者自己做的 OpenVE++，加了四类任务，改动作、改表情、身体动作、场景重新打光，一共 207 段视频，训练和测试的源视频完全不重叠。

打分主要靠 Gemini 2.5 Pro 当裁判，看指令遵循、和原视频的一致性、画面质量。每项 1 到 5 分，下面这两个基准的分数都是这个尺度。

先看基础编辑，表 1。三个模型全部提高，Bernini-R-1.3B 从 3.10 到 3.48，VideoCoF 从 2.29 到 3.19，Bernini-R-14B 从 3.33 到 3.68。

也不是每一项都涨得明显。VideoCoF 添加物体这一项几乎没变，从 2.50 到 2.51。

再看新能力，表 2，这是重点。原来的模型在这些任务上分数很低，Bernini-R-1.3B 新任务总分只有 1.66，训练后到 2.99，14B 版本从 2.25 到 3.27。

VideoCoF 涨得最多，从 1.17 到 3.33，起点最低，最后和训练后的 Bernini-R-14B 差不多。

这三类新任务，训练时用的全是单张图片。论文用改动作和改表情的例子说明，从图片学到的能力，确实用到了整段视频上。

还有一个担心，只用单帧训练，会不会把视频的连贯性弄坏。作者用 VBench 测了主体和背景一致性、闪烁、运动平滑度等指标，结论是基本持平。

再放到同行里比，表 3。在作者比较的开源视频编辑模型里，I2VT-14B 在 OpenVE-Bench 上 3.58 分，在 OpenVE++ 上 3.34 分，都是最高。

换成 Gemini 3.8 Flash 当裁判，I2VT-CoF 在 OpenVE++ 上 4.01 分，排第一。论文报告，两个裁判给出的排名很接近，排名相关系数是 0.91。

还有一个小规模的人工评测，20 个人，每人比较 16 对视频。I2VT-1.3B 对原模型，赢 68.1%，输 6.9%。对更强的 Bernini-R-14B，I2VT-14B 赢 50.6%，输 15%。

### 和以前方法的对比

最关键的对比，是和离线蒸馏比，表 4。同一个 1.3B 学生，同样的源图片，同样的老师。默认的离线设置和 I2VT 一样，每次更新用 24 张图，监督 240 个状态。

离线蒸馏也能学会新任务，但基础编辑有得有失，三种离线设置都让风格编辑变好，却让背景替换变差。I2VT 在两套任务上总分都是最高，基础编辑 3.48，新任务 2.99。

也要说清楚，在改动作和身体动作这两项上，有的离线设置分数比 I2VT 略高。

这张图是一个例子，让坐着的女人站起来，离线蒸馏的结果人还坐着，I2VT 让她站了起来。

消融实验里，学生这边的 CFG 强度设成 1 最好，调到 5，分数掉到 1.88，比原模型还低。老师的强度调高，对基础编辑没帮助，但在难的新任务上有提升。

论文附录对比了两篇同样不用视频数据训练的工作，它们是在文生视频模型上加模块，重新搭一个编辑器，而且只评测了外观类编辑，比如换风格、换颜色、加减物体。

需要注意，作者没有和这两篇直接对比，原因是一篇没公开代码，另一篇公开了代码但没有训练好的权重。

我的判断是，I2VT 的价值在于它是给现成的编辑模型做后训练，不改结构，成本低，还能教会模型改动作、改表情这类原来几乎做不了的事。

### 局限

论文没有单独的局限章节，下面几点来自它的实验和附录。

第一，打分主要靠 Gemini 当裁判。人工评测只有 20 个人，而且是作者认识的志愿者。

第二，在这三个模型上，原模型越强，提升越小。I2VT-14B 对原模型的人工评测里，身体动作这一类有一半是平局。

第三，换到外部基准 IVEBench 上，三个模型的总分都提高了，但 Bernini-R-1.3B 和原视频的保真度从 80.2 降到 77.1。

第四，方法有前提，老师和学生要共用同一个 VAE 隐空间。论文提到，另一篇类似工作的图像和视频模型 VAE 不同，就得额外学连接器来转换。

### 未来方向

作者在结论里说，这说明图像编辑的监督，不需要新的成对视频，也能提升视频编辑。OpenVE++ 会随论文一起公开。

作者还提到，Gemini 2.5 Pro 会在 2026 年 12 月 1 日从 API 下线，所以也报告了 Gemini 3.8 Flash 的分数，方便以后对比。

我的判断是，这篇最值得关注的是数据思路。图像编辑的数据和模型都比视频多，如果单帧就能把能力迁移过去，视频编辑就能跟着图像编辑一起进步，不必每次都去造昂贵的视频对。

我的判断是，也要看到，目前的效果主要是在作者自己的基准和大模型裁判上测出来的，改动作这类编辑在更长、更复杂的视频里表现如何，还需要更多验证。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Carnegie Mellon, Nanyang Technological University, the University of Technology Sydney and the University of Michigan, and its title is a small provocation: improving and expanding video editing capabilities, without videos.

The method is called I2VT, short for image-to-video transfer. The question it asks: can a video editing model get better, and even learn new kinds of edits, without any new paired video data?

### Problem

Start with the gap. Image editing models have moved fast. They follow detailed instructions and give you fine-grained control.

Video editors lag behind. The authors say they are only moderately good even at basic edits like removing an object, and they struggle more with changing a person's actions, facial expression or body movement, or a scene's lighting. Those are exactly the edits video production keeps asking for.

This figure is the motivation. On the left, image editors handle actions and relighting on a single frame. The goal is a video editor that can make the same edits and keep the whole clip temporally consistent.

The odd part is that the two kinds of model are built almost the same way. Same flow-matching objective, similar diffusion transformers, and many share one VAE latent space, the compressed representation the model actually works in.

So the authors point at data. A video editing pair needs a source clip and an edited clip that follows the instruction across every frame while leaving everything else alone. Those pairs are hard to produce, and harder the more complex the edit.

### Prior work

Earlier attempts to borrow from image editing fall into a few groups. One edits the first frame with an image editor, then has a video model propagate that change through the clip.

The appendix lists what goes wrong there. When the guidance is weak, the video model regenerates the motion instead of keeping the source motion, anything not visible in the first frame gets invented, and pair quality is capped by the propagation model.

Another group trains directly on image editing pairs, by repeating a frame into a clip or warping it into synthetic motion. A third changes the architecture or bolts on adapters.

The most direct option is offline distillation. The image editor edits each picture first, and the video model is fine-tuned to reproduce those edited pictures. That's the main baseline in this paper.

### Key idea

The starting intuition is simple. An image is just a one-frame video, so editing images well could be a foundation for editing video well.

So the video editor is asked to role-play as an image editor. It edits a single picture as if it were a one-frame clip, while a frozen image editor acts as the teacher.

The difference is in how the teaching happens. Offline distillation hands the student finished answers. I2VT lets the teacher step in at every denoising step of the student's own run and say how the current state should be updated.

### Method

A quick bit of background. A diffusion model starts from pure noise and removes it step by step. In flow matching, at each step the model predicts a velocity, basically which direction the current state should move to reach a clean result.

In this figure, offline distillation is on the left, I2VT in the middle, and inference on the right. During training, the student starts from noise and edits one image over ten denoising steps.

At each step, the student's current state goes to the teacher, the teacher returns the velocity it would use, and the student is trained to match it. Gradients only flow through the student; the teacher's answer is treated as a fixed target.

That only works because teacher and student share a VAE latent space, so the teacher can read the student's intermediate state directly. The teacher is JoyAI-Image-Edit, and it shares the Wan two point one VAE with all three students.

How hard each model follows the instruction is set by classifier-free guidance, or CFG, which you can read as how forcefully the model applies the prompt. All the scales default to one, and the teacher's is raised to two where the student is weak, namely actions and body movements, and for VideoCoF.

At inference the teacher is gone. The student edits whole clips with the same sampler and settings as the original editor, and even though it was only fine-tuned on single frames, the new skills carry over to the full video.

The training set is called I2VT-17K, built from individual frames of an existing video dataset, OpenVE-3M. That's seventeen thousand two hundred and twenty-seven images in total, each with one instruction for the teacher and one for the student, and no edited image or video anywhere.

For the new tasks, a vision-language model first describes each frame so a suitable edit can be picked. If the edit is to raise an arm, that arm has to be visible and not already raised.

Training is cheap. Each editor gets two hundred steps of LoRA fine-tuning, a method that trains only a small set of added weights. Because single frames mean far fewer visual tokens, the one point three billion parameter model needs about twenty A100 GPU-hours, and the fourteen billion parameter models twenty to twenty-five B300 GPU-hours.

### Results

The students are three existing video editors: Bernini-R at one point three billion and fourteen billion parameters, and VideoCoF.

There are two test sets. Basic edits use OpenVE-Bench. New skills use OpenVE++, which the authors built: action change, expression change, body movement and scene relighting, two hundred and seven clips, with source videos kept strictly apart from training.

Scoring is done mainly by Gemini two point five Pro acting as a judge, rating instruction compliance, consistency with the source, and visual quality. Each is on a one-to-five scale, which is the scale for every score on these two benchmarks.

Basic edits first, table one. All three editors go up: Bernini-R one point three B from three point one zero to three point four eight, VideoCoF from two point two nine to three point one nine, and Bernini-R fourteen B from three point three three to three point six eight.

Not every category moves much. VideoCoF's score for adding objects is essentially flat, two point five zero to two point five one.

Now the new skills, table two, which is the real headline. The base models barely do these edits. Bernini-R one point three B goes from one point six six to two point nine nine, and the fourteen B version from two point two five to three point two seven.

VideoCoF gains the most, from one point one seven to three point three three. It starts lowest and ends up level with the fine-tuned Bernini-R fourteen B.

All three of these new task groups were trained on single images only. The paper uses its action and expression examples to show that what was learned from pictures carries over to whole clips.

The obvious worry is that single-frame training could damage temporal consistency. Using VBench metrics like subject and background consistency, flicker and motion smoothness, the authors find the scores broadly comparable, with small ups and downs.

Against other editors, table three. Among the open-source video editors they compare, I2VT fourteen B scores three point five eight on OpenVE-Bench and three point three four on OpenVE++, the top score on both.

With Gemini three point eight Flash as the judge, the VideoCoF-based model reaches four point zero one on OpenVE++, the highest. The paper reports the two judges rank editors similarly, with a rank correlation of zero point nine one.

There's also a small human study: twenty people, each comparing sixteen pairs of edits. The one point three B model beats its base in sixty-eight point one percent of judgments and loses six point nine percent. Against the stronger Bernini-R fourteen B, the fourteen B version wins fifty point six percent and loses fifteen percent.

### Comparison

The comparison that matters most is against offline distillation, table four. Same one point three B student, same source images, same teacher. The default offline setup matches I2VT at twenty-four images and two hundred and forty supervised states per update.

Offline distillation does learn the new tasks, but on basic edits it trades one thing for another: every offline setting improves style edits and hurts background changes. I2VT has the best overall score on both pools, three point four eight on basic edits and two point nine nine on new tasks.

To be fair, some offline settings score slightly higher than I2VT on action changes and body movement.

This figure is one example. Asked to make the seated woman stand up, the offline model leaves her sitting, and I2VT has her standing.

In the ablations, a student guidance scale of one works best, and pushing it to five drops the score to one point eight eight, below the untouched base model. A stronger teacher scale doesn't help basic edits but does help the harder new tasks.

The appendix compares two related works that also learn video editing without video data. Both build a new editor on a text-to-video model by adding modules, and both evaluate only appearance edits such as style, color, adding or removing objects.

Worth flagging: the authors don't compare with those two directly, because one released no code and the other released code without trained weights.

My read is that the value here is post-training an existing editor with no architecture change, at low cost, and teaching it action and expression edits it could barely do before.

### Limitations

The paper has no dedicated limitations section, so these come from its own experiments and appendix.

First, scoring leans on Gemini as the judge. The human study had only twenty participants, volunteers recruited from the authors' acquaintances.

Second, across these three models, the stronger the base model, the smaller the gain. In the human study for the fourteen B model, half of the body-movement judgments were ties.

Third, on the outside benchmark IVEBench, all three models raise their overall score, but the one point three B model's fidelity to the source drops from eighty point two to seventy-seven point one.

Fourth, the method assumes teacher and student share a VAE latent space. The paper notes that a related method, whose image and video models use different VAEs, has to learn extra connectors to translate between them.

### Future work

The authors conclude that image editing supervision can improve video editing without new video training pairs. They say OpenVE++ will be released with the paper.

They also note that Google will retire Gemini two point five Pro from its API on December first, twenty twenty-six, so they report Gemini three point eight Flash scores for future comparisons.

My read is that the data idea is the interesting part. Image editing has far more data and models than video, and if single frames are enough to transfer the skill, video editing can ride along with image editing instead of waiting for expensive video pairs.

My read is also that most of the evidence comes from the authors' own benchmark and model judges, and it's still open how edits like changing an action hold up in longer, more complex footage.

That's today's Daily Frontier Papers. The paper is linked in the description.

