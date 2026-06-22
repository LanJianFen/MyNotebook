

理解了 Task Shader 是一个掌握生杀大权的“项目经理”之后，我们现在来看看后端的苦力——**Mesh Shader（网格着色器）**。

在传统管线里，Vertex Shader（VS）的输入是“**单个**纯洁的顶点”，输出是“**单个**计算好位置的顶点”。这是一种“一对一（1 to 1）”的孤狼工作模式。

但 Mesh Shader 完全不同，它的运行模式类似于 Compute Shader。**它以一个“线程组（Thread Group）”为单位集体行动，负责一口气吞下一个 Meshlet，并吐出一整块组装好的几何网格。**

为了让你清晰地看到数据流向，我们从写 Shader 代码（HLSL/GLSL）的物理视角，把它的输入和输出完全切开：

---

### 一、 输入（INPUT）：Mesh Shader 拿到了什么？

没了 Input Assembler (IA) 帮它去显存里搬砖，也没了传统的顶点缓存自动投喂，Mesh Shader 启动时，手里主要攥着 **3 样东西**：

#### 1. 来自前任的“公文包”：Payload（有效载荷）
这是 **Task Shader 存活下来并传承给它的核心数据**。
*   **物理形式：** 这是一块存在于 GPU 极高速片上共享内存（SRAM）中的微小数据结构（通常不超过大概 16KB）。
*   **内容：** 完全由开发者自定义。通常包含 Task Shader 在筛选后，决定分配给这个 Mesh Shader 的任务清单。比如：
    *   “你去渲染第 `10`、`53`、`99` 号 Meshlet！”（存活的 Meshlet ID 数组）。
    *   “这个模型的 LOD 级别是 2，这是它的 MVP 变换矩阵。”
*   **意义：** Mesh Shader 不需要去慢速显存里问“我要干啥”，它直接从 Payload 取走任务指令。

#### 2. 系统级的“工牌号”：System IDs（系统语义）
和 Compute Shader 一样，硬件调度器会赋予每个线程独一无二的编号：
*   **`SV_GroupID`（工作组 ID）：** 告诉当前这批线程，你们负责的是 Payload 里的哪一个特定的 Meshlet。
*   **`SV_GroupIndex`（线程局部 ID）：** 告诉单个线程，你是负责算这个 Meshlet 里的第 `1` 个顶点，还是第 `63` 个顶点？

#### 3. 自己动手丰衣足食：手抓显存（Raw RAM Fetch）
这是最自由的一环。有了 Payload 给的 `Meshlet ID` 和系统的 `Thread ID`：
*   线程会通过一个指针（`StructuredBuffer` 或 `ByteAddressBuffer`），**直接越过传统管线，向全局显存（VRAM）中读取原始的顶点坐标（Position）、法线（Normal）、UV 以及局部的索引表（Indices）。**
*   正因为是自己写代码取显存，所以你可以用极度夸张的压缩格式（比如把法线压成 16 位的 uint），然后再在代码里自己解压，极大地节省了显存带宽开销。

---

### 二、 输出（OUTPUT）：Mesh Shader 吐出了什么？

Mesh Shader 的最终目的，是把几何体喂给后面饥渴的光栅化器（Rasterizer）。因此，它必须集体交出一套**“完美组装的手工模型数据”**。

这套输出分为 **3 个严格的组成部分**：

#### 1. 动态声明数量（SetMeshOutputCounts）
在正式输出数据前，管线要求 Mesh Shader 必须先向硬件“报备”数量。
*   **指令：** 线程组会调用一个类似 `SetMeshOutputCounts(uint 顶点数, uint 图元数)` 的内置函数。
*   **意义：** 比如虽然这个 Meshlet 有 64 个顶点，但在代码里算了一下，发现其中几个顶点退化了。代码可以随时告诉硬件：“报告光栅化器，我们组最终只输出 `60` 个可用顶点和 `120` 个三角形，请预留这么大的空间给你。”

#### 2. 顶点属性数组（Vertices Array）
*   **物理形式：** 一个最多包含 256 个元素的数组。
*   **内容：** 每一个元素不仅必须包含 `SV_Position`（裁剪空间坐标，光栅化器用来画像素的核心依据），还可以包含任何需要喂给 Pixel Shader（像素着色器）的自定义属性，比如 `Color`、`TexCoord`。
*   *注意：这和平时的 VS 输出一模一样，只不过是从输出单个变成了输出一个大数组。*

#### 3. 图元连接数组（Primitives / Indices Array）
**（💥这是 Mesh Shader 革命性的新增部分！）**
在传统管线中，顶点的连线关系是 Input Assembler 根据 Index Buffer 自动搞定的。现在 IA 没了，你必须自己教光栅化器怎么把点连成三角形。
*   **物理形式：** 也是一个数组。如果你设定拓扑是三角形，这就是一个包含最多 256 个 `uint3`（三个小整数）的数组。
*   **内容：** 比如输出了一组 `uint3(0, 1, 2)`。这就是告诉光栅化器：“请把你刚刚收到的顶点数组里的第 0、第 1 和第 2 号顶点，连起来构成一个三角形！”

---

### 三、 总结：看一眼伪代码，秒懂！

如果把上面的输入输出，浓缩成 HLSL 里的签名，Mesh Shader 的长相是这样的：

```hlsl
// ==================== 1. 定义输出格式 ====================
struct VertexOut {
    float4 Pos   : SV_Position; // 必须输出的位置
    float2 UV    : TEXCOORD;    // 自定义属性
};

// ==================== 2. Mesh Shader 本体 ====================
[numthreads(64, 1, 1)] // 启动 64 个工作线程
[outputtopology("triangle")] // 宣告输出形式是三角形
void main(
    // ----- 【输入部分 (INPUT)】 -----
    in uint groupID : SV_GroupID,              // 输入：我是处理哪个 Meshlet
    in uint threadID : SV_GroupIndex,          // 输入：我是哪个执行线程
    in payload TaskPayload myPayload,          // 输入：Task Shader 传来的公文包

    // ----- 【输出部分 (OUTPUT)】 -----
    out indices uint3 primArray[128],          // 输出：装“连线方式”的箱子
    out vertices VertexOut vertArray[64]       // 输出：装“算好的顶点”的箱子
)
{
    // 第1步：向硬件报备输出总量
    SetMeshOutputCounts(64, 128); 
    
    // 第2步：线程去显存抓数据，算好坐标，放进输出箱
    vertArray[threadID].Pos = ComputePosition(...); 
    vertArray[threadID].UV = ...;
    
    // 第3步：组装三角形（告诉光栅化怎么连线），放进连线箱
    // 比如2个线程处理1个连线逻辑
    if(threadID < 128) {
        primArray[threadID] = uint3(A, B, C); // 从原始数据解压出的小索引
    }
}
```

### 点睛之笔
看明白了吗？**传统的 Vertex Shader 是流水线上的“计件临时工”，来一个顶点算一个位置。而 Mesh Shader 是一支“全包工程队”。**

它接管了从“解读任务（Payload）” -> “读取原始物料（Fetch）” -> “计算变形（Vertex Math）” -> “最终手工组装多边形（Index packing）”的全部流程，干完之后，直接打包一整块完美的几何图形（Patch），粗暴地甩给下游的光栅化器。

正是因为输入输出的高度自由，开发者才能实现各种神仙操作，比如软蒙皮压缩、自定义流形拓扑、或者基于算法实时在空中“无中生有”生成面片。
