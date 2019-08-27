# meshoptimizer [![Build Status](https://travis-ci.org/zeux/meshoptimizer.svg?branch=master)](https://travis-ci.org/zeux/meshoptimizer) [![codecov.io](https://codecov.io/github/zeux/meshoptimizer/coverage.svg?branch=master)](https://codecov.io/github/zeux/meshoptimizer?branch=master) ![MIT](https://img.shields.io/badge/license-MIT-blue.svg) [![GitHub](https://img.shields.io/badge/repo-github-green.svg)](https://github.com/zeux/meshoptimizer)

## Purpose

When a GPU renders triangle meshes, various stages of the GPU pipeline have to process vertex and index data. The efficiency of these stages depends on the data you feed to them; this library provides algorithms to help optimize meshes for these stages, as well as algorithms to reduce the mesh complexity and storage overhead.

The library provides a C and C++ interface for all algorithms; you can use it from C/C++ or from other languages via FFI (such as P/Invoke). If you want to use this library from Rust, you should use [meshopt crate](https://crates.io/crates/meshopt).

[gltfpack](#gltfpack), which is a tool that can automatically optimize glTF files, is developed and distributed alongside the library.

## Installing

meshoptimizer is hosted on GitHub; you can download the latest release using git:

```
git clone -b v0.12 https://github.com/zeux/meshoptimizer.git
```

Alternatively you can [download the .zip archive from GitHub](https://github.com/zeux/meshoptimizer/archive/v0.12.zip).

## Building

meshoptimizer is distributed as a set of C++ source files. To include it into your project, you can use one of the two options:

* Use CMake to build the library (either as a standalone project or as part of your project)
* Add source files to your project's build system

The source files are organized in such a way that you don't need to change your build-system settings, and you only need to add the files for the algorithms you use.

## Pipeline

When optimizing a mesh, you should typically feed it through a set of optimizations (the order is important!):

1. Indexing
2. Vertex cache optimization
3. Overdraw optimization
4. Vertex fetch optimization
5. Vertex quantization
6. (optional) Vertex/index buffer compression

## Indexing

Most algorithms in this library assume that a mesh has a vertex buffer and an index buffer. For algorithms to work well and also for GPU to render your mesh efficiently, the vertex buffer has to have no redundant vertices; you can generate an index buffer from an unindexed vertex buffer or reindex an existing (potentially redundant) index buffer as follows:

First, generate a remap table from your existing vertex (and, optionally, index) data:

```c++
size_t index_count = face_count * 3;
std::vector<unsigned int> remap(index_count); // allocate temporary memory for the remap table
size_t vertex_count = meshopt_generateVertexRemap(&remap[0], NULL, index_count, &unindexed_vertices[0], index_count, sizeof(Vertex));
```

Note that in this case we only have an unindexed vertex buffer; the remap table is generated based on binary equivalence of the input vertices, so the resulting mesh will render the same way.

After generating the remap table, you can allocate space for the target vertex buffer (`vertex_count` elements) and index buffer (`index_count` elements) and generate them:

```c++
meshopt_remapIndexBuffer(indices, NULL, index_count, &remap[0]);
meshopt_remapVertexBuffer(vertices, &unindexed_vertices[0], index_count, sizeof(Vertex), &remap[0]);
```

You can then further optimize the resulting buffers by calling the other functions on them in-place.

## Vertex cache optimization

When the GPU renders the mesh, it has to run the vertex shader for each vertex; usually GPUs have a built-in fixed size cache that stores the transformed vertices (the result of running the vertex shader), and uses this cache to reduce the number of vertex shader invocations. This cache is usually small, 16-32 vertices, and can have different replacement policies; to use this cache efficiently, you have to reorder your triangles to maximize the locality of reused vertex references like so:

```c++
meshopt_optimizeVertexCache(indices, indices, index_count, vertex_count);
```

## Overdraw optimization

After transforming the vertices, GPU sends the triangles for rasterization which results in generating pixels that are usually first ran through the depth test, and pixels that pass it get the pixel shader executed to generate the final color. As pixel shaders get more expensive, it becomes more and more important to reduce overdraw. While in general improving overdraw requires view-dependent operations, this library provides an algorithm to reorder triangles to minimize the overdraw from all directions, which you should run after vertex cache optimization like this:

```c++
meshopt_optimizeOverdraw(indices, indices, index_count, &vertices[0].x, vertex_count, sizeof(Vertex), 1.05f);
```

The overdraw optimizer needs to read vertex positions as a float3 from the vertex; the code snippet above assumes that the vertex stores position as `float x, y, z`.

When performing the overdraw optimization you have to specify a floating-point threshold parameter. The algorithm tries to maintain a balance between vertex cache efficiency and overdraw; the threshold determines how much the algorithm can compromise the vertex cache hit ratio, with 1.05 meaning that the resulting ratio should be at most 5% worse than before the optimization.

## Vertex fetch optimization 顶点存取优化

After the final triangle order has been established, we still can optimize the vertex buffer for memory efficiency. Before running the vertex shader GPU has to fetch the vertex attributes from the vertex buffer; the fetch is usually backed by a memory cache, and as such optimizing the data for the locality of memory access is important. You can do this by running this code:

最终三角形顺序确定好之后，我们依然可以优化顶点buffer的存储。gpu在运行vs之前必须从顶点buffer种中读取顶点属性。通常，这种存取通过backed 一个内存缓存，所以对于内存存储的存放非常重要。你可以这么干

To optimize the index/vertex buffers for vertex fetch efficiency, call:

```c++
meshopt_optimizeVertexFetch(vertices, indices, index_count, vertices, vertex_count, sizeof(Vertex));
```

This will reorder the vertices in the vertex buffer to try to improve the locality of reference, and rewrite the indices in place to match; if the vertex data is stored using multiple streams, you should use `meshopt_optimizeVertexFetchRemap` instead. This optimization has to be performed on the final index buffer since the optimal vertex order depends on the triangle order.
这个将对顶点buffer重排顶点，尝试去提升引用位置，并且重写索引。如果顶点数据是使用多个流存储德，你应该先用`meshopt_optimizeVertexFetchRemap`。 这个优化必须最后作用在索引buffer上，因为对于顶点顺序的优化依赖三角形顺序。


Note that the algorithm does not try to model cache replacement precisely and instead just orders vertices in the order of use, which generally produces results that are close to optimal.
既然这个算法不是尝试去精确的重新定位缓存（ model cache replacement precisely），而是依据使用顺序去排序顶点，通常产生的结果接近最优化的（这段不是 太理解）


## Vertex quantization  顶点量化（quantization）

To optimize memory bandwidth when fetching the vertex data even further, and to reduce the amount of memory required to store the mesh, it is often beneficial to quantize the vertex attributes to smaller types. While this optimization can technically run at any part of the pipeline (and sometimes doing quantization as the first step can improve indexing by merging almost identical vertices), it generally is easier to run this after all other optimizations since some of them require access to float3 positions.
为了优化顶点数据读取的内存带宽 ，减少存储mesh需要的内存。通常对顶点属性的量化转为更小的类型是很有意义的。当然，这个优化可以运行在流程的所有阶段（有时，第一步做quantization可以提升索引，会合并一些近似相同的顶点）。当所有其他的优化都完成后，运行这个非常容易。

Quantization is usually domain specific; it's common to quantize normals using 3 8-bit integers but you can use higher-precision quantization (for example using 10 bits per component in a 10_10_10_2 format), or a different encoding to use just 2 components. For positions and texture coordinate data the two most common storage formats are half precision floats, and 16-bit normalized integers that encode the position relative to the AABB of the mesh or the UV bounding rectangle.
量化通常在特定区域；通常使用3个8比特的整形来才存储法向量，当然你可以使用更高精度的方式（比如使用10比特，构造一个10_10_10_2形式）。或者仅仅使用2个元素来编码。对于顶点和纹理坐标

The number of possible combinations here is very large but this library does provide the building blocks, specifically functions to quantize floating point values to normalized integers, as well as half-precision floats. For example, here's how you can quantize a normal:

可能的链接方式个数非常多，但是这个库提供building block。一种特定的函数去quantize浮点数转为整形数字或者半精度的浮点数。比如下面展示如何对法向量quantize：
```c++
unsigned int normal =
	(meshopt_quantizeUnorm(v.nx, 10) << 20) |
	(meshopt_quantizeUnorm(v.ny, 10) << 10) |
	 meshopt_quantizeUnorm(v.nz, 10);
```

and here's how you can quantize a position:
下面是演示如何quantize顶点：
```c++
unsigned short px = meshopt_quantizeHalf(v.x);
unsigned short py = meshopt_quantizeHalf(v.y);
unsigned short pz = meshopt_quantizeHalf(v.z);
```

## Vertex/index buffer compression  顶点/索引 缓冲 压缩

In case storage size or transmission bandwidth is of importance, you might want to additionally compress vertex and index data. While several mesh compression libraries, like Google Draco, are available, they typically are designed to maximize the compression ratio at the cost of disturbing the vertex/index order (which makes the meshes inefficient to render on GPU) or decompression performance. They also frequently don't support custom game-ready quantized vertex formats and thus require to re-quantize the data after loading it, introducing extra quantization errors and making decoding slower.

事实上，存储大小或者传输带宽非常重要，你或者想增加其他的压缩顶点和索引的算法。同时，有若干种压缩库，就像谷歌的Draco，也是可以的，它专门被涉及用来最大化压缩和减小顶点/索引。他们也经常不支持自定义的游戏的quantized顶点格式，也需要在加载数据之后重新恢复，增加了额外的quantization错误几率，以及 使解码缓慢。


Alternatively you can use general purpose compression libraries like zstd or Oodle to compress vertex/index data - however these compressors aren't designed to exploit redundancies in vertex/index data and as such compression rates can be unsatisfactory.
另外你可以使用通用目的的压缩库，就像zstd 或者 Oodle去压缩顶点索引数据，可视这些压缩不是专门涉及为了顶点和索引压缩，所以压缩率可能不令人满意。

To that end, this library provides algorithms to "encode" vertex and index data. The result of the encoding is generally significantly smaller than initial data, and remains compressible with general purpose compressors - so you can either store encoded data directly (for modest compression ratios and maximum decoding performance), or further compress it with zstd/Oodle to maximize compression ratio.

最后，这个提供了算法去编码顶点和索引数据。编码后的结果一般比初始的数据小，并且保留了通用压缩算法。所以你可以存储压缩后的数据（最大化压缩比率和最大化解码性能） 或者更进一步 使用zstd/))odel去最大化压缩


To encode, you need to allocate target buffers (preferably using the worst case bound) and call encoding functions:

压缩前，你需要分配目标缓冲（或者使用最快情况的大小），调用压缩函数
```c++
std::vector<unsigned char> vbuf(meshopt_encodeVertexBufferBound(vertex_count, sizeof(Vertex)));
vbuf.resize(meshopt_encodeVertexBuffer(&vbuf[0], vbuf.size(), vertices, vertex_count, sizeof(Vertex)));

std::vector<unsigned char> ibuf(meshopt_encodeIndexBufferBound(index_count, vertex_count));
ibuf.resize(meshopt_encodeIndexBuffer(&ibuf[0], ibuf.size(), indices, index_count));
```

You can then either serialize `vbuf`/`ibuf` as is, or compress them further. To decode the data at runtime, call decoding functions:

```c++
int resvb = meshopt_decodeVertexBuffer(vertices, vertex_count, sizeof(Vertex), &vbuf[0], vbuf.size());
int resib = meshopt_decodeIndexBuffer(indices, index_count, &buffer[0], buffer.size());
assert(resvb == 0 && resib == 0);
```

Note that vertex encoding assumes that vertex buffer was optimized for vertex fetch, and that vertices are quantized; index encoding assumes that the vertex/index buffers were optimized for vertex cache and vertex fetch. Feeding unoptimized data into the encoders will produce poor compression ratios. Both codecs are lossless - the only lossy step is quantization that happens before encoding.
既然，顶点压缩鉴定顶点缓冲已经做过顶点存取的优化，所以这些顶点是quantized。所以编码嘉定 顶点/索引 缓冲已经经过vertex cache 和 vertex fetch优化。填充未优化的数据到压缩算法里，将产生很差的压缩率。两种压缩都是无损的，唯一的有损过程是在对数据进行quantization的时候。

Decoding functions are heavily optimized and can directly target write-combined memory; you can expect both decoders to run at 1-3 GB/s on modern desktop CPUs. Compression ratios depend on the data; vertex data compression ratio is typically around 2-4x (compared to already quantized data), index data compression ratio is around 5-6x (compared to raw 16-bit index data). General purpose lossless compressors can further improve on these results.
解压函数也被重点优化过了，直接设定为目标写内存；你可以期待 1-3GB/s 在现代化桌面cpu上。压缩率根据数据，顶点压缩在2-4x（对比已经quantized的数据）。索引压缩率是5-6x（对比原始16字节索引数据）。通用无损压缩可以显著提高结果

Due to a very high decoding performance and compatibility with general purpose lossless compressors, the compression is a good fit for the use on the web. To that end, meshoptimizer provides both vertex and index decoders compiled into WebAssembly and wrapped into a module with JavaScript-friendly interface, `js/meshopt_decoder.js`, that you can use to decode meshes that were encoded offline:
由于非常高得编码性能和兼容无损通用压缩算法，这种压缩非常适合在web环境下。当然，meshoptimizer也提供了顶点和索引解码算法在WebAssembly中，而且包装为JavaScript-friendly interface,很方便得调用方式。

```js
// ready is a Promise that is resolved when (asynchronous) WebAssembly compilation finishes
await MeshoptDecoder.ready;

// decode from *Data (Uint8Array) into *Buffer (Uint8Array)
MeshoptDecoder.decodeVertexBuffer(vertexBuffer, vertexCount, vertexSize, vertexData);
MeshoptDecoder.decodeIndexBuffer(indexBuffer, indexCount, indexSize, indexData);
```

[Usage example](https://meshoptimizer.org/demo/) is available, with source in `demo/index.html`; this example uses .GLB files encoded using `gltfpack`.

## Triangle strip conversion 三角strip 对比

On most hardware, indexed triangle lists are the most efficient way to drive the GPU. However, in some cases triangle strips might prove beneficial:
对于大多数硬件，索引三角形list在gpu里非常高效。可视，在一些情况下，三角形strips或者更高效。

- On some older GPUs, triangle strips may be a bit more efficient to render
在一些老的gpu上，triangle strips 或许更加高效的渲染
- On extremely memory constrained systems, index buffers for triangle strips could save a bit of memory
在内存极度限制的系统上， triangle strip的索引buffer可以节省一些内存。
This library provides an algorithm for converting a vertex cache optimized triangle list to a triangle strip:
这个库提供另一个算法把顶点cache优化后的三角形转为 triangle strip

```c++
std::vector<unsigned int> strip(meshopt_stripifyBound(index_count));
unsigned int restart_index = ~0u;
size_t strip_size = meshopt_stripify(&strip[0], indices, index_count, vertex_count, restart_index);
```

Typically you should expect triangle strips to have ~50-60% of indices compared to triangle lists (~1.5-1.8 indices per triangle) and have ~5% worse ACMR.
Note that triangle strips can be stitched with or without restart index support. Using restart indices can result in ~10% smaller index buffers, but on some GPUs restart indices may result in decreased performance.
尤其，对于三角形list(1.5~1.8个索引/三角形），triangle strip可以节省50-60% 。
既然triangle strip 可以切换 索引支持。使用restart 索引可以产生~10%左右的顶点缓冲，但是一些gpu上，restart indice可能降低性能。

## Deinterleaved geometry 交错几何体

All of the examples above assume that geometry is represented as a single vertex buffer and a single index buffer. This requires storing all vertex attributes - position, normal, texture coordinate, skinning weights etc. - in a single contiguous struct. However, in some cases using multiple vertex streams may be preferable. In particular, if some passes require only positional data - such as depth pre-pass or shadow map - then it may be beneficial to split it from the rest of the vertex attributes to make sure the bandwidth use during these passes is optimal. On some mobile GPUs a position-only attribute stream also improves efficiency of tiling algorithms.
上面所有的示例都假定几何体是有一个顶点buffer和一个索引buffer构成。这个要求存储所有的顶点属性：坐标，法向量，纹理坐标，skin weight等等。在一个连续的结构体中。可视，在一些情况下使用多个定点流或许更好。特殊的，比如一些pass中只需要顶点数据，就像深度 或者 印象pass。 此时把顶点属性从其他的顶点属性里剥离出来，可以使带宽更优。在一些移动gpu上。只有顶点属性的流还会提升光栅化算法。

Most of the functions in this library either only need the index buffer (such as vertex cache optimization) or only need positional information (such as overdraw optimization). However, several tasks require knowledge about all vertex attributes.
在这个库大部分函数仅仅需要索引缓冲（就像顶点cache优化）或者仅仅需要顶点信息（就像voerdaraw优化）。可视，一些任务需要处理所有的顶点属性

For indexing, `meshopt_generateVertexRemap` assumes that there's just one vertex stream; when multiple vertex streams are used, it's necessary to use `meshopt_generateVertexRemapMulti` as follows:

```c++
meshopt_Stream streams[] = {
    {&unindexed_pos[0], sizeof(float) * 3, sizeof(float) * 3},
    {&unindexed_nrm[0], sizeof(float) * 3, sizeof(float) * 3},
    {&unindexed_uv[0], sizeof(float) * 2, sizeof(float) * 2},
};

std::vector<unsigned int> remap(index_count);
size_t vertex_count = meshopt_generateVertexRemapMulti(&remap[0], NULL, index_count, index_count, streams, sizeof(streams) / sizeof(streams[0]));
```

After this `meshopt_remapVertexBuffer` needs to be called once for each vertex stream to produce the correctly reindexed stream.

Instead of calling `meshopt_optimizeVertexFetch` for reordering vertices in a single vertex buffer for efficiency, calling `meshopt_optimizeVertexFetchRemap` and then calling `meshopt_remapVertexBuffer` for each stream again is recommended.

Finally, when compressing vertex data, `meshopt_encodeVertexBuffer` should be used on each vertex stream separately - this allows the encoder to best utilize corellation between attribute values for different vertices.

## Simplification 简化

All algorithms presented so far don't affect visual appearance at all, with the exception of quantization that has minimal controlled impact. However, fundamentally the most effective way at reducing the rendering or transmission cost of a mesh is to make the mesh simpler.
所有的简化算法，都是为了不影响显示效果的前提下。可是，最基础最有效的手段就是让 渲染或者传输的 三角mesh更简单点。

This library provides two simplification algorithms that reduce the number of triangles in the mesh. Given a vertex and an index buffer, they generate a second index buffer that uses existing vertices in the vertex buffer. This index buffer can be used directly for rendering with the original vertex buffer (preferably after vertex cache optimization), or a new compact vertex/index buffer can be generated using `meshopt_optimizeVertexFetch` that uses the optimal number and order of vertices.

这个库提供了两种精简算法，它降低了mesh中的三角形数量。考虑一个顶点 和一个索引buffer， 产生第二个索引buffer，使用在顶点buffer中存在的数据。这个索引buffer可以直接去渲染使用原始vertexbuffer（或者可以在顶点cache优化之后）。或者一个新得组合顶点/索引 buffer 可以通过 `meshopt_optimizeVertexFetch`生成。

The first simplification algorithm, `meshopt_simplify`, follows the topology of the original mesh in an attempt to preserve attribute seams, borders and overall appearance. For meshes with inconsistent topology or many seams, such as faceted meshes, it can result in simplifier getting "stuck" and not being able to simplify the mesh fully; it's recommended to preprocess the index buffer with `meshopt_generateShadowIndexBuffer` to discard any vertex attributes that aren't critical and can be rebuilt later such as normals.

第一个简化算法 `meshopt_simplify`。依据原始mesh得拓扑结构，保留接缝属性、边界和总体效果。对于一些不一致得拓扑结构或者有缝隙的，如同faceted 三角网，会导致算法卡住，不能整体简化； 推荐预先处理 索引buffer，通过`meshopt_generateShadowIndexBuffer` 去地球一些顶点属性，这些不是很关键的，可以被重建的属性，比如法向量。

```c++
float threshold = 0.2f;
size_t target_index_count = size_t(index_count * threshold);
float target_error = 1e-2f;

std::vector<unsigned int> lod(index_count);
lod.resize(meshopt_simplify(&lod[0], indices, index_count, &vertices[0].x, vertex_count, sizeof(Vertex), target_index_count, target_error));
```

Target error is an approximate measure of the deviation from the original mesh using distance normalized to 0..1 (so 1e-2f means that simplifier will try to maintain the error to be below 1% of the mesh extents). Note that because of topological restrictions and error bounds simplifier isn't guaranteed to reach the target index count and can stop earlier.
taget erro是一个大概度量，原始mesh 和精简后mesh的归一化距离（所以1e-2f 表示，简化算法将尝试保证这个错误在1% mesh的尺度之下）。当然，因为拓扑限制或者一些错误边界，不一定能保证达到目标索引个数，算法将提前停止。

The second simplification algorithm, `meshopt_simplifySloppy`, doesn't follow the topology of the original mesh. This means that it doesn't preserve attribute seams or borders, but it can collapse internal details that are too small to matter better because it can merge mesh features that are topologically disjoint but spatially close.
第二个简化算法 `meshopt_simplifySloppy`，不考虑原始三角形的拓扑关键。意味着它不考虑接缝和边界，但是它可以塌陷内部细节，去匹配更好。因为它可以合并顶点属性，并且合并一些拓扑上不连续的顶点。

```c++
float threshold = 0.2f;
size_t target_index_count = size_t(index_count * threshold);

std::vector<unsigned int> lod(target_index_count);
lod.resize(meshopt_simplifySloppy(&lod[0], indices, index_count, &vertices[0].x, vertex_count, sizeof(Vertex), target_index_count));
```

This algorithm is guaranteed to return a result at or below the target index count. It is 5-6x faster than `meshopt_simplify` when simplification ratio is large, and is able to reach ~20M triangles/sec on a desktop CPU (`meshopt_simplify` works at ~3M triangles/sec).
这个算法保证输出的结果一定是在目标索引个数之下的。它大约是5-6x倍的速度比`meshopt_simplify` 快，当简化比率很大，它能偶达到大约20M 三角形/秒在桌面cpu上，第一个算法大约是 ~3M三角形/s

When a sequence of LOD meshes is generated that all use the original vertex buffer, care must be taken to order vertices optimally to not penalize mobile GPU architectures that are only capable of transforming a sequential vertex buffer range. It's recommended in this case to first optimize each LOD for vertex cache, then assemble all LODs in one large index buffer starting from the coarsest LOD (the one with fewest triangles), and call `meshopt_optimizeVertexFetch` on the final large index buffer. This will make sure that coarser LODs require a smaller vertex range and are efficient wrt vertex fetch and transform.
当一系列的LOD 网格产生后，都使用了原始顶点缓冲，注意应该考虑顶点优化顺序，不能移动端gpu仅仅能变换一个顶点缓冲范围序列。 这种情况下推荐使用第一个优化对于每个顶点cache。然后把所有LOd数据装配到一个大的索引buffer中。从最粗糙得LOD开始。然后调用`meshopt_optimizeVertexFetch`在最后的大索引缓冲中。这样确保粗糙的lod要求一个更小的顶点范围并且更有效的顶点存储和变换。

## Efficiency analyzers  有效性分析

While the only way to get precise performance data is to measure performance on the target GPU, it can be valuable to measure the impact of these optimization in a GPU-independent manner. To this end, the library provides analyzers for all three major optimization routines. For each optimization there is a corresponding analyze function, like `meshopt_analyzeOverdraw`, that returns a struct with statistics.

最精确的性能评价是在GPU里去度量。在gpu里测试这些优化的影像。最后，这个提供了分析工具，丢与所有三种主要的优化，对每种优化有一个分析函数，比如`meshopt_analyzeOverdraw`返回了一个分析结构。


`meshopt_analyzeVertexCache` returns vertex cache statistics. The common metric to use is ACMR - average cache miss ratio, which is the ratio of the total number of vertex invocations to the triangle count. The worst-case ACMR is 3 (GPU has to process 3 vertices for each triangle); on regular grids the optimal ACMR approaches 0.5. On real meshes it usually is in [0.5..1.5] range depending on the amount of vertex splits. One other useful metric is ATVR - average transformed vertex ratio - which represents the ratio of vertex shader invocations to the total vertices, and has the best case of 1.0 regardless of mesh topology (each vertex is transformed once).
`meshopt_analyzeVertexCache` 返回顶点 cache 统计。通用的衡量参数是 ACMR- 平均cach缺失比率，所有的顶点个数除以三角形个数。 最坏的情况是3；通常优化的接近0.5.常规的mesh是在0.5~1.5之间，去决议顶点分割。另一个有效的参数是ATVR 平均转换顶点比率，他表示了vs调用次数/总顶点个数，最佳是1.0无论顶点如何转换。


`meshopt_analyzeVertexFetch` returns vertex fetch statistics. The main metric it uses is overfetch - the ratio between the number of bytes read from the vertex buffer to the total number of bytes in the vertex buffer. Assuming non-redundant vertex buffers, the best case is 1.0 - each byte is fetched once.
`meshopt_analyzeVertexFetch` 返回一个顶点存取统计。主要的参数使用 overfetch- 读取的顶点数据总量在整个顶点缓冲中的比率，如果是非冗余的顶点缓冲，最好的值就是1.0.

`meshopt_analyzeOverdraw` returns overdraw statistics. The main metric it uses is overdraw - the ratio between the number of pixel shader invocations to the total number of covered pixels, as measured from several different orthographic cameras. The best case for overdraw is 1.0 - each pixel is shaded once.
`meshopt_analyzeOverdraw` 返回overdraw 统计。主要的测量参数是 overdraw - 调用ps的个数 / 总覆盖的像素，对于不同角度的相机。最好的值是1.0，也就是一个像素只由进行一次ps

Note that all analyzers use approximate models for the relevant GPU units, so the numbers you will get as the result are only a rough approximation of the actual performance.
既然所有的分析都是模拟了gpu单位，所以这个个数只是个粗略的估计

## Memory management 内存管理

Many algorithms allocate temporary memory to store intermediate results or accelerate processing. The amount of memory allocated is a function of various input parameters such as vertex count and index count. By default memory is allocated using `operator new` and `operator delete`; if these operators are overloaded by the application, the overloads will be used instead. Alternatively it's possible to specify custom allocation/deallocation functions using `meshopt_setAllocator`, e.g.

很多算法分配临时内存去存储临时结果为了加速处理。这个内存占用也会依据输入的参数，比如顶点个数或者索引个数。默认的内存分配使用 `operator new` and `operator delete`； 如果这些操作符被重载，将使用重载后的操作付。 可以替代设置用户自定义的内存分配方式，使用`meshopt_setAllocator`

```c++
meshopt_setAllocator(malloc, free);
```

> Note that the library expects the allocation function to either throw in case of out-of-memory (in which case the exception will propagate to the caller) or abort, so technically the use of `malloc` above isn't safe. If you want to handle out-of-memory errors without using C++ exceptions, you can use `setjmp`/`longjmp` instead.

既然这个库期望分配拿书，当内存超过的时候（将发出一个异常）或者放弃，所以，技术上使用 `malloc`不安全。如果你想处理内存越界错误而不是使用c++的异常，那么你可以使用`setjmp`/`longjmp` 函数替代。

Vertex and index decoders (`meshopt_decodeVertexBuffer` and `meshopt_decodeIndexBuffer`) do not allocate memory and work completely within the buffer space provided via arguments.
顶点和索引解码 (`meshopt_decodeVertexBuffer` and `meshopt_decodeIndexBuffer`) 不分配内存，并且完全是工作在参数提供的内存上。

All functions have bounded stack usage that does not exceed 32 KB for any algorithms.
所有函数都是栈上的

## gltfpack

meshoptimizer provides many algorithms that can be integrated into a content pipeline or a rendering engine to improve performance. Often integration requires some conscious choices for optimal results - should we optimize for overdraw or not? what should the vertex format be? do we use triangle lists or strips? However, in some cases optimality is not a requirement.

meshoptimizer提供了很多优化算法，可以被集成到一个内容生产工具里或者一个渲染引擎里去提高效率。通常集成都是为了优化结果。可是有时候有效并不是必须的。

For engines that want a relatively simple way to load meshes, and would like the meshes to perform reasonably well on target hardware and be reasonably fast to load, meshoptimizer provides a command-line tool, `gltfpack`. `gltfpack` can take an `.obj` or `.gltf` file as an input, and produce a `.gltf` or `.glb` file that is optimized for rendering performance and download size.
对于引擎来说，有一个相对简单的方式去加载mesh，并且期待三角网在目标硬件上能更快的载入。meshoptimizer提供另一个命令行工具，  `gltfpack`可以把obj或者gltf当作输入，产生一个gltf或者glb，优化后的数据。

To build gltfpack on Linux/macOS, you can use make:

```
make config=release gltfpack
```

On Windows (and other platforms), you can use CMake:

```
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TOOLS=ON
cmake --build . --config Release --target gltfpack
```

You can then run the resulting command-line binary like this (run it without arguments for a list of options):

```
gltfpack -i scene.gltf -o scene.glb
```

gltfpack substantially changes the glTF data by optimizing the meshes for vertex fetch and transform cache, quantizing the geometry to reduce the memory consumption and size, merging meshes to reduce the draw call count, quantizing and resampling animations to reduce animation size and simplify playback, and pruning the node tree by removing or collapsing redundant nodes.
gltfpack 基本上修改了gtlf数据，使用优化后的三角网，做了顶点存取，transform 缓存。quantizing了几何体见笑了额内存占用。合并了所有mesh减少调用次数。quantizing和重采样了动画去减小动画的大小，修建了nodetree 移除或者压缩了冗余节点。

gltfpack can produce two types of output files:

- By default gltfpack outputs regular `.glb`/`.gltf` files that have been optimized for GPU consumption using various cache optimizers and quantization. These files can be loaded by standard GLTF loaders present in frameworks such as [three.js](https://threejs.org/) (r107+) and [Babylon.js](https://www.babylonjs.com/) (4.1+).
默认gltfpack 输出标准的 `.glb`/`.gltf` 文件。这些文件可以被标准的gtlf 加载器加载，比如 [three.js](https://threejs.org/) (r107+) and [Babylon.js](https://www.babylonjs.com/) (4.1+).


- When using `-c` option, gltfpack outputs compressed `.glb`/`.gltf` files that use meshoptimizer codecs to reduce the download size further. Loading these files requires extending GLTF loaders with custom decompression support; `demo/GLTFLoader.js` contains a custom version of three.js loader that can be used to load them.
 如果使用-c 选项，gltfpack输出了压缩后的`.glb`/`.gltf`，使用meshoptimizer编码区减小了下载大小。在demo/gltfloader.js 报了一个自定义版本的three.js的加载器

> Note: files produced by gltfpack use `MESHOPT_quantized_geometry` and `MESHOPT_compression` pseudo-extensions; both of these have *not* been standardized yet but eventually will be. glTF validator doesn't recognize these extensions and produces a large number of validation errors because of this.

注意：gltfpack产生的文件，使用 `MESHOPT_quantized_geometry` and `MESHOPT_compression` 非标准扩展，这些并不是标准的。gltf 验证器可能会产生一些错误。

When using compressed files, `js/meshopt_decoder.js` needs to be loaded to provide the WebAssembly decoder module like this:
当使用压缩文件，`js/meshopt_decoder.js`需要被加载，提供一个WebAssembly 解压缩文件，如下：

```js
<script src="js/meshopt_decoder.js"></script>

...

var loader = new THREE.GLTFLoader();
loader.setMeshoptDecoder(MeshoptDecoder);
loader.load('pirate.glb', function (gltf) { scene.add(gltf.scene); });
```

## License

This library is available to anybody free of charge, under the terms of MIT License (see LICENSE.md).
