## 12章 通过基于微表面阴影函数的方案解决Bump Map阴影失真问题

### 摘要

我们提出了一种方法来隐藏当使用很强的BumpMap或NormalMap来模拟微观几何时会出现的阴影突变问题。我们的方法基于微面阴影函数，简单且足够高效。我们没有渲染富有细节且昂贵的高度场阴影，而是应用了一种基于以下假设的统计解决方案----法线遵循几乎正态的随机分布。我们还为GGX提供了一个有用的近似方差度量方式，否则这一方差在数学推导上将是未被定义的。

### 12.1 介绍

Bump map在游戏实时渲染和电影渲染中都得到了广泛的应用。它在曲面上添加了高频细节，否则使用细节丰富的实际几何体或displacement mapping进行渲染会非常昂贵。此外，它还负责向曲面添加那些最终的细密纹理。

Bump map对法线方向进行扰动，该扰动不是来自几何体，而是来自纹理贴图或某些程序纹理。然而，与任何其他trick一样，它可以产生不被期望的失真，特别是图12.1中所示的阴影硬边。这是因为在对入射光阴影进行处理时，由于法线受到扰动而导致的预期平滑表面中断。当法向没有扰动时，这个问题不会出现，因为当这种情况发生时，辐照度已经降至零(不理解的读者可以去引擎里，把光源阴影关掉再打开，观察物体背光面的变化.译注)。但是，当法线被向入射光方向倾斜，则照亮区域将穿过几何体的阴影检测(不论是光栅化的Shadowmap还是RT的ShadowRay.译注)，从而将灯光的影响扩展得太远。

![img](https://pic4.zhimg.com/v2-ad73ac65afe2e463db241841bf43aa5b_b.png)

我们利用微表面理论中的阴影函数来解决这个问题。Bump Map可以看作是一个大规模的正态分布，通过对其性质的假设，我们可以使用与GGX微面分布中相同的阴影项。尽管这一假设在许多情况下都是错误的，但是阴影项在实践中仍然有效，甚至在Bump Map或Normal Map显示出非随机结构时依然如此。

### 12.2 Previous Work

据我们所知，目前还没有明确的解决办法公开发表。在Max的相关工作中，有提到如何从Bump Map计算特写镜头的细节阴影，这是基于在每个点上找到高度偏移的方式。Onoue等人将此技术扩展到曲面。虽然这些方法得到的阴影十分精确，但是需要额外的辅助和更多的查找操作。且它们不适用于高频的Bump Map----在这种情况下阴影分割线更加引人注目。

尽管如此，阴影分割线问题在几乎每个渲染引擎中都是一个问题，通常所能提供的解决方案只是调整Bump Map的强度或同时使用置换贴图。与之相比，我们的解决方案快速简单，且不需要任何额外的数据或预计算。

另一方面，自库克和托伦斯提出微面理论以来，海茨、沃尔特等人对它及其阴影项进行了广泛的研究。我们从他们的工作中获得灵感，为本文中讨论的问题找到了一个合理的解决方案。

### 12.3 方法

造成这个问题的根本原因是，法线的扰动会改变光辐照度的自然余弦衰减，使计算出的照亮区域提前到阴影区域太远。由于BumpMap对表面的改变只是想象中的，渲染器不知道任何高度信息，因此灯光将突然消失，如图12.2所示。这些缺陷，尽管是意料之中的，但可能会分散注意力，造成不必要的卡通感。艺术家们通常希望这种从光到影的过渡是平滑的。

![img](https://pic2.zhimg.com/v2-fc9c8411e4a5eea18a21e3a360565fb9_b.png)

在图12.3中，我们展示了用Bump Map模拟不存在的曲面的凹凸是如何使亮区向阴影分割线靠近的。这是因为阴影因子（如图所示）被完全忽略。在微表面理论中，这个因素被称为阴影/遮罩项，它是在[0，1]间隔中的一个值，该值是根据光和观察方向计算的，以保持BSDF的互换性。

![img](https://pic4.zhimg.com/v2-eaaba2aac54fc7239407f9679947eeab_b.png)

我们同样对Bump Map使用史密斯阴影方法。它仅仅缩小了从掠射角度获得的分散能量，阴影过度将优雅地变暗，自然地混合亮区和暗区，而不会改变其余的外观。它的推导需要知道法线分布，这对于一个任意的Bump/Normal Map来说是未知的，但是我们会假设它是随机的正态分布的。虽然这几乎永远无法反应真实情况，但是如果仅仅是为了这一阴影问题，我们将会展示它的有效性。

12.3.1 萝莉分布

我们选择GGX分布函数是因为它的简单和高效的实现。与大多数分布一样，它有一个粗糙度参数α，用以调节微表面的分布，一个较低的粗糙度表征着轻微凹凸的表面。而我们主要的问题就是如何找到一个合适的α参数。

我们毙掉了了从纹理贴图预计算这个属性的想法。因为有时它们是程序化生成的，不可预测的，我们希望避免任何预计算过程。我们的想法是根据我们在计算光照时拿到手的扰动法线值来推测α，而不依赖任何额外信息。也就是说，我们的猜测是完全局部的，没有任何来自邻近点的信息。

让我们来看看扰动后法线与真实表面几何法线的夹角的正切值。在计算该法线的令人信服的阴影项时，如图12.4所示，我们让这个正切值等于两个正态分布标准差。随后，我们可以用GGX代替它，并应用众所周知的阴影项，其中 ![θ_i](https://www.zhihu.com/equation?tex=%CE%B8_i)θ_i 是入射光方向和真实表面法向的夹角。

![img](https://pic3.zhimg.com/v2-99ca41f286e182c0b84d0515e8dca81a_b.png)

![img](https://pic1.zhimg.com/v2-0924f7f38cba95b72b4cf60976fd4f4c_b.png)

接下来就是如何从分布方差计算GGX的α的问题。GGX是基于柯西分布的，柯西分布有一个不确定的均值和方差。Conty等人在数值上发现，如果忽略它长长的尾巴来保持大部分分布质量不变， ![σ^2 = 2α^2 ](https://www.zhihu.com/equation?tex=%CF%83%5E2%20%3D%202%CE%B1%5E2%20)σ^2 = 2α^2  是GGX方差的一个很好的近似值。因此，我们选用：

![img](https://pic1.zhimg.com/v2-4ac2cf2d7e756be1e4910fb53ec2ba04_b.png)

但与此同时我们把结果限制在[0,1]区间。这一度量反映了这样一个事实：GGX示的表面粗糙度高于Beckmann，其正切值方差为 ![α^2](https://www.zhihu.com/equation?tex=%CE%B1%5E2)α^2 。等价关系大致是： ![a_{beck}=\sqrt{2}\alpha_{ggx}](https://www.zhihu.com/equation?tex=a_%7Bbeck%7D%3D%5Csqrt%7B2%7D%5Calpha_%7Bggx%7D)a_{beck}=\sqrt{2}\alpha_{ggx} 

我们通过对扰动后的GGX表面进行全面的可视化研究，验证了我们的GGX方差近似。我们使用了来自Olano等人的法线过滤抗锯齿技术以及Dupy等人的BumpMap纹理编码mipmap技术。对于每个像素，我们通过获取该像素的mipmap级别并相应地扩展GGX粗糙度来估计正态分布的方差。我们将我们的GGX方差与一个原装的Beckmann映射和一个光线跟踪高采样率的非过滤方法进行了比较。在所有场景中，我们的映射都显示出更好地保留了由扰动法线引起的感知GGX粗糙度变化，如图12.6所示。

![img](https://pic2.zhimg.com/v2-66311d84f1f0b4e494ab9387b373dccd_b.png)



### 12.3.2 阴影函数

在一个典型的微表面BSDF中，需要对光和观察方向分别计算阴影遮蔽项，以保持互换性。在我们的实现中，我们只将Bump Shadow用于光照方向，以尽可能保留原始外观，因此会稍微破坏这一属性。与无阴影阴影微表面BSDF不同，Bump mapping不会在掠射视角下产生能量峰值，因此将公式12.1应用于观察方向会使边缘变暗太多，如图12.7所示。如果这种效果造成了问题，则可以对所有非主光线(原文non-primary ray,可能指非第一根相机发出的光线.译注)使用完全的双向阴影遮蔽项。然而，根据我们的实践经验，我们没有发现任何问题，即使是双向路径积分器。

![img](https://pic1.zhimg.com/v2-0feac76377a7ca6a71ec51e9096e65ec_b.png)

我们根据入射角对入射光进行标量乘法。如果着色模型包含多个具有不同扰动法线(Bump/Height map, procedural or whatever)的BSDF，则每个BSDF将获得不同的缩放比例，并应单独计算。程序清单12.1显示了执行这一过程所需的所有代码，展示了我们方法的简单性。

![img](https://pic3.zhimg.com/v2-42f46a3269fda7ab320ceaad54f0a596_b.png)

这个建议可能看起来有点违反直觉，因为每个阴影点都会得到不同的α值。这意味着，与曲面方向对齐的凹凸法线几乎不会收到阴影(指额外的该方法计算的阴影.译注)，而发散的法线将收到明显的阴影。但事实证明，这正是解决问题所需的行为。

### 12.4 结论

我们的方法设法消除了突然的阴影突变，而对结果的其余部分影响很小。我们将重点介绍一些允许无痛集成到工业渲染器中的特性：

- 当没有法线扰动时，结果将保持不变。观察式12.2，当没有扰动，粗糙度将为0，因此没有额外的阴影。
- 细微的法线扰动将得到极低的α，因此会引起难以察觉的变化。这种情况不受失真的影响，也不需要修复。
- 只有掠射光方向受到影响。与微面模型一样，入射光在直射表面的角度上不会受到影响。

虽然我们的推导是基于一个与实际脱节的正态分布，但我们证实了，这种分布在应付结构化的扰动纹理时也能得到合理的结果，如图12.8所示。在左侧的扰动幅度较低的情况下，我们的阴影项只会对不需要校正的图像进行最小程度的更改。随着阴影突变越来越碍眼，我们的技术发挥更大的影响力，平滑了过渡区域。这种方法对剧烈法线扰动特别有用。

![img](https://pic1.zhimg.com/v2-423f9a448541e3c21411873e5d75ce5c_b.png)

### References

[1] Conty Estevez, A., and Lecocq, P. Fast Product Importance Sampling of Environment Maps. In ACM SIGGRAPH 2018 Talks (2018), pp. 69:1–69:2. 

[2] Cook, R. L., and Torrance, K. E. A Reflectance Model for Computer Graphics. ACM Transactions on Graphics 1, 1 (Jan. 1982), 7–24. 

[3] Dupuy, J., Heitz, E., Iehl, J.-C., Pierre, P., Neyret, F., and Ostromoukhov, V. Linear Efficient Antialiased Displacement and Reflectance Mapping. ACM Transactions on Graphics 32, 6 (Sept. 2013), 211:1–211:11. 

[4] Heitz, E. Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs. Journal of Computer Graphics Techniques 3, 2 (June 2014), 48–107. 

[5] Max, N. L. Horizon Mapping: Shadows for Bump-Mapped Surfaces. The Visual Computer 4, 2 (Mar 1988), 109–117.

[6] Olano, M., and Baker, D. Lean Mapping. In Symposium on Interactive 3D Graphics and Games (2010), pp. 181–188. 

[7] Onoue, K., Max, N., and Nishita, T. Real-Time Rendering of Bumpmap Shadows Taking Account of Surface Curvature. In International Conference on Cyberworlds (Nov 2004), pp. 312–318. 

[8] Walter, B., Marschner, S. R., Li, H., and Torrance, K. E. Microfacet Models for Refraction Through Rough Surfaces. In Eurographics Symposium on Rendering (2007), pp. 195–206. 