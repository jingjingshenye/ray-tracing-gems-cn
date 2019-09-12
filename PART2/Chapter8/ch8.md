
# CHAPTER 8 有趣的面片：计算光线/双线性面片相交的几何尝试


## 摘要
我们通过简单的几何构建来计算光线和非平面双线性面片之间的交点。
新算法将现有算法的性能提高了6倍以上，并且相比使用两个三角形近似面片的算法更快。

## 8.1 简介与当前技术 

计算机图形学力求以其丰富的形状和颜色渲染现实世界。通常使用曲面细分以充分利用现代GPU的性能。当前两种主要的渲染模式：光栅化与光线追踪，都支持硬件优化的三角形图元。但是曲面细分的缺点在于需要大量内存占用来精确表达复杂形状。 

内容制作工具处于对简单性和表现力的需求而倾向于使用更高阶的曲面。这些曲面可以直接在DirectX 11硬件管线中进行细分与光栅化[7,11]。至今为止，现代GPU并不支持非平面图元的光线追踪。

我们重新审视对于高阶图元的光线追踪，试图在三角形的简单性与诸如细分曲面[3,16],NURBS[1]和Bézier曲面[2]等平滑形状之间找到平衡点。

通常，我们用三阶（或更高阶）的表达式来生成法线连续的平滑曲面。Peter[21]提出了一个通过二次和三次面片共同建模光滑曲面的方法。对于高度场，可以通过将每个三角形细分为24个小三角形来实现对任意三角形mesh的$C^{1}$二次插值。插值给定定点的位置与导数需要更多的控制点。如图8-1所示，对于一个仅有二次或分段线性面片组成的曲面，通过插值顶点法线，可以使用Phong着色法[22]实现光滑曲面的渲染。对于此模型，我们在接下来的部分中将要提到相交计算算法相较于Optix系统[20]中的优化后的光线/三角形相交计算算法快7%。

图8-1. 石像鬼模型的平坦着色和Phong着色[9]，模型包含21418个面片，其中33个面片是完全平整的。

Vlachos等人[26]引入了一种曲面点-法线(PN)三角形，它只使用三个顶点和三个顶点的法线来创建三次Bézier 面片. 对于这种曲面，通过二次插值渲染法线以模拟光滑光照。我们可以通过局部插值来将PN三角形转化成$G^{1}$连续曲面。

Boubekeur和Alexa[6]也试图使用完全的局部表达法，并提出了Phong曲面细分的方法。他们论文的基本思想是将几何体充分膨胀以避免出现刻面。

以上这些技术，使用参数域中的采样，都非常适合光栅化。如果选择光线追踪，计算光线与这些曲面的交点需要求解非线性方程。非线性方程通常需要通过迭代来求解[2,13].

三角形由其三个顶点定义。或许最简单的通过插值四个给定点$Q_{ij}$形成的，并且可以计算单步光线相交的曲面面片是双线性面片，表达式为:
$$
    Q(u,v) = (1-u)(1-v)Q_{00} + (1-u)vQ_{01} + u(1-v)Q_{10} + uvQ_{11}
$$

这样的双变量曲面在$\{u,v\} = \{i,j\}$时经过四个角点。如图8-4和图8-5所示，该曲面是由$u=const$和$v=const$形成的双重直纹曲面。当四个角点处于同一平面时，可以通过将四边形分成两个三角形来找到一个相交点。Lagae和Dutré[15]提出了更有效的算法。

对于非平面曲面，可能与光线$R(t) = o + t \hat{d}$有两个相交点。光线由起点$O$和单位方向$\hat{d}$定义。当前最好的算法由Ramsey等人[24]提出，该算法通过代数方法求解三个二次方程$R(t) = Q(u,v)$的系统。

在迭代方法中，可以通过增加迭代次数减少误差。在直接方法中，并没有对应的保证，二次方程可能会导致无边界的误差。讽刺的是，对于更平整的面片来说，出现重大误差的可能性会更大，尤其是当面片距离较远时。为了解决这个问题，Ramsey等人使用的是双精度浮点数(double)。我们将他们的算法以单精度浮点数(float)实现后发现在某些角度会产生明显误差，如图8-2所示。

图8-2 左侧两图：使用Ramesy等人[24]的算法渲染的具有曲面四边形面的立方体和菱形十二面体

找到光线和三角形的交点是一个相对简单的问题。该问题可以通过考虑基本几何结构(光线/平面交点，线之间的距离，三角形的面积，四面体体积等）简化。我们通过利用直纹曲面的特性(等式1)，将相似方法应用于光线/面片的相交问题。
虽然Hanrahan[11]提出了相似的方法，但该方法仅适用于平面情况。


## 8.1.1 性能测试
为了便于记忆，我们将我们的技术命名为GARP(Geometric Approach to Ray/bilinear Patches intersection的缩写)(计算光线/双线性面片相交的几何尝试)。这项技术相较于Ramsey等人[24]的使用单精度浮点数算法性能提升了大约两倍。由于光线追踪速度受到加速结构遍历和渲染的显著影响，真实的GARP算法的性能甚至高于此。

为了更好理解这些问题，我们创建了一个单面片模型并执行了多个相交测试，以消除遍历和渲染对于性能的影响。实验结果表明，GARP算法相较于Ramsey的单精度算法要快6.5倍。

实际上，GARP比在渲染时用两个三角形近似四边形的算法更快（它会导致渲染结果稍有不同）。我们还测量了在预处理阶段将四边形分成三角形然后构建BVH(Bounding Volume Hierarchy)的算法的性能。有趣的是，这种方法比其他两种版本要慢，GARP和实时三角形近似算法速度相似。我们根据Walter等人[27]的文章推测通过四边形表示几何体，相较于曲面细分的模型，可以起到更有效的凝聚聚类的作用。

参数化表达曲面的一个优点是曲面是从二维参数空间$\{u,v\} \in [0,1] \times [0,1]$到三维形状的映射。使用光栅化渲染的程序可以直接在二维参数域内采样。在光线追踪算法中，一旦找到相交点，可以判断相交点的$u$和$v$是否在$[0,1]$范围内来仅保留有效的相交点。

如果使用隐式曲面$f(x,y,z) = 0$作为渲染图元，则必须裁剪拼合不同面片来形成复合曲面。对于边缘式线段的双线性面片，这种裁剪相当简单。Stroll等人[25]提出了一种将双线性面片转换为二次隐式曲面的方法。我们并没有直接将该方法与GARP相比较，但是注意他们的方法需要将四个相交点通过四面体$\{Q_{00}, Q_{01}, Q_{10}, Q_{11}\}$的正面三角形来裁剪。GARP的性能比仅使用两个光线/三角形相交的算法更快。我们通过考虑双线性面片的特殊性质（其为规则曲面）来达到这个性能。另一方面，隐式二次曲线在自然中更为常见，并且可以表达圆柱体和球体。

## 8.1.2 网格四边形化
一个很重要的问题在于如何将一个三角形mesh转换为用四边形表示。我们测试了三个系统：

1. Blender
2. 实时场对齐网格方法，Jakob等人[12]
3. 通过Morse-参数化混合系统的四边形化方法，Fang等人[9]

只有最后一个系统可以创建完全四边形化的网格。有两种处理三角形/四边形混合网格的方法：将每个三角形视为退化四边形，或者对于三角形使用光线/三角形相交算法。我们选择使用前一种方法，因为它由于减少了额外的分支，稍快一些。将$Q_{11} = Q_{10}$带入等式1中，我们可以将面片的参数$\{u,v\}$变换为$\{(1-u)(1-v), u, (1-u)v\}$来表示三角形$\{Q_{00}, Q_{01}, Q_{01}\}$的重心坐标。作为替代方案，等式1的插值公式可以直接使用。

图8-3显示了斯坦福兔子模型在OptiX[20]中通过光线追踪渲染的不同版本。我们使用$1000 \times 1000$屏幕分辨率，对于每个像素发射一条主要光线，对于每个相交点，发射九条环境光遮挡(AO)光线。旨在模拟典型光线追踪中的主要光线和次要光线的的分布。通过计算每秒处理的光线总数来测量性能，从而减轻主光线未命中对总体性能的影响。我们将环境遮挡距离设为$\infty$，并让这些光线在命中任何模型后终止。

(a) 原始模型: 0 面片, 69451 三角形, 每秒770M光线
(b) Blender模型: 14962 面片, 3713 三角形, 每秒825M光线
(c) 实时场对齐模型: 14962 面片, 454 三角形, 每秒841M光线
(d) Blender模型使用了原始模型的顶点
(e) 减少50%顶点数量的实时场对齐模型

图8-3 使用Titan Xp光线追踪渲染环境光遮蔽的不同版本斯坦福兔子模型

Blender使用了原始模型的顶点，而实时网格系统尝试优化顶点的位置，并且可以设置顶点数量的目标值。图8-3d和8-3e显示了系统得到的网格。图8-3a和8-3c均使用Phong着色法渲染。

作为对比，Ramsey等人[24]的单精度方法在图8-3b的模型中达到了每秒409M条光线，在图8-3c中达到了每秒406M条光线。对于双精度方法，性能下降到了每秒196M和198M条光线。

这些四边形化系统并不知道我们想要渲染非平面图元。所以这些系统的生产的网格的平整度作为质量衡量标准，输出的四边形有1%是完全平坦的。我们认为这是我们目前四边形网格生成过程的局限性，也是未来利用双线性插值面片的非平面性质的机会。

## 8.2 GARP细节
光线和面片的交点由$t$（对于沿光线的交点）和$\{u,v\}$（对于面片上的交点）定义。仅仅知道$t$是不够的，因为还需要$\{u,v\}$来计算法线。虽然我们最终会需要全部三个变量，我们从简单的几何考虑先计算$t$的值（不通过求解代数方程）。

双线性面片的边（等式1）都是直线，我们首先在相对的边上定义两个点$P_{a}(u) = (1-u)Q_{00}  + u Q_{10}$和$P_{b}u(u) = (1-u)Q_{01} + uQ_{11}$。通过这些点，我们考虑一系列经过$P_{a}$和$P_{b}$的线段。对于任何$u \in [0,1]$，线段$(P_{a}(u), P_{b}(u))$一定在面片上。

图8-4 计算光线/面片的交点

第一步，我们首先推到出计算光线和线段$(P_{a}(u), P_{b}(u))$间的有正负的距离的方程，并将方程的值设置为0。距离方程为$(P_{a} - 0) \cdot \textbf{n} / \| \textbf{n} \|$，其中$\textbf{n} = (P_{a} - P_{b}) \times \hat{\textbf{d}}$。我们仅需要分子部分，将其设为0就可以得到关于$u$的二次方程。

分子部分是标量乘积$(P_{a} - 0) \dot (P_{b} - P_{a}) \times \hat{\textbf{d}}$，也是是由三个向量定义的平行六面体的（有正负的）体积。是关于$u$的二次多项式。通过一些简单的简化之后，多项式的系数简化到在8.4节中14-17行中计算的三个系数$a,b,c$。我们将$\textbf{q}_{n} = (Q_{10} - Q_{00}) \times (Q_{01} - Q_{11})$单独预先计算。如果该向量的长度为0，那么四边形就可以简化为平面梯形，$u^{2}$的系数$c$为0。我们在代码中使用显示分支来处理这种情况(8.4节中的第23行)。

对于非梯形的一般平面四边形，向量$\textbf{q}_{n}$和四边形平面垂直。显示计算和使用$\textbf{q}_{n}$的值有助于计算的准确性，因为在大多数模型中，面片几乎是平面的。很关键的是即使是对于平面面片，方程$a + bu + c u^{2}$也有两个解。图8-5的左图显示了这样一种情况。方程的两个解都落在$[0,1]$的区间内，我们需要计算$v$来舍掉其中一个解。这张图显示了一个自我重叠的面片。对于不重叠的平面四边形，只会有一个$u$和$v$都在$[0,1]$区间内的解。即便如此，也没有必要在代码中明确表达这种逻辑，因为这会增加没有必要的代码差异。

图8-5 左图：光线与平面在$\{u,v\} = \{0.3, 0.5\}$和$\{0.74, 2.66\}$处相交。
右图：光线和面片处在的的（扩展的）双线性表面没有交点

使用经典公式$\frac{-b \pm \sqrt{b^{2} - 4ac}}{2c}$有其危险性，根据$b$的符号不同，其中一个根需要计算两个（相对）大数的差。出于这个原因，我们先计算稳定解[23]，然后使用Vieta公式$u_{1}u_{2} = a/c$来计算第二个解（代码26行）。

第二步，我们对于每个$u \in [0,1]$的解，计算对应的$v$的值。最简单的方法是从$P_{a} + v(P_{b} - P_{a}) = O + t \hat{\textbf{d}}$的三个等式中任选两个。但是因为$P_{a}(u)$和$P_{b}(u)$的坐标不是精确计算的，并且并不能明显的选择最好的两个方程,可能会导致数值误差。

我们测试了不同方法。神奇的是，最好的方法是忽略$O + t \hat{\textbf{d}}$和$P_{a} + v(P_{b} - P_{a})$相交的事实。我们计算让这两条线距离最短的$v$和$t$（最短距离非常接近0）。通过计算和两条线都垂直的$\textbf{n} = (P_{b} - P_{a}) \times \hat{\textbf{d}}$来简化计算。对应的代码在8.4节中的31行和43行之后，包括了一些线性代数简化。

通常来讲，一条射线和一个平面，如果两者不平行，那么肯定存在交点。但是对于非平面双线性曲面来说，并不一定存在交点，如图8-5的右图所示。因此，我们对于行列式是负的部分，不进行相交测试。

把以上部分整合在一起写成简单干净的代码。代码可以通过把面片转换到光线为中心的空间: $O = 0, \hat{\textbf{d}} = \{0,0,1\}$来进一步简化。Duff等人[8]最近提出了类似的无分支转换算法。然而我们发现这种方法只能轻微的提升效率，因为GARP是已经优化到了下一阶段的算法。

## 8.3 结果讨论
交点可以根据已知参数$t,u,v$通过$X_{r} = R(t)$或$X_{q} = Q(u,v)$计算出。两者间的距离$\| X_{r} - X_{q} \|$给出了对于计算误差的真实估计（在理想情况中，这两个点为同一点）。为了得到一个无量纲的数值，我们将结果处以面片的周长。图8-6显示了在一些模型中的误差，其中蓝色表示没有误差，棕色表示误差大于$10^{-5}$。两步GARP计算在每一步中自动减少了可能的误差：首先我们找到$u$的最好估值，然后使用已知的$u$，进一步减少总误差。

图8-6 根据颜色显示渲染Fang等人[9]提供的模型中的误差，蓝色表示没有误差，棕色表示误差大于$10^{-5}$

网格四边形化在某些程度上提高了其质量。在这个过程中，顶点变得更为整齐，从而可以使用更好的光线追踪加速结构。根据原始模型的复杂度不同，存在理想的顶点缩减比率，在该比吕夏任然保持模型的原本特征，同时极大改善了光线追踪性能。我们在图8-7和图8-9中说明了这一点，在左图中是原始三角形网格（在OptiX中渲染），右图是简化后的面片网格，对于每个后续模型减少了大约50%顶点数量。

图8-7 使用OptiX的光线/三角形相交计算算法渲染的原始版本的笑面佛[7a]和三个四边形化的模型[7b-7d]。各个算法的性能数据(使用Titan Xp渲染时的每秒光线数量(百万条))已给出。

$\nearrow$: 世界坐标的GARP, $\uparrow$：光线中心坐标的GARP，$\triangleleft \triangleright$：将四边形视为两个三角形，[24] Ramsey等人的算法

图8-8 斯坦福泰国雕像
图8-9 斯坦福露西雕像

对于四边形化，我们使用了Jakob等人[12]的实时场对齐网格系统。该系统并不总是输出纯四边形的网格，根据我们的实验，输出中大约有1%到5%的三角形。我们将这些三角形视为退化的四边形（通过简单的复制第三个顶点）。对于斯坦福3D扫描的模型数据，都是曲面形状，生产的面片其中有1%都是完全平整的。

对于图8-7到图8-9中的每个模型，我们显示了了GARP算法的性能，包括使用光线中心系统的GARP，每个四边形都被视作两个三角形的情况，和作为参考的Ramsey等人[24]的算法。性能通过计算发出光线的总数计算（包括主光线和每个交点的九条环境光遮蔽光线）。GARP算法相对于Ramsey代码的现实时间的提升和模型的复杂度反比相关，因为更复杂的模型需要更多的遍历。

虽然我们的算法并不能与硬件光线/三角形相交器[19]相比，GARP显示了未来硬件开发的可能方向。我们展示了一个计算非平面图面的快速算法，该算法可能对于一些问题很有帮助。可能的研究方向包括高度场渲染，细分曲面[3]，碰撞检测[10]，位移贴图[16]，和其他的效果。也有许多基于CPU的光线追踪系统可能从GARP中受益，虽然我们目前并没有对于相应系统实现该算法。

## 8.4 代码

```cpp
RT_PROGRAM void intersectPatch(int prim_idx)
{
    // ray是OptiX中的rtDeclareVariable(Ray, ray, rtCurrentRay,)
    // patchdata是optix::rtBuffer
    const PatchData &patch = patchdata[prim_idx];
    const float3 *q = patch.coefficients();
    // 四个角和法线qn
    float3 q00 = q[0], q10 = q[1], q11 = q[2], q01 = q[3];
    float3 e10 = q10 - q00; // q01 ----------- q11
    float3 e11 = q11 - q10; // |                |
    float3 e00 = q01 - q00; // | e00       e11  | 我们预先计算
    float3 qn = q[4];       // |      e10       | qn = cross(q10-q00,
    q00 -= ray.origin;      // q00 ----------- q10 q01-q11)
    q10 -= ray.origin;
    float a = dot(cross(q00, ray.direction), e00); // 方程为
    float c = dot(qn, ray.direction);              // a + b u + c u^2
    float b = dot(cross(q10, ray.direction), e11); // 首先计算
    b -= a + c;                                    // a+b+c然后计算b
    float det = b * b - 4 * a * c;
    if (det < 0)
        return;               // 见图5的右图部分
    det = sqrt(det);          // 在CUDA_NVRTC_OPTIONS中使用-use_fast_math
    float u1, u2;             // 参数u的两个解
    float t = ray.tmax, u, v; // 需要最小的t>0的解
    if (c == 0)
    {                                       // 如果 c == 0，则是梯形
        u1 = -a / b;
        u2 = -1;                            // 并且只有一个解
    }
    else
    {                                      // 在斯坦福模型中 c != 0
        u1 = (-b - copysignf(det, b)) / 2; // 数值上的“稳定”解
        u2 = a / u1;                       // 使用韦达定理
        u1 /= c;
    }
    if (0 <= u1 && u1 <= 1)
    {                                   // 是否在面片中
        pa = lerp(q00, q10, u1);        // 在边e10上的点 (图4)
        float3 pb = lerp(e00, e11, u1); // 实际上是pb - pa
        float3 n = cross(ray.direction, pb);
        det = dot(n, n);
        n = cross(n, pa);
        float t1 = dot(n, pb);
        float v1 = dot(n, ray.direction); // 不用判断是否t1 < t
        if (t1 > 0 && 0 <= v1 && v1 <= det)
        { // if t1 > ray.tmax,
            t = t1 / det;
            u = u1;
            v = v1 / det; // 会在rtPotentialIntersection中被舍去
        }                 
    }
    if (0 <= u2 && u2 <= 1)
    {                                   // 稍有不同
        float3 pa = lerp(q00, q10, u2); // 因为u1可能是我们要的解
        float3 pb = lerp(e00, e11, u2); // 并且我们需要0 < t2 < t1
        float3 n = cross(ray.direction, pb);
        det = dot(n, n);
        n = cross(n, pa);
        float t2 = dot(n, pb) / det;
        float v2 = dot(n, ray.direction);
        if (0 <= v2 && v2 <= det && t > t2 && t2 > 0)
        {
            t = t2;
            u = u2;
            v = v2 / det;
        }
    }
    if (rtPotentialIntersection(t))
    {
        // 填充相交的结构体irec.
        // 最接近的相交点的法线会在shader中归一化
        float3 du = lerp(e10, q11 - q01, v);
        float3 dv = lerp(e00, e11, u);
        irec.geometric_normal = cross(du, dv);
#if defined(SHADING_NORMALS)
        const float3 *vn = patch.vertex_normals;
        irec.shading_normal = lerp(lerp(vn[0], vn[1], u),
                                   lerp(vn[3], vn[2], u), v);
#else
        irec.shading_normal = irec.geometric_normal;
#endif
        irec.texcoord = make_float3(u, v, 0);
        irec.id = prim_idx;
        rtReportIntersection(0u);
    }
}
```

## 致谢
我们使用了Blender[4]和实时场对齐网格系统[12]进行了网格四边形化。我们非常感谢能有机会使用斯坦福3D扫描数据库的模型和Fang等人[9]提供的模型进行研究。这些系统和模型在知识共享许可协议下使用。

作者还想要感谢匿名审稿人和编辑的宝贵的意见和有益的建议。

## 参考文献
[1] Abert, O., Geimer, M., and Muller, S. Direct and Fast Ray Tracing of NURBS Surfaces. In IEEE Symposium on Interactive Ray Tracing (2006), 161–168.

[2] Benthin, C., Wald, I., and Slusallek, P. Techniques for Interactive Ray Tracing of Bézier Surfaces. Journal of Graphics Tools 11, 2 (2006), 1–16.

[3] Benthin, C., Woop, S., Nießner, M., Selgrad, K., and Wald, I. Efficient Ray Tracing of Subdivision Surfaces Using Tessellation Caching. In Proceedings of High-Performance Graphics (2015), pp. 5–12.

[4] Blender Online Community. Blender—a 3D Modelling and Rendering Package. Blender Foundation, Blender Institute, Amsterdam, 2018.

[5] Blinn, J. Jim Blinn’s Corner: A Trip Down the Graphics Pipeline. Morgan Kaufmann Publishers Inc., 1996.

[6] Boubekeur, T., and Alexa, M. Phong Tessellation. ACM Transactions on Graphics 27, 5 (2008), 141:1–141:5.

[7] Brainerd, W., Foley, T., Kraemer, M., Moreton, H., and Nießner, M. Efficient GPU Rendering of Subdivision Surfaces Using Adaptive Quadtrees. ACM Transactions on Graphics 35, 4 (2016), 113:1–113:12. 

[8] Duff, T., Burgess, J., Christensen, P., Hery, C., Kensler, A., Liani, M., and Villemin, R. Building an Orthonormal Basis, Revisited. Journal of Computer Graphics Techniques 6, 1 (March 2017), 1–8.

[9] Fang, X., Bao, H., Tong, Y., Desbrun, M., and Huang, J. Quadrangulation Through Morse- Parameterization Hybridization. ACM Transactions on Graphics 37, 4 (2018), 92:1–92:15.

[10] Fournier, A., and Buchanan, J. Chebyshev Polynomials for Boxing and Intersections of Parametric Curves and Surfaces. Computer Graphics Forum 13, 3 (1994), 127–142.

[11] Hanrahan, P. Ray-Triangle and Ray-Quadrilateral Intersections in Homogeneous Coordinates, http://graphics.stanford.edu/courses/cs348b-04/rayhomo.pdf, 1989.

[12] Jakob, W., Tarini, M., Panozzo, D., and Sorkine-Hornung, O. Instant Field-Aligned Meshes. ACM Transactions on Graphics 34, 6 (Nov. 2015), 189:1–189:15.

[13] Kajiya, J. T. Ray Tracing Parametric Patches. Computer Graphics (SIGGRAPH) 16, 3 (July 1982), 245–254. [14] Kensler, A., and Shirley, P. Optimizing Ray-Triangle Intersection via Automated Search. IEEE

Symposium on Interactive Ray Tracing (2006), 33–38. [15] Lagae, A., and Dutré, P. An Efficient Ray-Quadrilateral Intersection Test. Journal of Graphics Tools 10, 4 (2005), 23–32. 

[16] Lier, A., Martinek, M., Stamminger, M., and Selgrad, K. A High-Resolution Compression Scheme for Ray Tracing Subdivision Surfaces with Displacement. Proceedings of the ACM on Computer Graphics and Interactive Techniques 1, 2 (2018), 33:1–33:17.

[17] Loop, C., Schaefer, S., Ni, T., and Castaño, I. Approximating Subdivision Surfaces with Gregory Patches for Hardware Tessellation. ACM Transactions on Graphics 28, 5 (2009), 151:1–151:9.

[18] Mao, Z., Ma, L., and Zhao, M. G1 Continuity Triangular Patches Interpolation Based on PN Triangles. In International Conference on Computational Science (2005), pp. 846–849.

[19] NVIDIA. NVIDIA RTX™ platform, https://developer.nvidia.com/rtx, 2018.

[20] Parker, S. G., Bigler, J., Dietrich, A., Friedrich, H., Hoberock, J., Luebke, D., McAllister, D., McGuire, M., Morley, K., Robison, A., and Stich, M. OptiX: A General Purpose Ray Tracing Engine. ACM Transactions on Graphics 29, 4 (2010), 66:1–66:13.

[21] Peters, J. Smooth Free-Form Surfaces over Irregular Meshes Generalizing Quadratic Splines. In International Symposium on Free-form Curves and Free-form Surfaces (1993), pp. 347–361.

[22] Phong, B. T. Illumination for Computer-Generated Images. PhD thesis, The University of Utah, 1973.

[23] Press, W. H., Teukolsky, S. A., Vetterling, W. T., and Flannery, B. P. Numerical Recipes 3rd Edition: The Art of Scientific Computing, 3 ed. Cambridge University Press, 2007.

[24] Ramsey, S. D., Potter, K., and Hansen, C. D. Ray Bilinear Patch Intersections. Journal of Graphics, GPU, & Game Tools 9, 3 (2004), 41–47.