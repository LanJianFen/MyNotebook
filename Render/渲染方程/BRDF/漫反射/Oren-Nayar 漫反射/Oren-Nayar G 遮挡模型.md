
### 一、Oren-Nayar 的V 型凹槽遮蔽函数 $G$ 是怎么设计出来的？

之前我们详细推导过 **Smith $G$ 项**（它假设微表面是杂乱无章的随机高度场，用背面投影面积守恒推导）。
而 1967 年 Torrance-Sparrow、1977 年 Blinn 以及 1994 年 Oren-Nayar 使用的，是另一种极其直观的几何模型——**对称 V 型凹槽模型（Symmetric V-Cavity Model）**

它的公式长这样：
$$ G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) = \min\left( 1, \ \frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)}{\mathbf{m}\cdot\boldsymbol{\omega}_o}, \ \frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_i)}{\mathbf{m}\cdot\boldsymbol{\omega}_i} \right) $$

**那个神秘的 $\frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)}{\mathbf{m}\cdot\boldsymbol{\omega}_o}$ 是怎么来的？我们用 3 行向量加法当场把它推出来！**

```
            宏观法线 n
                ↑
                |     V槽顶部开口 (宽度投影 = 左坡投影 + 右坡投影)
        *-------+-------*  
         \      |      /
   左坡 m \   ↗ | ↖   / 右坡 m' (与左坡关于 n 对称！)
  (面积 a) \ /  |  \ / (面积也是 a)
            \   |   /
             \  |  /
              \ | /
               \|/
```

1. **第一步：写出 V 型凹槽左右两面坡的法线关系**
   想象地面上有一个对称的 V 字形凹槽：
   * 左边这面坡的面积是 $a$，法线是 **$\mathbf{m}$**；
   * 右边对面那面坡的面积也是 $a$，法线是 **$\mathbf{m}'$**。
   因为左右两面坡是关于宏观竖直法线 $\mathbf{n}$ **左右完全对称**的，所以它们的水平分量刚好一正一负抵消，竖直分量都是 $(\mathbf{n}\cdot\mathbf{m})\mathbf{n}$！
   把两个法线向量加起来，得到一个极其优美的向量恒等式：
   **$$ \mathbf{m} + \mathbf{m}' = 2 (\mathbf{n} \cdot \mathbf{m}) \mathbf{n} $$**

2. **第二步：两边同时点乘视线向量 $\boldsymbol{\omega}_o$（算投影面积！）**
   我们在上面这个恒等式的左右两边，同时点乘视线 $\boldsymbol{\omega}_o$（再乘上坡面面积 $a$）：
   **$$ \underbrace{a (\mathbf{m} \cdot \boldsymbol{\omega}_o)}_{\text{左坡 }\mathbf{m}\text{ 的视线投影面积}} + \underbrace{a (\mathbf{m}' \cdot \boldsymbol{\omega}_o)}_{\text{右坡 }\mathbf{m}'\text{ 的视线投影面积}} = \underbrace{2 a (\mathbf{n} \cdot \mathbf{m}) (\mathbf{n} \cdot \boldsymbol{\omega}_o)}_{\text{整个 V 槽顶部开口的宏观视线投影面积！}} $$**

3. **第三步：判断什么时候发生遮挡，一行得出 $G_1$！**
   现在你顺着视线 $\boldsymbol{\omega}_o$ 往这个 V 型槽里看左坡 $\mathbf{m}$（已知左坡朝向你，$\mathbf{m}\cdot\boldsymbol{\omega}_o > 0$）：
   * **情况 A（视线比较高，对面右坡也朝向你：$\mathbf{m}' \cdot \boldsymbol{\omega}_o \ge 0$）**：
     你能同时看到左坡和右坡，一直看到 V 槽的最谷底，**没有任何遮挡**！此时左坡的可见比例是 **$G_1 = 1$**。
   * **情况 B（视线压得很低，对面右坡背对你了：$\mathbf{m}' \cdot \boldsymbol{\omega}_o < 0$）**：
     一旦右坡背对你，说明右坡的坡顶挡在了你的视线前面，把左坡的下半截（谷底部分）给挡住了！
     这时候，你能看到的左坡面积还剩多少？
     你只能透过**整个 V 槽顶部的宏观开口**看到左坡的上半截！也就是说，左坡能漏出来的可见投影面积，精确等于整个 V 槽顶部开口的投影面积 **$2 a (\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)$**！
     用这个“可见投影面积”除以“左坡原本完整的投影面积 $a(\mathbf{m}\cdot\boldsymbol{\omega}_o)$”，直接得出遮蔽比例：
     **$$ G_1(\boldsymbol{\omega}_o, \mathbf{m}) = \frac{2 \cancel{a} (\mathbf{n} \cdot \mathbf{m}) (\mathbf{n} \cdot \boldsymbol{\omega}_o)}{\cancel{a} (\mathbf{m} \cdot \boldsymbol{\omega}_o)} = \mathbf{\frac{2 (\mathbf{n} \cdot \mathbf{m}) (\mathbf{n} \cdot \boldsymbol{\omega}_o)}{\mathbf{m} \cdot \boldsymbol{\omega}_o}} ！！ $$**

   最后，同时考虑视线遮蔽 $G_1(\boldsymbol{\omega}_o, \mathbf{m})$ 和光线阴影 $G_1(\boldsymbol{\omega}_i, \mathbf{m})$，取两者中遮挡更严重的那一个（即 $\min$），就得到了完整的 V 型凹槽遮蔽公式：
   $$ G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{m}) = \min\left( 1, \ \frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)}{\mathbf{m}\cdot\boldsymbol{\omega}_o}, \ \frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_i)}{\mathbf{m}\cdot\boldsymbol{\omega}_i} \right) $$

---

### 二、微表面明明只是一个朝向 $\mathbf{m}$ 的平坦面，为什么会出现一个“V 字坑”？

你说得完全正确：当我们计算 $f_{\text{micro}}$ 时，我们研究的微表面确实只是一个**平坦的、朝向 $\mathbf{m}$ 的小斜面**。

**但是，请你思考一个问题：**
如果宇宙中真的只有这一块孤零零的平坦斜面 $\mathbf{m}$ 悬浮在半空中，周围什么都没有，**那它有可能被“遮挡（$G < 1$）”吗？**
**绝对不可能！** 一块孤立的平板，只要光线和视线在它的正面（$\mathbf{m}\cdot\boldsymbol{\omega}_i > 0, \ \mathbf{m}\cdot\boldsymbol{\omega}_o > 0$），它永远 $100\%$ 可见，$G$ 永远等于 $1$！

**为什么会有遮挡（$G < 1$）？**
因为这块斜面 $\mathbf{m}$ 不是孤立悬浮在空中的，它是长在一块**宏观上是水平的地面（法线为 $\mathbf{n}$）**上的！
*   如果地面上有一块斜坡是**朝右上方倾斜**的（法线为 $\mathbf{m}$），为了让整块宏观地面保持水平（不会一路斜着飞上天），在它的旁边，**必然得有另一块朝左上方倾斜的斜坡（法线为 $\mathbf{m}'$）把高度接回来**！
*   正是因为旁边立着另一块斜坡 $\mathbf{m}'$，当光线或视线压得很低时，**旁边的斜坡 $\mathbf{m}'$ 就会像一堵墙一样，挡住我们正在研究的这块斜坡 $\mathbf{m}$**！

在 1967 年，Torrance 和 Sparrow（以及 1994 年的 Oren-Nayar）为了用数学算出“旁边的斜坡挡住了多少光”，需要给每块微表面 $\mathbf{m}$ 假设一个最简单的“邻居形状”：
*   **Smith 模型（2014 Heitz）的假设**：假设邻居是随机起伏的连续山脉（只看全局统计平均）。
*   **V 型凹槽模型（1967 Torrance / 1994 Oren-Nayar）的假设**：假设每一块朝向 $\mathbf{m}$ 的斜坡（左坡），都刚好和一块与它左右对称的邻居斜坡 $\mathbf{m}'$（右坡）面对面拼在一起，组成一个**对称的 V 字形小凹槽**！

所以：**并不是微表面本身变成了弯曲的 V 字坑，微表面依然是左边的那块平坦斜面 $\mathbf{m}$；而右边的斜面 $\mathbf{m}'$ 是为了计算遮挡而给它配的“对称邻居”！**

---

### 三、 为什么左坡的可见面积，会等于 $2a(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)$？

上一局我用向量点乘写了一句 `a(m · ω_o) + a(m' · ω_o)`，让你产生了巨大的疑惑：*“左坡的面积加上右坡的面积，怎么反而变成左坡被挡住一部分后剩下的可见面积了？越加怎么还能越少呢？”*

这里有两个绝妙的视角，看完**视角 1（天窗几何法）**你一秒钟就能顿悟，看完**视角 2（正负号揭秘）**你就会明白为什么那个 `+` 号其实是**减法**！


#### 视角 1：最直观的“V 槽天窗定理”（不用任何向量公式，一眼看穿！）

请看下面这幅 V 型凹槽的二维剖面图：
*   **左坡 $AO$**：就是我们正在研究的微表面 $\mathbf{m}$，坡面长度为 $a$，倾角为 $\theta_m$。
*   **右坡 $BO$**：是它旁边的对称邻居 $\mathbf{m}'$，坡面长度也是 $a$，倾角也是 $\theta_m$。
*   **顶部水平开口 $AB$（V 槽的“天窗”）**：连接左右两个坡顶 $A$ 和 $B$ 的水平虚线（法线就是宏观法线 $\mathbf{n}$）。
    根据初中三角函数，左半边水平宽是 $a\cos\theta_m$，右半边水平宽也是 $a\cos\theta_m$，所以**顶部天窗 $AB$ 的总水平宽度是**：
    **$$ W_{\text{天窗 } AB} = 2 a \cos\theta_m = 2 a (\mathbf{n} \cdot \mathbf{m}) $$**

```
                  |<----- 顶部水平天窗 AB ----->|  (天窗总宽度 = 2a * cosθ_m = 2a(n·m))
                  A - - - - - - - - - - - - - - B
                   \             ↑             / ↖ 视线 ω_o (从右上方斜着往槽里看)
        【可见部分】 \           n           /    \
         (上半截 AP)  \                    /       \  <-- 刚好擦过右坡顶点 B 的视线边界！
                       P <----------------/---------*
         (下半截 PO)    \               / (右坡 BO 背对视线，挡住了左坡的下半截 PO！)
        【被挡住部分】   \            /
                          \         /
                           \      /
                            \   /
                              O (谷底)
```

现在，你的视线 $\boldsymbol{\omega}_o$ 从右上方以比较低的角度往这个 V 槽里看（此时右坡 $BO$ 背对着你，它的坡顶 $B$ 挡住了左坡的下半截 $PO$，你只能看到左坡的**上半截 $AP$**）：

**🔥 请问：你在视线 $\boldsymbol{\omega}_o$ 方向上，看到的左坡上半截 $AP$ 的投影宽度（光束粗细）是多少？**

你看图上的平行视线光束：
*   这束能照亮左坡上半截 $AP$ 的光束，它的**上边界**刚好擦过左坡顶点 **$A$**；
*   它的**下边界**刚好擦过右坡顶点 **$B$**（然后打在分界点 $P$ 上）！
*   **这意味着：能打中左坡可见部分 $AP$ 的那束光，不多不少，100% 就是刚好从【顶部水平天窗 $AB$】钻进来的那束光！**

既然是同一束平行光，那么**“左坡可见部分 $AP$ 垂直于视线的投影面积”**，在几何上就**绝对等于“顶部水平天窗 $AB$ 垂直于视线的投影面积”**！

而顶部天窗 $AB$ 是一块水平的平地（法线为 $\mathbf{n}$），它的宽度是 $2a(\mathbf{n}\cdot\mathbf{m})$，它朝向视线 $\boldsymbol{\omega}_o$ 的投影面积是多少？直接乘以夹角余弦 $(\mathbf{n}\cdot\boldsymbol{\omega}_o)$：

**$$ \text{左坡漏出来的可见投影面积} = \text{顶部天窗 } AB \text{ 的视线投影面积} = \mathbf{2 a (\mathbf{n} \cdot \mathbf{m}) (\mathbf{n} \cdot \boldsymbol{\omega}_o)} ！！ $$**

再用这个“漏出来的可见投影面积”，除以“左坡原本完整的投影面积 $a(\mathbf{m}\cdot\boldsymbol{\omega}_o)$”，直接就得到了左坡的可见比例 $G_1$：
$$ G_1 = \frac{\text{左坡可见部分 } AP \text{ 的投影}}{\text{左坡完整坡面 } AO \text{ 的投影}} = \frac{2 \cancel{a} (\mathbf{n} \cdot \mathbf{m}) (\mathbf{n} \cdot \boldsymbol{\omega}_o)}{\cancel{a} (\mathbf{m} \cdot \boldsymbol{\omega}_o)} = \frac{2(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)}{\mathbf{m}\cdot\boldsymbol{\omega}_o} $$

#### 视角 2：解开你对代数的疑惑 —— 为什么上一局写的 `a(m · ω_o) + a(m' · ω_o)` 其实是【减法】？

你刚才问：*“为什么把左坡的投影面积加上右坡的投影面积，会等于左坡漏出来的可见面积？”*

请仔细看当右坡 $BO$ 挡住左坡时，右坡的法线 $\mathbf{m}'$ 朝向哪里！
1. 右坡的法线 $\mathbf{m}'$ 是垂直于右坡指向凹槽内部左上方的；
2. 而你的视线 $\boldsymbol{\omega}_o$ 是指向右上方的，此时你是从右坡的**背面（泥土里面）**看过去的！
3. 所以，右坡法线 $\mathbf{m}'$ 与视线 $\boldsymbol{\omega}_o$ 的夹角**大于 $90^\circ$**，点乘 **$\mathbf{m}' \cdot \boldsymbol{\omega}_o$ 是一个【负数】**（即 $\mathbf{m}' \cdot \boldsymbol{\omega}_o = - |\mathbf{m}' \cdot \boldsymbol{\omega}_o| < 0$）！

因此，在恒等式：
$$ a(\mathbf{m} \cdot \boldsymbol{\omega}_o) + a(\mathbf{m}' \cdot \boldsymbol{\omega}_o) = 2a(\mathbf{n} \cdot \mathbf{m})(\mathbf{n} \cdot \boldsymbol{\omega}_o) $$
当中，当发生遮挡（$\mathbf{m}' \cdot \boldsymbol{\omega}_o < 0$）时，左边的 `+` 号加上一个负数，**在算术上其实是在做【减法】**：

**$$ \underbrace{a (\mathbf{m} \cdot \boldsymbol{\omega}_o)}_{\text{左坡完整面积 } AO} \mathbf{-} \underbrace{a |\mathbf{m}' \cdot \boldsymbol{\omega}_o|}_{\text{被右坡背面挡住的面积 } PO} = \underbrace{2a(\mathbf{n} \cdot \mathbf{m})(\mathbf{n} \cdot \boldsymbol{\omega}_o)}_{\text{剩下的可见面积 } AP\text{ (即天窗面积)}} ！！ $$**

这其实就是我们在几天前聊过的 **“迎面投影面积 $-$ 背面投影面积 $=$ 宏观天窗净面积”** 在单个 V 型槽上的完美体现：
*   当右坡没挡住左坡时（$\mathbf{m}' \cdot \boldsymbol{\omega}_o > 0$），右边是正数，天窗面积等于左坡加右坡，此时左坡没有被扣减，可见率 $G_1 = 1$；
*   一旦右坡转成了背面（$\mathbf{m}' \cdot \boldsymbol{\omega}_o < 0$），右坡的背面投影面积 $a|\mathbf{m}' \cdot \boldsymbol{\omega}_o|$ 就正好等于它替左坡挡掉的那截阴影面积 $PO$！从左坡完整面积里减去这截被挡掉的面积，剩下的正好就是顶部天窗的投影面积 $2a(\mathbf{n}\cdot\mathbf{m})(\mathbf{n}\cdot\boldsymbol{\omega}_o)$！

