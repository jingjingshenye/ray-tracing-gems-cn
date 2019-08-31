# Importance Sampling of Many Lights on the GPU

## 简介
引入用于光线跟踪的标准化API以及硬件加速，为实时渲染中的物理光照提供了可能性。光源的重要性采样是光线传输模拟中的基本操作之一，适用于直接照明和间接照明。  
本章描述了一种包围盒层次数据结构和相关的采样方法，从而加速局部光源的重要性采样。这项工作基于最近发布的用于渲染中的光采样方法，但将它使用Microsoft DirectX Raytracing来实现实时效果。  
## 18.1 引入
一个现实场景可能会包含数十万个光源。对灯光和它们投射的阴影的精确模拟在图形学中是真实感的最重要影响因素之一。使用光栅阴影贴图的传统实渲染的应用，在实际中被限制于使用手工调整的动态光源。而光线跟踪允许更大的灵活性，因为我们在每个像素点对于不同的光源追踪阴影光线。  
从数学上讲，选择这些样本的最佳方法是，选择光源，其概率与每个光源的贡献成比例。然而，贡献在空间上是变化的，并且这取决于局部的表面特性和光源的可见性。因此，找到一个适用于所有地方的单一全局概率密度函数(PDF)具有挑战性。  
我们在本章中探讨的解决方案是使用在光源上构建的分层加速结构来指导采样[[11][11],[22][22]]。数据结构中的每个节点代表一组光源。使用的想法是从上向下遍历树，在每一层估计每组光源的贡献，并根据在每层的随机决策选择向下遍历的路径。图[18-1][18-1]演示了这些概念。
![](https://i.postimg.cc/qM6YgZ7V/ch18-001.png)
**图片 18-1.** 全部光源都按照层次结构来组织。对于一个给定的着色点$x$，我们从根开始沿层次结构向下前进。在每一层，按照概率来估计每个直接子节点关于$x$的重要性。然后，均匀采样$\xi$来决定树上的路径，并在叶子处找到要采样的光源。最后，更重要的光源有更高的概率被采样到。  

这意味着光源的选择与其贡献近似成正比，但不需要显式计算和储存PDF在每个着色点。该技术的表现最终取决于我们如何准确估计光源的贡献。实际上，一个光源或一组光源的相关性取决于它的:  
- 辐射通量:光源越强，它的贡献就越大。  
- 到着色点的距离:光源越远，它所包围的立体角越小，使得到达着色点的能量就越少。  
- 朝向:光源可能不会在所有方向上发光，也不会均匀地发光。  
- 可见性:全部被遮挡的光源没有贡献。  
- 着色点的BRDF:位于BRDF主峰方向的光源将被反射更多的能量。  

光源重要性采样的关键优点在于，它与光源的类型和数量无关，因此场景可以有比我们能够追踪的阴影光线更多的光源，并且可以无缝支持大纹理的面积光源。因为概率贡献是在运行时计算的，场景可以完全动态并具有复杂的光源设置。随着最近降噪技术的进步，这有望减少渲染的时间，同时允许更大的艺术自由和更真实的结果。  
接下来，我们将更加详细地讨论光源重要性采样,并提供一种在光源上使用包围盒层次结构(BVH)的实时实现方式。该方法使用Microsoft DirectX Raytracing(DXR)API实现，并且源代码是公开的。  
## 18.2 回顾以前的算法
随着在工业生产中，渲染向路径追踪转变[[21][21],[31][31]]，通过追踪向光源上的采样点的阴影光线来解决可见性采样。当一条阴影光线在采样点到光源的路径上没有击中任何物体，采样点被认为是可见的。通过对光源表面很多的采样点的平均，一种不错的光源近似方法被实现。随着采样数量的增多，这种近似收敛于真实。但是，当处理大量光源时，这种全面的采样方法，即使在工业生产的渲染中，也并不是一种可行的策略。  
为了处理这种有大量光源的复杂动态照明场景，大多数技术通常依赖于在灯光上建立某种形式的空间加速结构，然后通过剔除，近似或重要采样灯来加速渲染。  
### 18.2.1  实时光源剔除  
游戏引擎转变为使用大多数基于物理的材质和以物理单位确定的光源[[19][19],[23][23]]。但是，出于性能原因和由于光栅管线的限制，只有一些类似点的光源可以使用阴影映射(shadow map)被实时渲染。每个光源的花费很高，性能与光源个数呈线性关系。对于面光源，非阴影的贡献可以使用线性变换的余弦项来计算[[17][17]]，但是评估可见性的问题仍然存在。  
为了去减少需要考虑的光源数量，通常人为地去限制每个光源的影响区域，例如，通过使用一个近似的二次衰减，使得在光源在某个距离变为零。通过细心摆放和调整光源的参数，影响给定点的光源数量被限制住。  
片源着色(Tiled shading)[[2][2],[28][28]]的工作原理是将这些光源放入屏幕空间中的片源，其中片源的深度约束有效地减少了在对每个片源进行着色时所需要处理的光源数量。现代。现在的变种通过分割深度(2.5D剔除)[[15][15]，聚类着色点或光源[[29][29],[30][30]]，或使用每个片源的光源树[[27][27]]来提高剔除率。  
这些剔除方法的一个缺点是加速结构是在屏幕空间中的。另一个缺点是所需的限制光源范围会引入明显的变暗。在许多昏暗的光源有重大贡献的情况下，例如圣诞树灯或室内办公室照明时，这一点尤为明显。为了解决这个问题，Tokuyoshi和Harada[[40][40]]提出了使用随机光源范围来随机拒绝不重要的光源，而不是安排一个固定的范围。他们还将这个技术应用于在光源上使用球形包围层次结构的路径追踪中，来展示了对这个想法的验证。  
### 18.2.2 多光源的算法  
虚拟点光源(VPLs)[[20][20]]长期以来被用于近似全局照明。这个想法是去追踪从光源发出的光子，并存储VPLs到路径节点上，之后被用来近似间接光照。VPL方法被概念性地近似于对光源的重要性采样方法。光源被聚类到树中节点上，并且在遍历期间计算估计贡献。主要的差异在于，对于重要性采样，估计值被用于计算光源选择概率，而不是直接用来近似光源。  
例如，光源切割(lightcuts)[[44][44],[45][45]]通过遍历每个着色点的树并计算估计贡献的误差界限来使用数百万VPLs加速渲染。当误差足够小时，这个算法选择去使用一组VPLs直接作为光源，避免细分为更细小的组或单个VPLs。我们参考Dachsbacher等人的研究[[12][12]]，对这些和其他多光源技术进行了不错的概述。另请参阅Christensen和Jarosz[[8][8]]对全局照明算法的概述。  
### 18.2.3 光源重要性采样
在早期加速对多光源的光线最终加速中，光源被按照贡献排序，并且只有大于阈值的光源被进行阴影检测(shadow test)[[46][46]]。然后基于其可见度的统计估计值来添加剩余光的贡献。  
Shirley等人[[37][37]]描述了对于不同类型光源的重要性采样。他们将光源按照亮或暗来分类，通过比较他们的估计贡献和一个用户定义的阈值。为了从多个光源中采样，他们使用一个分层细分的八叉树，划分直到明亮的灯光数量足够小。通过评估格子边界上的大量点的贡献来估计八叉树格子的贡献。Zimmerman和Shirley[[47][47]]使用了一种空间均匀细分的方法来代替，并且在格子里包含了对可见性的估计。对于多光源的实时光线追踪，Schmittler等人[[36][36]]限制了光源的影响区域，并且使用k-d树来快速找到影响每个点的光源。Bikker在Arauna引擎中使用一种类似的方法[[5][5],[6][6]]，但是它使用了一个球形节点的BVH去收紧光源的边界。着色使用了Whitted-style的方法来评估所有贡献光源。这些方法受到偏差的影响，因为光的贡献被切断了，但如前面提到的那样，可以通过随机光源范围来缓解这个问题[[40][40]]。  
在Brigade实时光线追踪引擎中，Bikker[[6][6]]使用重新抽样重要性抽样(resampled importance sampling)[[39][39]]。基于位置无关的概率密度函数来选择第一组光源，然后在该组光源中使用BRDF和距离来更精确地估计贡献并选择一个重要的光源。在这种方法中，没有分层数据结构。   
Iray渲染系统[[22][22]]中使用一个分层光源重要性采样方案。Iray只使用三角形，并为每个三角形安排一个辐射通量(功率)值。BVH建立在三角形光源上并且按照概率来遍历，在每个节点处计算每个子树的估计贡献。系统通过将单位球体划分为少量区域并且在每个区域存储一个代表性通量值来对每个节点处的方向信息进行编码。来自BVH节点的估计辐射通量值基于到节点中心的距离来计算。  
Conty Estevez和Kulla[[11][11]]采用类似的方法进行电影渲染。他们使用宽度为4的BVH，其中还包括解析形状光源，并且对光源及其朝向在世界空间中进行聚类，朝向使用边界锥体来确定。在遍历过程中，它们基于一个均匀的随机数按照概率来选择要遍历的分支。这个随机数在每一步被重新调整到单位范围，从而保留了分层属性(在分层样本抽样中使用相同的技术[[9][9]])。为了减少较大的节点估计不良的问题，他们使用一个度量值来在遍历期间自适应地分割这些节点。我们的实时实现方法基于他们的技术，并进行了一些简化。  
## 18.3 基础 (6-11)
本章中，在深入挖掘我们实时实现方法的技术细节之前，我们首次回顾一些基于物理的光照和重要性采样的基础。  
### 18.3.1 光照积分
在物体表面上从点$X$离开，沿着观察方向$\boldsymbol{v}$的辐射度$L_o$，是发光辐射度$L_{e}$和反射辐射度$L_{r}$之和。在[[18][18]]描述的几何光学近似下为：  
$$L_{o}(X,\boldsymbol{v})=L_{e}(X,\boldsymbol{v})+L_{r}(X,\boldsymbol{v})   \ \ \ \ (1)$$   
$$L_{r}(X,\boldsymbol{v})=\int_{\Omega}f(X,\boldsymbol{v},\boldsymbol{l})L_{i}(X,\boldsymbol{l})(\boldsymbol{n}\cdot\boldsymbol{l})d\omega\ \ \ \ (2)$$  
此处的$f$是双向反射函数BRDF，$L_{i}$是烟方向$\boldsymbol{l}$的入射辐射度。接下来，当讨论一个具体的点时，我们将去掉$X$标识。同时，令$L(X\leftarrow Y)$表示从点$Y$朝向点$X$方向发射的辐射度。  
在本章中，我们主要关注$L_{i}$来自放置在场景中潜在的大的局部光源的情况。但是，该算法同时也可以结合其他采样策略，从而用于处理远距离光源（例如太阳和天空）。  
半球上的积分可以重写为光源的整个表面上的积分。立体角和表面积之间的关系如图[18-2][18-2]所示。事实上，光源上点$Y$的一小块$dA$覆盖一个立体角为  
$$d\omega=\frac{|\boldsymbol{n}_{Y}\cdot-\boldsymbol{l}|}{‖X-Y‖^2}dA\ \ \ \ (3)$$  
![](https://i.postimg.cc/x1rnwqMx/ch18-002.png)
**图片 18-2.** 光源上点$Y$的一小块$dA$覆盖微分立体角$d\omega$是它的距离$‖X-Y‖$和角度$cosθ=|\boldsymbol{n}_{Y}\cdot-\boldsymbol{l}|$的函数。  
即，通过距离，光源的法线$\boldsymbol{n}_Y$与发射光方向$-\boldsymbol{l}$之间的点积来平方反比衰减。注意，在我们的实现中，光源可以是单面或双面的光源。对于单面的光源，我们设置当$(\boldsymbol{n}_{Y}\cdot-\boldsymbol{l})\le 0$时，发出的辐射度$L(X\leftarrow Y)=0$。  
我们也需要去知道着色点$X$与光源上的点$Y$的可见性信息，正式表示为  
$$v(X\leftrightarrow Y)=\left\{
\begin{aligned}
1 &\ \ 当X与Y互相可见 \\
0 &\ \ 否则 
\end{aligned}
\right.$$  
在实践中，我们通过在方向$\boldsymbol{l}$上追踪来自$X$的阴影射线来评估$v$，其中射线的最大距离为$t_{max}=‖X-Y‖$。请注意，为了避免由于数值问题导致的自相交，需要使用epsilons来偏移光线原点并略微缩短光线。详见第6章。  
现在，假设在场景中有$m$个光源，在方程[2]中反射的辐射度可以被写作  
$$L_{r}(X,\boldsymbol{v})=\sum_{i=1}^{m}L_{r,i}(X,\boldsymbol{v})$$  
$$L_{r,i}(X,\boldsymbol{v})=\int_{\Omega}f(X,\boldsymbol{v},\boldsymbol{l})L_{i}(X\leftarrow Y)v(X\leftrightarrow Y)\max(\boldsymbol{n}_{Y}\cdot\boldsymbol{l},0)\frac{|\boldsymbol{n}_{Y}\cdot-\boldsymbol{l}|}{‖X-Y‖^2}dA_{i}$$    
也就是说，$L_{r}$是来自每个单独光源$i=\{1,\dots,m\}$的反射光的总和。请注意，我们控制(clamp)$\boldsymbol{n}\cdot\boldsymbol{l}$，因为来自背面到阴影点的光没有贡献。复杂度与光源数量$m$是呈线性关系的，当$m$很大时，这可能变得昂贵。 这引导我们进入下一个主题。  
### 18.3.2 重要性采样  
如第[18.2]节所述，有两种完全不同的方法可以降低公式5的成本。一种方法是限制光源的影响区域，从而减小$m$。另一种方法是采样一小部分光源$n\ll m$。这可以使用一致(consistent)的方式来完成，即，随着$n$的增长它收敛于基本事实(ground truth)。  
##### 18.3.2.1 蒙特卡洛方法
令$Z$为离散随机变量，权值$z\in\{1,\dots,m\}$。$Z$的值为$z$的概率可以被表示为离散型PDF，$p(z)=P(Z=z)$，其中$\sum p(z)=1$。例如，如果所有值等概率，那么$p(z)=\frac{1}{m}$。如果我们对于随机变量$Z$有一个函数$g(Z)$，它的期望为  
$$\mathbb{E}[g(Z)]=\sum_{z\in Z}g(z)p(z)$$  
即，每个可能的结果按其概率加权。现在，如果我们从$Z$中取$n$个随机样本$\{z_1,\dots,z_n\}$，我们得到了对于$\mathbb{E}[g(Z)]$的$n$-样本蒙特卡洛估计值$\widetilde{g}_n(z)$如下： 
$$\widetilde{g}_n(z)=\frac{1}{n}\sum_{j=1}^{n}g(z_j)$$  
换句话说，可以通过取函数随机样本的均值来估计期望值。我们也可以说蒙特卡罗估计$\widetilde{g}_n(Z)$，是$n$个独立且相同分布的随机变量$\{Z_1,\dots,Z_n\}$的函数值的平均值。很容易证明$\mathbb{E}[\widetilde{g}_n(Z)]=\mathbb{E}[g(Z)]$，即，这个估计给了我们正确的值。  
因为我们取随机样本，估计值$\widetilde{g}_n(Z)$是有方差的。如我们在第15章中讨论的，方差的降低与$n$呈线性关系：  
$$\text{Var}[\widetilde{g}_n(Z)]=\frac{1}{n}\text{Var}[g(Z)]$$  
这些性质表明，蒙特卡洛估计是一致的。在极限情况下，当我们有无限多个样本时，方差为零，并且它已收敛到正确的期望值。  
为了使它有用，我们可以发现几乎所有问题都可以转化为求期望。所以我们有一种基于函数的随机样本估计解的一致方法。  
##### 18.3.2.2 光源选择重要性采样
在我们的例子中，我们感兴趣的是评估从所有光源反射的光的总和(公式[5]())。该总和可以表示为期望值(参见等式[7]())，如下所示：  
$$L_{r}(X,\boldsymbol{v}=\sum_{i=1}^{m}L_{r,i}(X,\boldsymbol{v})=\sum_{i=1}^{m}\frac{L_{r,i}(X,\boldsymbol{v})}{P(Z=i)}P(Z=i)=\mathbb{E}[\frac{L_{r,Z}(X,\boldsymbol{v})}{p(Z)}]$$  
接下来的方程[8]()，对于从所有光源反射的光的蒙特卡洛估计$\widetilde{L}_{r}$为  
$$\widetilde{L}_{r}(X,\boldsymbol{v})=\frac{1}{n}\sum_{j=1}^{n}\frac{L_{r,z_j}(X,\boldsymbol{v})}{p(z_j)}$$  
也就是说，我们将随机选择的一组灯$\{z_1,\dots,z_n\}$的贡献除以选择每个灯的概率。这个估计是一致的，与我们采样的样本数量无关。但是，我们采的样本越多，估计量的方差就越小。  
注意到目前为止没有对随机变量$Z$的分布做出任何假设。唯一的要求是对于$L_{r,z}>0$的所有光源的$p(z)>0$，否则我们会冒险忽略某些光源的贡献。可以证明，当$p(z)\propto L_{r,z}(X,v)$[[32](),[38]()]时，方差最小。虽然我们不会在这里详述，但是当概率密度函数与我们正在采样的函数完全成比例时，蒙特卡罗估计中的求和变为为常数项的求和。在这种情况下，估计的方差为零。  
实际上，这是不可能实现的，因为对于给定的着色点，$L_{r,z}$是未知的，但我们应该以尽可能接近它们对着色点的相对贡献的概率来选择光源。在第[18.4]()节中，我们将研究如何计算$p(z)$。  
##### 18.3.2.3 光源采样
为了使用公式[11]()估计反射辐射，我们还需要对于随机选择的一组光估计积分$L_{r,z_j}(X,\boldsymbol{v})$。方程[6]()中的表达式是光源表面上的积分，其中涉及BRDF项和可见性项。在图形学中，解析地计算值是不切实际的。因此，我们再次使用蒙特卡洛求积分。  
在光源表面均匀采样$s$个样本$\{Y_1,\dots,Y_s\}$。对于三角形网格的光源，每个光源都是一个三角形，这意味着我们使用标准技术在三角形上均匀地拾取点[32]()(参见第[16]()章)。三角形$i$上的样本的概率密度函数为$p(Y)=\frac{1}{A_i}$，其中$A_i$是第$i$个三角形的面积。然后使用蒙特卡罗估计来评估光源上的积分  
$$\widetilde{L}_{r,i}(X,\boldsymbol{v})=\frac{A_i}{s}\sum_{k=1}^{s}f(X,\boldsymbol{v},\boldsymbol{l}_{k})L_{i}(X\leftarrow Y_{k})v(X\leftrightarrow Y_k)\max{(\boldsymbol{n}\cdot\boldsymbol{l}_{k},0)}\frac{|\boldsymbol{n}_{Y_k}\cdot -\boldsymbol{l}_{k}|}{‖X-Y_{k}‖^2}$$  
在当前实现中，$s=1$因为对于$n$个采样的光源我们只追踪一条阴影光线，并且$\boldsymbol{n}_{Y_k}=\boldsymbol{n}_i$，因为我们在估计其发射辐射度时使用光源的几何法线。出于性能原因，默认情况下禁用平滑法线和法线贴图，因为它们通常对光分布的影响可以忽略不计。  
#### 18.3.3 对光源的光线追踪
在实时应用中，常见的渲染优化是将几何表示与实际的发光形状分开。例如，灯泡可以使用点光源或小的解析球形光源表示，而可见光灯泡则绘制为更复杂的三角形网格。  
在这种情况下，重要的是光源的几何发射属性与实际光源的强度是否匹配。否则，在直接观察的光源的亮度与其投射到场景中的光强之间会存在感知差异。注意，光源通常以光度(photometric)单位的光通量(luminous flux)(流明)来指定，而区域光的发射强度以亮度($cd/m^2$)给出。因此，从光通量到亮度的精确转换需要考虑光的几何形状的表面积。在渲染之前，这些光度单位最终被转换为我们使用的辐射量（通量和辐射度）。  
另一个考虑因素是当追踪到一个光源的阴影光线时，我们不希望无意中击中代表光源的网格并将光源记为被遮挡。因此，几何表示必须对阴影射线不可见，但对其他射线可见。Microsoft DirectX Raytracing API允许通过加速结构上的InstanceMask属性和TraceRay的InstanceInclusionMask参数来控制此行为。  
对于多重重要性采样(MIS)[[41]()]，这是一种重要的方差减少技术，我们必须能够在使用由其他采样策略生成的样本的情况下评估光源采样概率。例如，如果我们使用在BRDF重要性采样后击中光源的策略，那么我们计算概率时要考虑在光源上重要性采样。基于该概率以及BRDF采样概率，可以使用例如幂次启发式的方法[[41]()]来计算样本的新权重以减小总体方差。  
MIS的一个实际考虑因素是，如果光源时由解析形状表示，我们不能使用硬件加速三角测试来搜索给定方向的光源。另一种方法是使用自定义相交着色器来计算光线和光源形状之间的交点。这在我们的示例代码中尚未实现。而我们总是使用网格本身作为光源，即每个发光三角形被视为光源。  
## 18.4 算法
接下来，我们将会描述光源重要性采样的主要步骤。描述的顺序按操作执行的频率来排序。我们从创建时期执行的预处理步骤开始，然后介绍构建和更新每帧运行一次的光源数据结构。然后，描述采样过程，对于每个光源采样点执行一次采样。  
### 18.4.1 光源的预处理
对于网格光源，我们预先计算每个三角形$i$的单个通量值$\Phi_{i}$作为预处理，类似于Iray[[22]()]。通量是三角形发出的总辐射功率。对于漫射发光源，通量是  
$$\Phi_{i}=\iint_{\Omega}L_{i}(X)(\boldsymbol{n}_{i}\cdot\omega)d\omega dA_{i}$$  
其中$L_{i}(X)$是光源表面上$X$位置的发射辐射度。对于没有纹理的光源，通量值为$\Phi_{i}=πA_{i}L_{i}$，其中$L_{i}$是材料的恒定辐射度，$A_{i}$是三角形的面积。因子$π$来自半球上积分的余弦项。在我们的经验中，这带纹理的光源比没有纹理的发射器更常见，因此，为了处理带有纹理的光源，我们在加载时的预处理阶段计算方程[13]()。  
为了求解辐射度，我们栅格化纹理空间中的所有发光三角形。三角形被缩放和旋转，使得每个像素只代表一个纹素在最大的mip等级。然后通过加载在像素着色器中的相应纹素的辐射度，以及通过原子化地累加其值来计算积分。我们还计算纹素的数量，并在最后除以该值。  
像素着色器的唯一副作用是对于每个三角形的值的缓冲区的加法是原子化的。由于DirectX 12目前缺乏浮点数的原子化，我们通过NVAPI[[26]()]使用NVIDIA扩展来进行浮点数的原子化加法。  
由于像素着色器没有绑定渲染目标(即，它是一个void的像素着色器)，我们可以在API限制内使视口任意大，而不必担心内存消耗。顶点着色器从内存中加载UV纹理坐标，并将三角形放置在纹理空间中的适当坐标处，以使其始终位于视口内。例如，如果启用了纹理包装，则在三角形被栅格化到像素坐标  
$$(x,y)=(u-\lfloor u\rfloor,v-\lfloor v\rfloor )\cdot(w,h)$$  
其中$w,h$是发光纹理的最大mip级别的尺寸。通过此变换，三角形始终处于视图中，与其(预先包装的)UV坐标的大小无关。  
我们目前使用每个像素一个样本来光栅化三角形，因此仅累积其中心被覆盖的纹素。不覆盖任何纹素的微小三角形被赋予默认非零通量以确保收敛。在像素着色器中使用分析覆盖计算的超采样或保守光栅化可用于提高计算的通量值的准确度。  
所有$\Phi_{i}=0$的三角形都不执行进一步的处理。剔除零通量的三角形是一项重要的实用优化。在若干示例场景中，大多数发光三角形位于发光纹理的黑色区域中。这并不奇怪，因为通常将发光性放在在较大的纹理上，而不是将网格分成不同材质的发光和不发光网格。  
### 18.4.2 加速结构
我们使用了与Conty Estevez和Kulla[[11]()]类似的加速结构，即从上到下使用使用分箱(bin)[[43]()]构建的包围盒层次结构[[10](),[33]()]。我们的实现使用二叉的BVH，这意味着每个节点有两个子节点。在某些情况下，更大的分支因子可能是有益的。  
在介绍构建过程中使用的现有的不同启发式算法及其其他变体之前，我们将简要介绍分箱(bin)的工作原理。  
#### 18.4.2.1 建立BVH
当从上到下构建二叉BVH时，构建树的质量和速度取决于在每个节点处三角形在左和右子节点之间的分割方式。分析所有可能的分割位置将产生最佳结果，但这也将会很慢并且不适合实时应用。  
Wald[[43]()]采用的方为将每个节点的空间均匀划分为箱，然后仅对这些箱进行分割分析。这意味着拥有的箱越多，生成的树的质量就越高，但树的构建成本也会更高。  
####  18.4.2.2 光源的方向锥
为了帮助我们考虑不同光源的方向，Conty Estevez和Kulla[[11]()]在每个节点中存储了一个光源的方向锥。该锥体由轴和两个角度$\theta_{o}$和$\theta_{e}$组成：前者限定该节点内的所有光源的法线，而后者限定光源发射方向的区域(在每个法线周围)。  
例如，单侧发光的三角形的$\theta_{o}=0$(只有一个法线)，并且$\theta_{e}=\frac{\pi}{2}$(它在整个半球上发光)。再比如，发光球体的$\theta_{o}=\pi$(它具有指向所有方向的法线)，并且$\theta_{e}=\frac{\pi}{2}$，因为在每个法线周围，光仍然仅在整个半球上发射; $\theta_{e}$通常为$\frac{\pi}{2}$，除了具有定向发光属性的灯或点光源，它们将等于聚光灯的锥角。当计算父节点的锥体时，将计算其$\theta_{o}$，使其包含其子节点的所有法线，而$\theta_{e}$为所有子节点的$\theta_{e}$的最大值。  
#### 18.4.2.3 定义分割平面
如前面我们所说的，必须计算一个轴对齐的分割平面将该节点的光源分成两个子集，每个子集一个子节点。这通常通过计算每个可能拆分的花费指标并选择花费最低的来实现。在分箱(bin)BVH中，我们测试了基于表面积的启发式(SAH)(由Goldsmith和Salmon[[14]()]引入并由MacDonald和Booth[[24]()]完善)和表面积方向启发式(SAOH)[[11]()]，以及这两种方法的不同变种。  
对于下面给出的所有变种，在构建BVH时执行的分割可以仅在最大的轴(节点的轴对齐边界框(AABB))上或在所有三个轴上完成，并且选择具有最低成本的分割。只考虑最大的轴将导致较低的构建时间，也会降低树的质量，特别是考虑到光源方向的变种。有关这些权衡的更多细节可以在第[18.5]()节中找到。  
**SAH** SAH方法关注于所得到的子的AABB表面积以及它们包含的灯光数量。如果我们将左儿子节点定义为$L=\cup_{j=0}^{i}\text{bin}_{j}$，右儿子节点为$R=\cup_{j=i+1}^{k}\text{bin}_j$，其中$k$是箱(bin)的个数且$i\in[0,k-1]$，则创建$L$和$R$作为子节点的拆分成本为  
$$cost(L,R)=\frac{n(L)a(L)+n(R)a(R)}{n(L\cup R)a(L\cup R)}$$  
其中$n(C)$和$a(C)$分别返回节点$C$的光源个数和表面积。  
**SAOH** SAOH方法基于SAH，并且包含了两个额外的权重：一个基于围绕在光源发射方向的边界锥，另一个基于由所得的节点中所有光源发射的通量。花费指标为  
$cost(L,R,s)=k_{r}(s)\frac{\Phi(L)a(L)M_{\Omega}(L)+\Phi(R)a(R)M_{\Omega}(R)}{a(L\cup R)M_{\Omega}(L\cup R)}$  
其中$s$是被分割的轴，$k_{r}(s)=\text{length}_{max}/\text{length}_{s}$被用于避免较薄的包围盒，$M_{\Omega}$是一种关于方向的量[[11]()]。  
**VH** 体积启发式volume heuristic(VH)是一种基于SAH的方法，并且使用节点$C$的AABB的体积$v(C)$代替方程[15]()中的表面积$a(C)$。  
**VOH** 体积方向启发式volume orientation heuristic(VOH)相似地将SAOH(方程[16]())中的表面积替换为体积。  
### 18.4.3 光源重要性采样
我们现在看看如何根据前一节中描述的加速结构对光源进行采样。首先，根据概率遍历光源的BVH以便选择一个光源，然后在该光源的表面上采样(如果它是面光源)。见图[18-1]()。  
#### 18.4.3.1 BVH的概率遍历
当遍历加速度结构时，我们想要选择将我们引导到对当前着色点贡献最大的光源的节点，每个光源的概率与其贡献成比例。如第[18.4.2]()节所述，贡献取决于许多参数。我们将使用近似值或每个参数的精确值，并且尝试不同的组合方式来优化质量与性能。  
**距离** 这个参数是着色点与正在考虑的节点的轴对齐包围盒(AABB)中心之间的距离。如果节点具有较小的AABB，则这有利于接近着色点的节点(and by extension lights that are close??)。然而，在BVH的第一层中，节点的AABB较大，且包含大部分场景，对于阴影点与该节点内包含的光源之间的实际距离的近似结果较差。    
**光源的通量** 节点的通量是该节点内包含的所有光源发出的通量之和。出于性能原因，在构建BVH时实际上是预先计算的;如果一些光源随着时间的推移通量值会不断变化，预先计算并不会成为问题，因为无论如何都必须重建BVH，此时通量也用于指导建造步骤。    
**光源方向** 到目前为止的选择方法都没有考虑光源的方向，这会导致给直接照射在着色点上的光源与另一个照射着色点反向的光源相同的权重。为此，Conty Estevez和Kulla[[11]()]为节点的重要性函数引入了一个附加项，保守地估计光源的法线和节点的AABB中心到阴影点的方向之间的夹角。  
**光源可见性** 为了避免考虑位于着色点水平线下方的光源，我们在每个节点的重要性函数中使用clamp过的$\boldsymbol{n}\cdot\boldsymbol{l}$项。请注意，Conty Estevez和Kulla[[11]()]使用这个clamp过的项，乘以表面的反射率，作为漫反射BRDF的近似值，这将会得到与丢弃着色点水平线下方的光源相同的效果。    
**节点重要性** 使用刚刚定义的参数，给定着色点$X$的子节点$C$的重要性函数定义为  
$$\text{importance}(X,C)=\frac{\Phi(C)|\cos(\theta_{i}^{'})|}{\Vert X-C\Vert^2}\times\begin{cases}
\cos\theta^{'} &  当\theta^{'}<\theta_{e} \\
0 &否则 
\end{cases}$$
其中$\Vert X-C\Vert$是着色点$X$与$C$的AABB中心的距离，$\theta_{i}^{'}=\max(0,\theta_{i}-\theta_{u})$，$\theta^{'}=\max(0,\theta-\theta_{o}-\theta_{u})$。角度$\theta_{e}$和$\theta_{o}$来自节点$C$的光源方向锥。角$\theta$是光源方向锥的轴和节点$C$的AABB中心到着色点$X$的方向的夹角。最后，$\theta_{i}$是入射角，$\theta_{u}$是不确定角(???)；这些都可以在图片[18-3]()中找到。  
![](https://i.postimg.cc/j5LVvvY9/ch18-003.png)  
**图片 18-3.** 从着色点$X$来看，用于计算子节点$C$的重要性的几何的描述。在图片[18-1]()中，在遍历的每一步中计算两次重要性，每个子节点一次。角度$\theta_{u}$和从$X$到AABB中心的轴表示包含整个节点的最小边包围锥，并用于计算$\theta_{i}$和$\theta$的保守下界。  
#### 18.4.3.2 随机数的使用
一个均匀随机数用于决定是选择左节点分支还是右节点分支。然后重新调整该数字并用于下一层。这种技术保留了分层属性(参见，生成分层样本[[9]()])，同时也避免了在层次结构的每一层生成新随机数的花费。重新调整随机数$\xi$以找到新的随机数$\xi^{'}$的过程如下：  
$$\xi^{'}=\begin{cases}
\frac{\xi}{p(L)} & 当\xi<p(L)时 \\
\frac{\xi-p(L)}{p(R)} & 否则    
\end{cases}$$  
其中，$p(C)$是选择节点$C$的概率，是由节点的重要性除以总重要性得到的：  
$$p(L)=\frac{\text{importance}(L)}{\text{importance}(L)+\text{importance}(R)}$$  
由于浮点精度的限制，必须注意确保有足够的随机数位。对于具有大量光源的场景，可以交替使用两个或更多个随机数或使用更高的精度。  
#### 18.4.3.3 对叶子结点的采样
在遍历结束时，选择了包含一定数量光源的叶节点。要确定要采样的三角形，我们可以等概率选择一个存储在叶节点中的三角形，或者使用类似于在遍历期间用于计算节点重要性的方法。对于重要性采样，我们考虑到达三角形的最近距离和三角形的最大$\boldsymbol{n}\cdot\boldsymbol{l}$界限；考虑包括三角形的通量及其对着色点的方向可可以进一步提高效果。目前，每个叶节点最多存储10个三角形。  
#### 18.4.3.4 对光源的采样
在通过树遍历选择光源后，需要在该光源上生成光样本。我们使用Shirley等人提出的采样技术[[37]()]用于在不同类型的光源的表面上均匀地产生光源样本。  
## 18.5 结果
接下来我们演示在具有不同数量光源的多个场景下该算法的结果，其中我们测量误差降低的速率，构建BVH所花费的时间以及渲染的时间。  
渲染是通过首先使用DirectX 12在G-buffer中栅格化场景，然后对于全屏的每个像素使用一条阴影射线进行光线追踪从而进行光源采样，最后在没有移动的情况下累积该帧的结果来完成的。所有结果均在NVIDIA GeForce RTX 2080 Ti和3.60 GHz的Intel Xeon E5-1650上测量，场景的分辨率为1920×1080像素。对于本章中显示的所有结果，没有计算间接光照，而是使用该算法来改进直接光照的计算。  
我们在测试中使用以下场景，如图[18-4]()所示：  
**Sun Temple:** 这个场景有606,376个三角形，其中有67,374个发光纹理;但是，在第[18.4.1]()节描述的纹理预积分处理之后，只剩下1,095个发光三角形。整个场景由纹理火坑照亮; 图[18-4]()所示场景的一部分仅由位于右侧的两个火坑以及位于摄像机后面的其他小火坑照亮。场景完全是漫反射(diffuse)的。  
**Bistro:** 这个Bistro场景已被修改，使许多不同光源的网格实际上是发光的。在总共2,829,226个三角形中，总共有20,638个发光纹理三角形。总体而言，光源主要由小型灯泡组成，增加了几十个小型聚光灯和一些发光的商店标志。除了小酒馆的窗户和Vespa摩托车之外，场景大多是漫反射(diffuse)的。  
**Paragon Battlegrounds:** 这个场景由三个不同的部分组成，其中我们只使用两个部分：Dawn(PBG-D)和Ruins(PBG-R)。两者都包括位于地面的大型发光面灯光，以及雕刻在岩石上的符文或炮塔上的小灯等小型灯光。除了树木以外，大多数材料都是镜面(specular)的。PBG-D具有90,535个发光纹理三角形，其中53,210个在纹理预处理后留下;整个场景由2,467,759个三角形组成(包括发光三角形)。相比之下，PBG-R具有389,708个发光纹理三角形，其中199,830个在纹理预处理后留下; 整个场景由5,672,788个三角形组成(包括发光三角形)。  
![](https://i.postimg.cc/Kj71ftT6/ch18-004.png)  
**图片18-4.** 用于测试的所有场景的视图。  
需要注意的是，虽然所有这些场景当前都是静态的，但我们的方法通过重建每帧的光源加速结构可以支持动态场景。类似于DXR允许重新加载加速结构，而不改变其拓扑结构，如果灯光在帧之间没有明显的移动，我们可以选择仅更新预构建树中的节点。  
我们对本节中使用的某些方法使用不同的缩写。以"BVH_"开头的方法将遍历BVH层次结构以选择三角形。"BVH_"之后的后缀是指在遍历期间使用的信息："D"表示视点与节点中心之间的距离，"F"表示节点中包含的通量，"B"表示$\bold{n}\cdot\bold{l}$的范围，最后"O"表示节点方向锥。Uniform方法使用MIS[[41]()]将通过采样BRDF获得的样本与通过以均匀概率从场景中的所有发光三角形中随机选择发光三角形而获得的样本组合。  
当采用MIS[[41]()]时，我们使用幂指数为$2$的幂启发式。在采样BRDF和采样光源之间平均分配。  
### 18.5.1 性能
#### 18.5.1.1 加速结构的构建
使用SAH建立BVH，仅在最大的轴上有16个箱，在Sun Temple上需要大约2.3毫秒，Bistro需要26毫秒，在Paragon Battlegrounds上需要280毫秒。 请注意，BVH构建器的当前实现是基于CPU的，是单线程的，并且不使用向量操作。  
在每个步骤沿所有三个轴来分箱大约慢2倍，这是由于具有三倍的分割候选方案，但是得到的树在运行时可能不会表现得更好。此处显示的时间使用每轴16个箱的默认设置。减少该数量使得构建更快，例如，4个箱子大约快2倍，但质量再次受损。对于剩下的测量，我们使用了最高质量的设置，因为我们希望一旦将代码移植到GPU并用于具有数万个光源的类似游戏的场景中时，树的构建不会成为问题。  
SAOH的构建时间比SAH长3倍。差异主要是由于额外的光锥计算。我们在所有灯光上迭代一次以计算锥体方向，并第二次迭代计算角度边界。使用近似方法或自下而上计算边界可以加快速度。  
使用体积而不是表面积不会使构建的性能发生任何变化。    
#### 18.5.1.2 每帧的渲染时间
接下来我们测量渲染时间，使用不同的启发式方法构建树，并且打开所有的采样选项。见表格[18-1]()。与构建性能类似，使用基于体积的度量而不是表面区域不会显着影响渲染时间(通常在基于表面区域的时间的0.2毫秒内)。沿所有三个轴或仅最大轴的分组也不会对渲染时间产生显着影响(彼此相差0.5毫秒)。  
![](https://i.postimg.cc/CxSQc03X/ch18-005.png)  
**表格 18-1.** 每帧以毫秒为单位的渲染时间，每个像素有四条阴影射线，测量超过1,000帧，并使用不同构建参数的SAH和SAOH启发式算法。BVH_DFBO方法与MIS一起使用，16个箱用于分箱，每个叶节点最多存储一个三角形。  
当测试每个叶节点中保存不同最大三角形数量(1,2,4,8和10)时，发现渲染时间在彼此相差在的5％之内，其中1和10是最快的。两个场景的结果可以在图片[18-5]()中找到，在其他场景中也观察到类似的表现。计算每个三角形的重要性会增加明显的开销。相反，每个叶节点存储更多三角形将导致树高更低，因此更快地遍历。应当注意，叶节点的物理大小没有改变(即，它总是被设置为接受多达10个三角形ID)，只有BVH构建器被允许放入叶节点的量改变了。此外，对于这些测试，关注与整体的构建成本，而不是一个叶节点构建成本。  
![](https://i.postimg.cc/6qCMhbYB/ch18-006.png)  
**图片 18-5.** 对于Bistro(view 1))左)和PBG-R(右)每帧的渲染时间(以毫秒为单位)。每个叶节点的不同最大三角形数量，对叶节点内的三角形选择是否执行重要采样。在所有情况下，BVH使用SAOH沿所有三个轴构建16个箱，并且使用BVH_DFBO来遍历。  
使用SAH或SAOH的渲染时间总体相似，但是在光源上使用BVH以及考虑使用哪个重要性函数确实会产生显著影响，使用BVH_DFBO会比均匀采样慢2到3倍。如图18-6所示。这归结为从BVH获取节点所需的额外带宽以及在额外地计算$\bold{n}\cdot\bold{l}$的范围(bound)和基于方向锥的权重。这个额外花费可以通过压缩BVH节点来降低(例如，使用16位浮点数代替32位浮点数)；当前节点的内部节点为64字节，外部节点为96字节。  
![](https://i.postimg.cc/pLKnLbyW/ch18-007.png)    
**图片 18-6.** 比较在Bistro(view 1)(左)和PBG-R(右)两个场景中每一帧以毫秒为单位的渲染时间，比较使用不同的遍历方法和直接BRDF采样。所有方法使用每个像素4样本，并且基于BVH的方法使用在三个轴上取16个箱。   
### 18.5.2 图像质量
#### 18.5.2.1 构建选项
总体而言，使用体积的变种的表现比使用表面积的对应方法差，并且使用16个箱的方法比仅使用4个箱的对应方法表现更好。至于应该考虑使用多少轴来定义最佳分割，考虑到在大多数情况下，相比于仅使用最大轴的方法，使用三个轴可以得到均方误差更低的结果，但并非总是如此。最后，SAOH变种通常优于或至少与其SAH的对应方法相当。这可能在很大程度上取决于它们在BVH的顶部如何形成节点：当这些节点包含场景中的大部分光，它们对于它们包含的发光表面的空间和方向的近似较差。  
这可以在图片[18-7]()中看到，在药房商店标志周围的区域(由红色箭头指向)，例如在A点(由白色箭头指向)。当使用SAH时，A点比红色节点更接近绿色节点，导致选择绿色节点的可能性更高，因此即使绿灯很重要，也会错过十字标志发出的绿光，这个可以在图[18-4]()的Bistro(view 3)中看到。相反，对于SAOH，A点很有可能选择包含绿光的节点，从而提高该区域的收敛速度。然而，由于类似的原因，有可能找到某个区域使得SAH有比SAOH更好的结果。   
![](https://i.postimg.cc/WpFkDJHb/ch18-008.png)  
**图片 18-7.** 对使用SAH(左)和SAOH(右)构建BVH的第二层可视化;左子节点的AABB是绿色的，右子节点的AABB是紫色的。在这两种情况下，都使用了16个箱，并考虑了三个轴。  
#### 18.5.2.2 每个叶节点的三角形数量
随着更多三角形被存储在叶节点中，当均匀采样三角形时，质量将会下降，因为它将比在树中遍历的过程更差。与等均匀采样相比，使用重要性采样可降低质量下降，但它仍然比只使用树更糟糕。Bistro(view 3)的结果可以在图[18-8]()的右侧看到。  
![](https://i.postimg.cc/d18cX60V/ch18-009.png)    
**图片 18-8.** 在Bistro(view 3)中比较不同遍历方法的均方误差(MSE)结果，与采样BRDF以获得光采样方向(左)及对于不同的最大三角形数量对于BVH_DFBO的影响(右)。所有方法使用每个像素4样本，并且基于BVH的方法使用在三个轴上取16个箱。  
#### 18.5.2.3 采样方法
在图[18-9]()中，我们可以看到在使用和组合不同的采样策略时，渲染Bistro(view 2)场景得到的结果。  
![](https://i.postimg.cc/5ygd9tLc/ch18-010.png)  
**图片 18-9.** 使用第[18.4.3]()节中定义的不同采样策略，每个像素4样本(SPP)(左)和16SPP(右)的结果。所有基于BVH的方法都使用由SAOH构建的BVH，沿所有轴评估16个箱。采样时使用MIS：一半样本采样BRDF，一半采样光源加速结构。  
正如预期的那样，使用光源采样大大优于BRDF采样方法，因为能够在每个像素处找到一些有效的光路。使用距离作为重要性的BVH可以获取附近光源的贡献，正如小酒馆门两侧放置的两个白色光源所见，门面或窗户上放置不同的光源。  
当在遍历期间考虑光源的通量时，我们可以观察到从大部分蓝色(来自最接近地面的悬挂小的蓝色光源)到来自不同路灯的更黄色调的偏移，这些灯虽然位于更远处但是光强更大。  
除了Vespa上的反射(主要在16SPP图像上可见)之外，使用$\bold{n}\cdot\bold{l}$约束在这个场景中没有太大差别，但是效果在其他场景中可能更加明显。图片[18-10]()显示了Sun Temple的一个例子。在这个场景里，使用$\bold{n}\cdot\bold{l}$约束导致右边柱子的后面接收更多的光，并且可以与它在附近的墙壁上投下的阴影区分开来，以及雕像附近的建筑细节变得可见。   
![](https://i.postimg.cc/gJQxpR17/ch18-011.png)  
**图片 18-10.** 不使用$\bold{n}\cdot\bold{l}$约束(左)与使用(右)相比时的视觉结果。两个图像都使用8SPP(4个BRDF样本和4个光源样本)和使用三个轴分组的BVH，其中16个分箱使用SAH，并且都考虑了光源的距离和通量。  
即使没有使用SAOH，方向锥对最终图像的影响仍然很小;例如，与不使用方向锥相比，图[18-9]()中的面(在街道的尽头和图像的右下角)噪声较小。  
加速结构的使用显着提高了渲染质量，如图[18-8]()所示，与均匀采样方法相比，MSE得分提高了4到6倍，即使只考虑节点距离节点作为重要性也是这样。结合通量，$\bold{n}\cdot\bold{l}$约束和方向锥进一步提高了2倍。  
## 18.6 结论 (26-26)
我们提出了一种分层数据结构和采样方法，以加速实时光线跟踪中的光源重要性采样，类似于离线渲染中使用的[[11](),[22]()]。我们已经利用硬件加速的光线跟踪来探索GPU上采样性能。我们还使用不同的启发式构建方法呈现了结果。我们希望这项工作能够激发更多未来在游戏引擎研究方面的工作，以便采用更好的采样策略。   
虽然本章的重点是采样问题，但应该注意的是，任何基于采样的方法通常都需要与某种形式的降噪滤波器配对以消除残留噪声，我们推荐读者一些最近的基于高级双边核(bilateral kernel)的实时方法[[25](),[34](),[35]()]作为一个合适的起点。基于深度学习的方法[[3](),[7](),[42]()]也展示出巨大的希望。有关传统技术的概述，请参阅Zwicker等人的调查[[48]()]。  
对于采样，有许多值得改进的途径。在当前的实现中，我们使用$\bold{n}\cdot\bold{l}$来剔除在水平线下的光源。结合BRDF和可见性信息对于在树中的遍历期间提升采样效果是非常有帮助的。实际上，出于性能原因，我们希望将BVH构建代码移至GPU。这也对于支持动态或蒙皮几何体上的光源很重要。  
## 致谢
感谢Nicholas Hull和Kate Anderson创建测试场景。Sum Temple[[13]()]和Paragon Battlegrounds场景基于Epic Games的慷慨贡献。Bistro场景基于亚马逊Lumberyard的慷慨贡献[[1]()]。感谢Benty等人[[4]()]为创建Falcor渲染研究框架，以及He等人[[16]()]和Jonathan Small为Falcor使用的Slang着色器编译器的贡献。我们还要感谢Pierre Moreau在隆德大学的教授Michael Doggett。最后，感谢Aaron Lefohn和NVIDIA Research支持这项工作。   

## 参考

[18.2]:18.2
[18-1]:18-1
[18-2]:18-2
[1]:1  
[2]:2  
[3]:3  
[4]:4  
[5]:5  
[6]:6  
[7]:7  
[8]:8  
[9]:9  
[10]:10  
[11]:11  
[12]:12  
[13]:13  
[14]:14  
[15]:15  
[16]:16  
[17]:17  
[18]:18  
[19]:19  
[20]:20  
[21]:21  
[22]:22  
[23]:23  
[24]:24  
[25]:25  
[26]:26  
[27]:27  
[28]:28  
[29]:29  
[30]:30  
[31]:31  
[32]:32  
[33]:33  
[34]:34  
[35]:35  
[36]:36  
[37]:37  
[38]:38  
[39]:39  
[40]:40  
[41]:41  
[42]:42  
[43]:43  
[44]:44  
[45]:45  
[46]:46  
[47]:47  
[48]:48  


[1]  Amazon Lumberyard. Amazon Lumberyard Bistro, Open Research Content Archive (ORCA).  
http://developer.nvidia.com/orca/amazon-lumberyard-bistro, July 2017.  
[2]  Andersson, J. Parallel Graphics in Frostbite—Current & Future. Beyond Programmable Shading,SIGGRAPH Courses, 2009.  
[3]  Bako, S., Vogels, T., McWilliams, B., Meyer, M., Novák, J., Harvill, A., Sen, P., DeRose, T., and Rousselle, F. Kernel-Predicting Convolutional Networks for Denoising Monte Carlo Renderings.  
ACM Transactions on Graphics 36, 4 (2017), 97:1–97:14.  
[4]  Benty, N., yao, K.-H., Foley, T., Kaplanyan, A. S., Lavelle, C., Wyman, C., and Vijay, A. The Falcor Rendering Framework. https://github.com/NVIDIAGameWorks/Falcor, July 2017.  
[5]  Bikker, J. Real-Time Ray Tracing Through the Eyes of a Game Developer. In IEEE Symposium on Interactive Ray Tracing (2007), pp. 1–10.  
[6]  Bikker, J. Ray Tracing in Real-Time Games. PhD thesis, Delft University, 2012.  
[7]  Chaitanya, C. R. A., Kaplanyan, A. S., Schied, C., Salvi, M., Lefohn, A., Nowrouzezahrai, D., and Aila, T. Interactive Reconstruction of Monte Carlo Image Sequences Using a Recurrent Denoising Autoencoder. ACM Transactions on Graphics 36, 4 (2017), 98:1–98:12.  
[8]  Christensen, P. H., and Jarosz, W. The Path to Path-Traced Movies. Foundations and Trends in Computer Graphics and Vision 10, 2 (2016), 103–175.  
[9]  Clarberg, P., Jarosz, W., Akenine-Möller, T., and Jensen, H. W. Wavelet Importance Sampling: Efficiently Evaluating Products of Complex Functions. ACM Transactions on Graphics 24, 3 (2005), 1166–1175.  
[10]  Clark, J. H. Hierarchical Geometric Models for Visibility Surface Algorithms. Communications of the ACM 19, 10 (1976), 547–554.  
[11]  Conty Estevez, A., and Kulla, C. Importance Sampling of Many Lights with Adaptive Tree Splitting. Proceedings of the ACM on Computer Graphics and Interactive Techniques 1, 2 (2018), 25:1–25:17.  
[12]  Dachsbacher, C., Křivánek, J., Hašan, M., Arbree, A., Walter, B., and Novák, J. Scalable Realistic Rendering with Many-Light Methods. Computer Graphics Forum 33, 1 (2014), 88–104.  
[13]  Epic Games. Unreal Engine Sun Temple, Open Research Content Archive (ORCA). http://developer.nvidia.com/orca/epic-games-sun-temple, October 2017.  
[14]  Goldsmith, J., and Salmon, J. Automatic Creation of Object Hierarchies for Ray Tracing. IEEE Computer Graphics and Applications 7, 5 (1987), 14–20.  
[15]  Harada, T. A 2.5D Culling for Forward+. In SIGGRAPH Asia 2012 Technical Briefs (2012), pp. 18:1–18:4.  
[16]  He, y., Fatahalian, K., and Foley, T. Slang: Language Mechanisms for Extensible Real-Time Shading Systems. ACM Transactions on Graphics 37, 4 (2018), 141:1–141:13.  
[17]  Heitz, E., Dupuy, J., Hill, S., and Neubelt, D. Real-Time Polygonal-Light Shading with Linearly Transformed Cosines. ACM Transactions on Graphics 35, 4 (2016), 41:1–41:8.  
[18]  Kajiya, J. T. The Rendering Equation. Computer Graphics (SIGGRAPH) (1986), 143–150.  
[19]  Karis, B. Real Shading in Unreal Engine 4. Physically Based Shading in Theory and Practice, SIGGRAPH Courses, August 2013.  
[20]  Keller, A. Instant Radiosity. In Proceedings of SIGGRAPH (1997), pp. 49–56.  
[21]  Keller, A., Fascione, L., Fajardo, M., Georgiev, I., Christensen, P., Hanika, J., Eisenacher, C., and Nichols, G. The Path Tracing Revolution in the Movie Industry. In ACM SIGGRAPH Courses (2015), pp. 24:1–24:7.   
[22]  Keller, A., Wächter, C., Raab, M., Seibert, D., van Antwerpen, D., Korndörfer, J., and Kettner, L. The Iray Light Transport Simulation and Rendering System. arXiv, https://arxiv.org/abs/1705.01263, 2017.  
[23]  Lagarde, S., and de Rousiers, C. Moving Frostbite to Physically Based Rendering 3.0. Physically Based Shading in Theory and Practice, SIGGRAPH Courses, 2014.  
[24]  MacDonald, J. D., and Booth, K. S. Heuristics for Ray Tracing Using Space Subdivision. The Visual Computer 6, 3 (1990), 153–166.  
[25]  Mara, M., McGuire, M., Bitterli, B., and Jarosz, W. An Efficient Denoising Algorithm for Global Illumination. In Proceedings of High-Performance Graphics (2017), pp. 3:1–3:7.  
[26]  NVIDIA. NVAPI, 2018. https://developer.nvidia.com/nvapi.  
[27]  O’Donnell, y., and Chajdas, M. G. Tiled Light Trees. In Symposium on Interactive 3D Graphics and Games (2017), pp. 1:1–1:7.  
[28]  Olsson, O., and Assarsson, U. Tiled Shading. Journal of Graphics, GPU, and Game Tools 15, 4 (2011), 235–251.  
[29]  Olsson, O., Billeter, M., and Assarsson, U. Clustered Deferred and Forward Shading. In Proceedings of High-Performance Graphics (2012), pp. 87–96.  
[30]  Persson, E., and Olsson, O. Practical Clustered Deferred and Forward Shading. Advances in Real-Time Rendering in Games, SIGGRAPH Courses, 2013.  
[31]  Pharr, M. Guest Editor’s Introduction: Special Issue on Production Rendering. ACM Transactions on Graphics 37, 3 (2018), 28:1–28:4.  
[32]  Pharr, M., Jakob, W., and Humphreys, G. Physically Based Rendering: From Theory to Implementation, third ed. Morgan Kaufmann, 2016.   
[33]  Rubin, S. M., and Whitted, T. A 3-Dimensional Representation for Fast Rendering of Complex Scenes. Computer Graphics (SIGGRAPH) 14, 3 (1980), 110–116.  
[34]  Schied, C., Kaplanyan, A., Wyman, C., Patney, A., Chaitanya, C. R. A., Burgess, J., Liu, S., Dachsbacher, C., Lefohn, A., and Salvi, M. Spatiotemporal Variance- Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination. In Proceedings of High-Performance Graphics(2017), pp. 2:1–2:12.  
[35]  Schied, C., Peters, C., and Dachsbacher, C. Gradient Estimation for Real-Time Adaptive Temporal Filtering. Proceedings of the ACM on Computer Graphics and Interactive Techniques 1, 2 (2018), 24:1–24:16.  
[36]  Schmittler, J., Pohl, D., Dahmen, T., Vogelgesang, C., and Slusallek, P. Realtime Ray Tracing for Current and Future Games. In ACM SIGGRAPH Courses (2005), pp. 23:1–23:5.  
[37]  Shirley, P., Wang, C., and Zimmerman, K. Monte Carlo Techniques for Direct Lighting Calculations. ACM Transactions on Graphics 15, 1 (1996), 1–36.  
[38]  Sobol, I. M. A Primer for the Monte Carlo Method. CRC Press, 1994.  
[39]  Talbot, J. F., Cline, D., and Egbert, P. Importance Resampling for Global Illumination. In Rendering Techniques (2005), pp. 139–146.   
[40]  Tokuyoshi, y., and Harada, T. Stochastic Light Culling. Journal of Computer Graphics Techniques 5, 1 (2016), 35–60.  
[41]  Veach, E., and Guibas, L. J. Optimally Combining Sampling Techniques for Monte Carlo Rendering. In Proceedings of SIGGRAPH (1995), pp. 419–428.  
[42]  Vogels, T., Rousselle, F., McWilliams, B., Röthlin, G., Harvill, A., Adler, D., Meyer, M., and Novák, J. Denoising with Kernel Prediction and Asymmetric Loss Functions. ACM Transactions on Graphics 37, 4 (2018), 124:1–124:15.  
[43]  Wald, I. On Fast Construction of SAH-Based Bounding Volume Hierarchies. In IEEE Symposium on Interactive Ray Tracing (2007), pp. 33–40.  
[44]  Walter, B., Arbree, A., Bala, K., and Greenberg, D. P. Multidimensional Lightcuts. ACM Transactions on Graphics 25, 3 (2006), 1081–1088.  
[45]  Walter, B., Fernandez, S., Arbree, A., Bala, K., Donikian, M., and Greenberg, D. P. Lightcuts: A Scalable Approach to Illumination. ACM Transactions on Graphics 24, 3 (2005), 1098–1107.  
[46]  Ward, G. J. Adaptive Shadow Testing for Ray Tracing. In Eurographics Workshop on Rendering (1991), pp. 11–20.  
[47]  Zimmerman, K., and Shirley, P. A Two-Pass Solution to the Rendering Equation with a Source Visibility Preprocess. In Rendering Techniques (1995), pp. 284–295.  
[48]  Zwicker, M., Jarosz, W., Lehtinen, J., Moon, B., Ramamoorthi, R., Rousselle, F., Sen, P., Soler, C., and yoon, S.-E. Recent Advances in Adaptive Sampling and Reconstruction for Monte Carlo Rendering. Computer Graphics Forum 34, 2 (2015), 667–681.  