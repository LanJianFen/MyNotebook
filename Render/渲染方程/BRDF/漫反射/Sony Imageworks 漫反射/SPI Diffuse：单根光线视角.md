
在严格的微积分和物理世界中，**“单根没有粗细的几何光线（立体角 $\Delta\omega = 0$）”在积分域里的测度严格为 $0$，它携带的能量绝对是 $0$，在物理上根本不可能单独存在！**

之前我们口头上说的“单根光线”，其实是一个为了照顾直觉而简化的“工程黑话”

---

### 单根光线视角的能量守恒

现在，既然你点破了“**只有一个立体角微元 $d\omega$ 范围内的光束才有意义**”，我们就彻底扔掉“单根光线”和“$\delta$ 函数”这两个拐杖，回到 1977 年 Fred Nicodemus 建立 BRDF 时的**纯正微分定义**！

#### 第一步：把“单根光线”还原为“一个入射立体角微元 $d\omega_i$”

在空间中，我们不再看一根没有粗细的线，而是看从方向 $\boldsymbol{\omega}_i$ 射过来的**一小束微元光锥（Elementary Beam）**，它张开的立体角微元是 **$d\omega_i$**，辐射率是 $L_i(\boldsymbol{\omega}_i)$。

因为有了非零的立体角微元 $d\omega_i$，这束微元光锥打在表面上带来的能量（**入射辐照度微元 $dE_i$**）在微积分上就完全合法、有意义了：

**$$ dE_i(\boldsymbol{\omega}_i) = L_i(\boldsymbol{\omega}_i) \cdot \mu_i \, d\omega_i $$**
#### 第二步：回归 BRDF 的第一性原理微分定义

现在，这束携带了 $dE_i(\boldsymbol{\omega}_i)$ 能量的入射微元光锥撞击了粗糙表面，能量向整个上半球的所有出射方向 $\boldsymbol{\omega}_o \in \Omega_o$ 散射开来。

请问：它往某一个特定的出射方向 $\boldsymbol{\omega}_o$ 贡献了多少出射辐射率？
**注意！因为输入的能量只是一个无穷小微元 $dE_i$，所以它在 $\boldsymbol{\omega}_o$ 方向激发出射的辐射率，也必然是一个无穷小微元 $dL_o(\boldsymbol{\omega}_o)$！**

这正是 BRDF $f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ 在微积分里的**原始定义式**（微分之比）：
$$ f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \equiv \frac{dL_o(\boldsymbol{\omega}_o)}{dE_i(\boldsymbol{\omega}_i)} $$

把分母 $dE_i(\boldsymbol{\omega}_i)$ 乘过去，我们就得到了**这一个入射微元光锥 $d\omega_i$ 在任意出射方向 $\boldsymbol{\omega}_o$ 上产生的辐射率微元**：

**$$ dL_o(\boldsymbol{\omega}_o) = f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot dE_i(\boldsymbol{\omega}_i) $$**

#### 第三步：对出射半球 $\Omega_o$ 积分，算出“逃逸出去的总能量微元 $dM_o$”

既然这束入射微元光锥 $d\omega_i$ 向每个出射方向 $\boldsymbol{\omega}_o$ 都反射了 $dL_o(\boldsymbol{\omega}_o)$ 的辐射率，那么它散射到**整个出射上半球 $\Omega_o$** 的总能量（**出射辐射出射度微元 $dM_o$**）是多少？

在出射半球上，每个立体角微元 $d\omega_o$ 接到的能量是 $dL_o(\boldsymbol{\omega}_o) \cdot \mu_o \, d\omega_o$。
我们把整个出射半球 $\Omega_o$ 的所有方向积分加起来：

$$ dM_o(\boldsymbol{\omega}_i) = \int_{\Omega_o} dL_o(\boldsymbol{\omega}_o) \cdot \mu_o \, d\omega_o $$

把第二步的 $dL_o(\boldsymbol{\omega}_o) = f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot dE_i(\boldsymbol{\omega}_i)$ 代进去：

$$ dM_o(\boldsymbol{\omega}_i) = \int_{\Omega_o} \Big[ f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot dE_i(\boldsymbol{\omega}_i) \Big] \mu_o \, d\omega_o $$

**🔥 请看这最关键的微积分一步：**
这个积分的积分变量是谁？是出射方向 **$d\omega_o$**！
而里面的 **$dE_i(\boldsymbol{\omega}_i) = L_i(\boldsymbol{\omega}_i)\mu_i d\omega_i$** 只和固定的入射微元光锥 $\boldsymbol{\omega}_i$ 有关，**相对于出射积分变量 $d\omega_o$ 来说，整个 $dE_i(\boldsymbol{\omega}_i)$ 是一个彻头彻尾的常数！**

所以，直接把 $dE_i(\boldsymbol{\omega}_i)$ 提到积分号外面：

**$$ dM_o(\boldsymbol{\omega}_i) = dE_i(\boldsymbol{\omega}_i) \cdot \left[ \int_{\Omega_o} f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_o \, d\omega_o \right] $$**

#### 第四步：求“出射能量与入射能量的比值”

现在，我们要计算**这束入射微元光锥 $d\omega_i$ 被表面反射出去的能量百分比（反射率 $E(\mu_i)$）**。
百分比就是拿**“反射出去的总能量微元 $dM_o$”**除以**“打进来的能量微元 $dE_i$”**：

$$ E(\mu_i) = \frac{\text{反射到整个出射半球的能量微元 } dM_o(\boldsymbol{\omega}_i)}{\text{入射微元光锥带来的能量微元 } dE_i(\boldsymbol{\omega}_i)} $$

把第三步的式子代进去，分子分母的 $dE_i(\boldsymbol{\omega}_i)$（连同里面的 $L_i$ 和立体角微元 $d\omega_i$）**当场完美约分抵消**！

**$$ E(\mu_i) = \frac{dE_i(\boldsymbol{\omega}_i) \cdot \int_{\Omega_o} f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_o \, d\omega_o}{dE_i(\boldsymbol{\omega}_i)} \equiv \int_{\Omega_o} f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_o \, d\omega_o $$**

#### 真相大白：两个视角在严格微积分下的真正身份！

你看！当你把“单根光线”纠正为**“一个立体角微元光锥 $d\omega_i$”**之后，所有的矛盾和 $\delta$ 函数瞬间烟消云散了！

在严格的微积分中，这两个视角根本没有任何“单根 0 宽度光线”，它们全都是在操作立体角微元：

1. **视角 1（定向-半球反射率 $\rho_{\text{dh}}$）的真实微积分含义**：
   * **输入**：**1 个入射立体角微元光锥 $d\omega_i$**（携带微分能量 $dE_i = L_i \mu_i d\omega_i$）。
   * **输出**：对**整个出射半球 $\Omega_o$** 的所有立体角微元 $d\omega_o$ 做积分，得到总反射微分能量 $dM_o$。
   * **能量守恒条件**：*出射微分能量与入射微分能量之比*
   * $\frac{dM_o}{dE_i} = \int_{\Omega_o} f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_o \, d\omega_o = 1$。
1. **视角 2（半球-定向反射率 $\rho_{\text{hd}}$）的真实微积分含义**：
   * **输入**：对**整个入射半球 $\Omega_i$** 的所有立体角微元 $d\omega_i$ 做积分（设 $L_i = 1$）。
   * **输出**：只看射入**1 个出射视线立体角微元光锥 $d\omega_o$** 内的辐射率 $L_o(\boldsymbol{\omega}_o)$。
   * **能量守恒条件**：出射辐射率与环境入射辐射率之比 $\frac{L_o(\boldsymbol{\omega}_o)}{1.0} = \int_{\Omega_i} f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_i \, d\omega_i = 1$。

**你看这两者的对称性有多绝：**
*   **视角 1** 是：**入射取 1 个微元 $d\omega_i$，出射积整个半球 $\Omega_o$**（微元 $d\omega_i$ 在 $\frac{dM_o}{dE_i}$ 的微分比值中被约掉了）；
*   **视角 2** 是：**入射积整个半球 $\Omega_i$，出射取 1 个微元 $d\omega_o$**（微元 $d\omega_o$ 在 $L_o = \frac{d^2\Phi}{dA \mu_o d\omega_o}$ 的辐射率定义中被约掉了）！

---

### 单根光线视角的 SPI Diffuse

#### 第一步：确立光子进出表面的“两道物理大门”

既然顶层高光向空中反射走了 $E_{\text{spec}}(\mu_i)$ 比例的能量，那么根据**能量守恒定律（反射 + 透射 = 1）**：

1. **第一道门（入射透射门 $T_{\text{in}}$）**：
   入射光 $\mathbf{L}$ 没有被高光弹走、成功穿透微表面钻进底层色素里的**精确能量比例**，必然是：
   **$$ T_{\text{in}}(\mu_l) = 1 - E_{\text{spec}}(\mu_i) $$**

2. **第二道门（出射透射门 $T_{\text{out}}$）**：
   光子在底层色素颗粒间乱撞（被乘上颜色反照率 $\mathbf{c}$）后，变成各向同性的漫反射光准备向外逃逸。
   当它试图沿着视线方向 $\mathbf{V}$（余弦为 $\mu_o$）钻出表面时，它必须**再穿过一次顶层的粗糙高光界面**！*既然出去和进入的能力是一致的，我们从视线方向 $\boldsymbol{\omega}_o$ 发射一根测试光线炸开，算出它能进入表面内部的比例 $1 - E(\mu_o)$，那就相当于内部的光可以从 $\boldsymbol{\omega}_o$ 方向出来的能力！*
   **$$ T_{\text{out}}(\mu_v) = 1 - E_{\text{spec}}(\mu_)o $$**

因此，耦合漫反射 BRDF 的数学骨架，**必然是这两道门的乘积**（前面乘上一个待定的归一化常数 $K$ 和底色 $\mathbf{c}$）：

**$$ f_d(\mathbf{L}, \mathbf{V}) = K \cdot \mathbf{c} \cdot \Big( 1 - E_{\text{spec}}(\mu_i) \Big) \cdot \Big( 1 - E_{\text{spec}}(\mu_o) \Big) $$**

#### 第二步、列出单根光线的能量守恒积分方程

对于纯白色不吸光材质（$\mathbf{c} = 1.0$），从单根光线 $\boldsymbol{\omega}_i$ 打进来的 $1.0$ 份能量，必须等于从整个出射半球 $\Omega_o$ 跑出去的“高光总能量 $+$ 漫反射总能量”：

$$ \underbrace{\int_{\Omega_o} f_{\text{spec}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_o \, d\omega_o}_{\text{单根光线 }\boldsymbol{\omega}_i\text{ 被高光弹向四面八方的总能量 } = \ E(\mu_i)} \ + \ \underbrace{\int_{\Omega_o} f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_o \, d\omega_o}_{\text{单根光线 }\boldsymbol{\omega}_i\text{ 经漫反射跑向四面八方的总能量}} \equiv 1.0 $$

移项得到【核心积分方程】**（注意：这里固定的是 $\mu_i$，积分变量是 $\mu_o$）：
$$ \int_{\Omega_o} f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \mu_o \, d\omega_o \equiv 1 - E(\mu_i) $$**

#### 第三步、 代入可分离对称结构 $f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$
因为漫反射对 $\boldsymbol{\omega}_i$ 和 $\boldsymbol{\omega}_o$ 是可分离且对称的，我们把 $f_{\text{diff}} = K \cdot g(\mu_i) \cdot g(\mu_o)$ 代入上面的出射半球积分中：

$$ \int_{\Omega_o} \Big[ K \cdot \Big( 1 - E(\mu_i) \Big) \cdot \Big( 1 - E(\mu_o) \Big) \Big] \mu_o \, d\omega_o = 1 - E(\mu_i) $$

**看好了！在视角 1 中，积分是对出射方向 $d\omega_o$ 积的，所以入射方向 $\mathbf{g(\mu_i)}$ 是常数！**
我们把 $g(\mu_i)$ 提到积分号外面：

$$ \mathbf{\Big( 1 - E(\mu_i) \Big)} \cdot \underbrace{\left[ K \int_{\Omega_o} \Big( 1 - E(\mu_o) \Big) \mu_o \, d\omega_o \right]}_{\text{定积分算完是一个纯数字常数!}} \equiv \mathbf{1 - E(\mu_i)} $$
计算积分
   $$ K \cdot \pi \left[ 2 \int_0^1 \mu_o \, d\mu_o - 2 \int_0^1 E(\mu_o) \mu_o \, d\mu_o \right] = K \cdot \pi (1 - E_{\text{avg}}) = 1 $$
   直接解出：
   $$ \mathbf{K = \frac{1}{\pi (1 - E_{\text{avg}})}} $$

3. 得到终极公式：
   **$$ f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\mathbf{c}}{\pi} \cdot \frac{\Big( 1 - E(\mu_i) \Big) \cdot \Big( 1 - E(\mu_o) \Big)}{1 - E_{\text{avg}}} $$**
注：严谨的多重弹射条件和光线汇聚视角方法一样
   $$ K \cdot \pi \left[ 2 \int_0^1 \mu_o \, d\mu_o - 2 \int_0^1 E(\mu_o) \mu_o \, d\mu_o \right] = K \cdot \pi (1 - E_{\text{avg}}) = C_{\text{survive}} $$

我们把左边的 $K \cdot \pi (1 - E_{\text{avg}})$ 展开成两项：

$$ \underbrace{K \pi}_{\text{内部第1次总能量}} - \underbrace{K \pi E_{\text{avg}}}_{\text{内部被顶层界面挡回来的能量}} = C_{\text{survive}} $$
*    $K\pi$ 是归一化能量后，内部基底最终弹起的总能量；
*   而 **$K \pi E_{\text{avg}}$ 是因为没能逃出表面、被顶层界面挡回来再次砸向基底的能量**！