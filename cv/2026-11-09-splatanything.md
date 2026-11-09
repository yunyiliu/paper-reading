# SplatAnything: Gaussian Spacing Normalization for Feed-forward 3DGS across Arbitrary Scenes

**换个缩放因子3D重建更通用**

- 论文 Paper: [arXiv 2611.08264](https://arxiv.org/abs/2611.08264)
- 单位 Affiliation: 浙江大学 · 腾讯混元 · Causa Robotics
- 方向 Direction: CV
- 讲解日期 Covered: 2026-11-09
- 视频 Videos: [YouTube 中文](https://youtu.be/GLCZOxERZ9c) · [YouTube English](https://youtu.be/wRMto_4l1VU) · [B站](https://www.bilibili.com/video/BV1TWWb6XEtf)

## 中文讲解

### 开场

这里是每日前沿文章。今天讲一篇浙江大学和腾讯混元的论文，名字叫 SplatAnything。它要做的事是，给任意数量的照片，几秒钟内直接重建出一个 3D 场景。

这篇论文的核心改动其实很小，只是换了一种归一化方式，但作者认为，这正是现有方法换个场景就不好用的一个关键原因。

### 问题

先补一点背景。3D Gaussian Splatting，简称 3DGS，是用大量小的高斯椭球来表示一个场景，每个椭球有自己的位置、透明度、形状和颜色。

传统做法要对每个场景单独优化很久。前馈式 3DGS 不一样，它训练一个网络，输入照片，一次前向就直接输出这些高斯，省掉了逐场景优化。

理想的目标是一个模型通吃，场景大小不限，相机怎么拍都行，要看的视角也不限。但这种泛化，目前还很难做到。

### 以前的方法

现在主流的路线分两步。先用一个几何基础模型，从照片估计出每个像素的深度，再在这个几何的基础上，给每个像素预测一个高斯。

这些几何模型输出的场景，尺度是归一化过的。常用的做法叫径向距离归一化，简称 RDNorm，就是算出所有 3D 点到参考相机的平均距离，再把整个场景缩放，让这个平均距离等于 1。

问题就出在这里。RDNorm 管住了场景的整体大小，却没有管住相邻两个高斯中心之间的距离，论文把它叫作高斯间距。

高斯的大小，必须和这个间距匹配，才能把物体表面盖住。所以间距一变，网络要学的目标也跟着变。

论文这张图量化了这个问题。在 RDNorm 下，物体、室内、户外三类场景的中位高斯间距，分别是论文方法的 0.57 倍、1.36 倍和 0.30 倍，最多相差 4.5 倍。

结果就是，看起来差不多的局部画面，在不同场景里要预测的高斯大小却不一样，监督信号是模糊的。预测偏小，间距大的地方就会出现空洞，预测偏大，密集的地方又会糊成一片。

### 核心想法

作者的想法很直接。既然高斯的大小跟着间距走，那就别按场景整体的大小来归一化，直接按高斯间距来归一化。

这个方法叫高斯间距归一化，简称 GSNorm。这张图左右对比，左边是 RDNorm，径向距离一致了，但间距和监督目标都不一致。右边是 GSNorm，间距一致，监督目标也就一致了。

### 方法

高斯间距怎么算。每个有效像素对应一个高斯中心，按针孔相机模型，相邻两个像素反投影到 3D 之后的距离，大约等于这个像素的深度除以焦距。

横向和纵向各有一个焦距，论文取两个方向的几何平均，所以一个像素的局部间距，约等于深度除以两个焦距乘积的平方根。

这一点很关键。间距不只取决于场景大小，还跟相机和分辨率有关。论文指出，几何模型用的归一化，保证不了不同场景、相机和分辨率下的间距一致。

GSNorm 的做法是，把所有视角、所有有效像素的间距都算出来，取中位数，作为整个场景的缩放因子，再用它去缩放几何、深度和相机位移。

缩放之后，整个场景的中位间距就等于 1。因为只是一个全局缩放，局部间距之间的相对差别还保留着。用中位数，是因为它更稳健。

整体架构在这张图里。先用一个冻结的 WorldMirror 2.0 几何模型，估计深度、相机内参和位姿，然后把几何从径向归一化空间，换到高斯间距归一化空间。

接着，一个从 WorldMirror 2.0 初始化、再微调的 Gaussian Backbone，加上一个从头训练的轻量 GSHead，给每个像素预测透明度、旋转、大小和颜色系数。高斯的中心，就是反投影出来的那个点。

相机位姿可以给，也可以不给。训练只用渲染损失，像素级的 MSE 加上 LPIPS 感知损失。论文在 64 张 NVIDIA H20 上训练了两万步，每次输入 4 到 32 张图。

### 实验和效果

为了评测，作者还搭了一个新的基准，覆盖七个数据集，包括 ScanNet、DL3DV、Tanks and Temples、Mip-NeRF 360 和 DTU 等，这些数据集的物理尺度跨度很大。

更重要的是，测试视角特意选在输入照片覆盖范围之外，和输入的重叠控制在百分之二十到六十，考的是外推，不是插值。输入张数从 1 张到 64 张都测。

指标主要用 LPIPS 和 DISTS，两个都是感知距离，越低越好。作者的理由是，它们更能反映渲染质量和高频细节。

结果在表 1，是 4 到 64 张输入的平均。不给相机位姿时，在七个数据集上，SplatAnything 的 DISTS 全部第一，LPIPS 七个里有六个第一，只有 ETH3D 上输给了 QuerySplat 和 YoNoSplat。

差距最大的是 DTU。不给位姿时，SplatAnything 的 LPIPS 是 0.483，第二名 DA3-GS 是 0.582。

给位姿时，DISTS 也是七个全部第一，LPIPS 七个里有五个第一，Tanks and Temples 和 ETH3D 上 LPIPS 最好的是 ReSplat。

单张图输入的结果在表 2，对手是专门做单目新视角合成的 SHARP 和 InfiniSplat。这里是互有胜负，DL3DV、Tanks and Temples、ETH3D 和 DTU 上 SplatAnything 更好，ScanNet、NRGBD 和 Mip-NeRF 360 上 SHARP 更好。

最关键的消融在表 3，同样的架构和训练设置，只换归一化空间。4 张输入时，RDNorm 的 DISTS 和 LPIPS 是 0.223 和 0.278，GSNorm 降到 0.198 和 0.245。

有意思的是第三个对照，RDNorm 加尺度校准，也就是在 RDNorm 空间里预测完，再把高斯大小统一缩放到场景的中位间距。结果它比单纯的 RDNorm 还差，4 张输入时 LPIPS 是 0.309。

作者的解释是，预测完再修正尺度，解决不了监督目标本身模糊的问题，必须在预测之前就把空间对齐。

这张图能直观看到差别。RDNorm 在小物体上会出空洞，在大的户外场景里偏糊。加了校准，物体上的空洞往往变成了糊，户外还是糊。GSNorm 更完整，也更清晰。

附录还比了另外两种归一化。按相机之间的距离归一化，单张图输入时直接失效，DISTS 高达 0.793。按深度归一化，忽略了焦距和图像采样，视场角或者分辨率一变，监督目标还是会跑偏。

还有一个训练细节。附录表 13 显示，把高斯中心固定在几何模型给出的点上，而不是让网络再学一个偏移，是几项设计里提升最大、最稳定的。作者认为，让中心自由移动，会偏离几何先验，外推视角就变差。

### 和以前方法的对比

和以前的方法比，SplatAnything 改的不是几何模型，而是网络在哪个坐标空间里预测高斯。论文指出，以前基于几何基础模型的前馈方法，通常直接沿用几何骨干的径向归一化空间。

但有两个指标要看清楚。第一是 PSNR，也就是逐像素误差。附录表 10 里，给位姿时，4 到 32 张输入，PSNR 最高的都是 ReSplat，比如 4 张输入，ReSplat 是 19.03，SplatAnything 是 18.39。不给位姿、4 张输入时，YoNoSplat 的 18.23 也高于它的 17.85。作者认为，PSNR 偏爱平滑的结果，而 ReSplat 在大视角外推时会出现明显伪影。

第二是速度。附录表 11 里，在一张 H20 上，4 张输入的前向时间是 411 毫秒，64 张大约 11 秒，比同样不需要位姿的 Mirror2-GS 和 AnySplat 慢，但比 DA3-GS 和 QuerySplat 快很多。

我的判断是，这篇的价值在于找准了一个容易被忽视的细节。改动只是换一个缩放因子，消融又说明预测之后再补救不行，这个结论对沿用几何模型坐标系的前馈 3DGS 方法都有参考意义。

我的判断是，也要看到，主要结论建立在作者自己设计的基准和自己选的感知指标上。按 PSNR 看，它在多数设置下不是第一，单张图输入也没有全面胜过 SHARP。消融里到 64 张输入时，GSNorm 的 DISTS 只从 0.224 降到 0.217，优势明显变小。

### 局限

作者自己写了两条局限。

第一，它依赖几何基础模型给的几何先验，输入极其稀疏或者很难的时候，几何恢复不出来，效果就会下降。

第二，它主要重建输入照片里看得到的区域，没拍到的地方会留下空洞。

另外附录里也说明，GSNorm 只统一了单张图里的间距。多张图叠在一起，同一块区域的高斯密度还是会随输入张数变化，这一部分靠体素合并来处理。

### 未来方向

未来方向，作者提到两条，一是更稳健的几何估计，二是用生成式先验，去补全没拍到的区域。

附录里还试了一个应用，先用 WorldPano 生成一张全景图，切成视场角很宽的透视图，再交给 SplatAnything 重建，作者称新视角合成的质量很高。

我的判断是，接下来值得看的是，这种按采样间距来归一化的思路，能不能直接用到别的前馈 3DGS 模型上，论文只在自己的模型上验证过。

以上就是今天的每日前沿文章，论文链接在简介里。

## English walkthrough

### Intro

This is Daily Frontier Papers. Today's paper is from Zhejiang University and Tencent Hunyuan. It's called SplatAnything, and the pitch is simple: hand it any number of photos, and within seconds you get back a 3D scene.

What makes it interesting is how small the core change is. It swaps one normalization step, and the authors argue that this step is a key reason current methods break when you move to a different kind of scene.

### Problem

Some background first. 3D Gaussian Splatting, or 3DGS, represents a scene as a large set of small Gaussian blobs, each with a position, an opacity, a shape and a color.

The classic approach optimizes those blobs from scratch for every scene, which is slow. Feed-forward 3DGS trains a network instead: photos go in, and one forward pass produces the Gaussians directly.

The ideal is one model for everything, any scene scale, any camera setup, any viewpoint you want to render. That kind of generalization is still hard.

### Prior work

The dominant recipe today has two steps. A geometry foundation model first estimates the depth of every pixel, and then a second network predicts a Gaussian for each pixel on top of that geometry.

The geometry comes out in a normalized scale. The usual choice is radial distance normalization, RDNorm: take the average distance of all the 3D points from the reference camera, and rescale the whole scene so that average equals one.

Here's the catch. RDNorm fixes the overall size of the scene, but not the distance between neighboring Gaussian centers, which the paper calls Gaussian spacing.

A Gaussian has to be about as big as that spacing to cover the surface. So when the spacing shifts, the target the network is supposed to learn shifts with it.

This figure puts numbers on it. Under RDNorm, the median Gaussian spacing for objects, indoor scenes and outdoor scenes is zero point five seven, one point three six and zero point three zero times the paper's normalized value. That's up to a four point five times spread.

So two patches that look alike can need very different Gaussian sizes in different scenes, and the supervision becomes ambiguous. Guess too small and you get holes where the spacing is wide. Guess too big and dense regions blur together.

### Key idea

The authors' fix follows directly. If Gaussian size tracks spacing, don't normalize by the overall size of the scene. Normalize by the Gaussian spacing itself.

They call it Gaussian Spacing Normalization, or GSNorm. This figure shows the contrast. On the left, RDNorm makes the radial distance consistent, but the spacing and the supervision targets aren't. On the right, GSNorm makes the spacing consistent, and the targets line up too.

### Method

So how do you compute spacing? Each valid pixel becomes one Gaussian center. Under a pinhole camera, two neighboring pixels projected back into 3D sit roughly depth divided by focal length apart.

There's a focal length for each direction, and the paper takes the geometric mean. So a pixel's local spacing is about its depth divided by the square root of the two focal lengths multiplied together.

That's the key point. Spacing depends not just on how big the scene is, but on the camera and the image resolution. The paper notes that the normalization geometry models use doesn't keep spacing consistent across scenes, cameras and resolutions.

GSNorm computes the spacing for every valid pixel in every view, takes the median, and uses that as the scene's scale factor. Geometry, depth and camera translations are all divided by it.

After that, the scene's median spacing equals one. Because it's a single global rescale, the relative differences in local spacing are preserved. The median is used because it's robust.

Here's the full architecture. A frozen WorldMirror 2.0 geometry model estimates depth, camera intrinsics and poses, and the geometry is then moved from the radial-distance space into the Gaussian-spacing space.

Next, a Gaussian Backbone initialized from WorldMirror 2.0 and fine-tuned, plus a lightweight GSHead trained from scratch, predict each pixel's opacity, rotation, scale and color coefficients. The Gaussian's center is simply the back-projected point.

Camera poses are optional. Training uses only rendering losses, pixel-wise MSE plus the LPIPS perceptual loss. The model trains for twenty thousand iterations on sixty-four NVIDIA H20 GPUs, with four to thirty-two input views at a time.

### Results

To evaluate it, the authors built a new benchmark across seven datasets, including ScanNet, DL3DV, Tanks and Temples, Mip-NeRF 360 and DTU, which cover a wide range of physical scene scales.

More importantly, the target views are chosen outside the region the input photos cover, with only twenty to sixty percent overlap. That tests extrapolation, not interpolation. Inputs range from one view to sixty-four.

The main metrics are LPIPS and DISTS, both perceptual distances where lower is better. The authors pick them because they better reflect rendering quality and fine detail.

Table 1 averages over four to sixty-four input views. Without camera poses, SplatAnything has the best DISTS on all seven datasets and the best LPIPS on six of seven. The exception is ETH3D, where QuerySplat and YoNoSplat do better.

The biggest margin is on DTU. Without poses, SplatAnything's LPIPS is zero point four eight three. The runner-up, DA3-GS, is at zero point five eight two.

With poses, it again leads DISTS on all seven, and LPIPS on five of seven. On Tanks and Temples and ETH3D, ReSplat has the best LPIPS.

Table 2 covers single-image input, against SHARP and InfiniSplat, two methods built for monocular view synthesis. Here it's a split. SplatAnything wins on DL3DV, Tanks and Temples, ETH3D and DTU, while SHARP wins on ScanNet, NRGBD and Mip-NeRF 360.

The key ablation is Table 3: same architecture, same training, only the normalization space changes. With four input views, RDNorm scores zero point two two three DISTS and zero point two seven eight LPIPS. GSNorm brings that down to zero point one nine eight and zero point two four five.

The interesting row is the third one: RDNorm with scale calibration. It predicts in RDNorm space, then rescales every Gaussian to match the scene's median spacing afterwards. It ends up worse than plain RDNorm, with an LPIPS of zero point three zero nine at four views.

The authors' explanation is that fixing scale after prediction can't undo supervision that was ambiguous during training. The space has to be aligned before the network predicts anything.

You can see it in this figure. RDNorm leaves holes on small objects and blurs large outdoor scenes. Calibration often turns the holes into blur, and the outdoor scenes stay blurry. GSNorm is more complete and sharper.

The appendix tests two more options. Normalizing by distances between cameras collapses with a single input image, with a DISTS of zero point seven nine three. Normalizing by depth ignores focal length and image sampling, so a change in field of view or resolution still throws the targets off.

One more training detail. Appendix Table 13 shows that pinning Gaussian centers to the geometry model's points, rather than letting the network learn an offset, gives the largest and most consistent gain of the design choices tested. The authors' reading is that free-moving centers drift away from the geometry prior and hurt extrapolated views.

### Comparison

Compared with earlier work, SplatAnything doesn't change the geometry model. It changes the coordinate space where Gaussians are predicted. The paper points out that previous feed-forward methods built on geometry foundation models typically inherit their backbone's radial-distance space.

But two other numbers deserve attention. The first is PSNR, a pixel-level error measure. In appendix Table 10, ReSplat has the highest PSNR with poses from four to thirty-two views. At four views, ReSplat scores nineteen point zero three and SplatAnything eighteen point three nine. Without poses at four views, YoNoSplat's eighteen point two three also beats its seventeen point eight five. The authors argue PSNR rewards overly smooth output, and that ReSplat shows clear artifacts under large viewpoint changes.

The second is speed. In appendix Table 11, on a single H20, a forward pass takes four hundred and eleven milliseconds with four views and about eleven seconds with sixty-four. That's slower than Mirror2-GS and AnySplat, which also run without poses, but much faster than DA3-GS and QuerySplat.

My read is that the paper's value is in spotting a detail that's easy to miss. The change is just a different scale factor, and the ablation shows that patching scale after prediction doesn't work. I think that lesson is worth checking for any feed-forward 3DGS method that inherits a geometry model's coordinate frame.

My read is also that the headline claims rest on the authors' own benchmark and their chosen perceptual metrics. By PSNR it isn't first in most settings, and with a single image it doesn't beat SHARP across the board. At sixty-four views in the ablation, GSNorm's DISTS only drops from zero point two two four to zero point two one seven, a much smaller edge.

### Limitations

The authors list two limitations.

First, it depends on the geometry foundation model's priors. With extremely sparse or difficult inputs, where the geometry can't be recovered, quality drops.

Second, it mainly reconstructs what the input photos can see. Areas nobody photographed can be left as holes.

The appendix adds one more point. GSNorm only evens out spacing within a single view. When many views overlap the same region, Gaussian density still grows with the number of views, and that part is handled by merging Gaussians within voxels.

### Future work

For future work, the authors name two directions: more robust geometry estimation, and generative priors to fill in the regions nobody saw.

The appendix also tries an application. WorldPano generates a panorama, which is cut into very wide-angle perspective views and fed to SplatAnything. The authors report high-quality novel views from it.

My read is that the next thing to watch is whether this idea, normalizing by sampling spacing, carries over to other feed-forward 3DGS models. The paper only tests it on its own.

That's today's Daily Frontier Papers. The paper is linked in the description.

