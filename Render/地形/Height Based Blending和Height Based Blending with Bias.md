

作为Unity开发者，这是通过 Shader 提升地形质感最立竿见影的技术点。

这两种技术都是为了解决**“线性混合（Linear Blending）”**带来的“幽灵半透明”问题。让我们从原理到代码实现，深度剖析 **Height-Based Blending**（基于高度的混合）以及它的升级版 **带 Bias 的混合**。

---

### 1. 基础版：Height-Based Blending (Hard/Binary)

**核心理念：** **“赢者通吃” (Winner Takes All)**

在线性混合中，如果草地权重0.5，石头权重0.5，结果是 50%草 + 50%石头 = 一团模糊的混合物。
而在基于高度的混合中，Shader 会去读取纹理的**高度图（Height Map）**。它会比较当前像素点上，草的高度和石头的高度。

*   如果 `石头高度 > 草地高度`：**显示 100% 石头**。
*   如果 `石头高度 < 草地高度`：**显示 100% 草地**。

#### 视觉效果
想象你往一堆鹅卵石（高低起伏）上倒沙子。
*   刚开始（沙子权重低）：沙子只出现在鹅卵石的**缝隙（低处）**里。
*   随着沙子变多（权重高）：沙子逐渐填满缝隙，最后盖住石头顶部。
*   **结果：** 材质之间有极其清晰的物理分界线，不会出现半透明的叠加。

#### 算法逻辑 (伪代码)
```glsl
// inputs: height1, height2 (从纹理采样), blend1, blend2 (从Splatmap采样)

// 将Splatmap的权重加到高度上，权重越大，该材质越"高"
float h1 = height1 + blend1; 
float h2 = height2 + blend2;

// 谁高谁赢
if (h1 > h2) {
    return color1;
} else {
    return color2;
}
```

#### 缺点
**边缘过于锐利（Aliasing/Hard Edges）。**
由于是二元判断（非黑即白），材质交界处会有锯齿感，且缺乏光影的过渡，看起来像是一张贴纸剪下来贴在另一张上，不太自然。

---

### 2. 进阶版：Height-Based Blending with Bias (Soft/Gradient)

这是现代 3A 游戏（以及 Unity HDRP/URP Lit Shader）的标准做法。

**核心理念：** **“高度接近时，允许混合”**

为了解决硬边问题，我们引入一个 **Bias（偏移量/过渡区/Depth）** 参数。
*   如果 A 比 B 高出很多：A 赢。
*   如果 B 比 A 高出很多：B 赢。
*   **关键点：** 如果 A 和 B 的高度差在 **Bias** 范围内（比如 0.1 米），说明两者交织在一起，此时在这个微小的范围内进行**线性混合**。

#### 视觉效果
依然保持了“沙子填缝”的物理感，但在沙子和石头接触的那个极细微的边缘，有一层柔和的过渡。这就像现实中物体接触时的**环境光遮蔽（AO）**或者尘土的自然散落，看起来极度真实。

#### 核心算法 (HLSL 实现)
这是你在 Shader Graph 或手写 Shader 中最常用的逻辑（参考自 *Colin Barré-Brisebois* 的经典算法）：

```glsl
float3 BlendHeights(float3 color1, float h1, float3 color2, float h2, float blendWeight1, float blendWeight2, float bias)
{
    // 1. 将混合权重施加到高度上 (这里用加法，也有人用乘法，效果类似)
    float height1 = h1 + blendWeight1;
    float height2 = h2 + blendWeight2;
    
    // 2. 找出最顶层的那个高度
    float maxHeight = max(height1, height2);
    
    // 3. 计算“过渡区底线”：最高点往下减去 bias
    float threshold = maxHeight - bias;
    
    // 4. 计算新的混合权重 based on depth
    // 只有当某一层的高度高于这个 threshold 时，它才有资格参与混合
    // max(..., 0) 保证了低于底线的层权重为 0 (被完全遮挡)
    float w1 = max(height1 - threshold, 0);
    float w2 = max(height2 - threshold, 0);
    
    // 5. 归一化权重 (防止颜色过曝)
    // 哪怕 w1=0.2, w2=0.3 (都很小)，也要让它们加起来等于 1
    float epsilon = 0.0001; // 防止除以0
    float sum = w1 + w2 + epsilon; 
    w1 /= sum;
    w2 /= sum;
    
    // 6. 最终混合颜色
    return (color1 * w1) + (color2 * w2);
}
```

#### 参数 Bias 的作用
*   **Bias = 0:** 退化成第一种 Hard Blending（锐利边缘）。
*   **Bias = 1:** 接近传统的 Linear Blending（模糊）。
*   **通常取值 0.05 ~ 0.2:** 既有清晰的物理堆积感，边缘又足够柔和。

---

### 3. 对比总结

| 特性 | 线性混合 (Linear) | 高度混合 (Height-Based Hard) | 带 Bias 的高度混合 (Height-Based Soft) |
| :--- | :--- | :--- | :--- |
| **原理** | `lerp(a, b, weight)` | `heightA > heightB ? a : b` | 计算高度差，如果在 Bias 内则混合 |
| **视觉** | 幽灵般半透明，糊 | 物理堆积，但边缘像剪纸 | **物理堆积 + 真实柔和边缘** |
| **开销** | 极低 | 低 (主要是分支判断) | 中 (主要是一点点数学运算) |
| **适用** | 远景 LOD，低端手机 | 卡通渲染，或者对边缘要求不高的岩石 | **所有 PC/主机/高端手游的地形** |

### 4. 在 Unity 中的应用

如果你想在 Unity 中实现这个：

1.  **URP / HDRP:**
    *   Unity 的官方 `Lit` Shader 和 `Terrain Lit` Shader 里的 **"Height Blend"** 选项，底层就是 **带 Bias 的算法**。
    *   Bias 参数通常对应界面上的 **"Blend Depth"** 或 **"Transition"** 滑块。

2.  **Shader Graph:**
    *   你可以直接搜索节点 **"Height Blend"**。它封装好了上面的算法，直接把两张图的 Height 和颜色连进去，调节 Contrast/Bias 即可。

3.  **Better Shading (MicroSplat 等插件):**
    *   像 MicroSplat 这种顶级地形插件，默认就是用带 Bias 的高度混合，并且优化得极好（用了 Texture Arrays 减少采样开销）。

### 结论

**带 Bias 的 Height-Based Blending 是目前地形渲染的“版本答案”。**
它以极小的性能代价（几行 ALU 代码），换取了比线性混合强 10 倍的画面表现力。它让地表看起来是**“长”**在一起的，而不是**“画”**在一起的。
