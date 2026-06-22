

没问题！我们现在把学术规范严格拉满，全部使用标准的图形学符号（$\omega_i, \omega_o, n, h$），并且**直面菲涅尔项完整的数学形态，不做任何字母换元代称**。

这会是一场极其纯粹的代数推导，但逻辑依然如丝般顺滑。我们开始！

---

### 第一步：写出原始积分与蒙特卡洛公式

我们要解的 Sum 2（BRDF 积分）的原始形式是：
$$ \text{Sum 2} = \int_{\Omega^+} f_r(\omega_o, \omega_i) \cdot (n \cdot \omega_i) \, \mathrm{d}\omega_i $$

既然要用计算机求解，我们把它转成**蒙特卡洛积分估计量（除以 PDF 并求平均）**：
$$ \text{Sum 2} \approx \frac{1}{N} \sum_{k=1}^N \frac{f_r(\omega_o, \omega_i) \cdot (n \cdot \omega_i)}{p(\omega_i)} $$

---

### 第二步：展开分子和分母的“究极形态”

我们把分子（Cook-Torrance BRDF）和分母（带雅可比变换的 PDF）完全展开。

**1. 展开分子：**
$$ f_r(\omega_o, \omega_i) = \frac{D(h) F(\omega_o, h) G(\omega_o, \omega_i)}{4 (n \cdot \omega_o) (n \cdot \omega_i)} $$
别忘了蒙特卡洛公式里还有一个余弦投影项 $(n \cdot \omega_i)$，把它乘上去：
$$ \text{分子} = \frac{D(h) F(\omega_o, h) G(\omega_o, \omega_i)}{4 (n \cdot \omega_o) (n \cdot \omega_i)} \cdot (n \cdot \omega_i) = \frac{D(h) F(\omega_o, h) G(\omega_o, \omega_i)}{4 (n \cdot \omega_o)} $$

**2. 展开分母（入射光 $\omega_i$ 的 PDF）：**
这就是我们之前辛辛苦苦推导出来的结果（包含微面元法线分布和雅可比放缩）：
$$ \text{分母} = p(\omega_i) = \frac{D(h) (n \cdot h)}{4 (\omega_o \cdot h)} $$

---

### 第三步：大面积约分（The Great Cancellation）

把展开好的分子和分母代入蒙特卡洛算式的分式中：

$$ \frac{\text{分子}}{\text{分母}} = \frac{ \frac{D(h) F(\omega_o, h) G(\omega_o, \omega_i)}{4 (n \cdot \omega_o)} }{ \frac{D(h) (n \cdot h)}{4 (\omega_o \cdot h)} } $$

开始无情地消元：
*   上下约掉常数 $4$。
*   上下约掉最复杂的法线分布项 $D(h)$。
*   把最底层的分母 $(\omega_o \cdot h)$ 翻上去。

整理后，积分式内部坍缩成了极其清爽的形态：
$$ \frac{ F(\omega_o, h) \cdot G(\omega_o, \omega_i) \cdot (\omega_o \cdot h) }{ (n \cdot \omega_o) \cdot (n \cdot h) } $$

---

### 第四步：解构菲涅尔项（Schlick 近似）

公式里现在还剩下 $F(\omega_o, h)$。
我们把 Schlick 菲涅尔近似公式**原封不动**地写出来：
$$ F(\omega_o, h) = F_0 + (1 - F_0)(1 - \omega_o \cdot h)^5 $$

我们的目标是**把材质属性 $F_0$ 剥离出来**，让整个积分与具体材质无关。我们直接把公式乘开：
$$ F(\omega_o, h) = F_0 + (1 - \omega_o \cdot h)^5 - F_0(1 - \omega_o \cdot h)^5 $$

把含有 $F_0$ 的项合并在一起，提出 $F_0$：
$$ F(\omega_o, h) = F_0 \left[ 1 - (1 - \omega_o \cdot h)^5 \right] + (1 - \omega_o \cdot h)^5 $$

---

### 第五步：代入并拆分出 2D LUT 的两个通道

现在，把上面这个展开好的 $F(\omega_o, h)$，原样代回到我们第三步推导出的算式中：

$$ \frac{ \left\{ F_0 \left[ 1 - (1 - \omega_o \cdot h)^5 \right] + (1 - \omega_o \cdot h)^5 \right\} \cdot G(\omega_o, \omega_i) \cdot (\omega_o \cdot h) }{ (n \cdot \omega_o) \cdot (n \cdot h) } $$

因为分式满足分配律，我们以 $F_0$ 为界，直接把这个大分式**撕成两半**：

**第一半（被 $F_0$ 乘的那部分，对应 LUT 的 R 通道，Scale）：**
$$ F_0 \cdot \left( \frac{ \left[ 1 - (1 - \omega_o \cdot h)^5 \right] \cdot G(\omega_o, \omega_i) \cdot (\omega_o \cdot h) }{ (n \cdot \omega_o) \cdot (n \cdot h) } \right) $$

**第二半（纯常数偏移部分，对应 LUT 的 G 通道，Bias）：**
$$ \left( \frac{ (1 - \omega_o \cdot h)^5 \cdot G(\omega_o, \omega_i) \cdot (\omega_o \cdot h) }{ (n \cdot \omega_o) \cdot (n \cdot h) } \right) $$

---

### 终极推导结论

最后，我们把蒙特卡洛的外壳 $\left( \frac{1}{N} \sum_{k=1}^N \right)$ 套回来。

整个 Sum 2 积分，在最严格的数学符号下，被完美等价推导为了：

$$ \text{Sum 2} \approx F_0 \cdot \left[ \frac{1}{N} \sum_{k=1}^N \frac{ \left( 1 - (1 - \omega_o \cdot h)^5 \right) \cdot G(\omega_o, \omega_i) \cdot (\omega_o \cdot h) }{ (n \cdot \omega_o) \cdot (n \cdot h) } \right] + \left[ \frac{1}{N} \sum_{k=1}^N \frac{ (1 - \omega_o \cdot h)^5 \cdot G(\omega_o, \omega_i) \cdot (\omega_o \cdot h) }{ (n \cdot \omega_o) \cdot (n \cdot h) } \right] $$

**你仔细看这两个巨型方括号里的东西：**
里面没有任何 $F_0$（没有材质属性），没有任何光照环境，没有任何位置信息。
里面**只有点积运算**，而这些点积运算全部依赖于生成的向量序列（由粗糙度 Roughness 控制）以及视角角度 $(n \cdot \omega_o)$！

所以 Epic 就可以心安理得地建一张二维循环表：
*   算左边方括号的值，存进图片的红色 (R) 像素里。
*   算右边方括号的值，存进图片的绿色 (G) 像素里。

这就是不借用任何字母代换，最硬核、最原汁原味的 Sum 2 推导全过程！
