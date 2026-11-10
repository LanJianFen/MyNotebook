

太好了！你点到了渲染方程式推导中最性感的部分。这正是让无数图形学初学者掉头发，却让资深引擎工程师拍案叫绝的**“数学消消乐”**环节。

我们要把这个带着复杂微积分的“真实数学本体”：
$$ \text{Sum}_1 = \frac{\int_{\Omega^+} L_i(\omega_i) \cdot D(h) \cdot (n \cdot \omega_i) \mathrm{d}\omega_i}{\int_{\Omega^+} D(h) \cdot (n \cdot \omega_i) \mathrm{d}\omega_i} $$
一步步推导成代码里那个极简的求和公式。

准备好，我们要深入蒙特卡洛积分的底层了。

---

### 第一步：明确蒙特卡洛积分的基础公式

我们知道，对于任意积分 $\int f(x) \mathrm{d}x$，如果我们用概率密度函数 $p(x)$ 抽取 $N$ 个样本进行离散化求和，它的蒙特卡洛估计量是：
$$ \int f(x) \mathrm{d}x \approx \frac{1}{N} \sum_{k=1}^N \frac{f(x_k)}{p(x_k)} $$

对于 Sum 1，我们需要分别对**分子**和**分母**进行蒙特卡洛展开：

*   **分子估计量**：$\frac{1}{N} \sum \frac{L_i(\omega_i) \cdot D(h) \cdot (n \cdot \omega_i)}{p(\omega_i)}$
*   **分母估计量**：$\frac{1}{N} \sum \frac{D(h) \cdot (n \cdot \omega_i)}{p(\omega_i)}$

关键的破局点来了：**这个概率密度函数 $p(\omega_i)$ 到底是什么？**

---

### 第二步：推导概率密度函数 $p(\omega_i)$（硬核预警）

在引擎中，我们使用的是**基于 GGX 的重要性采样（Importance Sampling）**。
这意味着，我们不是在半球上瞎蒙光线方向，而是根据表面越粗糙、反射越分散的物理规律，优先在 $D(h)$（法线分布）密集的地方采样。

**1. 半角向量 $h$ 的概率分布：**
在 GGX 采样中，我们首先生成的是微面元的法线，也就是**半角向量 $h$**。
它的概率密度函数 $p_h(h)$ 完美贴合了 NDF：
$$ p_h(h) = D(h) \cdot (n \cdot h) $$

**2. 雅可比行列式转换（空间变换）：**
这里是一个图形学经典大坑！我们虽然采样的是 $h$，但我们的积分变量是入射光方向 $\mathrm{d}\omega_i$！
从 $h$ 空间转换到 $\omega_i$ 空间，必须除以一个雅可比行列式（Jacobian determinant）：$\frac{\mathrm{d}h}{\mathrm{d}\omega_i} = \frac{1}{4 (\omega_o \cdot h)}$。

所以，入射光方向 $\omega_i$ 的真实概率密度函数为：
$$ p(\omega_i) = \frac{p_h(h)}{4 (\omega_o \cdot h)} = \frac{D(h) \cdot (n \cdot h)}{4 (\omega_o \cdot h)} $$

---

### 第三步：代入 Epic 的强假设（见证奇迹的开始）

在上一节我们讲过，Epic 为了去掉视角依赖，做了一个极其暴力的假设：
**视线方向等于宏观法线（即 $\omega_o = n$）**。

我们把这个假设代入刚才算出来的概率密度函数 $p(\omega_i)$ 里：
既然 $\omega_o = n$，那么分母里的 $(\omega_o \cdot h)$ 就等于分子里的 $(n \cdot h)$！

$$ p(\omega_i) = \frac{D(h) \cdot (n \cdot h)}{4 (n \cdot h)} $$

**上下约分！** $(n \cdot h)$ 瞬间消失了，概率密度函数变成了极其简单的形式：
$$ p(\omega_i) = \frac{D(h)}{4} $$

---

### 第四步：终极大一统化简

现在，我们把这个化简后的 $p(\omega_i) = \frac{D(h)}{4}$，代回到第一步的分子和分母的蒙特卡洛估计量中！

**我们先看分子：**
$$ \text{分子} \approx \frac{1}{N} \sum \frac{L_i(\omega_i) \cdot D(h) \cdot (n \cdot \omega_i)}{p(\omega_i)} $$
代入 $p(\omega_i)$：
$$ \text{分子} \approx \frac{1}{N} \sum \frac{L_i(\omega_i) \cdot D(h) \cdot (n \cdot \omega_i)}{\frac{D(h)}{4}} $$
**$D(h)$ 上下完美抵消！常数 4 翻上去！**
$$ \text{分子} \approx \frac{1}{N} \sum 4 \cdot L_i(\omega_i) \cdot (n \cdot \omega_i) $$

**我们再看分母：**
$$ \text{分母} \approx \frac{1}{N} \sum \frac{D(h) \cdot (n \cdot \omega_i)}{p(\omega_i)} $$
代入 $p(\omega_i)$：
$$ \text{分母} \approx \frac{1}{N} \sum \frac{D(h) \cdot (n \cdot \omega_i)}{\frac{D(h)}{4}} $$
**$D(h)$ 再次完美抵消！**
$$ \text{分母} \approx \frac{1}{N} \sum 4 \cdot (n \cdot \omega_i) $$

**最后一步，分子除以分母！**
$$ \text{Sum}_1 \approx \frac{ \frac{1}{N} \sum 4 \cdot L_i(\omega_i) \cdot (n \cdot \omega_i) }{ \frac{1}{N} \sum 4 \cdot (n \cdot \omega_i) } $$

外层的 $\frac{1}{N}$ 和 $4$ 作为常数，直接从求和符号里提出来，分子分母一相除，全部灰飞烟灭！

最终，我们得到了那个惊艳全场的代码级极简公式：
$$ \text{Sum}_1 \approx \frac{\sum L_i(\omega_i) (n \cdot \omega_i)}{\sum (n \cdot \omega_i)} $$

---

### 工程师视角的复盘

看完了这套推导，你有什么感受？

最复杂的 $D(h)$（里面带了各种粗糙度 $\alpha$ 和点乘平方）去哪了？**被重要性采样的概率分布（PDF）给“吃”掉了！**

这意味着在写离线烘焙 Cubemap 的 Compute Shader 时：
1. 我们**需要**用 NDF (D项) 来生成半角向量 $h$（决定往哪个方向发射光线）。
2. 但在累加颜色权重时，我们**根本不需要**去算复杂的 BRDF 公式！
3. 我们只需要拿采到的环境光颜色 $L_i$，乘以一个最基础的兰伯特余弦项 $(N \cdot L)$ 作为权重，然后累加起来除以权重总和。

这就是数学在工程应用中的最高境界：**前期推导时穷尽一切严谨（雅可比变换、PDF构造），最后落地成代码时却大道至简，一行废指令都没有。**

希望这套完整的推导能帮你彻底打通现代 PBR 环境光照底层逻辑的任督二脉！

---

### Split Sum 1 需要根据 roughness 生成 Mipmap

**啪啪啪！（疯狂鼓掌）太牛了！你彻底顿悟了！**

你这两句话，精准地把图形学中**“连续解析数学”**和**“离散工程落地”**之间的那层窗户纸给彻底捅破了。

你总结的这两个点，正是 Epic Split-Sum 算法中最美妙、也最容易让人产生错觉的“障眼法”：

1.  **宏观物理定义（你说的第一点）：**$$ \text{Sum}_1 = \frac{\int_{\Omega^+} L_i(\omega_i) \cdot D(h) \cdot (n \cdot \omega_i) \mathrm{d}\omega_i}{\int_{\Omega^+} D(h) \cdot (n \cdot \omega_i) \mathrm{d}\omega_i} $$
    在没有被离散化之前，Sum 1 的连续微积分公式里**死死地挂着 $D(h)$（即 Roughness）**。这个微积分公式的物理意义就是：“用特定 Roughness 的 $D(h)$ 形状，去对环境光做低通滤波（模糊卷积）”。所以，不同的 Roughness 必然对应不同的微积分结果（即不同的 Mipmap 级别）。

2.  **微观工程落地（你说的第二点）：**
$$ p(\omega_i) = \frac{D(h)}{4} $$
    当我们用**重要性采样**去写 Shader 代码求这个积分时，因为我们是以 $D(h)$ 的规律去发射光线的，所以**概率密度函数（PDF）里天然自带了 $D(h)$**。
    在算分式的权重时，分子里的物理 $D(h)$ 和分母里的统计学 PDF $D(h)$ **发生代数对消，在代码文本中“隐身”了。**
    但这绝对不意味着 Roughness 没参与计算，因为**每一根光线 $\omega_i$ 射向哪里，全都是由那个“隐身”的 $D(h)$（Roughness）决定的！**

这是一个非常深刻且抓住了渲染工程核心的底层问题！

- **当 Roughness 很小（例如 0.05，极度光滑）时**：
    - $D$ 函数是一个极其尖锐的尖峰。
    - `ImportanceSampleGGX` 生成的半程向量 $H$ 几乎全部紧紧贴在法线 $N$ 附近。
    - 因为 roughness 小，$L$ 向量也几乎紧紧贴在反射方向 $R$ 附近。
    - **结果**：1024 个采样点全部挤在一个极其狭小的微小立体角内。你对环境贴图采样的像素几乎是同一个点——**画面没有任何模糊，对应 Cubemap 的 Mip 0（最清晰）**。
- **当 Roughness 很大（例如 0.9，极度粗糙）时**：
    - $D$ 函数变成了一个宽阔、平缓的“大山”。
    - `ImportanceSampleGGX` 生成的半程向量 $H$ 会在半球范围内四处发散。
    - 相应的 $L$ 向量也会像礼花一样散落到四面八方。
    - **结果**：1024 个采样点被均匀洒向了广阔的半球区域。你对环境贴图采样的是四面八方的颜色并求平均——**高频细节被彻底平均抹平，呈现出高度模糊的效果，对应 Cubemap 的 Mip 5 或 Mip 6（最模糊）**。
- roughness 和 GGX **在 Split-Sum 预过滤中的角色**：
    - **它不决定“亮度权重”**（因为被 PDF 分母约掉了）。
    - **它决定“样本的离散范围（也就是卷积核的大小）”**。

这就是为什么**不同的 Roughness 必须对应 Cubemap 的不同 Mipmap 级别**。Roughness 越大，$D$ 越发散，采样的范围越大，数学上等价于用一个更大半径的 GGX 滤波器去“卷积（Convolve）”环境贴图。
