## 13章 光线跟踪阴影：保持实时帧速率

### 摘要

高效、准确的阴影计算是计算机图形学中一个长期存在的挑战。在实时应用程序中，通常使用基于光栅化的管线来计算阴影。随着图形硬件的激动人心的新进展，现在可以在实时应用程序中使用光线跟踪，使光线跟踪阴影成为光栅化的可行替代方案。虽然光线跟踪阴影可以避免光栅化阴影中许多的固有问题，但是如果所需光线的数量增加（例如，对于高分辨率、具有多个灯光的场景或区域灯光），单独跟踪每个阴影光线可能会成为一个性能瓶颈。因此，计算应该集中在阴影覆盖的图像区域，特别是阴影的边界。

我们提出了一种在实时场景中应用光线跟踪阴影的实用方法。我们的方法使用标准的光栅化管线来分辨场景几何的可见度(g-buffer.译注)，使用光线追踪来分辨光源的可见度。提出了一种自适应的结合自适应阴影滤波和阴影光线追踪的算法。这两种技术允许在对每个像素使用有限的阴影光线数的前提下计算高质量阴影。我们使用最新的实时光线跟踪API（DXR）评估了我们的方法，并将结果与使用级联阴影(CSM)的传统方法进行了比较。

### 13.1 介绍

阴影有助于辨认真实的场景。由于阴影的重要性，过去有许多技术都是为阴影计算而设计的。虽然离线渲染应用程序使用光线跟踪进行阴影计算[20]，但实时应用程序通常使用ShadowMap[21]。ShadowMap在场景几何方面非常灵活，但它有几个重要问题：

- 透视失真，由于阴影贴图分辨率不足或其区域分布不当，呈现为锯齿状阴影。
- 自阴影失真和不连续阴影(Peter Panning)。
- 缺少半影(软阴影)。
- 缺少对半透明遮挡的支持。

为了解决这些问题，已经有许多技术[7,6]被提出。通常，他们需要场景设计师的手动微调才能获得较好的效果。这使得ShadowMap的简洁实现变得复杂，对于不同的场景通常需要不同的解决方案。

光线跟踪[20]是一种灵活的范例，它可以用简单的算法计算精确的阴影，并且能够以直观和可伸缩的方式处理复杂的照明（区域光、半透明遮挡物）。然而，很难使光线跟踪的性能满足实时应用的需要。这是由于硬件资源有限以及实时光线跟踪所需的底层算法的实现复杂性，例如空间数据结构的快速构建和维护。在用于实时应用程序的流行图形API中，也没有明确的光线跟踪支持。

随着NVIDIA RTX和DirectX Raytracing(DXR)的引入，现在可以直接使用DirectX和Vulkan API来利用光线跟踪。最新的NVIDIA图灵图形体系结构为DXR使用专用的RT核心提供硬件支持，这大大提高了光线跟踪的性能。这些新功能与新兴的混合渲染方法[11]很好地结合在一起，使用光栅化来解决主光线可见性，并使用光线跟踪以计算阴影、反射和其他照明效果。

然而，即使有了新的强大的硬件支持，在使用光线跟踪渲染高质量阴影时，我们也必须明智地使用我们的计算资源。一个简单的算法可能很容易投射过多的光线来对来自多个光源或区域光源的阴影进行采样，从而导致低帧速率。示例见图13.1。

![img](https://pic4.zhimg.com/v2-f93b2c2b13da5d77a61dd984d04a66f3_b.png)

在本章中，我们介绍了一种遵循混合渲染思想的方法。该方法基于光源能见度的时域空域分析，采用自适应阴影采样和自适应阴影滤波来优化光线跟踪阴影。我们使用Falcor[3]框架评估我们的方法，并将其与CSM[8]和普通光线跟踪阴影[20]进行比较。

### 13.2 相关工作

阴影从一开始就一直是计算机图形学研究的焦点。它们是纯白光线跟踪[20]的原生产物----对于每个命中点，投射阴影光线到每个光源以确定光源可见性。通过distributed ray tracing[4]引入软阴影，在该方法中阴影项为投射到区域光源的多条阴影光线的平均可见性。这一原理仍然是当今许多软阴影算法的基础。

实时阴影很早就已经通过ShadowMap[21]和ShadowVolume[5]算法实现了。由于其简单高效，目前大多数交互式应用程序都使用ShadowMap，尽管存在一些缺点和由其离散性引起的失真。目前有几种基于ShadowMap的软阴影算法，最引人注目的是percentage closer soft shadows[9]。然而，尽管已经对原始算法进行了许多改进(详细内容请参阅Eisemann等人的书和课程[6,7])，鲁棒快速的软阴影仍然是一个难以实现的目标。

受交互式光线跟踪技术发展的启发[18]，研究人员最近又开始研究光线跟踪在硬阴影和软阴影中的应用。但是，与执行完整的光线跟踪过程不同，一个关键的想法是对主要光线使用光栅化，并且仅对阴影光线使用光线跟踪[1,19]，这产生了混合渲染管线。为了使这种方法适用于软阴影，研究人员正在尝试以各种方式进行时域累计[2]。

Nvidia Turing架构最终将完全硬件加速光线跟踪引入消费级市场，DirectX(DXR)和Vulkan API中可与光栅化管道轻松集成。尽管如此，多光源软阴影仍然是一个挑战，需要智能的自适应采样和时域重投影技巧，我们将在本章中介绍。

实时光线跟踪的出现也为其他混合渲染技术打开了大门，例如自适应时间抗锯齿，在这种技术中，无法通过重投影渲染(就是传统TAA里被拒绝的采样.译注)的像素被光线跟踪[14]。在[17]之前，时间一致性被专用于软阴影，但是在这里我们引入了一个更简单的基于变化估计的时间一致性方案来评估所需的采样数。

### 13.3 光线追踪阴影

当一个场景对象(遮挡物)阻挡光线时，阴影就会出现，否则光线对接受阴影的另一个场景对象产生照明。直接或间接光照都可能会产生阴影。直接阴影是在评估主光源的可见性时产生的，间接光照阴影是在场景表面的强反射或折射被遮挡时产生的。在本章中，我们主要讨论直接光照的情况，间接光照可以使用一些标准的全局光照技术(如路径跟踪或many-light methods(可能指VPL/PM等方法?.译注))独立地进行评估。

点P在 ![ω_o](https://www.zhihu.com/equation?tex=%CF%89_o)方向的辐出 ![L(p, ω_o)](https://www.zhihu.com/equation?tex=L(p%2C%20%CF%89_o))由下式确定[12]:

![img](https://pic3.zhimg.com/v2-ffc1a302d1781ad45b989f61b15622be_b.png)

式中， ![L_e(ω_o)](https://www.zhihu.com/equation?tex=L_e(%CF%89_o))是自发光， ![f(P,ω_i,ω_o)](https://www.zhihu.com/equation?tex=f(P%2C%CF%89_i%2C%CF%89_o))是BRDF， ![L_i(P,ω_i)](https://www.zhihu.com/equation?tex=L_i(P%2C%CF%89_i))是 ![ω_i](https://www.zhihu.com/equation?tex=%CF%89_i)方向的辐入， ![\hat np](https://www.zhihu.com/equation?tex=%5Chat%20np)是P点表面法线。

对于具有一组点光源的直接光照，L的直接光照分量可以写成单个光源的贡献之和：

![img](https://pic3.zhimg.com/v2-5b44fd3e056df2c63b2529d6d517e6a2_b.png)

式中， ![P_l](https://www.zhihu.com/equation?tex=P_l)是光源 ![l](https://www.zhihu.com/equation?tex=l)的位置，光源方向为 ![ω_l=(P_l-P)/||P_l-P||](https://www.zhihu.com/equation?tex=%CF%89_l%3D(P_l-P)%2F%7C%7CP_l-P%7C%7C)， ![L_l(P_l,ω_l)](https://www.zhihu.com/equation?tex=L_l(P_l%2C%CF%89_l))是光源 ![l](https://www.zhihu.com/equation?tex=l)在方向 ![ω_l](https://www.zhihu.com/equation?tex=%CF%89_l)的辐出。 ![v(P,P_l)](https://www.zhihu.com/equation?tex=v(P%2CP_l))是可见项，当 ![P_l](https://www.zhihu.com/equation?tex=P_l)对 ![P](https://www.zhihu.com/equation?tex=P)可见时为1，否则为0。

对于 ![v](https://www.zhihu.com/equation?tex=v)的计算，可以通过执行一次从P到 ![P_l](https://www.zhihu.com/equation?tex=P_l)的光线求交并检查交点。必须小心防止表面或者光源由于浮点精度引起的自相交。这通常通过略微减小交点合法值范围来消除。

面光源的 ![L_d](https://www.zhihu.com/equation?tex=L_d)由下式给出：

![img](https://pic2.zhimg.com/v2-59baa015de954f11c47c20b75ce3b68d_b.png)

式中， ![A](https://www.zhihu.com/equation?tex=A)是光源 ![a](https://www.zhihu.com/equation?tex=a)的表面， ![\hat n_X](https://www.zhihu.com/equation?tex=%5Chat%20n_X)是 光源表面![X](https://www.zhihu.com/equation?tex=X)点法线， ![ω_X=(X-P)/||X-P||](https://www.zhihu.com/equation?tex=%CF%89_X%3D(X-P)%2F%7C%7CX-P%7C%7C)是P点到X点的方向， ![L_a(X,ω_X)](https://www.zhihu.com/equation?tex=L_a(X%2C%CF%89_X))是光源 ![a](https://www.zhihu.com/equation?tex=a)的表面![X](https://www.zhihu.com/equation?tex=X)点在 ![ω_X](https://www.zhihu.com/equation?tex=%CF%89_X)方向的辐出。

该积分通常通过使用一组分布良好的采样点 ![S](https://www.zhihu.com/equation?tex=S)的蒙特卡洛积分方法：

![img](https://pic4.zhimg.com/v2-826082dbbf5970735af8656645280d77_b.png)

式中， ![|S|](https://www.zhihu.com/equation?tex=%7CS%7C)是采样点个数。在我们的工作中，我们将着色和可见项分离，在着色时，我们使用质心 ![C](https://www.zhihu.com/equation?tex=C)来近似取代面光源：

![img](https://pic2.zhimg.com/v2-3dba3447539586411cc1e86212a19b25_b.png)

这使我们能够在给定的帧内累计每个灯光的可见性测试结果，并将它们存储在每个灯光的专用可见性缓冲区中。可见性缓冲区是一个屏幕大小的纹理(卧槽真的大手笔，大量光源怕是要翻车.译吐槽)，它保存每个像素的可见性。最近，Heitz等人提出了一种更为复杂的着色与可见性的分离方法。[11]可用于大面积或更复杂的BRDF光源。分离可见性使我们能够将可见性计算与着色分离，以及分析和使用可见性的时间一致性。图13.2所示为点光源和区域光源的可见性评估示意。结果之间的差异如图13.3所示。

![img](https://pic1.zhimg.com/v2-838461e4fc85ccf2bc06907a53189a5c_b.png)

## **13.4 自适应采样**

简单的光线跟踪阴影计算需要大量的光线来达到所需的阴影质量，特别是对于较大面积的面光，如图13.1所示。随着灯光数量或尺寸的增加，这将大大拖累性能。因为只有在图像的半影区域才需要大量的光线，所以我们的方法基于识别这些区域，然后使用更多的光线对其进行有效采样。对完全光照和完全遮挡的区域进行稀疏采样，节省的计算资源可用于其他光线跟踪任务，如反射。  

### **13.4.1 时间重投影**

为了有效地增加每个像素的样本数，我们使用时间重投影方法，这允许我们随着时间的推移累计可见场景表面的可见性值。在许多最近的实时渲染方法中，时间重投影渐渐成为一种标准工具[15]，在许多情况下，它已经在应用程序光栅化管道中实现。我们将累积值用于两个目的：第一，估计可见性变化以推理所需的样本计数；第二，确定用于过滤采样可见性的核大小。

我们将之前帧的可见性计算结果存储在包含四个帧的缓存中。为了确保动态场景的正确结果，我们使用反向投影[15]，它处理摄像机的移动。当开始对一个新帧进行评估时，我们将存储在缓存中的前三个帧反向重投影到当前帧。因此，我们总是有一个由四个连续帧中的值组成的四元组，这些值与当前帧的图像对应。

在第t帧的剪裁空间中给定一个点 ![P_t ](https://www.zhihu.com/equation?tex=P_t%20)，重新投影将在第t−1帧中找到相应的剪裁空间坐标 ![p_{t-1}](https://www.zhihu.com/equation?tex=p_%7Bt-1%7D)，如下式：

![img](https://pic2.zhimg.com/v2-b60483e6c4c6b5e9dcb956acae9348a5_b.png)

式中， ![C_t](https://www.zhihu.com/equation?tex=C_t)和 ![C_{t-1}](https://www.zhihu.com/equation?tex=C_%7Bt-1%7D)是相机的投影矩阵， ![V_t](https://www.zhihu.com/equation?tex=V_t)和 ![V_{t-1}](https://www.zhihu.com/equation?tex=V_%7Bt-1%7D)是相机视口矩阵。在重投影之后，我们计算深度差异并丢弃不合法对应。是否合法按下式判断：

![img](https://pic3.zhimg.com/v2-d6c03f5bb0a2400c7b655a4271d77e76_b.png)

式中，(那啥符号打不来...式中右边的)是深度相似性的最大阈值，![\hat n_z](https://www.zhihu.com/equation?tex=%5Chat%20n_z)是相机空间中对应像素的法向。 ![c_1 c_2 ](https://www.zhihu.com/equation?tex=c_1%20c_2%20)是用户定义的常数(我们使用了0.003和0.017)。自适应阈值允许斜坡表面上有效样本的深度差异更大。

对于成功的重投影点，我们将屏幕空间坐标存储在0到1的范围内。如果重投影失败，我们将存储负值以表明重投影失败，以便进行后续计算。请注意，由于在以前的重投影步骤中已经对齐了所有以前的帧，因此仅一个用于存储深度值的缓存就足够了(没看懂,但是也无关紧要,意会一下吧.译注)。

### 13.4.2 识别半影区域

场景中某点到光源所需的采样（光线）数量通常取决于光源大小、到阴影点的距离以及遮挡几何体的复杂性。由于这种复杂性很难分析，因此我们的方法建立在分析时域可见性变化 ![\Delta v(x)](https://www.zhihu.com/equation?tex=%5CDelta%20v(x))的基础上：

![img](https://pic3.zhimg.com/v2-d1652c5719504d2752481ec1fc8a263e_b.png)

式中 ![v_n](https://www.zhihu.com/equation?tex=v_n)为缓存的前几帧的可见性。注意，每个光源的可见性值被存储在一张四通道纹理中。

所述度量值对应于可见性变化，对可见性函数的极端值非常敏感。因此，与其他更平滑的变化度量方式(如方差)相比，此度量更容易检测半影区域。

对于完全照亮或完全遮挡的区域，可见性变化为零，而在半影区域通常大于零。我们的样本集是按4帧一循环重复的(为了稳定性.译注)，见第13.5.1节。

为了使结果更稳定性，我们在可见性变化测量上加上一个空间滤波器，再加上一个时间滤波器。空间滤波器表示为：

![img](https://pic4.zhimg.com/v2-e848a687f5f0931546dbbec53f1b02cf_b.png)

式中， ![M_{5×5}](https://www.zhihu.com/equation?tex=M_%7B5%C3%975%7D)是一个使用5×5邻域的非线性最大值滤波器，![T_{13×13}](https://www.zhihu.com/equation?tex=T_%7B13%C3%9713%7D)是13×13邻域的低通tent filter(三角形,长得像帐篷的那个.译注)。最大值滤波器确保在单个像素中检测到的高变化量也会导致在周围像素中使用更多的样本。这对于动态场景来说非常重要，以使该方法更加稳定，并且对于在附近像素中完全忽略半影的情况也很重要。帐篷过滤器防止变化值的突然变化，以避免闪烁。这两个过滤器是可分离的，因此我们分双Pass执行它们，以减少计算工作量。

最后，我们将空间滤波后的 ![\widetilde{\Delta v_t}](https://www.zhihu.com/equation?tex=%5Cwidetilde%7B%5CDelta%20v_t%7D)与前四帧的时间滤波值 ![∆v_t](https://www.zhihu.com/equation?tex=%E2%88%86v_t)相结合。对于时间过滤，我们使用一个简单的box filter(方形的,像直方图的那个.译注)，并且我们有意使用在空间过滤之前缓存的原始![∆v_t](https://www.zhihu.com/equation?tex=%E2%88%86v_t)值(可能是出于防止半影mask扩散太远引起性能降低的考量.译注)：

![img](https://pic2.zhimg.com/v2-9cccff1404cbf337bc3ab6e02489782d_b.png)

这种滤波器组合在我们的测试中被证明是有效的，因为它能够将变化传播到更大的区域（使用最大值和帐篷滤波器）。同时，通过将空间过滤变量与前一帧的时间过滤变量值相结合，它不会错过变化较大的小区域。

### 13.4.3 计算采样数目

对给定点使用的样本数量的决定是基于前一帧中使用的样本数量和当前过滤后的可见性变化度量![\overline{∆v_t}](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v_t%7D)。我们在可见性变化度量上应用一个阈值δ来决定是否增加或减少相应像素的采样密度。特别是，我们保持每个像素的样本计数![s(x)](https://www.zhihu.com/equation?tex=s(x))，并使用以下算法更新给定帧中的![s(x)](https://www.zhihu.com/equation?tex=s(x))：

1. 如果![\overline{∆v_t}(x)>δ](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v_t%7D(x)%3E%CE%B4)且 ![s_{t−1}(x) < s_{max}](https://www.zhihu.com/equation?tex=s_%7Bt%E2%88%921%7D(x)%20%3C%20s_%7Bmax%7D)，增加采样数目： ![s_t(x) = s_{t−1}(x) + 1](https://www.zhihu.com/equation?tex=s_t(x)%20%3D%20s_%7Bt%E2%88%921%7D(x)%20%2B%201)
2. 如果![\overline{∆v_t}(x)<δ](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v_t%7D(x)%3C%CE%B4)且前四帧的采样数恒定，减少采样数目：![s_t(x) = s_{t−1}(x)-1](https://www.zhihu.com/equation?tex=s_t(x)%20%3D%20s_%7Bt%E2%88%921%7D(x)-1)

每个光源的最大采样数![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)确保每帧每盏灯的开销有限(标准设置![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)=5，高质量设置![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)=8)。步骤2中使用的四个先前帧的稳定性约束导致算法中出现滞后，目的是防止样本数量和变化之间的反馈循环导致的样本数量振荡。所述技术具有足够的时间稳定性，并有着比直接从![\overline{∆v_t}](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v_t%7D)计算![s(x)](https://www.zhihu.com/equation?tex=s(x))更好的结果。

对于反向重投影失败的像素，我们使用![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)采样并用当前结果替换所有缓存的可见性值。当屏幕上所有像素的反向重投影失败时，例如，当相机姿态发生显著变化时，由于每个像素中使用的采样数太多，性能会突然下降。为了防止性能下降，我们可以在CPU上检测到相机姿态的巨大变化，并且可以暂时减少后续几帧的最大采样数(![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D))。这会暂时导致更糟糕的结果，但它会防止突然的卡顿，而这通常更令人不悦。

### 13.4.4 采样Mask

当采样数等于零的像素表示一个没有时间和空间变化的区域。这主要适用于图像中完全亮起和完全遮挡的区域。对于这些像素，我们可以完全跳过可见性的计算，并使用前一帧中缓存的值。但是，这可能会导致这些区域中随着时间的推移而累积错误，例如，当光源快速移动或相机缓慢缩放时（在这两种情况下，重投影都将成功，但可见性可能会改变）。因此，我们使用一个遮罩，强制对至少四分之一的像素进行采样。因为性能测试表明，四个邻近像素中的一个像素投射一条光线会产生与这些像素中每个像素均投射光线相似的性能（可能是由于warp(4个像素一组执行.译注)）。因此，我们在屏幕上对一个Nb×Nb像素块进行采样(对于Nb=8，我们获得了最佳的性能提升)(没太理解他的意思是怎么实现的，不过意会一下，自己实现的时候注意一下就好.译注)。

为了确保每个像素在四帧中至少采样一次，我们使用一个矩阵来检查是否应在当前帧中强制采样。(此段接下来为意译.译注)将整个屏幕划分为无数4x4的块，并将每2x2个邻近块合并称为组。我们可以通过整除、取余等方法找到某个像素所属的块在组中的编号，若该编号与当前帧循环号(4帧一循环)相等，则对该块中所有像素进行强制采样。通过这样的方式，每个像素将在四个连续帧中至少采样一次，以确保检测到新的阴影。如图13.4所示。使用自适应采样的样本分布示例如图13.5所示。

![img](https://pic1.zhimg.com/v2-00fd17f18af65156d4711be8ec4f84ec_b.png)

### 13.4.5 计算可见性

作为算法的最后一步，我们对可见性计算本身使用了两种过滤技术(与可见性变化度量相反)：时间过滤(利用以前帧的结果)和空间过滤(对可见性值应用低通过滤并去除剩余噪声)。

全局光照的最新去噪方法，如Schied等人的时空方差引导滤波(spatiotemporal variance guided filtering.SVGF)。[16]和基于人工智能的AI降噪器，可以从随机采样图像序列中产生无噪声的结果，即使每个像素只有一个样本。这些方法在去噪后(尤其是在有纹理的材料上)要注意保持边缘清晰度，通常使用无噪声的albedo和normal buffer的信息。我们使用一个更简单的解决方案，专门针对阴影计算进行定制，并与阴影光线的自适应采样策略结合得很好。

**时间滤波**

为了利用可见性值的时间累计，我们计算平均可见性值，有效地对缓存的重投影可见性值应用时间过滤器：

![img](https://pic2.zhimg.com/v2-c621d06d1a6dcca4eff8eb194e592b31_b.png)

使用box filter可以获得最佳的画面效果，因为我们的样本集是在最后四帧中生成的。请注意，我们的方法并没有明确地对光源的移动建模。我们的结果表明，对于交互帧速率(>30fps)和仅缓存四个前帧，这种简化所引入的失真非常小。

**空间滤波**

空间滤波在已经由时间过滤步骤处理的可见性Buffer上进行。我们使用传统的可变大小高斯核的交叉双边滤波器来过滤可见性。滤波核的大小选择在1×1和9×9像素之间，并通过可见性变化度量![\overline{∆v_t}](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v_t%7D)(原文写的是 ![\widetilde{\Delta v_t}](https://www.zhihu.com/equation?tex=%5Cwidetilde%7B%5CDelta%20v_t%7D) ,我觉得应该是![\overline{∆v_t}](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v_t%7D),下同.译注)给出。给定区域内的更多变化导致更积极的去噪。滤波器的大小根据![\overline{∆v_t}](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v_t%7D)线性缩放，而最大的内核大小是根据预定义的η实现的(我们使用了η=0.4)。为了防止相邻像素从一个内核大小切换到另一个时出现跳变，我们为每个像素存储计算好的高斯核大小，并进行线性差值。这对于与最小的内核大小混合以在需要时保留硬边特别重要。

![img](https://pic1.zhimg.com/v2-3b6d4dcb5e72f731f12f953655a08fac_b.png)

我们利用depth和normal信息来防止阴影在几何不连续面上泄漏。这使得过滤器不可分离(不能分pass.译注)，但它具有相当好的结果，如图13.6所示。深度不满足方程式13.7要求的样本不予考虑。此外，我们丢弃了所有对应法线相似性测试未通过的样本，测试方法如下：

![img](https://pic2.zhimg.com/v2-a9a1f27da0e959a3e151899dca833da9_b.png)

对于有效对应，其法线点乘结果应大于ζ，我们使用了ζ=0.9。

时间滤波将每四个灯光的过滤后的可见性缓冲区打包为一个四通道纹理。然后，每个空间过滤过程同时对其中两个纹理进行操作，每次对八个可见性Buffer降噪。

## 13.5 实现

本节描述了有关算法实现的详细信息。

### 13.5.1 采样点集生成

我们的自适应采样方法假设我们处理四帧一循环的样本。由于该方法对每个像素使用不同的样本计数，因此我们为实现中使用的每个大小（1到8）生成一组优化的样本。在我们的实现中，我们使用了两种不同的质量设置：![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)=5的标准质量设置和 ![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)=8的高质量设置。

考虑到我们的目标是在每四帧上循环样本，最小有效空间滤波器尺寸为3×3（用于空间滤波），因此我们的集合包含：![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)×4×3×3个样本。这样，每像素1次采样时有效使用总计36个样本，每像素2次采样时有效使用72个，每像素8次采样时有效使用288个。

在四个连续帧中的每一帧中，使用由四分之一样本组成的不同子集。此外，在每个像素中，我们使用该子集的不同九分之一。根据3×3像素块内的像素位置选择到底使用哪个子集。

我们对泊松分布生成器的直接输出进行了优化，以减少连续帧中使用的四个子集、用于不同像素的九个子集以及整个样本集的差异。此过程根据样本集在时间和空间滤波中的使用情况优化样本集，并减少视觉失真。示例示例集如图13.7所示。

![img](https://pic2.zhimg.com/v2-aae5b92a6e731aa96fea1b15e3396025_b.png)

### 13.5.2 按距离剔除灯光

即使在投射阴影光线之前，我们也可以剔除远处和低强度的光线以提高性能。为了做到这一点，我们计算每种光的范围，这是光的强度由于其衰减函数而变得可以忽略的距离。在评估可见度之前，我们将光的距离与其范围进行比较，并简单地将不起作用的光存储为零。典型的衰减函数（平方距离的倒数）永远不会达到零，因此需要修改该函数使其最终达到零，例如，抛弃低于某个阈值的线性衰减。这将减少光的范围，使剔除更加有效，同时防止一个光源在剔除后仍有贡献造成跳变。

### 13.5.3 限制总采样数

因为我们的自适应算法在半影中进行更多的采样，所以当半影覆盖屏幕的大部分时，性能可能会显著降低。对于动态场景，这可能会表现为令人不悦的卡顿。

我们提供了一种基于计算整个图像上可见性变化度量![\overline{∆v}](https://www.zhihu.com/equation?tex=%5Coverline%7B%E2%88%86v%7D)之和的在全局限制样本计数的方法(我们使用mipmaps的分层归约计算总和)。如果总和超过某个阈值，我们将逐步限制每个像素中可使用的采样数。这样的限制需要经过精心的取舍权衡。这将导致渲染质量瞬间下降，但与卡顿相比，这更可取。

### 13.5.4 在前向渲染管线中集成

我们在前向渲染管线中实现了我们的算法。与延迟渲染相比，此管线具有以下优点：更简单的透明度处理、支持更复杂的材质、硬件抗锯齿(MSAA)和更低的内存要求。

我们的实现建立在Harada等人引入的f+管道之上。[10]，它利用了一个Depth Prepass，并增加了一个灯光剔除阶段，以解决透支和许多灯光的问题。DXR使光线跟踪与现有渲染器的集成变得简单，因此在添加光线跟踪特征（如阴影）时，在材质、特殊效果等方面的大量早先工作成果得以保留。

我们的方法概述如图13.8所示。首先，我们执行深度预处理来填充没有附加Color Buffer的Depth Buffer。在Depth Prepass后，我们根据摄像机运动和Normal Buffer生成Motion Vector，这将在后期去噪时使用。Normal Buffer由深度值生成。因为它不是用于着色，而是去噪，所以这种近似法相当有效。

![img](https://pic3.zhimg.com/v2-f0cfed03754c527dc80df243d5825a02_b.png)

我们方法中使用的Buffer布局如图13.9所示。可见性缓存、可见性变化度量和采样数在每个灯光的最后四帧缓存。过滤后的可见性缓冲区和过滤后的变化度量Buffer只存储每个灯光的最后一帧。请注意，采样数和可见性变化度量被打包到同一个缓冲区中。

![img](https://pic1.zhimg.com/v2-f1faf5196c255203792da7d00971796c_b.png)

然后，我们使用光线跟踪为所有灯光生成可见性Buffer。我们使用深度值来重建世界空间位置的可见像素使用反向投影。世界空间像素位置也可以直接从G缓冲区读取（如果可用），或者通过投射主光线来评估以获得更高的精度。从这些位置，我们使用自适应采样算法向光源发射阴影光线，以评估其可见性。结果被去噪并存储在可见性缓冲区中，然后传递到最终的照明阶段。图13.10显示了我们阴影计算所使用的可见性变化度量、采样数和过滤核大小的可视化。

照明阶段使用单个光栅化过程，在此过程中计算所有场景灯光。一个光栅化的点由一个循环中的所有场景灯光照亮，并累计结果。请注意，每个灯光的可见性Buffer都是在着色之前查询的，而着色只对可见灯光进行，这提供了隐式的灯光剔除以提高性能。

## 13.6 结论

我们评估了计算硬阴影和软阴影的方法，并将其与参考用的ShadowMap实现进行了比较。我们用三个20秒动画序列的测试场景和一个移动摄像头。酒吧和度假村的场景具有相似的几何复杂度，但酒吧场景包含更大的区域灯光。早餐场景有一个明显更大的三角形规模。酒吧和早餐场景是室内场景，因此它们使用点光源，而外部度假村场景使用方向灯。对于计算软阴影，这些灯光被视为盘状面光源。我们使用Falcor框架的SM实现，它使用级联阴影映射（CSM）和指数方差阴影映射（EVSM）[13]过滤。我们使用四个CSM层作为方向灯，一个层作为点光源，最大的层使用尺寸为2048×2048的阴影图。所有测试的屏幕分辨率为1920×1080。

我们评估了四种阴影计算方法：使用阴影映射（sm hard）计算的硬阴影，使用我们的方法计算的硬阴影（rt hard），使用 ![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D)=5（rt soft sq）的光线跟踪计算的软阴影，以及使用 ![s_{max}](https://www.zhihu.com/equation?tex=s_%7Bmax%7D) =8（rt soft hq）的光线跟踪计算的软阴影。测量结果汇总在表13.1中。

![img](https://pic1.zhimg.com/v2-83c13c404d7de5105eaf74a416797f40_b.png)

###  13.6.1 Comparison with Shadow Mapping

表13.1中的测试结果表明，对于具有四个灯光的早餐和度假村场景，光线跟踪硬阴影（rt-hard）比阴影映射（sm-hard）耗时分别高出40%和60%。对于早餐场景，我们将其归因于大量的三角形。增加三角形的数量似乎比RT核心更快地减慢了阴影映射使用的光栅化管道。室外度假村场景要求生成和过滤所有四个CSM级联，从而显著延长阴影映射的执行时间。

对于酒馆场景（图13.11）和早餐场景（图13.12），只有一盏灯，阴影贴图的速度大约是硬光线跟踪阴影的两倍。这是因为只有一个CSM级联用于点光源，但它以视觉效果为代价。对于pub场景，透视失真发生在相机附近（屏幕边界中）和后面的墙上。此外，椅子投射的阴影与地面分离。试图补救这些伪影导致阴影在图像的其他部分产生锯齿。而光线跟踪的阴影不受这些失真的影响。

![img](https://pic2.zhimg.com/v2-a6ff2252e8f1d9d73a0029425bc23a41_b.png)

![img](https://pic1.zhimg.com/v2-ef309ad53e587cb9414cfa2818ee8f1c_b.png)

对于早餐场景，EVSM过滤会在桌子下产生非常柔和和不聚焦的阴影。这可能是由于该区域的阴影图分辨率不足，而阴影图分辨率由更强的过滤进行补偿。使用轻微的过滤会导致混叠失真，这更令人不悦。对于度假村场景，光线跟踪和阴影映射的视觉结果非常相似；但是，光线跟踪阴影在大多数测试中优于阴影映射。

### 13.6.2 软阴影与硬阴影对比

比较软硬光线跟踪阴影，在我们的测试中，计算软阴影需要大约2-3倍的时间。然而，这高度依赖于灯光的大小。对于设置了产生较大半影的灯光的酒馆场景，四个灯光的计算速度比同样复杂的度假场景慢40%。这是因为我们必须在更大的区域使用大量的采样数目。图13.13显示了RT Soft SQ和RT Soft HQ方法的视觉比较。请注意，对于大的早餐场景，对于rt hard方法，执行时间不会随灯光数量线性增加。这表明，对于单个光源的情况，RT核心尚未被全数使用。

![img](https://pic3.zhimg.com/v2-7a4b9f6470b3575d5c08cd033f165272_b.png)

与每像素8个样本的未优化计算相比，我们的自适应采样方法为测试场景提供了大约40–50%的组合加速。此外，由于时间累计，我们的方法获得了更好的视觉效果。

### 13.6.3 局限性

我们对所提出的方法目前有几个限制，这些限制可能在完全动态的场景中显示为失真。在当前的实现中，我们不考虑移动物体的运动矢量，这会降低重新投影的成功率，同时也会为相机和物体移动的特定组合引入错误的重新投影成功（尽管这种情况应该很少见）。

更重要的是，移动物体无法很好的被该方法处理，这可能会引入时间阴影失真。在积极方面，我们的方法使用有限的时间缓冲区（只考虑最后四帧），并且结合积极的可变性测量，它通常会强制动态半影的密集采样。另一个有问题的例子是移动光源，我们目前没有明确地解决这个问题。这种情况类似于移动物体：快速移动的光源会导致阴影发生严重变化，从而降低自适应采样的可能性，并可能导致重影瑕疵。

目前保持每帧光线追踪开销的算法相对比较简单，并且希望使用一种将变化测量直接与样本数量相关联的技术，同时尽量减少感知误差（包括阴影）。在这种情况下，在获得尽可能高质量的阴影的同时，更容易保证帧速率。

## 13.7 结论与未来工作

在本章中，我们介绍了一种在光栅化前向渲染管线中使用现代DXR API计算光线跟踪阴影的方法。提出了一种自适应阴影采样方法，该方法是基于对摄像机所见表面的能见度函数变化进行估计。我们的方法使用不同大小的灯光产生硬阴影和软阴影。我们已经评估了光采样和阴影过滤技术的各种配置，并提供了最佳结果的建议。

在视觉质量和性能方面，我们将我们的方法与最先进的阴影映射实现进行了比较。总的来说，我们得出的结论是，在支持DXR的硬件上，光线跟踪阴影具有较高的视觉质量、更简单的实现和更高的性能，使其优于阴影映射。这也会将阴影计算的负担从光栅化转移到光线跟踪硬件单元，从而为光栅化任务提供更高的性能。在专用的GPU内核上使用基于人工智能的降噪器可以在这方面提供更多帮助。

对于阴影贴图，场景设计者经常面临着通过设置技术参数（如近/远平面、阴影贴图分辨率以及与物理照明无关的偏移和半影大小）来最小化技术瑕疵的挑战。对于光线跟踪阴影，设计师仍然需要负担通过使用合理的灯光大小、范围和位置来提高阴影计算效率和抑制噪音。然而，我们相信这些参数更直观，更接近于基于物理的照明。

### 13.7.1 未来工作

我们的方法并没有明确地处理灯光的移动，这会导致快速灯光移动产生重影效果。一个正确的方法是，当光线在帧之间移动后，它不再有效时，放弃以前帧中缓存的可见性。

阴影映射不依赖于视图，常见的优化是仅在灯光或场景更改时计算阴影映射。此优化不适用于光线跟踪，因为光线跟踪可见性缓冲区需要在每次相机移动后重新计算。因此，对于很少更新阴影映射的场景，阴影映射仍然更可取。因此，对于重要光源而言，高质量的光线跟踪阴影和场景中大多数静态部分或较少的贡献灯光的阴影映射相结合是可取的。

如第13.3节所述，将使用我们的方法评估的阴影与分析直接光照相结合的改进方法，如Heitz等人介绍的方法。[11]可用于提高渲染图像的正确性。

## References

[1] Anagnostou, K. Hybrid Ray Traced Shadows and Reflections. Interplay of Light Blog, https://interplayoflight.wordpress.com/2018/07/ 04/hybrid-raytraced-shadows-and-reflections/, July 2018. 

[2] Barre-Brisebois, Colin. Hal ´ en, H. ´ PICA PICA & NVIDIA Turing. RealTime Ray Tracing Sponsored Session, SIGGRAPH, 2018. 

[3] Benty, N., Yao, K.-H., Foley, T., Kaplanyan, A. S., Lavelle, C., Wyman, C., and Vijay, A. The Falcor Rendering Framework. https: //github.com/NVIDIAGameWorks/Falcor, July 2017. 

[4] Cook, R. L., Porter, T., and Carpenter, L. Distributed Ray Tracing. Computer Graphics (SIGGRAPH) 18, 3 (July 1984), 137–145. 

[5] Crow, F. C. Shadow Algorithms for Computer Graphics. Computer Graphics (SIGGRAPH) 11, 2 (August 1977), 242–248. 

[6] Eisemann, E., Assarsson, U., Schwarz, M., Valient, M., and Wimmer, M. Efficient Real-Time Shadows. In ACM SIGGRAPH Courses (2013), pp. 18:1–18:54. 

[7] Eisemann, E., Schwarz, M., Assarsson, U., and Wimmer, M. Real-Time Shadows, first ed. A K Peters Ltd., 2011. 

[8] Engel, W. Cascaded Shadow Maps. In ShaderX5 : Advanced Rendering Techniques, W. Engel, Ed. Charles River Media, 2006, pp. 197–206. 

[9] Fernando, R. Percentage-Closer Soft Shadows. In ACM SIGGRAPH Sketches and Applications (July 2005), p. 35. 

[10] Harada, T., McKee, J., and Yang, J. C. Forward+: Bringing Deferred Lighting to the Next Level. In Eurographics Short Papers (2012), pp. 5–8. 

[11] Heitz, E., Hill, S., and McGuire, M. Combining Analytic Direct Illumination and Stochastic Shadows. In Symposium on Interactive 3D Graphics and Games (2018), pp. 2:1–2:11. 

[12] Kajiya, J. T. The Rendering Equation. Computer Graphics (SIGGRAPH) 20, 4 (August 1986), 143–150. 

[13] Lauritzen, A. T. Rendering Antialiased Shadows using Warped Variance Shadow Maps. Master’s thesis, University of Waterloo, 2008. 

[14] Marrs, A., Spjut, J., Gruen, H., Sathe, R., and McGuire, M. Adaptive Temporal Antialiasing. In Proceedings of High-Performance Graphics (2018), pp. 1:1–1:4. 

[15] Scherzer, D., Yang, L., Mattausch, O., Nehab, D., Sander, P. V., Wimmer, M., and Eisemann, E. A Survey on Temporal Coherence Methods in Real-Time Rendering. In Eurographics State of the Art Reports (2011), pp. 101–126.

[16] Schied, C., Kaplanyan, A., Wyman, C., Patney, A., Chaitanya, C. R. A., Burgess, J., Liu, S., Dachsbacher, C., Lefohn, A., and Salvi, M. Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination. In Proceedings of High-Performance Graphics (2017), pp. 2:1–2:12. 

[17] Schwarzler, M., Luksch, C., Scherzer, D., and Wimmer, M. ¨ Fast Percentage Closer Soft Shadows Using Temporal Coherence. In Symposium on Interactive 3D Graphics and Games (March 2013), pp. 79–86. 

[18] Shirley, P., and Slusallek, P. State of the Art in Interactive Ray Tracing. ACM SIGGRAPH Courses, 2006. 

[19] Story, J. Hybrid Ray Traced Shadows, https://developer.nvidia.com/ content/hybrid-ray-traced-shadows. NVIDIA Gameworks Blog, June 2015. 

[20] Whitted, T. An Improved Illumination Model for Shaded Display. Communications of the ACM 23, 6 (June 1980), 343–349. 

[21] Williams, L. Casting Curved Shadows on Curved Surfaces. Computer Graphics SIGGRAPH() 12, 3 (August 1978), 270–274.


  