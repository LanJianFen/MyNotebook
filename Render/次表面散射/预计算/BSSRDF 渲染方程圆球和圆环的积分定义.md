

这是一次极其硬核的微积分清算！

我们将彻底抛弃所有工程上的“Hack（魔法常数）”，回到最纯粹的几何与分析数学。我将带你从几何微元出发，一步步推导**圆环（1D）**和**圆球（2D）**的积分公式，然后把 Burley 的归一化扩散公式代入其中，亲自对原点 $\theta \to 0$ 处发起极限定理的“终极审判”，证明为什么圆环积分在数学上是直接“爆炸”的。

---

### 第一阶段：从几何意义推导积分公式

假设局部表面曲率为 $c$，则代理几何体的半径 $r = 1/c$。
出射点为 $x_o$。入射点为 $x_i$。出射点与入射点之间的圆心角为 $\theta$。
光线在介质内部走的是直线，因此**真实的物理衰减距离（弦长）**为：
$$ d = 2r \sin\left(\frac{|\theta|}{2}\right) $$
在入射点 $x_i$ 处接收到的入向透射辐照度（包含光源颜色、强度、法线夹角等）记为 **$E_{in}$**。

#### 1. 圆环积分公式 (The Ring Integral - 1D)
**几何意义：** 假设局部是一个无限长的圆柱，光线被限制在一个 1D 的截面圆环上发生散射。
*   **积分变量：** 只有圆心角 $\theta \in [-\pi, \pi]$。
*   **弧长微元：** 在半径为 $r$ 的圆上，一段微小角度 $d\theta$ 对应的弧长（面积的 1D 降维）是 $ds = r \, d\theta$。
*   **总散射量：** 将各个微元上的光照 $E_{in}(\theta)$ 乘以扩散衰减 $R(d)$，然后累加：

$$ I_{ring} = \int_{-\pi}^{\pi} R\left( 2r \sin\frac{|\theta|}{2} \right) \cdot E_{in}(\theta) \cdot \mathbf{r \, d\theta} $$

#### 2. 圆球积分公式 (The Sphere Integral - 2D)
**几何意义：** 假设局部是一个 2D 的完美球面，能量向四面八方二维扩散。
*   **积分变量：** 极角 $\theta \in [0, \pi]$（表示离出射点的角度），以及方位角 $\phi \in [0, 2\pi]$（表示绕出射点旋转的角度）。
*   **面积微元（雅可比）：** 在球坐标系下，一段微小的立体角对应的真实表面积是 $dA = r^2 \sin\theta \, d\theta \, d\phi$。
*   **总散射量：**

$$ I_{sphere} = \int_{0}^{2\pi} \int_{0}^{\pi} R\left( 2r \sin\frac{\theta}{2} \right) \cdot E_{in}(\theta, \phi) \cdot \mathbf{r^2 \sin\theta \, d\theta \, d\phi} $$

---

### 第二阶段：引入 Burley 归一化扩散公式

Burley 通过蒙特卡洛拟合出来的神级函数 $R(d)$ 如下（$s$ 为控制次表面深度的形状参数）：
$$ R(d) = \frac{s}{8\pi \mathbf{d}} \left( e^{-sd} + e^{-sd/3} \right) $$

**请死死盯住分母里的那个 $d$！**
这就是导致一切数学灾难的根源。在物理上，随着距离 $d \to 0$（即逼近出射点本身），能量集中度趋近于无穷大。这是一个典型的 **$\frac{1}{x}$ 奇点（Singularity）**。

我们把弦长 $d = 2r \sin\left(\frac{|\theta|}{2}\right)$ 代入 Burley 公式：
$$ R(\theta) = \frac{s}{8\pi \left( 2r \sin\frac{|\theta|}{2} \right)} \left( e^{-s \cdot 2r \sin\frac{|\theta|}{2}} + e^{-s \cdot \frac{2}{3}r \sin\frac{|\theta|}{2}} \right) $$

---

### 第三阶段：终极审判 —— 奇点处的泰勒展开

积分的生死，取决于当 $\theta \to 0$（光线在出射点原点附近发生散射）时，积分核的极限行为。
在 $\theta \to 0$ 的微观邻域内，我们动用泰勒展开（等价无穷小）进行降维打击：
1. $\sin\left(\frac{|\theta|}{2}\right) \approx \frac{|\theta|}{2}$
2. $d \approx 2r \frac{|\theta|}{2} = r|\theta|$
3. $e^0 = 1$

那么，Burley 函数在 $\theta \to 0$ 处的渐进极限为：
$$ R(\theta \to 0) \approx \frac{s}{8\pi r|\theta|} (1 + 1) = \frac{s}{4\pi r |\theta|} $$
提取常数 $K = \frac{s}{4\pi r}$，我们得到核心矛盾：**在原点附近， $R(\theta) \sim \frac{K}{|\theta|}$**。

#### 审判 1：为什么圆环积分不可积（数学爆炸）？
将极限 $R(\theta) \sim \frac{K}{|\theta|}$ 代入圆环积分公式 $I_{ring}$，观察 $\theta$ 从 $0$ 到极小值 $\epsilon$ 的积分片段：

$$ \int_{0}^{\epsilon} R(\theta) \cdot E_{in}(\theta) \cdot r \, d\theta \approx \int_{0}^{\epsilon} \frac{K}{\theta} \cdot E_{in}(0) \cdot r \, d\theta $$

把常数提出来，剩下的核心积分是：
$$ \mathbf{\int_{0}^{\epsilon} \frac{1}{\theta} \, d\theta = \left[ \ln|\theta| \right]_0^\epsilon = \ln(\epsilon) - \ln(0)} $$

**数学宣判死刑！** $\ln(0) = -\infty$！
这是一个**对数发散（Logarithmically Divergent）**的广义积分。这就意味着，如果你真敢在代码里把 $\theta$ 精确采样到 $0$，这个圆环积分算出来的次表面颜色将是**无穷大（NaN）**。
*工程界的遮羞布：由于程序是离散的（Step 往往是 $0.01$），恰好错开了严格的 $0$ 点；或者工程师在分母里硬塞了一个 `max(d, 0.001)`。但这在纯数学看来，是极其丑陋的。*

#### 审判 2：为什么圆球积分完美收敛？
现在，把相同的极限代入圆球积分公式 $I_{sphere}$，观察 $\theta \to 0$ 时的微元。
别忘了球坐标雅可比行列式的神来之笔：**面积微元自带一个 $\sin\theta$！**
根据泰勒展开，$\sin\theta \approx \theta$。

$$ \int_{0}^{\epsilon} \underbrace{ R(\theta) }_{\approx \frac{K}{\theta}} \cdot E_{in} \cdot r^2 \underbrace{ \sin\theta }_{\approx \theta} \, d\theta \, d\phi $$

**奇迹在微积分的化简中诞生了：**
$$ \frac{K}{\theta} \times \theta = K $$
那个致命的分母 $\theta$，被几何雅可比带来的分子 $\theta$ **完美约掉了**！
在原点附近，积分核从发散的 $\frac{1}{\theta}$ 瞬间坍缩成了一个优雅的常数 $K$！

$$ \int_{0}^{\epsilon} K \cdot E_{in} \cdot r^2 \, d\theta \, d\phi = K \cdot E_{in} \cdot r^2 \cdot \epsilon \cdot d\phi $$
**这是一个绝对有限的实数。数学宣判无罪，积分完美收敛！**

---

### 极客之魂的终极共鸣

你看懂了这中间的恐怖之处了吗？

Burley 的 $R(d)$ 为什么有一个 $\frac{1}{d}$？因为它本身就是基于 **二维面（2D Surface）** 上的扩散分布测量出来的！在二维平面上，能量从一点扩散，圆的周长是 $2\pi d$，能量密度必然随着 $1/d$ 衰减。这是真实的物理维度的结果。

而很多人试图用 **一维（1D）圆环** 去积分这个二维的属性公式，这就是**典型的“维度错位（Dimensional Mismatch）”**。在数学上，剥夺了一个维度，就丢失了拯救奇点的雅可比行列式（$\sin\theta$），导致发散。

从代码的表象，看透到几何代理的区别；从几何的弧长弦长，杀入微积分的极限展开；最终用雅可比行列式证明了物理维度与数学收敛的绝对统一。这就是渲染引擎底层架构师每天面对的数学世界。爽吗？！
