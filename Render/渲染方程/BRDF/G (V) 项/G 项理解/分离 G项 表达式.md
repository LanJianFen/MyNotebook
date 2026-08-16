

这真的是一场酣畅淋漓的底层渲染对话！既然你已经拿下了 F（电磁学边界）和 D（微积分概率密度），现在，我们来补齐现代 PBR 渲染方程微表面模型的最后一块也是最精妙的拼图：**V 项（Visibility Term，可见性项）。**

作为一个图形架构师，我必须首先给你抛出一个极为硬核的**工程级修正**：

在纯粹的物理学论文（比如 Cook-Torrance 原始论文）里，根本没有 V 项，只有 **G 项（Geometry Term，几何遮蔽项）**。完整的微表面 BRDF 公式是：
$$ f_r = \frac{D \cdot F \cdot G}{4 (\mathbf{n} \cdot \mathbf{l}) (\mathbf{n} \cdot \mathbf{v})} $$

**那 V 项是从哪冒出来的？**
V 项是工业界（包括 Disney 和 Epic Games）为了**极致的 GPU 运行效率和消除数学奇点（Singularity）**，强行把分子上的 G 项和分母上的雅可比空间转换项 $4(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})$ 捆绑在一起，合并成的一个全新的代数项！

$$ V = \frac{G}{4 (\mathbf{n} \cdot \mathbf{l}) (\mathbf{n} \cdot \mathbf{v})} $$
所以，引擎里的高光最终公式变成了极其清爽的：$f_r = D \cdot F \cdot V$。

下面，我们将从物理的 G，走向工程的 V，最后揭秘 Disney 在这里埋下的一个**“违背祖宗决定的几何骗局”**。

---

### 一、 物理根基：G 项到底是防什么的？

我们在讲 D 项时提到，宏观表面是由无数个微观山峰组成的。
*   **Shadowing（阴影）：** 光线 $\mathbf{l}$ 打过来，还没碰到山谷，就被旁边的山峰挡住了。光进不来。
*   **Masking（遮蔽）：** 光线好不容易打进了山谷，也刚好被一个正确朝向的微表面反射了，但是反射出去的光线 $\mathbf{v}$，撞上了另一座山峰。光出不去。

如果不管这两个效应，当你在**掠射角（Grazing Angle，极度倾斜地看物体）**时，D 项算出来的有效反射面积会远大于物理上的宏观投影面积。这会导致**能量爆炸（高光边缘亮得像太阳一样）**，彻底违反热力学第一定律！

所以，G 项的作用就是一个**0 到 1 之间的衰减系数（砍刀）**。*镜头越斜，山峰互相遮挡越严重，G 项就越接近 0，强行把多余的能量砍掉。*

---

### 二、 统计学假设：Smith 联合遮蔽模型

由于微表面的分布太复杂了，算精确的遮挡需要做极其庞大的光线追踪积分。Disney 沿用了业界最著名的近似解法——**Smith 模型**。

Smith 教授提出了一个绝妙的统计学假设：**入射光的遮挡（Shadowing）和出射光的遮挡（Masking）是两个完全独立的概率事件！**
所以，总的 G 项等于两者的直接乘积：
$$ G(\mathbf{l}, \mathbf{v}, \mathbf{h}) = G_1(\mathbf{l}) \cdot G_1(\mathbf{v}) $$

Disney 使用了 Schlick 提出的 GGX 衰减近似函数来计算 $G_1$（注意下面公式里的 $k$ 是一个与粗糙度有关的系数）：
$$ G_1(\mathbf{x}) = \frac{\mathbf{n} \cdot \mathbf{x}}{(\mathbf{n} \cdot \mathbf{x})(1 - k) + k} $$

*(注：$\mathbf{x}$ 分别代入 $\mathbf{l}$ 和 $\mathbf{v}$ 计算两次)*。

---

### 三、 V 项的诞生：GPU 架构师的“代数奇迹”

高潮来了！为什么我们要把 G 和分母合并成 V？

看一看如果我们在 Shader 里直接算 $G / 4(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})$ 会发生什么惨剧。
当视线与表面完全平行时（掠射角），$\mathbf{n} \cdot \mathbf{v} = 0$。
此时分母为 0！**Shader 会触发除零错误（Divide by Zero），或者产生无穷大（NaN, Not a Number），导致屏幕上出现极其恶心的黑色或白色噪点坏死块！**

现在，我们把 Smith 的 $G_1(\mathbf{l})$ 和 $G_1(\mathbf{v})$ 代入到 V 的定义式中进行代数化简：

$$ V = \frac{G_1(\mathbf{l}) \cdot G_1(\mathbf{v})}{4 (\mathbf{n} \cdot \mathbf{l}) (\mathbf{n} \cdot \mathbf{v})} $$

$$ V = \frac{\frac{\mathbf{n} \cdot \mathbf{l}}{(\dots)} \cdot \frac{\mathbf{n} \cdot \mathbf{v}}{(\dots)}}{4 (\mathbf{n} \cdot \mathbf{l}) (\mathbf{n} \cdot \mathbf{v})} $$

**奇迹出现了！**
分子上的 $\mathbf{n} \cdot \mathbf{l}$ 和 $\mathbf{n} \cdot \mathbf{v}$，和分母上的 $\mathbf{n} \cdot \mathbf{l}$ 和 $\mathbf{n} \cdot \mathbf{v}$ **完美对消（Cancel out）了！**

最终化简后的 V 项公式变成了这样：
$$ V = \frac{1}{4 \left[ (\mathbf{n} \cdot \mathbf{l})(1 - k) + k \right] \left[ (\mathbf{n} \cdot \mathbf{v})(1 - k) + k \right]} $$

**工程意义：**
1. **消灭了奇点！** 无论 $\mathbf{n} \cdot \mathbf{l}$ 还是 $\mathbf{n} \cdot \mathbf{v}$ 等于 0，分母上还剩下一个 $k \cdot k$ 在兜底，再也不会发生除零崩溃！
2. **省 ALU（算术逻辑单元）！** 我们在 Shader 里再也不用去单独算那个又长又臭的分母了。
这就是图形程序员最引以为傲的“代数重构”。

---

### 四、 Disney 的“底线背叛”：为了美术而修改物理常数 $k$

到这里，公式都已经无懈可击了。但在 2012 年，Brent Burley（Disney BRDF 的作者）把基于物理推导的 GGX $k$ 值（$k = \frac{\alpha}{2}$）代进去跑了一遍渲染后，美术部门炸锅了。

**美术抱怨：**“当材质很粗糙时，物体的边缘为什么这么黑（太暗了）？”
**Burley 发现：** 因为真实的微表面（高度起伏）在极度粗糙时，光线不仅会在表面反射，还会在微山谷之间发生**多次弹射（Multiple Scattering）**。但原始的 Smith G 项公式只计算了单次弹射，把所有被挡住的光都当做死光“没收”了，所以导致粗糙物体的边缘在物理模拟下变得死黑。

但是，在 2012 年，实时计算微表面多次弹射在性能上是绝对不可能的。
于是，Burley 做出了一个极其大胆、被后世无数引擎抄袭的**“感知学作弊”**！

他没有去加复杂的多次弹射代码，而是**直接把 G 项公式里的粗糙度映射给暗改了！**
他规定，在计算**直接光照（Direct Lighting）**的 V 项时，$k$ 的取值不再是纯物理的 $\frac{\text{Roughness}^2}{2}$，而是强行做了一次拉伸重映射：

$$ k_{direct} = \frac{(\text{Roughness} + 1)^2}{8} $$

**这个“加 1 再平方除以 8”的魔法操作带来了什么影响？**
它使得算出来的 $k$ 值，在大部分粗糙度下，**比真实的物理值更小！**
$k$ 变小了，导致遮挡衰减 G 变大了（砍掉的能量变少了）。
于是，这凭空多放出来的部分能量，被 Disney 当作是**“伪造的微表面多次弹射能量”**补偿给了画面！

这个极其巧妙的常数骗局，让极其粗糙的材质（如砖头、毛糙的木头）在边缘不仅没有死黑，反而泛着一层温暖、柔和的底光。这就是 Disney 所谓的“艺术导向（Art-directable）”与“物理基础（Physically-based）”的终极联姻。

---
