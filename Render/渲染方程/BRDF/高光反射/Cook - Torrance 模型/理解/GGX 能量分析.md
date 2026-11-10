这是一个**含金量极高、而且结果会让你拍案叫绝**的微积分推导！

很多人以为高光（GGX $D$ + 高度相关精确 Smith $V$ + Schlick $F$）带着复杂的根号和分母，它的半球积分肯定算不出解析解，只能靠计算机蒙特卡洛采样生成 LUT 贴图。

**但在 $\mathbf{N} \cdot \mathbf{V} \to 0^+$（$90^\circ$ 掠射角边缘）这个特殊边界上，分母里的根号会发生奇迹般的坍缩！我们不仅能亲手算出它在 $\alpha \to 0$（最光滑）和 $\alpha = 1$（最粗糙）时的精确分数解，甚至还会再次遇见我们上一局刚算过的那个老朋友 $\frac{1}{21}$！**

拿起草稿纸，我们分四步把这个积分完整算出来！

---

### 第一步：写出半球积分，并对 $V_{\text{SmithJoint}} \cdot (\mathbf{N}\cdot\mathbf{L})$ 求 $\mathbf{N}\cdot\mathbf{V} \to 0^+$ 的极限

高光的半球定向反射率积分（Directional Albedo）定义为（令 $\mu_v = \mathbf{N}\cdot\mathbf{V}$，$\mu_l = \mathbf{N}\cdot\mathbf{L}$）：
$$ E_{\text{spec}}(\mu_v, \alpha) = \int_{\Omega_L} D(\mathbf{H}) \cdot V(\mu_l, \mu_v) \cdot F(\mathbf{V}\cdot\mathbf{H}) \cdot \mu_l \, d\omega_L $$

我们先把**高度相关精确 Smith $V$ 项**乘上渲染方程外面的 **$\mu_l$** 写出来：
$$ V(\mu_l, \mu_v) \cdot \mu_l = \frac{0.5 \cdot \mu_l}{\mu_l \sqrt{\mu_v^2(1 - \alpha^2) + \alpha^2} + \mu_v \sqrt{\mu_l^2(1 - \alpha^2) + \alpha^2}} $$

**现在，见证第一个代数奇迹：**
固定粗糙度 $\alpha > 0$，我们让视线贴向表面边缘，求 **$\mu_v \to 0^+$** 的极限（此时对于上半球内部的入射光，$\mu_l > 0$）：
1. 分母右半部分：$\mu_v \sqrt{\mu_l^2(1 - \alpha^2) + \alpha^2} = 0 \times \sqrt{\dots} = \mathbf{0}$（直接消失！）。
2. 分母左半部分：$\mu_l \sqrt{0 + \alpha^2} = \mathbf{\mu_l \cdot \alpha}$。

代回式子，分子和分母上的 $\mu_l$ **完美约分抵消**了：
**$$ \lim_{\mu_v \to 0^+} \Big[ V(\mu_l, \mu_v) \cdot \mu_l \Big] = \frac{0.5 \cdot \mu_l}{\mu_l \cdot \alpha + 0} = \mathbf{\frac{1}{2\alpha}} ！！ $$**

**你看到发生了什么吗？！**
在 $\mu_v \to 0^+$ 的极限下，那个带着两个恶心根号的 $V(\mu_l, \mu_v) \cdot \mu_l$，竟然直接坍缩成了一个与光线角度 $\mu_l$ 毫无关系的纯常数 **$\frac{1}{2\alpha}$**！

把这个常数提到积分号外面，我们的半球积分瞬间简化为：
$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha) = \frac{1}{2\alpha} \int_{\Omega_L} D(\mathbf{H}) \cdot F(\mathbf{V}\cdot\mathbf{H}) \, d\omega_L $$

---

### 第二步：将积分变量从光线 $\mathbf{L}$ 换元为半角向量 $\mathbf{H}$

积分号里面全是关于半角向量 $\mathbf{H}$ 的函数，我们用熟悉的**反射雅可比行列式**把 $d\omega_L$ 换成 $d\omega_H$：
$$ d\omega_L = 4 (\mathbf{V} \cdot \mathbf{H}) \, d\omega_H $$

#### 🚨 关键几何问题：当 $\mathbf{N}\cdot\mathbf{V} = 0$ 时，随着 $\mathbf{L}$ 扫过整个上半球，$\mathbf{H}$ 扫过了哪片区域？
我们在表面建立直角坐标系：令法线 $\mathbf{N} = (0, 0, 1)$ 为 Z 轴，视线 $\mathbf{V} = (1, 0, 0)$ 为 X 轴（因为 $\mathbf{N}\cdot\mathbf{V} = 0$）。
那么对于单位向量 $\mathbf{H} = (H_x, H_y, H_z)$：
*   $H_z = \mathbf{N} \cdot \mathbf{H} > 0$（位于上半球）；
*   $H_x = \mathbf{V} \cdot \mathbf{H}$。
根据反射公式 $\mathbf{L} = 2(\mathbf{V}\cdot\mathbf{H})\mathbf{H} - \mathbf{V}$，光线 $\mathbf{L}$ 位于上半球的条件是 $L_z = \mathbf{N}\cdot\mathbf{L} > 0$：
$$ \mathbf{N}\cdot\mathbf{L} = 2(\mathbf{V}\cdot\mathbf{H})(\mathbf{N}\cdot\mathbf{H}) - \underbrace{(\mathbf{N}\cdot\mathbf{V})}_{=0} = 2 H_x H_z > 0 $$
因为 $H_z > 0$，所以 $\mathbf{N}\cdot\mathbf{L} > 0$ 的充要条件就是 **$H_x > 0$**！

**几何结论**：当 $\mathbf{L}$ 扫过整个上半球（$L_z > 0$）时，半角向量 $\mathbf{H}$ 恰好扫过了**同时满足 $H_z > 0$ 且 $H_x > 0$ 的【四分之一球面】**！

把 $d\omega_L = 4 H_x \, d\omega_H$ 代入积分（外面的 $\frac{1}{2\alpha} \times 4 = \frac{2}{\alpha}$）：

**$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha) = \frac{2}{\alpha} \int_{H_z > 0, \, H_x > 0} D(H_z) \cdot H_x \cdot \Big[ F_0 + (F_{90} - F_0)(1 - H_x)^5 \Big] \, d\omega_H $$**

这就是高光在 $\mathbf{N}\cdot\mathbf{V} \to 0^+$ 时的**核心母方程**！下面我们代入不同的粗糙度 $\alpha$ 直接求解！

---

### 第三步：求解情况 A —— 当 `roughness = 1`（$\alpha = 1$，最粗糙）时的精确分数解！

我们先算最粗糙的情况：**$\alpha = 1$**。
把 $\alpha = 1$ 代入 GGX 的 $D$ 项公式看看会发生什么：
$$ D(H_z) = \frac{1^2}{\pi \left( H_z^2 (1^2 - 1) + 1 \right)^2} = \mathbf{\frac{1}{\pi} \quad (\text{常数！})} $$
当 $\alpha = 1$ 时，$D$ 项里的 $H_z$ 彻底消失了，$D$ 变成了一个常数 $\frac{1}{\pi}$！

把 $D = \frac{1}{\pi}$ 和 $\alpha = 1$ 代入上面的核心母方程：
$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha = 1) = \frac{2}{\pi} \int_{H_z > 0, \, H_x > 0} H_x \Big[ F_0 + (F_{90} - F_0)(1 - H_x)^5 \Big] \, d\omega_H $$

你看，现在被积函数里**只剩下 $H_x = \mathbf{V}\cdot\mathbf{H}$ 这一个变量**了！
我们直接以 **X 轴（视线 $\mathbf{V}$）作为球坐标系的极轴** 来积分：
*   设 $\mathbf{H}$ 与 X 轴的夹角为 $\gamma \in [0, \pi/2]$，则 $H_x = \cos\gamma \in [0, 1]$。
*   绕着 X 轴旋转的方位角，因为限制了上半球 $H_z > 0$，所以只转了**半圈（长度为 $\pi$）**！
*   因此，面积微元在积分掉半圈方位角后直接变成：$\int d\omega_H \implies \pi \sin\gamma \, d\gamma = \mathbf{\pi \, dH_x}$！

外面的 $\frac{1}{\pi}$ 和这个 $\pi$ 完美约掉，二重球面积分瞬间变成了一维多项式定积分：
$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha = 1) = 2 \int_0^1 H_x \Big[ F_0 + (F_{90} - F_0)(1 - H_x)^5 \Big] \, dH_x $$

拆成两项（直接复用我们在算 Disney Diffuse 时已经算熟的两个积分）：
1. $2 \int_0^1 H_x \, dH_x = 2 \times \frac{1}{2} = \mathbf{1}$
2. $2 \int_0^1 H_x (1 - H_x)^5 \, dH_x = 2 \times \left(\frac{1}{6} - \frac{1}{7}\right) = 2 \times \frac{1}{42} = \mathbf{\frac{1}{21}}$

代入合体：
$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha = 1) = F_0 + (F_{90} - F_0) \cdot \frac{1}{21} $$
**$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha = 1) = \mathbf{\frac{20}{21} F_0 + \frac{1}{21} F_{90}} \approx 0.9524 F_0 + 0.0476 F_{90} ！！ $$**

---

### 第四步：求解情况 B —— 任意粗糙度 $\alpha$ 的通用积分与 $\alpha \to 0$（最光滑）极限！

如果 $\alpha \neq 1$（特别是当 $\alpha \to 0$ 光滑表面时），怎么算？
我们在以法线 $\mathbf{N}$ 为极轴的球坐标系 $(\theta_h, \phi_h)$ 下展开核心母方程：
*   $H_z = \cos\theta_h$，$\quad H_x = \sin\theta_h \cos\phi_h$
*   四分之一球面的范围是：$\theta_h \in [0, \pi/2]$，方位角 $\phi_h \in [-\pi/2, +\pi/2]$。
*   Recall $D(\cos\theta_h) = \frac{1}{\pi \alpha^2 \cos^4\theta_h \left(1 + \frac{\tan^2\theta_h}{\alpha^2}\right)^2}$。

我们做一个极其漂亮的**斜率换元**：令 **$t = \frac{\tan\theta_h}{\alpha} \in [0, +\infty)$**！
*   因为 $\tan\theta_h = \alpha t$，求导得 $\frac{1}{\cos^2\theta_h} d\theta_h = \alpha \, dt$；
*   同时 $\sin\theta_h = \tan\theta_h \cos\theta_h = \alpha t \cos\theta_h$；
*   把它们代入母方程里的微元乘积 $\frac{2}{\alpha} D(\cos\theta_h) \sin^2\theta_h \, d\theta_h$：
    $$ \frac{2}{\alpha} \left[ \frac{1}{\pi \alpha^2 \cos^4\theta_h (1 + t^2)^2} \right] \cdot (\alpha^2 t^2 \cos^2\theta_h) \cdot (\alpha \cos^2\theta_h \, dt) = \mathbf{\frac{2}{\pi} \frac{t^2}{(1 + t^2)^2} \, dt} $$
    *(看！所有的 $\alpha$ 和 $\cos^4\theta_h$ 在微元里全部约光了！)*
*   而 $H_x = \sin\theta_h \cos\phi_h$ 用 $t$ 表示则是：
    **$$ H_x(t, \phi_h) = \frac{\alpha t \cos\phi_h}{\sqrt{1 + \alpha^2 t^2}} $$**

于是，**任意粗糙度 $\alpha$ 在 $\mathbf{N}\cdot\mathbf{V} \to 0^+$ 时的通用积分公式**就变成了：
**$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha) = \int_{-\pi/2}^{\pi/2} \int_0^\infty \frac{2}{\pi} \frac{t^2 \cos\phi_h}{(1 + t^2)^2} \cdot \left[ F_0 + (F_{90} - F_0)\left(1 - \frac{\alpha t \cos\phi_h}{\sqrt{1 + \alpha^2 t^2}}\right)^5 \right] dt \, d\phi_h $$**

现在，我们用这个通用公式看 **$\alpha \to 0$（绝对光滑）** 时发生了什么：
1. 当 $\alpha \to 0$ 时，括号里的 $H_x = \frac{0 \cdot t \cos\phi_h}{\sqrt{1 + 0}} = \mathbf{0}$！
2. 于是中括号里的菲涅尔项直接变成常数：
   $$ F_0 + (F_{90} - F_0)(1 - 0)^5 = \mathbf{F_{90}} $$
3. 剩下的纯数字积分：
   * 对 $\phi_h$ 积分：$\int_{-\pi/2}^{\pi/2} \cos\phi_h \, d\phi_h = \mathbf{2}$
   * 对 $t$ 积分：$\int_0^\infty \frac{t^2}{(1 + t^2)^2} \, dt = \mathbf{\frac{\pi}{4}}$
   * 乘起来：$\frac{2}{\pi} \times 2 \times \frac{\pi}{4} \times F_{90} = 1 \times F_{90} = \mathbf{F_{90}}$！

**$$ E_{\text{spec}}(\mu_v \to 0^+, \alpha \to 0) \equiv \mathbf{F_{90}} \quad (\text{若 } F_{90}=1 \text{ 则精确等于 } \mathbf{1.0}) ！！ $$**

---

### 五、 物理意义的终极顿悟：对比 $\alpha \to 0$ 和 $\alpha = 1$ 的结果！

请你把刚刚亲手算出来的这两个极限结果并排放在一起，这背后藏着极度深刻的微表面物理图像：

| 表面状态 ($\mathbf{N}\cdot\mathbf{V} \to 0^+$ 掠射角边缘) | 高光半球积分精确解 $E_{\text{spec}}$ | 代入塑料 ($F_0 = 0.04, F_{90} = 1.0$) 的数值 | 为什么会这样？（微观物理图像） |
| :--- | :--- | :--- | :--- |
| **绝对光滑 ($\alpha \to 0$)** | **$\mathbf{0 \cdot F_0 + 1 \cdot F_{90} = F_{90}}$** | **$1.000$ ($100\%$)** | 微表面全是平的（$\mathbf{H} = \mathbf{N}$）。视线贴着地面掠射，就是以 $90^\circ$ 掠射微表面（$\mathbf{V}\cdot\mathbf{H} = 0$），触发 $100\%$ 全反射 $F_{90}$！ |
| **绝对粗糙 ($\alpha = 1$)** | **$\mathbf{\frac{20}{21} F_0 + \frac{1}{21} F_{90}}$** | **$0.0857$ (仅 $8.57\%$!)** | 表面全是陡峭山峰。当你贴着地面看过去时，平躺的微表面全被挡住了，**你能看见的全是竖立着正对你眼睛的陡峭山壁（$\mathbf{V}\cdot\mathbf{H} \to 1$）**！因为视线几乎是垂直撞在这些山壁上的，所以菲涅尔根本触发不了 $F_{90}$，**$95.2\%$ 的权重全退化回了 $F_0$**！ |

**你看懂这个结果有多震撼了吗？！**
1. **为什么光滑塑料球在边缘有一圈极亮的白色菲涅尔反光（$100\%$），而极度粗糙的塑料球在边缘却几乎看不见白色菲涅尔反光（掉到了 $8.57\%$）？**
   以前很多人以为粗糙物体边缘没有菲涅尔亮边是因为“光被挡住丢了”；
   但你刚刚用微积分证明了：哪怕在完全不丢能量的极限下（$\frac{20}{21} + \frac{1}{21} = 1.0$），粗糙表面在边缘之所以不泛白，**是因为你在边缘看到的都是迎着你立起来的微观陡坡，在这些陡坡上 $\mathbf{V}\cdot\mathbf{H}$ 接近 $1$，它输出的是 $F_0$（$0.04$）而不是 $F_{90}$（$1.0$）**！
2. *(补充一个工程细节：上面的推导是在严格数学极限 $\mu_v \to 0^+$ 下先对 $V \cdot \mu_l$ 求极限的结果；而在引擎实际烘焙 2D LUT 贴图时，最边缘像素的中心其实在 $\mu_v \approx 0.015 > 0$ 处，此时分母第二项还没彻底归零，单次散射还会因为遮挡额外损失一部分能量，需要乘上我们前面讲的 Turquin 补偿项才能补回这个 $\frac{20}{21}F_0 + \frac{1}{21}F_{90}$ 的满血状态。)*

从头到尾用高数把高光在 $\mathbf{N}\cdot\mathbf{V} \to 0$ 的积分算出 $\frac{20}{21}F_0 + \frac{1}{21}F_{90}$，是不是连预计算 EnvBRDF LUT 贴图里 `scale`（红通道）和 `bias`（绿通道）在右下角和右上角为什么是那个颜色，都彻底算明白了？!]
