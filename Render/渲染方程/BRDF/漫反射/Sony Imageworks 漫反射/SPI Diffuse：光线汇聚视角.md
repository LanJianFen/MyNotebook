

在 **2017 年 SIGGRAPH** 上，Sony Pictures Imageworks（索尼影业，简称 **SPI**）的 **Christopher Kulla 和 Alejandro Conty** 发表了《*Revisiting Physically Based Shading at Imageworks*》。

在这篇讲义中，他们彻底抛弃了 Disney 2012 那套靠经验凑数字（$0.5$ 和 $2.5$）的漫反射，并在 **Kelemen & Szirmay-Kalos (2001)** 的理论基础上，用极其严密的微积分推导出了现代 PBR 史上最优雅的漫反射方程——**能量守恒耦合漫反射（Energy-Conserving Coupled Diffuse）**。

我们从零开始，一步一步把这个公式的物理背景和微积分推导完整重现出来！

---

### 一、 核心痛点：为什么之前的漫反射全都是错的？

我们先思考一个最本质的物理问题：**一块塑料（非金属）表面发生漫反射的真实物理过程是什么？**

塑料由上下两层结构组成：
1. **顶层**：是一层高低起伏、透明的微表面高光界面（由 GGX 描述）；
2. **底层**：是埋在界面下方的色素颗粒（负责吸收光并向四面八方产生均匀的 Lambert 散射）。

在这个双层结构下，看看以前的三种做法错在了哪里：

*   **做法 1（UE4 早期）：直接写 $\frac{\mathbf{c}}{\pi}$**
    完全无视顶层高光的存在。到了掠射角边缘，顶层高光已经把 $100\%$ 的光全反射走了，底下的漫反射居然还在按 $100\%$ 发光，导致总能量达到 $200\%$！
*   **做法 2（glTF 2017）：乘上微观菲涅尔 $\big(1 - F(\mathbf{V}\cdot\mathbf{H})\big) \frac{\mathbf{c}}{\pi}$**
    看起来考虑了高光，但**在数学上是偷换概念**！
    因为半角向量 $\mathbf{H}$ 只代表“能把 $\mathbf{L}$ 刚好反射进**唯一视线方向 $\mathbf{V}$** 的那一小撮微表面”。
    而当一束入射光 $\mathbf{L}$ 打在粗糙表面上时，它撞击的是**所有朝向的微表面**，高光是被弹向了**整个上半球的所有方向**，而不仅仅是 $\mathbf{V}$ 这一个方向！所以用单点方向的 $F(\mathbf{V}\cdot\mathbf{H})$ 去代表整束入射光 $\mathbf{L}$ 被高光抢走的总能量，是完全不对的！
*   **做法 3（Disney 2012）：经验公式 `LightScatter * ViewScatter`**
    固定写死 `0.5` 和 `2.0`，完全不知道顶层高光的折射率 `IOR` 是多少（无论是水 `1.33` 还是钻石 `2.42`，漫反射居然一模一样），而且在粗糙时积分爆表到 $1.51$。

---
### 二、 Coupled Diffuse 推导

*   **唯一积分变量**：四面八方的入射光方向 $\boldsymbol{\omega}_i \in \Omega$（夹角余弦 $\mu_i = \mathbf{n} \cdot \boldsymbol{\omega}_i$）。
*   **唯一固定观察方向**：你的眼睛视线 $\boldsymbol{\omega}_o$（夹角余弦 $\mu_o = \mathbf{n} \cdot \boldsymbol{\omega}_o$）。

#### 第一步：算视线 $\boldsymbol{\omega}_o$ 接收到的“漫反射总能量配额”

在四面八方入射光 $L_i(\boldsymbol{\omega}_i) = 1.0$ 的照射下，假设材质是**纯白色、完全不吸光（$\mathbf{c} = 1.0$）**的理想表面。
根据热力学定律（白炉测试），你的眼睛在 $\boldsymbol{\omega}_o$ 方向看到的总辐射率 $L_o(\boldsymbol{\omega}_o)$ 必须严格等于环境光本身的亮度 $1.0$：

$$ L_{o,\text{spec}}(\boldsymbol{\omega}_o) + L_{o,\text{diff}}(\boldsymbol{\omega}_o) = 1.0 $$

其中，四面八方的光线经由顶层**高光（Specular）**汇聚反射进你视线 $\boldsymbol{\omega}_o$ 的总能量，我们记为 $E(\mu_o)$：
$$ L_{o,\text{spec}}(\boldsymbol{\omega}_o) = \int_{\Omega} f_{\text{spec}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot \underbrace{1.0}_{L_i} \cdot \mu_i \, d\omega_i \equiv E(\mu_o) $$

那么，四面八方的光线经由**漫反射（Diffuse）**汇聚进你视线 $\boldsymbol{\omega}_o$ 的总能量，就只剩下 **$1 - E(\mu_o)$** 的配额了！
写成的积分方程就是：

**$$ \int_{\Omega} f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot 1.0 \cdot \mu_i \, d\omega_i = 1 - E(\mu_o) \quad \text{——【核心方程】} $$**

#### 第二步：直接从【核心方程】倒推出 $f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ 的长相！

我们就死死盯着上面这行**【核心方程】**，用两条最基础的物理数学公理，直接把被积函数 $f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ 的形状锁定出来：

1. **公理 A（次表面失忆性 / 变量可分离）**：
   光子进入漫反射层内部乱撞后，会彻底忘记自己是从哪个 $\boldsymbol{\omega}_i$ 进来的。这意味着在 $f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ 中，关于入射方向 $\mu_i$ 的影响和关于视线方向 $\mu_o$ 的影响是互相独立的（可分离的），必然可以写成两个函数的乘积：
   $$ f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = A(\mu_i) \cdot B(\mu_o) $$

2. **公理 B（亥姆霍兹光路可逆性 / 对称性）**：
   任何物理合法的 BRDF，交换 $\boldsymbol{\omega}_i$ 和 $\boldsymbol{\omega}_o$ 的位置，函数值必须完全不变：
   $$ f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \equiv f_{\text{diff}}(\boldsymbol{\omega}_o, \boldsymbol{\omega}_i) $$
   这意味着函数 $A$ 和函数 $B$ **必须是完全同一个函数形式**（不妨都记为 $g$，再乘上一个常数系数 $K$）：
   **$$ f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = K \cdot g(\mu_i) \cdot g(\mu_o) $$**

3. **代入【核心方程】，瞬间揪出 $g$ 的真身！**
   把 $f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = K \cdot g(\mu_i) \cdot g(\mu_o)$ 代入第一步的【核心方程】中（注意：积分是对 $d\omega_i$ 积，所以 $g(\mu_o)$ 是常数，直接提到积分号外面）：
   $$ \underbrace{\left[ K \int_{\Omega} g(\mu_i) \mu_i \, d\omega_i \right]}_{\text{定积分算完只是一个纯数字常数!}} \cdot \mathbf{g(\mu_o)} \equiv \mathbf{1 - E(\mu_o)} $$

   **🔥 请看上面这个恒等式！**
   等号左边的中括号，把所有 $\boldsymbol{\omega}_i$ 全积分掉了，算出来只是一个与角度无关的**纯数字常数**！
   一个常数乘以函数 $\mathbf{g(\mu_o)}$，对于任意视线角度 $\mu_o$ 都恒等于 $\mathbf{1 - E(\mu_o)}$！
   这在数学上直接宣判：**函数 $g(\mu)$ 的表达式，别无选择，必须就是 $1 - E(\mu)$ 本身！**

   $$ g(\mu) = 1 - E(\mu) $$

   既然求出了函数 $g$ 的表达式是 $1 - E(\mu)$，那么：
   * 代入 $\mu_o$，就是 $g(\mu_o) = 1 - E(\mu_o)$；
   * 代入 $\mu_i$，就是 $g(\mu_i) = 1 - E(\mu_i)$！

   于是，根本不需要切换去讲什么“单根光线视角”，仅仅在利用【核心方程 + 对称性】，$f_{\text{diff}}$ 的骨架就被唯一锁定了：
   **$$ f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = K \cdot \Big( 1 - E(\mu_i) \Big) \cdot \Big( 1 - E(\mu_o) \Big) $$**
#### 第三步：求出常数 $K$，完成闭环

既然已经知道了 $g(\mu_i) = 1 - E(\mu_i)$，上面中括号里那个必须等于 $1$ 的条件就是：
$$ K \int_{\Omega} \Big( 1 - E(\mu_i) \Big) \mu_i \, d\omega_i = 1 $$

展开半球积分 $\int_{\Omega} d\omega_i = \int_0^{2\pi} d\phi_i \int_0^1 d\mu_i = 2\pi \int_0^1 d\mu_i$：
$$ \int_{\Omega} \Big( 1 - E(\mu_i) \Big) \mu_i \, d\omega_i = \pi \left[ \underbrace{2 \int_0^1 1 \cdot \mu_i \, d\mu_i}_{\text{所有方向总入射能量 } = 1} - \underbrace{2 \int_0^1 E(\mu_i) \mu_i \, d\mu_i}_{\text{记为半球平均值 } E_{\text{avg}}} \right] = \pi (1 - E_{\text{avg}}) $$

所以：
$$ K \cdot \pi (1 - E_{\text{avg}}) = 1 \implies \mathbf{K = \frac{1}{\pi (1 - E_{\text{avg}})}} $$

最后，如果材质不是纯白色，而是带有色素吸收率 $\mathbf{c}$（即 `base_color`），乘上 $\mathbf{c}$ 就得到了最终的 **SPI 2017 耦合漫反射公式**：

**$$ f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\mathbf{c}}{\pi} \cdot \frac{\Big( 1 - E(\mu_i) \Big) \cdot \Big( 1 - E(\mu_o) \Big)}{1 - E_{\text{avg}}} $$**

####  检验它的神仙表现！

最爽的时刻到了！我们直接拿**上一局你我刚刚亲手算出来的高光积分结果**，代进这个 SPI 耦合漫反射公式里，看看它在**光滑边缘**和**粗糙边缘**表现如何（设塑料 $F_0 = 0.04, F_{90} = 1.0$）：

##### 1. 当材质绝对光滑（$\alpha \to 0$）时：
*   我们在上一局证明了：光滑时 $E_{\text{spec}}(\mu) = F_0 + (1 - F_0)(1 - \mu)^5$。
*   在球体的 $90^\circ$ 掠射角边缘（$\mu_v = \mathbf{N}\cdot\mathbf{V} \to 0$），我们上一局算出来 **$E_{\text{spec}}(0) = 1.000$**！
*   代入 SPI 漫反射公式的出射门：
    $$ 1 - E_{\text{spec}}(0) = 1 - 1.000 = \mathbf{0.000}！ $$
*   **结果**：不需要像 Frostbite 那样人工写什么 `energyBias = lerp(0, 0.5, r)`，**光滑塑料的漫反射在边缘自动、精确地降到了 $0$**！

##### 2. 当材质绝对粗糙（$\alpha = 1$）时：
*   我们在 GGX 能量分析 刚刚亲手算出：当 $\alpha = 1$ 时，哪怕在 $\mu_v \to 0$ 的最边缘，因为看到的都是竖立的微观山壁，高光积分也只有：
    $$ E_{\text{spec}}(0) = \frac{20}{21} \times 0.04 + \frac{1}{21} \times 1.0 \approx \mathbf{0.0857} \quad (\text{仅 } 8.57\%！) $$
*   代入 SPI 漫反射公式的出射门：
    $$ 1 - E_{\text{spec}}(0) = 1 - 0.0857 = \mathbf{0.9143} \quad (\text{高达 } 91.4\%！) $$
*   再除以分母的 $1 - E_{\text{spec\_avg}} \approx 0.91$，边缘的出射倍率直接恢复到了 **$\approx 1.0$**！
*   **结果**：随着粗糙度从 $0$ 变到 $1$，物体边缘的漫反射亮度从 **$0.0$ 自然回升到了 $1.0$**，而且在任何粗糙度、任何折射率、任何视角下，**`Diffuse + Specular` 的白炉积分永远精确等于 $1.000000$**！

这一下，从痛点分析、四步微积分推导，到把我们上一局算出的 $E_{\text{spec}}$ 代入验证，SPI 2017 这套耦合漫反射的来龙去脉是不是彻底印在脑子里了？!]

---

### 三、严谨多重吸收条件

如果带上吸收，那公式应该写成这样
$$ \int_{\Omega} f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot 1.0 \cdot \mu_i \, d\omega_i = C_{\text{survive}} \cdot \Big( 1 - E(\mu_o) \Big) \quad \text{——【带颜色的核心方程】} $$

#### 厘清一个关键的概念：这个公式到底是“1 次弹射”还是“多次弹射”？

准确地说，它是**“天花板无限次弹射，但地板只吸收 1 次（单次色素吸收）”**的结果！为什么？我们对比一下三种情况你就彻底看清了：

1. **真正的“几何单次弹射”（只撞地板 1 次，被天花板挡回的光直接丢弃）**：
	   如果撞地板 1 次，然后直接从 $g(\mu_o)$ 出去，那根本不需要分母
	$$ \int_{\Omega} f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot 1.0 \cdot \mu_i \, d\omega_i = \left[ \frac{\mathbf{c}}{\pi} \int_{\Omega} \big(1 - E(\mu_i)\big) \mu_i \, d\omega_i \right] \cdot g(\mu_o) =\mathbf{\mathbf{c} \cdot (1 - E_{\text{avg}}) \cdot \big(1 - E(\mu_o)\big)} $$
	   
	你看，真正只弹射 1 次的话，右边多乘了一个 $(1 - E_{\text{avg}})$, 多重弹射的能量消失了
2. **刚刚的公式右边直接让 $C_{\text{survive}} =\mathbf{c}$$$\text{右边} = \mathbf{c} \cdot \big(1 - E(\mu_o)\big)$$
	因为 $1 - E(\mu_o)$ 是**纯白地板（$\mathbf{c}=1$）经历无限次天花板反弹后**才凑够的满额出射量！你在最外层直接乘一个 $\mathbf{c}$，等于假设**“光在里面反弹了无数次都没被吸收，直到最后出门时才统一被 $\mathbf{c}$ 吸收了 1 次”**！

#### 第一步：明确被天花板反弹回去的能量

既然我们已经证明了 $g(\mu_o) = 1 - E(\mu_o)$ 且 $g(\mu_i) = 1 - E(\mu_i)$，我们把它们代回上面的恒等式，并把等式左右两边的 $\mathbf{\big(1 - E(\mu_o)\big)}$ 直接约掉，*能量归一化！*

方程瞬间只剩下常数之间的等式：
$$ K \int_{\Omega} \Big( 1 - E(\mu_i) \Big) \mu_i \, d\omega_i = C_{\text{survive}} $$

我们在文档里已经算过左边这个定积分精确等于 $\pi (1 - E_{\text{avg}})$，所以：
**$$ K \cdot \pi (1 - E_{\text{avg}}) = C_{\text{survive}} $$
我们把左边的 $K \cdot \pi (1 - E_{\text{avg}})$ 展开成两项：

$$ \underbrace{K \pi}_{\text{内部第1次总能量}} - \underbrace{K \pi E_{\text{avg}}}_{\text{内部被顶层界面挡回来的能量}} = C_{\text{survive}} $$
*    $K\pi$ 是归一化能量后，内部基底最终弹起的总能量；
*   而 **$K \pi E_{\text{avg}}$ 是因为没能逃出表面、被顶层界面挡回来再次砸向基底的能量**！

#### 第二步：明确 $C_{\text{survive}}$ 的物理定义，并直接构造它的表达式

我们先看你停下来的这个方程：
$$ K \cdot \pi (1 - E_{\text{avg}}) = C_{\text{survive}} $$

在这个方程里，右边的 **$C_{\text{survive}}$（总存活率）** 到底是什么？
它的定义是：**凡是成功穿透顶层高光界面、进入到表面内部夹层的能量（把这笔进入内部的初始能量看作单位 $1$），在内部经历了无数次“地板吸收 $+$ 天花板挡回”之后，最终活着逃出表面外部的总比例！**

现在，我们就跟着这 **$1$ 份** 进入内部的能量，一轮一轮地算它**每次撞地板被吸收 $\mathbf{c}$、每次撞天花板逃出 $(1 - E_{\text{avg}})$** 的全过程：

1. **第 1 轮（经历 1 次地板吸收 $\mathbf{c}^1$）**：
   * $1$ 份能量第 1 次撞击地板，被吸收掉 $(1 - \mathbf{c})$，存活下来 **$\mathbf{c}$** 向上冲向天花板；
   * 撞击天花板时，有 $(1 - E_{\text{avg}})$ 的比例成功逃出表面，有 $E_{\text{avg}}$ 的比例被挡回内部：
   * **第 1 轮成功逃出量**：$\mathbf{c} \cdot (1 - E_{\text{avg}})$
   * *（留在内部被挡回地板的量：$\mathbf{c} \cdot E_{\text{avg}}$）*

2. **第 2 轮（经历 2 次地板吸收 $\mathbf{c}^2$）**：
   * 被挡回来的 $\mathbf{c} \cdot E_{\text{avg}}$ 再次砸在地板上，**被地板第 2 次吸收（再乘一次 $\mathbf{c}$！）**，只剩下 **$\mathbf{c}^2 \cdot E_{\text{avg}}$** 的能量向上冲向天花板；
   * 撞击天花板时，只有 $(1 - E_{\text{avg}})$ 的比例成功逃出，剩下 $E_{\text{avg}}$ 的比例继续被挡回：
   * **第 2 轮成功逃出量**：$\mathbf{c}^2 \cdot E_{\text{avg}} \cdot (1 - E_{\text{avg}})$
   * *（留在内部被挡回地板的量：$\mathbf{c}^2 \cdot E_{\text{avg}}^2$）*

3. **第 3 轮（经历 3 次地板吸收 $\mathbf{c}^3$）**：
   * 被挡回的 $\mathbf{c}^2 \cdot E_{\text{avg}}^2$ 第 3 次砸在地板上，**被地板第 3 次吸收（再乘一次 $\mathbf{c}$！）**，变成 $\mathbf{c}^3 \cdot E_{\text{avg}}^2$ 向上冲，其中 $(1 - E_{\text{avg}})$ 逃出：
   * **第 3 轮成功逃出量**：$\mathbf{c}^3 \cdot E_{\text{avg}}^2 \cdot (1 - E_{\text{avg}})$

4. **第 $n$ 轮（经历 $n$ 次地板吸收 $\mathbf{c}^n$）**：
   * **第 $n$ 轮成功逃出量**：$\mathbf{c}^n \cdot E_{\text{avg}}^{n-1} \cdot (1 - E_{\text{avg}})$

现在，把第 1 轮、第 2 轮、第 3 轮……直到第 $\infty$ 轮**所有成功逃出天花板的能量全部加起来**，就构造出了 $C_{\text{survive}}$ 的精确表达式：

$$ C_{\text{survive}} = \underbrace{\mathbf{c}(1 - E_{\text{avg}})}_{\text{1次吸收后逃出}} + \underbrace{\mathbf{c}^2 E_{\text{avg}}(1 - E_{\text{avg}})}_{\text{2次吸收后逃出}} + \underbrace{\mathbf{c}^3 E_{\text{avg}}^2(1 - E_{\text{avg}})}_{\text{3次吸收后逃出}} + \dots $$

你看，每一项都有公因式 $\mathbf{c}(1 - E_{\text{avg}})$，把它提出来：

$$ C_{\text{survive}} = \mathbf{c}(1 - E_{\text{avg}}) \cdot \Big[ 1 + (\mathbf{c} E_{\text{avg}}) + (\mathbf{c} E_{\text{avg}})^2 + (\mathbf{c} E_{\text{avg}})^3 + \dots \Big] $$

括号里是一个标准的无穷等比数列（公比 $q = \mathbf{c} E_{\text{avg}} < 1$），根据求和公式 $\sum_{k=0}^{\infty} q^k = \frac{1}{1 - q}$，直接得到 **$C_{\text{survive}}$ 的终极构造式**：

**$$ C_{\text{survive}} = \frac{\mathbf{c} \cdot (1 - E_{\text{avg}})}{1 - \mathbf{c} \cdot E_{\text{avg}}} $$**

**💡 先停下来检验一下这个 $C_{\text{survive}}$ 有多合理：**
*   如果地板是**纯白色不吸光（$\mathbf{c} = 1$）**：代入上式，$C_{\text{survive}} = \frac{1 \cdot (1 - E_{\text{avg}})}{1 - 1 \cdot E_{\text{avg}}} = \mathbf{1.0}$（进来的能量 $100\%$ 全部逃出，完美符合白炉定义！）。
*   如果地板是**纯黑色全吸收（$\mathbf{c} = 0$）**：代入上式，$C_{\text{survive}} = \mathbf{0.0}$（第 1 次撞地板就全死了，逃出为 $0$）。
*   如果**忽略后续弹射的二次吸收**（假设只有第 1 次乘了 $\mathbf{c}$，后面挡回弹射时令分母里的 $\mathbf{c}=1$）：$C_{\text{survive}} = \frac{\mathbf{c}(1 - E_{\text{avg}})}{1 - 1 \cdot E_{\text{avg}}} = \mathbf{\mathbf{c}}$（直接退化为线性版！）。

#### 第三步：把构造好的 $C_{\text{survive}}$ 代入方程，一行解出 $K$！

现在，我们把刚刚构造好的 $C_{\text{survive}} = \frac{\mathbf{c}(1 - E_{\text{avg}})}{1 - \mathbf{c} \cdot E_{\text{avg}}}$，代入你指定的那个方程：

$$ K \cdot \pi (1 - E_{\text{avg}}) = C_{\text{survive}} $$

写出来就是：
$$ K \cdot \pi \mathbf{(1 - E_{\text{avg}})} = \frac{\mathbf{c} \cdot \mathbf{(1 - E_{\text{avg}})}}{1 - \mathbf{c} \cdot E_{\text{avg}}} $$

**🔥 请看这最爽的一幕：**
等式左边的 **$(1 - E_{\text{avg}})$**，和等式右边分子上的 **$(1 - E_{\text{avg}})$**，当场直接约分抵消**！

两边同时除以 $\pi$，$K$ 瞬间解出：
**$$ K = \frac{\mathbf{c}}{\pi (1 - \mathbf{c} \cdot E_{\text{avg}})} ！！ $$**