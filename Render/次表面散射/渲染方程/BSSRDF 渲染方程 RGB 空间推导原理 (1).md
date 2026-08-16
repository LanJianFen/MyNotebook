

收到！你是对的，引入具体的红光匹配函数确实让符号显得非常臃肿，而且丢失了现代向量化运算的简洁美感。

按照你的思路，我们直接使用**向量值函数** $\mathbf{C}_{rgb}(\lambda)$ 来代表波长到 RGB 三通道的转换因子。这是一个包含了响应权重的 3D 向量。我们最终求得的像素颜色也是一个 3D 向量 $\mathbf{I}_{rgb}$。

这次我们用最干净的数学符号，直击“交换积分顺序”的命门！

---

### 第一步：纯物理的光谱级渲染方程（连续波长域）

针对任意单一波长 $\lambda$，出射的光谱辐射亮度为 $L_o(\lambda)$（这是一个标量）。
我们写出未删减的纯血版解耦 BSSRDF：

$$ L_o(\lambda) = \frac{1}{\pi} F_{to}(\lambda) \int_{A} R(\lambda) \left[ \int_{\Omega_i} F_{ti}(\lambda) \cdot L_i(\lambda) \cdot (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \right] dA $$

> *注：为了符号极简，我隐去了几何参数。$F_{to}$ 是出射菲涅尔，$R$ 是距离扩散，$F_{ti}$ 是入射菲涅尔，$L_i$ 是入射光源。它们都是依赖 $\lambda$ 的标量函数。*

### 第二步：引入感光因子 $\mathbf{C}_{rgb}(\lambda)$（从光谱到向量）

要得到最终在屏幕上显示的 3D 颜色向量 $\mathbf{I}_{rgb}$，我们需要把出射光谱 $L_o(\lambda)$ 与感光因子向量 $\mathbf{C}_{rgb}(\lambda)$ 相乘，并在整个可见光波长域 $\Lambda$ 上积分：

$$ \mathbf{I}_{rgb} = \int_{\Lambda} L_o(\lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda $$

现在，把第一步的 $L_o(\lambda)$ 完整代入到这个积分中：

$$ \mathbf{I}_{rgb} = \int_{\Lambda} \left\{ \frac{1}{\pi} F_{to}(\lambda) \int_{A} R(\lambda) \left[ \int_{\Omega_i} F_{ti}(\lambda) \cdot L_i(\lambda) \cdot (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \right] dA \right\} \mathbf{C}_{rgb}(\lambda) \, d\lambda $$

### 第三步：交换积分顺序（物理真理的显现）

这个大括号里有三重积分：波长积分 $\int_{\Lambda}$、面积积分 $\int_A$、立体角积分 $\int_{\Omega_i}$。
根据多重积分的性质，由于几何项 $(\mathbf{n}_i \cdot \omega_i)$ 根本不依赖波长 $\lambda$，我们完全可以把与 $\lambda$ 无关的项提到外边，**把波长积分 $\int_{\Lambda}$ 压到最深处**。

交换积分顺序后，我们得到**绝对严谨的物理真理方程**：

$$ \mathbf{I}_{rgb(物理)} = \frac{1}{\pi} \int_A \int_{\Omega_i} \underbrace{\mathbf{ \left[ \int_{\Lambda} \Big( F_{to}(\lambda) \cdot R(\lambda) \cdot F_{ti}(\lambda) \cdot L_i(\lambda) \Big) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] }}_{\text{核心波长干涉积分 (输出一个 3D 向量)}} (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \, dA $$

**停下来观察一下中括号里的内容：**
在真实的物理世界里，光线在 $x_i$ 射入、在肉里扩散 $R$、再从 $x_o$ 射出。这所有的光谱属性（4个标量函数），必须在**波长级别（$\lambda$）先相乘**，最后再统一乘以 $\mathbf{C}_{rgb}(\lambda)$ 做积分，最终在这个极微小的局部产生出一个 3D 的 RGB 向量。

---

### 第四步：游戏引擎的“RGB 提前量化”（Shader 逻辑）

现在我们来看看 Shader 是怎么写的。
Shader 无法在运行时计算 $\int_{\Lambda}$。所以，技术美术和引擎底层在**数据输入阶段**，就强行把每个物理属性单独用 $\mathbf{C}_{rgb}(\lambda)$ 积成了 RGB 向量。

也就是，我们给 Shader 传入了 4 个 **3D 向量**：
1.  **灯光颜色：** $\mathbf{L}_{rgb} = \int_{\Lambda} L_i(\lambda) \mathbf{C}_{rgb}(\lambda) \, d\lambda$
2.  **材质 SSS 颜色：** $\mathbf{R}_{rgb} = \int_{\Lambda} R(\lambda) \mathbf{C}_{rgb}(\lambda) \, d\lambda$
3.  **入射高光反相：** $\mathbf{F}_{ti, rgb} = \int_{\Lambda} F_{ti}(\lambda) \mathbf{C}_{rgb}(\lambda) \, d\lambda$
4.  **出射高光反相：** $\mathbf{F}_{to, rgb} = \int_{\Lambda} F_{to}(\lambda) \mathbf{C}_{rgb}(\lambda) \, d\lambda$

然后在 Shader 里面，引擎用**向量的逐元素相乘（Hadamard Product，符号记为 $\odot$）**来代替原本的波长相乘，代入空间积分：

$$ \mathbf{I}_{rgb(Shader)} = \frac{1}{\pi} \int_A \int_{\Omega_i} \underbrace{ \mathbf{ \Big( \mathbf{F}_{to, rgb} \odot \mathbf{R}_{rgb} \odot \mathbf{F}_{ti, rgb} \odot \mathbf{L}_{rgb} \Big) } }_{\text{Shader 里的逐元素向量乘法}} (\mathbf{n}_i \cdot \omega_i) \, d\omega_i \, dA $$

---

### 第五步：终极对比，为什么它们绝对不等价？

我们把物理方程的核心积分，和 Shader 的向量乘法抽出来直接对比。
它们两者相等的前提，是下面这个数学等式必须成立：

$$ \int_{\Lambda} \big( f_1(\lambda) f_2(\lambda) f_3(\lambda) f_4(\lambda) \big) \mathbf{C}_{rgb}(\lambda) \, d\lambda \quad \overset{\text{?}}{=} \quad \Big( \int_{\Lambda} f_1 \mathbf{C}_{rgb} d\lambda \Big) \odot \dots \odot \Big( \int_{\Lambda} f_4 \mathbf{C}_{rgb} d\lambda \Big) $$

**结论极为惨烈：这个等式在微积分中是绝对荒谬的！**

乘积的积分 $\int (A \cdot B) dx$，根本不可能等于积分的逐元素乘积 $(\int A dx) \odot (\int B dx)$。

#### 具体误差体现在哪？（同色异谱的毁灭）
假设 $L_i(\lambda)$ 在 $500nm$ 有个极高的能量峰值，而材质 $R(\lambda)$ 恰好在 $500nm$ 吸收光线（值为0）。
*   **物理真理中：** 在那个 $\int_{\Lambda}$ 积分号内部，$L_i(\lambda) \times R(\lambda)$ 在 $500nm$ 处当场等于 $0$。这部分能量完美消失。
*   **Shader 中：** 灯光积分出了一个 RGB 向量，材质也积分出了一个 RGB 向量。你在 Shader 里用 `LightRGB * MaterialRGB`，它们根本不知道对方在 $500nm$ 处发生过什么！它们只知道自己整体偏绿，两个偏绿的向量一乘，凭空就造出了本不该存在的能量！

### 结语

使用你提议的 $\mathbf{C}_{rgb}(\lambda)$ 向量表示法，整个推演过程变得如手术刀般精准。

通过严谨的交换积分顺序，我们证明了：
**将所有属性降维成 RGB 向量再相乘（实时渲染/游戏引擎），在数学上是对“波长函数先相乘再做向量积分（物理世界/光谱渲染）”的一次极其粗暴的降维打击。**

这就是为什么顶级离线渲染器（如 Arnold, Renderman）都在积极向**全光谱渲染管线（Spectral Rendering Pipeline）**迁移，而实时图形学只能依靠各种 Hack 和 Trick（比如色彩空间转换、ACES 曲线映射）来极力掩盖这层数学上永远无法闭合的误差！
