# Wave Forcing: Towards Speed-of-Light Streaming Video Generation

**让视频生成像流水线一样**

- 论文 Paper: [arXiv 2611.00859](https://arxiv.org/abs/2611.00859)
- 单位 Affiliation: 上海交通大学 DENG Lab
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-04
- 视频 Videos: [YouTube 中文](https://youtu.be/jZEIDHPD5RM) · [YouTube English](https://youtu.be/IC4HIAGiQco) · [B站](https://www.bilibili.com/video/BV1xAxQ6pE2C)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇上海交通大学 DENG Lab 牵头的论文，名字叫 Wave Forcing，副标题是，迈向光速的流式视频生成。

它要解决的问题是，怎么让 AI 一边生成视频，一边连续不断地往外输出，而且又快，画面又稳。

### 问题

先说背景。现在做实时长视频，主流办法是把视频切成一小段一小段，论文叫 chunk，一段接一段往后生成，每一段都参考前面已经生成好的内容。

每一段要经过几步去噪，从纯噪声慢慢变成清晰的画面。去噪完了，还要再跑一遍完整的模型，把这段干净的画面写进缓存，给后面的段落参考，论文把这一步叫 store。

按顺序执行的话，去噪和 store 都挡在两段输出之间。论文指出，哪怕只用一步去噪，算上 store，每出一段也要跑两次完整的前向计算。

另一个问题是质量。生成得越长，画面越容易变差，镜头还会往一个方向漂移。

### 以前的方法

以前的代表方法有几种。Self Forcing 让模型在训练时就用自己生成的历史，减少训练和推理之间的差异。Causal Forcing 也是一段一段地生成。

Rolling Forcing 更进一步，让几段处在不同噪声水平的画面一起去噪，后面更模糊的段落，也会反过来影响前面的段落。

至于多卡加速，常用的是上下文并行，比如 Ulysses 和 Ring，它们把一次前向计算拆到多张卡上，卡和卡之间要交换数据。

### 核心想法

这篇的核心想法，借用了一种叫波前调度的思路。你看这张图，横着是第几段，竖着是第几步去噪。

同一时刻，越早的段落在做越靠后的去噪步骤，越新的段落在做越靠前的步骤，这些同时进行的任务排成一条斜线，像波浪一样往前推。

每一步去噪，加上 store，各自占一张卡，放一份完整的模型。流水线灌满之后，每一轮并行计算就能出一段完整的视频，论文把这叫 1F1C。

要做到这一点，有个前提，新的段落不能等前面的段落彻底去噪完。所以作者规定，新的、更模糊的段落，可以看前面更清晰、但还没完成的段落，反过来不行，这就是因果注意力。

### 方法

方法分三块。第一块是注意力怎么看。每一段内部是完整的注意力，同时能看到开头一段作为锚点，一段有限长度的干净历史，store 刚写入的内容，以及前面比它更干净的段落。

这样就去掉了 Rolling Forcing 里，后面段落反过来影响前面段落的那条路。论文报告，如果不重新训练，直接这么改，短视频的质量和动态程度都会下降，所以要专门训练。

第二块是训练，一共三个阶段，底座是 Wan2.1 的 1.3B 文生视频模型。第一阶段，Rolling Forcing 和 Self Forcing 两种生成方式各用一半，做分布匹配蒸馏，也就是 DMD，让学生的输出分布去贴近老师。

第二阶段，拿老师模型生成的视频，把连续几段加上不同程度的噪声，让学生去还原，除了 DMD，还加了一个感知损失 LPIPS，权重 0.25。老师是 Wan2.1 的 14B 模型。作者说，这一步给了更强的动作和主体指导。

第三阶段，让学生用自己生成的历史继续训练，每次从 Rolling Forcing、Wave Forcing、Self Forcing 三种生成方式里随机选一种，各占三分之一，但最后部署的只有 Wave Forcing。

5 步去噪的版本，直接从 Rolling Forcing 公开的权重开始，跳过第一阶段。

第三块是执行系统，叫 Wave Runtime。难点在于，每张卡算注意力时，都要等负责 store 的卡，和更干净的段落所在的卡，把 K/V，也就是注意力用的键和值，送过来。

作者的做法是，每一层算完 Q、K、V 和位置编码，马上把 K/V 发出去，等到真正要算注意力时，才检查到没到。更干净的段落要读的 key 更少，往往跑在前面，所以它的 K/V 常常提前送到，通信就藏在了计算后面。

K/V 直接写进接收方注意力的缓冲区，不用再拼接。论文证明，在固定条件下，这种异步执行和同步执行，算出来的结果完全一样。

最后，把画面解码成视频的 VAE，放在单独的几张卡上，按实测耗时切成流水线，让解码跟得上生成。

### 实验和效果

先看质量，表 1。评测的分辨率是 832 乘 480，每秒 16 帧，长视频用 200 条 MovieGen 提示词，每条生成 100 秒。

5 步的 Wave Forcing，VBench 总分 81.64，Rolling Forcing 是 81.15，长视频质量分是 81.26 对 81.18，基本持平。

差别在动作。动态程度这一项，短视频从 51.39 提到 94.44，100 秒的长视频从 36.63 提到 91.45。

作者还自己设计了两个稳定性指标，在 100 条以主体为中心的提示词上算。主体保持，Wave Forcing 是 97.10，Rolling Forcing 是 92.75。镜头漂移越低越好，Wave Forcing 是 0.37，Rolling Forcing 是 0.17，这一项 Rolling Forcing 更好。

Causal Forcing 的长视频质量分 82.15 和动态程度 96.61 都是最高，但主体保持只有 71.11，镜头漂移是 11.14。论文的例子里，主体在第 30 秒和第 75 秒消失，画面里只剩枕头和床头板。

消融实验在表 2，用的是 4 步模型。去掉第二阶段，质量从 80.89 掉到 76.68，动态程度从 74.97 掉到 15.73。去掉 Self Forcing 这条分支，质量是 75.35，动态程度只有 23.66。

去掉 Rolling Forcing 分支，动态程度变高，但主体保持掉到 77.48，镜头漂移升到 6.24。去掉 Wave Forcing 分支，画面最稳，但动态程度只有 48.12。所以三条分支都留下了。

再看速度。4 步的 Wave Forcing，在一台 8 卡 H200 上，只算 DiT 部分每秒 147.6 帧，加上解码，端到端每秒 134.8 帧，是单卡基线 19.06 帧的 7.07 倍。

要注意，速度测试用的是随机初始化的模型权重，只有 VAE 是真实权重。14B 的规模，论文说拿不到对应的预训练权重，也没有足够算力去训练，所以只测了速度，最好的配置端到端每秒 33.5 帧，单卡是 5.51 帧。

系统消融里，从同步交换换成直接写进缓冲区，1.3B 模型每一轮耗时的中位数，从 91.96 毫秒降到 78.88 毫秒，少了 14.2%。

### 和以前方法的对比

跟上下文并行比，论文在 1.3B 到 14B 六个规模上，测了只算 DiT 部分的吞吐。BF16 精度下，Wave Parallelism 在六个规模里赢了五个。

1.3B 时它每秒 150.09 帧，Ulysses 是 80.60，Ring 是 41.19。但到了 9B，它的 45.70 帧比 Ulysses 的 46.41 低了 1.53%。换上 SageAttention 加 8 位量化，它在每个规模都是第一，是更快那个基线的 1.025 到 2.530 倍。作者也承认，模型越大，优势越小。

我的判断是，这篇的提速来自让更多的卡同时干活，而不是少干活。论文自己写明，它不减少计算量，也不降低第一段画面出来的延迟，8 张卡换来的是 7 倍左右的吞吐。

我的判断是，质量上它和 Rolling Forcing 基本打平，真正的进步是动作幅度大了很多，同时主体没丢，而镜头漂移这一项，还略逊于 Rolling Forcing。

### 局限

作者列的局限很具体。第一，VBench 分数不能完整衡量长视频，他们自己的稳定性指标只用了 100 条提示词，依赖跟踪和几何规则，和人的判断是否一致，还没有验证。

第二，每一步去噪都要一整份模型和一张卡，去噪步数一固定，单条流水线的规模就被限制住了。训练流程也是多阶段的，比较复杂。

第三，吞吐高不等于响应快。用户中途改提示词，控制镜头和动作，这些交互场景都还没有评测。

### 未来方向

作者计划先把解码这条链路分析清楚，再研究和上下文并行结合，简化训练，然后支持同时服务多路视频流，生成过程中改提示词，最后往可交互的世界模型方向走。

我的判断是，这篇最值得借鉴的，是把生成方式、训练和执行系统放在一起设计，模型的注意力结构，就是为了能并行才这样改的。

我的判断是，弱点也很明显，质量只在 1.3B 模型上验证，速度用的是随机权重，稳定性指标是自己定义的。能不能用到更大的模型和真实的交互场景，还要看后续工作。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is led by the DENG Lab at Shanghai Jiao Tong University. It's called Wave Forcing, and the subtitle is ambitious: towards speed-of-light streaming video generation.

The goal is a video model that streams frames out continuously as it generates them, fast, and without the picture falling apart.

### Problem

Some background. Real-time long video is usually made in chunks. The model generates a short chunk, then the next one, and each new chunk looks back at what has already been generated.

Each chunk goes through a few denoising steps, from pure noise to a clean picture. Then there's one more full pass through the model to write that clean chunk into the cache, so later chunks can refer to it. The paper calls that pass store.

Run everything in order and both the denoising and the store pass sit between one output chunk and the next. The paper points out that even with a single denoising step, every chunk still costs two full forward passes once you count store.

The other problem is quality. The longer the rollout, the more the picture tends to degrade, and the camera can drift off in one direction.

### Prior work

A few earlier methods matter here. Self Forcing trains the model on its own generated history, to close the gap between training and inference. Causal Forcing also generates chunk by chunk.

Rolling Forcing goes further. It denoises several chunks together at different noise levels, so later, noisier chunks also feed back into earlier ones.

For multi-GPU speed, the standard tool is context parallelism, like Ulysses and Ring. They split a single forward pass across GPUs, and the GPUs have to exchange data as they go.

### Key idea

The key idea borrows an old trick called wavefront scheduling. Look at this figure. One axis is the chunk, the other is the denoising step.

At any moment, older chunks are working on later denoising steps, and newer chunks on earlier ones. These simultaneous tasks line up along a diagonal, and that diagonal moves forward like a wave.

Each denoising step, plus store, gets its own GPU with a full copy of the model. Once the pipeline fills, every round of parallel compute finishes one complete chunk. The paper calls this one-F-one-C.

For that to work, a new chunk can't wait for the previous one to finish. So the authors set a rule: newer, noisier chunks may look at earlier, cleaner chunks that are still in progress, but never the other way round. That's causal attention.

### Method

The method has three parts. Part one is attention. Inside a chunk, attention is full. Across chunks, each one sees the first chunk as an anchor, a bounded window of clean history, whatever store just wrote, and the earlier chunks that are cleaner than itself.

That removes Rolling Forcing's backward path, where later chunks influence earlier ones. The paper reports that making this change without retraining hurts short-video quality and motion, so the model needs training for it.

Part two is training, in three stages, starting from Wan2.1, a 1.3B text-to-video model. Stage one samples Rolling Forcing or Self Forcing rollouts half the time each and applies distribution matching distillation, or DMD, which pulls the student's outputs toward the teacher's distribution.

Stage two takes videos generated by the teacher, adds noise to a run of consecutive chunks at staggered levels, and has the student recover them, with DMD plus a perceptual loss, LPIPS, at a weight of zero point two five. The teacher is the 14B Wan2.1 model. The authors say this stage gives stronger guidance on motion and on the subject.

Stage three trains the student on its own generated history. Each step picks one of three rollout styles at random, Rolling Forcing, Wave Forcing or Self Forcing, a third of the time each. Only Wave Forcing is used at deployment.

The five-step version starts from Rolling Forcing's released weights and skips stage one.

Part three is the execution system, Wave Runtime. The hard part is that every GPU, before it computes attention, has to wait for the K/V, the keys and values, from the store GPU and the GPUs holding cleaner chunks.

So each layer sends out its K/V as soon as the Q, K, V projections and position encoding are done, and only checks that incoming K/V has arrived right before attention. Cleaner chunks read fewer keys, so they tend to run ahead, their K/V often arrives early, and the communication hides behind computation.

The K/V is written straight into the receiver's attention buffers, with no reassembly. And the paper proves that, under fixed conditions, this asynchronous execution gives exactly the same result as the synchronized version.

Finally, the VAE that decodes latents into video runs on its own GPUs, split into a pipeline balanced by measured stage times, so decoding keeps up with generation.

### Results

Quality first, Table 1. Videos are eight thirty-two by four eighty at sixteen frames per second. Long videos use two hundred MovieGen prompts, one hundred seconds each.

Five-step Wave Forcing scores eighty-one point six four on VBench total, against eighty-one point one five for Rolling Forcing. On long-video quality it's eighty-one point two six versus eighty-one point one eight. Basically a tie.

The difference is motion. Dynamic degree goes from fifty-one point three nine to ninety-four point four four on short videos, and from thirty-six point six three to ninety-one point four five on hundred-second videos.

The authors also built two stability metrics, computed on a hundred subject-focused prompts. On subject preservation, Wave Forcing scores ninety-seven point one, Rolling Forcing ninety-two point seven five. On camera drift, lower is better: Wave Forcing is zero point three seven, Rolling Forcing zero point one seven. Rolling Forcing wins that one.

Causal Forcing has the best long-video quality, eighty-two point one five, and the most motion, ninety-six point six one. But its subject preservation is only seventy-one point one one, and its camera drift is eleven point one four. In the paper's example, the subject disappears at thirty and seventy-five seconds, leaving pillows and a headboard in frame.

The ablations are in Table 2, on the four-step model. Skip stage two, and quality falls from eighty point eight nine to seventy-six point six eight, and dynamic degree from seventy-four point nine seven to fifteen point seven three. Drop the Self Forcing branch, and quality is seventy-five point three five with dynamic degree at twenty-three point six six.

Drop the Rolling Forcing branch and motion goes up, but subject preservation falls to seventy-seven point four eight and camera drift rises to six point two four. Drop the Wave Forcing branch and the picture is steadiest, but dynamic degree is only forty-eight point one two. So all three branches stay.

Now speed. Four-step Wave Forcing on one eight-GPU H200 node runs the DiT alone at one hundred forty-seven point six frames per second, and end to end, with decoding, at one hundred thirty-four point eight. That's seven point zero seven times the single-GPU baseline of nineteen point zero six.

One caveat. The speed runs use randomly initialized model weights; only the VAE is real. At 14B, the paper says matching pretrained weights weren't available and they lacked the compute to train one, so 14B is measured for speed only. The best 14B setup reaches thirty-three point five frames per second end to end, against five point five one on a single GPU.

In the system ablations, switching from synchronized exchange to writing straight into the buffers cuts the median time per round for the 1.3B model from ninety-one point nine six to seventy-eight point eight eight milliseconds, fourteen point two percent less.

### Comparison

Against context parallelism, the paper measures DiT-only throughput at six model sizes, from 1.3B to 14B. In BF16, Wave Parallelism wins at five of the six.

At 1.3B it reaches one hundred fifty point zero nine frames per second, against eighty point six for Ulysses and forty-one point one nine for Ring. But at 9B, its forty-five point seven trails Ulysses' forty-six point four one by one point five three percent. With SageAttention and eight-bit quantization it comes first at every size, at one point zero two five to two point five three times the faster baseline. The authors also say the advantage narrows as models grow.

My read is that the speedup comes from putting more GPUs to work at once, not from doing less work. The paper says so itself: it doesn't reduce compute and doesn't cut the time to the first output. Eight GPUs buy roughly seven times the throughput.

My read on quality is that it ties Rolling Forcing overall. The real gain is much more motion without losing the subject, while on camera drift it's still slightly behind Rolling Forcing.

### Limitations

The authors' limitations are specific. First, VBench scores don't fully capture long-video quality, and their own stability metrics use only a hundred prompts, rely on tracking and geometric rules, and haven't been checked against human judgment.

Second, every denoising step needs a full model copy on its own GPU, so with the step count fixed, the size of a single pipeline is capped. The training recipe is also multi-stage and fairly complex.

Third, high throughput isn't the same as fast response. Changing the prompt mid-stream, or steering the camera and actions, hasn't been evaluated at all.

### Future work

Next, the authors plan to profile the decoding path, combine their approach with context parallelism, and simplify training. After that come serving many video streams at once, changing prompts during generation, and eventually interactive world models.

My read is that the lesson worth taking is the co-design. Generation, training and the runtime are designed together, and the attention pattern is changed precisely so the work can run in parallel.

My read on the weak spots: quality is only validated on a 1.3B model, speed is measured with random weights, and the stability metrics are the authors' own. Whether it holds up on bigger models and real interactive use is for follow-up work to show.

That's today's Daily Frontier Papers. The paper is linked in the description.

