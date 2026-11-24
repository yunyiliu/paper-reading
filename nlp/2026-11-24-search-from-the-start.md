# Search From the Start: Search-Native Pretraining for Language Models

**搜索能力该从预训练教起**

- 论文 Paper: [arXiv 2611.26809](https://arxiv.org/abs/2611.26809)
- 单位 Affiliation: EPFL · 多伦多大学
- 方向 Direction: NLP
- 讲解日期 Covered: 2026-11-24
- 视频 Videos: [B站](https://www.bilibili.com/video/BV1ysyfzKED4)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇洛桑联邦理工学院，也就是 EPFL，和多伦多大学的论文，题目叫 Search From the Start，从一开始就学搜索，讲的是一种搜索原生的预训练方法。

问题很直接，现在的大模型上线后大多要配搜索工具，那为什么搜索，要等到预训练结束才开始教。

### 问题

语言模型能在参数里记住的事实是有限的。小模型尤其吃亏，很多事实没法从规律里推出来，只能硬背，背下来的知识还会过时。

让模型学会搜索，知识就可以放在外面，随时更新，不用重新训练。但现在的做法，基本是先用静态文本做预训练，只学预测下一个 token，搜索能力到后训练阶段才加进去。

作者想知道，能不能把普通的预训练文档大规模改造一下，在预训练阶段就教会搜索，同时不改变预测下一个 token 这个目标。

### 以前的方法

推理时接搜索，或者在后训练里教模型调用工具，这类工作很多。和这篇最接近的是 Toolformer，它把工具调用的示范直接插进文本，再去微调模型。

也有人在预训练阶段下功夫，比如联合训练一个检索器，或者加一个专门读检索结果的层。最近的 LMLM 和 Co-LMLM，是从数据里建一个以实体为键的知识库，让模型去查。

作者说，这篇的不同在于，模型从头预训练，学的是写自由形式的搜索词，不改模型结构，也不需要按实体整理的知识库。

### 核心想法

核心想法看这张图就清楚了。在一篇普通文档里，找到那些需要外部知识才能接着写下去的位置，插进一条搜索词，和一段搜索结果，然后照常做预训练。

关键在损失怎么算。原文和搜索词都参与训练，模型要学会在哪里搜，搜什么。搜索结果那一段不算损失，模型不用背它，只是借它来预测后面的原文。

### 方法

第一步，决定在哪里插搜索。作者先用 GLM-5.2 标注了一万篇维基百科文章，标出名字，日期，数量这类具体事实前面，适合搜索的位置。

这里有个细节。老师模型先看全文挑位置，但写搜索词的时候，只能看这个位置前面的文字，和模型推理时能看到的一样，这样能降低搜索词里直接带上答案的风险。

然后用这批数据，微调一个小的标注模型，Gemma 4 E4B。它在整篇文档上跑一遍前向，就能算出每个位置后面插入搜索的概率，超过阈值的位置，再由它写出搜索词。

第二步，搜索结果从哪里来。给整个预训练语料调用商业搜索 API 太贵，所以作者用 BM25 这种经典的关键词检索，在一个叫 Sphere 的公开网页集合上建了本地索引，里面有 1.34 亿篇网页文档。

每个搜索词取前 10 个段落，挑出和搜索词重合最多的句子，当作搜索结果。之后还用 ROUGE 重合度，过滤掉和下文关系不大的插入。

训练语料有两种。维基百科用来做受控实验，DCLM-Edu 是更接近真实情况的网页语料，加上搜索之后超过 1000 亿 token。模型从头训练，规模从 3.6 亿到 33 亿参数，作者把它叫做 QueryLM。

公平对比是这篇的关键。每个 QueryLM 都配一个同结构的普通模型，训练算力大致对齐，搜索结果虽然不算损失，但它占用的计算也算了进去。所以同样的算力下，QueryLM 读到的原文，比基线少。

### 实验和效果

评测用八个知识密集型问答任务，四个是事实问答，比如 TriviaQA，四个是多跳问答，比如 HotpotQA，要把两条以上的证据连起来。主结果的搜索来源是 Tavily，一个通过 API 调用的搜索引擎。

先看还没做微调的基础模型。要注意，基线也能看到搜索结果，它用的是 QueryLM 写的搜索词，和搜回来的内容，比的是谁更会用这些证据。

在 DCLM-Edu 上，八个任务平均，4 亿参数的 QueryLM 是 24.8%，基线是 7.5%。14 亿是 30.3% 对 18.6%，33 亿是 32.0% 对 23.9%。差距随着规模变小，从 17.3 个点降到 8.1 个点。

作者还拿它和别家的开源模型比。4 亿的 QueryLM 开着搜索，平均 24.8%，超过不开搜索的 SmolLM3-3B 的 23.6%，和 Qwen2.5-3B 的 23.0%。按论文的估算，它的预训练算力只有 SmolLM3-3B 的大约六千七百分之一。

和直接的竞争方法比，结果就没那么一边倒。用同样的检索来源，DCLM-Edu 上的 QueryLM 和 Co-LMLM 基本打平，19.8% 对 19.1%。维基百科上，QueryLM 的平均分还略低于 LMLM，虽然八个任务里赢了六个。

最重要的问题是，这个优势会不会在后训练里被抹平。作者做了两种微调。一种是标准微调，参照 Olmo 3 的配方，里面有一部分搜索数据。另一种叫 Search SFT，搜索示范的比例高得多。

标准微调之后，三个规模分别是 22.2% 对 16.5%，27.8% 对 20.4%，33.7% 对 27.5%，差距在 5.7 到 7.4 个点之间。

Search SFT 之后，差距缩小到 3.8，2.9 和 2.5 个点，但还在。作者的结论是，他们试过的后训练配方，不能完全补回预训练里学到的搜索能力。他们推测，小模型参数里记得少，所以受益更多。

摘要里还有一句，14 亿的 QueryLM，在知识密集型任务上，总是打平或者超过算力对齐的 33 亿基线。看表里的数，Search SFT 之后是 36.0% 对 36.9%，在置信区间之内，算打平，而它的训练算力大约只有对方的五分之一。

那它到底强在哪里。作者做了一个反事实测试，把搜索结果改成错的，比如说爱因斯坦出生在 1889 年，实际是 1879 年，看模型会不会跟着证据改答案。

这张图是问答准确率。给被改过的证据时，按证据作答的比例，QueryLM 三个规模是 57.9%，62.0% 和 64.3%，基线是 51.7%，55.4% 和 55.1%。给原始证据时，两边很接近。不给证据时，QueryLM 反而更差。

搜索词的质量也有差别。把模型自己写的搜索词，换成更强的 Qwen3-30B 写的，4 亿的基线从 15.7% 涨到 23.6%，QueryLM 只从 23.7% 涨到 25.0%，说明 QueryLM 自己写的搜索词已经不错。如果两边都用 Qwen 的搜索词，4 亿和 14 亿上 QueryLM 还领先，到了 33 亿就反过来了。

成本方面，作者试了两个省钱的办法。在 3.6 亿的维基百科模型上，把搜索结果从 600 个字符截到 50 个字符，或者每篇文档只留两次搜索，训练算力都降到基线的一半多一点，搜索能力弱了，但还在。

每篇只留两次搜索的版本，作者又在 33 亿规模上训了一个。知识问答的成绩落在 QueryLM 和基线之间，通用能力和基线在置信区间内持平。

### 和以前方法的对比

和后训练才教搜索相比，这篇最有说服力的地方，是对照做得严。同样的结构，同样的算力，同样的微调数据，只差预训练时有没有见过搜索，两种微调之后，优势都还在。

和 LMLM，Co-LMLM 相比，它不用按实体建知识库，搜索词是自由文本，能直接接网页搜索。论文报告，换成 Tavily 后，QueryLM 超过了这两个方法，但那时双方的检索来源已经不同。

我的判断是，这篇的价值，在于把搜索当成一种从预训练就开始培养的能力，并且用算力对齐的对照说明，后训练不能完全补回来。对想做小模型加搜索的团队，这是一个很实际的方向。

我的判断是，几个亮眼的数字要打折看。六千七百分之一算力那个比较，一边开着网页搜索，一边是闭卷，不是同一个设定。而且 QueryLM 的优势随规模在缩小，实验最大只到 33 亿参数。

### 局限

论文没有单独的局限一节，但它的结果里有几处要注意。第一，闭卷能力下降。不让搜索时，微调后的 33 亿 QueryLM，八个任务平均只有 5.0%，同规模的基线是 15.4%。作者的解释是，模型把预训练里很多靠死记的部分，转成了靠搜索。

第二，通用能力有一点代价。在 ARC，HellaSwag 这类不用搜索的常识推理测试上，基础模型平均低 1.1 到 2.5 个点。做完 Search SFT 之后，差距变小，有高有低，标准微调之后，平均还是低 0.7 到 2.2 个点。

第三，在 FreshQA 这个考时效性知识的测试上，两边都做完 Search SFT，都能搜索之后，成绩基本一样。微调前那个很大的领先，是拿能搜索的 QueryLM 和闭卷的基线比出来的。

第四，更愿意跟着证据走，也意味着证据错了，它更容易跟着错。反事实实验里，证据被改错时，QueryLM 按错误证据作答的比例比基线更高。

### 未来方向

作者在结论里说，把搜索放进预训练，可以补充小模型有限的参数记忆，教它去获取和使用外部知识。论文另外提到，标注数据每个语料只要做一次，可以给不同的模型重复使用。

我的判断是，接下来最值得看两件事。一是到了更大的规模，这个优势会不会继续缩小。二是检索质量差的时候，一个更相信证据的模型，要怎么判断该不该信，这篇还没有回答。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from EPFL and the University of Toronto. It's called Search From the Start, and it proposes what the authors call search-native pretraining.

The question is simple. Most deployed language models now get a search tool. So why do we only teach them to search after pretraining is over?

### Problem

A language model can only store so many facts in its weights. Small models feel this most. Facts that don't follow from general patterns have to be memorized, and memorized facts go stale.

Teaching a model to search lets that knowledge live outside, where it can be updated without retraining. But today, pretraining is still about predicting the next token of static text, and search gets added later, in post-training.

The authors ask whether ordinary pretraining documents can be augmented at scale to teach search during pretraining, while keeping the plain next-token objective.

### Prior work

There's a lot of work on plugging search in at inference time, or teaching tool use in post-training. The closest relative is Toolformer, which inserts tool-call demonstrations into text and then fine-tunes the model.

Others have worked at the pretraining stage, with a jointly trained retriever, or a special layer that attends to retrieved text. Recent methods called LMLM and Co-LMLM build an entity-keyed knowledge store from data and train the model to look things up in it.

The difference here, according to the authors, is that the model is pretrained from scratch to write free-form search queries, with no architecture change and no entity store.

### Key idea

This figure sums it up. Take an ordinary document, find the spots where you'd need outside knowledge to keep writing, insert a search query and a search result there, and pretrain as usual.

The trick is in the loss. The original text and the queries are trained on, so the model learns when and what to search. The search result is masked out of the loss. The model never learns to recite it. It only uses it to predict the text that follows.

### Method

Step one is deciding where to search. The authors had GLM-5.2 annotate ten thousand Wikipedia articles, marking good search points just before concrete facts like names, dates and quantities.

One detail matters here. The teacher picks positions while seeing the whole article, but it writes each query seeing only the text before that point, just like the model at inference. That lowers the risk of the answer leaking into the query.

That data is used to fine-tune a small annotator, Gemma 4 E4B. In one forward pass over a document, it scores the chance of a search after every token, and at positions above a threshold, it writes the query.

Step two is where results come from. Calling a commercial search API for a whole pretraining corpus is too expensive, so the authors built a local BM25 keyword index over Sphere, a public collection of one hundred thirty-four million web documents.

For each query, they take the top ten passages and keep the sentences that overlap the query most. Then a ROUGE overlap filter drops insertions whose results have little to do with the text that follows.

There are two corpora. Wikipedia is the controlled setting, and DCLM-Edu is realistic web text, which grows to over one hundred billion tokens with search added. Models are trained from scratch, from three hundred sixty million to three point three billion parameters, and they're called QueryLM.

The comparison is what makes this convincing. Every QueryLM gets a baseline with the same architecture and roughly the same training compute, counting the masked search tokens too. So at equal compute, QueryLM actually sees fewer document tokens than its baseline.

### Results

Evaluation uses eight knowledge-heavy QA tasks. Four are factual, like TriviaQA, and four are multi-hop, like HotpotQA, where you need to chain two or more pieces of evidence. The main results search with Tavily, an API-based search engine.

Start with base models, before any fine-tuning. Note that the baseline also sees search results, using QueryLM's queries and the retrieved text. So the test is who makes better use of the evidence.

On DCLM-Edu, averaged over the eight tasks, the four hundred million parameter QueryLM scores twenty-four point eight percent against seven point five for the baseline. At one point four billion it's thirty point three versus eighteen point six, and at three point three billion, thirty-two versus twenty-three point nine. The gap shrinks with scale, from seventeen point three points to eight point one.

The authors also compare against outside open models. The four hundred million QueryLM with search averages twenty-four point eight, above SmolLM3-3B at twenty-three point six and Qwen2.5-3B at twenty-three, both without search. By the paper's estimate, that's about six thousand seven hundred times less pretraining compute than SmolLM3-3B.

Against its direct competitors, the picture is less lopsided. With the same retrieval source, QueryLM on DCLM-Edu roughly ties Co-LMLM, nineteen point eight versus nineteen point one. On Wikipedia, QueryLM's average is slightly below LMLM, even though it wins six of the eight tasks.

The key question is whether post-training erases the advantage. The authors try two kinds of fine-tuning. Standard SFT follows the Olmo 3 recipe and includes some search data. Search SFT uses a much larger share of search demonstrations.

After standard SFT, the three sizes score twenty-two point two versus sixteen point five, twenty-seven point eight versus twenty point four, and thirty-three point seven versus twenty-seven point five. That's a gap of five point seven to seven point four points.

After Search SFT, the gap narrows to three point eight, two point nine and two point five points, but it doesn't close. The authors conclude that the post-training recipes they tested don't fully recover what search pretraining built. They suspect small models gain most, because they remember less in their weights.

The abstract also claims the one point four billion QueryLM always matches or beats the compute-matched three point three billion baseline on these tasks. In the tables, after Search SFT, that's thirty-six point zero versus thirty-six point nine, inside the confidence interval, so a tie, with roughly a fifth of the training compute.

So where does the edge come from? The authors run a counterfactual test. They edit the evidence to be wrong, say that Einstein was born in eighteen eighty-nine instead of eighteen seventy-nine, and check whether the model follows it.

This chart shows QA accuracy. With edited evidence, QueryLM answers the way the evidence says fifty-seven point nine, sixty-two and sixty-four point three percent of the time across the three sizes, versus fifty-one point seven, fifty-five point four and fifty-five point one for the baseline. With the original evidence, the two are close. With no evidence, QueryLM is actually worse.

Query quality differs too. Swap each model's own queries for ones written by the stronger Qwen3-30B, and the four hundred million baseline jumps from fifteen point seven to twenty-three point six, while QueryLM only goes from twenty-three point seven to twenty-five. So QueryLM's own queries were already decent. When both use Qwen's queries, QueryLM still leads at four hundred million and one point four billion, but at three point three billion the order flips.

On cost, the authors try two cheaper variants. On the three hundred sixty million Wikipedia model, cutting search results from six hundred characters to fifty, or keeping only two searches per document, brings training compute down to a bit over half the baseline's. Search ability drops, but it's still there.

They then trained the two-searches-per-document version at three point three billion. Its QA scores land between QueryLM and the baseline, and its general ability matches the baseline within the confidence intervals.

### Comparison

Compared with teaching search only in post-training, the strength of this paper is the tight control. Same architecture, same compute, same fine-tuning data, the only difference is whether search appeared in pretraining, and the advantage survives both fine-tuning recipes.

Compared with LMLM and Co-LMLM, there's no entity store to build, and the queries are free text that can go straight to a web search engine. The paper reports that with Tavily, QueryLM beats both methods, but at that point the two sides aren't using the same retrieval source.

My read is that the real contribution is treating search as a skill grown from pretraining, and showing with compute-matched controls that post-training doesn't fully make up for it. For teams building small models that rely on search, that's a practical direction.

My read is also that a few headline numbers need a discount. The six thousand seven hundred times comparison pits a model with web search against closed-book models, which isn't the same setting. And QueryLM's edge shrinks with scale, with the largest model at only three point three billion parameters.

### Limitations

The paper has no separate limitations section, but its own results raise a few. First, closed-book ability drops. Without search, the fine-tuned three point three billion QueryLM averages just five percent on the eight tasks, against fifteen point four for the same-size baseline. The authors read this as the model shifting much of its memorization over to search.

Second, there's a small cost in general ability. On commonsense tests without search, like ARC and HellaSwag, the base models trail by one point one to two point five points on average. After Search SFT the gap shrinks and goes either way, while after standard SFT QueryLM still trails by zero point seven to two point two points on average.

Third, on FreshQA, a test of time-sensitive facts, once both models have Search SFT and both can search, they score about the same. The big lead before fine-tuning came from comparing a searching QueryLM with a closed-book baseline.

Fourth, being more willing to follow evidence also means following bad evidence. In the counterfactual test, when the evidence was edited to be wrong, QueryLM went along with it more often than the baseline.

### Future work

In their conclusion, the authors say putting search into pretraining can complement a small model's limited memory by teaching it to find and use outside knowledge. Elsewhere, the paper notes that the annotated data is built once per corpus and can be reused across models.

My read is that two things are worth watching. First, whether the advantage keeps shrinking at larger scales. Second, when retrieval is poor, how a model that trusts evidence more decides what to believe, which this paper doesn't answer yet.

That's today's Daily Frontier Papers. The paper is linked in the description.

