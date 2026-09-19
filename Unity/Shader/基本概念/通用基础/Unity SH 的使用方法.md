

##  数学上的系数对比

如果你去翻看数学书，球谐基函数 $Y_{l}^{m}$ 的原始定义是这样的（非常吓人）：

*   **L0 (Band 0):** $Y_{0}^{0} = \frac{1}{2} \sqrt{\frac{1}{\pi}}$
*   **L1 (Band 1):** $Y_{1}^{-1} = \sqrt{\frac{3}{4\pi}} \cdot y$, $Y_{1}^{0} = \sqrt{\frac{3}{4\pi}} \cdot z$, $Y_{1}^{1} = \sqrt{\frac{3}{4\pi}} \cdot x$

你会发现，每一项都有一个 **恶心的常数因子**（比如 $\sqrt{\frac{3}{4\pi}} \approx 0.4886$）。

#### 1. Unity 的“偷懒”策略（CPU端预计算）

在 Shader（GPU）里做开方和除法是非常昂贵的，而且这些系数 $\pi$ 永远不会变。

所以，Unity 在 **CPU 端（C++层）** 准备数据的时候，做了一个操作：

> **`unity_SHAr` = (原始光照系数 L) × (漫反射卷积系数 A) × (基函数常数 Y_const)**

它把这三个固定值都乘到一起了！

*   **原始定义：** $Lighting = \sum (L_{lm} \times Y_{const} \times N)$
*   **Unity做法：** 令 $\text{PackedCoeff} = L_{lm} \times Y_{const}$
*   **Shader做法：** $Lighting = \text{PackedCoeff} \cdot N$

#### 2. 具体扣出来了什么？

正如你所说，Unity 把公式里**除了 $N$（法线向量）以外的所有东西**都扣出来并由 CPU 提前乘好了。

我们拆解一下 `unity_SHAr`：

#### .xyz 分量 (L1)
*   **数学原本：** $L_{1} \cdot \sqrt{\frac{3}{4\pi}} \cdot N$
*   **Unity 存的：** $L_{1} \cdot \sqrt{\frac{3}{4\pi}}$
*   **Shader 做的：** `dot(Stored, N)` $\rightarrow$ 刚好补上了 $N$。

#### .w 分量 (L0)
*   **数学原本：** $L_{0} \cdot \frac{1}{2\sqrt{\pi}} \cdot 1$
*   **Unity 存的：** $L_{0} \cdot \frac{1}{2\sqrt{\pi}}$
*   **Shader 做的：** `dot(Stored, 1.0)` $\rightarrow$ 刚好补上了 $1$。


## Unity 的系数分组逻辑

**“虽然 3阶 在数学上是 9 个系数，但在 Shader 里是 7 个 float4”**。这段代码正是把分散的内置 Uniform 变量，打包成一个整齐的数组，方便后续进行数学运算（通常是点积）。

我来为你详细拆解这个 `SHco[7]` 里的数据是如何排列的（这在写自定义光照 Shader 时非常重要）：

这个数组总共有 28 个浮点位（7 个 float4 × 4），实际使用了 27 个，刚好放下 RGB 三个通道的 9 个系数。

#### **第一组：SHco[0] ~ SHco[2] (对应 unity_SHAr/g/b)**
这里存放的是 **L0（常数项/环境光亮度）** 和 **L1（线性项/光的方向）**。

*   **SHco[0] (Red通道)**, **SHco[1] (Green通道)**, **SHco[2] (Blue通道)**
*   **`.w` 分量**：存放 **L0** 系数（环境光的基础亮度，$Y_{0,0}$）。
*   **`.xyz` 分量**：存放 **L1** 系数（光照的主要方向，$Y_{1,-1}, Y_{1,0}, Y_{1,1}$）。

> **计算逻辑**：后续计算时，Shader 会拿法线向量 (Normal) 和这三个变量的 `.xyz` 做点积，再加上 `.w`。

---

#### **第二组：SHco[3] ~ SHco[5] (对应 unity_SHBr/g/b)**
这里存放的是 **L2（二次项）的前 4 个系数**。L2 负责描述光照中更细微的明暗变化。

*   **SHco[3] (Red)**, **SHco[4] (Green)**, **SHco[5] (Blue)**
*   这 4 个分量对应 L2球谐基函数中的 4 个形状（$x^2-y^2$, $z^2$, $xy$, $yz$ 等的混合）。

> **计算逻辑**：Shader 会预先计算法线的二次方项（如 $n_x \cdot n_y$ 等），然后和这些变量做点积。

---

#### **第三组：SHco[6] (对应 unity_SHC)**
L2 一共有 5 个系数，前 4 个被上面占了（Group B），**第 5 个系数（$x^2 - y^2$ 相关项）比较特殊，单独放在这里**。

*   因为每个通道只剩 1 个系数了，为了省空间，Unity 把 R、G、B 三个通道的这第 5 个系数 **塞进了同一个 float4** 里：
    *   **`SHco[6].x`** = **Red** 通道的 L2 第5系数
    *   **`SHco[6].y`** = **Green** 通道的 L2 第5系数
    *   **`SHco[6].z`** = **Blue** 通道的 L2 第5系数
    *   `SHco[6].w` = (通常闲置或存放 1.0 用作填充)

---

####  为什么要这么转存一下？

你可能会问：*“为什么不直接用 unity_SHAr，还要赋值给 SHco 数组？”*

主要有三个原因：

1.  **代码复用与函数传参**：
    通常在底层库（比如 `Common.hlsl` 或 `Lighting.hlsl`）里，通用的 SH 计算函数（如 `SampleSH9`）接受的参数类型是 `float4 SH[7]` 数组。把 uniform 变量塞进数组，就能直接传给这些通用函数，不用重写一遍公式。

2.  **GPU 指令优化**：
    在某些旧的硬件或特定的 Shader 编译器（Compiler）优化策略下，对数组进行索引访问（Looping or Indexing）或者连续内存访问，比访问散乱的全局变量更高效。

3.  **支持 Instance / SRP Batcher**：
    如果是自定义的渲染管线，或者使用了 GPU Instancing，这些数据可能不是来自全局变量，而是来自 Constant Buffer 里的数组。定义一个本地数组 `SHco` 可以作为中间层，统一数据来源（无论是来自 Global 还是 Instanced Buffer），让后面的光照计算逻辑保持不变。

####  后续它会怎么被计算？

在赋值完这个数组后，你通常会紧接着看到类似这样的代码（简化版逻辑）：

```csharp
// 1. 准备法线数据
float3 n = normalWS;
float4 vB = n.xyzz * n.yzzx; // 预计算法线的二次项

// 2. 计算 L0 + L1 (基础环境 + 方向)
// 这里的 dot(SHco[0].xyz, n) 计算方向， + SHco[0].w 加基础亮度
float3 res = float3(
    dot(SHco[0].xyz, n) + SHco[0].w, 
    dot(SHco[1].xyz, n) + SHco[1].w, 
    dot(SHco[2].xyz, n) + SHco[2].w
);

// 3. 加上 L2 (更细致的亮部暗部)
// 前4个系数
res.r += dot(SHco[3], vB);
res.g += dot(SHco[4], vB);
res.b += dot(SHco[5], vB);

// 加上第5个系数 (来自 SHC)
float vC = n.x*n.x - n.y*n.y; 
res += SHco[6].xyz * vC;

return max(0, res); // 最终光照结果
```

**总结：**
这行代码是在为 **“最高品质的烘焙/探针环境光计算”** 备料。它把 Unity 乱序存放的 27 个系数整理好，准备通过一系列复杂的点积运算，还原出物体表面那个位置的漫反射环境光。
