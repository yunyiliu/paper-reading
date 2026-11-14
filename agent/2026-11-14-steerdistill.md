# SteerDistill: Quantizing LLM Agents via Closed-Loop Teacher Steering

**模型压到4位Agent几乎不掉分**

- 论文 Paper: [arXiv 2611.13410](https://arxiv.org/abs/2611.13410)
- 单位 Affiliation: 清华大学智能产业研究院 · 中国科学技术大学
- 方向 Direction: Agent
- 讲解日期 Covered: 2026-11-14
- 视频 Videos: [YouTube 中文](https://youtu.be/_1egnpvfJf8) · [YouTube English](https://youtu.be/JhQRaIlljFU) · [B站](https://www.bilibili.com/video/BV1qwsG6oEan)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇清华大学智能产业研究院的论文，合作单位还有中国科学技术大学，名字叫 SteerDistill。

它要解决的问题很实际，把 Agent 背后的模型压成低比特，省下推理成本，同时尽量不让它变笨。

### 问题

先说背景。Agent 要多轮调用工具，和环境来回交互，模型会被调用很多次，所以推理成本是部署时绕不开的问题。常用的办法是低比特量化，简单说，就是把模型权重从 16 位压到 4 位，甚至 2 位。

可问题是，量化后的 Agent，任务成功率经常比原版低。作者翻看了失败的轨迹，发现量化模型常常忽略任务要求或者环境条件，它对环境造成的改变，和原版成功时不一样。作者把这叫做行为分歧，behavioral divergence。

举个例子。读文件的工具只能读当前目录，模型却在进入目录之前就去读文件。还有的任务要求建好文件再写内容，模型建了文件，却有几个留成了空的。

### 以前的方法

以前怎么补救。一类是离线训练和蒸馏，用事先收集好的数据去训，但这些数据和学生自己的交互无关，学生真正出错的那些局面，可能根本没覆盖到。

另一类是 on-policy，让学生自己跑，在自己的轨迹上学。覆盖的问题解决了，可在出错的那些局面里，学生自己很少能给出正确的回答，也就没什么可学的。

论文在 BFCL V4 的多轮任务上做了个诊断。在出现分歧的那些轮次，让学生不带任何提示重试四次，只有 9.6% 的轮次至少做对一次。

### 核心想法

SteerDistill 的思路是，全精度的老师不直接替学生答题，而是看出学生漏掉了什么，给它一条有针对性的提示，让学生自己改。

学生改完马上执行，看结果对不对，不对的话，老师根据执行反馈再改提示，再试一次。作者把这叫闭环老师引导，closed-loop teacher steering。

在同一个诊断里，有了老师引导，四次以内做对的轮次比例，从 9.6% 升到了 82.3%。最后再通过蒸馏，把这些纠正内化到学生身上，推理时不需要老师，也不需要提示。

这里的老师和学生要说清楚。老师就是同一个模型的全精度版本，学生是从同一个检查点出发的量化版本。所以这里要做的，是把量化丢掉的能力找回来。

### 方法

方法分两步，这张图就是一个训练周期。第一步叫老师引导的轨迹修订。对每条失败的学生轨迹，先找一条老师在同一个任务、同一个初始环境下成功的参考轨迹。

然后按环境状态的变化，把两条轨迹对齐，只看环境状态，动作不同但效果一样，就算等价。找到第一个行为分歧的轮次，把对话历史和环境状态都恢复到这一轮之前。

老师对比失败的回答和成功的参考，写一段引导提示，讲清楚哪里不一样，这条规则在什么条件下适用，应该怎么改。学生带着这段提示重新回答，并执行动作。

接着做局部验证，比较执行后的环境状态和返回信息，是不是和参考一致。不一致，老师就拿着这次尝试的反馈改提示，学生从同一个出错前的状态再试，每个分歧点最多试四次。

验证通过，就保留这次纠正，学生再不带提示继续往下跑。如果任务还是失败，就再找下一个分歧点。哪怕最后任务没完成，已经验证过的纠正也会留下来用于训练。

第二步叫内化。训练时，学生完全看不到引导提示。在被纠正的那些轮次，学生去对齐带着提示的自己，也就是冻结的学生快照看到提示之后的预测，用的是前向 KL。

在其他轮次，学生去对齐全精度老师的预测，用的是反向 KL。KL 散度可以理解成两个概率分布差了多远，前向和反向是两种不同的衡量方向。

这样训完，学生在没有老师、也没有提示的情况下，就能做出那些被纠正过的决定。

### 实验和效果

实验设置。五个 Agent 基准，BFCL V4、τ2-Bench、WebShop、ALFWorld，还有一组搜索问答。三个模型，Qwen3-8B、Qwen3.5-4B 和 Gemma 4 12B-it。

每个模型先用 GRPO 做任务相关的后训练，得到全精度检查点，它既是参考，也是老师，所有量化方法都从它出发。对比的五个基线是 GPTQ、EfficientQAT、BitDistiller、按 NVIDIA NeMo-RL 流程实现的 On-Policy QAD，还有强化学习方法 QeRL。

主结果在表 1。主要设置是 W4A16，也就是权重 4 位、激活 16 位。在 Qwen3-8B 上，SteerDistill 五个基准都是量化方法里最好的，平均分 53.18，比最强的基线 QeRL 高 2.18 分，离全精度的 53.54 只差 0.36 分。

在另外两个模型上，它比 QeRL 分别高 1.71 分和 1.67 分。三个模型的平均分，都在全精度版本的一分以内。

精度再低一些，优势还在。Qwen3-8B 在 W4A8 下比 QeRL 高 1.93 分，在权重只有 2 位的 W2A16 下，比最强的基线 On-Policy QAD 高 3.39 分。唯一的例外是 W2A16 的搜索问答，BitDistiller 是 34.22，略高于它的 34.05。

训练成本看图 3。在 τ2-Bench 上，SteerDistill 训练到 32 个 GPU 小时，成功率就到了 31.68%，比 On-Policy QAD 收敛后的分数高大约 3 个百分点。作者称，它比训练了 96 个 GPU 小时的 QeRL 高 1.84 个百分点，GPU 时间少用 67%，这个时间已经算上了老师引导和验证的开销。

推理时也更省。成功完成一个任务，它平均输出大约 4000 个 token，全精度模型是 3900，量化基线在 4100 到 5200 之间。

论文还给了案例。比如订火车票，用户没说从哪到哪，全精度模型和 SteerDistill 都会先问路线，On-Policy QAD 自己编了一条波士顿到纽约的路线，EfficientQAT 没拿到路线，就说票已经订好了。

### 和以前方法的对比

消融实验在表 2，都是在 BFCL V4 上，用 Qwen3-8B 的 W4A16 做的。先看在哪里干预。按环境状态找分歧点，比按第一个动作不一致来找，高 1.83 分。随机挑一轮干预，只有 43.52，比 On-Policy QAD 的 44.43 还低。

再看怎么干预。老师引导学生自己改，比学生不带提示重试高 2.38 分，比老师直接替学生回答高 1.78 分。

提示的写法也有影响。结构化的提示比自由格式高 2.06 分。提示由全精度老师来写，比学生自己写高 5.10 分，比用 GPT-5.6-Sol 来写还高 1.57 分。

去掉局部验证，分数从 46.84 掉到 42.18。反过来，如果在局部验证之外，再要求整条任务成功才保留，分数是 46.30，反而略低一点。

KL 的方向也有讲究。纠正过的轮次用前向，其他轮次用反向，比全程反向高 1.55 分，比全程前向高 2.26 分，两个方向反过来用，分数掉 2.56 分。

我的判断是，这篇最有价值的地方，是把老师从替学生答题，变成指出学生漏了什么，再用环境去验证纠正是不是真的有效。也要注意，On-Policy QAD 和 QeRL 的参考配置用的是 NVFP4，作者把它们改成了整数量化在 A100 上跑，和原版设置不完全一样。

### 局限

作者自己写的局限，可以归成三点。第一，这个方法需要老师有成功的参考轨迹。没有成功参考的失败轨迹，只能原样留下，拿不到任何纠正。

第二，它要求能把对话历史和环境状态恢复到出错之前，还要有可靠的局部验证。在不能恢复状态、或者没法可靠验证的环境里，能不能用，还没有测试。

第三，局部验证只检查这一步在当前局面里的效果，不保证任务最终成功，也不保证回答里的每一步推理都对。

我的判断是，还有两点值得注意。消融只在一个基准、一个模型上做。而且在 2 位权重下，它虽然领先基线，平均分 42.97，离全精度的 53.54 还差十多分，极低比特的问题并没有解决。

### 未来方向

论文没有单独写未来方向。从局限能看出，下一步要验证的是，在状态没法恢复、结果不好自动验证的真实环境里，这套方法还能不能用。

还有一个有意思的现象。附录里，训练后的学生在推理时遇到执行错误，会自己先建文件再重写，或者先登录再发消息，作者说这超出了训练时直接针对的那些纠正。

我的判断是，这套先找分歧、再引导、再验证的思路，不只适用于量化。只要有一个更可靠的参考模型，和一个能回放、能检查状态的环境，就可以拿来修补小模型或者压缩模型的 Agent 行为。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from the Institute for AI Industry Research at Tsinghua University, together with the University of Science and Technology of China. It's called SteerDistill.

The problem is a practical one. You want to squeeze the model behind an agent down to low precision to cut inference cost, without making the agent noticeably worse at its job.

### Problem

Some context first. Agents call tools over many turns, so the model gets called again and again, and inference cost becomes a real deployment concern. A standard fix is low-bit quantization, which means storing the weights in four bits, or even two, instead of sixteen.

The catch is that quantized agents often succeed less often than the full-precision originals. Looking at failed runs, the authors find the quantized model tends to overlook a task requirement or a condition in the environment, so it changes the environment differently from a successful full-precision run. They call this behavioral divergence.

Here's an example. The file-reading tool only reads the current directory, but the agent tries to read the file before entering that directory. In another task, the agent creates the requested files but leaves some of them empty.

### Prior work

So how do people repair this today? One option is offline training and distillation on data collected in advance. But that data was gathered independently of the student's own behavior, so it may never cover the situations where the student actually goes wrong.

The other option is on-policy training, where the student learns from its own rollouts. That fixes coverage, but in exactly those failure situations, the student rarely produces a correct answer, so there's little to learn from.

The paper puts a number on this with a diagnostic on BFCL V4 multi-turn tasks. At the turns where the student diverged, retrying four times with no help produced a correct action at only nine point six percent of them.

### Key idea

SteerDistill's idea is that the full-precision teacher shouldn't answer for the student. Instead, it should spot what the student missed and give it a targeted hint, so the student fixes the mistake itself.

The student then executes its revised action and the result is checked. If it's still wrong, the teacher uses that execution feedback to sharpen the hint, and the student tries again. The authors call this closed-loop teacher steering.

In the same diagnostic, steering raised the share of divergent turns solved within four attempts from nine point six to eighty-two point three percent. Those corrections are then distilled into the student, so at inference it needs neither the teacher nor the hints.

One thing to be clear about. The teacher is simply the full-precision version of the same model, and the student is the quantized copy, starting from the same checkpoint. So the goal here is to win back what quantization lost.

### Method

The method has two stages, and this figure shows one training cycle. Stage one is teacher-steered trajectory revision. For each failed student rollout, they look for a successful teacher run on the same task from the same starting environment.

They line up the two runs by how the environment state changes, and only by that. Different actions with the same effect count as equivalent. At the first behavioral divergence, they restore both the conversation history and the environment to just before that turn.

The teacher compares the failed response with the successful reference and writes a steering prompt. It says what went wrong, when the rule applies, and what to do instead. The student answers again with that prompt and executes the action.

Next comes local verification, which compares the resulting environment state and returned information with the reference. If they don't match, the teacher revises its hint using feedback from that attempt, and the student retries from the same pre-error state, with up to four attempts per divergence.

Once a fix passes, it's kept, and the student carries on without any hint. If the task still fails, they look for the next divergence. Verified fixes are kept for training even when the whole task never succeeds.

Stage two is internalization. During training, the student never sees the steering prompts. At the corrected turns, it learns to match its own steered self, meaning a frozen snapshot of the student that does see the prompt, using forward KL.

At every other turn, it learns to match the full-precision teacher, using reverse KL. KL divergence is just a measure of how far apart two probability distributions are, and forward and reverse are the two directions you can measure it in.

The result is a student that makes the corrected decisions on its own, with no teacher and no hints.

### Results

The setup. Five agent benchmarks: BFCL V4, tau-two Bench, WebShop, ALFWorld, and a set of search question-answering tasks. Three models: Qwen3 8B, Qwen3.5 4B, and Gemma 4 12B instruct.

Each model is first post-trained on the tasks with GRPO. That full-precision checkpoint is the reference and the teacher, and every quantized method starts from it. The five baselines are GPTQ, EfficientQAT, BitDistiller, an On-Policy QAD following NVIDIA's NeMo-RL workflow, and QeRL, a reinforcement learning method.

The main results are in Table 1. The headline setting is W4A16, four-bit weights with sixteen-bit activations. On Qwen3 8B, SteerDistill is the best quantized method on all five benchmarks. Its average is fifty-three point one eight, two point one eight above the strongest baseline, QeRL, and just zero point three six below full precision at fifty-three point five four.

On the other two models, it beats QeRL by one point seven one and one point six seven. On all three models, its average lands within one point of the full-precision version.

At lower precision the lead holds. On Qwen3 8B it beats QeRL by one point nine three under W4A8, and under W2A16, with two-bit weights, it beats the strongest baseline, On-Policy QAD, by three point three nine. The one exception is search QA under W2A16, where BitDistiller edges it out, thirty-four point two two to thirty-four point zero five.

Training cost is in Figure 3. On tau-two Bench, SteerDistill reaches thirty-one point six eight percent after thirty-two GPU hours, about three points above On-Policy QAD's converged score. The authors report it beats QeRL trained for ninety-six GPU hours by one point eight four points with sixty-seven percent fewer GPU hours, and that count includes the steering and verification work.

It's leaner at inference too. A successful task takes about four thousand output tokens on average, against three point nine thousand at full precision and four point one to five point two thousand for the quantized baselines.

The paper also shows cases. Take booking a train ticket when the user hasn't said where from or where to. The full-precision model and SteerDistill both ask for the route. On-Policy QAD invents one, Boston to New York, and EfficientQAT claims the ticket is booked without ever getting a route.

### Comparison

The ablations are in Table 2, all on BFCL V4 with Qwen3 8B at W4A16. First, where to intervene. Finding the divergence by environment state beats finding the first mismatched action by one point eight three. Intervening at a random turn scores forty-three point five two, below On-Policy QAD's forty-four point four three.

Second, how to intervene. Steering the student beats an unguided retry by two point three eight, and beats letting the teacher simply take over the turn by one point seven eight.

Third, the hint itself. A structured prompt beats a free-form one by two point zero six. Having the full-precision teacher write it beats the student writing its own by five point one zero, and beats GPT-5.6-Sol writing it by one point five seven.

Remove local verification, and the score falls from forty-six point eight four to forty-two point one eight. Going the other way, also requiring the whole task to succeed before keeping a fix gives forty-six point three zero, slightly lower.

The KL direction matters as well. Forward at corrected turns and reverse elsewhere beats reverse everywhere by one point five five and forward everywhere by two point two six. Swapping the two costs two point five six.

My read is that the valuable move here is turning the teacher from someone who answers into someone who points out what was missed, and then letting the environment confirm the fix actually works. One caveat: the reference setups for On-Policy QAD and QeRL use NVFP4, and the authors adapted both to integer quantization on A100s, so those baselines don't run exactly as their originals do.

### Limitations

The authors' limitations come down to three points. First, the method needs a successful teacher reference. A failed run without one is kept as it is and gets no corrections at all.

Second, it needs to restore the history and the environment state to just before the error, and it needs reliable local verification. Whether it works in environments without those is untested.

Third, local verification only checks the effect of one step in its current context. It doesn't guarantee the task eventually succeeds, or that every step of the reasoning is right.

My read is that two more things are worth noting. The ablations use one benchmark and one model. And at two-bit weights, while it leads the baselines, its average of forty-two point nine seven is still more than ten points below full precision at fifty-three point five four, so very low precision remains an open problem.

### Future work

The paper has no separate future-work section. But the limitations point to the next test: real environments where state can't be restored and outcomes are hard to check automatically.

There's also an interesting observation in the appendix. At inference, the trained student recovers from execution errors on its own, creating a missing file and retrying the write, or logging in before sending a message. The authors describe this as going beyond the corrections training explicitly targeted.

My read is that this find-the-divergence, hint, then verify recipe isn't really specific to quantization. Given a more reliable reference model and an environment you can replay and inspect, the same loop could patch the agent behavior of small or compressed models in general.

That's today's Daily Frontier Papers. The paper is linked in the description.

