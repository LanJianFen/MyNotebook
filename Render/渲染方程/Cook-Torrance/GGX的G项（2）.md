

**你观察得非常敏锐！** 这是一个极好的问题。作为一名前线渲染工程师，我非常欣赏你这种死磕公式细节的态度，这正是写好 Shader 最需要的特质。

你刚才指出的疑惑，恰好揭开了**图形学中最漂亮的一个“数学巧合”（代数消元）**。

我在上一个回答里给你的确实是高度相关的 G 项通用公式骨架：
$$ G_{correlated} = \frac{1}{1 + \Lambda(l) + \Lambda(v)} $$

**那个“1”去哪儿了？为什么它会变成我在第一次回答里写的那个全都是根号的复杂公式？**

答案是：**因为针对 GGX 分布，代数展开后，那个“1”被完美抵消掉了！**

我来带你硬核推导一遍，看看这个精妙的化简过程：

### 第一步：拆解 Lambda ($\Lambda$) 函数

在微表面理论中，辅助函数 $\Lambda$ 是用来描述微表面坡度导致遮挡的统计学项。对于 **GGX 分布**，数学家们推导出的 $\Lambda$ 函数的精确表达式是这样的（设 $\cos\theta = n \cdot m$）：

$$ \Lambda(m) = \frac{-1 + \sqrt{1 + \alpha^2 \tan^2\theta}}{2} $$

在写 Shader 的时候我们不用 `tan`，所以要把 $\tan^2\theta$ 替换成点乘 $\frac{1 - (n \cdot m)^2}{(n \cdot m)^2}$。代入进去化简后，$\Lambda(m)$ 会变成：

$$ \Lambda(m) = \frac{ \sqrt{\alpha^2 + (1 - \alpha^2)(n \cdot m)^2} - (n \cdot m) }{ 2 (n \cdot m) } $$

为了看着不头晕，我们把上面那个长长的根号部分简写为 $A_m$。那么：

$$ \Lambda(m) = \frac{A_m - (n \cdot m)}{2 (n \cdot m)} $$

把这一项拆开，你会得到一个极其关键的结果：
$$ \Lambda(m) = \frac{A_m}{2 (n \cdot m)} - \frac{1}{2} $$

**看！这里出现了一个 $-\frac{1}{2}$！**

### 第二步：见证“1”的消失

现在，我们把针对光线 $l$ 的 $\Lambda(l)$ 和针对视线 $v$ 的 $\Lambda(v)$，一起代回到那个带“1”的通用分母中：

$$ 分母 = 1 + \Lambda(l) + \Lambda(v) $$
$$ 分母 = 1 + \left( \frac{A_l}{2 (n \cdot l)} - \frac{1}{2} \right) + \left( \frac{A_v}{2 (n \cdot v)} - \frac{1}{2} \right) $$

合并常数项：
$$ 分母 = 1 - \frac{1}{2} - \frac{1}{2} + \frac{A_l}{2 (n \cdot l)} + \frac{A_v}{2 (n \cdot v)} $$

**$$ 1 - 0.5 - 0.5 = 0 $$**

**那个“1”就这样在代数展开中被完美消灭了！**

### 第三步：推导最终的 G 项和 V 项

分母的“1”消失后，分母变成了：
$$ \frac{A_l}{2 (n \cdot l)} + \frac{A_v}{2 (n \cdot v)} $$

为了把分母里的分母去掉（通分），我们上下同乘 $2 (n \cdot l)(n \cdot v)$，整个 G 项就变成了：

$$ G_{correlated}(l, v) = \frac{2 (n \cdot l)(n \cdot v)}{(n \cdot v)A_l + (n \cdot l)A_v} $$

把 $A_l$ 和 $A_v$ 的根号全称写回来，**这就是我在第一个回答里展示给你的那个极其复杂的 G 项表达式！**

$$ G(l,v) = \frac{2 (n \cdot l)(n \cdot v)}{(n \cdot l)\sqrt{\alpha^2 + (1-\alpha^2)(n \cdot v)^2} + (n \cdot v)\sqrt{\alpha^2 + (1-\alpha^2)(n \cdot l)^2}} $$

然后，正如我之前强调的，作为渲染工程师，我们在 Shader 里还要把它除以 Cook-Torrance 框架本身的配平分母 $4(n \cdot l)(n \cdot v)$ 来得到**可见性函数 V**：

$$ V = \frac{G}{4(n \cdot l)(n \cdot v)} = \frac{0.5}{(n \cdot l)A_v + (n \cdot v)A_l} $$

### 总结

你不仅看得很仔细，而且你现在已经经历了现代图形学中一个非常经典的公式推导过程：

1. 学术界提出：用联合概率 $G = \frac{1}{1 + \Lambda(l) + \Lambda(v)}$ 来保证物理正确（不丢失能量）。
2. 数学展开：代入 GGX 的 $\Lambda$ 后，常数 1 被抵消。
3. 渲染工程落地：与 Cook-Torrance 分母合并，约掉 $(n \cdot l)$ 和 $(n \cdot v)$，得到极致精简、不会出现 `NaN` 爆点、可以直接写进 GPU 的 V 项代码。

这就是软硬件渲染工程师的日常——**拿着物理学家和数学家的理论，把它榨干水分，揉成硬件最喜欢的样子。**
