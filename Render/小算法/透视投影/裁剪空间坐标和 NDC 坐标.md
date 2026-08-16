

没问题，眼见为实！把一个具体的代数点带进去算一遍，是彻底理解图形管线数据流转的最好方法。

我们假设在**观察空间（View Space）**中，有一个点 $P$，它的坐标用参数表示为 $(x, y, z)$。
为了能和 $4 \times 4$ 矩阵相乘，我们给它补上齐次坐标 $w=1$，所以输入点是：
$$ P_{view} = \begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix} $$

我们将它与我们刚刚推导出来的**左手系 DX 标准投影矩阵**相乘：

$$
\begin{bmatrix} x_c \\ y_c \\ z_c \\ w_c \end{bmatrix}
=
\begin{bmatrix}
\frac{2n}{r-l} & 0 & -\frac{r+l}{r-l} & 0 \\
0 & \frac{2n}{t-b} & -\frac{t+b}{t-b} & 0 \\
0 & 0 & \frac{f}{f-n} & -\frac{nf}{f-n} \\
0 & 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix}
$$

下面我们执行矩阵的行与列点乘，直接来看结果。

---

### 第一阶段：Clip Space（裁剪空间）

经过矩阵乘法，点 $P$ 变成了裁剪空间坐标 $P_{clip}$。这也是我们在 Vertex Shader（顶点着色器）中通过 `return` 或赋值给 `gl_Position` / `SV_POSITION` 输出的那个值。

我们重点看你最关心的 **Z** 和 **W** 分量：

*   **$x_c$** $= \frac{2n}{r-l}x - \frac{r+l}{r-l}z$
*   **$y_c$** $= \frac{2n}{t-b}y - \frac{t+b}{t-b}z$
*   **$z_c$** $= 0 \cdot x + 0 \cdot y + \frac{f}{f-n}z - \frac{nf}{f-n} \cdot 1 = \mathbf{\frac{f}{f-n}z - \frac{nf}{f-n}}$
*   **$w_c$** $= 0 \cdot x + 0 \cdot y + 1 \cdot z + 0 \cdot 1 = \mathbf{z}$

**工程师观察笔记（Clip Space）：**
	看 $w_c$**：完美！$w_c$ 确确实实地把 $P$ 点原本的真实深度 $z$ 存了进去。

### 为什么叫裁剪空间？
不要在 Clip Space 里死磕，我们往后看一步。
我们知道，Clip Space 的下一步，是除以 $w_c$，变成 NDC 空间。

在 NDC 空间里，视锥体变成了一个完美的**正方体**（DirectX 标准）：
1. 屏幕最左边：$x_{ndc} = -1$
2. 屏幕最右边：$x_{ndc} = 1$
所以 X 必须满足： **$-1 \le x_{ndc} \le 1$**

现在，我们把透视除法的公式 $x_{ndc} = \frac{x_c}{w_c}$ 代入这个不等式：
$$ -1 \le \frac{x_c}{w_c} \le 1 $$

因为我们要渲染的物体都在摄像机前面，所以 $w_c$（也就是 $Z_{view}$）一定是一个**正数**。既然是正数，我们在不等式的三边同时乘以 $w_c$，不等号方向不变：
$$ -w_c \le x_c \le w_c $$

Y 和 Z 也同理

---

### 第二阶段：NDC Space（标准化设备坐标）

点进入光栅化阶段，GPU 硬件强制执行**透视除法（Perspective Divide）**。它将 $x_c, y_c, z_c$ 统统除以 $w_c$（也就是除以 $z$）。

我们来看看除法之后，$P$ 点变成了什么样子：

$$ P_{ndc} = \begin{bmatrix} \frac{x_c}{w_c} \\ \frac{y_c}{w_c} \\ \frac{z_c}{w_c} \end{bmatrix} = \begin{bmatrix} \frac{x_c}{z} \\ \frac{y_c}{z} \\ \frac{z_c}{z} \end{bmatrix} $$

我们具体展开算一下：

*   **$x_{ndc}$** $= \frac{2n}{r-l} \cdot \frac{x}{z} - \frac{r+l}{r-l}$  *(实现了近大远小)*
*   **$y_{ndc}$** $= \frac{2n}{t-b} \cdot \frac{y}{z} - \frac{t+b}{t-b}$  *(实现了近大远小)*

接下来见证奇迹，来看看 **$z_{ndc}$**：
$$ z_{ndc} = \frac{z_c}{z} = \frac{\frac{f}{f-n}z - \frac{nf}{f-n}}{z} $$
把分母的 $z$ 除进去：
$$ \mathbf{z_{ndc} = \frac{f}{f-n} - \frac{nf}{f-n} \cdot \frac{1}{z}} $$

**关于 W 去哪了：**
在纯粹的 3D 几何意义上，NDC 只是一个 $(x, y, z)$ 的三维空间，齐次分量变成了 $w_{ndc} = w_c / w_c = 1$，通常被丢弃。
*（但作为硬件工程师，我必须严谨补充一句：实际上底层的 Rasterizer 会把 $w_c$ 缓存下来。因为在后续算贴图 UV 和顶点颜色的插值时，由于透视变形，普通的线性插值是错的，必须要带上 $1/w_c$ 进行**透视矫正插值 Perspective Correct Interpolation**，这是极为重要的硬件细节。）*

---

### 极值验证（验证公式对不对）

最后，我们拿这个 $z_{ndc}$ 的公式验证一下 DirectX 的规定（[0, 1] 范围）：

1.  **当物体在近裁剪面（带入 $z = n$）**：
    $$ z_{ndc} = \frac{f}{f-n} - \frac{nf}{f-n} \cdot \frac{1}{n} = \frac{f}{f-n} - \frac{f}{f-n} = \mathbf{0} $$
    完美！近处的物体深度存入 Z-Buffer 的值是 0。

2.  **当物体在远裁剪面（带入 $z = f$）**：
    $$ z_{ndc} = \frac{f}{f-n} - \frac{nf}{f-n} \cdot \frac{1}{f} = \frac{f}{f-n} - \frac{n}{f-n} = \frac{f-n}{f-n} = \mathbf{1} $$
    完美！远处的物体深度存入 Z-Buffer 的值是 1。

你看，把参数代进去推演一遍，所有的黑盒就都打开了。你现在看着 $z_{ndc} = \text{常数} - \text{常数} \cdot \frac{1}{z}$ 里面那个极其抢眼的 **$1/z$**，是不是对我们上一个回合讨论的“非线性深度的灾难（Z-Fighting）”有了更直观的代数感受？
