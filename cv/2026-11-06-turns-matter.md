# Turns Matter: Advancing Streaming Video RL via Turn-Level Credit

**每一轮单独算功劳AI少说重复话**

- 论文 Paper: [arXiv 2611.06728](https://arxiv.org/abs/2611.06728)
- 单位 Affiliation: 复旦大学、京东等
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-06
- 视频 Videos: [YouTube 中文](https://youtu.be/jYf-iheAhwU) · [YouTube English](https://youtu.be/uZobW1OsgZE) · [B站](https://www.bilibili.com/video/BV1Ao4z6QEJq)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇复旦大学和京东等单位的论文，题目叫 Turns Matter，意思是每一轮都很重要，方法名叫 TurnWise。

它研究的是流式视频理解里的强化学习，要回答的问题是，一段多轮对话拿到一个总分之后，功劳和过错应该怎么分到每一轮头上。

### 问题

先说什么是流式视频理解。普通的视频问答，是把整段视频看完再回答。流式不一样，视频一边在播，模型一边在看，每到一个决策时间点，它都要决定，现在开口，还是继续看下去。

好的助手，有新证据就回答，需要时补充，没有新信息就保持安静。所以该不该说，不光看画面，还要看自己之前说过什么。

论文第一张图举了个例子。用户问，这道菜最后做出来是什么样。模型第一次回答，烤鸡配米饭，第二次补了一个细节，鸡肉上有绿色的香草碎，后面四次，就是换着说法重复，最后两次一字不差。作者注明，这段对话是示意的。

问题出在训练信号上。按原来的 GRPO，这六次回答拿到的是同一个值，都是正的 0.984，重复的几句也一起被鼓励。按 TurnWise 的分法，第一句还是 0.984，补充细节是小小的正值，后面四次重复全是负值。图注说明，这些值是训练中实际记录的 TurnWise 优势，和对应的 GRPO 对照值。

### 以前的方法

现在给这类模型做强化学习，常用的是 GRPO。对同一个样本采样一组对话，每条打一个分，再和组内平均分比较，高于平均就鼓励，低于平均就压制，这个相对的分数叫优势。

之前的工作，比如 MMDuet2、ThinkStream 和 StreamPro，主要在奖励设计上下功夫，考虑回答质量、回答时机、格式，StreamPro 还加入了对单条回答的评分。

但作者指出，这些方法最后还是把整条对话的总分，变成一个优势值，原样分给每一轮。换句话说，打分可以细到每一轮，学习信号却是整条对话共用一个。

这就带来两种错位。一条好的对话里，可能有糟糕的一轮，前面答得好，后面的重复也跟着被奖励。一条差的对话里，也可能有好的一轮，前面没答好，后面有用的补充反而被惩罚。

还有一个更隐蔽的问题。损失通常按 token 平均，长回答 token 多，不说话只输出一个很短的等待标记，于是长回答在训练里的分量天然更大。

### 核心想法

作者的主张是，每一轮决策的学习信号，应该看从这一轮开始往后还能挣多少分，而不是把之前已经挣到的分也算进来。这在强化学习里叫 return-to-go，也就是剩余回报。

所以 TurnWise 做了两件事。第一，给每一轮单独算优势。第二，让每一轮在损失里的权重一样，不管这一轮说了多少字。两件事都以单轮决策为单位，作者称，既不需要额外训练一个价值模型，也不需要额外采样。

### 方法

先看整体流程，这张图从左往右读。输入是视频流、用户问题、对话历史，还有一个观察计划，规定在哪些时间点做决策。模型对同一个样本采样一组对话，每条对话得到一个总分。

总分沿用 MMDuet2 的设计。主体是一个叫 PAUC 的指标，衡量回答在标注的时间窗口里答得好不好、及不及时，同样的质量，答得越早贡献越大。另外有三项扣分，内容重复、在窗口外说话，以及共享前缀过长。

关键的一步叫静默补全的前缀打分。拿一条对话，保留前几轮模型真实的动作，把后面所有轮都换成不说话，再用同一个打分器打分。这个分数的意思是，如果模型从这里开始一直沉默，前面这些话一共能挣多少分。

用整条对话的总分，减去某一轮之前那段前缀的分数，就是从这一轮开始的剩余回报。然后在同一组对话的同一轮之间比较，减去组内平均，再按和 GRPO 一样的尺度归一化，就得到这一轮自己的优势。

换个角度看，就是从原来 GRPO 的优势里，扣掉这一轮之前已经领先或落后组内平均的那部分。所以总分低于平均的对话，后面有用的一轮也能拿正分，反过来也一样。

在流式场景里，不说话本身就是一个明确的动作，所以沉默是个自然的参照。作者称，这些前缀分数复用已经算好的评分，不需要新的采样，也不用再调用打分模型。

训练时还有一个系数，在前百分之十的更新步数里，从共享的优势线性过渡到完全的单轮优势，之后保持不变。

第二个设计是按轮平均的损失。先在每一轮内部，把 token 的损失取平均，再在所有轮之间取平均。这样不管是一大段回答，还是一个等待标记，每一轮的权重都一样。

### 实验和效果

实验设置。底座是 Qwen2.5-VL-3B 和 Qwen3-VL-4B 两个小模型，先在 100874 条指令数据上做一轮监督微调，再在 3983 个流式样本上做强化学习，每个样本采样 4 条对话，训练奖励由 gpt-oss-20b 来判。

评测用三个基准。ProactiveVQA 看主动回答的质量和时机，OVO-Timing 看事件发生时能不能按时回应，StreamingBench 看实时画面理解和主动输出。对比的基线是 MMDuet2、ThinkStream 和 StreamPro，作者在同一套 GRPO 框架里实现。

结果在表 1。两个底座上，TurnWise 在三个基准的总分都是第一。

3B 模型上，ProactiveVQA 是 43.56，比其他方法里最好的成绩高 3.96，OVO-Timing 平均 21.59，高 3.87，StreamingBench 平均 49.43，只高 0.69。这里的其他方法，也包括只做了监督微调的模型。

4B 模型上，三个分数是 42.44、11.15 和 58.96，分别高 6.26、1.92 和 2.38。StreamingBench 的主动输出准确率，从其他方法里最好的 35.6 提到 38.4。

消融在表 2，看两个设计各起什么作用。只加单轮优势，重复率是 15.53，只加按轮平均，OVO-Timing 只有 8.17。两个一起用，三项任务分数都是最高，重复率降到 5.98，比只用单轮优势低了 61.5%。

表 3 测了修正系数。在完整方法里，系数取 1，也就是完全修正时，四项指标都是最好的。

训练曲线也能看出行为变化。和只用单轮优势相比，加上按轮平均后，最后 50 步的平均奖励从 0.657 升到 0.711，开口说话的比例从 62.2% 降到 37%。论文的说法是，奖励更高，说话也更有选择性。

最后，作者拿真实训练数据做了分析。在 11116 条训练对话、84184 个决策里，有 21.2% 的决策，优势的正负号和原来的 GRPO 相反。原来拿正值的 722 条重复回答里，有 432 条被改成负值，占 59.83%。

### 和以前方法的对比

和 MMDuet2、ThinkStream、StreamPro 比，TurnWise 没有改奖励函数，打分器是固定的。它改的是，同一个总分怎么分到每一轮，以及每一轮在损失里占多大权重。

和需要价值模型的做法比，它不用额外训练评估网络，而是把沉默当参照，用现成的打分器来算。

分项上也不是全胜。4B 模型上，OVO-Timing 的 CRR 子项，MMDuet2 是 11.88，TurnWise 是 8.68，ProactiveVQA 的 TV 子项，ThinkStream 是 38.49，也比 TurnWise 的 37.73 略高。

我的判断是，这篇的价值在于把问题拆得很干净，奖励不动，只改信号怎么分，而且按作者的说法不需要额外采样，也不用多调用打分模型，做多轮强化学习的团队很容易拿去试。

我的判断是，提升幅度要放回整体分数里看。3B 模型在 StreamingBench 上只高 0.69，OVO-Timing 的平均分也只有 21.59 和 11.15，离真正好用的流式助手还有不小的距离。

### 局限

论文没有单独写局限这一节，下面几点，是从论文自己的数据里能看到的。

第一，实验规模不大。只有 3B 和 4B 两个小模型，强化学习用了不到 4000 个样本，论文里也没有报告多次运行的误差范围。

第二，基线是作者在同一个框架里实现的，条件一致，但不能直接和原论文报告的数字对照。

第三，单用其中一个设计时，效果并不稳。只用单轮优势，重复率从 MMDuet2 的 6.08 升到 15.53，完整方法里系数取 0.5 时，重复率甚至到了 36.04。在论文的实验里，两个设计一起用、系数取 1，效果才最好。

第四，方法依赖打分器。训练奖励由 gpt-oss-20b 判断，ProactiveVQA 的评测由 GPT-5.6-luna 打分，所以评测分数里，也包含了判分模型的判断。

### 未来方向

论文首页给出了代码、模型和数据集的链接，正文没有单独讨论未来方向。

我的判断是，最值得看的，是这个思路能不能走出流式视频。只要多轮任务里有明确的不行动选项，比如 Agent 决定要不要打断用户，原理上都能用沉默补全来分功劳，但这还要实验验证。

我的判断是，另一个方向是更长的视频。现在的训练片段只有 20 秒，最多 90 帧，到了几分钟甚至几小时的直播里，前面积累的分数更多，单轮信号会不会更关键，值得继续看。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Fudan University, JD.com and collaborators. It's called Turns Matter, and the method is TurnWise.

The setting is reinforcement learning for streaming video models. The question is simple: when a whole multi-turn conversation gets one score, how much of the credit, or the blame, belongs to each turn?

### Problem

First, what streaming video understanding means. In ordinary video question answering, the model watches the whole clip, then answers. In streaming, the video is still playing. At every decision point, the model has to choose: speak now, or keep watching.

A useful assistant answers when new evidence shows up, adds a follow-up when it's needed, and stays quiet when another reply would add nothing. So whether to speak depends not just on the picture, but on what the model has already said.

The first figure gives an example. The user asks what the finished dish looks like. The first reply says browned chicken served over rice. The second adds a detail, green herb flecks on the chicken. The next four just repeat the first answer, and the last two are word for word. The authors note this dialogue is illustrative.

The problem is the training signal. Under standard GRPO, all six replies get the same value, plus zero point nine eight four, so the repeats are reinforced too. Under TurnWise, the first reply keeps that value, the added detail gets a small positive one, and all four repeats go negative. Per the caption, these are recorded TurnWise advantages next to the matching GRPO counterfactual.

### Prior work

The usual recipe for RL here is GRPO. You sample a group of conversations for the same input, score each one, and compare it with the group average. Above average gets pushed up, below average gets pushed down. That relative score is called the advantage.

Earlier work, like MMDuet2, ThinkStream and StreamPro, mostly focused on reward design: answer quality, timing, format. StreamPro also adds scores for individual responses.

But the authors point out that these methods still turn the conversation's total score into one advantage and broadcast it to every turn. In other words, the scoring can be per turn, but the learning signal is shared by the whole conversation.

That creates two kinds of mismatch. A good conversation can contain a bad turn: after a strong start, a redundant reply still gets rewarded. A bad conversation can contain a good turn: after an early miss, a useful follow-up still gets penalized.

There's a second, quieter problem. The loss is usually averaged over tokens. A long answer has many tokens, while choosing to wait produces one short marker. So long answers automatically carry more weight in training.

### Key idea

The authors' claim is that each decision should be judged by what's earned from that turn onward, not by rewards collected before it. In RL terms, that's the return-to-go, the remaining return.

So TurnWise does two things. First, it computes a separate advantage for each turn. Second, it gives every turn equal weight in the loss, no matter how many words it uses. Both treat the single decision as the unit, and the authors say this needs no learned critic and no extra rollouts.

### Method

Here's the overall pipeline, read left to right. The input is the video stream, the user's question, the dialogue history, and an observation plan that fixes when decisions happen. The model samples a group of conversations, and each gets a total score.

That score follows MMDuet2. Its core is a metric called PAUC, which measures how good and how timely the answers are inside annotated reply windows. For the same quality, answering earlier counts for more. Then there are three penalties: repeated content, speaking outside the window, and overly long shared prefixes.

The key step is called silence-completed prefix scoring. Take a conversation, keep the model's real actions up to some turn, replace every later turn with silence, and score it with the same scorer. That number tells you what the earlier replies are worth if the model says nothing more.

Subtract the score of the prefix before a given turn from the full score, and you get the remaining return from that turn on. Then compare across the group at the same turn, subtract the group mean, and normalize on the same scale GRPO uses. That's the turn's own advantage.

There's an equivalent way to write it. Take the usual GRPO advantage and subtract the above or below average reward already earned before this turn. So a conversation that scored above average can still get a negative value at a later turn, and one below average can still earn a positive value for a useful later turn.

This works because, in streaming, staying silent is an explicit action, so silence is a natural reference. And the prefix scores reuse evaluations that were already computed. The authors say no new rollouts and no extra judge calls are needed.

In training, they don't switch to full per-turn credit right away. A coefficient ramps linearly from the shared advantage to full per-turn credit over the first ten percent of updates, and then stays at full correction.

The second piece is turn-mean loss. First average the token losses inside each turn, then average across turns. A long answer and a short wait marker end up with exactly the same weight.

### Results

The setup. The base models are two small ones, Qwen2.5-VL-3B and Qwen3-VL-4B. They get one epoch of supervised fine-tuning on about a hundred thousand instruction examples, then RL on three thousand nine hundred eighty-three streaming instances, with four conversations sampled per instance. The training reward is judged by gpt-oss-20b.

There are three benchmarks. ProactiveVQA measures answer quality and timing, OVO-Timing checks whether the model responds on time when events happen, and StreamingBench covers real-time visual understanding and proactive output. The baselines, MMDuet2, ThinkStream and StreamPro, were implemented by the authors in the same GRPO framework.

The results are in Table 1. On both base models, TurnWise has the best aggregate score on all three benchmarks.

On the 3B model, ProactiveVQA reaches forty-three point five six, three point nine six above the best of the other methods. OVO-Timing averages twenty-one point five nine, up three point eight seven. StreamingBench averages forty-nine point four three, up only zero point six nine. Here, the other methods include the fine-tuned model before RL.

On the 4B model, the three scores are forty-two point four four, eleven point one five, and fifty-eight point nine six, gains of six point two six, one point nine two, and two point three eight. Proactive output accuracy on StreamingBench rises from thirty-five point six, the best of the other methods, to thirty-eight point four.

Table 2 is the ablation. With per-turn credit alone, the repetition rate is fifteen point five three. With turn-mean loss alone, OVO-Timing is only eight point one seven. Together, all three task scores are the highest, and repetition drops to five point nine eight, sixty-one and a half percent lower than credit alone.

Table 3 sweeps the correction coefficient. In the full method, a coefficient of one, meaning full correction, is best on all four metrics.

The training curves show the behavior change. Compared with credit alone, adding turn-mean loss lifts the mean reward over the last fifty updates from zero point six five seven to zero point seven one one. The share of turns where the model speaks falls from sixty-two point two percent to thirty-seven percent, and the share of replies outside the window falls from zero point five three seven to zero point two eight nine. The paper's reading: higher reward, and more selective speaking.

Finally, the authors look at real training data. Across eleven thousand one hundred sixteen conversations and eighty-four thousand one hundred eighty-four decisions, twenty-one point two percent of decisions flip sign compared with GRPO. Of the seven hundred twenty-two repetitive replies that GRPO scored positive, four hundred thirty-two are now negative. That's fifty-nine point eight three percent.

### Comparison

Compared with MMDuet2, ThinkStream and StreamPro, TurnWise doesn't touch the reward. The scorer is fixed. What changes is how one total score is split across turns, and how much each turn weighs in the loss. The authors say this explicitly: a fixed scorer makes credit assignment and loss aggregation the variables under test.

Compared with methods that need a value model, it doesn't train a separate critic. It uses silence as the reference and the existing scorer to measure how much was earned before each turn.

It doesn't win every sub-score. On the 4B model, for the CRR part of OVO-Timing, MMDuet2 scores eleven point eight eight against TurnWise's eight point six eight. And on the TV category of ProactiveVQA, ThinkStream's thirty-eight point four nine edges out TurnWise's thirty-seven point seven three.

My read is that the value here is a clean separation. Keep the reward fixed and change only how the signal is split. By the authors' account it needs no extra rollouts or judge calls, so teams doing multi-turn RL can try it cheaply.

My read is also that the gains need context. On the 3B model, StreamingBench improves by only zero point six nine, and the OVO-Timing averages are just twenty-one point five nine and eleven point one five. That's still a long way from a streaming assistant you'd rely on.

### Limitations

The paper has no dedicated limitations section. These points come from its own numbers.

First, the scale is small. There are only two models, at 3B and 4B, RL uses under four thousand instances, and the paper reports no error bars across multiple runs.

Second, the baselines were implemented by the authors in one framework, not taken from the original papers. That keeps conditions equal, but these numbers can't be compared directly with the original papers' results.

Third, the pieces aren't stable on their own. With credit alone, repetition rises from MMDuet2's six point zero eight to fifteen point five three. In the full method with the coefficient at zero point five, repetition hits thirty-six point zero four. In the paper's experiments, it's both pieces together, at a coefficient of one, that work best.

Fourth, the method leans on its judges. The training reward is judged by gpt-oss-20b, and ProactiveVQA evaluation is scored by GPT-5.6-luna, so the reported scores also carry those judge models' calls.

### Future work

The first page links to code, a model and a dataset. The paper doesn't discuss future work separately.

My read is that the thing to watch is whether this idea leaves streaming video. Any multi-turn task with an explicit do-nothing option, say an agent deciding whether to interrupt the user, could in principle use silence completion to score each turn. That still needs experiments.

My read is that the other direction is longer video. Training clips here are only twenty seconds, at most ninety frames. In a stream that runs for minutes or hours, far more reward piles up early, and per-turn credit may matter even more. That's worth following.

That's today's Daily Frontier Papers. The paper is linked in the description.

