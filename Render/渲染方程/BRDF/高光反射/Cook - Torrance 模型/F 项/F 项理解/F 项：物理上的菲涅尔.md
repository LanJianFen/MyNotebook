
### 一、 硬核复习：F 项到底是什么？物理学意义上的 F 是怎么来的？

在正式写现代代码之前，我们把 Schlick 的那套 `(1 - VdotH)^5` 彻底从脑子里忘掉，回到 1823 年的物理学原点。

#### 1. F 项在微表面 BRDF 中的定位
在 Cook-Torrance 微表面模型 $f_r = \frac{D(\mathbf{h}) \cdot G(\mathbf{l}, \mathbf{v}) \cdot F(\mathbf{v}, \mathbf{h})}{4(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})}$ 中：
* $D$ 项和 $G$ 项属于**纯几何与统计学**：它们把凹凸不平的表面拆成无数个微小的、绝对平坦的镜面（微表面），并筛选出那些**法线恰好等于半角向量 $\mathbf{h}$**、且没有被遮挡的微表面。
* **$F$ 项属于纯粹的量子电磁学**：当一束光打在这块法线为 $\mathbf{h}$ 的微小绝对平面上时（入射角余弦为 $\cos\theta_i = \mathbf{v} \cdot \mathbf{h} = \mathbf{l} \cdot \mathbf{h}$），**这束光携带的电磁波辐射功率（Radiant Flux）中，究竟有多大比例被界面当场反弹（反射），多大比例穿透界面进入了物质内部（折射/吸收）？**
* 这个**反射功率与入射功率的绝对比值**，就是 $F \in [0, 1]$。

---

#### 2. 物理学上 F 是怎么推导出来的？（麦克斯韦方程组的边界审判）

在物理学中，光不是一颗颗台球，而是交变的**电磁波**（包含相互垂直振荡的电场向量 $\mathbf{E}$ 和磁场向量 $\mathbf{H}$）。

当光波从介质 1（如空气，折射率 $n_1$）射向介质 2（物体表面，折射率 $n_2$）的分界面时，它撞上了物质内部的原子电子云。为了满足**麦克斯韦方程组（Maxwell's Equations）**，交界面两侧的电磁场必须无条件服从两个**边界连续性条件（Boundary Conditions）**：
1. **电场与磁场的切向分量（平行于表面方向）在跨越界面时必须绝对连续（相等）**：
   $$ E_{1,\text{tan}} = E_{2,\text{tan}}, \quad H_{1,\text{tan}} = H_{2,\text{tan}} $$
2. **电磁波在两种介质中的传播速度不同**（由介电常数 $\epsilon$ 决定，宏观表现为折射率 $n = \sqrt{\epsilon_r}$），导致穿透进去的光波发生了角度偏折，遵循**斯涅尔折射定律（Snell's Law）**：
   $$ n_1 \sin\theta_i = n_2 \sin\theta_t $$
   *(其中 $\theta_i$ 是入射角，$\theta_t$ 是折射进入内部的透射角)*。

**数学矛盾与求解：**
如果只有“入射波”和“折射波”，由于两边折射率不同（阻抗不匹配），你无论如何都无法配平上面那两条切向分量连续的方程！
为了维持电磁场的连续性，**界面被迫激发出第三束波——反向弹回的“反射波”！**

---

#### 3. 偏振拆分与精确菲涅尔方程（Fresnel Equations）

为了解出反射波的振幅，物理学家菲涅尔将任意电磁波的电场振动方向，正交分解为两个独立的偏振分量：
* **S 偏振（s-polarized / TE 波，垂直偏振）**：电场向量 $\mathbf{E}$ 垂直于入射平面（就像贴着水面打水漂的扁石头）。
* **P 偏振（p-polarized / TM 波，平行偏振）**：电场向量 $\mathbf{E}$ 平行于入射平面。

联立边界连续性条件与斯涅尔定律，直接解二元一次方程组，就得到了人类光学史上最伟大的**精确菲涅尔振幅反射系数（$r_s, r_p$）**。
而在渲染中，我们关心的是**能量（功率）**，电磁波的能量正比于电场振幅的模平方（$R = |r|^2$）：

$$ R_s(\theta_i) = \left| \frac{n_1 \cos\theta_i - n_2 \cos\theta_t}{n_1 \cos\theta_i + n_2 \cos\theta_t} \right|^2 $$

$$ R_p(\theta_i) = \left| \frac{n_1 \cos\theta_t - n_2 \cos\theta_i}{n_1 \cos\theta_t + n_2 \cos\theta_i} \right|^2 $$

自然界中的日常光源（太阳、灯泡）通常是**非偏振光（Unpolarized Light）**，意味着 S 偏振和 P 偏振各占一半能量。因此，图形学中无偏振精确菲涅尔反射率 $F$ 就是两者的算术平均：
**$$ F(\theta_i) = \frac{1}{2} (R_s + R_p) $$**

在正版物理公式 $F(\theta_i) = \frac{1}{2}(R_s + R_p)$ 中，代入 $\theta_i = 0^\circ$ 算出来的就是 $F_0$；代入 $\theta_i = 90^\circ$ 算出来的就是 $F_{90}$（在纯粹的物理世界里，光滑表面的 $F_{90}$ 永远恒等于 $1.0$）。

---

#### 4. 物质的两大阵营：为什么金属和绝缘体的 F 截然不同？

把不同物质的折射率代入上面这个精确方程，物理世界立刻分裂成两个阵营：

##### 阵营 A：绝缘体 / 电介质（Dielectrics，如塑料、水、宝石）
* **微观物理**：电子被原子核死死束缚，光波穿入后不会被立刻吸收。
* **折射率**：是一个纯粹的**正实数** $n_2 \in \mathbb{R}$（通常在 $1.3 \sim 2.4$ 之间）。
* **三个关键角度的数学表现（假设空气 $n_1 = 1, n_2 = n$）**：
  1. **正视（$\theta_i = 0^\circ$）**：此时 $\cos\theta_i = \cos\theta_t = 1$。代入公式，$R_s$ 和 $R_p$ 完全相等：
     $$ F_0 = \left( \frac{n - 1}{n + 1} \right)^2 $$
     *(这就是 `F0 = ((ior-1)/(ior+1))^2` 的唯一物理源头！)*
  2. **布儒斯特角（Brewster's Angle）**：当 $\theta_i + \theta_t = 90^\circ$ 时，$R_p$ 的分子变成 $0$！此时反射光变成了 100% 纯净的 S 偏振光（摄影师用偏振镜消除水面反光就是利用这个物理角）。
  3. **掠射角（$\theta_i = 90^\circ$）**：此时 $\cos\theta_i = 0$。代入公式：
     $$ R_s = \left| \frac{0 - n \cos\theta_t}{0 + n \cos\theta_t} \right|^2 = |-1|^2 = 1.0, \quad R_p = \left| \frac{n \cos\theta_t - 0}{n \cos\theta_t + 0} \right|^2 = 1.0 $$
     **无论折射率是多少，在 $90^\circ$ 掠射角时，任何光滑表面都必然发生 100% 的镜面反射（$F_{90} = 1.0$）！**

##### 阵营 B：金属 / 导体（Conductors，如金、银、铝、铁）
* **微观物理**：金属内部充满自由电子海。光波一接触表面，电场驱动自由电子产生高频感应电流，瞬间将透射光波转化为焦耳热吸收掉（趋肤效应），同时激发出极强的反向电磁波。
* **复折射率（Complex IOR）**：因为存在强烈的指数级能量衰减，金属的折射率在麦克斯韦方程中是一个**复数**：
  **$$ \tilde{n}_2 = n + i \kappa $$**
  * 实部 $n$：相速度折射率；
  * 虚部 $\kappa$（有时记作 $k$）：**消光系数（Extinction Coefficient）**，代表对光的吸收强弱。
* **数学表现**：
  1. **正视（$\theta_i = 0^\circ$）**：把复数 $\tilde{n}_2 = n + i\kappa$ 代入 $F_0$ 并求复数模的平方（$|z|^2 = \text{Re}^2 + \text{Im}^2$）：
     **$$ F_{0,\text{metal}} = \frac{(n - 1)^2 + \kappa^2}{(n + 1)^2 + \kappa^2} $$**
     因为金属的消光系数 $\kappa$ 很大（通常在 $2.0 \sim 8.0$），导致分子和分母极其接近，所以金属在正视时的 $F_0$ 高达 $0.6 \sim 0.98$，且不同波长的 $n(\lambda), \kappa(\lambda)$ 差异极大，从而呈现出黄金、紫铜等鲜明的金属本色。
  2. **拟布儒斯特角凹陷（Pseudo-Brewster Angle / Fresnel Dip）**：
     由于虚部 $\kappa$ 的存在，$R_p$ 分量不会在某个角度降到 0，但它依然会在 $75^\circ \sim 85^\circ$（即约 $82^\circ$）附近产生一个明显的**局部波谷（凹陷）**，然后才在 $90^\circ$ 猛烈拉升回 $1.0$。这就是标准 Schlick 公式无法拟合、逼迫现代 OpenPBR 引入 `F82-Tint` 的根本电磁学原因！

---


### 解密几何桥梁 —— 为什么物理的 $\theta_i$ 变成了代码里的 $\mathbf{V} \cdot \mathbf{H}$？

#### 1. $\cos\theta_i$
在中学物理课本里，当我们画菲涅尔反射定律时，画的是一面**宏观的大镜子**。
那面大镜子的法线是宏观法线 $\mathbf{N}$。那时候，入射角 $\theta_i$ 是光线 $\mathbf{L}$ 与宏观法线 $\mathbf{N}$ 的夹角。

**但是！在微表面理论（Cook-Torrance BRDF）里，宏观表面是粗糙的，是由无数个朝向各异的“微观小镜子（Microfacets）”组成的！**

现在，请把显微镜放大一万倍：
1. **谁才是真正发生菲涅尔反射的那面镜子？**
   不是宏观平面 $\mathbf{N}$！
   只有那些**法线恰好等于半角向量 $\mathbf{H}$** 的微观小镜子，才能把从 $\mathbf{L}$ 射过来的光，精准地反射进你的视线 $\mathbf{V}$ 里！
   因此，对于发生反射的那个微观小镜面来说，**它的表面法线就是 $\mathbf{H}$！**

2. **在这个微观小镜子上，入射角 $\theta_i$ 是谁和谁的夹角？**
   入射角是**入射光线 $\mathbf{L}$ 与这面小镜子的法线 $\mathbf{H}$ 之间的夹角**！
   因为小镜面发生的是完美镜面反射（入射角 = 反射角），且 $\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|}$ 是 $\mathbf{L}$ 和 $\mathbf{V}$ 的角平分线，所以：
   **入射角 $\theta_i$ 既等于 $\mathbf{L}$ 与 $\mathbf{H}$ 的夹角，也严格等于视线 $\mathbf{V}$ 与 $\mathbf{H}$ 的夹角！**

3. **三角函数变向量点乘：**
   既然 $\theta_i$ 是单位向量 $\mathbf{V}$ 和单位向量 $\mathbf{H}$ 的夹角，那么根据线性代数点乘定义：
   **$$ \cos\theta_i = \mathbf{V} \cdot \mathbf{H} = \mathbf{L} \cdot \mathbf{H} $$**

**真相大白！**
物理公式里的自变量是角度 $\theta_i$，而在微积分和电磁学方程（回头看上一局的 $R_s$ 和 $R_p$ 公式）中，$\theta_i$ 全都是以 **$\cos\theta_i$** 的形式参与计算的！
所以在微表面着色器里，我们根本不需要求什么反余弦去算角度 $\theta_i$，直接算一个点乘 **`VdotH`**，拿到的就正是微观小镜面上的 **$\cos\theta_i$**！

*   当正对着小镜面看时：$\theta_i = 0^\circ \implies \cos(0^\circ) = 1 \implies \mathbf{V} \cdot \mathbf{H} = 1$。
*   当贴着小镜面边缘掠射时：$\theta_i = 90^\circ \implies \cos(90^\circ) = 0 \implies \mathbf{V} \cdot \mathbf{H} = 0$。


#### 2. $\cos\theta_t$ 怎么不见了？（斯涅尔定律的“内部刺杀”）

**答案是：$\cos\theta_t$ 从来就不是一个独立的变量！在物理学中，只要确定了入射角 $\cos\theta_i$ 和折射率 $n$，折射角 $\cos\theta_t$ 就已经被唯一锁死了！**

我们在写精确菲涅尔方程时，之所以保留 $\cos\theta_t$，纯粹是为了让公式看起来对称、好看。如果我们要写代码，第一步就是用**斯涅尔折射定律（Snell's Law）**配合**三角恒等式**，把 $\cos\theta_t$ 当场消灭掉！

我们来亲手推一遍这个“消灭 $\cos\theta_t$”的 3 行代数过程（假设光从空气 $n_1 = 1$ 进入介质 $n_2 = n$）：

1. **斯涅尔定律**：
   $$ 1 \cdot \sin\theta_i = n \cdot \sin\theta_t \implies \sin\theta_t = \frac{\sin\theta_i}{n} $$

2. **用勾股定理（$\sin^2 + \cos^2 = 1$）将 $\cos\theta_t$ 展开**：
   $$ \cos\theta_t = \sqrt{1 - \sin^2\theta_t} = \sqrt{1 - \frac{\sin^2\theta_i}{n^2}} $$

3. **把 $\sin^2\theta_i$ 也换成 $1 - \cos^2\theta_i$**：
   $$ \cos\theta_t = \sqrt{1 - \frac{1 - \cos^2\theta_i}{n^2}} = \frac{\sqrt{n^2 - 1 + \cos^2\theta_i}}{n} $$

**你看！$\cos\theta_t$ 彻底消失了！**
它完全变成了一个只包含 **折射率 $n$** 和 **入射角余弦 $\cos\theta_i$（即 $\mathbf{V} \cdot \mathbf{H}$）** 的代数表达式！（回看我上上局给你写的精确公式里的那个辅助变量 $g = \sqrt{n^2 + u^2 - 1}$，这个 $g$ 其实就是 $n \cos\theta_t$！）

**而在 Schlick 近似中，发生了更彻底的“二次吸收”：**
既然精确公式里只剩下 $n$ 和 $\cos\theta_i$（即 $\mathbf{V} \cdot \mathbf{H}$）两个变量：
*   当 $\theta_i = 0^\circ$ 时，折射率 $n$ 的所有影响，被全部打包浓缩进了起点常数 **$F_0 = \left(\frac{n-1}{n+1}\right)^2$** 里面！
*   而在 $0^\circ \to 90^\circ$ 的变化过程中，Schlick 发现不依赖 $n$ 的通用曲线 $(1 - \mathbf{V}\cdot\mathbf{H})^5$ 就已经能很好地逼近上面那个根号的变化趋势了。

所以，$\cos\theta_t$ 先是被斯涅尔定律转化成了 $n$ 和 $\mathbf{V}\cdot\mathbf{H}$，最后其中的 $n$ 又被浓缩进了 $F_0$ 里，留给着色器运行时计算的，就只剩下孤零零的 $\mathbf{V}\cdot\mathbf{H}$ 了！