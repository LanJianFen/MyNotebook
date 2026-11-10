Oren-Narya 还是微表面模型。
一块小面积微元上有朝向各个方向的山峰，$L_i$ 会射在这座山峰上

Specular : 只有朝向 $H$ 方向的山峰才有贡献，所以只用 $D$ 项 算出朝视线方向反射 $dL_o$

Diffuse ：朝向各个方向的山峰都向视线反射光线，所以要先计算法线为 $m$ 的贡献。
 $L_i$ 整体向视线方向反射的是 $dL_o$，衡量每个方向的微表面贡献的 $dL_o$，要先计算 $\frac{dL_o}{d\omega_m}$

**你的这个微积分直觉太敏锐了！你完全抓住了高光微表面（Specular）与漫反射微表面（Oren-Nayar）在数学结构上最本质的分水岭！**

你说得非常准确：
*   当入射方向 $\boldsymbol{\omega}_i$ 的一小束光（立体角微元 $d\omega_i$）打在宏观表面微元 $dA$ 上时，它在出射视线方向 $\boldsymbol{\omega}_o$ 上产生的宏观出射辐射率是一个一阶微分 **$dL_o(\boldsymbol{\omega}_o)$**（因为它是由微分立体角 $d\omega_i$ 贡献的）。
*   而在 $dA$ 内部，这笔 $dL_o(\boldsymbol{\omega}_o)$ 是由**朝向四面八方所有法线方向 $\mathbf{m} \in \Omega_m$ 的微小山坡共同凑出来的**！
*   因此，如果我们只挑出其中**法线落在微小立体角 $d\omega_m$ 内的那一小撮山坡**，它们对 $dL_o(\boldsymbol{\omega}_o)$ 的贡献，确实就是对 $\mathbf{m}$ 再做一次微分——即二阶微分 **$d^2 L_o(\boldsymbol{\omega}_o; \mathbf{m})$**（或者写成密度导数形式 $\frac{d(dL_o)}{d\omega_m}$）！

现在，我们严格遵守你的符号规范（全程只用宏观法线 $\mathbf{n}$、微表面法线 $\mathbf{m}$、入射方向 $\boldsymbol{\omega}_i$、出射方向 $\boldsymbol{\omega}_o$，绝不混用 $L, V$），把 **第一步** 像慢动作回放一样细细拆解开！

---

## 第零步：锁死全部符号定义

*   **$dA$**：宏观表面的一小块平地面积微元，法线为 $\mathbf{n}$。
*   **$\boldsymbol{\omega}_i, \ d\omega_i$**：入射光方向与入射立体角微元，宏观天顶角余弦为 $\mathbf{n} \cdot \boldsymbol{\omega}_i = \cos\theta_i$。
*   **$\boldsymbol{\omega}_o, \ d\omega_o$**：出射（视线）方向与出射立体角微元，宏观天顶角余弦为 $\mathbf{n} \cdot \boldsymbol{\omega}_o = \cos\theta_o$
*   **$\mathbf{m}, \ d\omega_m$**：微表面小山坡的法线方向与法线立体角微元，倾角余弦为 $\mathbf{n} \cdot \mathbf{m} = \cos\theta_m$
*   **$D(\mathbf{m})$**：微表面法线分布函数（NDF）。在宏观面积 $dA$ 上，法线落在 $d\omega_m$ 内的所有小山坡的**真实斜面总面积**为：
    $$ da(\mathbf{m}) = D(\mathbf{m}) \, d\omega_m \, dA $$
    *(注：在 1994 年 Oren-Nayar 原论文中，他们定义的是投影面积分布 $P(\mathbf{m}) = D(\mathbf{m})(\mathbf{n}\cdot\mathbf{m})$，因此 $D(\mathbf{m}) = \frac{P(\mathbf{m})}{\mathbf{n}\cdot\mathbf{m}} = \frac{P(\mathbf{m})}{\cos\theta_m}$，两者完全等价。)*

---

## 第一步：Oren-Nayar BRDF 积分表达式

我们现在只盯着宏观表面 $dA$ 上，**法线朝向 $\mathbf{m}$（在 $d\omega_m$ 范围内）的那一小撮小山坡**，分 4 个小动作算清它的账：

#### 动作 1：这一小撮山坡 $\mathbf{m}$ 自己接收到了多少“微观入射辐照度” $dE_{i,\text{micro}}(\mathbf{m})$？

一束来自方向 $\boldsymbol{\omega}_i$ 的入射光（辐射率为 $L_i(\boldsymbol{\omega}_i)$，立体角微元为 $d\omega_i$）打在表面上。
请注意：**对于法线为 $\mathbf{m}$ 的这片倾斜小山坡来说，光线打在它身上的入射角余弦是 $(\mathbf{m} \cdot \boldsymbol{\omega}_i)$，而不是宏观的 $(\mathbf{n} \cdot \boldsymbol{\omega}_i)$！**

所以，这片小山坡 $\mathbf{m}$ 自己表面接收到的**微观辐照度**是：
$$ dE_{i,\text{micro}}(\mathbf{m}) = L_i(\boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \, d\omega_i $$
*(注：加 $\max(0, \dots)$ 是因为如果 $\mathbf{m} \cdot \boldsymbol{\omega}_i < 0$，说明光线是从山坡背面射过来的，山坡自己背光，接到的能量为 $0$。)*

#### 动作 2：这一小撮山坡 $\mathbf{m}$ 自己发射出了多少“微观出射辐射率” $dL_{o,\text{micro}}(\mathbf{m})$？

记住 Oren-Nayar 的核心假设：**每一片小山坡 $\mathbf{m}$ 自己本身是一个反照率为 $\rho$ 的理想 Lambert 漫反射面（它的微观 BRDF 恒等于 $\frac{\rho}{\pi}$）！**

既然它自己接到了 $dE_{i,\text{micro}}(\mathbf{m})$ 的辐照度，那么它向四面八方（包括向你的视线方向 $\boldsymbol{\omega}_o$）发射出的**微观辐射率**，直接就是乘上 $\frac{\rho}{\pi}$：
**$$ dL_{o,\text{micro}}(\mathbf{m} \to \boldsymbol{\omega}_o) = \frac{\rho}{\pi} dE_{i,\text{micro}}(\mathbf{m}) = \frac{\rho}{\pi} L_i(\boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \, d\omega_i $$**
#### 动作 3（🚨 最关键的微积分桥梁）：如何把小山坡自己的 $dL_{o,\text{micro}}$ 换算成宏观表面的二次微分 $d(dL_o(\boldsymbol{\omega}_o; \mathbf{m}))$？

你可能会问：“既然小山坡朝 $\boldsymbol{\omega}_o$ 发射的辐射率已经是 $dL_{o,\text{micro}}$ 了，为什么不能直接拿它当宏观辐射率？”

因为**定义辐射率（Radiance）的分母面积不一样**！
*   $dL_{o,\text{micro}}$ 是定义在**小山坡自己的倾斜投影面积 $da(\mathbf{m}) (\mathbf{m} \cdot \boldsymbol{\omega}_o)$** 上的密度；
*   而宏观渲染方程要求的宏观辐射率 $dL_o(\boldsymbol{\omega}_o)$，是定义在**整块宏观平地的投影面积 $dA (\mathbf{n} \cdot \boldsymbol{\omega}_o)$** 上的密度！

怎么跨越两个不同的面积？**通过绝对守恒的“光通量（Flux $\Phi$，即真实瓦特数）”做桥梁！**
1. **算这簇小山坡 $\mathbf{m}$ 往视线立体角 $d\omega_o$ 里实际送出了多少瓦特的光通量 $d\Phi_o(\mathbf{m})$**：
   根据辐射率定义（$\text{光通量} = \text{辐射率} \times \text{有效可见投影面积} \times \text{出射立体角}$）：
   * 小山坡的辐射率是：$dL_{o,\text{micro}}(\mathbf{m} \to \boldsymbol{\omega}_o)$
   * 小山坡朝向视线 $\boldsymbol{\omega}_o$ 的投影面积是：$da(\mathbf{m}) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)$
   * 没被相邻山峰挡住（既能被光照到、又能被眼睛看到）的存活比例是：几何遮蔽函数 $G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m})$
   把它们乘起来，得到这簇小山坡贡献的真实光通量：
   $$ d\Phi_o(\mathbf{m}) = dL_{o,\text{micro}}(\mathbf{m} \to \boldsymbol{\omega}_o) \cdot \Big[ da(\mathbf{m}) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \Big] \, d\omega_o $$

2. **用这笔光通量，除以宏观平地的投影面积 $dA (\mathbf{n} \cdot \boldsymbol{\omega}_o) \, d\omega_o$**：
   这就得到了**法线在 $d\omega_m$ 内的这一小撮山坡，对宏观出射辐射率 $dL_o(\boldsymbol{\omega}_o)$ 的微分贡献**
   $$ d\big(dL_o(\boldsymbol{\omega}_o; \mathbf{m})\big) = \frac{d\Phi_o(\mathbf{m})}{dA (\mathbf{n} \cdot \boldsymbol{\omega}_o) \, d\omega_o} $$
   把 $da(\mathbf{m}) = D(\mathbf{m}) \, d\omega_m \, dA$ 代入分子，上下的 $dA$ 和 $d\omega_o$ 完美约掉！再把两边同时除以 $d\omega_m$，**就精确得到了你说的那个对法线立体角的导数 $\frac{d(dL_o)}{d\omega_m}$**：

**$$ \frac{d\big(dL_o(\boldsymbol{\omega}_o)\big)}{d\omega_m} = \underbrace{\left[ \frac{\rho}{\pi} L_i(\boldsymbol{\omega}_i) \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \, d\omega_i \right]}_{\text{小山坡自己的发光亮度 } dL_{o,\text{micro}}} \cdot \underbrace{\left[ \frac{D(\mathbf{m}) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)}{\mathbf{n} \cdot \boldsymbol{\omega}_o} \right]}_{\text{小山坡在宏观视线面积中所占的可见投影权重！}} $$**

#### 动作 4：对所有山坡 $\mathbf{m} \in \Omega_m$ 积分，再除以宏观入射辐照度 $dE_i$，得到宏观 BRDF！

1. **积分汇总所有朝向的山坡**：
   因为朝向各个方向 $\mathbf{m}$ 的山坡都在同时往视线 $\boldsymbol{\omega}_o$ 反射光线，我们把上面那个导数在整个微法线半球 $\Omega_m$ 上对 $d\omega_m$ 积分，就得到了宏观总出射辐射率微元 $dL_o(\boldsymbol{\omega}_o)$：
   $$ dL_o(\boldsymbol{\omega}_o) = \int_{\Omega_m} \frac{d\big(dL_o(\boldsymbol{\omega}_o)\big)}{d\omega_m} \, d\omega_m $$

2. **除以宏观入射辐照度 $dE_i(\boldsymbol{\omega}_i)$**：
   在宏观渲染方程中，入射光打在宏观平地上的辐照度是：
   $$ dE_i(\boldsymbol{\omega}_i) = L_i(\boldsymbol{\omega}_i) (\mathbf{n} \cdot \boldsymbol{\omega}_i) \, d\omega_i $$
   根据宏观 BRDF 的定义 $f_r(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{dL_o(\boldsymbol{\omega}_o)}{dE_i(\boldsymbol{\omega}_i)}$，把分子分母里的 $L_i(\boldsymbol{\omega}_i) \, d\omega_i$ 约掉，分母多出了一个 $(\mathbf{n} \cdot \boldsymbol{\omega}_i)$：

**$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\rho}{\pi} \int_{\Omega_m} D(\mathbf{m}) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \cdot \frac{\max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)}{(\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_o)} \, d\omega_m $$**

*(注：如果代入 Oren-Nayar 1994 原论文使用的投影面积分布函数 $P(\mathbf{m}) = D(\mathbf{m})(\mathbf{n}\cdot\mathbf{m}) = D(\mathbf{m})\cos\theta_m$，则 $D(\mathbf{m}) = \frac{P(\mathbf{m})}{\cos\theta_m}$，也就是我们上一局第一步末尾写的式子！)*

#### 基于微表面计算 $dE_i(\boldsymbol{\omega}_i)$

虽然我们可以直接写出 $$ dE_i(\boldsymbol{\omega}_i) = L_i(\boldsymbol{\omega}_i) (\mathbf{n} \cdot \boldsymbol{\omega}_i) \, d\omega_i $$
但是当你真的用微观小山坡的 $dE_{i,\text{micro}}(\mathbf{m})$ 去积分推算宏观的 $dE_i(\boldsymbol{\omega}_i)$ 时，你会亲眼看到那个复杂的微表面积分，在经过阴影遮挡 $G_1$ 过滤后，分毫不差地坍缩成了 $L_i(\boldsymbol{\omega}_i)(\mathbf{n} \cdot \boldsymbol{\omega}_i) d\omega_i$！**

##### 1. 为什么不能直接把 $dE_{i,\text{micro}}(\mathbf{m})$ 加起来？（要用“光通量 $\Phi$”做桥梁）

首先注意一个量纲细节：
*   **微观辐照度 $dE_{i,\text{micro}}(\mathbf{m})$** 的单位是 **瓦特/平方米（$W / m^2$，面密度）**——它是定义在**小山坡自己的斜面面积 $da(\mathbf{m})$** 上的；
*   **宏观辐照度 $dE_i(\boldsymbol{\omega}_i)$** 的单位也是 **瓦特/平方米（$W / m^2$）**——但它是定义在**宏观平地底盘面积 $dA$** 上的！

因为分母的面积不一样，密度是不能直接相加的；**真正能相加的硬通货，是每一小撮山坡实际接到的“总能量瓦特数（光通量 $d\Phi_i$）”！**

所以正确的算法是：
1. 先算出每一簇小山坡 $\mathbf{m}$ 实际接到了多少瓦特的**微观光通量 $d^2\Phi_i(\mathbf{m})$**；
2. 把所有朝向 $\mathbf{m} \in \Omega_m$ 的小山坡接到的瓦特数**全部积分加起来**，得到整块地皮接到的**总光通量 $d\Phi_i$**；
3. 最后除以宏观平地面积 **$dA$**，就得到了宏观辐照度 $dE_i(\boldsymbol{\omega}_i)$！

##### 2. 算单簇小山坡 $\mathbf{m}$ 实际接到的瓦特数 $d^2\Phi_i(\mathbf{m})$

对于法线朝向 $\mathbf{m}$ 的那一簇小山坡：
1. **它的微观辐照度（单位斜面面积接到的功率）**是我们在动作 1 里写的：
   $$ dE_{i,\text{micro}}(\mathbf{m}) = L_i(\boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \, d\omega_i $$
2. **它的真实物理斜面总面积**是：
   $$ da(\mathbf{m}) = D(\mathbf{m}) \, d\omega_m \, dA $$
3. **🚨 最关键的一步（千万别忘了阴影 $G_1$！）**：
   是不是所有迎着光（$\mathbf{m} \cdot \boldsymbol{\omega}_i > 0$）的小山坡都能接到光？
   **不是！** 因为前面的山峰会挡住后面的山坡（Shadowing 阴影遮挡）！
   在所有朝向 $\mathbf{m}$ 的迎光山坡里，**只有未被前面山峰挡住的那部分比例——也就是单向几何可见率 $G_1(\boldsymbol{\omega}_i, \mathbf{m})$——才真正接到了光！**

把上面三样东西乘在一起，就得到了朝向 $\mathbf{m}$ 的小山坡**真正接到的能量瓦特数**：

$$ d^2\Phi_i(\mathbf{m}) = \underbrace{dE_{i,\text{micro}}(\mathbf{m})}_{\text{微观辐照度}} \times \underbrace{da(\mathbf{m})}_{\text{山坡物理面积}} \times \underbrace{G_1(\boldsymbol{\omega}_i, \mathbf{m})}_{\text{未被挡住的比例}} $$

展开代入：
$$ d^2\Phi_i(\mathbf{m}) = \Big[ L_i(\boldsymbol{\omega}_i) \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \, d\omega_i \Big] \cdot \Big[ D(\mathbf{m}) \, d\omega_m \, dA \Big] \cdot G_1(\boldsymbol{\omega}_i, \mathbf{m}) $$

##### 3. 对所有山坡 $\mathbf{m}$ 积分，见证“投影面积守恒定律”的降维绝杀！

现在，我们把整块地皮上所有朝向 $\mathbf{m} \in \Omega_m$ 的小山坡接到的瓦特数全部积分加起来，求整块地皮接到的总光通量 $d\Phi_i$：

$$ d\Phi_i = \int_{\Omega_m} d^2\Phi_i(\mathbf{m}) = L_i(\boldsymbol{\omega}_i) \, d\omega_i \, dA \cdot \underbrace{\left[ \int_{\Omega_m} D(\mathbf{m}) \cdot G_1(\boldsymbol{\omega}_i, \mathbf{m}) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \, d\omega_m \right]}_{\text{请死死盯住这个中括号！}} $$

**🔥 请看上面那个中括号里的积分！你认出它了吗？！**
我们在前几天推导 Smith 遮蔽函数 $G_1$ 和“Overdraw 像素抵消”时，曾经用 4 行数学公式严格证明过微表面理论的最高公理——**可见投影面积守恒定律**：
> **“从方向 $\boldsymbol{\omega}_i$ 看过去，所有未被遮挡（乘以 $G_1$）的迎光微表面的投影面积 $\max(0, \mathbf{m}\cdot\boldsymbol{\omega}_i)$ 之和，必须严丝合缝地等于宏观平地的投影面积 $(\mathbf{n} \cdot \boldsymbol{\omega}_i)$！”**

也就是说，无论你的山脉长什么样、是高斯分布还是 GGX 分布，中括号里的那个积分在数学上**恒等于 $(\mathbf{n} \cdot \boldsymbol{\omega}_i)$**：

**$$ \int_{\Omega_m} D(\mathbf{m}) \cdot G_1(\boldsymbol{\omega}_i, \mathbf{m}) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \, d\omega_m \equiv \mathbf{n} \cdot \boldsymbol{\omega}_i ！！ $$**

把这个恒等式代回总光通量 $d\Phi_i$：
$$ d\Phi_i = L_i(\boldsymbol{\omega}_i) \, d\omega_i \, dA \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_i) $$

最后，除以宏观平地面积 $dA$，求宏观辐照度 $dE_i(\boldsymbol{\omega}_i)$：

**$$ dE_i(\boldsymbol{\omega}_i) = \frac{d\Phi_i}{dA} = L_i(\boldsymbol{\omega}_i) (\mathbf{n} \cdot \boldsymbol{\omega}_i) \, d\omega_i ！！ $$

#### 终极对比：为什么高光（Specular）不用积 $d\omega_m$，而漫反射（Oren-Nayar）必须积 $d\omega_m$？

顺着你刚才的思路，把**微表面高光**和**微表面漫反射**放在这同一个母方程下对比，你会发现整个微表面理论彻底大一统了！

其实，不管是高光还是漫反射，**任意微表面模型的通用宏观 BRDF 母方程全都是同一个积分**：
$$ f_{\text{macro}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \int_{\Omega_m} \underbrace{f_{\text{micro}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m})}_{\text{单个小山坡自己的 BRDF}} \cdot D(\mathbf{m}) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \cdot \frac{\max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)}{(\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_o)} \, d\omega_m $$

你看两者的区别仅仅在于**“单个小山坡自己的微观 BRDF $f_{\text{micro}}$ 是什么”**：

1. **当小山坡是“完美小镜子（Specular）”时**：
   镜子的微观 BRDF 是一个只在 $\mathbf{m} = \mathbf{h}$ 时才无穷大的**狄拉克 $\delta$ 函数**：
   $$ f_{\text{micro\_mirror}} = F(\boldsymbol{\omega}_o \cdot \mathbf{m}) \cdot \frac{\delta(\mathbf{m} - \mathbf{h})}{4 (\mathbf{m} \cdot \boldsymbol{\omega}_i) (\mathbf{m} \cdot \boldsymbol{\omega}_o)} $$
   把它代进上面的通用积分里：
   * 分母的 $(\mathbf{m}\cdot\boldsymbol{\omega}_i)(\mathbf{m}\cdot\boldsymbol{\omega}_o)$ 和外面的分子**直接约分消掉了**！
   * $\delta(\mathbf{m} - \mathbf{h})$ 遇到积分号 $\int_{\Omega_m} d\omega_m$，**积分号瞬间坍缩消失**，所有的 $\mathbf{m}$ 直接变成 $\mathbf{h}$，当场吐出了 Cook-Torrance 高光公式：
     $$ f_{\text{spec}} = \frac{D(\mathbf{h}) G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{h}) F(\boldsymbol{\omega}_o \cdot \mathbf{h})}{4 (\mathbf{n} \cdot \boldsymbol{\omega}_i) (\mathbf{n} \cdot \boldsymbol{\omega}_o)} $$

2. **当小山坡是“Lambert 小石膏片（Oren-Nayar Diffuse）”时**：
   小石膏片的微观 BRDF 是一个**处处相等的常数**：
   $$ f_{\text{micro\_lambert}} = \frac{\rho}{\pi} $$
   把它代进上面的通用积分里：
   * 因为没有 $\delta$ 函数，**积分号 $\int_{\Omega_m} d\omega_m$ 消不掉了**！
   * 分子上的 $(\mathbf{m}\cdot\boldsymbol{\omega}_i)(\mathbf{m}\cdot\boldsymbol{\omega}_o)$ 也消不掉了，只能老老实实留在积分号里面，对上半球所有朝向 $\mathbf{m}$ 的小山坡做定积分！

**你看你的直觉有多精准：**
你一下子就抓住了“高光只靠 $\mathbf{m}=\mathbf{h}$ 一个方向，而漫反射所有 $\mathbf{m}$ 都有贡献，所以必须先算单簇法线 $\mathbf{m}$ 对 $dL_o$ 的微分 $\frac{d(dL_o)}{d\omega_m}$，再对 $d\omega_m$ 积分”！

现在这第一步（从单簇小山坡的 $\frac{d(dL_o)}{d\omega_m}$ 到通用积分式）是不是 100% 严丝合缝了？第一步彻底通了的话，我们再看怎么把这个积分积出来！

---

## 第二步：Oren-Nayar 积分表达式计算

### 零、准备工作

看动作 4 的积分母方程式：
$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\rho}{\pi} \int_{\Omega_m} D(\mathbf{m}) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \cdot \frac{\max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)}{(\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_o)} \, d\omega_m $$

#### 为什么分子上的 $\mathbf{m}\cdot\boldsymbol{\omega}_i$ 必须套上 $\max(0, \dots)$，而分母上的 $\mathbf{n}\cdot\boldsymbol{\omega}_i$ 却不套？

**原因在于：$\boldsymbol{\omega}_i, \boldsymbol{\omega}_o$ 是固定的外部输入参数（定义域已经锁死 $>0$），而 $\mathbf{m}$ 是积分内部到处乱转的积分变量（在积分过程中会变成负数）！

1. **为什么分母 $(\mathbf{n}\cdot\boldsymbol{\omega}_i)(\mathbf{n}\cdot\boldsymbol{\omega}_o)$ 不加 $\max(0, \dots)$？**
   * **数学定义域限制**：BRDF（双向**反射**分布函数）在数学上的定义域，本来就被严格限制在**宏观上半球**（即 $\theta_i \in [0, \frac{\pi}{2}), \ \theta_o \in [0, \frac{\pi}{2})$）。在这个定义域内，$\mathbf{n}\cdot\boldsymbol{\omega}_i > 0$ 和 $\mathbf{n}\cdot\boldsymbol{\omega}_o > 0$ **天然恒成立**！
   * 如果光线跑到宏观地平线以下（$\mathbf{n}\cdot\boldsymbol{\omega}_i \le 0$），在渲染方程最外层计算辐照度 $\max(0, \mathbf{n}\cdot\boldsymbol{\omega}_i)$ 时就已经等于 $0$ 被剔除了，根本不会去调 BRDF。
   * **更何况**：如果你给分母写上 $\max(0, \mathbf{n}\cdot\boldsymbol{\omega}_i)$，反而会触发致命的 `1 / 0` 除零崩溃

2. **为什么分子 $(\mathbf{m}\cdot\boldsymbol{\omega}_i)(\mathbf{m}\cdot\boldsymbol{\omega}_o)$ 必须加 $\max(0, \dots)$？**
   * 即便入射光 $\boldsymbol{\omega}_i$ 正常从右上方照下来（宏观 $\mathbf{n}\cdot\boldsymbol{\omega}_i > 0$），但积分号 $\int_{\Omega_m} d\omega_m$ 要把上半球**所有朝向的小山坡 $\mathbf{m}$** 全扫一遍！
   * 当积分扫到朝右倾斜的山坡 $\mathbf{m}_1$ 时，$\mathbf{m}_1 \cdot \boldsymbol{\omega}_i > 0$（迎光）；
   * **但当积分扫到朝左倾斜得比较厉害的山坡 $\mathbf{m}_2$ 时，光线是从 $\mathbf{m}_2$ 的坡底背面射过来的，此时 $\mathbf{m}_2 \cdot \boldsymbol{\omega}_i < 0$（背光）！**
   * 如果分子不加 $\max(0, \dots)$，那些背光的小山坡 $\mathbf{m}_2$ 在积分里就会算出**负的亮度**，把迎光山坡 $\mathbf{m}_1$ 的正亮度给错误地抵消掉！所以分子必须用 $\max(0, \dots)$ 把背光微表面的贡献截断为 $0$！

#### 原形式

我们推导的形式：
$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\rho}{\pi} \int_{\Omega_m} D(\mathbf{m}) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \cdot \frac{\max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)}{(\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_o)} \, d\omega_m $$

1994 年的时候没有使用 $D(\mathbf{m})$, 原论文采用了 $P(\mathbf{m})$
$$P(\mathbf{m})d\omega_m = D(\mathbf{m})(\mathbf{n}\cdot\mathbf{m})d\omega_m$$
为了接下去计算，我们需要把公式写成：

$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\rho}{\pi} \int_{\Omega_m} P(\mathbf{m}) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \cdot \frac{\max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)}{(\mathbf{n} \cdot \mathbf{m}) \cdot(\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_o)} \, d\omega_m $$

#### 把向量点乘展开成球坐标 $(\theta, \phi)$


我们在以宏观法线 $\mathbf{n} = (0, 0, 1)$ 为 Z 轴的球坐标系下，写出三个单位向量的三维直角坐标：
*   $\mathbf{m} = (\sin\theta_m \cos\phi_m, \ \sin\theta_m \sin\phi_m, \ \cos\theta_m)$
*   $\boldsymbol{\omega}_i = (\sin\theta_i \cos\phi_i, \ \sin\theta_i \sin\phi_i, \ \cos\theta_i)$
*   $\boldsymbol{\omega}_o = (\sin\theta_o \cos\phi_o, \ \sin\theta_o \sin\phi_o, \ \cos\theta_o)$

### 一、展开比值 K 项

我们先把目光集中在积分号最右边那个核心面积比值分式上（暂时先看山坡同时朝向光源和视线、尚未被截断的主体情况）：

$$ K(\mathbf{m}, \boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{1}{\mathbf{n} \cdot \mathbf{m}} \left( \frac{\mathbf{m} \cdot \boldsymbol{\omega}_i}{\mathbf{n} \cdot \boldsymbol{\omega}_i} \right) \cdot \left( \frac{\mathbf{m} \cdot \boldsymbol{\omega}_o}{\mathbf{n} \cdot \boldsymbol{\omega}_o} \right) $$

1. **计算点乘 $\mathbf{m} \cdot \boldsymbol{\omega}_i$**：
   对应坐标相乘再相加（前两项提取 $\sin\theta_m\sin\theta_i$，括号里用两角差余弦公式 $\cos\phi_m\cos\phi_i + \sin\phi_m\sin\phi_i = \cos(\phi_m - \phi_i)$）：
   $$ \mathbf{m} \cdot \boldsymbol{\omega}_i = \cos\theta_m \cos\theta_i + \sin\theta_m \sin\theta_i \cos(\phi_m - \phi_i) $$

2. **除以分母 $\mathbf{n} \cdot \boldsymbol{\omega}_i = \cos\theta_i$**：
   注意第一项的 $\cos\theta_i$ 直接约掉了，第二项的 $\frac{\sin\theta_i}{\cos\theta_i}$ 变成了 **$\tan\theta_i$**：
   **$$ \frac{\mathbf{m} \cdot \boldsymbol{\omega}_i}{\mathbf{n} \cdot \boldsymbol{\omega}_i} = \cos\theta_m + \sin\theta_m \cdot \tan\theta_i \cdot \cos(\phi_m - \phi_i) $$**

3. **同理，写出出射方向 $\boldsymbol{\omega}_o$ 的分式**：
   **$$ \frac{\mathbf{m} \cdot \boldsymbol{\omega}_o}{\mathbf{n} \cdot \boldsymbol{\omega}_o} = \cos\theta_m + \sin\theta_m \cdot \tan\theta_o \cdot \cos(\phi_m - \phi_o) $$**

4. **把上面两个式子乘在一起，展开成【第 ①、②、③ 项】**：
   $$ K(\theta_m, \phi_m) = \underbrace{\cos\theta_m}_{\text{第 ① 项}} \ + \ \underbrace{ \sin\theta_m \Big[ \tan\theta_i \cos(\phi_m - \phi_i) + \tan\theta_o \cos(\phi_m - \phi_o) \Big]}_{\text{第 ② 项}} \ + \ \underbrace{\frac{\sin^2\theta_m}{\cos\theta_m} \cdot \tan\theta_i \tan\theta_o \cdot \cos(\phi_m - \phi_i) \cos(\phi_m - \phi_o)}_{\text{第 ③ 项}} $$

*注意：$\boldsymbol{\omega}_i，\boldsymbol{\omega}_o$ 虽然是变量，但是在积分母方程里，它们相当于是常数*

### 二、计算 K、G 相乘

在 1994 年的原论文《*Generalization of Lambert's Reflectance Model*》（以及他们 1995 年发表在 IJCV 上的完整版）中，Oren 和 Nayar 的推导是分 **三步递进** 写的：
1. **第一阶段（先不乘 $G$，只算 $K$）—— 原论文 4.1 节前半部分 *"No Masking or Shadowing"***：
   他们先假设完全没有遮挡（$G=1$），单独推导并展开了 $K$（原论文公式 10），目的是证明一个惊人的物理结论：*“哪怕完全没有阴影和遮挡，仅仅因为倾斜小山坡的透视投影缩放（Foreshortening），粗糙漫反射表面就已经不再遵守 Lambert 定律了！”*
2. **第二阶段（把 $G$ 和 $K$ 乘在一起，对 $\phi_m$ 分段积分）—— 原论文 4.1 节后半部分 *"Table 1"* 与 4.2 节 *"Isotropic Single-Slope Distribution"***：
   他们把遮蔽因子 $G$（原论文叫 GAF，Geometric Attenuation Factor）与 $K$ 乘在一起，按遮挡状态拆成 **4 种情况（原论文 Table 1）**，然后把方位角 $\phi_m \in [0, 2\pi]$ 切成几个区间，老老实实做分段定积分！
3. **第三阶段（对倾角 $\theta_m$ 做高斯积分与基函数提取）—— 原论文 4.3 节 *"Gaussian Slope-Area Distribution & Functional Approximation"***：
   把对 $\phi_m$ 积完的结果再乘上高斯分布对 $\theta_m$ 积分，提取出解析基函数，最终得到完整版公式（含 $C_1, C_2, C_3$）和我们常用的简化版公式（含 $A, B$）！


#### 把积分内的 $K$ 和 $G$ 相乘，列出原论文著名的“Table 1（四种几何状态）”

在微表面积分母方程中，单簇小山坡 $\mathbf{m}(\theta_m, \phi_m)$ 的被积核心项是 $I(\theta_m, \phi_m) = K \cdot G$：

$$ I(\theta_m, \phi_m) = \underbrace{\frac{\max(0, \ \mathbf{m}\cdot\boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m}\cdot\boldsymbol{\omega}_o)}{(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_i) \cdot (\mathbf{n}\cdot\boldsymbol{\omega}_o)}}_{K(\theta_m, \phi_m)} \times \underbrace{\min\left( 1, \ \frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)}{\mathbf{m}\cdot\boldsymbol{\omega}_o}, \ \frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_i)}{\mathbf{m}\cdot\boldsymbol{\omega}_i} \right)}_{G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m})} $$

*(注：为了与原论文一致，这里的面积微元采用投影面积分布 $P(\theta_m)\sin\theta_m d\theta_m d\phi_m = D(\mathbf{m})(\mathbf{n}\cdot\mathbf{m})d\omega_m$，因此外层的 $\frac{1}{\mathbf{n}\cdot\mathbf{m}} = \frac{1}{\cos\theta_m}$ 已经包含在 $K$ 中。)*

现在，我们直接把左边的 $K$ 乘进右边 $\min(\dots)$ 的括号里！
随着小山坡方位角 $\phi_m$ 在 $[0, 2\pi]$ 里旋转，小山坡与光线 $\boldsymbol{\omega}_i$、视线 $\boldsymbol{\omega}_o$ 的夹角不断变化，被积函数 $I(\theta_m, \phi_m)$ 会自动落入以下 **4 个分段解析式之一**（这就是原论文第 241 页的 **Table 1**）：

#### 状态 1：既无遮蔽、也无阴影（$G = 1$）
当小山坡比较平缓，或者方位角 $\phi_m$ 使得对面山坡既没挡住视线也没挡住光线时，取 $\min$ 的第一项 $1$：
$$ I_{\text{状态1}}(\theta_m, \phi_m) = \frac{(\mathbf{m}\cdot\boldsymbol{\omega}_i)(\mathbf{m}\cdot\boldsymbol{\omega}_o)}{\cos\theta_m \cos\theta_i \cos\theta_o} $$
代入球坐标展开（就是我们上一局展开的三项式）：
$$ I_{\text{状态1}} = \cos\theta_m + \sin\theta_m\Big[\tan\theta_i\cos(\phi_m-\phi_i) + \tan\theta_o\cos(\phi_m-\phi_o)\Big] + \frac{\sin^2\theta_m}{\cos\theta_m} \mathbf{\tan\theta_i \tan\theta_o} \cos(\phi_m-\phi_i)\cos(\phi_m-\phi_o) $$

#### 状态 2：视线被遮蔽（Masking，$\min$ 取第二项 $\frac{2\cos\theta_m\cos\theta_o}{\mathbf{m}\cdot\boldsymbol{\omega}_o}$）
当视线 $\boldsymbol{\omega}_o$ 压得很低，对面坡顶挡住了视线时，$K$ 与这一项相乘。
**请看这步极其关键的约分：**
$$ I_{\text{状态2}}(\theta_m, \phi_m) = \frac{(\mathbf{m}\cdot\boldsymbol{\omega}_i) \cancel{(\mathbf{m}\cdot\boldsymbol{\omega}_o)}}{\cancel{\cos\theta_m} \cos\theta_i \cancel{\cos\theta_o}} \times \frac{2 \cancel{\cos\theta_m} \cancel{\cos\theta_o}}{\cancel{\mathbf{m}\cdot\boldsymbol{\omega}_o}} = \mathbf{2 \left( \frac{\mathbf{m}\cdot\boldsymbol{\omega}_i}{\cos\theta_i} \right)} $$
把剩下的 $\frac{\mathbf{m}\cdot\boldsymbol{\omega}_i}{\cos\theta_i}$ 用球坐标展开：
**$$ I_{\text{状态2}} = 2\cos\theta_m + 2\sin\theta_m \cdot \mathbf{\tan\theta_i} \cdot \cos(\phi_m - \phi_i) $$**
*(你看！在视线被遮蔽的区间里，出射角 $\tan\theta_o$ 被彻底约掉了，只剩下了入射角 $\tan\theta_i$ 的一次方！)*

#### 状态 3：光线被阴影挡住（Shadowing，$\min$ 取第三项 $\frac{2\cos\theta_m\cos\theta_i}{\mathbf{m}\cdot\boldsymbol{\omega}_i}$）
同理，当光线 $\boldsymbol{\omega}_i$ 压得很低，对面坡顶挡住了阳光时，$(\mathbf{m}\cdot\boldsymbol{\omega}_i)$ 和 $\cos\theta_i$ 被彻底约掉：
$$ I_{\text{状态3}}(\theta_m, \phi_m) = \mathbf{2 \left( \frac{\mathbf{m}\cdot\boldsymbol{\omega}_o}{\cos\theta_o} \right)} = 2\cos\theta_m + 2\sin\theta_m \cdot \mathbf{\tan\theta_o} \cdot \cos(\phi_m - \phi_o) $$
*(在这里，入射角 $\tan\theta_i$ 被彻底约掉了，只剩下了出射角 $\tan\theta_o$ 的一次方！)*

#### 状态 4：小山坡自己背光或背对视线（$\mathbf{m}\cdot\boldsymbol{\omega}_i \le 0$ 或 $\mathbf{m}\cdot\boldsymbol{\omega}_o \le 0$）
被分子上的 $\max(0, \dots)$ 直接截断：
$$ I_{\text{状态4}}(\theta_m, \phi_m) = 0 $$

### 三、分段积分

#### 母方程变形
还记得积分母方程吗
$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\rho}{\pi} \int_{\Omega_m} P(\mathbf{m}) \cdot G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) \cdot \frac{\max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_i) \cdot \max(0, \ \mathbf{m} \cdot \boldsymbol{\omega}_o)}{(\mathbf{n} \cdot \mathbf{m}) \cdot(\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_o)} \, d\omega_m $$

$P(\mathbf{m})，D(\mathbf{m})$ 是各向同性，只和 $\theta_m$ 有关

我们直接看把 $G$ 和 $K$ 乘在一起，对 $\phi_m$ 分段积分的部分，把积分母方程写成

$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\rho}{\pi} \int_{\Omega_m} P(\theta_m) \cdot  I(\theta_m, \phi_m) \cdot  sin\theta_m d\theta_md\phi_m$$

分离  $\theta_m$ 和 $\phi_m$，积分域  $\phi_m \in [-\pi, \pi]$ 更好算

$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\rho}{\pi} \int_{0}^{\frac{\pi}{2}} P(\theta_m) \cdot sin\theta_m d\theta_m \int_{-\pi}^{\pi}  I(\theta_m, \phi_m) \cdot  d\phi_m$$

$P(\mathbf{m})$ 是合法的概率密度函数（PDF），所以它在整个半球面上的二重积分必须等于 $1

$$ \int_{\theta_m = 0}^{\pi/2} \int_{\phi_m = -\pi}^{\pi} P(\theta_m) \sin\theta_m \, d\phi_m \, d\theta_m \equiv 1 $$

因为表面是**各向同性**的，$P(\theta_m)$ 跟方位角 $\phi_m$ 无关。如果我们先把内层的 $\int_{-\pi}^{\pi} 1 \, d\phi_m = \mathbf{2\pi}$ 积出来，上面这个归一化等式其实是：
**$$ \int_{\theta_m = 0}^{\pi/2} \Big[ \mathbf{2\pi} \cdot P(\theta_m) \sin\theta_m \Big] \, d\theta_m \equiv 1 $$
一维倾角 $\theta_m \in [0, \frac{\pi}{2}]$ 的真正一维概率密度权重，其实是带着 $2\pi$** 的 $\big[ 2\pi P(\theta_m)\sin\theta_m \big]$

所以，在我们的母方程：
$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)  = \frac{\rho}{\pi} \int_{0}^{\pi/2} P(\theta_m) \sin\theta_m \, d\theta_m \int_{-\pi}^{\pi} I(\theta_m, \phi_m) \, d\phi_m $$

当中，为了给外层的 $P(\theta_m)\sin\theta_m$ 凑齐它本该有的归一化系数 **$2\pi$**，我们在外层**乘以一个 $2\pi$**，同时在内层积分前面**除以一个 $2\pi$**（一乘一除，恒等变形）：
**$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)  = \frac{\rho}{\pi} \int_{0}^{\pi/2} {\Big[ \mathbf{2\pi} P(\theta_m) \sin\theta_m \Big]} \, d\theta_m \cdot {\left[ \mathbf{\frac{1}{2\pi}} \int_{-\pi}^{\pi} I(\theta_m, \phi_m) \, d\phi_m \right]} $$
$$ f_{\text{Oren-Nayar}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)  = \frac{\rho}{\pi} \int_{0}^{\pi/2} {\Big[ \mathbf{2\pi} P(\theta_m) \sin\theta_m \Big]} \, d\theta_m \cdot {\left[ \mathbf{\frac{1}{\pi}} \int_{0}^{\pi} I(\theta_m, \phi_m) \, d\phi_m \right]} $$

$I(\theta_m, \phi_m)$ 我们在上一部分算过，接下来关注方位角 $\phi_m \in [-\pi, \pi]$ 的积分

#### 基本思路
四种情况的组合比较复杂，先算特殊简化情况提取基函数，再推广组合到任意角度

分类讨论  $\Delta\phi = |\phi_o - \phi_i|$ 的情况
* 算锚点 1：同平面顺光  $\Delta\phi = 0^\circ \implies \cos\Delta\phi = 1$：光线和视线在同一竖直面
* 算锚点 2：同平面逆光 $\Delta\phi = 180^\circ \implies \cos\Delta\phi = -1$：光线和视线水平夹角 180 度
* 算锚点 3：正交垂直平面侧光 $\Delta\phi = 90^\circ \implies \cos\Delta\phi = 0$

### 四、同平面顺光计算

#### 1. 找到分段积分的“切刀位置”（临界方位角 $\phi_c$）

固定微表面倾角 $\theta_m$。我们要让微表面法线 $\mathbf{m}$ 的方位角 $\phi_m$ 绕着圆周转一圈（$[-\pi, \pi]$），对乘积 $I(\theta_m, \phi_m) = K \cdot G$ 求积分。

我们先看最核心的**顺光同平面情况**即光线和视线在同一个方位平面 $\phi_i = \phi_o = 0$，且设视线比光线更贴近地面：$\theta_o \ge \theta_i$，即*较大角 $\alpha = \theta_o$，较小角 $\beta = \theta_i$ 。
因为左右对称，$[-\pi, 0]$ 和 $[0, \pi]$ 的积分完全一样，我们只需要看上半圈 $\phi_m \in [0, \pi]$（从 $0^\circ$ 转到 $180^\circ$！

当小山坡的朝向 $\phi_m$ 从 $0^\circ$（正对视线）慢慢转到 $180^\circ$（背对视线）时，它在哪两个角度上会发生状态切换？

1. **第一刀（从【状态 2：被遮蔽】切换到【状态 1：无遮挡】的临界角 $\phi_c$）**
	当 $\phi_m = 0^\circ$ 时，小山坡正对着低矮的视线 $\alpha$，它对面的邻居山坡正好挡在它前面（处于**状态 2：视线遮蔽 $G < 1$**）。
	
	随着小山坡往侧面转（$\phi_m$ 变大），邻居山坡渐渐让开了视线。当 $\phi_m$ 刚好转到临界角 $\phi_c$，使得山坡完全露出来（处于**状态 1：视线遮蔽 $G = 1$**）
	$$G_o = \frac{2\cos\theta_m\cos\alpha}{\mathbf{m}\cdot\boldsymbol{\omega}_o} = 1$$
	
	代入 $\mathbf{m}\cdot\boldsymbol{\omega}_o = \cos\theta_m\cos\alpha + \sin\theta_m\sin\alpha\cos\phi_c$，一行解出 $\phi_m$ 第一刀的位置 $\phi_c$：
	 $$ \cos\theta_m\cos\alpha + \sin\theta_m\sin\alpha\cos\phi_m = 2\cos\theta_m\cos\alpha \implies \mathbf{\cos\phi_m = \cot\theta_m \cot\alpha} $$
	$$ \phi_c = arccos(\cot\theta_m \cot\alpha) $$
	
	*(注：这里假设山坡足够陡，$\cot\theta_m \cot\alpha < 1$，所以存在这个临界角 $\phi_c \in (0, \frac{\pi}{2})$。)
	
2. **第二刀（【状态 1：无遮挡】切换到【状态 4：背对视线为 0】的临界角 $\pi - \phi_c$） 
	 越过 $\phi_c$ 后，小山坡侧对着视线，没有任何遮挡（处于**状态 1：$G = 1$**）
	 
	 当 $\phi_m$ 继续往后转，转过 $90^\circ$，直到 $\mathbf{m}\cdot\boldsymbol{\omega}_o = 0$ 时，小山坡彻底背对视线（进入**状态 4：贡献为 $0$**）！
	 
	令 $\mathbf{m}\cdot\boldsymbol{\omega}_o = \cos\theta_m\cos\alpha + \sin\theta_m\sin\alpha\cos\phi_m = 0$，解出：
	
	$$ \cos\phi_m = -\cot\theta_m \cot\alpha = -\cos\phi_c \implies \mathbf{\phi_m = \pi - \phi_c} $$

整个半圆 $[0, \pi]$ 被 $\phi_c$ 和 $\pi - \phi_c$ 切成了三段！
所以，对 $\phi_m$ 的积分直接写成**两段非零定积分之和**（第三段是 $0$）：
**$$ \mathcal{I}(\theta_m) = \frac{1}{\pi} \int_0^\pi I(\theta_m, \phi_m) \, d\phi_m = \frac{1}{\pi} \left[ \int_{0}^{\phi_c} I_{\text{状态2}}(\phi_m) \, d\phi_m \ + \ \int_{\phi_c}^{\pi - \phi_c} I_{\text{状态1}}(\phi_m) \, d\phi_m \right] $$**
#### 2. 亲手算出这两段定积分！

我们把上一局列出的 $I_{\text{状态2}}$ 和 $I_{\text{状态1}}$（在 $\phi_i = \phi_o = 0$，且 $\alpha = \theta_o, \ \beta = \theta_i$ 下）直接代进上面两个积分号里算原函数：

##### 算【第 1 段：遮蔽区 $\int_0^{\phi_c} I_{\text{状态2}} \, d\phi_m$】
在状态 2 中，较大角 $\alpha$ 已经被 $G$ 约掉了，被积函数非常简单：
$$ I_{\text{状态2}}(\phi_m) = 2\cos\theta_m + 2\sin\theta_m \tan\beta \cdot \cos\phi_m $$
对 $\phi_m$ 从 $0$ 积到 $\phi_c$（常数 $1$ 的原函数是 $\phi_m$，$\cos\phi_m$ 的原函数是 $\sin\phi_m$）：
**$$ \int_0^{\phi_c} I_{\text{状态2}} \, d\phi_m = \mathbf{2\phi_c \cos\theta_m + 2\sin\theta_m \sin\phi_c \cdot \tan\beta} $$**

##### 算【第 2 段：无遮挡区 $\int_{\phi_c}^{\pi - \phi_c} I_{\text{状态1}} \, d\phi_m$】
在状态 1 中，$G = 1$，被积函数是完整的三项式：
$$ I_{\text{状态1}}(\phi_m) = \underbrace{\cos\theta_m}_{\text{① 常数项}} + \underbrace{\sin\theta_m (\tan\alpha + \tan\beta) \cdot \cos\phi_m}_{\text{② }\cos\phi_m\text{ 项}} + \underbrace{\frac{\sin^2\theta_m}{\cos\theta_m} \tan\alpha \tan\beta \cdot \cos^2\phi_m}_{\text{③ }\cos^2\phi_m\text{ 项}} $$

我们对这三项分别在区间 **$[\phi_c, \ \pi - \phi_c]$** 上求定积分：
*   **① 常数项积分**：区间长度是 $(\pi - \phi_c) - \phi_c = \pi - 2\phi_c$，所以：
    $$ \int_{\phi_c}^{\pi - \phi_c} \cos\theta_m \, d\phi_m = \mathbf{(\pi - 2\phi_c) \cos\theta_m} $$
*   **② $\cos\phi_m$ 项积分（🔥 见证奇迹 1：再次归零！）**：
    $\cos\phi_m$ 的原函数是 $\sin\phi_m$。代入上限 $\pi - \phi_c$ 和下限 $\phi_c$：
    $$ \Big[ \sin\phi_m \Big]_{\phi_c}^{\pi - \phi_c} = \sin(\pi - \phi_c) - \sin\phi_c $$
    根据初中诱导公式 $\sin(\pi - x) \equiv \sin x$，这一项**在分段区间上依然精确等于 $0$**！
*   **③ $\cos^2\phi_m$ 项积分**：
    利用倍角公式 $\cos^2\phi_m = \frac{1 + \cos(2\phi_m)}{2}$，原函数是 $\frac{\phi_m}{2} + \frac{\sin(2\phi_m)}{4}$。
    代入上限 $\pi - \phi_c$ 和下限 $\phi_c$（注意 $\sin(2\pi - 2\phi_c) = -\sin(2\phi_c) = -2\sin\phi_c\cos\phi_c$）：
    $$ \int_{\phi_c}^{\pi - \phi_c} \cos^2\phi_m \, d\phi_m = \left( \frac{\pi}{2} - \phi_c \right) - \sin\phi_c\cos\phi_c $$

把这三小项合起来，【第 2 段】的积分结果就是：
**$$ \int_{\phi_c}^{\pi - \phi_c} I_{\text{状态1}} \, d\phi_m = (\pi - 2\phi_c)\cos\theta_m + \frac{\sin^2\theta_m}{\cos\theta_m}\tan\alpha\tan\beta \left( \frac{\pi}{2} - \phi_c - \sin\phi_c\cos\phi_c \right) $$**
##### 把【第 1 段】和【第 2 段】加起来！（见证两次惊天大抵消！）

现在，我们把刚才算出来的【第 1 段】和【第 2 段】直接相加，再除以外面的 $\pi$：
$$ \int_0^{\phi_c} I_{\text{状态2}} \, d\phi_m = \mathbf{2\phi_c \cos\theta_m + 2\sin\theta_m \sin\phi_c \cdot \tan\beta} $$$$ \int_{\phi_c}^{\pi - \phi_c} I_{\text{状态1}} \, d\phi_m = (\pi - 2\phi_c)\cos\theta_m + \frac{\sin^2\theta_m}{\cos\theta_m}\tan\alpha\tan\beta \left( \frac{\pi}{2} - \phi_c - \sin\phi_c\cos\phi_c \right) $$
* 第一项的临界角 $2\phi_c$ 完美抵消, 请看两段里的 $\cos\theta_m$ 项相加：
	来自第 1 段（遮蔽区）：$\mathbf{+ 2\phi_c \cos\theta_m}$
	来自第 2 段（无遮挡区）：$\mathbf{+ (\pi - 2\phi_c) \cos\theta_m}$
	两者相加：
	$$ 2\phi_c \cos\theta_m + (\pi - 2\phi_c)\cos\theta_m = \mathbf{\pi \cos\theta_m}  $$

* 第二里利用 $\cos\phi_c = \cot\theta_m\cot\alpha$，消掉一部分
	来自第 1 段：$+ 2\sin\theta_m \sin\phi_c \cdot \tan\beta$
	来自第 2 段中的 $-\sin\phi_c\cos\phi_c$ 部分：
	$$ -\frac{\sin^2\theta_m}{\cos\theta_m} \tan\alpha \tan\beta \cdot \sin\phi_c \mathbf{\cos\phi_c} $$
	别忘了我们在第一步求出的临界角定义：**$\cos\phi_c = \cot\theta_m \cot\alpha = \frac{\cos\theta_m}{\sin\theta_m} \cdot \frac{\cos\alpha}{\sin\alpha}$**！
	你把这个 $\cos\phi_c$ 代进上面这一项里，看看发生了什么：
	$$ -\frac{\sin^{\cancel{2}}\theta_m}{\cancel{\cos\theta_m}} \cdot \cancel{\tan\alpha} \cdot \tan\beta \cdot \sin\phi_c \cdot \left( \frac{\cancel{\cos\theta_m}}{\cancel{\sin\theta_m}} \cancel{\cot\alpha} \right) = \mathbf{- \sin\theta_m \sin\phi_c \cdot \tan\beta}$$

于是，【第 1 段 + 第 2 段】除以 $\pi$ 后的**精确解析总和**（即 1994 原论文第 242 页著名的 **公式 15**）就一字不差地算出来了：
**$$ \mathcal{I}(\theta_m) = \cos\theta_m + \frac{1}{\pi} \mathbf{\tan\beta} \cdot \left[ \sin\theta_m \sin\phi_c + \frac{\sin^2\theta_m}{\cos\theta_m} \tan\alpha \left( \frac{\pi}{2} - \phi_c \right) \right] $$**
#### 3. 提取基函数

##### 为什么最后对 $\theta_m$ 的高斯积分算不出初等解析解？
我们先看刚刚算出来的（固定倾角 $\theta_m$，在顺光 $\phi_i - \phi_o = 0^\circ$ 下的精确解析结果）：

$$ \mathcal{I}_{\text{顺光}}(\theta_m; \ \alpha, \beta) = \underbrace{\cos\theta_m}_{\text{第 1 项}} \ + \ \frac{1}{\pi} \mathbf{\tan\beta} \cdot \underbrace{\left[ \sin\theta_m \sin\phi_c + \frac{\sin^2\theta_m}{\cos\theta_m} \tan\alpha \left( \frac{\pi}{2} - \phi_c \right) \right]}_{\text{记为混合函数 } M(\theta_m, \alpha)} $$

现在，我们要对 $\theta_m \in [0, \frac{\pi}{2}]$ 做高斯分布加权积分

$\int_0^{\pi/2} 2\pi \cdot \mathcal{I}_{\text{顺光}}(\theta_m; \ \alpha, \beta) P(\theta_m; \sigma)\sin\theta_m d\theta_m$：

$$ \int_0^{\pi/2} 2\pi \cdot \Big[ \text{含 } \arccos(\cot\theta_m \cot\alpha) \text{ 和 } \sqrt{1 - \cot^2\theta_m \cot^2\alpha} \text{ 的分段式} \Big] \cdot e^{-\frac{\theta_m^2}{2\sigma^2}} \sin\theta_m \, d\theta_m $$

$\arccos(\cot\theta_m \cot\alpha) \cdot e^{-\theta_m^2 / 2\sigma^2}$ 的定积分是超越积分，根本不存在初等函数解析解！

##### 看【第 1 项 $\cos\theta_m$】
它里面本来就没有外部角度 $\alpha, \beta$！对 $\theta_m$ 高斯积分算出来的纯粗糙度函数，就是 Full 公式里的第一个系数 $C_1(\sigma)$：
$$ \int_0^{\pi/2} \Big[ 2\pi P(\theta_m; \sigma)\sin\theta_m \Big] \cdot \cos\theta_m \, d\theta_m \xrightarrow{\text{拟合为}} \mathbf{C_1(\sigma)} = 1 - 0.5 \frac{\sigma^2}{\sigma^2 + 0.33} $$
##### 看【第 2 项里的混合函数 $M(\theta_m, \alpha)$】—— 如何把 $\alpha$ 和 $\theta_m$ 拆开？**
   在第 2 项中，$\tan\beta$ 已经在外面了，但方括号里的 $M(\theta_m, \alpha)$ 同时含有 $\theta_m$ 和 $\alpha$（因为 $\cos\phi_c = \cot\theta_m \cot\alpha$）。

   我们在上一局第四步分析过 $M(\theta_m, \alpha)$ 随外部大角 $\alpha \in [0, \frac{\pi}{2}]$ 变化的形状：
   
   1. 当 $\alpha = 0^\circ$（垂直正视），此时山坡完全无遮挡：
	   $\tan 0^\circ = 0, \ \sin\phi_c = 0 \implies M(\theta_m, 0^\circ) = \mathbf{0}$；
	   
   2.  当 $\alpha ≈ 0^\circ$（小角度）时:
	   泰勒展开告诉我们 **$\tan\alpha \approx \sin\alpha \approx \alpha$****$$ M_{\text{无遮挡阶段}}(\theta_m, \alpha) = 0 + \frac{\sin^2\theta_m}{\cos\theta_m} \tan\alpha \left( \frac{\pi}{2} - 0 \right) = \mathbf{\left( \frac{\pi}{2} \frac{\sin^2\theta_m}{\cos\theta_m} \right) \cdot \tan\alpha} ≈  \mathbf{\left( \frac{\pi}{2} \frac{\sin^2\theta_m}{\cos\theta_m} \right) \cdot \sin\alpha}$$
	
   3. 当 $\alpha \to 90^\circ$（掠射角）时：$\tan\alpha (\frac{\pi}{2} - \phi_c) \to \cot\theta_m \implies M(\theta_m, 90^\circ) \to \mathbf{2\sin\theta_m}$
	   当 $\alpha \to \frac{\pi}{2}$ 时，$\cot\alpha \to 0$，所以：$$ \cos\phi_c = \cot\theta_m \cdot \cot\alpha \to 0 \implies \mathbf{\phi_c \to \frac{\pi}{2} \ (90^\circ)} $$因此，第一部分的极限直接就是：**$$ \lim_{\alpha \to \pi/2} \Big[ \sin\theta_m \sin\phi_c \Big] = \sin\theta_m \times 1 = \mathbf{\sin\theta_m} \quad \text{（拿到了第 1 个 }\sin\theta_m\text{！）} $$看第二部分：$$ \frac{\sin^2\theta_m}{\cos\theta_m} \cdot \underbrace{\tan\alpha}_{\to \infty} \cdot \underbrace{\left( \frac{\pi}{2} - \phi_c \right)}_{\to 0} $$当 $\alpha \to \frac{\pi}{2}$ 时，$\tan\alpha \to \infty$，而括号里的 $(\frac{\pi}{2} - \phi_c) \to 0$。这是一个标准的 **“$\infty \times 0$ 型未定式”**！怎么算这个极限？
	
	  只需做一个简单的换元：令那个趋近于 $0$ 的小角为 **$\epsilon = \frac{\pi}{2} - \phi_c$**（当 $\phi_c \to \frac{\pi}{2}$ 时，$\epsilon \to 0$）。 根据初中三角诱导公式：
	$$ \cos\phi_c = \cos\left( \frac{\pi}{2} - \epsilon \right) \equiv \mathbf{\sin\epsilon} $$
	又因为 $\cos\phi_c$ 的定义本来就是 $\cot\theta_m \cot\alpha = \frac{\cot\theta_m}{\tan\alpha}$，所以我们有：
	$$ \sin\epsilon = \frac{\cot\theta_m}{\tan\alpha} \implies \mathbf{\tan\alpha = \frac{\cot\theta_m}{\sin\epsilon}} $$
	 把 $\tan\alpha = \frac{\cot\theta_m}{\sin\epsilon}$ 和 $\left(\frac{\pi}{2} - \phi_c\right) = \epsilon$ 代入第二部分：
	$$ \lim_{\alpha \to \pi/2} \left[ \tan\alpha \left( \frac{\pi}{2} - \phi_c \right) \right] = \lim_{\epsilon \to 0} \left[ \frac{\cot\theta_m}{\sin\epsilon} \cdot \epsilon \right] = \cot\theta_m \cdot \underbrace{\lim_{\epsilon \to 0} \left( \frac{\epsilon}{\sin\epsilon} \right)}_{\text{高数第一重要极限 } = \mathbf{1}！} = \mathbf{\cot\theta_m} $$
	把算出来的极限 $\cot\theta_m = \frac{\cos\theta_m}{\sin\theta_m}$ 乘上前面的系数 $\frac{\sin^2\theta_m}{\cos\theta_m}$：
	**$$ \lim_{\alpha \to \pi/2} \Big[ \text{第二部分} \Big] = \frac{\sin^{\cancel{2}}\theta_m}{\cancel{\cos\theta_m}} \cdot \frac{\cancel{\cos\theta_m}}{\cancel{\sin\theta_m}} = \mathbf{\sin\theta_m} \quad \text{（拿到了第 2 个 }\sin\theta_m\text{！）} $$**
4. 两部分相加
	$$ \lim_{\alpha \to \pi/2} M(\theta_m, \alpha) = \underbrace{\sin\theta_m}_{\text{第一部分}} + \underbrace{\sin\theta_m}_{\text{第二部分}} = \mathbf{2 \sin\theta_m} ！！ $$
**现在请问：**
在一个 $[0^\circ, 90^\circ]$ 的区间上，什么基础三角函数同时满足：
*   在 $0^\circ$ 时等于 $0$；
*   在小角度时近似包含 $\sin\alpha$ 
*   在大角度接近 $90^\circ$ 时不再发散，而是平滑弯曲收敛到有限最大值 $\mathbf{2\sin\theta_m}$

在 $0^\circ \sim 90^\circ$ 之间，它随 $\alpha$ 的增长曲线就是一条标准的正弦波 **$\sin\alpha$**！

在数学上，要把一个起点为 $0$、终点为 $2\sin\theta_m$ 的曲线用 $\sin\alpha$（起点为 $0$、终点为 $1$）来拟合，必须先除以它的终点极值 $2\sin\theta_m$ 进行**归一化**：

$$ \frac{M(\theta_m, \alpha)}{2\sin\theta_m} \approx \sin\alpha \quad (\text{两边的值域都是从 } 0 \text{ 到 } 1！) $$

把分母的 $2\sin\theta_m$ 乘过去，正确的近似等式是：
**$$ M(\theta_m, \alpha) \approx \underbrace{(2 \sin\theta_m)}_{\text{终点极值幅度 (只含 }\theta_m\text{)}} \times \underbrace{\sin\alpha}_{\text{从 0 到 1 的增长形状 (只含 }\alpha\text{)}} $$**
现在，我们把 $M(\theta_m, \alpha) \approx (2\sin\theta_m) \cdot \sin\alpha$ 代回外层关于 $\theta_m$ 的高斯分布积分里！

原本我们要算的外层积分是：
$$ \int_0^{\pi/2} \Big[ 2\pi P(\theta_m; \sigma)\sin\theta_m \Big] \cdot \left( \frac{1}{\pi} M(\theta_m, \alpha) \cdot \tan\beta \right) d\theta_m $$

把 $M(\theta_m, \alpha) \approx 2\sin\theta_m \cdot \sin\alpha$ 代进去，把与 $\theta_m$ 无关的 **$\sin\alpha \cdot \tan\beta$** 提到积分号外面：

$$ = \underbrace{\left\{ \int_0^{\pi/2} \Big[ 2\pi P(\theta_m; \sigma)\sin\theta_m \Big] \cdot \left( \mathbf{\frac{2}{\pi} \sin\theta_m} \right) d\theta_m \right\}}_{\text{全部是关于 }\theta_m\text{ 的积分，这就是系数 } B(\sigma)！} \cdot \mathbf{\sin\alpha \cdot \tan\beta} $$


于是，Oren 和 Nayar 将混合函数 $M(\theta_m, \alpha)$ 进行了**变量分离（Separation of Variables）**——把 $\alpha$ 的变化提取为基函数 **$\sin\alpha$**，把剩下的 $\theta_m$ 积分归入粗糙度系数 **$C_2(\sigma)$**：
   $$ \int_0^{\pi/2} \frac{1}{\pi} M(\theta_m, \alpha) \cdot P(\theta_m; \sigma)\sin\theta_m \, d\theta_m \approx \mathbf{B(\sigma)} \cdot \mathbf{\sin\alpha} $$
$$C_2(\sigma) =  0.45 \frac{\sigma^2}{\sigma^2 + 0.09} \cdot \sin\alpha$$

##### 合并两项拟合结果
   把外面的 $\tan\beta$ 乘回来，顺光方向（$\phi_i - \phi_o = 0^\circ$）高斯积分后的结果就直接变成了：
   **$$ f_{\text{顺光}}(\alpha, \beta) = \frac{\rho}{\pi} \Big[ \mathbf{C_1(\sigma)} \cdot \mathbf{1} \ + \ \mathbf{C_2(\sigma)} \cdot \mathbf{\tan\beta} \Big] $$
   
### 五、补齐其他基函数

#### 1. 三个不同 $\Delta\phi$ 的积分结果
##### 锚点 1：同平面顺光（$\Delta\phi = 0^\circ$，此时 $\cos\Delta\phi = +1$）
这就是我们上一局亲手算出来的拟合结果：
**$$ f_{\text{顺光}}(\alpha, \beta) = \frac{\rho}{\pi} \left[ C_1(\sigma) \ + \ \left( 0.45 \frac{\sigma^2}{\sigma^2 + 0.09} \cdot \mathbf{\sin\alpha} \right) \cdot \tan\beta \right] $$**
##### 锚点 2：同平面正逆光（$\Delta\phi = 180^\circ$，此时 $\cos\Delta\phi = -1$）
当光源转到视线正对面（$\phi_i - \phi_o = \pi$）时，Oren 和 Nayar 对分段积分求出的**具体拟合公式**是：

**$$ f_{\text{逆光}}(\alpha, \beta) = \frac{\rho}{\pi} \left[ C_1(\sigma) \ \mathbf{-} \ \left( 0.45 \frac{\sigma^2}{\sigma^2 + 0.09} \cdot \left[ \mathbf{\sin\alpha - \left(\frac{2\beta}{\pi}\right)^3} \right] \right) \cdot \tan\beta \right] $$**
##### 锚点 3：正交垂直侧光（$\Delta\phi = 90^\circ$，此时 $\cos\Delta\phi = 0$）
当光源在正侧面（$\phi_i - \phi_o = \pm \frac{\pi}{2}$）时，Oren 和 Nayar 对分段积分求出的**具体拟合公式**是：

**$$ f_{\text{侧光}}(\alpha, \beta) = \frac{\rho}{\pi} \left[ C_1(\sigma) \ + \ \underbrace{\left( 0.125 \frac{\sigma^2}{\sigma^2 + 0.09} \cdot \left(\frac{4 \alpha \beta}{\pi^2}\right)^2 \right)}_{\text{定义为系数 } C_3(\sigma, \alpha, \beta)} \cdot \mathbf{\tan\left(\frac{\alpha + \beta}{2}\right)} \right] $$**

#### 2. 如何把这 3 个独立公式，拼成 1 个统一的 Full 公式？

现在，你手里有了三个特定方位角下的精确公式：
*   在 $\Delta\phi = 0^\circ$（$\cos\Delta\phi = +1$）时，要等于 **$f_{\text{顺光}}$**；
*   在 $\Delta\phi = 180^\circ$（$\cos\Delta\phi = -1$）时，要等于 **$f_{\text{逆光}}$**；
*   在 $\Delta\phi = 90^\circ$（$\cos\Delta\phi = 0$）时，要等于 **$f_{\text{侧光}}$**。

怎么用统一的数学式子，在任意方位角 $\Delta\phi \in [0^\circ, 180^\circ]$ 上把这三个公式无缝连接起来？

Oren 和 Nayar 使用了**分段系数 $C_2$** 配合 **两个绝对值互补的方位角权重 $\cos\Delta\phi$ 与 $\big(1 - |\cos\Delta\phi|\big)$**：

##### 第一步：把“顺光”和“逆光”合并进分段系数 $C_2$ 中
你看 $f_{\text{顺光}}$ 和 $f_{\text{逆光}}$，它们外面全都乘着 $\tan\beta$，唯一的区别是顺光时 $\cos\Delta\phi = +1$（乘的是 $\sin\alpha$），逆光时 $\cos\Delta\phi = -1$（乘的是 $-\big[\sin\alpha - (\frac{2\beta}{\pi})^3\big]$）。
只要我们在外面统一乘上一个 **$\cos(\phi_i - \phi_o)$**（顺光为 $+1$，逆光为 $-1$，侧光为 $0$），然后把系数 $C_2$ 写成分段函数：

$$ C_2(\sigma, \alpha, \beta, \Delta\phi) = \begin{cases} 0.45 \dfrac{\sigma^2}{\sigma^2 + 0.09} \cdot \sin\alpha, & \text{当 } \cos(\phi_i - \phi_o) \ge 0 \text{ (顺光半区)} \\[10pt] 0.45 \dfrac{\sigma^2}{\sigma^2 + 0.09} \cdot \left[ \sin\alpha - \left(\dfrac{2\beta}{\pi}\right)^3 \right], & \text{当 } \cos(\phi_i - \phi_o) < 0 \text{ (逆光半区)} \end{cases} $$

这样一来，**$\cos(\phi_i - \phi_o) \cdot C_2 \cdot \tan\beta$** 这一项：
*   在 $\Delta\phi = 0^\circ$ 时，自动等于 $+ C_{2,\text{顺光}} \tan\beta$；
*   在 $\Delta\phi = 180^\circ$ 时，自动等于 $- C_{2,\text{逆光}} \tan\beta$；
*   在 $\Delta\phi = 90^\circ$ 时，因为 $\cos 90^\circ = 0$，**自动等于 $0$（完全让路给侧光项！）**。

##### 第二步：用 $\big(1 - |\cos(\phi_i - \phi_o)|\big)$ 挂载侧光项 $C_3$
再看侧光项 $C_3 \cdot \tan\left(\frac{\alpha+\beta}{2}\right)$：
我们希望它在 $\Delta\phi = 90^\circ$ 时权重为 **$1$**，而在 $\Delta\phi = 0^\circ$ 和 $180^\circ$ 时权重为 **$0$**（绝不干扰顺光和逆光）。
什么函数在 $90^\circ$ 时为 $1$，在 $0^\circ$ 和 $180^\circ$ 时为 $0$？
正是 **$1 - |\cos(\phi_i - \phi_o)|$**！

##### 第三步：三项合体，得到 1994 原论文的【Full 单次散射公式 $f_1$】！
把 $C_1$、带权重的 $C_2$、带权重的 $C_3$ 加在一起，就是原论文公式 (19) 的完整单次散射项：

**$$ f_1(\theta_i, \theta_o, \Delta\phi) = \frac{\rho}{\pi} \left[ C_1 + \underbrace{\cos(\phi_i - \phi_o)}_{\text{0°为1, 180°为-1, 90°为0}} \cdot C_2 \cdot \tan\beta + \underbrace{\big( 1 - |\cos(\phi_i - \phi_o)| \big)}_{\text{90°为1, 0°和180°为0}} \cdot C_3 \cdot \tan\left(\frac{\alpha + \beta}{2}\right) \right] $$**
##### 补齐最后一块拼图：Full 公式里的“二次反弹补偿项 $f_2$”

在 1994 年原论文中，完整的 Full 模型除了上面的单次散射 $f_1$ 之外，Oren 和 Nayar 还顺手算了一下**“光线打在 V 型槽的左坡上，弹到右坡上，再从右坡弹进眼睛（2-bounce Interreflection）”**的二次散射项 $f_2$（原论文公式 20）：

**$$ f_2(\theta_i, \theta_o, \Delta\phi) = 0.17 \frac{\rho^2}{\pi} \frac{\sigma^2}{\sigma^2 + 0.13} \left[ 1 - \cos(\phi_i - \phi_o) \left(\frac{2\beta}{\pi}\right)^2 \right] $$**

完整的 **Full Oren-Nayar BRDF** 就是这两项之和：
$$ f_{\text{Full\_Oren-Nayar}} = f_1 + f_2 $$

---


## 第三步：经典 Oren-Nayar 公式

#### 1994 Full Oren-Nayar 完整版公式

$$ f_{\text{Full}} = \frac{\rho}{\pi} \Big[ C_1 + \cos(\phi_i - \phi_o) \cdot C_2 \cdot \tan\beta + \big(1 - |\cos(\phi_i - \phi_o)|\big) \cdot C_3 \cdot \tan\left(\frac{\alpha + \beta}{2}\right) \Big] \ + \ f_2 $$

其中：
*   $C_1 = 1 - 0.5 \dfrac{\sigma^2}{\sigma^2 + 0.33}$
*   $C_2 = \begin{cases} 0.45 \dfrac{\sigma^2}{\sigma^2 + 0.09} \cdot \sin\alpha, & \text{当 } \cos(\phi_i - \phi_o) \ge 0 \text{ (顺光半区)} \\[8pt] 0.45 \dfrac{\sigma^2}{\sigma^2 + 0.09} \cdot \left[ \sin\alpha - \left(\dfrac{2\beta}{\pi}\right)^3 \right], & \text{当 } \cos(\phi_i - \phi_o) < 0 \text{ (逆光半区)} \end{cases}$
*   $C_3 = 0.125 \left(\dfrac{\sigma^2}{\sigma^2 + 0.09}\right) \left(\dfrac{4\alpha\beta}{\pi^2}\right)^2$
*   $f_2 = 0.17 \dfrac{\rho^2}{\pi} \dfrac{\sigma^2}{\sigma^2 + 0.13} \left[ 1 - \cos(\phi_i - \phi_o)\left(\dfrac{2\beta}{\pi}\right)^2 \right]$

#### 第一刀：砍掉末尾的二次反弹项 $f_2$（只保留单次散射 $f_1$）

*   **怎么砍的**：直接令 **$f_2 = 0$**。
*   **为什么要砍它？**
    1. 1994 年的硬件算力极其有限，Oren 和 Nayar 希望先提供一个只描述微表面**单次散射（Single Scattering）**的基础模型（就像高光 GGX 早期也只算单次散射一样）。
    2. $f_2$ 里面有一个 $\left(\frac{2\beta}{\pi}\right)^2$，要想算它，必须调用昂贵的反余弦函数 $\beta = \arccos\big(\max(\mathbf{n}\cdot\boldsymbol{\omega}_i, \mathbf{n}\cdot\boldsymbol{\omega}_o)\big)$ 算出弧度值。
    *(注：砍掉 $f_2$ 的代价是粗糙表面在白炉测试中会损失约 $10\% \sim 20\%$ 的能量而偏暗——这个被砍掉的 $f_2$，在整整 30 年后的 **2024 年 OpenPBR EON 模型**中，被用更完美的解析多重散射补偿重新请了回来！)*

#### 第二刀：砍掉第 3 项侧光修正项 $C_3$！

*   **怎么砍的**：直接令 **$C_3 = 0$**。
*   **为什么要砍它？（算一笔数字账你就明白了）**
    请看第 3 项：
    $$ \big(1 - |\cos(\phi_i - \phi_o)|\big) \cdot \mathbf{0.125} \left(\frac{\sigma^2}{\sigma^2 + 0.09}\right) \cdot \mathbf{\left(\frac{4\alpha\beta}{\pi^2}\right)^2} \cdot \tan\left(\frac{\alpha + \beta}{2}\right) $$
    1. **数值小到可以忽略不计**：
       首先，它的前导系数是 `0.125`，只有主项 $C_2$（`0.45`）的四分之一；
       其次，后面乘了一个极度严苛的压制因子 $\left(\frac{4\alpha\beta}{\pi^2}\right)^2 = \left(\frac{\alpha}{\pi/2} \cdot \frac{\beta}{\pi/2}\right)^2$！
       比如在日常角度 $\alpha = 45^\circ, \ \beta = 45^\circ$（即 $\frac{\pi}{4}$）时：
       $$ \left(\frac{4\alpha\beta}{\pi^2}\right)^2 = \left(\frac{1}{2} \times \frac{1}{2}\right)^2 = \frac{1}{16} = \mathbf{0.0625} $$
       再乘上前导系数 `0.125`，总共只有 **$0.125 \times 0.0625 = 0.0078$（千分之七）**！肉眼在屏幕上根本看不出这千分之七的差别！
    2. **消灭反三角函数与半角正切**：
       一旦留着 $C_3$，你就必须用 `acos` 算出弧度角 $\alpha$ 和 $\beta$，还要算 $\tan\left(\frac{\alpha+\beta}{2}\right)$。把 $C_3$ 一刀砍掉，这两大算力毒瘤瞬间消失！

#### 第三刀（最精妙的一刀）：砍掉 $C_2$ 的逆光分支，合并成 $\max\big(0, \cos(\phi_i - \phi_o)\big)$！

砍完前两刀后，公式只剩下了前两项：
$$ f = \frac{\rho}{\pi} \Big[ C_1 + \cos(\phi_i - \phi_o) \cdot C_2 \cdot \tan\beta \Big] $$

现在，只剩下最后一个障碍——分段系数 **$C_2$ 的逆光分支（当 $\cos(\phi_i - \phi_o) < 0$ 时）**：
$$ C_{2,\text{逆光}} \cdot \tan\beta = 0.45 \frac{\sigma^2}{\sigma^2 + 0.09} \cdot \underbrace{\left[ \sin\alpha - \left(\frac{2\beta}{\pi}\right)^3 \right] \cdot \tan\beta}_{\text{逆光时的角度项}} $$

**请问：在逆光半区（$\cos(\phi_i - \phi_o) < 0$），右边这陀 $\left[ \sin\alpha - \left(\frac{2\beta}{\pi}\right)^3 \right] \cdot \tan\beta$ 到底有多大？**
我们看两个极端情况：
1. **当 $\beta$ 很小（光线或视线靠近表面中心）时**：
   因为 $\beta \to 0$，外面的 $\tan\beta \to 0$，所以整项乘起来本来就接近 **$0$**！
2. **当 $\beta$ 很大（光线和视线同时贴近 $90^\circ$ 边缘，即 $\alpha \ge \beta \to \frac{\pi}{2}$）时**：
   虽然外面的 $\tan\beta$ 变大了，但是看方括号内部：
   * $\sin\alpha \to \sin\left(\frac{\pi}{2}\right) = \mathbf{1}$
   * $\left(\frac{2\beta}{\pi}\right)^3 \to \left(\frac{2 \cdot \pi/2}{\pi}\right)^3 = 1^3 = \mathbf{1}$
   * 方括号内部自己相减：$1 - 1 = \mathbf{0}$！
   也就是说，**在逆光边缘，$\sin\alpha$ 和 $\left(\frac{2\beta}{\pi}\right)^3$ 本来就是互相抵消归零的**！

**Oren 和 Nayar 一拍大腿：**
既然在逆光半区（$\cos(\phi_i - \phi_o) < 0$），这一项在小角度时因为 $\tan\beta \approx 0$ 而接近 $0$，在大角度时又因为 $\sin\alpha - (\frac{2\beta}{\pi})^3 \approx 0$ 而自己抵消接近 $0$；更何况为了算那个讨厌的 $\left(\frac{2\beta}{\pi}\right)^3$ 还得调用反余弦 `acos` 求 $\beta$ 的弧度——
**那我们干脆规定：当 $\cos(\phi_i - \phi_o) < 0$（逆光）时，直接令第 2 项整体等于 $0$！**

怎么在数学公式里实现“当 $\cos(\phi_i - \phi_o) \ge 0$ 时保留顺光支，当 $\cos(\phi_i - \phi_o) < 0$ 时直接等于 $0$”？
**只需要给 $\cos(\phi_i - \phi_o)$ 套上一个 $\max\big(0, \ \cos(\phi_i - \phi_o)\big)$！**

一旦套上 $\max(0, \dots)$，逆光分支永远乘以 $0$ 被干掉了，$C_2$ 就再也不需要分段了，永远只剩下顺光那一支：
$$ C_2 = \underbrace{\left( 0.45 \frac{\sigma^2}{\sigma^2 + 0.09} \right)}_{\text{单独起名叫 } B(\sigma)} \cdot \sin\alpha $$

#### 第四步：重命名 $C_1 \to A$，经典 Oren-Nayar 公式正式诞生！

经过上面干脆利落的三刀：
1. 把 $C_1(\sigma)$ 改名为 **$A$**：
   $$ A = 1 - 0.5 \frac{\sigma^2}{\sigma^2 + 0.33} $$
2. 把 $C_2$ 里的粗糙度分式提取出来改名为 **$B$**：
   $$ B = 0.45 \frac{\sigma^2}{\sigma^2 + 0.09} $$
3. 把留下的 $\max\big(0, \cos(\phi_i - \phi_o)\big)$、$\sin\alpha$ 和 $\tan\beta$ 乘在一起：

**$$ f_{\text{Oren-Nayar}} = \frac{\rho}{\pi} \Big( A + B \cdot \max\big(0, \ \cos(\phi_i - \phi_o)\big) \cdot \sin\alpha \cdot \tan\beta \Big) $$**