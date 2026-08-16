物体包围盒的最近深度怎么算，是光栅化以后遍历每个像素吗

**绝对不是！千万不能遍历像素！**

如果为了算个包围盒深度还要去光栅化（遍历像素），那 Hi-Z 就没有任何性能优势了，甚至比直接画还要慢。

Hi-Z 剔除之所以快，是因为它**纯粹是数学运算（几何计算）**，完全不涉及光栅化。

我们只需要对包围盒的 **8 个顶点** 进行矩阵变换，就能算出“最近深度”。

---

### 1. 核心逻辑：8点求极值

一个立方体包围盒（AABB 或 OBB）只有 8 个角（Corner）。
不管这个物体在屏幕上占了 10 万个像素还是 5 个像素，只要算这 **8 个点** 就够了。

#### 步骤一：拿到 8 个顶点的坐标
假设物体的包围盒中心是 `Center`，半长宽高是 `Extents`。
我们可以轻易算出 8 个顶点的模型空间坐标（比如 `Center + Extents`，`Center - Extents` 等）。

#### 步骤二：投影变换 (MVP Matrix)
我们需要把这 8 个点，从 **模型空间 (Local Space)** 转换到 **裁剪空间 (Clip Space)**。
*   这就用到著名的 MVP 矩阵（Model-View-Projection Matrix）。
*   **运算量**：做 8 次 `Matrix4x4 * Vector4` 的乘法。这对 GPU（哪怕是 CPU）来说简直是沧海一粟，几十个时钟周期就搞定了。

#### 步骤三：透视除法 (Perspective Divide)
得到的坐标是齐次坐标 `(x, y, z, w)`。
真正的深度值（NDC Z）通常是 `z / w`。
*   我们需要算出这 8 个点的 `z/w` 值。

#### 步骤四：找最小值 (Min Z)
在渲染管线中，通常 **0.0 (或 -1.0)** 是最近（近裁剪面），**1.0** 是最远（远裁剪面）。（具体看图形API是 OpenGL 还是 DirectX/Vulkan）。
我们假设 **越小越近**（Reverse-Z 则是越大越近）。

*   **结果**：`Closest_Depth = min(v0.z, v1.z, ... v7.z)`。

**这就是我们要找的“最近深度”。**

---

### 2. 特殊情况：穿插近裁剪面 (Near Plane Clipping)

有一个极其重要的“坑”需要处理：**如果相机已经钻到包围盒里面去了，或者包围盒的一部分在相机后面怎么办？**

*   这时候，哪怕算出来的 8 个顶点的 Min Z，也不代表物体最近的地方。
*   因为物体离相机最近的地方，其实就是相机的镜片（近裁剪面 Near Plane）本身！

**修正算法：**
1.  检查包围盒是否与视锥体的近平面（Near Plane）相交。
2.  如果不相交：直接取 8 个顶点的 Min Z。
3.  如果相交（或者相机在包围盒内部）：**直接把最近深度设为近裁剪面的深度（通常是 0 或 1）**。因为这个物体已经怼到脸上来了，这是它所能达到的最近距离。

---

### 3. 代码层面的伪代码 (Compute Shader)

在 Hi-Z 的 Compute Shader 中，这段逻辑通常是这样写的：

```glsl
// 1. 获取包围盒的中心和范围
float3 center = BoundsBuffer[id].center;
float3 extents = BoundsBuffer[id].extents;

// 2. 构造8个角 (这里简化写)
float3 corners[8];
corners[0] = center + float3( extents.x,  extents.y,  extents.z);
corners[1] = center + float3(-extents.x,  extents.y,  extents.z);
// ... 剩下6个点

// 3. 初始化最小深度为最远
float minDepth = 1.0; // 假设1.0是最远

// 4. 遍历8个点，投影并找最近
for(int i=0; i<8; i++)
{
    // MVP 矩阵变换到 Clip Space
    float4 clipPos = mul(ViewProjectionMatrix, float4(corners[i], 1.0));
    
    // 如果点在近裁剪面后面(w < near)，说明可能穿插了
    // 这是一个简化判断，严谨的需要做视锥裁剪判断
    
    // 透视除法算 NDC 深度
    float depth = clipPos.z / clipPos.w;
    
    // 更新最小值
    if(depth < minDepth) minDepth = depth;
}

// 5. 现在的 minDepth 就是这个物体的“最近深度”
// 拿这个值去和 Hi-Z 纹理比对
```

---

### 4. 总结

*   **没有光栅化**：不画任何像素。
*   **纯代数运算**：只算 8 个顶点的矩阵乘法。
*   **极快**：即便是处理 10,000 个物体，也只需要做 80,000 次简单的向量运算，现代显卡做这个就像呼吸一样简单。

所以 Hi-Z 的流程是：
1.  **算 8 个点** -> 得到物体深度（几何阶段）。
2.  **查 Hi-Z 纹理** -> 得到遮挡深度（纹理采样阶段）。
3.  **比大小** -> 决定生死。
