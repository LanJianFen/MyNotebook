

太棒了！你能主动要求推导这一部分，说明你已经不满足于“知道它能这么做”，而是要“亲眼看着它在数学上发生”。

将 12 Pass 压缩到 2 Pass，是整个 SSSS 算法中最精彩的**“跨学科魔法”**。在这里，我们将告别**微积分（Calculus）**，正式踏入**线性代数（Linear Algebra）**与**数值最优化（Numerical Optimization）**的最高殿堂。

请准备好，我们将一步步撕开这个“秩一近似（Rank-1 Approximation）”的硬核推导过程。

---

### 第一步：12 Pass 的代数死局（为什么不能直接合并？）

让我们回到 d'Eon 2007 年的 12 Pass 起点。
物理上的漫反射轮廓，用 $K$（通常 $K=6$）个二维高斯函数叠加来表示：
$$ R(x, y) \approx \sum_{k=1}^6 w_k \Big( G_{1D}(x, \sigma_k) \cdot G_{1D}(y, \sigma_k) \Big) $$

如果我们要算完整的 2D 卷积，公式是：
$$ M_{final}(x_o) = \sum_{k=1}^6 w_k \left[ G_{1D}(\sigma_k) *_x \Big( G_{1D}(\sigma_k) *_y E_{in} \Big) \right] $$
*(注：$*_x$ 和 $*_y$ 代表在水平和垂直方向的一维卷积。)*

因为有 6 个高斯函数，所以你需要横着扫 6 次，竖着扫 6 次，总共 **12 个 Pass**。

**【梦幻的贪心想法】：**
如果我们把这 6 个 1D 高斯函数先加起来，变成一个“超级 1D 核” $a(x)$，不就能 2 个 Pass 搞定了吗？
设 $a(x) = \sum_{k=1}^6 \sqrt{w_k} G_{1D}(x, \sigma_k)$。

**【数学的无情打脸】：**
如果你用 $a(x) \cdot a(y)$ 展开，你会得到：
$$ a(x) \cdot a(y) = \left(\sum_{k=1}^6 \sqrt{w_k} G(x, \sigma_k)\right) \left(\sum_{j=1}^6 \sqrt{w_j} G(y, \sigma_j)\right) $$
展开后不仅包含 $G(x, \sigma_1) G(y, \sigma_1)$（我们要的），还包含了大量类似 $G(x, \sigma_1) G(y, \sigma_2)$ 的**交叉项（Cross Terms）**！

在线性代数中，这叫做：**“多维正态分布的线性组合，绝对不是可分离的！”**
只要 $\sigma_k$ 不一样，这 12 个 Pass 就仿佛被焊死了一样，代数上**绝无可能**进行精确合并。

---

### 第二步：降维打击 —— 从“连续函数”跌入“矩阵空间”

既然代数公式走不通，Jimenez 决定动用计算机科学最暴力的武器：**离散化（Discretization）**。

我们将原本连续的 2D 函数 $R(x, y)$，在一个 $N \times N$ 的像素网格上采样，变成一个 $N \times N$ 的庞大实数矩阵 $\mathbf{R}$：
$$ \mathbf{R} = \begin{bmatrix} R(-n,-n) & \dots & R(-n,n) \\ \vdots & \ddots & \vdots \\ R(n,-n) & \dots & R(n,n) \end{bmatrix} $$

根据高斯分离性质，矩阵 $\mathbf{R}$ 其实是 6 个秩为 1（Rank-1）的矩阵的加权和。设 $\mathbf{g}_k$ 是第 $k$ 个高斯在 1D 网格上的列向量：
$$ \mathbf{R} = \sum_{k=1}^6 w_k (\mathbf{g}_k \mathbf{g}_k^T) $$

**【核心洞察】：**
在线性代数中，由 6 个线性无关的列向量外积（Outer Product）相加得到的矩阵 $\mathbf{R}$，它的**矩阵的秩（Rank）等于 6**。
而如果我们要用 2 个 Pass（一横一竖）搞定，意味着我们必须找到一个唯一的 1D 列向量 $\mathbf{a}$，使得 $\mathbf{R}_{approx} = \mathbf{a} \mathbf{a}^T$。
**而一个列向量乘以自己的转置，其结果矩阵的秩，必须绝对等于 1！**

这就是 12 Pass 压缩到 2 Pass 的数学本质：**在 $N \times N$ 的高维空间里，寻找一个秩为 1 的矩阵 $\mathbf{a} \mathbf{a}^T$，去逼近一个秩为 6 的矩阵 $\mathbf{R}$！**

---

### 第三步：奇异值分解（SVD）与 Eckart-Young-Mirsky 定理

如何寻找最好的 Rank-1 近似？线性代数中有一颗皇冠上的明珠——**奇异值分解（Singular Value Decomposition, SVD）**。

我们将 $\mathbf{R}$ 进行 SVD 展开：
$$ \mathbf{R} = \mathbf{U} \mathbf{\Sigma} \mathbf{V}^T $$
因为 $\mathbf{R}$ 是一个圆对称（Radially Symmetric）且正定的次表面散射核，所以 $\mathbf{U} = \mathbf{V}$。它的特征展开为：
$$ \mathbf{R} = \lambda_1 \mathbf{u}_1 \mathbf{u}_1^T + \lambda_2 \mathbf{u}_2 \mathbf{u}_2^T + \dots + \lambda_6 \mathbf{u}_6 \mathbf{u}_6^T $$
*(其中 $\lambda$ 是特征值，按从大到小排列：$\lambda_1 \ge \lambda_2 \dots \ge \lambda_6$)*

**【核能预警：Eckart-Young-Mirsky 定理】**
该定理证明了：要用一个秩为 1 的矩阵去近似 $\mathbf{R}$，并且让它们之间的误差（Frobenius 范数）最小，**唯一的、最优的数学解**就是无情地砍掉后面 5 个特征值，只保留最大的第一个！

$$ \mathbf{R}_{approx} = \lambda_1 \mathbf{u}_1 \mathbf{u}_1^T $$

现在，我们把 $\lambda_1$ 拆开，吸收进向量里：
令 $\mathbf{a} = \sqrt{\lambda_1} \, \mathbf{u}_1$。

奇迹诞生了！
$$ \mathbf{R} \approx \mathbf{R}_{approx} = \mathbf{a} \mathbf{a}^T $$
这个从 SVD 中剥离出来的、长度为 $N$ 的一维向量 $\mathbf{a}$，就是我们在 Shader 里用到的**最终 1D 卷积核！**

---

### 第四步：物理的修正（非线性最优化）

数学上的 SVD 虽然找到了最小二乘误差（L2 Error）的最优解，但 Jimenez 发现直接把 SVD 的结果放到渲染器里，画面还是有问题。

**为什么？因为人眼和物理光学的特性，不符合 L2 误差！**
1. **能量不守恒：** 砍掉后面 5 个特征值后，光能丢失了，核的总和不再等于 1。画面会变暗。
2. **中心失真：** SSS 最重要的视觉特征在中心峰值附近（也就是光晕的最亮处），但 SVD 是一视同仁的，这会导致中心过曝或模糊。

所以，SVD 只是提供了一个极佳的**初始猜测（Initial Guess）**。
Jimenez 在离线生成工具里，使用了 **非线性最小二乘法（如 Levenberg-Marquardt 算法）**，重新定义了目标代价函数（Cost Function）：

$$ \min_{\mathbf{a}} \sum_{i, j} W(i, j) \cdot \Big| R(i, j) - a(i) \cdot a(j) \Big|^2 $$

**关键绝杀：权重掩码 $W(i, j)$**
他给中心的像素赋予了极高的权重，边缘像素赋予较低权重，并加上了 $\sum a(i) = 1$ 的硬性约束。

经过 CPU 几千次的迭代下降，最终吐出了一个由 15 到 25 个浮点数组成的“神之数组”。这个数组既满足 1D 可分离，又在人类视觉上无限逼近 6 个高斯核的叠加效果。

---

### 终极复盘：星形伪影的数学原罪

现在我们把推导的闭环合上。你还记得上一课我提到的“星形伪影（Star Artifacts）”吗？现在用刚才的数学知识，你可以一眼看穿它的本质。

误差矩阵 $\mathbf{E} = \mathbf{R}_{real} - \mathbf{a} \mathbf{a}^T$。
因为我们用秩一矩阵去逼近了圆对称矩阵，这个近似在 $X$ 轴和 $Y$ 轴上是极度精确的（因为是一横一竖扫出来的），但是在**对角线方向（45度角）**上，误差 $\mathbf{E}$ 达到了最大！

这种代数上的不对称，投影到 2D 屏幕上，就让原本圆润的次表面光晕，在 45 度角的地方凹陷了进去，变成了一个尖锐的**四芒星（Cross-Star）**。

---

### 总结

这就是 12 Pass 压缩到 2 Pass 的完整数理推导：

1. **起点**：物理 BSSRDF 是一个 Rank-6 的矩阵，必须 12 步分离。
2. **破局**：将其强制离散化为网格矩阵。
3. **降维**：利用 **奇异值分解 (SVD)** 和 **Eckart-Young 定理**，强行提取出主特征向量（Rank-1）。
4. **修正**：利用**非线性梯度下降**，代入视觉权重 $W$，修正特征向量的中心与能量守恒。
5. **落地**：得到唯一的一个 1D 数组 $\mathbf{a}$。Shader 中只需一次横向 Pass 配合 $\mathbf{a}$，一次纵向 Pass 配合 $\mathbf{a}$，2 Pass 完美收官。

至此，你见证了微积分、线性代数和数值优化这三门学科，在计算机图形学里最为水乳交融、最暴力但也最优雅的一次演出！
