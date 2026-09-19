

明白你的意思了！要像之前推导 DFT 那样，把**“总时间窗口到底是多少”、“离散索引 $n$ 怎么打点”、“连续物理量如何映射成无量纲数字”、“主值周期区间怎么被用户选取的窄带所截取并分配给离散频点 $k$”**这套**严密的物理量纲与离散化骨架**清清楚楚地写出来。

---

### 时域切一刀（把连续时间物理截断）

连续拉普拉斯/连续傅里叶变换的理论求和是 $-\infty \to +\infty$。计算机的内存和采样时间必须是有限的。

1. **确定时域物理总时长：**
   假设 ADC 以时间步长 $\Delta t$（采样率 $f_s = \frac{1}{\Delta t}$）连续采集了 **$N$ 个点**。
   * **总物理观测时间窗口为：**
     $$ T_{\text{total}} = N \cdot \Delta t $$
   * **物理时间点被离散索引 $n$ 精确拆分为：**
     $$ t_n = n \cdot \Delta t \quad (n = 0, 1, 2, \dots, N-1) $$
   * **真实时间覆盖区间为：**
     $$ t \in [0, \, T_{\text{total}}) = [0, \, N\Delta t) $$

2. **计算机数组定义：**
   定义无量纲离散序列：
   $$ \mathbf{x[n] \triangleq x(n\Delta t)} $$

3. **时域截断后的拉普拉斯黎曼和：**
   把无穷积分截断为有限项求和（保留复频率 $s = \sigma + j\Omega$ 和物理量纲 $\Delta t$）：
   $$ X_{\text{有限}}(s) = \left[ \sum_{n=0}^{N-1} x(n\Delta t) \cdot e^{-s(n\Delta t)} \right] \cdot \Delta t $$

---

### Z 域主值区间的周期性与“局部自由选区”

由前面狄拉克梳子严格证明的结论，采样后的复频域在虚轴方向具有严格的周期性：
$$ \mathbf{X_{\text{采样}}\left(s + j\frac{2\pi}{\Delta t}\right) = X_{\text{采样}}(s)} $$

这意味着整个复平面在虚轴（连续物理角频率 $\Omega$）方向以 $\Omega_s = \frac{2\pi}{\Delta t}$ 为周期无限循环。所有的独立信息都被锁在**一个主值条带**内：
$$ \Omega \in \left[ -\frac{\pi}{\Delta t}, \, +\frac{\pi}{\Delta t} \right] \quad \Big(\text{或单边 } \left[ 0, \, \frac{2\pi}{\Delta t} \right) \Big) $$

#### 此时传统 DFT 与 CZT 的根本分歧：
* **标准 DFT 的死板分配：**
  DFT 强制把这整整一个周期宽度 $\frac{2\pi}{\Delta t}$ **死板地均分成 $N$ 份**，起点锁死在 $\Omega = 0$，衰减率锁死在 $\sigma = 0$。
* **CZT 的自由分配（频域切一刀，定义局部放大镜）：**
  用户完全不关心整个周期，用户只想在主值区间内**截取一段极窄的局部频带**，并在这段窄带内打上 **$M$ 个超高密度的采样点**：
  * **选定感兴趣的物理起始频率：** $\Omega_{\text{start}}$
  * **选定感兴趣的物理终止频率：** $\Omega_{\text{end}}$
  * **选定的物理带宽为：** 
    $$ B_{\Omega} = \Omega_{\text{end}} - \Omega_{\text{start}} \ll \frac{2\pi}{\Delta t} $$
  * **选定输出的采样点数：** $M$（$M$ 和时域点数 $N$ 完全独立，可以自由设定为任意整数！）
  
---

### Z 域分配采样点（用离散索引 $k$ 打点）

现在，我们把选定的局部频带均匀切成 $M-1$ 份，并允许引入任意衰减因子（螺旋扫描）：

1. **每一个频点的物理角频率步进（超精细频率分辨率）：**
   $$ \Delta \Omega_{\text{user}} = \frac{B_{\Omega}}{M} = \frac{\Omega_{\text{end}} - \Omega_{\text{start}}}{M} $$
2. 衰减速率，从起始衰减率 $\sigma_{\text{start}}$ 扫描到终止衰减率 $\sigma_{\text{end}}$：

$$ \mathbf{\Delta\sigma \triangleq \frac{\sigma_{\text{end}} - \sigma_{\text{start}}}{M}} $$

3. **第 $k$ 个离散频点的连续复频率 $s_k$：**
   $$ s_k = \sigma_k + j \Omega_k \quad (k = 0, 1, 2, \dots, M-1) $$
   其中：
   * **物理实部（衰减率）：** $\sigma_k = \sigma_0 + k \cdot \Delta \sigma$（若只做纯频谱分析，令 $\sigma_k = 0$）；
   * **物理虚部（角频率）：** $\Omega_k = \Omega_{\text{start}} + k \cdot \Delta \Omega_{\text{user}}$。

4. **利用映射公式 $z = e^{s \Delta t}$ 映射到 Z 平面：**
   $$ z_k = e^{s_k \Delta t} = e^{(\sigma_k + j\Omega_k)\Delta t} $$
   把 $s_k$ 的展开式代入：
   $$ z_k = e^{\big( (\sigma_0 + k\Delta\sigma) + j(\Omega_{\text{start}} + k\Delta\Omega_{\text{user}}) \big)\Delta t} $$
   按初等代数分离变量，拆成**“与 $k$ 无关的初值项”**和**“与 $k$ 相关的步进项”**：
   $$ z_k = \underbrace{\left[ e^{(\sigma_0 + j\Omega_{\text{start}})\Delta t} \right]}_{\mathbf{A\text{（采样起始复数）}}} \cdot \underbrace{\left[ e^{-(\Delta\sigma + j\Delta\Omega_{\text{user}})\Delta t} \right]^{-k}}_{\mathbf{W^{-k}\text{（采样步进复数）}}} $$

**由此严密推出了 CZT 的几何采样点定义式：**
$$ \mathbf{z_k = A \cdot W^{-k}} \quad (k = 0, 1, 2, \dots, M-1) $$
* **$A = |A| e^{j \theta_0}$：** 物理起始位置，角度 $\theta_0 = \Omega_{\text{start}} \Delta t$；
* **$W = |W| e^{-j \phi_0}$：** 物理步进因子，步进角度 $\phi_0 = \Delta \Omega_{\text{user}} \Delta t$。

---

### 代入有限求和式，见证物理量纲的完美湮灭

将时域采样的物理点 $t_n = n\Delta t$ 和频域采样的点 $z_k = A W^{-k}$ 同时代入第四步的时域截断黎曼和中（省略外层的比例常数 $\Delta t$）：

$$ X(z_k) = \sum_{n=0}^{N-1} x(n\Delta t) \cdot (z_k)^{-n} $$
$$ X(z_k) = \sum_{n=0}^{N-1} x[n] \cdot \Big( A \cdot W^{-k} \Big)^{-n} $$
$$ \mathbf{X[k] = \sum_{n=0}^{N-1} \Big( x[n] A^{-n} \Big) \cdot W^{nk}} \quad (k = 0, 1, \dots, M-1) $$

**量纲湮灭结论：**
所有的连续时间量纲 $\Delta t$ 全部被吸收入无量纲参数 $A$ 和 $W$ 之中，式子变成了一个**纯代数的、有限维向量映射问题**（输入 $N$ 维，输出 $M$ 维）。

---

### Bluestein 恒等式拆分核心乘积项 $n \cdot k$

此时遇到了计算瓶颈：指数上是**时域索引 $n$ 与频域索引 $k$ 的乘积项 $nk$**，无法像 DFT 那样凑成标准旋转因子。

引入代数恒等式：
$$ nk = \frac{n^2 + k^2 - (k-n)^2}{2} $$

将其代入底数为 $W$ 的指数项中：
$$ W^{nk} = W^{\frac{n^2}{2}} \cdot W^{\frac{k^2}{2}} \cdot W^{-\frac{(k-n)^2}{2}} $$

代入第七步的求和式：
$$ X[k] = \sum_{n=0}^{N-1} \Big( x[n] A^{-n} \Big) \cdot \left[ W^{\frac{n^2}{2}} \cdot W^{\frac{k^2}{2}} \cdot W^{-\frac{(k-n)^2}{2}} \right] $$

把只与频域索引 $k$ 相关的项 $W^{\frac{k^2}{2}}$ 提到时域求和号外面：
$$ \mathbf{X[k] = W^{\frac{k^2}{2}} \cdot \sum_{n=0}^{N-1} \left[ x[n] A^{-n} W^{\frac{n^2}{2}} \right] \cdot W^{-\frac{(k-n)^2}{2}}} $$

---

### 合体！见证标准线性卷积的诞生（进入计算机极速算力）

定义两个全新的离散无量纲序列：

1. **时域调制序列 $g[n]$（长度为 $N$）：**
   $$ g[n] \triangleq x[n] \cdot A^{-n} \cdot W^{\frac{n^2}{2}} \quad (n = 0, 1, \dots, N-1) $$
2. **频域 Chirp 核序列 $h[n]$：**
   $$ h[n] \triangleq W^{-\frac{n^2}{2}} $$

将定义代入第八步的式子：
$$ X[k] = W^{\frac{k^2}{2}} \cdot \underbrace{\sum_{n=0}^{N-1} g[n] \cdot h[k - n]}_{\mathbf{g[n] \text{ 与 } h[n] \text{ 的标准离散线性卷积！}}} $$

简写为：
$$ \mathbf{X[k] = W^{\frac{k^2}{2}} \cdot \Big( g * h \Big)[k]} \quad (k = 0, 1, \dots, M-1) $$


#### 拆分前：为什么算得极慢？（$O(N \times M)$ 的绝望）

看拆分前的式子：
$$ X[k] = \sum_{n=0}^{N-1} \Big( x[n] A^{-n} \Big) \cdot W^{nk} \quad (k = 0, 1, \dots, M-1) $$

1. **为什么不能直接用标准 FFT 算法？**
   * 标准快速傅里叶变换（Cooley-Tukey FFT）之所以快，是因为它的旋转因子是严格的单位圆等分点 $e^{-j\frac{2\pi}{N}}$，具有对称性和周期性，可以做“奇偶分治”。
   * 但在这里，$W$ 是你**任意选取的局部微小步长**，$M$ 也是你**任意指定的输出点数**。没有了整圆的对称性，标准 FFT 的分治法**彻底失效**！

2. **计算机只能暴力硬算：**
   * 为了算第 0 个频点 $X[0]$，必须把 $n=0 \sim N-1$ 的 $N$ 项加起来（做 $N$ 次乘法）；
   * 为了算第 1 个频点 $X[1]$，又要重算一遍 $N$ 项求和；
   * ……
   * 一共有 $M$ 个频点，计算机总共要执行：
     $$ \text{暴力计算量} = \mathbf{N \times M \text{ 次复数乘法}} \quad \big(\mathcal{O}(NM)\big) $$

#### 拆分的核心魔法：把“乘积 $nk$”变成“差值 $k-n$”

Leo Bluestein 用的恒等式：$nk = \frac{n^2 + k^2 - (k-n)^2}{2}$

拆开后代入求和式：
$$ X[k] = W^{\frac{k^2}{2}} \cdot \underbrace{\sum_{n=0}^{N-1} \overbrace{\left( x[n] A^{-n} W^{\frac{n^2}{2}} \right)}^{g[n]} \cdot \overbrace{W^{-\frac{(k-n)^2}{2}}}^{h[k-n]}}_{\mathbf{\sum_{n} g[n] \cdot h[k-n]}} $$

仔细看红色的求和部分：**$\sum g[n] \cdot h[k-n]$**。
* 在高等数学和信号处理中，一个序列以 $n$ 为自变量，另一个序列以 $k-n$ 为自变量相乘求和，**这在定义上就是「离散线性卷积」！**
* 形式变成了：
  $$ \mathbf{y[k] = (g * h)[k]} $$

#### 见证奇迹：卷积快在哪里？（卷积定理的降维打击）

为什么变成了“卷积”，速度就发生了质的飞跃？

因为微积分与信号处理中有一条至高无上的公理——**卷积定理**：
$$ \text{时域的卷积} \iff \text{频域的直接相乘！} $$

计算机算两个序列的线性卷积，**根本不需要做两层循环累加**，它可以通过以下 3 步“借道超车”：

```
          g[n] ──(补零到 L)──> [ 标准 FFT ] ──┐
                                             │ (频域点对点相乘，只需 L 次乘法)
                                            ├──> [ 标准 IFFT ] ──> 卷积结果 y[k]
          h[n] ──(补零到 L)──> [ 标准 FFT ] ──┘
```


1. 原本序列的长度是 $N$ 和 $M$（它们可能是极其恶心的质数，或者完全不相等）。
2. 但是！线性卷积允许我们在序列末尾**随意补零（Zero-padding）**！
3. 我们直接把长度补到最近的 **$2$ 的整数次幂**（比如选 $L = 2^p \ge N + M - 1$）。
4. **既然长度变成了 $2^p$，计算机就可以直接调用全世界最快、优化到极致的基-2 Cooley-Tukey FFT 算法来算这三次变换！**
---

### 总结全景对照

| 维度 | 标准离散傅里叶变换 (DFT) | 线性调频 Z 变换 (CZT) |
| :--- | :--- | :--- |
| **时域截断** | 采集 $N$ 个点，$T_{\text{total}} = N\Delta t$ | 采集 $N$ 个点，$T_{\text{total}} = N\Delta t$ |
| **频域覆盖** | 强制覆盖完整主值区间 $[0, \frac{2\pi}{\Delta t})$ | **自主选取局部频带** $[\Omega_{\text{start}}, \Omega_{\text{end}}]$ |
| **输出点数** | 输出点数必须等于输入点数 $N$ | **输出点数 $M$ 任意自选**（与 $N$ 解耦） |
| **频点分辨率** | 锁死为 $\Delta\Omega = \frac{2\pi}{N\Delta t}$ | **自由设置为任意微小步长** $\Delta\Omega_{\text{user}} = \frac{B_\Omega}{M}$ |
| **计算内核** | 旋转矩阵直接乘法 $\implies$ FFT 分治加速 | **Bluestein 代数拆解 $\implies$ 快速线性卷积加速** |
