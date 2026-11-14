# DexForesight: Grounding Dexterous Policies with Implicit Spatial Foresight

**训练时学会预判上场不用想未来**

- 论文 Paper: [arXiv 2611.11166](https://arxiv.org/abs/2611.11166)
- 单位 Affiliation: Sharpa Robotics、上海交通大学、香港中文大学、复旦大学
- 方向 Direction: 具身智能 / Embodied AI
- 讲解日期 Covered: 2026-11-13
- 视频 Videos: [YouTube 中文](https://youtu.be/TnvuUeSkZ_k) · [YouTube English](https://youtu.be/7a4IQl9am10) · [B站](https://www.bilibili.com/video/BV1WMWz6CEYU)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇 Sharpa Robotics 和上海交通大学等单位的论文，名字叫 DexForesight，讲的是机器人灵巧手操作。

一句话概括，它让机器人策略在训练时，跟一个会预测未来的世界模型学，学会预判手和场景接下来怎么变，但真正上机器人跑的时候，世界模型整个拿掉，不用生成任何未来画面。

### 问题

先说背景。灵巧手，就是有好几根手指、很多关节的机器人手。控制它，要提前想到每根手指相对物体和周围环境会怎么动。

难点在于，抓东西的时候，手指要绕着物体表面走，经常有一部分手被挡住，摄像头看不见。所以只认清当前画面不够，策略还得知道，动作做下去，手和场景会变成什么样。作者把这种预判叫做 spatial foresight，空间预见。

### 以前的方法

以前有一类方法，叫世界动作模型，英文 World Action Model，简称 WAM。它把预测未来和生成动作放在一起学，天然适合学这种预判。

但用到灵巧手上，有两个问题。第一，就算深度图很准，被挡住的手指还是看不见。而如果部署时也要实时估计深度，就得多跑一套重建流程，既费算力，深度视频还得在手和物体运动时保持前后一致。

第二，有些 WAM 在推理时要想象未来，在出动作的同时或之前，生成未来的画面或运动，用来指导选动作，每一步控制都要多算一遍。而且对灵巧手来说，关键是动作怎么改变手和场景的关系，未来画面里的纹理、背景，和选动作关系不大。

### 核心想法

所以作者问了一个问题，能不能只在训练时学到这种预判，部署时一个未来都不用生成。

他们的做法是把两件事拆开。先单独训练一个空间世界模型，专门学手和场景怎么变，再把它学到的特征，搬进策略的隐藏表示里。这张图就是整体思路。

训练世界模型时，可以用上部署时不想实时算的信息，比如离线估计的深度、从机器人运动学算出的手的形状、录下来的未来状态。部署时，策略只看当前的 RGB 图像、语言指令和机器人状态。

### 方法

具体看架构图。左边是空间世界模型，右边是策略，中间靠一组 query 的对齐连起来。

先说世界模型的输入。手的部分，作者在手的每个连杆模型上取一些固定的点，用录下来的关节状态，通过正运动学算出这些点在机器人坐标系里的位置。正运动学，就是从关节角度算出手的每个部位在哪。这样，被挡住的手指也有准确的位置。

深度的部分，真机数据用一个叫 StereoAnyVideo 的方法，从录下来的双目视频里离线估计深度，仿真里直接渲染。深度特征通过交叉注意力，分几层注入世界模型的主干网络。

然后是关键的一步，以动作为条件。世界模型里有一组可学习的 world query，它们读取主干的特征，再接收示范数据里的动作，相当于回答一个问题，如果这样动，未来会怎样。

预测目标有两个。一个是 Hand Flow，也就是手上每个点在未来的位移，另一个是 Visual Dynamics，用冻结的 DINOv3 提取图像特征，预测这些特征在未来的变化量。注意，它预测的是特征的变化，不是生成未来的图片。

未来窗口是 16 帧，分成四段，每段取一个时间点来预测。深度只描述当前的场景，不需要未来的深度当目标。

世界模型的训练数据，是大约 370 小时的真机示范，覆盖 60 个任务，分成空间摆放、装配、旋转、可变形物体和使用工具五类，最后加深度的阶段用其中 100 小时。

世界模型训好之后就冻结。策略这边，作者在视觉语言模型的主干上加一组 policy query，让它们去读主干的隐藏层，然后要求它们和世界模型的 world query 对齐。

对齐的损失有两部分，一部分让每个 policy query 跟对应的 world query 方向一致，另一部分让 query 之间的相似关系也和老师一样。同时，策略原来的动作损失照样保留。

对齐后的 query 会拼进动作专家的上下文，帮它生成动作。注意，策略的输入里没有老师用的那组示范动作，所以这种对齐只是辅助的学习目标，并不是要策略复现老师的预测。

部署时，世界模型、深度编码器、手部几何编码器和两个预测专家全部拿掉，只保留策略，外加那组 query 的读取器和一个投影层。

### 实验和效果

实验分仿真和真机。仿真用的是 Franka 机械臂加 16 自由度的 Allegro 手，真机是 Sharpa North 机器人，配 22 自由度的 Sharpa Wave 灵巧手。

仿真用 DexJoCo 的 11 个任务，有单臂也有双臂，每个任务跑 100 次。DexForesight 加在 GR00T N1.7 上，对比的有两个 VLA 策略，π0.5 和 GR00T N1.7，VLA 就是看图、听指令、直接出动作的模型，还有两个世界动作模型，Fast-WAM 和 Cosmos3。

结果在表 1。DexForesight 总成功率 60.5%，比它自己的底座 GR00T N1.7 的 49.4% 高 11.1 个百分点，比最强的基线 Fast-WAM 高 9.6 个百分点。单臂 76.3%，双臂 41.6%，两组都是最高。

真机上选了三个难任务，关盒子、插电池、拧盖子，分别考验手和物体的几何关系、物体被挡住时的对准，以及手指被挡住时的关节运动。物体位置都是训练里没见过的，每个任务测 20 次。这里 DexForesight 加在 QwenVLA 上。

结果在表 2。DexForesight 总成功率 58.3%，最强的基线是 WLA，50.0%，GR00T N1.7 是 48.3%，Cosmos3 是 30.0%，Fast-WAM 只有 10.0%。它自己的底座 QwenVLA 是 30.0%。

三个任务分开看，拧盖子 50%，和 WLA 打平，关盒子 45%，插电池 80%，都是表里最高。

速度方面，同样的硬件下，QwenVLA 一次推理 227.4 毫秒，DexForesight 是 230.4 毫秒，只多了 3 毫秒。作为参考，Cosmos3 光生成 16 帧未来视频就要 930 毫秒。

那策略里到底有没有学到预判。作者做了一个探针实验，把训练好的真机策略冻结，只用策略自己的特征，训练一个线性模型，去预测手上的点在未来 0.13 秒、0.27 秒和 0.54 秒的位移。

结果在表 3。三个时间点上，DexForesight 的误差都比同架构、只学动作的 QwenVLA 低，而且越往后差得越多。到 0.54 秒，误差从 13.59 毫米降到 10.01 毫米，少了 26.3%。作者的结论是，世界模型拿掉之后，策略特征里还能读出更多关于未来手部运动的信息。

### 和以前方法的对比

一个自然的问题是，为什么不直接在策略上加预测头，让策略自己预测未来。作者在仿真里试了四种直接加的做法，在表 4 的上半部分。

最好的是加一个手部位移预测头，53.1%，比 GR00T 高 3.7 个百分点。DexForesight 是 60.5%，又高了 7.4 个百分点。而把两个预测头一起加上，反而只有 49.5%。

老师给什么样的特征也很关键。如果用还没接收动作的 world query 当老师，只有 52.7%。把配对的动作打乱，掉到 41.5%，比底座 GR00T 的 49.4% 还低。把手部位移这个目标换成预测未来动作，是 48.4%。

世界模型里的深度也有作用。去掉离线深度，仿真里掉 10.8 个百分点，真机上从 58.3% 掉到 43.3%。把交叉注意力式的深度融合，换成直接把深度 token 放进上下文，仿真只有 42.0%。

我的判断是，这篇最有意思的发现在这组对比里。同样的信息，直接让策略去预测，提升有限，有的反而更差，但通过一个以动作为条件的老师间接传给策略，效果明显更好。关键不只是给什么监督信号，还有用什么形式把它交给策略。

我的判断是，和 Spatial Forcing 这类把策略特征对齐到 3D 基础模型的方法比，DexForesight 多走了一步，对齐的不是静态的场景几何，而是动作之下手和场景怎么变。不过论文没有把这类方法放进基线里比。

### 局限

论文没有单独写局限，下面几点是从实验设置里能直接看到的。

第一，真机只有三个任务，每个任务 20 次。DexForesight 比最强基线高 8.3 个百分点，换算下来，60 次里多成功 5 次，论文也没有给真机成功率的置信区间。

第二，作者自己说明，WLA 和 Fast-WAM 跟它在架构和训练上都不一样，不能当成只改一个变量的对照。而且消融基本都在仿真里做，真机上只测了去掉深度这一项。

第三，老师需要的数据不少。世界模型用了大约 370 小时的真机示范，其中 100 小时还要离线估计深度。部署时不用深度，训练时还是离不开它。

### 未来方向

论文没有专门讨论未来工作，结论部分强调的是，部署时策略只用 RGB、语言和机器人状态，成功率还提高了。

我的判断是，这条路线值得继续看的地方在于，昂贵的信号可以只在训练时用，部署成本几乎不变。下一步要看的是，换更多真机任务、做更多次试验，它的优势还能不能稳住。

我的判断是，也要看到绝对水平还不高。仿真里双臂任务的成功率只有 41.6%，真机关盒子是 45%，离稳定可用还有距离。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper comes from Sharpa Robotics and Shanghai Jiao Tong University, among others. It's called DexForesight, and it's about dexterous robot hands.

The short version: a robot policy learns, during training, from a world model that predicts the future, so it picks up a sense of how the hand and the scene are about to change. Then, when the robot actually runs, the world model is thrown away, and nothing about the future is ever generated.

### Problem

Some background first. A dexterous hand is a robot hand with several fingers and lots of joints. To control one, you have to anticipate how each finger will move relative to the object and everything around it.

The hard part is that during a grasp, the fingers wrap around the object, and part of the hand is often hidden from the cameras. So recognizing the current image isn't enough. The policy also needs to know what the hand and the scene will look like once an action plays out. The authors call this spatial foresight.

### Prior work

One existing approach is the world action model, or WAM. It learns to predict the future and to generate actions together, which makes it a natural fit for learning this kind of foresight.

But on dexterous hands, there are two problems. First, even a perfect depth map can't see a finger hidden behind an object. And if you need depth at deployment, you have to run a reconstruction pipeline online, which costs compute, and the depth video has to stay consistent while the hand and the object move.

Second, some WAMs imagine the future at inference time, generating future images or motion alongside or before their actions to guide action selection, which adds compute at every control step. And for a dexterous hand, what matters is how actions change the hand and the scene, not the texture and background in a predicted frame.

### Key idea

So the authors ask: can a policy learn this foresight during training, and then generate no future at all when it's deployed?

Their answer is to split the job in two. First train a separate spatial world model that only learns how the hand and the scene evolve, then move what it learned into the policy's hidden features. This figure shows the overall idea.

The world model can use signals you wouldn't want to compute live, like depth estimated offline, hand shape computed from the robot's kinematics, and recorded future states. At deployment, the policy sees only the current RGB images, the language instruction, and the robot state.

### Method

Here's the architecture. The spatial world model is on the left, the policy is on the right, and the two are connected by aligning a set of queries in the middle.

Start with the world model's inputs. For the hand, the authors sample fixed points on each link of the hand model, and use the recorded joint states and forward kinematics to place those points in the robot's frame. Forward kinematics just means computing where each part of the hand is from its joint angles. So even fingers hidden from the cameras get accurate positions.

For depth, real-robot data gets depth estimated offline from the recorded stereo video with a method called StereoAnyVideo, and simulation simply renders it. Depth features are injected into several layers of the world model's backbone through cross-attention.

Then comes the key step: conditioning on actions. The world model has a set of learnable world queries that read the backbone's features and also receive the actions from the demonstration. In effect they answer a question: if the robot moves like this, what happens next?

There are two prediction targets. One is hand flow, the future displacement of every point on the hand. The other is visual dynamics: frozen DINOv3 image features, and how they change in the future. Note that it predicts a change in features, not a future picture.

The future window is sixteen frames, split into four bins, with one time point predicted from each. Depth only describes the current scene, so no future depth is needed as a target.

The world model is trained on roughly three hundred seventy hours of real-robot demonstrations across sixty tasks in five categories: spatial arrangement, assembly, rotation, deformable objects and tool use. The final, depth stage uses one hundred of those hours.

Once trained, the world model is frozen. On the policy side, the authors add a set of policy queries to the vision-language backbone, let them read its hidden layers, and require them to match the world model's queries.

The alignment loss has two parts. One makes each policy query point the same way as its matching world query. The other makes the similarities among queries match the teacher's. The policy's original action loss stays in place.

The aligned queries are added to the action expert's context to help it generate actions. Note that the policy never receives the teacher's demonstrated actions as input, so this alignment is only an auxiliary learning target, not an attempt to reproduce the teacher's predictions.

At deployment, the world model, the depth encoder, the hand-geometry encoder and both prediction experts are all removed. Only the policy stays, plus the query reader and one projection layer.

### Results

The experiments cover simulation and a real robot. Simulation uses a Franka arm with a sixteen-degree-of-freedom Allegro hand. The real robot is a Sharpa North with twenty-two-degree-of-freedom Sharpa Wave hands.

In simulation there are eleven DexJoCo tasks, single-arm and bimanual, with a hundred trials each. DexForesight is built on GR00T N1.7 and compared with two vision-language-action policies, pi zero point five and GR00T N1.7, and two world action models, Fast-WAM and Cosmos3.

The results are in Table 1. DexForesight's overall success rate is sixty point five percent, eleven point one points above its own base, GR00T N1.7, at forty-nine point four, and nine point six points above the strongest baseline, Fast-WAM. It's the best in both groups, seventy-six point three single-arm and forty-one point six bimanual.

On the real robot there are three hard tasks, close a box, insert a battery, and screw on a lid, testing hand-object geometry, alignment when the object is occluded, and joint motion when the fingers are occluded. Every trial uses object positions held out from training, with twenty trials per task. Here DexForesight is built on QwenVLA.

The results are in Table 2. DexForesight succeeds fifty-eight point three percent of the time. The strongest baseline, WLA, gets fifty, GR00T N1.7 forty-eight point three, Cosmos3 thirty, and Fast-WAM only ten. Its own base, QwenVLA, gets thirty.

Task by task, it hits fifty percent on the lid, tied with WLA, forty-five on the box, and eighty on the battery, the best in the table on each.

On speed, with the same hardware, QwenVLA takes two hundred twenty-seven point four milliseconds per inference and DexForesight two hundred thirty point four, just three milliseconds more. For reference, Cosmos3 needs nine hundred thirty milliseconds just to generate a sixteen-frame future video.

But does the policy actually contain foresight? The authors run a probe. They freeze the trained real-robot policy, take only its own features, and fit a linear model to predict where the hand points will be zero point one three, zero point two seven and zero point five four seconds ahead.

That's Table 3. At every horizon, DexForesight's error is lower than the architecture-matched QwenVLA trained only on actions, and the gap grows with time. At zero point five four seconds, error drops from thirteen point five nine to ten point zero one millimeters, a twenty-six point three percent reduction. The authors conclude that even with the world model gone, more information about future hand motion can be read out of the policy.

### Comparison

The obvious question: why not just bolt prediction heads onto the policy and let it predict the future itself? The authors tried four direct additions to the policy in simulation, shown in the top block of Table 4.

The best is a hand-flow head, at fifty-three point one percent, three point seven points above GR00T. DexForesight, at sixty point five, is another seven point four points higher. Adding both prediction heads together actually drops to forty-nine point five.

What the teacher provides matters too. Using world queries from before the actions are fed in gives only fifty-two point seven percent. Shuffle the paired actions and it falls to forty-one point five, below the GR00T base at forty-nine point four. Swap the hand-flow target for predicting future actions and you get forty-eight point four.

Depth in the world model matters as well. Remove the offline depth and simulation drops ten point eight points, while the real robot falls from fifty-eight point three to forty-three point three. Replace the cross-attention depth fusion with plain depth tokens in the context, and simulation reaches only forty-two percent.

My read is that the most interesting finding is in this comparison. The same information, asked of the policy directly, helps a little or even hurts, but passed through an action-conditioned teacher, it helps clearly. What matters is not only which supervision you use, but the form in which you hand it to the policy.

My read is also that compared with methods like Spatial Forcing, which align policy features to a 3D foundation model, DexForesight goes a step further: it aligns not to static scene geometry but to how the hand and scene change under an action. The paper doesn't include those methods as baselines, though.

### Limitations

The paper has no separate limitations section, so here's what the experimental setup itself shows.

First, the real robot has only three tasks, twenty trials each. DexForesight's eight point three point lead over the strongest baseline works out to five more successes out of sixty, and the paper gives no confidence intervals for the real-robot success rates.

Second, the authors themselves note that WLA and Fast-WAM differ in architecture and training, so they aren't one-variable controls. And the ablations are almost all in simulation; on the real robot, only removing depth was tested.

Third, the teacher needs a lot of data. The world model uses about three hundred seventy hours of real-robot demonstrations, and one hundred of them need depth estimated offline. The deployed policy doesn't need depth, but training still does.

### Future work

The paper doesn't discuss future work. Its conclusion stresses that the deployed policy uses only RGB, language and robot state, and still improves success.

My read is that what makes this direction worth following is that expensive signals can be used only at training time, with almost no added deployment cost. The thing to watch is whether the advantage holds up with more real-robot tasks and more trials.

My read is also that absolute performance is still modest. Bimanual success in simulation is only forty-one point six percent, and closing the box on the real robot is forty-five percent, a long way from dependable.

That's today's Daily Frontier Papers. The paper is linked in the description.

