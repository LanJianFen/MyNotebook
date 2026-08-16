
---

### 第一步：把积分号内的函数，分别映射到 SH 空间

回看你的公式：
$$ E(\mathbf{n}) = \int_{\Omega} L_i(\omega_i) \cdot \max(\mathbf{n} \cdot \omega_i, 0) d\omega_i $$

积分号里面有两个部分相乘：
1. **环境光函数：** $L_i(\omega_i)$ 
2. **余弦衰减函数（Cosine Lobe）：** $C(\omega_i) = \max(\mathbf{n} \cdot \omega_i, 0)$

既然 SH（$Y_i(\omega)$）是一组**完备的正交基**，那么球面上的**任何**平滑函数，都可以写成这组基底的线性组合（就像把一个 3D 向量拆成 $x\vec{i} + y\vec{j} + z\vec{k}$）：

我们可以把 $L_i$ 拟合（展开）为：
$$ L_i(\omega_i) \approx \sum_{j=0}^{8} l_j Y_j(\omega_i) $$
*（注：这里我用 $j$ 代表 0~8 共 9 个系数的扁平化索引，等价于前面的 $l,m$ 下标，方便你阅读线性代数的形式。）*

同理，我们也可以把余弦衰减函数 $C(\omega_i)$ 拟合展开为：
$$ C(\omega_i) \approx \sum_{k=0}^{8} c_k Y_k(\omega_i) $$

### 第二步：见证奇迹的时刻 —— 积分号的消失

现在，把上面两个展开式，**代回你提问的那个积分公式里**：
$$ E(\mathbf{n}) = \int_{\Omega} \left( \sum_{j=0}^{8} l_j Y_j(\omega_i) \right) \cdot \left( \sum_{k=0}^{8} c_k Y_k(\omega_i) \right) d\omega_i $$

根据微积分的线性性质，我们可以把求和号 $\sum$ 和常数系数（$l_j, c_k$）全部**提取到积分号的外面**：
$$ E(\mathbf{n}) \approx \sum_{j=0}^{8} \sum_{k=0}^{8} l_j c_k \left( \int_{\Omega} Y_j(\omega_i) Y_k(\omega_i) d\omega_i \right) $$

**这里是整个 SH 算法最核心的数学灵魂！**
请看括号里那个积分： $\int_{\Omega} Y_j(\omega_i) Y_k(\omega_i) d\omega_i$
因为 SH 是一组**标准正交基（Orthonormal Basis）**，根据正交函数的微积分定义，它满足克罗内克函数（Kronecker delta, $\delta_{jk}$）：
*   当 $j \neq k$ 时，积分为 **0**。（就像 X 轴点乘 Y 轴等于 0）
*   当 $j = k$ 时，积分为 **1**。（就像 X 轴点乘 X 轴等于 1）

这就意味着，上面那个复杂的 $9 \times 9 = 81$ 项的二次求和中，**所有 $j$ 不等于 $k$ 的交叉项全被干掉了（变成 0）**！只剩下 $j = k$ 的 9 个项。

于是，极其昂贵的球面积分，瞬间坍缩成了一个 $O(1)$ 复杂度的**向量点积**：
$$ E(\mathbf{n}) \approx \sum_{j=0}^{8} l_j \cdot c_j $$

**翻译成大白话就是：两个函数乘积的球面积分，等于它们各自 SH 系数向量的点积。**

### 第三步：法线 $\mathbf{n}$ 去哪了？（球面卷积的引入）

你可能发现了一个问题：“等等，刚才算出来的 $E(\mathbf{n})$ 怎么变成一个常数了？我的法线 $\mathbf{n}$ 怎么不见了？”

这是因为，余弦衰减函数 $C(\omega_i) = \max(\mathbf{n} \cdot \omega_i, 0)$ 是**随着法线 $\mathbf{n}$ 的变化而改变朝向的**。这意味着，对于屏幕上的每一个像素，法线 $\mathbf{n}$ 都不一样，那我们难道要对每个像素都重新计算一遍余弦函数的 SH 系数 $c_j$ 吗？那样 GPU 依然会宕机。

这就是我们必须引入**球面卷积（Funk-Hecke 定理）**的原因。

数学家发现，余弦函数是一个**环带谐和函数（Zonal Harmonics）**，即它是绕着法线 $\mathbf{n}$ 旋转对称的。
对于这种特殊的函数，它指向任意法线 $\mathbf{n}$ 时的 SH 系数 $c_j(\mathbf{n})$，可以由它指向正上方（Z轴）时的常量系数（设为 $A_j$）直接推导出来，公式为：
$$ c_j(\mathbf{n}) = A_j \cdot Y_j(\mathbf{n}) $$
*(注：这里为了便于理解省略了 $\sqrt{\frac{4\pi}{2l+1}}$ 的缩放常数，它被吸收进 $A_j$ 里了)*

我们把这个公式代回到刚才坍缩的点积公式中：
$$ E(\mathbf{n}) \approx \sum_{j=0}^{8} l_j \cdot (A_j \cdot Y_j(\mathbf{n})) $$

我们重新结合一下括号（结合律）：
$$ E(\mathbf{n}) \approx \sum_{j=0}^{8} (l_j \cdot A_j) \cdot Y_j(\mathbf{n}) $$

### 总结：软硬件工程的完美落地

看看最终的这个公式，作为渲染工程师，我们会这么处理：

1. **预计算阶段（离线/CPU/Compute Shader）：**
   环境光的系数 $l_j$ 是已知的，余弦衰减的形状常量 $A_j$ 是固定不变的死常数（比如 0 阶是 3.14，1 阶是 2.09 等）。
   我们在烘焙时，直接把这两个数乘起来，算出一个新的系数： $\hat{L}_j = l_j \cdot A_j$。
   这就相当于我们在频域上完成了一次**模糊操作（卷积）**，把原本锐利的环境光变成了一张“模糊的辐照度环境光”。

2. **运行时阶段（GPU Pixel Shader）：**
   此时积分公式变成了极其简单的：
   $$ E(\mathbf{n}) \approx \sum_{j=0}^{8} \hat{L}_j \cdot Y_j(\mathbf{n}) $$
   GPU 只需要拿到预计算好的 9 个 $\hat{L}_j$（通过 Uniform Buffer 或 3D 纹理），然后根据当前像素的法线 $\mathbf{n}=(x,y,z)$ 计算出 $Y_j(\mathbf{n})$（它就是简单的多项式，比如 $Y_1=y, Y_2=z, Y_3=x$ 等），然后做 9 次乘法加法（MAD 指令）即可。

**回答你的问题：** 我们没有用 SH 去拟合积分 $E(\mathbf{n})$。我们是用 SH 分别拟合了 $L_i$ 和 $\max(\mathbf{n} \cdot \omega_i, 0)$，然后利用**微积分中的基底正交性**消灭了积分号，又利用**环带谐和函数的旋转特性**提取出了法线变量，最终把积分化简成了几个简单的代数乘法。
