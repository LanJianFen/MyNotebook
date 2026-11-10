

太棒了！你的要求极其合理。上一局我为了讲清“降维”的物理概念，把数学符号稍微口语化了。但在真正的图形学论文推导中，**严谨的符号定义是一切的基石**。

今天我们就挂上最高级别的数学挡位。严格规范符号，把那个阴魂不散的 **$\cos^4\theta_h$** 从雅可比矩阵里彻彻底底地“扒”出来！

---

### 第一步：明确符号，$\theta_h$ 到底是谁和谁的夹角？

在微表面理论中，我们需要两根法线：
1.  **宏观法线 $n$**：也就是整个大表面的朝向。在切线空间（Tangent Space）里，它永远笔直朝上，即 $n = (0, 0, 1)$。
2.  **微观法线 $h$**：也就是那座微小山峰的朝向（半角向量）。

**定义：$\theta_h$ 就是宏观法线 $n$ 和微观法线 $h$ 之间的夹角！**
由于 $n = (0,0,1)$，所以微观法线 $h$ 与 $n$ 的点乘，刚好就等于 $\cos\theta_h$：
$$ n \cdot h = \cos\theta_h $$

我们再看一眼物理面积守恒的积分公式：
$$ \int_{\Omega} D(h) \cos\theta_h d\omega_h = 1 $$
**这里的 $\cos\theta_h$ 是物理上的“投影系数”**。
*物理意义*：$D(h) d\omega_h$ 算出来的是“这批朝向 $h$ 的微小斜坡的**真实面积**”。把它乘以 $\cos\theta_h$，就是把这些倾斜的面积，**垂直投影**到平坦的宏观表面上。所有这些投影面积加起来，必须等于宏观表面的总面积（1）。

---

### 第二步：写出斜率空间和球面坐标的关系式

根据我们在上一局得出的共识，微法线 $h$ 的方向，既可以用**球面极坐标 $(\theta_h, \phi_h)$** 表示，也可以用**二维斜率空间 $(x_m, y_m)$** 表示。

它们的精确转换关系是：
$$ x_m = \tan\theta_h \cos\phi_h $$
$$ y_m = \tan\theta_h \sin\phi_h $$

---

### 第三步：硬核推导！算出雅可比行列式

既然要把斜率平面的面积微元 $(dx_m \, dy_m)$ 转换到球面坐标的角度微元 $(d\theta_h \, d\phi_h)$，我们就必须求这两个方程组成的**雅可比矩阵的行列式 $J$**。

雅可比行列式的定义是：
$$ J = \left| \begin{matrix} \frac{\partial x_m}{\partial \theta_h} & \frac{\partial x_m}{\partial \phi_h} \\ \frac{\partial y_m}{\partial \theta_h} & \frac{\partial y_m}{\partial \phi_h} \end{matrix} \right| $$

我们分别来求这 4 个偏导数（记住微积分基础：$\tan x$ 的导数是 $\frac{1}{\cos^2 x}$）：

1.  $\frac{\partial x_m}{\partial \theta_h} = \frac{1}{\cos^2\theta_h} \cos\phi_h$
2.  $\frac{\partial x_m}{\partial \phi_h} = -\tan\theta_h \sin\phi_h$
3.  $\frac{\partial y_m}{\partial \theta_h} = \frac{1}{\cos^2\theta_h} \sin\phi_h$
4.  $\frac{\partial y_m}{\partial \phi_h} = \tan\theta_h \cos\phi_h$

现在，计算行列式 $J = (1 \times 4) - (2 \times 3)$：
$$ J = \left( \frac{\cos\phi_h}{\cos^2\theta_h} \right) \left( \tan\theta_h \cos\phi_h \right) - \left( -\tan\theta_h \sin\phi_h \right) \left( \frac{\sin\phi_h}{\cos^2\theta_h} \right) $$

把 $\frac{\tan\theta_h}{\cos^2\theta_h}$ 提取出来作为公因式：
$$ J = \frac{\tan\theta_h}{\cos^2\theta_h} \left( \cos^2\phi_h + \sin^2\phi_h \right) $$

因为 $\cos^2\phi_h + \sin^2\phi_h = 1$，所以：
$$ J = \frac{\tan\theta_h}{\cos^2\theta_h} = \frac{\frac{\sin\theta_h}{\cos\theta_h}}{\cos^2\theta_h} = \mathbf{\frac{\sin\theta_h}{\cos^3\theta_h}} $$

根据多重积分的变量代换法则，微小面积的关系是：
$$ dx_m \, dy_m = J \, d\theta_h \, d\phi_h = \frac{\sin\theta_h}{\cos^3\theta_h} d\theta_h \, d\phi_h $$

**关键来了！**
物理上的立体角微元 $d\omega_h$，其标准定义正是 $d\omega_h = \sin\theta_h \, d\theta_h \, d\phi_h$。
我们把上式中的 $\sin\theta_h \, d\theta_h \, d\phi_h$ 打包替换为 $d\omega_h$，就得到了空间转换的终极公式：
$$ \mathbf{dx_m \, dy_m = \frac{1}{\cos^3\theta_h} d\omega_h} $$

*(注意：到这一步，仅仅是纯数学的空间变换。分母上只有 3 次方！那第 4 次方去哪了？往下看！)*

---

### 第四步：物理与数学汇合，见证 $\cos^4\theta_h$ 的诞生！

在斜率空间里，我们定义了二维概率密度函数 $P^2(x_m, y_m)$，它代表的是在斜率空间里的“投影面积占比”。
而在三维球面上，我们有法线分布函数 $D(h)$。

**这两者在物理上代表的是同一堆微小山坡的面积！**
所以，它们对应的**微分投影面积必须绝对相等**：
$$ P^2(x_m, y_m) \, dx_m \, dy_m = D(h) \cos\theta_h d\omega_h $$
*   **左边**：是在斜率平面上直接计算出的一小块“投影面积”。
*   **右边**：是在球面上计算出真实面积 $D(h)d\omega_h$ 后，乘以 $\cos\theta_h$ 强制投影到底面上的“投影面积”。

现在，把第三步算出来的 $dx_m \, dy_m = \frac{1}{\cos^3\theta_h} d\omega_h$ 代入到左边：
$$ P^2(x_m, y_m) \left( \frac{1}{\cos^3\theta_h} d\omega_h \right) = D(h) \cos\theta_h d\omega_h $$

两边同时约掉极小的立体角 $d\omega_h$：
$$ P^2(x_m, y_m) \frac{1}{\cos^3\theta_h} = D(h) \cos\theta_h $$

最后，把右边的 $\cos\theta_h$ 除到左边来：
$$ \mathbf{D(h) = P^2(x_m, y_m) \frac{1}{\cos^4\theta_h}} $$

**BOOM！！！真相大白！**

### 终极复盘

你看清楚这个 $\cos^4\theta_h$ 的基因组成了吗？
它其实是 **$3+1$** 拼出来的！

*   其中的 **$3$ 次方**：来源于纯粹的几何数学。是你把“平面的二维坐标”强行扭曲成“球面的角度坐标”时，**雅可比行列式（Jacobian Determinant）** 产生的空间缩放补偿。
*   最后的 **$1$ 次方**：来源于物理学法则。是微表面理论要求把倾斜面积**投影（Projection）** 到宏观表面上，保证能量/面积不凭空多出来的物理修正（也就是那个 $n \cdot h$）。

这就是图形学推导的巅峰浪漫：纯粹的空间代数（雅可比）与刚性的物理法则（面积投影），在这一刻完美咬合，共同缔造了这个统治游戏界的高光公式的核心变换因子！
