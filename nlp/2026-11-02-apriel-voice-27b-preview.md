# Apriel Voice 27B Preview: An Open Model for Speech, Reasoning, and Tool Use

**给大模型装上耳朵和双手**

- 论文 Paper: [arXiv 2610.40738](https://arxiv.org/abs/2610.40738)
- 单位 Affiliation: ServiceNow
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-02
- 视频 Videos: [YouTube 中文](https://youtu.be/q1m8McxWodw) · [YouTube English](https://youtu.be/cHS4KCcpBGM) · [B站](https://www.bilibili.com/video/BV1514w6eEW4)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 ServiceNow 的技术报告，Apriel Voice 27B Preview，一个开放的语音语言模型，能听，能推理，还能调用工具。

一句话概括，他们在 Gemma 4 的一个现成模型上，接了一对耳朵，让同一个模型直接听用户说话，自己思考，自己决定要不要调用工具。

### 问题

先说为什么难。论文开头举了个例子，员工打电话说账号被锁了，可能还发了张截图，中途改了一下编号，最后让助手开个工单。

这时候，光把话听准是不够的。模型可能转写得一字不差，却不会遵守约束，也不会调用该调用的工具。反过来，文字上很强的模型，换成语音输入，也不一定还能可靠地推理。

所以作者的目标是，把语音当成通向模型全部能力的一个入口来评估，而不只是测识别准不准。

### 以前的方法

现在做语音助手，最常见的是三段式级联，先用 ASR，也就是语音识别，把话转成文字，交给大模型，再用 TTS 把回答读出来。

另一类是语音条件模型，直接把声学特征喂给大模型，还有的连输出的声音都由模型自己生成，也就是端到端的语音到语音。

还有一种部署方式，前台一个负责聊天的语音模型，把推理和工具调用交给后台另一个模型。这些设计，都是把听、想、说和对话控制，分给不同的组件。

### 核心想法

Apriel Voice 的思路是，听懂请求，接着对话，决定什么时候调用哪个工具，全部交给同一个推理主干。原来的文字和图片输入也都保留着。

论文把这种结构叫两段式，语音先投影进语言主干，推理时直接用声学表示，中间不先转写成文字。

输出端的语音合成是独立的，换声音，换部署方案，都不用重新训练这个推理模型。

真正的难点在训练，怎么给一个已经很强的模型加上语音接口，同时不把它原来的本事弄丢。

### 方法

先看结构，就是这张图。底座是 Gemma 4 26B-A4B，一个混合专家模型，也就是 MoE，总参数大约 260 亿，每个 token 只激活大约 40 亿。

声音进来，先过一个冻结的 Whisper large-v3-turbo 编码器，再过一个学出来的投影层，映射到主干的输入空间。

投影层沿用 Ultravox 的做法，把相邻的 8 帧声学特征拼成一组，序列长度缩到大约八分之一，再映射成主干能用的向量。整个投影层只有大约 4770 万参数。

训练分两个阶段，这张图画得很清楚。第一阶段只训练投影层，编码器和主干都冻住，任务是多语言的语音转写、翻译和续写，先让主干听得懂。

这一阶段用了 1817 万条训练样本。音频每轮大约 3.2 万小时，训练了两轮。

第二阶段，投影层、主干，还有 MoE 的路由器一起训练。数据混了数学、科学、日常对话、语音任务、带噪声的交互、工具调用，还有一部分纯文字的回放数据。

对话数据是他们内部合成的。先写好文字版的对话，包括用户请求、助手回答、工具调用和返回结果，再挑出用户说的部分，用声音克隆的 TTS 读出来。

这里有个细节。像代码、表格、区分大小写的编号，说出来反而不清楚，所以留成文字。图里的例子，资产编号 LT-042 保持文字，登录不了、帮我开个工单这句，变成语音。

同一段对话里，所有语音用同一个参考声音合成，合成之后还要做保真检查，比如把合成的语音再识别一遍，跟原文对比，不合格就重做。

作者也说得很清楚，这些检查只能减少语音和原文对不上的情况，不能保证合成的任务和标准答案本身是对的。

损失函数是这篇的核心设计。每个数据来源可以单独选，用交叉熵，也就是直接学标准答案，或者用前向 KL 散度，去贴近老师模型在整个词表上的输出分布，或者两者按权重混合。

老师是谁呢，就是训练开始前那个原版 Gemma 的冻结副本。学生听的是语音，老师看的是同一段内容的文字版，两边只在回答的位置上对齐，不需要声音帧和文字一一对应。

论文推导了一下，两种损失合在一起，相当于用一个软标签做交叉熵，软标签是标准答案和老师分布的加权混合。交叉熵教新的回答和工具调用，KL 把老师的行为迁移到语音上，并在回放数据上约束模型别跑偏。

工程上，在线蒸馏意味着每一步都要多跑一遍老师。他们每个节点 2 张卡跑老师，6 张卡跑学生，老师先算下一批，学生同时算这一批，把两边的时间叠起来。

第二阶段一共用了 8 个节点，16 张 H100 跑老师，48 张跑学生，大约 3 天训练完，总共过了 526 亿个 token。

### 实验和效果

评测分三块，语音上的推理和工具调用，完整的对话和任务完成，还有加了语音之后，文字和图像能力有没有退步。

对比对象有三类，闭源的端到端语音模型，比如 GPT-realtime-2.1 和 Gemini 3.8 系列，三段式级联，比如 Parakeet 识别加 Gemma4-26B，还有开源的两段式模型，比如 UltraVox。

先看语音推理，在表 5。开启思考模式，Big Bench Audio 98.90，函数调用的 BFCL 81.13，八项平均 85.84，在开源两段式模型里最高。

但同一张表里，开源的三段式 Parakeet 加 Gemma4-26B，平均 86.63，比它还略高一点。最高的是闭源的 Gemini 3.8 系列，90.75。

语音识别和翻译在表 4。LibriSpeech 干净测试集的词错误率是 2.23，各识别基准平均 12.61，比 Parakeet 的 10.06 和 Audex 的 11.40 要高，识别不是它最强的地方。德语、法语、西班牙语译成英语，平均 BLEU 42.87，和 GPT-realtime-2.1 的 43.72 接近。

更重要的是完整任务，在表 6。EVA 是一个服务场景的语音 Agent 评测，同时看任务准确率和对话体验。Apriel Voice 准确率 51.27，体验 74.13。

对照一下，闭源的 Gemini 3.8 Live 是 54.82 和 73.5，GPT-realtime-2.1 是 59.03 和 61.6。也就是说，准确率它还落后闭源模型，体验分是表里最高的。开源三段式 Parakeet 加 Gemma4-26B，准确率 39.97，体验只有 3.93。

体验分差这么多，看分项，差距最大的是轮流说话。Apriel Voice 的 turn-taking 得分 86.52，三段式基线只有 40.54。但它的 faithfulness，也就是回答和服务流程是否一致，只有 43.49，比基线的 66.33 低不少。

再看 τ-Voice，一个在真实音频条件下测任务完成的基准。Apriel Voice 成功率 65.51%，平均延迟 1.91 秒，延迟包括生成回答和合成语音。

同样基于 Gemma 4 的两个三段式级联，成功率 65.92% 和 67.78%，跟它差不多，但延迟是 2.92 秒和 3.01 秒。闭源的 Gemini 3.8 Live 成功率最高，84.19%，延迟 1.50 秒。

最后是有没有变笨，看表 8。文字题上，MMLU-Pro 从 85.27 降到 84.42，GPQA Diamond 从 77.80 升到 80.40，AIME 和 AMC 持平。

图像上，九项里三项上升，六项下降，比如 OCRBench 从 859 降到 836，MathVision 从 64.14 升到 67.10。

### 和以前方法的对比

和以前的工作比，作者自己说，编码器加投影层这个结构，是沿用 Ultravox 和 SLAM-ASR 这类成熟做法。用文字模型的输出来教语音模型，BLSP 和 BLSP-KD 也做过。

他们说自己的贡献在于怎么挑选和实现这些监督信号，两阶段训练，按数据来源选监督方式，合成语音对话，再加上撑得住全词表在线蒸馏的训练工程。

我的判断是，这篇最有价值的结果，不是哪个单项分数，而是同样基于 Gemma 4，做成两段式之后，任务成功率和三段式级联差不多，延迟却少了大约一秒。对做语音客服的团队来说，这一秒很实在。

我的判断是，也要看到差距。准确率上，它在 EVA 上还落后 GPT-realtime-2.1 大约 8 个百分点，在 τ-Voice 上落后 Gemini 3.8 Live 将近 19 个百分点。

### 局限

作者列了几条局限。第一，它主要针对英语，多语言能力和评测都有限，主要靠语音翻译来评，英语的效果不代表其他语言。

第二，内部评测和公开结果的设置不一样，提示词、推理预算、打分方式都有差别。这些汇总分数说明不了显著性，也不是和所有前沿系统的严格排名。

第三，合成数据和蒸馏，可能把错误的答案、老师的错误预测一起传下去。论文也没有做消融，说不清哪个设计起了多大作用，换到别的模型家族也还没测。

### 未来方向

作者说，下一版的重点是提升多语言的语音理解，以及在不同语系上的对话稳定性。

部署方面，他们做了一个共享的实时 API，通过 WebSocket 接入，还提供了一个叫 Apriel Live 的交互界面。

我的判断是，这篇说明，把一个强文字模型改成直接听语音，基本保住文字能力，对话体验还能追上闭源系统，是走得通的。下一步真正要看的，是回答的忠实度和多语言能不能补上，这两项现在是它的短板。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is a technical report from ServiceNow on Apriel Voice 27B Preview, an open speech-language model that listens, reasons, and calls tools.

The short version: they took an existing Gemma 4 model and gave it ears, so one model hears the user directly, thinks, and decides on its own whether to call a tool.

### Problem

Why is that hard? The paper opens with an example. An employee calls about a locked account, maybe shares a screenshot, corrects an ID number halfway through, and asks the assistant to open a ticket.

Hearing the words correctly isn't enough here. A model can transcribe a request perfectly and still ignore a constraint or fail to call the right tool. And a model that's strong on text won't necessarily reason as reliably when the same request arrives as audio.

So the authors set out to evaluate speech as a way into everything the model can do, not just as a recognition test.

### Prior work

The most common voice assistant today is a three-stage cascade. Speech recognition, or ASR, turns audio into text, a language model answers, and text-to-speech reads the answer out.

A second family is speech-conditioned models, which feed acoustic features straight into the language model. Some go further and generate the output audio too, end-to-end speech to speech.

There's also a deployment pattern where a conversational voice model sits in front and hands reasoning and tool work to a separate backend model. All of these split listening, thinking, speaking and turn control across different components.

### Key idea

Apriel Voice puts understanding the request, holding the conversation, and deciding when to call which tool into one reasoning backbone. The original text and image inputs stay in place.

The paper calls this a two-stage design. Speech is projected into the language backbone, and inference uses those acoustic representations directly, with no transcript in between.

Speech synthesis on the output side is a separate, swappable piece, so you can change the voice or the deployment without retraining the reasoning model.

The real challenge is training. How do you give an already capable model a speech interface without losing what made it useful?

### Method

Start with the architecture, shown in this figure. The base is Gemma 4 26B-A4B, a mixture-of-experts model with about twenty-six billion parameters in total and about four billion active per token.

Audio first goes through a frozen Whisper large-v3-turbo encoder, then through a learned projector that maps it into the backbone's input space.

The projector follows Ultravox. It stacks eight adjacent frames of acoustic features into one group, which cuts the sequence to roughly an eighth of its length, then passes it through two linear layers and a gate. The whole projector is only about forty-seven point seven million parameters.

Training happens in two stages, and this figure lays them out. Stage one trains only the projector, with the encoder and backbone frozen, on multilingual transcription, translation and continuation, so the backbone learns to make sense of speech.

That stage uses eighteen point one seven million training examples. Each pass covers about thirty-two thousand hours of audio, and they ran two passes.

Stage two trains the projector, the backbone, and the mixture-of-experts router together. The mix includes math, science, everyday conversation, speech tasks, noisy interactions, tool use, and some text-only replay data.

The conversation data is synthesized in-house. They first write the dialogue as text, with user requests, assistant replies, tool calls and results, then pick out parts of what the user says and render them as audio with voice-cloning text-to-speech.

One nice detail. Things like code, tables or case-sensitive IDs are clearer as text than as speech, so they stay as text and only the spoken part becomes audio. In this figure's example, the asset ID LT-042 stays as text, while "I can't sign in, please open a ticket" becomes audio.

Within one conversation, every spoken turn uses the same reference voice. After synthesis there are fidelity checks, such as transcribing the generated audio and comparing it with the source, and failures are regenerated.

The authors are careful to say these checks only reduce mismatches between the intended text and the audio. They don't make the synthetic task or its target answer correct.

The loss function is the central design choice. Each data source can choose cross-entropy, which learns the target answer directly, or forward KL divergence, which pulls the model toward a teacher's full output distribution over the vocabulary, or a weighted mix of both.

The teacher is a frozen copy of the original Gemma checkpoint. The student hears audio, the teacher reads the matching text version, and the two are aligned only on the response positions, so audio frames never have to line up one-to-one with text tokens.

The paper shows that combining the two losses is, up to a constant, the same as a scaled cross-entropy against a soft target, a weighted blend of the correct answer and the teacher's distribution. Cross-entropy teaches new answers and tool behavior, while KL carries the teacher's behavior over to speech and keeps the model from drifting on replay data.

On the engineering side, online distillation means an extra teacher forward pass at every step. Each node runs the teacher on two GPUs and the student on six. The teacher works on the next batch while the student trains on the current one, and this pipeline overlaps the two.

Stage two used eight nodes, sixteen H100s for the teacher and forty-eight for the student, finished in about three days, and processed about fifty-two point six billion tokens in total.

### Results

The evaluation asks three questions. Can it reason and call tools from speech? Does that carry through to full conversations and completed tasks? And did adding speech hurt its text and image skills?

The comparison set has three groups: closed speech-to-speech models like GPT-realtime-2.1 and the Gemini 3.8 family, three-stage cascades like Parakeet recognition plus Gemma4-26B, and open two-stage models like UltraVox.

Speech reasoning first, in Table 5. With thinking turned on, it scores ninety-eight point nine on Big Bench Audio and eighty-one point one three on the BFCL function-calling test, for an eight-benchmark average of eighty-five point eight four, the best among open two-stage models.

But in the same table, the open three-stage cascade, Parakeet plus Gemma4-26B, averages eighty-six point six three, slightly higher. The top score belongs to the closed Gemini 3.8 family, at ninety point seven five.

Recognition and translation are in Table 4. Word error rate on LibriSpeech test-clean is two point two three, and the average across recognition benchmarks is twelve point six one, higher than Parakeet's ten point zero six and Audex's eleven point four, so recognition isn't its strongest point. Translating German, French and Spanish into English, it averages forty-two point eight seven BLEU, close to GPT-realtime-2.1 at forty-three point seven two.

What matters more is finishing real tasks, in Table 6. EVA is a voice-agent benchmark for service settings that scores both task accuracy and conversational experience. Apriel Voice gets fifty-one point two seven on accuracy and seventy-four point one three on experience.

For reference, the closed Gemini 3.8 Live gets fifty-four point eight two and seventy-three point five, and GPT-realtime-2.1 gets fifty-nine point zero three and sixty-one point six. So it still trails the closed models on accuracy, but its experience score is the highest in the table. The open cascade, Parakeet plus Gemma4-26B, gets thirty-nine point nine seven on accuracy and just three point nine three on experience.

Among the parts of that experience score, the biggest difference is turn-taking. Apriel Voice scores eighty-six point five two on turn-taking, against forty point five four for the cascade baseline. But its faithfulness score, whether the response is consistent with the service interaction, is only forty-three point four nine, well below the baseline's sixty-six point three three.

Next is tau-Voice, which tests task completion under realistic audio conditions. Apriel Voice succeeds sixty-five point five one percent of the time, with an average latency of one point nine one seconds, counting response generation and speech synthesis.

The two Gemma 4 based cascades land at sixty-five point nine two and sixty-seven point seven eight percent, about the same, but their latencies are two point nine two and three point zero one seconds. The closed Gemini 3.8 Live has the highest success rate, eighty-four point one nine percent, at one point five seconds.

Finally, did it get worse at everything else? Table 8. On text, MMLU-Pro dips from eighty-five point two seven to eighty-four point four two, GPQA Diamond rises from seventy-seven point eight to eighty point four, and AIME and AMC are unchanged.

On images, three of nine scores go up and six go down. OCRBench, for example, drops from eight hundred fifty-nine to eight hundred thirty-six, while MathVision rises from sixty-four point one four to sixty-seven point one.

### Comparison

Against earlier work, the authors are upfront: the encoder-plus-projector design follows established approaches like Ultravox and SLAM-ASR. And using a text model's outputs to teach a speech model was done before in BLSP and BLSP-KD.

They place their contribution in how these supervision signals are selected and implemented: the two-stage recipe, per-source choice of supervision, synthetic spoken dialogues, and the training engineering that makes full-vocabulary online distillation practical.

My read is that the most useful result here isn't any single score. It's that, starting from Gemma 4, the two-stage version completes about as many tasks as the cascades, with roughly a second less latency. For teams building voice support agents, that second is real.

My read is also that the gap is still there. On accuracy, it trails GPT-realtime-2.1 on EVA by about eight points, and Gemini 3.8 Live on tau-Voice by almost nineteen.

### Limitations

The authors list several limitations. First, it targets English. Multilingual ability and evaluation are limited, mostly measured through speech translation, and English results don't necessarily carry over to other languages.

Second, internal and public evaluations differ in prompts, reasoning budgets, judging and task limits. The aggregate scores don't establish significance, and they aren't a controlled ranking against every frontier system.

Third, synthetic data and distillation can pass on wrong targets and wrong teacher predictions. There are no ablations, so we can't tell which design choice did how much, and transfer to other model families is untested.

### Future work

The authors name multilingual speech understanding and conversational robustness across language families as the priority for future versions.

For deployment, they built a shared real-time API over WebSocket that plugs straight into voice benchmarks, plus an interactive interface called Apriel Live.

My read is that this shows you can turn a strong text model into one that listens directly, keep most of its text ability, and match closed systems on conversational experience. What to watch next is whether faithfulness and multilingual support catch up, because those are its weak spots right now.

That's today's Daily Frontier Papers. The paper is linked in the description.

