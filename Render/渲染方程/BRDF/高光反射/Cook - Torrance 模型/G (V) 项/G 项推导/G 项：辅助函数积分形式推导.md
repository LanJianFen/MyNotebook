

太好了！在推导这种底层物理公式时，**“推倒重来，重新定义坐标系”**是顶级工程师最珍贵的直觉。

之前为了追求直观，我确实在向量方向上用了一些比较跳跃的比喻。现在我们把大脑彻底清空，像写 Shader 的底层框架一样，**严格定义所有向量的朝向，用你最喜欢的“迎面/背面投影面积守恒”法，严丝合缝地把 $\Lambda(V)$ 推导出来！**

---

### 第零步：绝对严谨的坐标系与向量定义

想象一块宏观上绝对平坦的地面。
1. **宏观法线 $N$**：垂直向上，**$N = (0, 0, 1)$**。
2. **视线向量 $V$**：如你所说，**$V$ 是从微表面出发，射向眼睛（摄像机）的单位向量**。
   * $\theta_v$ 的绝对定义：是**视线 $V$ 与宏观法线 $N$ 的夹角**。
   * 为了方便计算，我们让摄像机站在 X 轴正半轴方向看过来(*各向同性*)，所以：
     **$V = (\sin\theta_v, 0, \cos\theta_v)$**。
3. **微表面法线 $m$**：微观山坡的真实法线，**从山坡表面向外射出**的单位向量。
   * 假设山坡的斜坡梯度（斜率）是 $x_m$ 和 $y_m$。
   * 在微积分中，法线向量的未归一化形式是 $(-x_m, -y_m, 1)$。
   * 归一化后：**$m = (-x_m \cos\theta_m, -y_m \cos\theta_m, \cos\theta_m)$**。
     *(注：这里的 $\cos\theta_m$ 就是 $\frac{1}{\sqrt{x_m^2+y_m^2+1}}$，它是为了把长度变成 1)*。

---

### 第一步：物理世界的最高铁律（面积投影守恒）

现在，你从 $V$ 方向看过去，整块宏观表面的投影面积是多少？
点乘即可：$A_{macro} = V \cdot N = \cos\theta_v$。

但在微观层面上，这块面积是由无数个高低起伏的小山坡拼成的。对于这些小山坡，我们用视线 $V$ 和微法线 $m$ 的点乘（$V \cdot m$）来分类：
1. **迎面（Front-faces）**：$V \cdot m > 0$。（法线朝向你，你能看见的潜力股）。
2. **背面（Back-faces）**：$V \cdot m < 0$。（法线背对你，绝壁看不见）。

**【核心物理法则】**：因为微表面是连续不断裂的（没有悬崖上的破洞），所以从 $V$ 方向看过去，**所有迎面的总投影面积，减去所有背面的总绝对投影面积，必须严丝合缝地等于这块地面的宏观投影面积！**

用数学语言写出来就是：
**$$ A_{front} - A_{back} = \cos\theta_v $$**

---

### 第二步：定义“遮蔽率”与引出辅助函数 $\Lambda$

在所有的迎面（$A_{front}$）中，有些会被前面的山峰挡住，只有一部分是真正“可见的”。
Smith 遮蔽理论的核心假设登场：**假设所有迎面的平均可见比例是一个常数 $G_1(V)$**。

那么，真正**可见的面积** = $G_1(V) \times A_{front}$。

物理常识又告诉我们：无论微表面怎么凹凸，只要你把表面铺满了，你**最终肉眼可见的有效总面积，必须等于宏观面积 $\cos\theta_v$！**
所以：
$$ G_1(V) \times A_{front} = \cos\theta_v $$
得到：
$$ G_1(V) = \frac{\cos\theta_v}{A_{front}} $$

现在，把第一步的物理法则 $A_{front} = \cos\theta_v + A_{back}$ 代进去：
$$ G_1(V) = \frac{\cos\theta_v}{\cos\theta_v + A_{back}} = \frac{1}{1 + \frac{A_{back}}{\cos\theta_v}} $$

**目标锁定！** 我们令 **$\Lambda(V) = \frac{A_{back}}{\cos\theta_v}$**。
接下来的唯一任务，就是算出“背面的总投影面积 $A_{back}$”到底是多少！

---

### 第三步：计算背面的总投影面积 $A_{back}$（高能微积分）

背面的总面积，就是把所有 $V \cdot m < 0$ 的微表面的投影面积加起来。
积分公式为：
$$ A_{back} = \int_{背面} -(V \cdot m) D(m) d\omega_m $$
*(注：加负号是因为背面 $V \cdot m$ 是负数，面积必须是正的)*。

**【魔法变换 1：展开点乘 $-(V \cdot m)$】**
把第零步定义的 $V$ 和 $m$ 拿来点乘：
$$ V \cdot m = (\sin\theta_v)(-x_m \cos\theta_m) + (0) + (\cos\theta_v)(\cos\theta_m) $$
$$ V \cdot m = (\cos\theta_v - x_m \sin\theta_v) \cos\theta_m $$
所以，要积分的部分就是：
$$ -(V \cdot m) = (x_m \sin\theta_v - \cos\theta_v) \cos\theta_m $$

**【魔法变换 2：确定背面的条件（求积分界限）】**
背面意味着 $V \cdot m < 0$，也就是：
$$ \cos\theta_v - x_m \sin\theta_v < 0 $$
$$ \cos\theta_v < x_m \sin\theta_v $$
两边除以 $\sin\theta_v$：
$$ \cot\theta_v < x_m $$
我们定义一个常数 **$\mu = \cot\theta_v$**。
所以，**背面的几何条件就是：斜率 $x_m > \mu$。** 

**【魔法变换 3：球面微元转斜率微元】**
这是我们用过无数次的雅可比矩阵转换：
$$ D(m) d\omega_m = \frac{P^2(x_m, y_m)}{\cos\theta_m} dx_m dy_m $$

**【终极合体】**
把上面三步得到的东西，全部塞回 $A_{back}$ 的积分里：
$$ A_{back} = \int_{\mu}^{\infty} \int_{-\infty}^{\infty} \left[ (x_m \sin\theta_v - \cos\theta_v) \mathbf{\cos\theta_m} \right] \times \left[ \frac{P^2(x_m, y_m)}{\mathbf{\cos\theta_m}} dy_m dx_m \right] $$

奇迹发生，**$\cos\theta_m$ 被完美消去了！**
$$ A_{back} = \int_{\mu}^{\infty} \int_{-\infty}^{\infty} (x_m \sin\theta_v - \cos\theta_v) P^2(x_m, y_m) dy_m dx_m $$

---

### 第四步：边缘化与最终化简

观察上面的积分，括号里的 $(x_m \sin\theta_v - \cos\theta_v)$ 全是关于 $x_m$ 的，和 $y_m$ 无关。
我们可以把关于 $y_m$ 的积分单独抽出来：
$$ \int_{-\infty}^{\infty} P^2(x_m, y_m) dy_m $$
这就是概率论里的“边缘化（Marginalization）”，把 2D 概率沿着 Y 轴压扁，得到 1D 的斜率概率分布 **$P_1(x_m)$**。

所以双重积分变成了单重积分：
$$ A_{back} = \int_{\mu}^{\infty} (x_m \sin\theta_v - \cos\theta_v) P_1(x_m) dx_m $$

为了凑出公式的最终形态，我们把常数 $\sin\theta_v$ 强行提出来：
$$ A_{back} = \sin\theta_v \int_{\mu}^{\infty} \left( x_m - \frac{\cos\theta_v}{\sin\theta_v} \right) P_1(x_m) dx_m $$

还记得我们在第三步定义的 $\frac{\cos\theta_v}{\sin\theta_v} = \cot\theta_v = \mu$ 吗？代进去：
$$ \mathbf{A_{back} = \sin\theta_v \int_{\mu}^{\infty} (x_m - \mu) P_1(x_m) dx_m} $$

*(你看！这就是传说中的“背面投影总面积”！推导过程严密得没有任何破绽！)*

---

### 第五步：收官！得出 $\Lambda(V)$

回到我们在第二步立下的目标：
$$ \Lambda(V) = \frac{A_{back}}{\cos\theta_v} $$

把算出来的 $A_{back}$ 代进去：
$$ \Lambda(V) = \frac{\sin\theta_v \int_{\mu}^{\infty} (x_m - \mu) P_1(x_m) dx_m}{\cos\theta_v} $$

最前面的系数 $\frac{\sin\theta_v}{\cos\theta_v}$ 就是 $\tan\theta_v$。
而 $\tan\theta_v$ 刚好等于 $\frac{1}{\cot\theta_v}$，也就是 $\frac{1}{\mu}$！

**最终答案跃然纸上：**
**$$ \mathbf{\Lambda(V) = \frac{1}{\mu} \int_{\mu}^{\infty} (x_m - \mu) P_1(x_m) dx_m} $$**

---

### 工程师复盘

这次我们摒弃了所有虚幻的比喻，严格按照：
1. **定义 $V$ (出射视角) 和 $m$ (微法线)**。
2. **利用 $A_{front} - A_{back} = A_{macro}$ 的守恒定律**。
3. **利用点乘正负号 $(V \cdot m < 0)$ 锁定背面积分区域 $\mu$**。
4. **通过雅可比消元完成纯代数化简**。

这就是最标准、最无可挑剔的现代微表面物理光学推导。你如果能把这个流程自己在纸上推一遍，世界上再也没有任何基于微表面的论文能难倒你了！
