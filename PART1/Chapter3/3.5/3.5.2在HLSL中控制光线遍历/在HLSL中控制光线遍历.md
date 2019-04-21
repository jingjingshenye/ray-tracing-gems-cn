除了在光线启动时指定标记外，DirectX 还提供了另外三个函数来控制相交和 any-hit 着色器中的光线行为。在自定义相交着色器中调用 ReportHit()，以确定射线与基元的碰撞位置。这方面的一个例子如下:

```c++
if(doesIntersect( ray, curPrim ) ) {
   PrimHitAttrib hitAttribs = { ... <initialize here>... };
   uint hitType = <user-defined-value>;
   ReportHit( distToHit, hitType, hitAttribs );
}
```
ReportHit()的输入是到射线交点的距离、指定碰撞类型的用户定义整数和用户定义的碰撞属性结构。碰撞类型可以作为一个由HitKind()返回的8位无符号整数用于碰撞着色器。它对于确定光线/基元交点的属性很有用，比如面方向，但是由于它是用户定义的，所以是高度可定制的。当内置的三角形相交点记录碰撞时，HitKind()返回D3D12_HIT_KIND_TRIANGLE_FRONT_FACE或D3D12_HIT_KIND_TRIANGLE_
BACK_FACE。碰撞属性作为参数传递给任意碰撞和最近碰撞着色器。当使用内置的三角形相交点时，碰撞着色器使用类型为BuiltInTriangleIntersectionAttributes的参数。另外，注意ReportHit()返回true，如果这个碰撞被接受为到目前为止遇到的最近的碰撞。

在任意碰撞着色器中调用函数IgnoreHit()来停止处理当前碰撞点。这将返回对交集着色器的执行(ReportHit()返回false)，其行为类似于光栅中的丢弃调用，只是保留了对射线有效负载的修改。

在任意碰撞着色器中调用函数AcceptHitAndEndSearch()来接受当前碰撞，跳过任何未搜索的BVH节点，然后立即使用当前最近的碰撞继续到最近碰撞的着色器。这对于优化阴影光线遍历非常有用，因为这些光线只确定是否有东西被击中，而不会触发更复杂的阴影和光照评估。


