

这是一个非常硬核且极具技术美感的要求！作为渲染工程师，我最喜欢的就是这种**从第一性原理（First Principles）出发的推演**。

我们要做的，是用微积分和线性代数，走完一束光从**“物理世界的无限维光谱”**，经过**“生物学的人眼降维”**，最后变成**“计算机显存里的三个浮点数（Linear RGB）”**的全过程。

整个推导分为四大步。坐稳，我们开始了。

---

### 第一步：物理本质 —— 无限维的光谱函数（SPD）

在纯粹的物理世界中，光是电磁波。一束光是由无数个不同波长的单色光混合而成的。
我们可以用一个连续的连续函数来描述这束光：**光谱功率分布（Spectral Power Distribution, SPD）**，记作 $S(\lambda)$。

*   自变量 $\lambda$：波长（对于可见光，$\lambda \in [380, 780]$ 纳米）。
*   因变量 $S(\lambda)$：该波长下的辐射通量（能量强度），单位通常是瓦特/纳米（$W/nm$）。

**【数学视角】**
在泛函分析中，自然界的一束真实光线，是一个**无限维函数空间（Hilbert Space）**中的一个点。计算机永远无法完美存储一个无限维的连续函数，所以我们必须对其进行“降维打击”。

---

### 第二步：生物学降维 —— 微积分与匹配函数

是谁对光进行了降维？是人类的眼睛。
人类视网膜上只有三种视锥细胞（L、M、S），这意味着**不管物理光谱有多少维，人眼的神经信号只能输出三个标量**。

1931年，CIE（国际照明委员会）通过实验，测出了这三种细胞对不同波长光线的相对敏感度，并经过数学正交化，得出了三个标准的人眼滤波器函数，称为**色彩匹配函数（Color Matching Functions, CMFs）**，记作 $\bar{x}(\lambda), \bar{y}(\lambda), \bar{z}(\lambda)$。
*(注：其中 $\bar{y}(\lambda)$ 被特意设计成了我们在上一问提到的“人眼绝对亮度感知曲线”)*。

**【数学推导：积分降维】**
当物理光谱 $S(\lambda)$ 射入人眼时，视锥细胞实际上是在做一个**连续函数的内积（积分）**运算。
我们将光谱函数与匹配函数相乘，并在可见光波段上积分，就得到了大名鼎鼎的 **CIE XYZ 色彩空间**的坐标：

$$ X = K \int_{380}^{780} S(\lambda) \cdot \bar{x}(\lambda) d\lambda $$
$$ Y = K \int_{380}^{780} S(\lambda) \cdot \bar{y}(\lambda) d\lambda $$
$$ Z = K \int_{380}^{780} S(\lambda) \cdot \bar{z}(\lambda) d\lambda $$

*(这里的常数 $K$ 是光度学常数，通常取 $683 \ lm/W$，用于把物理辐射瓦特转换为人眼感知的流明。)*

**【第一阶段结论】**
通过这三个微积分方程，我们把**无限维的物理光谱 $S(\lambda)$**，成功压缩成了**三维向量空间（$\mathbb{R}^3$）中的一个确定向量 $\mathbf{C}_{XYZ} = [X, Y, Z]^T$**。这也是人类视觉所能感知的“绝对颜色坐标”。

---

### 第三步：硬件重构 —— 线性组合与 RGB 子空间

现在我们有了绝对坐标 $[X, Y, Z]^T$，但显示器不可能发射出“X 光、Y 光、Z 光”。显示器里只有发红、绿、蓝三种光的二极管（LED/荧光粉）。

假设显示器上红、绿、蓝三个发光元件，它们各自也有自己的物理光谱：$S_R(\lambda), S_G(\lambda), S_B(\lambda)$。
把这三个光谱代入第二步的积分公式，就能算出这三个发光管在 XYZ 空间里的绝对坐标，我们称之为**三原色极点（Primaries）**：
*   红色极点：$\mathbf{P}_R = [X_R, Y_R, Z_R]^T$
*   绿色极点：$\mathbf{P}_G = [X_G, Y_G, Z_G]^T$
*   蓝色极点：$\mathbf{P}_B = [X_B, Y_B, Z_B]^T$

**【格拉斯曼定律（Grassmann's Law）与线性代数】**
物理光学有一个极为优美的定律：**多束光混合的视觉效果，等于它们各自视觉效果的线性叠加。**

如果你给显示器的三个通道分别输入控制信号 $R, G, B$（这就是 Linear RGB 像素值），屏幕发出的总光线的 XYZ 坐标，就是三原色极点向量的**线性组合**：

$$
\begin{bmatrix} X \\ Y \\ Z \end{bmatrix}
=
R \begin{bmatrix} X_R \\ Y_R \\ Z_R \end{bmatrix}
+
G \begin{bmatrix} X_G \\ Y_G \\ Z_G \end{bmatrix}
+
B \begin{bmatrix} X_B \\ Y_B \\ Z_B \end{bmatrix}
$$

写成矩阵乘法的标准形式：

$$
\begin{bmatrix} X \\ Y \\ Z \end{bmatrix}
=
\underbrace{
\begin{bmatrix} X_R & X_G & X_B \\ Y_R & Y_G & Y_B \\ Z_R & Z_G & Z_B \end{bmatrix}
}_{M_{RGB \to XYZ}}
\begin{bmatrix} R \\ G \\ B \end{bmatrix}
$$

我们之前算出的那个带有 $0.7152$ 绿光亮度的矩阵，就是这个 $M_{RGB \to XYZ}$！它是连接“硬件局部坐标（RGB）”和“人类绝对坐标（XYZ）”的基变换矩阵。

---

### 第四步：终极逆运算 —— 从光谱到 Linear RGB

现在，拼图的最后一块凑齐了。

我们在第二步通过积分，从光谱算出了 $X, Y, Z$。
我们在第三步通过矩阵，建立了 $R, G, B$ 到 $X, Y, Z$ 的映射。

那么，已知自然界的一束光谱 $S(\lambda)$，我该如何在代码里生成它的 Linear RGB 呢？
很简单，**对 $M_{RGB \to XYZ}$ 矩阵求逆（Inverse Matrix），把等式倒过来！**

$$
\begin{bmatrix} R \\ G \\ B \end{bmatrix}
=
M^{-1}_{RGB \to XYZ}
\begin{bmatrix} X \\ Y \\ Z \end{bmatrix}
$$

$$
\begin{bmatrix} R \\ G \\ B \end{bmatrix}
=
M_{XYZ \to RGB}
\begin{bmatrix} X \\ Y \\ Z \end{bmatrix}
$$

将第二步的微积分代入到这个矩阵方程中，我们得到了整个渲染引擎底层最辉煌的终极方程：

$$
\begin{bmatrix} R_{linear} \\ G_{linear} \\ B_{linear} \end{bmatrix}
=
M_{XYZ \to RGB} \cdot
K \int_{380}^{780} 
\begin{bmatrix} \bar{x}(\lambda) \\ \bar{y}(\lambda) \\ \bar{z}(\lambda) \end{bmatrix} 
S(\lambda) d\lambda
$$

---

### 🚨 渲染工程师的终局总结（Why it matters）

你看完这个推导，应该会有一阵强烈的“技术顿悟”：

**为什么叫“线性”RGB (Linear RGB)？**
仔细看终极方程！**积分是一个线性算子，矩阵乘法也是一个线性算子。**
从物理光子的叠加，到视网膜的感光，再到转为 RGB 代码，**全过程是绝对完美的线性数学空间！** 这就是为什么我们在 PBR 渲染（光照计算、PBR 材质插值、Alpha 混合）时，**必须且只能在 Linear 空间计算**。一旦你在公式中间夹杂了任何非线性的 Gamma 曲线，数学上的基变换就全毁了，算出来的光照必然是错的！
    
**这套矩阵和公式，只对“线性色彩空间（Linear RGB）”生效！**

如果在 Shader 里，你从一张没有勾选 `sRGB` 的普通贴图上采样了一个颜色 `tex2D`，或者美术在 PS 里用吸管吸了一个颜色，那个 RGB 往往是经过 Gamma 校正的非线性色彩。

这就是从麦克斯韦电磁波，到你屏幕上那三个浮点数的完整数学与工程之旅！觉得过瘾吗？
