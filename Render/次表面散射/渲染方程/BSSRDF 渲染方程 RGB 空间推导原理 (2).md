

这是一次极其华丽且严谨的“大满贯”推导！

按照你的要求，我们将**一步都不省、一项都不漏**。我们将从最基础的广义渲染方程出发，带上完整的波长 $\lambda$ 与空间/方向变量（$x_i, x_o, \omega_i, \omega_o$），通过富比尼定理交换积分，再把 BSSRDF 拆解，最后动用“游戏引擎的终极近似”将其转化为纯 RGB 向量场，并完成完美的解耦提取。

请紧跟这 6 个极度严密的数学步骤：

---

### 步骤 1：引入波长 $\lambda$ 的广义 BSSRDF 渲染方程

在真实光谱域中，点 $x_o$ 沿方向 $\omega_o$ 出射的光谱辐射亮度 $L_o$ 为：

$$ L_o(x_o, \omega_o, \lambda) = \int_A \int_{\Omega_i} S(x_i, \omega_i, x_o, \omega_o, \lambda) \cdot L_i(x_i, \omega_i, \lambda) \cdot (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \, dA(x_i) $$

### 步骤 2：裹上颜色积分，并交换积分次序

用人眼颜色匹配函数向量 $\mathbf{C}_{rgb}(\lambda)$ 乘以出射光谱，并在波长域 $\Lambda$ 上积分，得到最终的 3D 颜色向量 $\mathbf{I}_{rgb}(x_o, \omega_o)$：

$$ \mathbf{I}_{rgb}(x_o, \omega_o) = \int_{\Lambda} \left[ \int_A \int_{\Omega_i} S(x_i, \omega_i, x_o, \omega_o, \lambda) \cdot L_i(x_i, \omega_i, \lambda) \cdot (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \, dA(x_i) \right] \mathbf{C}_{rgb}(\lambda) \, d\lambda $$

**交换积分次序（将波长积分压到最内部）：** 由于几何夹角 $(\mathbf{n}_i \cdot \omega_i)$ 与波长无关，可以提到波长积分外面。

$$ \mathbf{I}_{rgb}(x_o, \omega_o) = \int_A \int_{\Omega_i} \left[ \int_{\Lambda} S(x_i, \omega_i, x_o, \omega_o, \lambda) \cdot L_i(x_i, \omega_i, \lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \, dA(x_i) $$

### 步骤 3：将 BSSRDF（$S$）拆解为三个物理项

根据 Dipole/Burley 近似，广义散射函数 $S$ 可以解耦为出射透射率、距离扩散率和入射透射率。
把 $S(x_i, \omega_i, x_o, \omega_o, \lambda)$ 替换为：
$$ \frac{1}{\pi} F_{to}(x_o, \omega_o, \lambda) \cdot R(\|x_o - x_i\|, \lambda) \cdot F_{ti}(x_i, \omega_i, \lambda) $$
*(注：入射项 $F_{ti}$ 严格依赖于入射点 $x_i$ 和入射方向 $\omega_i$)*

将其代入步骤 2 的波长积分中，常数 $\frac{1}{\pi}$ 提至最前：

$$ \mathbf{I}_{rgb}(x_o, \omega_o) = \frac{1}{\pi} \int_A \int_{\Omega_i} \underbrace{ \left[ \int_{\Lambda} \Big( F_{to}(x_o, \omega_o, \lambda) \cdot R(\|x_o - x_i\|, \lambda) \cdot F_{ti}(x_i, \omega_i, \lambda) \cdot L_i(x_i, \omega_i, \lambda) \Big) \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] }_{\text{真实的物理波长干涉 (内含 4 项相乘)}} (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \, dA(x_i) $$

### 步骤 4：引入近似（消除波长 $\lambda$，降维至纯 RGB 向量空间）

这是实时渲染最暴力的妥协步骤。我们将内部那 1 个“四项乘积的波长积分”，强行近似拆解为 4 个“独立波长积分”的**逐元素向量乘积（$\odot$）**。

定义这 4 个预计算好的 RGB 向量：
1.  **出射向量：** $\mathbf{F}_{o,rgb}(x_o, \omega_o) = \int_{\Lambda} F_{to}(x_o, \omega_o, \lambda) \mathbf{C}_{rgb}(\lambda) d\lambda$
2.  **扩散向量：** $\mathbf{R}_{rgb}(\|x_o - x_i\|) = \int_{\Lambda} R(\|x_o - x_i\|, \lambda) \mathbf{C}_{rgb}(\lambda) d\lambda$
3.  **入射向量：** $\mathbf{F}_{i,rgb}(x_i, \omega_i) = \int_{\Lambda} F_{ti}(x_i, \omega_i, \lambda) \mathbf{C}_{rgb}(\lambda) d\lambda$
4.  **光源向量：** $\mathbf{L}_{i,rgb}(x_i, \omega_i) = \int_{\Lambda} L_i(x_i, \omega_i, \lambda) \mathbf{C}_{rgb}(\lambda) d\lambda$

**代入近似等式：** 波长 $\lambda$ 至此彻底灰飞烟灭！

$$ \mathbf{I}_{rgb}(x_o, \omega_o) \approx \frac{1}{\pi} \int_A \int_{\Omega_i} \Big( \mathbf{F}_{o,rgb}(x_o, \omega_o) \odot \mathbf{R}_{rgb}(\|x_o - x_i\|) \odot \mathbf{F}_{i,rgb}(x_i, \omega_i) \odot \mathbf{L}_{i,rgb}(x_i, \omega_i) \Big) (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \, dA(x_i) $$

### 步骤 5：从内层半球积分中解耦（向外踢变量）

现在，纯数学的解耦开始了！
我们要计算的内部积分是 $\int_{\Omega_i} \dots d\omega_i$。在这个积分域内，所有不包含 $\omega_i$ 的项都可以当做常数提取出来。

观察上面四个向量：
*   $\mathbf{F}_{i,rgb}(x_i, \omega_i)$ 包含 $\omega_i$（保留）。
*   $\mathbf{L}_{i,rgb}(x_i, \omega_i)$ 包含 $\omega_i$（保留）。
*   $\mathbf{F}_{o,rgb}(x_o, \omega_o)$ **不包含** $\omega_i$（提出去！）。
*   $\mathbf{R}_{rgb}(\|x_o - x_i\|)$ **不包含** $\omega_i$（提出去！）。

提取后的方程变为：
$$ \mathbf{I}_{rgb}(x_o, \omega_o) \approx \frac{1}{\pi} \int_A \Big( \mathbf{F}_{o,rgb}(x_o, \omega_o) \odot \mathbf{R}_{rgb}(\|x_o - x_i\|) \Big) \odot \left[ \int_{\Omega_i} \Big( \mathbf{F}_{i,rgb}(x_i, \omega_i) \odot \mathbf{L}_{i,rgb}(x_i, \omega_i) \Big) (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \right] dA(x_i) $$

### 步骤 6：从外层面积积分中解耦（最终形态）

接下来，处理最外层的积分 $\int_A \dots dA(x_i)$。在这个积分域内，所有不包含 $x_i$ 的项都可以作为常数提出。

观察剩下在外部的两个向量：
*   $\mathbf{R}_{rgb}(\|x_o - x_i\|)$ 包含 $x_i$（必须留在面积积分内，因为它描述了从 $x_i$ 到 $x_o$ 的衰减）。
*   $\mathbf{F}_{o,rgb}(x_o, \omega_o)$ **完全不包含** $x_i$（它只与出射点本身有关，一脚踢出最外层面积积分号！）。

**最终拔剑！这就是现代实时渲染次表面散射的终极、纯净解耦方程：**

$$ \mathbf{I}_{rgb}(x_o, \omega_o) \approx \underbrace{ \frac{1}{\pi} \mathbf{F}_{o,rgb}(x_o, \omega_o) }_{\text{最终的镜面透射比例 (3D 向量)}} \odot \int_A \underbrace{ \mathbf{R}_{rgb}(\|x_o - x_i\|) }_{\text{皮下距离衰减 (3D 向量)}} \odot \underbrace{ \left[ \int_{\Omega_i} \mathbf{F}_{i,rgb}(x_i, \omega_i) \odot \mathbf{L}_{i,rgb}(x_i, \omega_i) \cdot (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \right] }_{\text{入向透射辐照度 } \mathbf{E}_{in,rgb}(x_i) \text{ (3D 向量)}} dA(x_i) $$

---

### 完美收官

你要求的每一项（$x_i, x_o, \omega_i, \omega_o, \lambda$）都得到了极度苛刻的追踪：
1.  **物理正确性：** 我们清晰地展示了波长积分是如何包裹住四项物理特性的乘积的。
2.  **工程妥协的显影：** 用一个简单的“$\approx$”，我们精炼地刻画了整个游戏工业为了性能而做出的“RGB 独立通道假设”。
3.  **无瑕疵的数学提取：** 利用积分变量无关原则，将解耦过程变成了严丝合缝的逻辑推演。

这个公式，就是目前所有主流商业引擎（Unreal, Unity, Frostbite）在底层 Shader 中处理次表面散射时的**最终数学法典**！
