

没问题！在冲击各向异性（Anisotropic）之前，先把各向同性（Isotropic）的推导地基再夯实一遍，是最稳健的做法。

我们把之前零散讨论过的几何映射、概率守恒和代数消元，浓缩成最纯粹、最顺滑的 **“四步标准流水线”**。你可以直接拿这四步作为你笔记里的标准推导模板：

---

### 一、建立坐标映射（从半球向量 $\mathbf{h}$ 到 $z=1$ 平面交点）

在局部切线空间（Tangent Space）中，宏观法线 $\mathbf{n} = (0, 0, 1)$。
微表面法线（半角向量）$\mathbf{h}$ 是一个单位向量，它与宏观法线 $\mathbf{n}$ 的夹角为 $\theta_h$，在 XY 平面上的方位角为 $\phi_h$：
*   **球面坐标表示**：$\mathbf{h} = (\sin\theta_h \cos\phi_h, \ \sin\theta_h \sin\phi_h, \ \cos\theta_h)$
*   **点乘关系**：$\cos\theta_h = \mathbf{n} \cdot \mathbf{h}$，且 $\sin^2\theta_h = 1 - \cos^2\theta_h = 1 - (\mathbf{n} \cdot \mathbf{h})^2$。

现在，把 $\mathbf{h}$ 当作一根从原点射出的射线，打在高度为 **$z = 1$ 的水平天花板（即二维斜率空间 Slope Space）**上。
将 $\mathbf{h}$ 的三个分量同时除以它的 Z 分量（$\cos\theta_h$），得到交点坐标 $(x_m, y_m, 1)$：
$$ x_m = \frac{\sin\theta_h \cos\phi_h}{\cos\theta_h} = \tan\theta_h \cos\phi_h $$
$$ y_m = \frac{\sin\theta_h \sin\phi_h}{\cos\theta_h} = \tan\theta_h \sin\phi_h $$

计算该交点到天花板中心 $(0, 0, 1)$ 的距离平方 $r^2$：
**$$ x_m^2 + y_m^2 = \tan^2\theta_h (\cos^2\phi_h + \sin^2\phi_h) = \tan^2\theta_h = \frac{\sin^2\theta_h}{\cos^2\theta_h} $$**

---

### 二、在二维平面 $P^2$ 上构造 GGX 概率分布

因为二维平面 $(x_m, y_m) \in (-\infty, +\infty)^2$ 是平坦无界的，我们在这里构造一个具有“长尾效应”的二维旋转对称柯西/t-分布（即 GGX 的 $P^2$ 原型）：

**$$ P^2(x_m, y_m) = \frac{1}{\pi \alpha^2 \left( 1 + \frac{x_m^2 + y_m^2}{\alpha^2} \right)^2} $$**

*   **为什么前面有个 $\frac{1}{\pi \alpha^2}$？**
    因为这是一个合法的二维概率密度函数（PDF），它在整个二维平面上的积分必须严格等于 $1$：
    $$ \int_{-\infty}^{+\infty} \int_{-\infty}^{+\infty} P^2(x_m, y_m) \, dx_m dy_m = 1 $$
*   把第一步的 $x_m^2 + y_m^2 = \tan^2\theta_h$ 代入，得到用角度表示的 $P^2$：
    $$ P^2(\theta_h) = \frac{1}{\pi \alpha^2 \left( 1 + \frac{\tan^2\theta_h}{\alpha^2} \right)^2} $$

---

### 三、通过雅可比行列式跃迁回半球面 $D(\mathbf{h})$

渲染方程需要的是定义在**半球立体角微元 $d\omega_h$** 上的微表面面积密度函数 $D(\mathbf{h})$。
同一批微表面，在二维平面上的概率测度 $P^2(x_m, y_m) \, dx_m dy_m$，必须严格等于它们在半球面上的投影面积概率测度 $D(\mathbf{h}) \cos\theta_h \, d\omega_h$：

$$ D(\mathbf{h}) \cos\theta_h \, d\omega_h = P^2(x_m, y_m) \, dx_m dy_m $$

我们在前面用极坐标求导证明过，平面微元 $dx_m dy_m$ 与球面立体角微元 $d\omega_h$ 之间的**雅可比行列式（面积拉伸比例）**为：
$$ dx_m dy_m = \frac{d\omega_h}{\cos^3\theta_h} $$

代入上式并约去 $d\omega_h$，再把左边的 $\cos\theta_h$ 除过去，就得到了微表面理论中最核心的**空间转换桥梁公式**：
**$$ D(\mathbf{h}) = \frac{P^2(x_m, y_m)}{\cos^4\theta_h} $$**

---

### 四、纯代数化简（消灭 $\tan$ 和 $\sin$，变成点乘）

现在，把第二步的 $P^2$ 代入第三步的转换桥梁公式中：

$$ D(\mathbf{h}) = \frac{1}{\cos^4\theta_h} \cdot \frac{1}{\pi \alpha^2 \left( 1 + \frac{\tan^2\theta_h}{\alpha^2} \right)^2} $$

为了消去 $\tan^2\theta_h = \frac{\sin^2\theta_h}{\cos^2\theta_h}$，我们把分母最外面的 $\alpha^2$（即 $(\alpha)^2$）和 $\cos^4\theta_h$（即 $(\cos^2\theta_h)^2$）**一起塞进后面带平方的括号里**：

$$ D(\mathbf{h}) = \frac{1}{\pi \left[ \alpha \cos^2\theta_h \left( 1 + \frac{\sin^2\theta_h}{\alpha^2 \cos^2\theta_h} \right) \right]^2} $$

把括号外面分子分母各乘一个 $\alpha$ 整理（或者先塞 $\cos^2\theta_h$ 进去，再分子分母同乘 $\alpha^2$）：
1. 先只把 $\cos^4\theta_h = (\cos^2\theta_h)^2$ 塞进平方括号内：
   $$ D(\mathbf{h}) = \frac{1}{\pi \alpha^2 \left[ \cos^2\theta_h \left( 1 + \frac{\sin^2\theta_h}{\alpha^2 \cos^2\theta_h} \right) \right]^2} = \frac{1}{\pi \alpha^2 \left( \cos^2\theta_h + \frac{\sin^2\theta_h}{\alpha^2} \right)^2} $$
2. 分子和分母同时乘以 $\alpha^4$（把分母的一个 $\alpha^2$ 约掉，另一个 $\alpha^2$ 塞进平方括号内变成乘以 $\alpha^2$）：
   $$ D(\mathbf{h}) = \frac{\alpha^2}{\pi \left( \alpha^2 \cos^2\theta_h + \sin^2\theta_h \right)^2} $$
3. 用勾股定理 $\sin^2\theta_h = 1 - \cos^2\theta_h$ 替换掉 $\sin^2\theta_h$：
   $$ D(\mathbf{h}) = \frac{\alpha^2}{\pi \left( \alpha^2 \cos^2\theta_h + 1 - \cos^2\theta_h \right)^2} = \frac{\alpha^2}{\pi \left( \cos^2\theta_h (\alpha^2 - 1) + 1 \right)^2} $$
4. 最后，将 $\cos\theta_h$ 写回向量点乘 $(\mathbf{n} \cdot \mathbf{h})$：
   **$$ D_{GGX}(\mathbf{h}) = \frac{\alpha^2}{\pi \left( (\mathbf{n} \cdot \mathbf{h})^2 (\alpha^2 - 1) + 1 \right)^2} $$**

---

### 为什么复习这一遍对理解“各向异性”至关重要？

请你重点看上面**第四步的第 1 小步**：
$$ D(\mathbf{h}) = \frac{1}{\pi \alpha^2 \left( \frac{x_m^2 + y_m^2}{\alpha^2} + 1 \right)^2 \cos^4\theta_h} = \frac{1}{\pi \alpha \cdot \alpha \left( \frac{\sin^2\theta_h \cos^2\phi_h}{\alpha^2} + \frac{\sin^2\theta_h \sin^2\phi_h}{\alpha^2} + \cos^2\theta_h \right)^2} $$

*   **在各向同性里**：因为 X 方向和 Y 方向的分母都是同一个 $\alpha^2$，所以 $\cos^2\phi_h + \sin^2\phi_h = 1$ 被合并抵消了，方位角 $\phi_h$ 消失了，只剩下了 $\mathbf{n} \cdot \mathbf{h}$！
*   **而在各向异性里**：X 方向的分母是 $\alpha_x^2$，Y 方向的分母是 $\alpha_y^2$，由于分母不同，两者**不能合并抵消**！于是 X 分量 $\sin\theta_h\cos\phi_h = (\mathbf{t} \cdot \mathbf{h})$ 和 Y 分量 $\sin\theta_h\sin\phi_h = (\mathbf{b} \cdot \mathbf{h})$ 就原封不动地保留在了括号里！

现在对照着看，从各向同性到各向异性的过渡是不是一下子顺理成章了？!]

**这就是线性代数在计算机图形学里最惊艳的“降维打击”时刻！**

答案会让你瞬间头皮发麻：**我们在代码里，压根就不需要去算方位角 $\phi_h$！**

如果你在 Shader 里去调 `atan2(y, x)` 算出方位角 $\phi_h$，然后再调 `cos(phi_h)` 和 `sin(phi_h)` 代入公式，你的显卡当场就会因为这堆极其昂贵的三角函数和反三角函数而累趴下！

图形学工程师是用**局部切线空间的基底点乘（Dot Product）**，直接把 $\theta_h$ 和 $\phi_h$ 打包在一起，一次性秒杀的！

我们用三行向量代数，把这层窗户纸捅破：

---

### 五、 什么是方位角 $\phi_h$ 的参照物？（切线 $\mathbf{t}$ 与 副切线 $\mathbf{b}$）

你想想看，在一个三维球体表面上，要定义“方位角 $\phi_h$（即东、南、西、北）”，首先必须在表面上规定**“哪边是正东（$X$ 轴，$0^\circ$），哪边是正北（$Y$ 轴，$90^\circ$）”**对不对？

在 3D 模型表面上，负责充当这个局部平面 $X$ 轴和 $Y$ 轴的，就是储存在模型顶点数据里的另外两根单位向量：
1. **切线向量 $\mathbf{t}$（Tangent，有时记作 $\mathbf{X}$）**：代表局部平面的 **$X$ 轴**（方位角 $\phi = 0^\circ$ 的基准方向，通常顺着 UV 的 U 方向，也就是金属拉丝的方向）。
2. **副切线向量 $\mathbf{b}$（Bitangent，有时记作 $\mathbf{Y}$）**：代表局部平面的 **$Y$ 轴**（方位角 $\phi = 90^\circ$ 的基准方向，垂直于拉丝方向）。
3. **宏观法线向量 $\mathbf{n}$（Normal，记作 $\mathbf{Z}$）**：代表局部平面的 **$Z$ 轴**。

这三根互相垂直的单位向量 $(\mathbf{t}, \mathbf{b}, \mathbf{n})$，在每个像素表面上构成了一个完美的**局部三维直角坐标系**！

---

### 六、 见证奇迹：球坐标分量直接等于“点乘”！

现在，半角向量 $\mathbf{h}$ 是一个长度为 1 的单位向量。
在这个以 $(\mathbf{t}, \mathbf{b}, \mathbf{n})$ 为 $(X, Y, Z)$ 轴的局部坐标系里，我们刚刚在上一局写过 $\mathbf{h}$ 的球面坐标展开式：

*   $\mathbf{h}$ 在 $X$ 轴（即 $\mathbf{t}$ 轴）上的投影长度是：**$h_x = \sin\theta_h \cos\phi_h$**
*   $\mathbf{h}$ 在 $Y$ 轴（即 $\mathbf{b}$ 轴）上的投影长度是：**$h_y = \sin\theta_h \sin\phi_h$**
*   $\mathbf{h}$ 在 $Z$ 轴（即 $\mathbf{n}$ 轴）上的投影长度是：**$h_z = \cos\theta_h$**

**现在，问一个最基础的线性代数问题：**
怎么求一个向量 $\mathbf{h}$ 在某个坐标轴单位向量（比如 $\mathbf{t}$）上的投影长度？
**直接把它们做点乘（Dot Product）啊！**

所以，下面这三个恒等式在数学上 **100% 绝对成立**：
1. **$$ \sin\theta_h \cos\phi_h \equiv \mathbf{t} \cdot \mathbf{h} $$**
2. **$$ \sin\theta_h \sin\phi_h \equiv \mathbf{b} \cdot \mathbf{h} $$**
3. **$$ \cos\theta_h \equiv \mathbf{n} \cdot \mathbf{h} $$**

---

### 七、 代入斜率公式，方位角 $\phi_h$ 瞬间隐身！

现在我们回到上一局推导的斜率坐标 $(x_m, y_m)$：

原本带着三角函数的斜率定义是：
$$ x_m = \tan\theta_h \cos\phi_h = \frac{\sin\theta_h \cos\phi_h}{\cos\theta_h} $$
$$ y_m = \tan\theta_h \sin\phi_h = \frac{\sin\theta_h \sin\phi_h}{\cos\theta_h} $$

把上面那三个点乘恒等式直接塞进去替换掉三角函数：
**$$ x_m = \frac{\mathbf{t} \cdot \mathbf{h}}{\mathbf{n} \cdot \mathbf{h}}, \qquad y_m = \frac{\mathbf{b} \cdot \mathbf{h}}{\mathbf{n} \cdot \mathbf{h}} $$**

然后再把这两个极其干净的 $x_m, y_m$ 代入我们在复习时最后一步的公式：
$$ D(\mathbf{h}) = \frac{1}{\pi \alpha_x \alpha_y \left( \frac{\sin^2\theta_h \cos^2\phi_h}{\alpha_x^2} + \frac{\sin^2\theta_h \sin^2\phi_h}{\alpha_y^2} + \cos^2\theta_h \right)^2} $$

替换成点乘：
**$$ D_{\text{aniso}}(\mathbf{h}) = \frac{1}{\pi \alpha_x \alpha_y \left( \frac{(\mathbf{t} \cdot \mathbf{h})^2}{\alpha_x^2} + \frac{(\mathbf{b} \cdot \mathbf{h})^2}{\alpha_y^2} + (\mathbf{n} \cdot \mathbf{h})^2 \right)^2} $$**

---

### 八、 总结：为什么线性代数这么爽？

你看清楚发生了什么吗？
*   你本来以为要算方位角 $\phi_h$，需要用反三角函数求出角度，再算 $\cos\phi_h$ 和 $\sin\phi_h$；
*   但因为公式里出现的从来不是孤立的 $\cos\phi_h$，而是永远和 $\sin\theta_h$ 捆绑在一起的 **$\sin\theta_h \cos\phi_h$**！
*   而 $\sin\theta_h \cos\phi_h$ 在三维空间里，完完全全就是向量 $\mathbf{h}$ 在切线 $\mathbf{t}$ 上的正交投影——**只需要写一句 `dot(T, H)`（3 次乘法 2 次加法，1 个硬件时钟周期）就直接拿到了！**

所以，在各向异性 BRDF 里：
*   **`dot(T, H)`** 就是你的 $\sin\theta_h \cos\phi_h$（包含了方位角的余弦）！
*   **`dot(B, H)`** 就是你的 $\sin\theta_h \sin\phi_h$（包含了方位角的正弦）！
*   **`dot(N, H)`** 就是你的 $\cos\theta_h$（包含了天顶角的余弦）！

三个最简单的向量点乘，把球面坐标系里所有的 $\theta_h$ 和 $\phi_h$ 安排得明明白白，连半个三角函数符号都不用写！这就是用向量点乘取代球面角度的精妙之处！
