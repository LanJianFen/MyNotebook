

**你这个问题，直接触及了图形学渲染史上最经典的一次“UI/UX（用户体验）对底层物理的越权修改”！**

你的怀疑是**100%正确**的！
在最原始的物理学和数学推导中，那个精确版 $G_1$ 公式里的 $\alpha$ **根本不是**什么“roughness 的平方”！
**是后来的图形学工程师（特指 Disney 的 Brent Burley），为了拯救美术师的血压，强行给它套上了一个平方的壳子！**

我们把这本时间账翻开，看看 $\alpha$ 是怎么在短短五年内，从一个“冷冰冰的物理常数”变成“美术手中的玩具”的。

---

## 原版 GGX  $\alpha$ 的真身：Bruce Walter (2007 年)

在 2007 年，康奈尔大学的 Bruce Walter 发表了那篇提出 GGX 分布的论文《Microfacet Models for Refraction through Rough Surfaces》。

在他的论文里，无论是在 $D$ 项还是带根号的精确 $G$ 项里，写的全都是 $\alpha_g$。
Walter 在物理上给它的严谨定义是：**微表面山坡斜率的均方根（RMS Slope）。**

在 Walter 的世界里，$\alpha$ 就是 $\alpha$。它代表了真实的物理陡峭程度。
但是，如果直接把这个物理 $\alpha$ 暴露给美术去调参，**一场人机交互的灾难就降临了**。

**【物理参数的灾难表现】**
如果让美术在一个 0 到 1 的滑杆上拖动物理 $\alpha$：
*   当滑杆从 `0.0` 拖到 `0.1` 时：高光会从一个针尖，瞬间“爆炸”成一个巨大的光晕！
*   当滑杆从 `0.5` 拖到 `1.0` 时：你在屏幕上肉眼几乎看不出高光有任何变化！

物理世界的斜率变化，在人类的肉眼感知中是**极度非线性**的！美术师会抓狂地抱怨：“这个参数太难调了！稍微手一抖数值就爆了！”

---

## Disney BRDF：Brent Burley (2012 年)
### G 项
#### α 的参数化

##### 第一宗案：是谁把 $\alpha$ 定义为 $Roughness^2$？

**案犯：Brent Burley (Disney, 2012)**

在 2012 年之前，学术界（比如 GGX 的作者 Walter）公式里的 $\alpha$，就是指微表面斜率的均方根。
但是，Burley 在做美术编辑器时发现了一个要命的体验问题：
*   如果你让滑杆值直接等于 $\alpha$，那么在 $0 \sim 0.1$ 这个区间，高光会发生“剧烈爆炸”式的变化；而在 $0.5 \sim 1.0$ 这个区间，肉眼几乎看不出变化。

为了让滑杆（$0 \sim 1$）的变化在人类肉眼看来是**“线性且均匀”**的（Perceptually Linear），Burley 做了一个极其伟大的重映射：
**$$ \alpha = Roughness^2 $$**
这相当于把美术在低粗糙度时的控制精度强行放大了。从此，整个游戏界的美术都在享受这个“平方”带来的丝滑调参体验。
*（注：这个 $\alpha = Roughness^2$ 毫无争议地用在了 $D$ 项里。）*

##### 第二宗案：是谁把它改成了 $\frac{Roughness + 1}{2}$ 的平方？

**主谋依然是：Brent Burley (Disney, 2012)**

当 Burley 把 $\alpha = Roughness^2$ 代入严谨的 **Smith G 项（精确带根号版）** 时，他发现了一个物理惨案：
由于原版 G 项是一个“单次散射”模型，它把所有被山峰挡住的光全都当成 0（死光）。当材质很粗糙时，边缘（掠射角）因为遮挡太严重，会变得**死黑死黑**的，完全失去了现实中粗糙物体那种“温暖的漫反射边缘”。

为了解决这个“边缘过暗”的 Bug，Burley 做出了一个极其大胆的**参数精神分裂**操作：
他不改 D 项，但他**专门针对 G 项（且仅限于直接光照）**，把传入的粗糙度强行“捏扁”了！

他在论文里写道：
> *“为了减弱这种极端的发黑现象，我们在把 roughness 平方之前，先把它重映射到了 $0.5 \sim 1.0$ 的区间。”*

数学表达式就是：
**$$ \text{用于 G 项的 } \alpha_g = \left( \frac{Roughness + 1}{2} \right)^2 = \frac{(Roughness + 1)^2}{4} $$**
*(你看，哪怕是极度光滑的 Roughness = 0，塞给 G 项的粗糙度也被强行抬高到了 0.25！这让 G 项的遮挡变弱了，凭空“漏”出了一部分光，强行补足了丢失的多重散射能量！)*

##### Burley 的“双重 $\alpha$”

我们在看 Disney 2012 论文时，习惯性地认为粗糙度就是 $\alpha = \text{Roughness}^2$。
**但实际上，Burley 为了不同的目的，定义了两个截然不同的 $\alpha$！**

1. **用于 D 项的 $\alpha$（物理真实的 $\alpha$）**：
   为了让高光的范围变化在视觉上是线性的，Burley 规定：
   **$$ \alpha_D = \text{Roughness}^2 $$**
   （这个 $\alpha_D$ 被塞进了 GGX 的 $D$ 项分母里）。

2. **用于 G 项的 $\alpha$（为了好看而捏造的 $\alpha$）**：
   Burley 发现，如果把上面那个真实的 $\alpha_D$ 直接塞进 G 项，粗糙物体的边缘会死黑。
   为了提亮边缘，他强行在算 G 项之前，把 Roughness 重新映射（Remap）了一下：
   $Roughness_{new} = \frac{Roughness + 1}{2}$
   然后把这个新的 Roughness 平方，得到了专门**只给 G 项用**的 $\alpha$：
   **$$ \alpha_G = \left( \frac{\text{Roughness} + 1}{2} \right)^2 = \frac{(\text{Roughness} + 1)^2}{4} $$**

**Burley 论文里的 $\alpha$ 本身就是分裂的。$D$ 项用的是 $\alpha_D$，$G$ 项用的是 $\alpha_G$！

#### 分离式精确 G 项形式
##### Separable （分离式）
Disney BRDF 发表在 **2012 年**。而指出“分离式是错的，必须用高度相关（Joint / Correlated）”的那篇神级论文（Eric Heitz），发表于 **2014 年**。
在 2012 年的时候，整个地球上的图形学界，都理所当然地认为光线 $L$ 和视线 $V$ 被挡住是两个独立事件。
因此，Burley 顺理成章地使用了 **Separable（分离式）**：$G = G_1(L) \times G_1(V)$。

##### 带根号的精确 GGX
迪士尼是干嘛的？是拍动画电影的（当时正在做《无敌破坏王》）！他们用的是**离线渲染器（Offline Renderer）**。
对于皮克斯和迪士尼的农场机房来说，一帧算几个小时，Shader 里多写几个 `sqrt`（开平方根）根本不痛不痒。

所以，Burley 在原版论文里，**直接照搬了 Walter 2007 年论文里那套极其严格的、带根号的 Smith-GGX 精确 $G_1$ 公式**。根本不屑于用 Christophe Schlick 的廉价多项式去近似它！

迪士尼原版的 G 项代码，还原成数学公式是这样的：
**$$ G_{Disney\_2012} = G_{1\_exact}(L) \times G_{1\_exact}(V) $$**
其中，带根号的精确 $G_{1\_exact}$ 是：
$$ G_{1\_exact}(X) = \frac{2 (N \cdot X)}{(N \cdot X) + \sqrt{\alpha_g^2 + (1 - \alpha_g^2)(N \cdot X)^2}} $$

*(注意：Burley 在这里把传入的粗糙度改成了 $\alpha_g = \left( \frac{Roughness + 1}{2} \right)^2$。也就是我之前提过的，为了压制直接光照下粗糙边缘的高光过曝，而做的“美术重映射”。)*

---
## Epic (2013)
### G项
#### α 的参数化
到了 2013 年，Epic Games 的 Brian Karis 要把这套东西搬进 UE4。
Karis 通过严格的数学求极限（就是我们之前推导的 $\lim_{v \to 0}$），算出了 Schlick 近似公式与精确 Smith-GGX 完美贴合的数学真理：
**$$ k = \frac{\alpha_{真实的}}{2} $$**

**请注意，这里的 $\alpha_{真实的}$，在纯粹的数学推导上，指的绝对是上面那个物理真实的 $\alpha_D$！**

但是，Karis 遇到和 Burley 一样的问题：如果真用物理真实的 $\alpha_D$，直射光的边缘太黑了。
于是，Karis 决定全盘继承 Burley 的“艺术妥协”。他在计算 **直接光照（Direct Lighting）** 时，捏着鼻子把 Burley 那个“造假的 $\alpha_G$”代入了自己的 $k$ 公式里！

代入过程如下：
$$ k_{Direct} = \frac{\alpha_G}{2} $$
$$ k_{Direct} = \frac{\frac{(\text{Roughness} + 1)^2}{4}}{2} $$
**$$ k_{Direct} = \frac{(\text{Roughness} + 1)^2}{8} $$**

你在 UE4 直接光照代码里看到的 $\frac{(R+1)^2}{8}$，本质上是 **Epic 的数学真理 $k = \alpha/2$** 碰上了 **Disney 的造假变量 $\alpha_G = \frac{(R+1)^2}{4}$**，两者杂交生出来的工程怪胎！

#### IBL 环境光，Karis 的物理洁癖大爆发！

如果你以为故事到这里就结束了，那就太小看图形学架构师的物理洁癖了。

Karis 在处理**直接光照**时，向迪士尼的艺术效果妥协了，用了 $\frac{(R+1)^2}{8}$。
但是，当他开始编写 **IBL（环境光 / Split-Sum 预计算 LUT）** 的代码时，他发现：
环境光是四面八方打过来的，如果继续用那个“造假的 $\alpha_G$”，会导致预计算的 DFG 积分出现极其严重的能量不守恒（白炉测试直接爆炸，金属暗部会发光）！

**在物理的底线面前，Karis 忍无可忍，彻底抛弃了 Disney 的 $\alpha_G$ 妥协！**

在 UE4 烘焙环境光 BRDF LUT 的底层源码中，Karis 强制切回了**绝对物理正确**的状态：
他把真正的物理粗糙度 $\alpha_D$ 代入了 $k$ 公式！
$$ k_{IBL} = \frac{\alpha_D}{2} $$
**$$ k_{IBL} = \frac{\text{Roughness}^2}{2} $$**
#### 分离式 Schlick 近似 G 项形式
如果你用的是【UE4 的 Schlick 近似版 G】
Schlick 近似公式是：
$$ G_1(V) = \frac{N \cdot V}{(N \cdot V)(1 - k) + k} $$

注意看分母，**没有根号，而且 $k$ 是一次方的！**
把 $k = \frac{(Roughness + 1)^2}{8}$ 代进去：
分母 = $(N \cdot V) \left(1 - \frac{(Roughness + 1)^2}{8}\right) + \frac{(Roughness + 1)^2}{8}$

**发现了吗？！**
因为 Schlick 大神用绝妙的多项式插值干掉了根号，所以在简化版里，**`Roughness` 的最高次幂被死死按在了 2 次方！**
这就避免了 GPU 去算极度昂贵的 4 次方（`x*x*x*x`），用最极简的乘加指令跑出了近乎 4 次方曲线的效果！

---

## Frostbite（2014）
