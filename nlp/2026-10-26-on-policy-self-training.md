# On-Policy Self-Training: Data-Free Self-Improvement in Large Language Models

**不给题也不给答案模型自己变强**

- 论文 Paper: [arXiv 2610.32770](https://arxiv.org/abs/2610.32770)
- 单位 Affiliation: Google · 乌尔姆大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-10-26
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1Rgnd6vEkW)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇谷歌和德国乌尔姆大学的论文，名字叫 On-Policy Self-Training，简称 OPST，副标题是，大语言模型不靠数据的自我提升。

它问的问题很大胆。不给模型任何人工写的题目，也不给答案，只告诉它一些学科的名字，比如抽象代数，它能不能自己出题，自己教自己，变得更强？

### 问题

先说背景。大模型预训练之后，还有一个后训练阶段，能力的提升，很大一部分来自这里。

但几乎每一种后训练方法，都离不开外部监督。SFT 要参考解答，带可验证奖励的强化学习要标准答案，在线蒸馏要一个更强的老师模型。

这些监督很贵。论文举了个例子，光是给一个数学数据集做过程标注，就用了大约 80 万条人工的步骤标签。

已经有方法让模型自己生成答案或者推理过程，但题目还是人写的。作者说，这意味着人挑的题目，仍然决定了模型能学到什么。

### 以前的方法

这篇最直接的基础，叫 on-policy self-distillation，在线自蒸馏，简称 OPSD。

先解释在线蒸馏。学生模型自己答题，老师模型在学生写出的每个位置，给出下一个 token 的概率分布，学生去贴近它。这样的监督逐 token，很密集，而且落在学生自己真正会走到的地方。

自蒸馏不需要更强的老师。老师和学生是同一个模型，只是老师额外看到一份学生看不到的信息，叫特权信息，通常就是标准解答。论文这张图，画的就是这个结构。

问题是，现有的 OPSD 还是假设有一个现成的题库。有工作用模型多次作答的多数投票，代替了参考解答，但题目还是人给的。

### 核心想法

作者的想法分两步。第一步，连题目都不要了，只给模型一个粗略的主题，让它自己出题，同时写出解题思路和一个候选答案。

第二步更关键。自己生成的答案可能是错的，所以不拿它当学生要模仿的目标，而是当成提示，只给老师看。

老师看到提示以后，用自己的知识写出完整的推理，学生学的是老师的推理。作者认为，这样学生得到的监督，可以超过生成答案本身包含的信息。

论文的第一张图就是这个循环。按主题生成数据，过滤，训练，再生成，一轮一轮地转。

### 方法

具体分三块。第一块是出题。主题用的是现成的类别名，比如 MMLU 用大学数学这样的学科名，GSM8K 用行程问题这样的题型名。

第二块是过滤，有两道关。第一道叫伪验证。让模型在全新的上下文里，不看自己写的解答，重新做一遍，答案要一致。再把题目换个说法，再做一遍，答案还要一致，换个说法答案就变了，通常说明这道题有歧义。

第二道关叫共识门。学生对这道题作答多次，如果每次都和候选答案一致，说明学生已经会了，这道题就扔掉。

两道关方向相反。一道留下足够稳定的题，一道去掉太简单的题，剩下的正好在学生能力的边缘，而且难度会随着学生的变化，跟着调整。

第三块是训练。学生只看题目，自己写解答，老师看题目加特权信息，在学生写的每个位置给出下一个 token 的分布，学生按 token 去对齐。

老师也不是固定的。它从学生初始化，每一步训练之后，用指数滑动平均，也就是 EMA，慢慢跟上学生的权重，这样一批噪声数据不会马上把老师带偏。最后拿去评测的，也是这个 EMA 老师。

整个流程重复多轮，每一轮重新生成一批新题，去掉和前几轮重复的。

这套方法还能用到图像上，叫 Visual OPST。不需要标注，用的是没有标注的图片和视觉主题，模型针对图片自己出题，特权信息是它自己写的图片描述，或者从它检测出的物体位置推出来的空间关系。

### 实验和效果

实验用的是 Gemma 4 的 E2B 和 E4B 两个小模型，主题覆盖 MMLU、GPQA、GSM8K、HellaSwag 四个基准的领域。另外三个基准没有任何主题针对，用来看模型会不会忘掉别的能力。

第一组对比最干净，在表 1。同一批自己生成的数据，三种用法。SFT 直接模仿生成的答案，GRPO 把和它一致当成奖励，OPST 把它作为老师的上下文。

结果是，E2B 不开思考模式，MMLU 从 37.5 提到 47.7，涨了 10.2 个点。平均分上，OPST 比同样数据的 SFT 高 5.6 个点，E4B 上高 1.3 个点。开了思考模式，两个模型分别高 2.7 和 3.8 个点。

两者唯一的区别，就是生成的答案拿来模仿，还是拿来给老师当提示。只有一处是 SFT 领先，E4B 不开思考的 HellaSwag，领先 0.4 个点。

多领域实验在表 3。把三个领域的主题混在一起训练，领域内平均涨了 7.6 个点，和分开单独训练只差 1 个点以内。

但领域外有好有坏。平均涨了 0.8 个点，可 MATH-500 掉了 4.9 个点。

换到图像上，看表 4。E2B 只用没有标注的图片训练，VQAv2 准确率从 49.1 提到 56.0，涨了 6.9 个点。

附录里还在 Qwen3-4B-Base 上试了，这是没做过指令微调的基础模型。四项平均从 66.7 提到 72.7。

### 和以前方法的对比

更硬的对比在表 2，把 OPST 和用了人工数据的方法放在一起。其中 u-OPSD 用人工题目，R-Zero 和 OPST 一样完全不用人工数据，这两个基线都是作者自己复现的。

在不用人工参考解答的方法里，OPST 四个设置的平均分都是最好的，比 u-OPSD 高 1.0 到 2.8 个点，比 R-Zero 高 1.4 到 5.2 个点。

和完全有监督的 OPSD 比，E2B 不开思考时，OPST 拿回了它 88% 的提升。开了思考模式，两个模型上都和它持平，分别高 0.2 和 0.7 个点。

但也要看到，E4B 不开思考时，OPST 平均 52.2，有监督的 OPSD 是 54.3，用人工数据的 SFT 是 52.3，OPST 没有超过它们。

表 5 的消融很说明问题。两道过滤分别贡献 2.7 和 2.3 个点。去掉特权信息，MMLU 的提升从 10.2 缩到 3.1 个点，换成同一主题下另一道题的提示，只剩 2.7。所以起作用的，是和这道题对应的提示。

还有一个抗噪实验。关掉过滤，故意把一部分候选答案换成错的。错到 75% 的时候，OPST 只掉 1.9 个点，还比基础模型高 3.3，SFT 掉了 5.7 个点，比基础模型还低 4.4。

我的判断是，这篇最值得记住的不是分数，而是一个设计选择，自己生成的东西，不当答案，只当提示。抗噪实验说明，这样做能容忍相当多的错误答案。

我的判断是，也要注意，主题是按评测基准的类别来定的，比如直接用 MMLU 的学科名。所以这更像是有方向地补强，而不是模型完全自主地学新东西。

### 局限

作者自己写了几条局限。第一，OPST 只能放大模型已经有的知识。

第二，伪验证检查的是自洽，不是正确。如果模型每次都错得一样，这个错误照样会进入训练。

第三，主题描述是人工挑的，实验也只在小模型上做过。

另外，训练轮数多了不会一直涨。在 E2B 的 MMLU 上，第 7 轮最好，再往后没有新的提升，稳定在 46 左右。

### 未来方向

作者提到两个方向。一是不需要标签的停止准则，也就是不靠标注数据，就能判断训练什么时候该停。二是借助工具或者检索来做验证。

我的判断是，这篇把自我提升往前推了一步，从自己找答案，走到了连题目都自己出。如果在更大的模型上也成立，后训练对人工数据的依赖，可能会进一步降低。

我的判断是，它的天花板也很清楚。验证只靠自洽，上限由模型自己的知识决定。在小模型上效果明显，换到大模型还有多少空间，论文没有回答。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Google and Ulm University in Germany. It's called On-Policy Self-Training, or OPST, and the subtitle is data-free self-improvement in large language models.

The question is a bold one. Give a model no human-written problems and no answers, just the names of some subjects, like abstract algebra. Can it write its own exercises, teach itself, and get better?

### Problem

Some background first. After pretraining, a language model goes through post-training, and that's where a lot of its capability gains come from.

But almost every post-training method leans on outside supervision. Supervised fine-tuning needs reference solutions. Reinforcement learning with verifiable rewards needs gold answers. On-policy distillation needs a stronger teacher.

That supervision is expensive. The paper's example: process-level labels for a single math dataset took about eight hundred thousand human step annotations.

Some methods already let a model generate its own answers or rationales, but people still write the problems. As the authors put it, human curation still decides what the model can learn.

### Prior work

The direct foundation here is on-policy self-distillation, or OPSD.

Start with on-policy distillation. The student answers on its own, and at every position of its output, a teacher supplies a distribution over the next token, which the student learns to match. The signal is dense, token by token, and it lands exactly where the student actually goes.

Self-distillation drops the stronger teacher. Teacher and student are the same model, except the teacher also sees privileged information the student doesn't, usually the reference solution. This figure shows that setup.

The catch is that existing OPSD methods still assume a curated problem set. One line of work replaced the reference solution with a majority vote over the model's own answers, but people still supply the problems.

### Key idea

The authors' idea has two steps. Step one: drop the problems too. Give the model only a coarse topic, and have it write a question, a rationale, and a candidate answer.

Step two is the important one. A self-generated answer may be wrong, so it isn't a target for the student to copy. It's a hint, and only the teacher sees it.

With that hint in view, the teacher writes out a full line of reasoning from its own knowledge, and the student learns from that. The authors argue this gives the student more than the generated answer alone contains.

The paper's first figure is this loop. Generate data from topics, filter it, train, then generate again, round after round.

### Method

There are three parts. Part one is writing problems. Topics are existing category labels: a subject like college mathematics for MMLU, or a problem type like rate, time and distance for GSM8K.

Part two is filtering, with two gates. The first is pseudo-verification. In a fresh context, without its own solution, the model solves the question again and must get the same answer. Then it rewords the question and solves it once more, and the answer still has to match, because an answer that changes with the wording usually means an ambiguous question.

The second gate is the consensus gate. The student answers the question several times. If every attempt agrees with the candidate answer, the student already knows it, and the problem is dropped.

The two gates pull in opposite directions. One keeps problems stable enough to trust, the other removes the ones that are too easy, so what's left sits at the edge of what the student can do, and its difficulty shifts as the student improves.

Part three is training. The student sees only the question and writes its own answer. The teacher sees the question plus the privileged information, and at every position of the student's answer it gives a next-token distribution for the student to match.

The teacher isn't frozen either. It starts as a copy of the student and, after every step, follows the student's weights through an exponential moving average, an EMA, so one noisy batch can't knock it off course. And the EMA teacher is the model they evaluate.

The loop repeats over several rounds, each with a fresh set of problems, minus any that duplicate earlier rounds.

The method also extends to images, as Visual OPST. It needs no labels, just unlabeled images and visual topics. The model poses questions about each image, and the privileged information is its own detailed caption, or spatial relations derived from the objects it detects.

### Results

The experiments use two small models, Gemma 4 E2B and E4B, with topics covering the domains of four benchmarks: MMLU, GPQA, GSM8K and HellaSwag. Three more benchmarks that no topic targets check whether the model forgets other skills.

The cleanest comparison is Table 1. Same self-generated data, used three ways. SFT imitates the generated answer, GRPO rewards agreeing with it, and OPST hands it to the teacher as context.

On E2B without thinking mode, MMLU goes from thirty-seven point five to forty-seven point seven, up ten point two points. On average, OPST beats SFT on the same data by five point six points, and by one point three on E4B. With thinking on, the margins are two point seven and three point eight.

The only difference between the two is whether the generated answer is something to imitate or a hint for the teacher. SFT comes out ahead in just one spot, HellaSwag on E4B without thinking, by zero point four.

The multi-domain test is Table 3. Mixing the topics of all three in-domain benchmarks raises the in-domain average by seven point six points, within one point of training on each domain separately.

Out of domain, it's mixed. The average rises by zero point eight, but MATH-500 drops four point nine points.

On images, Table 4. Trained on unlabeled images only, E2B's VQAv2 accuracy climbs from forty-nine point one to fifty-six, up six point nine.

The appendix also tries Qwen3-4B-Base, a pretrained model with no instruction tuning. Its four-benchmark average goes from sixty-six point seven to seventy-two point seven.

### Comparison

The tougher comparison is Table 2, which sets OPST beside methods that use human data. u-OPSD uses human-written questions, while R-Zero, like OPST, uses no human data at all. Both baselines are the authors' own re-implementations.

Among methods without reference solutions, OPST has the best average in all four settings, beating u-OPSD by one to two point eight points and R-Zero by one point four to five point two.

Against fully supervised OPSD, on E2B without thinking, OPST recovers eighty-eight percent of its gain. With thinking, it matches it at both sizes, coming out zero point two and zero point seven points ahead.

But on E4B without thinking, OPST averages fifty-two point two, while supervised OPSD gets fifty-four point three and supervised SFT on human data gets fifty-two point three. There, OPST doesn't beat them.

The ablations in Table 5 are telling. The two filters add two point seven and two point three points. Remove the privileged information and the MMLU gain shrinks from ten point two to three point one, and a hint from a different problem on the same topic leaves just two point seven. So the gain comes from a hint tied to the specific problem.

There's also a noise test. With the filters off, they replace a share of the candidate answers with wrong ones. At seventy-five percent wrong, OPST loses only one point nine points and stays three point three above the base model, while SFT loses five point seven and ends up four point four below it.

My read is that the thing to remember here isn't a score but a design choice: treat what the model generates as a hint, not an answer. The noise test suggests that choice tolerates a lot of wrong answers.

My read is also that the topics follow the benchmarks' own categories, MMLU's subject names for example. So this looks more like targeted strengthening than a model independently learning something new.

### Limitations

The authors list several limitations. First, OPST can only amplify knowledge the model already has.

Second, pseudo-verification checks self-consistency, not correctness. If the model is wrong the same way every time, that error still goes into training.

Third, the topic descriptors are hand-chosen, and the experiments only cover small models.

Also, more rounds don't keep helping. On E2B's MMLU, accuracy peaks at round seven, then stops improving and holds around forty-six.

### Future work

The authors point to two directions. One is a label-free stopping criterion, a way to know when to stop training without labeled data. The other is verification grounded in tools or retrieval.

My read is that this pushes self-improvement a step further, from finding its own answers to writing its own problems too. If it holds at larger scale, post-training could depend even less on human data.

My read is that the ceiling is also clear. Verification rests on self-consistency, so the model's own knowledge sets the limit. The gains are clear on small models. How much room there is on large ones, the paper doesn't say.

That's today's Daily Frontier Papers. The paper is linked in the description.

