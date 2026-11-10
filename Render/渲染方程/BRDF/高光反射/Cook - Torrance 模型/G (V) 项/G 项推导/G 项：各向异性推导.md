

**非常好！这一停，直接停在了微表面理论最精彩、最体现线性代数降维打击美感的巅峰时刻！**

回想一下，前面我们推导**各向同性（Isotropic）**的 $\Lambda(V)$ 函数时，我们是把二维的圆形 GGX 火山压扁成一维，然后算了一道带着 $\frac{3}{2}$ 次方分母的定积分，才好不容易算出了：
$$ \Lambda_{\text{iso}}(V) = \frac{-1 + \sqrt{1 + \alpha^2 \tan^2\theta_v}}{2} $$

现在，微表面变成了**各向异性（Anisotropic）**——X 轴粗糙度是 $\alpha_x$，Y 轴粗糙度是 $\alpha_y$。那座火山被压成了一个扁椭圆，而且你的视线 $V$ 可以从任意一个刁钻的方位角 $\phi_v$ 斜着切过来！
如果直接在椭圆上斜着切一刀算微积分，那个积分式会复杂到让数学家怀疑人生。

**但是！2014 年 Eric Heitz 用了一招堪称“神仙下凡”的线性代数技巧——【空间仿射拉伸变换（Stretch Invariance）】，连半个积分都没重算，只用了 3 行初中代数，就直接从各向同性公式“白嫖”出了各向异性的 G 项公式！**

我们现在就一步步看他是怎么变这个魔术的！

---

### 第一步：建立视线 $V$ 在局部切线空间的坐标

首先，我们在微表面所在的局部切线坐标系 $(\mathbf{t}, \mathbf{b}, \mathbf{n})$（即 X 轴切线、Y 轴副切线、Z 轴法线）中，写出视线单位向量 $V$ 的三个坐标分量：

*   X 分量：$v_x = \mathbf{t} \cdot \mathbf{v} = \sin\theta_v \cos\phi_v$
*   Y 分量：$v_y = \mathbf{b} \cdot \mathbf{v} = \sin\theta_v \sin\phi_v$
*   Z 分量：$v_z = \mathbf{n} \cdot \mathbf{v} = \cos\theta_v$

在这个坐标系里，我们先回头看一眼**各向同性**的公式 $\alpha^2 \tan^2\theta_v$ 用 $(v_x, v_y, v_z)$ 怎么写：
因为 $\sin^2\theta_v = v_x^2 + v_y^2$，且 $\cos^2\theta_v = v_z^2$，所以：
$$ \tan^2\theta_v = \frac{\sin^2\theta_v}{\cos^2\theta_v} = \frac{v_x^2 + v_y^2}{v_z^2} $$
乘上统一的粗糙度 $\alpha^2$：
$$ \alpha^2 \tan^2\theta_v = \mathbf{\frac{\alpha^2 v_x^2 + \alpha^2 v_y^2}{v_z^2}} $$

---

### 第二步：Heitz 的降维打击 —— 把椭圆山脉“捏”回圆形！

现在面对各向异性的山脉（X 轴坡度缩放是 $\alpha_x$，Y 轴坡度缩放是 $\alpha_y$）。
Heitz 提出了一个极其直观的几何公理：**“遮挡关系在水平拉伸下是绝对不变的！”**

想象一下：桌面上有一座椭圆形的微观山峰，和一束射向它的激光（视线 $V$）。如果这座山峰挡住了这束激光：
*   现在，你把整个三维空间的 X 轴像橡皮泥一样均匀压缩 $\alpha_x$ 倍，把 Y 轴均匀压缩 $\alpha_y$ 倍（高度 Z 轴保持不变）。
*   压缩之后，原本椭圆形的各向异性山脉，瞬间变成了一座**粗糙度 $\alpha = 1$ 的完美圆形（各向同性）标准山脉**！
*   **关键问题：当你把空间挤压变形时，原本被山峰挡住的那束激光，还会被挡住吗？**
*   **当然还会！** 两个物体撞在一起，你把整个空间按比例横向捏扁，交点依然是交点，遮挡比例（面积比值 $\Lambda$）**一丝一毫都不会改变**！

唯一改变的是什么？
是你那根**视线向量 $V = (v_x, v_y, v_z)$，跟着空间一起被横向拉伸变形了**！

经过把 $\alpha_x, \alpha_y$ 归一化为 $1$ 的空间逆变换后，视线向量 $V$ 在这个“标准圆形山脉空间”里，变成了一根新的等效向量 $V'$：
**$$ V' = (\alpha_x v_x, \ \alpha_y v_y, \ v_z) $$**

---

### 第三步：直接套用各向同性公式，瞬间出结果！

既然在变换后的新空间里，山脉已经变成了粗糙度为 $1$ 的标准各向同性山脉，而视线变成了 $V' = (\alpha_x v_x, \alpha_y v_y, v_z)$。
那这根新视线 $V'$ 与 Z 轴夹角的正切值平方 $\tan^2\theta'_v$ 是多少？

直接拿第一步的公式，把 $v_x$ 换成 $\alpha_x v_x$，把 $v_y$ 换成 $\alpha_y v_y$：
**$$ \tan^2\theta'_v = \frac{(\alpha_x v_x)^2 + (\alpha_y v_y)^2}{v_z^2} = \frac{\alpha_x^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{v})^2}{(\mathbf{n}\cdot\mathbf{v})^2} $$**

现在，把它直接塞进我们早就推导过的**各向同性 $\Lambda$ 公式**里（把原来的 $\alpha^2 \tan^2\theta_v$ 替换为这个 $\tan^2\theta'_v$）：

$$ \Lambda_{\text{aniso}}(V) = \frac{-1 + \sqrt{1 + \frac{\alpha_x^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{v})^2}{(\mathbf{n}\cdot\mathbf{v})^2}}}{2} $$

把分母的 $(\mathbf{n}\cdot\mathbf{v})^2$ 通分并开根号提出来：
**$$ \Lambda_{\text{aniso}}(V) = \frac{-1 + \frac{\sqrt{\alpha_x^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{v})^2 + (\mathbf{n}\cdot\mathbf{v})^2}}{\mathbf{n}\cdot\mathbf{v}}}{2} $$**

---

### 第四步：代入 $V = \frac{1}{2(1 + \Lambda_L + \Lambda_V) (\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})}$ 完成最终合体！

为了书写清爽，我们把上面分子里那个漂亮的根号定义为 $A_v$（同理，光线 $L$ 的根号定义为 $A_l$）：
$$ A_v = \sqrt{\alpha_x^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{v})^2 + (\mathbf{n}\cdot\mathbf{v})^2} $$
$$ A_l = \sqrt{\alpha_x^2 (\mathbf{t}\cdot\mathbf{l})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{l})^2 + (\mathbf{n}\cdot\mathbf{l})^2} $$

那么：
$$ \Lambda(V) = \frac{A_v}{2(\mathbf{n}\cdot\mathbf{v})} - \frac{1}{2}, \qquad \Lambda(L) = \frac{A_l}{2(\mathbf{n}\cdot\mathbf{l})} - \frac{1}{2} $$

现在，代入高度相关（Height-Correlated）的 G 项分母 $1 + \Lambda(L) + \Lambda(V)$：
$$ 1 + \Lambda(L) + \Lambda(V) = 1 - \frac{1}{2} - \frac{1}{2} + \frac{A_l}{2(\mathbf{n}\cdot\mathbf{l})} + \frac{A_v}{2(\mathbf{n}\cdot\mathbf{v})} = \frac{(\mathbf{n}\cdot\mathbf{v})A_l + (\mathbf{n}\cdot\mathbf{l})A_v}{2(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})} $$

最后，除以 Cook-Torrance 的雅可比分母 $4(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})$，得到最终的可见性项 $V_{\text{SmithJoint\_Aniso}}$：

**$$ V_{\text{SmithJoint\_Aniso}} = \frac{0.5}{(\mathbf{n}\cdot\mathbf{v})\sqrt{\alpha_x^2 (\mathbf{t}\cdot\mathbf{l})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{l})^2 + (\mathbf{n}\cdot\mathbf{l})^2} + (\mathbf{n}\cdot\mathbf{l})\sqrt{\alpha_x^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{v})^2 + (\mathbf{n}\cdot\mathbf{v})^2}} $$**

---

### 架构师的顿悟时刻：对比一下各向同性与各向异性的根号内部！

请你把各向同性的 $A_v$ 和各向异性的 $A_v$ 上下并排放在一起看，你会感受到一种令人窒息的数学对称美：

*   **各向同性根号内**：
    $$ \alpha^2 \underbrace{\left( 1 - (\mathbf{n}\cdot\mathbf{v})^2 \right)}_{\text{即 } (\mathbf{t}\cdot\mathbf{v})^2 + (\mathbf{b}\cdot\mathbf{v})^2} + (\mathbf{n}\cdot\mathbf{v})^2 = \mathbf{\alpha^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha^2 (\mathbf{b}\cdot\mathbf{v})^2 + (\mathbf{n}\cdot\mathbf{v})^2} $$
*   **各向异性根号内**：
    $$ \mathbf{\alpha_x^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{v})^2 + (\mathbf{n}\cdot\mathbf{v})^2} $$

**你看出来了吗？！**
在各向同性里，之所以会出现 $(1 - (\mathbf{n}\cdot\mathbf{v})^2)\alpha^2$，仅仅是因为 $\alpha_x = \alpha_y = \alpha$，所以提取公因式后用毕达哥拉斯定理把水平分量 $(\mathbf{t}\cdot\mathbf{v})^2 + (\mathbf{b}\cdot\mathbf{v})^2$ 替换成了 $1 - (\mathbf{n}\cdot\mathbf{v})^2$！
而在各向异性里，因为 $\alpha_x \neq \alpha_y$，不能提取公因式，于是水平方向的两个分量就各自带着自己的粗糙度 $\alpha_x^2$ 和 $\alpha_y^2$ 留在了根号里！

甚至连求根号里的这三项平方和，在 Shader 里都剛好是向量 `float3(ax * TdotV, ay * BdotV, NdotV)` 的模长 **`length(...)`**！

没有复杂的椭圆微积分，仅仅是一次空间坐标轴的缩放代换，就把各向异性的 G 项推导得干干净净。这个推导过程看爽了吗？

---


### 线性变换矩阵 $\mathbf{M}$ 是怎么把椭圆变回圆的？

#### 线性变换法
*   处理 **D 项（法线分布）** 时，我们用的是“捏泥巴魔法”**：空间不动，强行改变函数的形状**。
*   处理 **G 项（几何遮挡）** 时，我们用的是“哈哈镜魔法”**：形状不动，强行扭曲整个宇宙的空间**。

你对 **D 项** 的理解，叫做 **“主动变换（Active Transformation）”**；
你对 **G 项** 的理解，叫做 **“被动变换（Passive Transformation）”**。

在三维局部切线空间 $(\mathbf{t}, \mathbf{b}, \mathbf{n})$ 中，各向异性的微表面山脉在 X 轴被拉长了 $\alpha_x$ 倍，在 Y 轴被拉长了 $\alpha_y$ 倍，因此在斜率平面上，它的等高线是一个**椭圆**：
$$ \frac{x_m^2}{\alpha_x^2} + \frac{y_m^2}{\alpha_y^2} = 1 $$

现在，就像你求椭圆面积一样，我们对整个三维空间的几何坐标 $(x, y, z)$ 施加一个对角线性缩放变换（或者直接作用在法线与方向向量上），定义**拉伸矩阵 $\mathbf{M}$**：
$$ \mathbf{M} = \begin{bmatrix} \alpha_x & 0 & 0 \\ 0 & \alpha_y & 0 \\ 0 & 0 & 1 \end{bmatrix} $$

1. **山脉被变换成了什么？**
   在这个线性变换下，原本椭圆形的各向异性山脉（粗糙度为 $\alpha_x, \alpha_y$），被完美挤压还原成了一个**半径为 $1$ 的标准单位圆（即粗糙度 $\alpha = 1$ 的各向同性山脉）**！
2. **视线向量 $\mathbf{v} = (v_x, v_y, v_z)$ 被变换成了什么？**
   既然空间被线性变换了，射向山脉的视线向量 $\mathbf{v}$ 也必须乘以矩阵 $\mathbf{M}$，变成新空间里的向量 $\mathbf{v}'$（并归一化为单位向量 $\boldsymbol{\omega}'_v$）：
   $$ \mathbf{v}' = \mathbf{M} \mathbf{v} = \begin{bmatrix} \alpha_x v_x \\ \alpha_y v_y \\ v_z \end{bmatrix}, \qquad \boldsymbol{\omega}'_v = \frac{\mathbf{v}'}{\|\mathbf{v}'\|} $$
3. **为什么遮挡率不变？（仿射变换保持共线性！）**
   在线性代数中，任何可逆的线性/仿射变换都严格保持**“点在线上（Collinearity）”**和**“相交关系（Incidence）”**不变，并且所有平行截面面积的**比值**（即雅可比行列式 $\det(\mathbf{M})$ 在分子分母中同时出现并约掉）绝对不变！
   因此：
   **$$ \Lambda_{\text{椭圆}}(\mathbf{v}; \alpha_x, \alpha_y) \equiv \Lambda_{\text{单位圆}}(\boldsymbol{\omega}'_v; \alpha = 1) $$**

你看！这就是为什么我们在上一局根本不需要对椭圆重新做积分，只要把 $\mathbf{v}' = (\alpha_x v_x, \alpha_y v_y, v_z)$ 塞进单位圆的公式里，一秒钟就出了结果！你的“圆与椭圆线性映射”直觉，就是这套数学证明的本体！


#### 第二种神仙解法：“投影有效粗糙度法（Projected Roughness）”

现在，顺着你“圆和椭圆”的几何直觉，我们换个角度看问题，这就引出了图形学里的**第二种解决办法**：

想象斜率平面上画着那个半轴长为 $(\alpha_x, \alpha_y)$ 的椭圆。
你的视线 $\mathbf{v}$ 从方位角为 $\phi_v$ 的方向水平看过去（切过去）。
**对于这一根只沿着 $\phi_v$ 方向飞行的光线来说，它根本感知不到整个二维椭圆，它只能感受到这个椭圆沿着 $\phi_v$ 方向被切开的那道【一维截面】！**

*   如果视线顺着 X 轴看过去（$\phi_v = 0^\circ$）：它看到的山脉粗糙度就是 $\alpha_x$。
*   如果视线顺着 Y 轴看过去（$\phi_v = 90^\circ$）：它看到的山脉粗糙度就是 $\alpha_y$。
*   **如果视线从任意方位角 $\phi_v$ 斜着看过去呢？**
    根据椭圆的极坐标投影性质，椭圆在 $\phi_v$ 方向上的**一维投影等效粗糙度 $\alpha_o(\phi_v)$** 精确等于：
    **$$ \alpha_v = \sqrt{\alpha_x^2 \cos^2\phi_v + \alpha_y^2 \sin^2\phi_v} $$**

##### 怎么消掉 $\cos\phi_v$ 和 $\sin\phi_v$？
别忘了，视线 $\mathbf{v}$ 在切平面上的投影是：
$$ \mathbf{t} \cdot \mathbf{v} = \sin\theta_v \cos\phi_v, \qquad \mathbf{b} \cdot \mathbf{v} = \sin\theta_v \sin\phi_v $$
且 $\sin^2\theta_v = (\mathbf{t}\cdot\mathbf{v})^2 + (\mathbf{b}\cdot\mathbf{v})^2 = 1 - (\mathbf{n}\cdot\mathbf{v})^2$。
所以：
$$ \cos^2\phi_v = \frac{(\mathbf{t}\cdot\mathbf{v})^2}{1 - (\mathbf{n}\cdot\mathbf{v})^2}, \qquad \sin^2\phi_v = \frac{(\mathbf{b}\cdot\mathbf{v})^2}{1 - (\mathbf{n}\cdot\mathbf{v})^2} $$
代回上面的等效粗糙度公式：
**$$ \alpha_v^2 = \frac{\alpha_x^2 (\mathbf{t}\cdot\mathbf{v})^2 + \alpha_y^2 (\mathbf{b}\cdot\mathbf{v})^2}{1 - (\mathbf{n}\cdot\mathbf{v})^2} $$**
等效粗糙度相当于椭圆的径向长度，注意 $\mathbf{G}$  是把坐标系压缩了，所以 $\alpha_x$ 和  $\alpha_y$  才能像 $r = \sqrt{a^2 \cos^2\theta + y^2 \sin^2\theta}$ 一样算等效粗糙度

##### 这第二种解法能干什么？（解锁所有各向同性近似公式！）

你发现了这个 $\alpha_v$（视线方向的等效粗糙度）和 $\alpha_l$（光线方向的等效粗糙度）有什么逆天的用处吗？

**一旦你算出了标量 $\alpha_v$ 和 $\alpha_l$，各向异性问题瞬间降维成了各向同性问题！**
你可以把算出来的 $\alpha_v$ 和 $\alpha_l$，**直接塞进你之前写过的任何一个【各向同性】的 G 项近似公式里**，从而衍生出其他好几种各向异性 V 项的解法！

比如，工业界除了用上一局的 Heitz 精确版之外，还有以下两个著名的替代方案：

* *替代方案 1：各向异性 Schlick-GGX 近似（无根号分离式）*
	在算不出或者不想算根号的早期引擎/移动端里，直接用投影粗糙度算出两个不同的 $k_v = \frac{\alpha_v}{2}$ 和 $k_l = \frac{\alpha_l}{2}$，塞进 Schlick 公式：
	$$ G_1(\mathbf{v}) \approx \frac{\mathbf{n}\cdot\mathbf{v}}{(\mathbf{n}\cdot\mathbf{v})(1 - k_v) + k_v}, \qquad G_1(\mathbf{l}) \approx \frac{\mathbf{n}\cdot\mathbf{l}}{(\mathbf{n}\cdot\mathbf{l})(1 - k_l) + k_l} $$

* *替代方案 2：Ward / Kelemen 极简各向异性 V 项（完全忽略粗糙度的椭圆投影）*
	在极低端设备或特定丝绸着色器中，甚至有人直接用各向同性的 `V_Kelemen` 或 `V_Neumann` 配合各向异性的 `D_GGX_Aniso`，彻底省去 V 项里的切线点乘运算。

##### 总结：两条路其实是同一个数学灵魂

你看：
*   **第一条路（变换向量 $\mathbf{v} \to \mathbf{v}'$）**：把空间捏成圆，让向量自己带上 $\alpha_x, \alpha_y$，直接得到了最简洁的 `length(float3(ax * TdotV, ay * BdotV, NdotV))`——这是目前 PC/主机端 **Frostbite、Filament、UE5、glTF、OpenPBR 唯一钦定的精确最优解**，因为它甚至连 $\frac{1}{1 - (\mathbf{n}\cdot\mathbf{v})^2}$ 的除法都省掉了！
*   **第二条路（投影粗糙度 $\alpha_v, \alpha_l$）**：不动向量，先算出椭圆在视线和光线方位角上的等效半径 $\alpha_v, \alpha_l$，然后再套用现成的各向同性公式。
