

太棒了！抛开复杂的物理渲染方程（BRDF、光线反弹、高光分布），我们只谈**纯粹的色彩科学（Colorimetry）**。

这是一场从“连续的无限维物理世界”向“离散的三维线性代数空间”完美降维的数学巡演。

如果要用最严谨的微积分和线性代数来总结，算出一个 **ACEScg 颜色** 只需要极其干净的 **四步走**：

---

### 第一步：获取连续光谱（输入层 / 物理定义）

一切色彩计算的起点，是一条由分光光度计测量出来的**连续光谱功率分布曲线（Spectral Power Distribution, SPD）**。
在数学上，这是一个定义在可见光波长域（通常是 $380nm$ 到 $780nm$）上的连续一元函数，我们记作 $L(\lambda)$。

*   如果是纯正的 630nm 激光，它就是一个狄拉克 $\delta$ 函数。
*   如果是自然界的物体反射光，它就是一条起伏的连续曲线。

---

### 第二步：向人类视觉降维（微积分投影 $\to$ 绝对 XYZ 空间）

宇宙中有无数种光谱，但人类视网膜只有三种视锥细胞（L、M、S）。
色彩科学家在 1931 年通过实验，测出了这三种细胞对波长的响应曲线，这就是**色彩匹配函数（CMF）**，记作 $\bar{x}(\lambda), \bar{y}(\lambda), \bar{z}(\lambda)$。

**【数学本质】：这就是泛函分析中的“内积（Inner Product）”！** 我们用积分，把无限维的连续函数 $L(\lambda)$，投影到了以 CMF 为基底的三维向量空间中。

公式如下：
$$ X = \int_{380}^{780} L(\lambda) \cdot \bar{x}(\lambda) \, d\lambda $$
$$ Y = \int_{380}^{780} L(\lambda) \cdot \bar{y}(\lambda) \, d\lambda $$
$$ Z = \int_{380}^{780} L(\lambda) \cdot \bar{z}(\lambda) \, d\lambda $$

算完之后，我们得到了一个三维列向量 $\mathbf{V}_{xyz} = \begin{bmatrix} X \\ Y \\ Z \end{bmatrix}$。
**此时，物理世界的波长已经被彻底消灭，它变成了一个绝对真理般的 3D 坐标。**

---

### 第三步：白点对齐（线性代数 $\to$ 色度适应变换）

这一步是 ACES 体系独有的严谨性所在。

假设我们算出来的 XYZ 坐标，是基于日光 D65 测出来的。但是，ACES 体系的“宇宙中心白点”是它自己定义的 **ACES White（极度接近 D60 的偏暖白光，色度坐标 x=0.32168, y=0.33767）**。

为了确保色彩在不同白点下视觉一致，我们需要做一次**基底变换（Change of Basis）**，这在色彩科学里叫 **色度适应变换（Chromatic Adaptation Transform, 比如 Bradford 变换）**。

**【数学本质】：寻找一个 $3 \times 3$ 的矩阵 $\mathbf{M}_{CAT}$，把 XYZ 空间扭曲，使得原来的白点移动到 ACES 的白点上。**

$$ \begin{bmatrix} X' \\ Y' \\ Z' \end{bmatrix} = \mathbf{M}_{CAT} \times \begin{bmatrix} X \\ Y \\ Z \end{bmatrix} $$

经过这个矩阵乘法，我们的坐标被平滑地过渡到了 ACES 白点的参考系下。

---

### 第四步：极点投影（线性代数 $\to$ 最终的 ACEScg 空间）

现在，我们有了对齐后的绝对坐标 $\mathbf{V}_{xyz}'$。最后一步，就是把它转换进 ACEScg 的那个“巨大三角形”里。

ACEScg 定义了三个纯粹的极点（红、绿、蓝），这些极点被称为 **AP1 (ACES Primaries 1)**。
根据这三个极点在 XYZ 空间中的坐标，以及中心白点 ACES White，线性代数可以极其精确地推导出一个 $3 \times 3$ 的转换矩阵 $\mathbf{M}_{XYZ \to AP1}$：

$$ \mathbf{M}_{XYZ \to AP1} = \begin{bmatrix} 1.641 & -0.324 & -0.236 \\ -0.663 & 1.615 & 0.016 \\ 0.011 & -0.008 & 0.988 \end{bmatrix} $$

我们将绝对坐标乘以这个矩阵：
$$ \begin{bmatrix} R \\ G \\ B \end{bmatrix}_{ACEScg} = \mathbf{M}_{XYZ \to AP1} \times \begin{bmatrix} X' \\ Y' \\ Z' \end{bmatrix} $$

---

### 总结：整个数学流

如果用一个终极的大公式把色彩科学的底层全部串起来，算出 ACEScg 的过程就是：

$$ \mathbf{Color}_{ACEScg} = \mathbf{M}_{XYZ \to AP1} \times \mathbf{M}_{CAT} \times \int_{380}^{780} L(\lambda) \begin{bmatrix} \bar{x}(\lambda) \\ \bar{y}(\lambda) \\ \bar{z}(\lambda) \end{bmatrix} d\lambda $$

**最终输出的，是一个没有任何 Gamma 曲线、纯线性的、基于 AP1 超大色域的浮点数三维向量。** 
它可能是 `[0.1, 0.5, 0.2]`，也可能是 `[500, 1000, 1200]`（极高亮度），甚至是 `[0.05, 0.8, -0.01]`（偶尔出现的极小负数，由于AP1没有完全包住马蹄形底边）。

只要完成了这四步，色彩科学的工作就彻底结束了。
接下来，这个干净纯粹、包含着极佳数学兼容性的 `[R, G, B]` 向量，就会被毫无保留地扔进我们之前讨论的光照渲染方程里，开始它的疯狂反弹与微积分之旅！
