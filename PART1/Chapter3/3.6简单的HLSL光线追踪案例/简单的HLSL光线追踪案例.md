## 3.6一个简单的HLSL光线追踪案例
考虑下面的HLSL代码片段，它提供了一个在实践中更具体的光线追踪怎么工作的案例。首先它通过函数ShadowRay()定义了一条光线的实例化，如果光线是被遮蔽的就返回0，除此以外返回1（例如：一条“阴影光线”）。由于ShadowRay()调用了TraceRay()，因此它只能在光线生成（ray generation），最近碰撞（closesthit），
或miss着色器中调用（译者注：因为渲染管线原因）。逻辑上，我们假设一条光线是被遮挡（除非未命中（miss）着色器被执行，我们才明确知道这条光线是未被遮挡的）。这样做允许我们避免执行最近命中（closest-hit）着色器(RAY_FLAG_SKIP_CLOSEST_HIT_SHADER)，同时在any hit（遮蔽发生的地方）着色器触发后能够停止(RAY_FLAG_ACCEPT_FIRST_HIT_
AND_END_SEARCH)。

```
1 RaytracingAccelerationStructure scene; // C++ 构建层次包围盒（BVH）的地方
2
3 struct ShadowPayload { // 定义一个光线有效载荷
4   float isVisible; // 0: 遮挡, 1: 可见
5 };
6
7 [shader("miss")] //定义miss着色器#0
8 void ShadowMiss(inout ShadowPayload pay) {
9   pay.isVisible = 1.0f; //未命中！光线遮蔽
10 }
11
12 [shader("anyhit")] // anyhit着色器 添加至hit group 0 #0
13 void ShadowAnyHit(inout ShadowPayload pay,
14                  BuiltInTriangleIntersectionAttributes attrib) {
15   if ( isTransparent( attrib, PrimitiveIndex() ) )
16     IgnoreHit(); // Skip transparent hits
17 }
18
19 float ShadowRay( float3 orig, float3 dir, float minT, float maxT ) {
20   RayDesc ray = { orig, minT, dir, maxT }; //定义一条新光线。
21   ShadowPayload pay = { 0.0f }; // 假设光线是遮挡的
22   TraceRay( scene,
23       (RAY_FLAG_SKIP_CLOSEST_HIT_SHADER |
24       RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH),
25       0xFF, 0, 1, 0, ray, pay ); // hit group: 0; miss index: 0 （详见3.5.1）
26      return pay.isVisible; // 返回光线有效载荷
27 }
```

请注意这段代码中，自定义了函数isTransparent()，该函数请求材质系统（基于图元ID和命中点）去运行透明度测试。

在合适的地方使用ShadowRay()函数，阴影很容易就被其他着色器生成；例如，一个简单的环境光遮蔽渲染器可能类似于下面代码：

```
1 Texture2D<float4> gBufferPos, gBufferNorm; // 输入几何缓冲区（G-buffer）
2 RWTexture2D<float4> output; // 输出环境光遮蔽缓冲区
3
4 [shader("raygeneration")]
5 void SimpleAOExample() {
6   uint2 pixelID = DispatchRaysIndex().xy; //正在处理的像素
7   float3 pos = gBufferPos[ pixelID ].rgb; // 环境光遮蔽光线来自何方
8   float3 norm = gBufferNorm[ pixelID ].rgb; // G-buffer法线
9   float aoColor = 0.0f;
10  for (uint i = 0; i < 64; i++) // 使用64条光线.
11      aoColor += (1.0f/64.0f) * ShadowRay(pos, GetRandDir(norm), 1e-4);
12  output[ pixelID ] = float4( aoColor, aoColor, aoColor, 1.0f );
13 }
```

函数GetRandDir()返回一个在由表面法线定义的单位半球内，随机选择的方向。同时传递给ShdowRay()函数的最小值1e−4是一个为了防止自相交的偏移量（第6章节会提供更多高级选项）。