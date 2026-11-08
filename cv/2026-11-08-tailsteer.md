# TailSteer: Trajectory-Tail Steering for Drift-Resistant Multi-turn Image Editing

**改图二十轮走样少很多**

- 论文 Paper: [arXiv 2611.06741](https://arxiv.org/abs/2611.06741)
- 单位 Affiliation: 香港中文大学 MMLab · 商汤研究院
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-08
- 视频 Videos: [YouTube 中文](https://youtu.be/3OSwAQfz6Vg) · [YouTube English](https://youtu.be/LW5oeRpM2jE) · [B站](https://www.bilibili.com/video/BV1LDWJ6VEZ5)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇香港中文大学 MMLab 和商汤研究院的论文，合作方还有上海人工智能实验室，名字叫 TailSteer，意思是只调整去噪轨迹的尾巴，让多轮修图不再越改越走样。

你可能有过这种体验，让 AI 改一张图，改一次挺好，可在结果上接着改，改上十几二十轮，画面就慢慢变色、变糊，人脸也不像了。这篇论文专门治这个毛病。

### 问题

先说问题。现在的图像编辑模型，单轮效果已经很强，越来越多的产品，也让用户像聊天一样一轮轮地改图。

麻烦在于，每一轮的输出，就是下一轮的输入。每次多出一点没被要求的改动，都会被一轮轮继承下来，积累成明显的漂移，论文叫 drift。

表现就是，细节纹理被洗掉，不该动的物体和人脸被改，背景变形，最后出现明显瑕疵。所以单轮编辑强，不等于多轮编辑稳。

### 以前的方法

以前大致有三类做法。第一类是在模型外面加东西，比如用 mask 限定修改区域，加专门训练的 LoRA，或者做频率修正。

问题是，这种保真是靠约束模型换来的，会牺牲一部分生成能力，也往往只适合比较窄的局部编辑。

第二类是改表示空间，比如重新设计 VAE，或者绕过 VAE，直接在像素空间做扩散。作者说，这能减少单轮误差，但多轮之后还是明显退化。

第三类是在工作流层面加记忆，比如缓存之前的状态。它们记住了历史，但没有改进模型本身，通常还多了推理开销。

还有专门用多轮数据训练的方法，比如用视频序列，或者跨轮次的强化学习，但这等于让模型重学一遍编辑。

### 核心想法

这篇的思路是，不加外挂，直接把基础模型改得更耐改。那漂移从哪来？作者去看了扩散模型的去噪轨迹，也就是从纯噪声一步步还原出图像的过程。

这张图是关键。上面的曲线显示，低频部分，也就是大的轮廓和结构，很早就和原图很接近了，高频细节要到最后几步才急剧恢复。下面显示，不该改的区域，误差一直在降，但到最后也没降到零。

作者由此认为，轨迹的尾巴，也就是最后几步去噪，是细节定型的地方，残留误差在这里也还来得及纠正。

所以 TailSteer 的原则就一句话，前面的去噪步骤尽量和原模型保持一致，只在尾巴上，把结果往保真的方向推一推。

### 方法

方法分三步。第一步，往哪推。常规做法是准备人工整理的编辑前后图，但作者认为不合适。同一条指令，比如加一个苹果，模型每次生成的苹果，大小颜色都可能不同，都算对。

他们发现，目标其实就藏在原图里。没被要求修改的部分，本来就应该和原图一样。

具体来说，用冻结的原模型做两类编辑。一类是身份编辑，指令就是保持图片不变，整张图都要保留。另一类是局部编辑，比如把手里的羽毛换成一只小松鸡，修改之外的区域都要保留。

生成之后要过检查。身份编辑要求和原图高度相似、改动面积小，局部编辑要求改动足够集中，通过的才留下。

然后拼出训练目标，保留区域用原图，修改区域用模型自己的结果。也就是说，模型自己选的编辑方式不变，只纠正它不小心改动的地方，不需要人工标注的数据对。

第二步，作者先试了最简单的办法，只微调最后一步去噪。实验用的是 FLUX.2 klein 9B，每次编辑跑 50 步。

结果很有意思。前 49 步用原模型，只在第 50 步换成微调后的模型，人脸漂移明显减少，改到 20 轮还很稳定。但如果 50 步全用微调后的模型，图像会发灰、过度平滑，丢掉细节。

原因是所有步骤共用一套参数，只训练最后一步，也会扰乱前面的步骤。想让一个模型从头跑到尾，训练时就得把前面也管住。

第三步才是完整的 TailSteer。轨迹以一个叫 steering onset 的起点为界，分成两段。起点之前叫先验区，目标就是复现原模型的输出。起点之后叫过渡区，目标逐渐从原模型的方向，转向通往保真目标的方向。

做法是把这两个方向按权重混合，在起点处完全听原模型的，到最后一步完全指向保真目标。

另一个关键设计是 on-policy 训练。训练用的中间状态，不是从原模型的轨迹上直接拿，而是让正在训练的模型自己往下跑，在它真正走到的状态上算目标。这样纠正的，就是它自己实际会犯的偏差。

推理的时候，就是这一个模型从头跑到尾，不用切换模型，也不加插件。原模型和 mask，都只在训练时用。

### 实验和效果

实验覆盖三个家族的四个模型，FLUX.1 Kontext、FLUX.2 klein 9B、SenseNova U1.5，还有 Qwen-Image-2.1。

训练数据来自 11947 张源图，每张配三条局部编辑指令和一条身份编辑指令。每个模型训练 2000 步，用 8 张 H100。

先看定性结果。这张图是对一张桌面照片连续改 20 轮。论文报告，几种对比方法都有累积漂移，TailSteer 版本到第 20 轮，桌面颜色、咖啡杯、拉花和饼干都还在。

为了单独量出漂移，作者做了一个基准，叫 Add & Preserve。100 张细节丰富的照片，在随机位置放一个红框，连续 10 轮在框里加或删物体，测框外和原图的差别，以及框里的物体对不对。

结果在表 2。以 FLUX.2 klein 为例，起点设在第 46 步，第 10 轮框外的 PSNR，从原模型的 13.85 提到 19.96，SSIM 从 0.289 提到 0.621，LPIPS 从 0.524 降到 0.290，这一项越低越好。

作为参照，GPT-Image-2.5 Sunburst 通过 API 测出的这三项，是 18.55、0.540 和 0.378。作者称，TailSteer 版的 klein，三项保真指标都超过了它。

SenseNova 的提升也很大，PSNR 从 11.23 提到 18.22，框内物体的准确率基本没变。

这张图是 10 轮里漂移的累积曲线，所有模型上，TailSteer 都减缓了漂移。作者还发现，保真最好的是起点偏早或居中的版本，不是最靠后的。

那原有能力掉了没有？表 1 测了文生图和单轮编辑。比如 klein 的 GEdit 英文分，原模型 7.47，加了 TailSteer 是 7.52，基本持平。

对比之下，klein 加 ConsLoRA 会掉到 6.58，加 FreqEdit 掉到 7.20。

消融在表 3，同样的数据，不同的训练方式。普通监督微调能减少漂移，但编辑能力变差，GEdit 掉到 5.24，图里它删不掉物体。只训练最后一步，GEdit 更是掉到 2.24。

去掉 on-policy，GEdit 和保真都接近完整版，但考推理的 RISE 从 13.61 掉到 11.39。图里它没能把奶酪摆成三角形。

表 4 显示，数据越多，抗漂移通常越好，两类轨迹混着用，保真和编辑的平衡最好。正式设置用了 38540 条轨迹，局部和身份编辑大约三比一。

### 和以前方法的对比

和以前比，最大的区别在下手的位置。外挂模块和推理时修正，是在模型外面管住它，多轮数据训练，是让模型重学编辑，TailSteer 是把能力直接训练进模型，推理时没有额外组件。

作者还强调，它只用单轮编辑的轨迹训练，却改善了多轮表现，在多图合成和骨架引导编辑上也有效。

我的判断是，这篇最值得学的是诊断思路。先看清漂移发生在去噪的哪个阶段，再只在那里动手，所以改动小，副作用也小。

我的判断是，也要看清弱点。表 2 里，klein 不管加不加 TailSteer，框内物体准确率都只有百分之三十一左右，GPT-Image-2.5 是 98.29。所以超过 GPT 只是在保住背景这一项，该加的东西加没加对，差距还很大。

### 局限

论文没有单独的局限一节，下面几点来自它自己的数据。第一，在 MSE-Bench 这个通用多轮基准上，提升不稳定，klein 有两个设置，成功率从百分之四十掉到三十二和三十三。作者也承认，指令遵循的提升不一致。

第二，考推理的 RISE 分数，多数设置下降，比如 Qwen-Image 从 20.56 掉到 18 点几。

第三，提升幅度因模型而异。Qwen-Image 第 10 轮的 PSNR，只从 16.93 提到最多 18.38，略低于 GPT 的 18.55。

第四，起点设在哪一步，要每个模型分别挑，论文每个模型试了三个起点，结果差别不小。

第五，作者推测，颜色偏移可能还有一个来源，就是每轮之间图片反复经过 VAE 编码解码。他们试了跳过 VAE、直接传 latent，和 TailSteer 一起用，第 10 轮的 PSNR 能和普通设置下的第 5 轮相当。

### 未来方向

作者认为，更大、更多样的自生成轨迹数据，还能继续提升抗漂移能力，复用 latent 或者更好的 VAE，也能和 TailSteer 互补。

我的判断是，这个方法对做产品的人很实用。它只是最后阶段的一次微调，不要人工标注，推理成本不变，论文在三个模型家族上都验证过。

我的判断是，接下来值得看两件事，一是更长的编辑链条，论文最多测到 20 轮，二是框内的编辑准确率，能不能和保真一起提上去。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from MMLab at the Chinese University of Hong Kong and SenseTime Research, with the Shanghai AI Laboratory. It's called TailSteer, and the idea is to adjust only the tail of the denoising trajectory, so that images stop falling apart when you edit them over and over.

You may have seen this yourself. Ask an AI model to edit a picture once and it looks great. Keep editing the result, ten or twenty rounds, and the colors shift, the details go soft, and the face slowly stops looking like the same person. This paper goes after exactly that problem.

### Problem

Here's the setup. Single-turn image editors are now very good, and more and more products let you refine an image turn by turn, the way you'd keep talking in a chat.

The trouble is that in a multi-turn workflow, each edited output becomes the next input. Every edit adds a tiny change nobody asked for, and those changes get inherited turn after turn until they add up to visible drift.

The paper lists the symptoms: fine textures washed out, details of untouched objects and faces altered, background structure distorted, and eventually obvious artifacts. So being strong at one edit doesn't make a model reliable over many.

### Prior work

Earlier fixes fall into roughly three families. The first bolts things onto the model: masks that limit where the edit can happen, specially trained LoRA adapters, or corrections in the frequency domain.

The catch is that these buy preservation by constraining the model. They give up part of its generative ability, and they tend to work only for a narrow range of local edits.

The second family changes the representation, for example a redesigned VAE, or skipping the VAE and running diffusion directly in pixel space. The authors say these reduce error within one edit, but still degrade noticeably across repeated edits.

The third adds memory at the workflow level, caching earlier states or using a reasoning model to plan. That keeps history around, but it doesn't make the model itself any better, and it usually costs extra inference.

There's also training on multi-turn data directly, using video sequences or reinforcement learning across turns. That amounts to teaching the model to edit all over again.

### Key idea

This paper takes a different route: no add-ons, just make the base model itself hold up better. So where does drift come from? The authors look at the denoising trajectory, the path from pure noise to a finished image, step by step.

This figure is the key. The top curves show that low-frequency content, the broad shapes and layout, matches the source image early on, while high-frequency detail only comes back sharply in the last few steps. The bottom row shows that error in regions that shouldn't change keeps falling, but never reaches zero at the end.

From that, the authors conclude that the tail of the trajectory, the final few denoising steps, is where details get settled, and where leftover errors can still be corrected.

So TailSteer's principle fits in one line. Keep the early denoising steps as close to the original model as possible, and only nudge the tail toward preserving what should stay the same.

### Method

The method comes in three steps. Step one is deciding what to steer toward. The usual answer is curated before-and-after editing pairs, but the authors argue that doesn't fit. Ask a model to add an apple, and across seeds you get apples of different sizes and colors, all of them valid.

Their observation is that the target is already sitting in the source image. Whatever the user didn't ask to change should simply match the original.

Concretely, they run the frozen original model on two kinds of edits. One is an identity edit, where the instruction is to keep the image exactly unchanged, so the whole image should be preserved. The other is a local edit, like replacing the feather someone is holding with a small grouse chick, where everything outside the edit should be preserved.

The results then go through checks. Identity edits must stay highly similar to the source with only a small changed area. Local edits must keep the change localized. Only samples that pass are kept.

Then they assemble the training target: the source image in the preserved region, and the model's own result in the edited region. In other words, the model keeps the edit it chose, and only its accidental changes get corrected. No human-labeled editing pairs are needed.

Step two is the simplest possible intervention: fine-tune only the final denoising step. The test model is FLUX.2 klein 9B, with fifty sampling steps per edit.

The result is telling. Run the original model for the first forty-nine steps and the fine-tuned one only for step fifty, and facial drift drops sharply, staying stable even after twenty turns. But run the fine-tuned model for all fifty steps, and the images come out desaturated and overly smooth, losing detail.

The reason is that every step shares the same parameters. Training only the last step also disturbs what the model does earlier. If you want one model to run the whole trajectory, training has to keep the early steps in check too.

Step three is the full TailSteer. A point called the steering onset splits the trajectory in two. Before it is the prior region, where the target is simply to reproduce the original model. After it is the transition region, where the target gradually shifts from the original model's direction toward the direction of the preservation target.

In practice, the two directions are blended with a weight. At the onset, the weight follows the original model completely. By the final step, it points entirely at the preservation target.

One more key design is on-policy training. The intermediate states used for training don't come straight from the original model's trajectory. Instead, the model being trained rolls forward itself, and the target is computed at the state it actually reaches. That way it learns to correct the deviations it really makes.

At inference, this single model runs the entire trajectory on its own. No model switching and no plug-ins. The original model and the masks are used only during training.

### Results

The experiments cover four models from three families: FLUX.1 Kontext, FLUX.2 klein 9B, SenseNova U1.5, and Qwen-Image-2.1.

Training data comes from eleven thousand nine hundred forty-seven source images, each paired with three local-edit instructions and one identity instruction. Each model is trained for two thousand updates on eight H100 GPUs.

Start with the qualitative results. This figure shows twenty consecutive edits to a photo of a tabletop. The paper reports that the competing methods all drift, while the TailSteer version still keeps the table color, the coffee cup, the latte art and the cookie at turn twenty.

To measure drift on its own, the authors built a benchmark called Add and Preserve. It has one hundred detail-rich photos. A red box is placed at a random spot, and over ten turns objects are added or removed inside it. They measure how far everything outside the box has moved from the original, and whether the objects inside the box are right.

The numbers are in Table 2. Take FLUX.2 klein with the onset at step forty-six. At turn ten, PSNR outside the box rises from thirteen point eight five to nineteen point nine six. SSIM goes from zero point two eight nine to zero point six two one. And LPIPS, where lower is better, falls from zero point five two four to zero point two nine.

For reference, GPT-Image-2.5 Sunburst, tested through its API, scores eighteen point five five, zero point five four, and zero point three seven eight on those three. The authors report that TailSteer klein beats it on all three preservation metrics.

SenseNova improves a lot too, with PSNR going from eleven point two three to eighteen point two two, while accuracy on the objects inside the box barely moves.

This figure plots drift accumulating over ten turns. TailSteer slows it on every model. The authors also find that the best preservation comes from early or middle onsets, not the latest ones.

So did the models lose anything? Table 1 covers text-to-image and single-turn editing. On the English GEdit score, klein goes from seven point four seven to seven point five two with TailSteer. Basically unchanged.

Compare that with the add-ons: klein with the ConsLoRA adapter drops to six point five eight, and with FreqEdit to seven point two.

The ablations are in Table 3: same data, different training schemes. Standard supervised fine-tuning reduces drift but hurts editing, with GEdit falling to five point two four, and in the figure it fails to remove objects. Training only the last step is worse still, at two point two four.

Without on-policy training, GEdit and preservation stay close to the full method, but RISE, a reasoning-heavy editing benchmark, drops from thirteen point six one to eleven point three nine. In the figure, that version fails to arrange the cheese into a triangle.

On data, Table 4 shows that more data generally means better drift resistance, and mixing both trajectory types gives the best balance between preservation and editing. The main setting uses thirty-eight thousand five hundred forty trajectories, local to identity edits at about three to one.

### Comparison

Compared with earlier work, the big difference is where the fix lives. Add-on modules and inference-time corrections hold the model in from the outside. Multi-turn training makes the model relearn editing. TailSteer trains the robustness into the model itself, with nothing extra at inference.

The authors also stress that it's trained only on single-edit trajectories, yet it improves multi-turn behavior, and it carries over to multi-image compositing and skeleton-guided editing.

My read is that the most reusable part is the diagnosis. Find out which stage of denoising the drift comes from, then intervene only there. That's why the change is small and the side effects are small.

My read is that the weak spot deserves a clear look. In Table 2, klein's accuracy on the objects inside the box sits around thirty-one percent with or without TailSteer, while GPT-Image-2.5 is at ninety-eight point two nine. So beating GPT applies to keeping the background intact. On getting the requested edits right, the gap is still large.

### Limitations

The paper has no dedicated limitations section, so these points come from its own numbers. First, on MSE-Bench, a general multi-turn benchmark, the gains aren't consistent. Two klein settings drop in success rate from forty percent to thirty-two and thirty-three. The authors acknowledge that instruction-following gains are less consistent.

Second, the reasoning-heavy RISE score goes down in most settings. Qwen-Image, for example, falls from twenty point five six to around eighteen.

Third, the size of the gain depends on the model. Qwen-Image's turn-ten PSNR only rises from sixteen point nine three to at most eighteen point three eight, still a bit below GPT's eighteen point five five.

Fourth, the onset step has to be picked per model. The paper tries three onsets for each, and the results vary quite a bit between them.

Fifth, the authors suggest another likely source of color drift: the image goes through VAE encoding and decoding again between every turn. When they skip that and pass the latent forward instead, combined with TailSteer, turn-ten PSNR is comparable to turn five with the standard setup.

### Future work

The authors expect larger and more diverse self-generated trajectory data to push drift resistance further, and they see latent reuse or better VAEs as complementary to TailSteer.

My read is that this is practical for people shipping products. It's a single final-stage fine-tune, with no labeled data and no added inference cost, and it was validated on three model families.

My read is that two things are worth watching next: longer editing chains, since the paper tests at most twenty turns, and whether in-box editing accuracy can rise along with preservation.

That's today's Daily Frontier Papers. The paper is linked in the description.

