## 3.8基础的DXR初始化和设置
主机侧的初始化和DXR的建立都是继承于DirectX 12定义的进程的。所以例如适配器，命令分配器，命令队列等基础对象的创建是完全没有变化的。一个新的设备类型：ID3D12Device5，它包括请求GPU光线支持，为光线追踪加速结构确定内存请求，同时创建光线追踪管线状态对象(RTPSOs)。光线追踪的函数存在于一个新的命令列表类型，ID3D12GraphicsCommandList4，它包括建立和维护光线追踪加速结构，创建和设置光线追踪管线状态类型，和发射光线。以下是创建设备，请求光线追踪支持和创建一个光线追踪命令列表的例子代码：

```
1 IDXGIAdapter1* adapter; // Create as in raster-based code
2 ID3D12CommandAllocator* cmdAlloc; // Create as in raster-based code
3 ID3D12GraphicsCommandList4* cmdList; // 光线追踪命令列表
4 ID3D12Device5* dev; // 光线追踪设备
5 HRESULT hr; // Return type for D3D12 calls
6
7 // Create a D3D12 device capable of ray tracing.
8 hr = D3D12CreateDevice(adapter, D3D_FEATURE_LEVEL_12_1,
9 _uuidof(ID3D12Device5), (void**)&dev);
10 if (FAILED(hr)) Exit("Failed to create device");
11
12 // Check if the D3D12 device actually supports ray tracing.
13 D3D12_FEATURE_DATA_D3D12_OPTIONS5 caps = {};
14 hr = dev->CheckFeatureSupport(D3D12_FEATURE_D3D12_OPTIONS5,
15 &caps, sizeof(caps));
16
17 if (FAILED(hr) || caps.RaytracingTier < D3D12_RAYTRACING_TIER_1_0)
18 Exit("Device or driver does not support ray tracing!");
19
20 // Create a command list that supports ray tracing.
21 hr = dev->CreateCommandList(0, D3D12_COMMAND_LIST_TYPE_DIRECT,
22 cmdAlloc, nullptr, IID_PPV_ARGS(& cmdList));
```

在创建设备之后，使用新的D3D12_FEATURE_DATA_OPTIONS5结构通过CheckFeatureSupport()查询是否支持光线追踪。光线追踪踪支持属于D3D12_RAYTRACING_TIER枚举定义的层。目前，存在两个等级:D3D12_RAYTRACING_TIER_1_0和D3D12_RAYTRACING_TIER_NOT_SUPPORTED。

*3.8.1几何和加速器结构*

分层场景表示对于高性能的光线追踪非常重要，因为它们减少了追踪复杂度（从线性到对数的大量光线与图元的交点）。最近几年，研究者探索了各种光线追踪加速结构的方案。今天的共识是层次包围盒（BVHs）变体具有最好的表现。除了层次结构的图元分组，BVHs还可以保证有限的内存使用。

DirectX加速结构是不透明的，驱动程序和底层硬件决定数据结构和内存布局。
现有的实现依赖于层次包围盒（BVHs），但是供应商可以选择其他结构。DXR加速结构通常是运行的时候在GPU上构建的，包含两个层次:底层和顶层。底层加速结构包括几何图形和程序式图元。顶层加速结构包括一个或多个底层加速结构。这就允许几何体实例把同样的底层加速结构多次地插入到一个顶层加速结构，每个底层加速结构具有不同的转换矩阵。底层结构相对较慢，但能快速传递光线相交（ deliver fast ray intersection）。顶层结构构建快速，提高了几何体的灵活性和复用性，但过渡使用会减少性能。为了更好的性能，底层结构应该重叠部分尽可能的少。

在动态场景中，如果几何体的拓扑结构没有改变（只是节点包围盒发生改变），那么加速结构只需要“改装（refit）”，而不需要重建（rebuild）层次包围盒。改装消耗的数量级是低于重建的，但长时间的重复改装通常会降低光线追踪的性能。为了平衡追踪和构建的消耗，使用了一种改装和重建恰当合并的方法。

*# 3.8.1.1 底层加速器结构*

要创建一个加速结构，从构建底层开始。首先,使用D3D12_RAYTRACING_GEOMETRY_DESC结构，用于指定顶点、索引和底层结构中包含的几何变换数据。值得注意的是,光线追踪顶点和索引缓冲区并不特殊，它们与光栅化中缓冲区相同。下面是一个演示如何指定不透明几何图形的例子:


```
1 struct Vertex {
2   XMFLOAT3 position;
3   XMFLOAT2 uv;
4 };
5
6 vector<Vertex> vertices;
7 vector<UINT> indices;
8 ID3D12Resource* vb; // 顶点缓冲区
9 ID3D12Resource* ib; // 索引缓冲区
10
11 // 描述几何。
12 D3D12_RAYTRACING_GEOMETRY_DESC geometry;
13 geometry.Type = D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES;
14 geometry.Triangles.VertexBuffer.StartAddress =
15      vb->GetGPUVirtualAddress();
16 geometry.Triangles.VertexBuffer.StrideInBytes = sizeof(Vertex);
17 geo metry.Triangles.VertexCount = static_cast<UINT>(vertices.size());
18 geometry.Triangles.VertexFormat = DXGI_FORMAT_R32G32B32_FLOAT;
19 geometry.Triangles.IndexBuffer = ib->GetGPUVirtualAddress();
20 geometry.Triangles.IndexFormat = DXGI_FORMAT_R32_UINT;
21 geometry.Triangles.IndexCount = static_cast<UINT>(indices.size());
22 geometry.Triangles.Transform3x4 = 0;
23 geometry.Flags = D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;
```

当描述底层加速结构几何体时，使用标记来通知光线追踪着色器几何体是否透明。例如，我们在第3.6节中看到的，着色器知道相交的几何图形是不透明的还是透明的。如果几何是不透明的，指定D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;否则,指定* _FLAG_NONE。接下来，查询构建BLAS所需的内存并存储完整构建的结构。使用device新的GetRaytracingAccelerationStructurePrebuildInfo()
函数获取划痕（scratch）和结果（result）缓冲区的大小。在构建过程中会使用到划痕（scratch）缓冲区，结果缓冲区存储完成的BLAS。构建标识符描述了预期的BLAS使用情况，允许内存和性能优化。D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_ FLAG_MINIMIZE_MEMORY 和 *_ALLOW_COMPACTION标识符帮助减少内存需求.其他标识符表示其他的期望特性，比如更快的跟踪或构建时间(*_PREFER_FAST_TRACE或*_PREFER_FAST_BUILD)或允许动态修改层次包围盒(*_ALLOW_UPDATE)。接下来的是简单的例子：

```
1 //  描述底层加速结构输入。
2 D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS ASInputs = {};
3 ASInputs.Type =
4 D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL;
5 ASInputs.DescsLayout = D3D12_ELEMENTS_LAYOUT_ARRAY;
6
7 // 来自之前的代码段
8 ASInputs.pGeometryDescs = &geometry;
9
10 ASInputs.NumDescs = 1;
11 ASInputs.Flags =
12 D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE;
13
14 // Get the memory requirements to build the BLAS.
15 D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO ASBuildInfo = {};
16 dev->GetRaytracingAccelerationStructurePrebuildInfo(
17 &ASInputs, &ASBuildInfo);
```

在确定所需内存之后，为底层加速结构分配GPU缓冲区。划痕（scratch）和结果（result）缓冲区都必须支持无序访问视图，所以使用D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS标识符设置给它。使用D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE作为最终底层加速结构缓冲区的初始状态。在指定几何体和分配底层加速结构后，我们就能构建我们加速结构。以下是参考代码：

```
1 ID3D12Resource* blasScratch; // Create as described in text.
2 ID3D12Resource* blasResult; // Create as described in text.
3
4 // 描述底层加速结构
5 D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC desc = {};
6 desc.Inputs = ASInputs; // 来自于之前的代码片段
7
8 desc.ScratchAccelerationStructureData =
9   blasScratch->GetGPUVirtualAddress();
10 desc.DestAccelerationStructureData =
11  blasResult->GetGPUVirtualAddress();
12
13 // 构建底层加速结构
14 cmdList->BuildRaytracingAccelerationStructure(&desc, 0, nullptr);
```


因为底层加速结构可能在GPU上异步构建，所以需要等到构建完成后才能使用它。
为此，需要在引用BLAS结果缓冲区的命令列表中添加一个无序访问视图屏障（ UAV barrier）。


*# 3.8.1.2 顶层加速结构*

建立顶层加速结构与底层结构大致一致，只有一些小但重要的变化。每个顶层加速结构包含来自于底层加速结构的一系列几何实例，用此来取代提供几何描述。每个实例都有一个掩码允许在每条光线的基础上拒绝整个实例，而不需要任何图元相交，连同TraceRay()的参数(参见第3.5.1节)。

For example, an instance mask could disable shadowing on a per-object basis.
例如，实例掩码可以基于每个对象禁用阴影。
Instances can each uniquely transform the BLAS geometry.
允许额外的标识符覆盖到透明，正面缠绕，和剔除。以下的代码定义了一个顶层加速结构：
```
1 // Describe the top-level acceleration structure instance(s).
2 D3D12_RAYTRACING_INSTANCE_DESC instances = {};
3 // Available in shaders
4 instances.InstanceID = 0;
5 // 选择hit group shader下标
6 instances.InstanceContributionToHitGroupIndex = 0;
7 // 与TraceRay()的参数进行按位与
8 instances.InstanceMask = 1;
9 instances.Transform = &identityMatrix;
10 // Transparency? Culling?
11 instances.Flags = D3D12_RAYTRACING_INSTANCE_FLAG_NONE;
12 instances.AccelerationStructure = blasResult->GetGPUVirtualAddress();
```


创建描述实例后，将它们上传到GPU缓冲区。在查询内存需求时，引用此缓冲区作为TLAS输入。与BLAS一样，查询内存需要使用GetRaytracingAccelerationStructurePrebuildInfo()，但是使用d3d12_raytracing_acceleration_structure_type_top_level类型指定TLAS构造。接下来，分配scratch和结果缓冲区，然后调用BuildRaytracingAccelerationStructure()来构建TLAS。与底层一样，放置一个无序访问视图障屏障 （UAV barrier）在顶层结果缓冲区之上，可以确保在使用前完成加速结构构建。


# 3.8.2 根签名
类似于c++中的函数签名，DirectX 12根签名定义了一些参数它们可在着色器间传递。这些参数储存了可以定位显存资源（例如缓冲区、纹理、或常量）的信息。DXR根签名继承于DirectX的根签名，但有两个明显的改动。

首先是，光线追踪着色器可以使用本地根签名或全局根签名。本地根签名直接从DXR着色器表（详见3.10）获取数据且使用the D3D12_ROOT_SIGNATURE_
FLAG_LOCAL_ROOT_SIGNATURE标识符给D3D12_ROOT_SIGNATURE_DESC结构初始化，
这个标识符只提供给了光线追踪，因此避免了和其他签名标识符合并。全局根签名的原始数据来自于DirectX命令列表，它不需要指定标识符且能够被图像、计算和光线追踪共享。本地签名和全局签名之间的区别对于
以不同的更新速率(例如，逐图元对逐帧)分离资源是有益的。


其次是，所有光线跟踪着色器都应该使用D3D12_SHADER_VISIBILITY_ALL作为
D3D12_ROOT_PARAMETER中的可视性参数，无论是使用本地还是全局
根签名。因为光线追踪根签名在计算时，会共享命令列表状态
，同时本地根参数也总是对所有光线追踪着色器可见。所以减少可见度是不可能的。
# 3.8.3着色器编译
紧跟在建立加速结构体和定义根签名之后的是，用DirectX着色器编译器（dxc）加载和编译着色器。初始化这种编译器时需要用到多种帮助类：

```
1 dxc::DxcDllSupport dxcHelper;
2 IDxcCompiler* compiler;
3 IDxcLibrary* library;
4 CComPtr<IDxcIncludeHandler> dxcIncludeHandler;
5
6 dxcHelper.Initialize();
7 dxcHelper.CreateInstance(CLSID_DxcCompiler, &compiler);
8 dxcHelper.CreateInstance(CLSID_DxcLibrary, &library);
9 library->CreateIncludeHandler(&dxcIncludeHandler);
```

接下来，使用IDxcLibrary类来加载着色器资源。这个帮助类编译着色器代码；
指定lib_6_3作为目标简概（target profile）。编译DirectX中间语言字节码储存在IDxcBlob中，我们在之后会用IDxcBlob来建立我们的光线追踪管线状态对象。大部分的程序都会使用很多着色器，所以封装一个帮助编译的函数是很有用的。我们将在下面展示并使用一个这样的函数：

```
1 void CompileShader(IDxcLibrary* lib, IDxcCompiler* comp,
2                    LPCWSTR fileName, IDxcBlob** blob)
3 {
4   UINT32 codePage(0);
5   IDxcBlobEncoding* pShaderText(nullptr);
6   IDxcOperationResult* result;
7
8   // 加载并编码着色器文件。
9   lib->CreateBlobFromFile(fileName, & codePage, & pShaderText);
10
11  // 编译着色器; "main" 是执行开始的地方。
12  comp->Compile(pShaderText, fileName, L"main", "lib_6_3",
13   nullptr, 0, nullptr, 0, dxcIncludeHandler, &result);
14
15   // 获得着色器字节码的结果
16   result->GetResult(blob);
17 }
18
19   // 编译着色器DXIL字节码
20  IDxcBlob *rgsBytecode, *missBytecode, *chsBytecode, *ahsBytecode;
21
22   // Call our helper function to compile the ray tracing shaders.
23   CompileShader(library, compiler, L"RayGen.hlsl", &rgsBytecode);
24   CompileShader(library, compiler, L"Miss.hlsl", &missBytecode);
25   CompileShader(library, compiler, L"ClosestHit.hlsl", &chsBytecode);
26   CompileShader(library, compiler, L"AnyHit.hlsl", &ahsBytecode);
```
