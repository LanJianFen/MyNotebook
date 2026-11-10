
### 一、物理起源 ——“满月悖论（The Full Moon Paradox）”与微观假设的颠覆

在 1994 年之前，所有人算漫反射都用 **Lambert 定律**（$f_r = \frac{\rho}{\pi}$）。
如果一个球体是标准的 Lambert 漫反射体，当你正对着它、并且太阳从你背后顺光照射它（$\boldsymbol{\omega}_i = \boldsymbol{\omega}_o$）时，球体表面的出射亮度是：
$$ L_o = \frac{\rho}{\pi} L_i \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_i) = \frac{\rho}{\pi} L_i \cos\theta_i $$
*   在球体正中心（$\theta_i = 0^\circ$）：$\cos 0^\circ = 1$，最亮；
*   到了球体的轮廓边缘（$\theta_i \to 90^\circ$）：$\cos 90^\circ \to 0$，**球体边缘应该逐漸变黑，呈现出明显的立体球感**。

**但是！请你抬头看一看十五的满月（或者看一眼桌上的粉笔、干燥的黏土、石膏像、水泥墙）：**
满月明明是一个被太阳顺光照射的巨大球体，而且月球表面没有任何镜面高光；但你在夜空中看到的满月，**边缘居然和正中心一模一样亮！

#### Oren 和 Nayar 提出的微观假设（与 高光微表面的本质区别）：
我们之前学的微表面模型，假设微观山坡是由无数面**“绝对光滑的小镜子（Specular Microfacets）”**组成的。

而 Oren 和 Nayar 指出：像月壤、石膏、黏土、粗布料这类粗糙表面，它的微观凸起尺寸远大于光波波长，且每个凸起表面本身就布满了孔隙——**它的每一座微观小山坡（法线为 $\mathbf{m}$），自己本身就不是镜子，*而是一块理想的【Lambert 小漫反射面（小石膏片）】！*

---

### 二、 用“锯齿台阶”看懂 Oren-Nayar 的唯一核心思想

为什么我们需要 Oren-Nayar？它的核心思想其实只有一句话：
**“当表面布满粗糙的漫反射颗粒（锯齿）时，同一束斜着打过来的光，顺着光看（看到锯齿向阳面）和逆着光看（看到锯齿背阴面），亮度是天差地别的！”**

#### 现在把平地换成“锯齿地面（Oren-Nayar 粗糙表面）”！
我们直接在地面上画**一块刻满锯齿（V 型凹槽）的粗糙地面**。

假设我们在地面上刻满了倾斜 $45^\circ$ 的小锯齿，*每一块锯齿斜面自己还是一张普通的 Lambert（Oren-Nayar的核心假设，每个微表面依然遵循均匀反射的基底)*
同样让太阳光从左边以很平的角度（比如 $70^\circ$）斜着照过来：

```
  ☀️ 太阳光 L (从左上方斜射过来)
       \       \       \
        \       \       \
         v       v       v
         /\      /\      /\
  👁️    /  \    /  \    /  \    👁️
 眼睛A  / 暗 \  / 暗 \  / 暗 \  眼睛B
(顺光看)      \/      \/       (逆光看)
       (亮面!)  (亮面!)  (亮面!)
```

请你仔细看上面这幅图，奇迹发生了：

*   **每一座小锯齿都有两面坡**：
    *   **左边的坡（向阳坡）**：它几乎是**正对着太阳光**的！太阳光垂直砸在左坡上（局部夹角接近 $0^\circ, \ \cos 0^\circ \approx 1.0$），所以**左坡被照得雪白雪亮（$100\%$ 亮度）*
    *   **右边的坡（背阴坡）**：它背对太阳，要么斜率极大，要么直接被左边的山峰挡住了阳光，所以**右坡一片漆黑（$0\%$ 亮度）**！

**🔥 现在，关键来了！你把眼睛放在不同位置看这块锯齿地：**

1. **当你站在左边【眼睛 A（顺光看，视线 $\mathbf{v}$ 和光线 $\mathbf{l}$ 在同一侧）】**：
   你顺着太阳光的方向低头看过去。
   因为锯齿一个挨着一个，**那些漆黑的右坡（背阴坡）全部躲在山峰后面，被山峰自己挡住了，你的眼睛根本看不见暗面！**
   你的视野里，**100% 全是那些被太阳垂直照得雪亮的左坡（向阳坡）**！
   **视觉结果**：如果是平地，在这个掠射角下本来应该只有 $17\%$ 的昏暗亮度；但因为在粗糙锯齿地上你只看见了向阳坡，这块粗糙地面在边缘看起来**亮得惊人**！这就是**粗糙漫反射的逆反射（月球边缘不暗的原因）**！

2. **当你站在右边【眼睛 B（逆光看，视线 $\mathbf{v}$ 和光线 $\mathbf{l}$ 在相反侧）】**：
   你迎着太阳光的方向低头看过去。
   这一次反过来了！**那些雪亮的左坡（向阳坡）全部背对着你，被挡在山峰背面看不见了！**
   你的视野里，**全都是漆黑阴暗的右坡（背阴坡）**！
   **视觉结果**：在逆光侧，粗糙地面看起来**比普通平地还要黑、还要暗**！

3. **当你站在正上方往下看（或者垂直打光）**：
   那些倾斜的锯齿坡面相对于正上方的光都是斜着的（没有平地正对太阳那么亮），所以在正中心，粗糙表面反而比平地**略微暗一点点**。

---

### 三、 1994 年原版公式是如何用 3 个数学项表达上面这幅图的？

看懂了上面那幅“锯齿向阳坡 vs 背阴坡”的图，你再看 1994 年 Oren 和 Nayar 写出的最终公式，每一个字母都活过来了：

$$ f_{\text{Oren-Nayar}} = \frac{\rho}{\pi} \Big( \underbrace{A}_{\text{1. 基础压暗}} + \underbrace{B}_{\text{2. 粗糙度强度}} \times \underbrace{\max\big(0, \ \cos(\phi_i - \phi_o)\big)}_{\text{3. 顺光/逆光判断器}} \times \underbrace{\sin\alpha \cdot \tan\beta}_{\text{4. 边缘仰角放大器}} \Big) $$

#### 成分构造

1. **积木 1：基础项 $A \le 1$（正中心压暗）**
   $$ A = 1 - 0.5 \frac{\sigma^2}{\sigma^2 + 0.33} $$
   * 如果表面光滑（$\sigma = 0$），$A = 1$（就是普通 Lambert）。
   * 如果表面粗糙（$\sigma > 0$），$A < 1$（对应刚才说的：锯齿斜面在正上方看去不如平地亮，先把基础亮度压低一点）。

2. **积木 2：粗糙度权重 $B \ge 0$（控制逆反射有多强）**
   $$ B = 0.45 \frac{\sigma^2}{\sigma^2 + 0.09} $$
   * 光滑时 $B = 0$（关闭后面的逆反射项）；越粗糙 $B$ 越大。

3. **积木 3：$\max\big(0, \ \cos(\phi_i - \phi_o)\big)$（判断你是【眼睛 A】还是【眼睛 B】！）**
   * $\phi_i - \phi_o$ 是太阳光和你的视线在水平面上的夹角,*它是各向同性*。
   * **如果你是【眼睛 A（顺光，和太阳在同一边）】**：$\phi_i = \phi_o \implies \cos(0^\circ) = \mathbf{1.0}$！开关打开，你看到了向阳坡，享受后面的提亮加成！
   * **如果你是【眼睛 B（逆光，在太阳对面）】**：$|\phi_i - \phi_o| > 90^\circ \implies \cos < 0$，被 $\max(0, \dots)$ **直接切成了 $0$**！开关关闭，你只能拿到前面被压暗的基础值 $A$！

4. **积木 4：$\sin\alpha \cdot \tan\beta$（判断你的视线和太阳是不是都贴近地面边缘！）**
   * 其中 $\alpha = \max(\theta_i, \theta_o)$ 是较大的天顶角，$\beta = \min(\theta_i, \theta_o)$ 是较小的天顶角。
   * 只有当太阳和你的眼睛**全都压低到贴近地面的边缘**（$\theta_i$ 和 $\theta_o$ 都很接近 $90^\circ$）时，较小角 $\beta$ 也很大，$\tan\beta = \frac{\sin\beta}{\cos\beta}$ 才会爆发变大，抵消掉宏观平地的 $\cos\theta_i$ 衰减，让向阳坡的亮度彻底显现出来！
   
#### 基底成分分析

你把整个公式拆解为 **“均匀漫反射底座（$A$） + 逆反射增量（$B \times \text{方位角开关} \times \text{边缘放大器}$）”**，这个视角不仅 100% 准确，而且直接看穿了它和 Disney Diffuse 在代数结构上的同构性！

我们来重点验算你第 2 条提出的猜想：**“$B$ 作为权重分配，$A + B$ 是不是和 $1$ 差不多？”**
把 1994 年的公式拉出来看：
$$ A = 1.0 - \mathbf{0.5} \frac{\sigma^2}{\sigma^2 + 0.33}, \qquad B = \mathbf{0.45} \frac{\sigma^2}{\sigma^2 + 0.09} $$

我们代入不同的粗糙度标准差 $\sigma$，算一算 $A + B$ 的值：
*   **当 $\sigma = 0.0$（绝对光滑）时**：
    $$ A = 1.000, \quad B = 0.000 \implies \mathbf{A + B = 1.000} $$
*   **当 $\sigma = 0.3$ 时**：
    $$ A = 1 - 0.5 \times \frac{0.09}{0.42} \approx 0.893, \quad B = 0.45 \times \frac{0.09}{0.18} = 0.225 \implies \mathbf{A + B = 1.118 \approx 1} $$
*   **当 $\sigma = 1.0$（非常粗糙）时**：
    $$ A = 1 - 0.5 \times \frac{1.0}{1.33} \approx 0.624, \quad B = 0.45 \times \frac{1.0}{1.09} \approx 0.413 \implies \mathbf{A + B = 1.037 \approx 1} $$
*   **当 $\sigma \to \infty$（理论极限粗糙）时**：
    分式 $\frac{\sigma^2}{\sigma^2 + c} \to 1$，$A$ 最多往下扣掉 **$0.5$**（降到 $0.5$），而 $B$ 刚好往上增加 **$0.45$**！
    $$ A \to 0.500, \quad B \to 0.450 \implies \mathbf{A + B = 0.950 \approx 1} ！！ $$

**🔥 你的直觉完全命中了靶心！**
你看公式里的那两个系数：为什么 $A$ 前面是 **$-0.5$**，而 $B$ 前面是 **$+0.45$**？
就是因为**能量守恒的“拆东墙补西墙”**！随着粗糙度变大，Oren-Nayar 从均匀漫反射 $A$ 里面扣掉了最多 $0.5$ 的能量，然后几乎等量地（$0.45$）转移给了控制边缘逆反射的 $B$！

#### 逆反射成分分析

##### 第一组正面对比：$\cos(\phi_i - \phi_o)$ vs Disney 的 $F_{D90}$

你问：**“Disney 的 $F_{D90}$ 只和 $\mathbf{L}$、$\mathbf{V}$ 有关，夹角越小越强；Oren-Nayar 的 $\max\big(0, \cos(\phi_i - \phi_o)\big)$ 的趋势和目标和它是一致的吗？”**

**完全一致！它俩都是用来检测“光线 $\mathbf{L}$ 和视线 $\mathbf{V}$ 是不是在同一侧（顺光逆反射）”的方向开关！**

而且，它俩在数学上只差了一个**“三维夹角 vs 二维水平投影夹角”**的微小区别：

1. **Disney 的 $(\mathbf{L}\cdot\mathbf{H})^2 = \frac{1 + \mathbf{L}\cdot\mathbf{V}}{2}$**：
   它直接用了 $\mathbf{L}$ 和 $\mathbf{V}$ 在**三维空间里的点乘 $\mathbf{L} \cdot \mathbf{V}$**。
   * 当 $\mathbf{L}$ 与 $\mathbf{V}$ 重合（夹角 $0^\circ$）时，$\mathbf{L}\cdot\mathbf{V} = 1$，开关达到最大值；
   * 当 $\mathbf{L}$ 与 $\mathbf{V}$ 相反（夹角 $180^\circ$）时，$\mathbf{L}\cdot\mathbf{V} = -1$，开关降到 $0$。

2. **Oren-Nayar 的 $\max\big(0, \cos(\phi_i - \phi_o)\big)$**：
   它把光线 $\mathbf{L}$ 和视线 $\mathbf{V}$ 先垂直拍扁（投影）到表面的**二维水平切平面**上，再看它们在水平面上的夹角 $\Delta\phi = \phi_i - \phi_o$！
   * 当 $\mathbf{L}$ 和 $\mathbf{V}$ 在水平面上同向（$\Delta\phi = 0^\circ$，顺光）时，$\cos 0^\circ = \mathbf{1.0}$（开关全开）；
   * 当 $\mathbf{L}$ 和 $\mathbf{V}$ 成 $90^\circ$ 侧光或 $>90^\circ$ 逆光时，$\max(0, \cos\Delta\phi) = \mathbf{0.0}$（开关彻底关闭）。

3. 怎么把 $\cos(\phi_i - \phi_o)$ 写成 $\mathbf{L}$、$\mathbf{V}$、$\mathbf{N}$ 的点乘？
	我们在两局前刚写过球面余弦定理（三维点乘展开式）：
	$$ \mathbf{L} \cdot \mathbf{V} = \cos\theta_i \cos\theta_o + \sin\theta_i \sin\theta_o \cos(\phi_i - \phi_o) $$
	因为 $\cos\theta_i = \mathbf{N}\cdot\mathbf{L}$，$\cos\theta_o = \mathbf{N}\cdot\mathbf{V}$，我们把第一项移到等号左边，再把 $\sin\theta_i \sin\theta_o$ 除过去，立刻得到：
	
	**$$ \cos(\phi_i - \phi_o) = \frac{(\mathbf{L} \cdot \mathbf{V}) - (\mathbf{N} \cdot \mathbf{L})(\mathbf{N} \cdot \mathbf{V})}{\sin\theta_i \sin\theta_o} $$**
	
	*(请先记住分母上这个看似讨厌的 $\sin\theta_i \sin\theta_o$，它在下一秒就会和 $\sin\alpha \tan\beta$ 发生奇迹般的同归于尽！)*

##### 第二组正面对比：$\sin\alpha \cdot \tan\beta$ vs Disney 外层的两个 `Scatter`

你问：**“Disney 外层的两个 `Scatter` 由 $\mathbf{N}\cdot\mathbf{L}$ 和 $\mathbf{N}\cdot\mathbf{V}$ 控制，是不是跟这里的 $\alpha$ 和 $\beta$（即 $\sin\alpha \cdot \tan\beta$）描述的目标是一致的？”**

**100% 完全一致！它们都是纯粹由 $\theta_i$ 和 $\theta_o$（即 $\mathbf{N}\cdot\mathbf{L}$ 和 $\mathbf{N}\cdot\mathbf{V}$）控制的“边缘仰角放大器”！**

我们来看看它们俩在球体中心和球体边缘的趋势是不是一模一样：

1.  **球体正中心（光线和视线都垂直于表面，$\theta_i = \theta_o = 0^\circ \implies \mathbf{N}\cdot\mathbf{L} = \mathbf{N}\cdot\mathbf{V} = 1$）**：
    *   **Disney 的 `Scatter`**：$(1 - \mathbf{N}\cdot\mathbf{L})^5 = (1 - 1)^5 = \mathbf{0}$（没有任何边缘提亮，保持中心基准值）。
    *   **Oren-Nayar 的 $\sin\alpha \tan\beta$**：因为 $\alpha = \max(0^\circ, 0^\circ) = 0^\circ, \ \beta = \min(0^\circ, 0^\circ) = 0^\circ$，所以 $\sin 0^\circ \cdot \tan 0^\circ = \mathbf{0}$（同样没有任何边缘提亮，保持中心基准值 $A$）！
2.  **当走向球体的掠射角边缘（$\theta_i, \theta_o \to 90^\circ \implies \mathbf{N}\cdot\mathbf{L}, \mathbf{N}\cdot\mathbf{V} \to 0$）**：
    *   **Disney 的 `Scatter`**：$(1 - \mathbf{N}\cdot\mathbf{L})^5 \to \mathbf{1}$，把边缘逆反射拉高到最大值。
    *   **Oren-Nayar 的 $\sin\alpha \tan\beta$**：$\alpha \to 90^\circ, \ \beta \to 90^\circ$，$\sin\alpha \to 1$，而 $\tan\beta = \frac{\sin\beta}{\cos\beta}$ 因为分母 $\cos\beta \to 0$ 而急剧冲向最大值，把边缘逆反射猛烈拉高！
3. 我们来看 $\sin\alpha \cdot \tan\beta$ 用 $\theta_i$ 和 $\theta_o$ 怎么写：
	因为 $\alpha = \max(\theta_i, \theta_o)$，$\beta = \min(\theta_i, \theta_o)$，无论谁大谁小，它俩一个叫 $\theta_i$、另一个就叫 $\theta_o$！
	所以两个正弦相乘：
	$$ \sin\alpha \cdot \sin\beta \equiv \mathbf{\sin\theta_i \cdot \sin\theta_o} ！！ $$
	而分母上的 $\cos\beta$（较小角的余弦，因为角度越小余弦越大），刚好就是两个余弦里的**较大值**：
	$$ \cos\beta = \cos\big(\min(\theta_i, \theta_o)\big) \equiv \mathbf{\max(\cos\theta_i, \ \cos\theta_o)} = \mathbf{\max(\mathbf{N}\cdot\mathbf{L}, \ \mathbf{N}\cdot\mathbf{V})} ！！ $$
	
	因此，$\sin\alpha \cdot \tan\beta$ 的真身就是：
	**$$ \sin\alpha \cdot \tan\beta = \frac{\sin\alpha \cdot \sin\beta}{\cos\beta} = \frac{\sin\theta_i \sin\theta_o}{\max(\mathbf{N}\cdot\mathbf{L}, \ \mathbf{N}\cdot\mathbf{V})} $$**

#### 把 $\cos(\phi_i - \phi_o)$ 和 $\sin\alpha \cdot \tan\beta$ 乘在一起

$$ \cos(\phi_i - \phi_o) \cdot \sin\alpha \cdot \tan\beta = \left[ \frac{(\mathbf{L} \cdot \mathbf{V}) - (\mathbf{N} \cdot \mathbf{L})(\mathbf{N} \cdot \mathbf{V})}{\cancel{\sin\theta_i \sin\theta_o}} \right] \times \left[ \frac{\cancel{\sin\theta_i \sin\theta_o}}{\max(\mathbf{N}\cdot\mathbf{L}, \ \mathbf{N}\cdot\mathbf{V})} \right] $$

**BOOM！！！分子和分母上的 $\sin\theta_i \sin\theta_o$ 完美约分、同归于尽了！**

留下的式子里，**连半个 $\sin$、$\tan$、$\alpha$、$\beta$、$\phi$ 都没有了，只剩下了纯粹的三个向量点乘**：
**$$ \max\big(0, \cos(\phi_i - \phi_o)\big) \cdot \sin\alpha \cdot \tan\beta \equiv \mathbf{\frac{\max\Big(0, \ (\mathbf{L} \cdot \mathbf{V}) - (\mathbf{N} \cdot \mathbf{L})(\mathbf{N} \cdot \mathbf{V})\Big)}{\max(\mathbf{N} \cdot \mathbf{L}, \ \mathbf{N} \cdot \mathbf{V})}}  $$

---
