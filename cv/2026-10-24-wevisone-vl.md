# WeVisOne-VL: Unified Visual Perception from Objects to Pixels

**会找会数还会说没有**

- 论文 Paper: [arXiv 2610.31329](https://arxiv.org/abs/2610.31329)
- 单位 Affiliation: 腾讯微信视觉团队
- 方向 Direction: CV
- 讲解日期 Covered: 2026-10-24
- 视频 Videos: [YouTube 中文](https://youtu.be/7JefVNuVqpQ) · [YouTube English](https://youtu.be/eaHd6GokK0E) · [B站](https://www.bilibili.com/video/BV122HK6DECw)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇腾讯微信视觉团队的论文，模型叫 WeVisOne-VL，意思是微信视觉的一体化模型。

它要做的事情很具体，让一个视觉语言模型，不光能看懂图，还能把东西准确地找出来、数清楚，一直精确到每一个像素。

### 问题

先说背景。视觉语言模型，简称 VLM，就是能同时看图和读文字的大模型，描述一张图已经做得不错了。但论文开头就说，在细粒度的视觉感知上，它们还不可靠。

作者列了三种典型毛病。会把图里没有的东西说成有，碰到不熟悉的描述会搞错用户要的是哪个，还会漏掉不显眼的物体，找不全。

### 以前的方法

以前的路大概是这样。先是专门的检测和分割模型，比如 YOLO 和 DETR，后来有开放词表的 Grounding-DINO，和能分割任何东西的 SAM，再到现在的视觉语言模型，能认的东西越来越多。

但作者认为，把效果和效率放在一起算，现在的模型离实际部署还差得远。

如果要像素级的输出，比如把一个物体的轮廓抠出来，有的方法会在大模型后面再接一个单独的大分割模型，比如 Sa2VA 接的就是 SAM 2。这篇论文想把这一层去掉。

### 核心想法

核心想法一句话，把所有视觉任务都变成按格式说话，也就是结构化的下一个 token 预测。检测、指代、计数、指点、文字识别，输出的都是一串带坐标的文字。

坐标怎么写。他们加了一组专门的 token，从 0 到 1000，表示归一化之后的坐标位置。一个框是四个这样的 token，一个点是两个，目标不存在，就输出 None。

需要像素级输出的时候，模型在每个框后面多生成一个特殊 token，叫 OBJ。这个 token 的隐藏状态就代表这个物体，交给一个很小的像素解码器，画出掩码或者透明度蒙版。

这里解释两个词。掩码，就是每个像素属不属于这个物体，非黑即白。抠图，英文叫 matting，输出的是 alpha 蒙版，每个像素有一个零到一之间的透明度，头发丝、半透明的边缘都能保留下来。

### 方法

先说模型。底座是 Qwen3-VL，做了 2B 和 4B 两个尺寸。

像素解码器在这张图里，分三部分。第一部分，整理视觉编码器中间几层的特征。第二部分借鉴 Deformable DETR，用 OBJ token 的隐藏状态和预测出来的框，生成每个物体的查询，去图像特征里取信息。第三部分把查询和特征结合，输出整张图的掩码，抠图再多一个由粗到细的细化模块。

整个解码器，连细化模块在内，只有大约 960 万参数。

再说数据。他们搭了一个数据引擎，标注库超过 1 亿张图，大约 36 亿个框。

流程在这张图里。先用知识图谱整理类别和描述语，有针对性地找图。然后让 SAM3 和自研的开放词表检测器一起出候选框，尽量不漏，再让 Qwen3-VL 逐个核对，框和描述对不对得上。框确认之后，用 SAM2 生成掩码，抠图的标注，再单独训练一个标注模型来生成。

还有一个细节，叫难负样本。比如把红色的车改成蓝色的车，图里不一定就没有蓝色的车，所以每个候选的负样本，都要对着整张图复核。模型仍然在这种查询上画了框的，就变成难负样本，专门教它说没有。

训练分四步，前三步是监督微调，按课程一步步加难度。第一阶段只训练语言模型部分，学会新 token 和输出格式，用了大约 9000 万条样本。

第二阶段全部解冻，把每张图最多的视觉 token 提到 1960 个，一次回答里最多的框数提到 500 个，专门练小物体和密集场景。

第三阶段接上像素解码器。先用 2.5 亿条分割样本预热解码器，再和模型一起训练，下一个 token 预测和像素任务一比一交替，保住定位能力。之后冻结模型，用 1.3 亿条抠图样本单独微调解码器。

第四步是强化学习，用的是 GRPO。先挑模型最没把握的样本，也就是同一道题采样几次，有时对有时错的那种，挑出大约 14 万条，加上补充数据，一共 17.8 万条。

奖励按整个回答来算。预测框和真实框一一匹配，按 IoU 的三次方累加，漏检和多检都会拉低分数。IoU 就是两个框的重叠程度，取三次方，是让高精度定位的奖励更敏感。分割和抠图在强化学习里也只用框的奖励，没有直接评价掩码。

### 实验和效果

实验覆盖七类任务，五个物体级，两个像素级，作者称一共超过 30 个基准。对比对象主要是 Qwen 系列的通用模型，和两个同类的定位模型，Rex-Omni 和 LocateAnything。

先看检测，表 2。指标是 F1，在 IoU 0.5 到 0.95 之间取平均。COCO 上，4B 版本是 57.1，Rex-Omni 是 52.9，LocateAnything 是 54.7。

提升最大的是密集场景。Dense200 上，4B 是 71.3，对比模型里最好的 LocateAnything 是 58.7，Qwen3-VL-4B 只有 15.2。

指代检测在表 3。论文自己做了一个中英文测试集，叫 WeRef-Bench，2880 条查询。4B 版本平均 F1 是 78.7，对图里没有目标的查询，拒绝率是 96.4%，Rex-Omni 和 LocateAnything 分别是 43.3% 和 73.0%。

RefCOCOg 的拒绝测试集更明显，里面每条描述都改过，图里没有对应的物体。4B 拒绝了 67.4%，Rex-Omni 和 LocateAnything 都不到 1%，Qwen3-VL-8B 是 22.5%。

计数在表 4，指标是完全数对的比例，差一个都算错。PixMo-Count 测试集上，4B 是 89.6%，难度更高的 MixCount 测试集上是 40.4%，LocateAnything 是 15.2%。

论文这张图里有个例子，问图里有多少个假的绿苹果，正确答案是 42，Rex-Omni 数出了 537，WeVisOne-VL 数对了。当然，这是作者挑出来的例子。

文字识别在表 6，要求框对，字也完全对。在票据数据集 SROIE 上，4B 的平均 F1 是 68.1，专门的 OCR 系统 PP-OCRv5 是 58.6。

再看像素级任务。指代分割在表 7，RefCOCO 验证集上，4B 是 81.4，比表里最好的对比方法 UniPixel-7B 的 80.8 略高。差距更大的是 Ref-ZOM 的拒绝率，4B 是 86.0%，对比方法里最高的 GSVA 是 41.9%，其余对比方法都在 26% 以下。

抠图在表 8，误差越低越好。在 AM-2K 上，用同样的真实框做提示，4B 的平均绝对误差是 5.0，SAM2-Matting 最好的一档是 11.0。在他们自建的 WeRefMat 上，用文字描述指定目标，4B 是 9.8，先用 Qwen3.5-9B 找框、再交给 SAM2-Matting 的组合是 22.4。

### 和以前方法的对比

和以前的方法比，思路上并不是全新的。论文自己在相关工作里写了，LISA 就是把生成的分割 token 交给一个掩码解码器，坐标 token 的做法，也借鉴了 Rex-Omni 和 LocateAnything。

它的不同在于组合。一个大约 960 万参数的解码器，用框做引导，用模型自己的视觉特征，不再接一个单独的大分割模型。同一个模型，从框、点、计数、文字，一直到掩码和抠图，全包了。

我的判断是，这篇最有价值的地方不是某一个分数，而是会说没有。各个任务里，拒绝率的差距比定位精度的差距大得多，而这正是实际部署时最容易出问题的地方。

我的判断是，也要看到几处短板。COCO 上最严格的 IoU 0.95 标准下，专门检测器 GroundingDINO 是 23.0，比 4B 的 19.5 高，LVIS 的 0.95 标准下，LocateAnything 也更高。抠图的几个公开数据集上，2B 有时反而比 4B 好。论文也没有消融实验，看不出数据引擎和强化学习各自贡献了多少，对比模型最大只到 13B。

### 局限

作者自己写了三条局限。第一，通用训练数据有限，模型的通用能力，可能比底座模型差。

第二，为了推理效率，模型直接输出结构化结果，不做思维链推理，复杂的推理任务可能会吃亏。

第三，现在只支持单轮预测，还不能通过多轮调用工具，一步步看得更清楚。

另外作者也提到，基准里的描述都很清楚，但真实用户的指令往往含糊、犹豫，理解意图会难很多。

### 未来方向

未来方向，作者提到可以接入 Agent 框架，通过多轮工具交互逐步细化理解，并且继续加强意图推理。

论文还展示了一个附带的能力，带框的细粒度图像描述，描述里提到的每个物体，都对应图上的一个框。作者认为，这种数据可能对训练别的视觉语言模型有用。

我的判断是，这类小而专的感知模型，很适合当大模型 Agent 的眼睛，大模型负责想，它负责准确地找和抠，4B 这个尺寸，部署成本也相对可控。

我的判断是，接下来真正要看两件事，一是在作者自建的测试集之外，别人复现的结果怎么样，二是它的通用能力到底掉了多少，论文没有给出这方面的数字。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the WeChat Vision team at Tencent, and the model is called WeVisOne-VL, short for WeChat Vision All-in-One.

The goal is concrete. Take a vision-language model that can already describe a picture, and make it find things precisely, count them correctly, and outline them down to the pixel.

### Problem

Some background first. Vision-language models, or VLMs, read images and text together, and they're good at describing a scene. But the paper opens by saying they're still unreliable at fine-grained visual perception.

The authors name three typical failures. Models claim objects that aren't there, they misread which object you mean when the description is unfamiliar, and they skip objects that aren't prominent, so they don't find them all.

### Prior work

The path so far looks like this. First came dedicated detectors and segmenters like YOLO and DETR, then open-vocabulary detectors like Grounding-DINO and segment-anything models like SAM, and now large vision-language models. Each step widened the concepts a model knows and the instructions it can follow.

But the authors argue that once you count both quality and efficiency, current models are still far from ready for real deployment.

For pixel-level output, like cutting out an object's exact outline, some approaches bolt a separate large segmentation model onto the VLM. Sa2VA, for instance, couples one with SAM 2. This paper wants to remove that extra layer.

### Key idea

The key idea in one line: treat every vision task as structured next-token prediction. Detection, referring, counting, pointing and text recognition all come out as a string of text with coordinates in it.

How are coordinates written? The authors add a set of dedicated tokens, from zero to one thousand, for normalized positions. A box is four of these tokens, a point is two, and if the target isn't in the image the model writes None.

When pixel-level output is needed, the model emits one extra special token after each box, called OBJ. That token's hidden state stands for the object, and it goes to a small pixel decoder that draws a mask or a transparency matte.

Two terms here. A mask says, for each pixel, whether it belongs to the object, yes or no. Matting produces an alpha matte instead, where each pixel gets a transparency between zero and one, so strands of hair and soft edges survive.

### Method

Start with the model. It's built on Qwen3-VL, in two sizes, two billion and four billion parameters.

The pixel decoder is in this figure, and it has three parts. The first gathers features from several intermediate layers of the vision encoder into one shared multi-level memory. The second borrows from Deformable DETR: it turns the OBJ token's hidden state and the predicted box into a query for each object, which pulls information from those image features. The third combines query and features into a full-image mask, and matting adds a coarse-to-fine refinement module.

The whole decoder, refinement module included, has only about nine point six million parameters.

Now the data. A large part of the paper describes their data engine, whose annotation pool covers over one hundred million images and about three point six billion boxes.

The pipeline is in this figure. A knowledge graph organizes categories and referring phrases and steers which images to collect. Then SAM3 and an in-house open-vocabulary detector both propose boxes, to miss as little as possible, and Qwen3-VL checks each box against the query. Once a box is confirmed, SAM2 makes the mask, and the matting labels come from a separately trained annotator model.

One detail is hard negatives. Changing red car to blue car doesn't mean the image has no blue car, so every candidate negative is reviewed against the whole image. If the model still draws a box for such a query, it becomes a hard negative that teaches the model to say nothing is there.

Training has four steps, and the first three are supervised fine-tuning with rising difficulty. Stage one trains only the language model, so it learns the new tokens and output formats, on about ninety million examples.

Stage two unfreezes everything, raises the cap to one thousand nine hundred sixty visual tokens per image and five hundred boxes per answer, and drills small objects and crowded scenes.

Stage three attaches the pixel decoder. The decoder is first warmed up on two hundred fifty million segmentation samples, then trained together with the model, alternating next-token prediction and pixel tasks one to one so localization doesn't degrade. After that the model is frozen and the decoder alone is fine-tuned on one hundred thirty million matting samples.

Step four is reinforcement learning with GRPO. They first pick the examples the model is least sure about, the ones where repeated samples are sometimes right and sometimes wrong. That gives about one hundred forty thousand, and with supplementary data, one hundred seventy-eight thousand in total.

The reward scores the whole answer. Predicted and true boxes are matched one to one and the IoU of each pair is cubed and summed, so both missed and extra objects pull the score down. IoU is how much two boxes overlap, and cubing it makes the reward more sensitive to very precise boxes. For segmentation and matting, reinforcement learning still uses only the box reward, with no direct score on the masks.

### Results

The experiments cover seven task types, five at the object level and two at the pixel level, and the authors say over thirty benchmarks in all. The main comparisons are the general-purpose Qwen models and two grounding models of the same kind, Rex-Omni and LocateAnything.

Detection first, in Table 2. The metric is F1 averaged over IoU thresholds from zero point five to zero point nine five. On COCO, the four billion model scores fifty-seven point one, against fifty-two point nine for Rex-Omni and fifty-four point seven for LocateAnything.

The biggest jump is in crowded scenes. On Dense200, the four billion model reaches seventy-one point three, the best baseline, LocateAnything, has fifty-eight point seven, and Qwen3-VL at four billion has just fifteen point two.

Referring detection is in Table 3. The authors built their own Chinese and English test set, WeRef-Bench, with two thousand eight hundred eighty queries. The four billion model gets a mean F1 of seventy-eight point seven, and on queries whose target isn't in the image it refuses ninety-six point four percent of the time, versus forty-three point three for Rex-Omni and seventy-three for LocateAnything.

The RefCOCOg rejection set makes the gap starker. Every description there has been altered so nothing in the image matches. The four billion model rejects sixty-seven point four percent, Rex-Omni and LocateAnything both stay under one percent, and Qwen3-VL at eight billion gets twenty-two point five.

Counting is in Table 4, scored as exact matches, so being off by one counts as wrong. On the PixMo-Count test split the four billion model gets eighty-nine point six percent, and on the harder MixCount test split it gets forty point four, against fifteen point two for LocateAnything.

This figure has an example: how many fake green apples are in the image? The answer is forty-two. Rex-Omni says five hundred thirty-seven, and WeVisOne-VL gets it right. Of course, the authors chose these examples.

Text recognition is in Table 6, where both the box and the exact text must be right. On the SROIE receipts set, the four billion model has a mean F1 of sixty-eight point one, while the dedicated OCR system PP-OCRv5 has fifty-eight point six.

Now the pixel-level tasks. Referring segmentation is in Table 7. On RefCOCO validation the four billion model scores eighty-one point four, just above the best baseline in the table, UniPixel at seven billion, with eighty point eight. The bigger gap is the rejection rate on Ref-ZOM: eighty-six percent for the four billion model, while the best baseline, GSVA, reaches forty-one point nine and the other baselines stay below twenty-six.

Matting is in Table 8, where lower error is better. On AM-2K, with the same ground-truth box as the prompt, the four billion model's mean absolute difference is five point zero, and the best SAM2-Matting variant is eleven point zero. On their own WeRefMat set, where a text description picks the target, the four billion model scores nine point eight, while Qwen3.5 at nine billion finding the box and handing it to SAM2-Matting scores twenty-two point four.

### Comparison

Compared with earlier work, the idea isn't entirely new. The paper's own related-work section notes that LISA already sends a generated segmentation token to a mask decoder, and the coordinate tokens follow Rex-Omni and LocateAnything.

What's different is the combination. A decoder of about nine point six million parameters, guided by the box and reading the model's own visual features, with no separate large segmentation model. One model covers boxes, points, counts and text, all the way to masks and mattes.

My read is that the most valuable thing here isn't any single score, it's the ability to say nothing is there. Across tasks, the gaps in rejection rate are far larger than the gaps in localization accuracy, and that's exactly where deployed systems tend to break.

My read is that there are also weak spots. At the strictest IoU of zero point nine five on COCO, the specialized detector GroundingDINO scores twenty-three, above the four billion model's nineteen point five, and LocateAnything is ahead at that threshold on LVIS. On several public matting sets the two billion model beats the four billion one. There are no ablations, so we can't see how much the data engine or reinforcement learning each contribute, and the largest comparison model is thirteen billion parameters.

### Limitations

The authors list three limitations. First, general-purpose training data is limited, so the model's general abilities may fall below the base model's.

Second, for speed, the model outputs structured results directly with no chain-of-thought reasoning, which may hurt on complex reasoning tasks.

Third, it only does single-turn prediction, so it can't yet refine what it sees through several rounds of tool use.

The authors also point out that benchmark prompts are clear, while real users are often vague and hesitant, which makes understanding intent much harder.

### Future work

For future work, the authors suggest plugging the model into an agent framework that refines its understanding over multiple tool calls, and strengthening intent reasoning.

The paper also shows a side ability: detailed captions where every object mentioned is tied to a box in the image. The authors think such data could help train other vision-language models.

My read is that small, specialized perception models like this fit well as the eyes of a large-model agent. The big model does the thinking, this one finds and cuts out objects precisely, and at four billion parameters it's reasonably cheap to deploy.

My read is that two things matter next. How the results hold up when others reproduce them outside the authors' own test sets, and how much general ability the model actually lost, since the paper gives no numbers on that.

That's today's Daily Frontier Papers. The paper is linked in the description.

