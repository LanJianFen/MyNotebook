

明白！是我上一步太激动，为了尽快推到终点，在步骤上跳跃了。

这次我们严丝合缝、不跳过任何一个步骤，用最严谨的微积分公式，把**“纯辐射度学转化为普通RGB”**和**“辐射度学转化为光度学RGB”**这两条路径，从头到尾、一步一步地平行推导出来。

为了公式清晰，我们定义：
*   波长积分范围是 $\int_{\lambda}$ (即 380nm 到 780nm)。
*   半球积分范围是 $\int_{\Omega}$。
*   连续的物理 BRDF 为 $f_r(\lambda)$。
*   摄像机 RGB 响应曲线函数为 $\mathbf{C}_{rgb}(\lambda)$。

---

### 第一部分：纯辐射度量学（如何通过交换积分，先把光谱转为普通 RGB）

在这一部分，我们**完全不考虑人眼的绝对物理亮度（没有 683，没有光度学单位）**，仅仅是讨论如何把物理光谱变成屏幕上的普通 RGB。

**第 1 步：物理学的基础（波长级半球积分）**
对于一束入射辐射光谱 $L_{e,in}(\lambda)$，它打在表面上发生反射。对于空间中的**每一个单一波长**，我们必须算一次半球积分，得到出射辐射光谱 $L_{e,out}(\lambda)$：
$$ L_{e,out}(\lambda) = \int_{\Omega} f_r(\lambda) \cdot L_{e,in}(\lambda) \cdot (\omega_i \cdot \mathbf{n}) \, d\omega_i $$

**第 2 步：摄像机的感光（波长级色彩积分）**
摄像机接收到上述完整的连续出射光谱 $L_{e,out}(\lambda)$ 后，与它的 RGB 响应曲线进行积分，得到最终画面的普通 $\mathbf{RGB}_{out}$：
$$ \mathbf{RGB}_{out} = \int_{\lambda} \mathbf{L_{e,out}(\lambda)} \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda $$

**第 3 步：嵌套的双重积分（物理界的绝对公式）**
把第 1 步的 $L_{e,out}(\lambda)$ 完整代入第 2 步的公式中：
$$ \mathbf{RGB}_{out} = \int_{\lambda} \left[ \int_{\Omega} f_r(\lambda) \cdot L_{e,in}(\lambda) \cdot (\omega_i \cdot \mathbf{n}) \, d\omega_i \right] \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda $$
*(此时，先算半球光线交互，后算颜色)*

**第 4 步：交换积分次序（数学等价）**
根据富比尼定理，我们将 $\int_{\lambda}$ 移到里面，把 $\int_{\Omega}$ 移到外面，交换算子顺序：
$$ \mathbf{RGB}_{out} = \int_{\Omega} \left[ \int_{\lambda} f_r(\lambda) \cdot L_{e,in}(\lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] (\omega_i \cdot \mathbf{n}) \, d\omega_i $$
*(注意看中括号里，波长积分被挪到了半球积分的内部)*

**第 5 步：离散化近似（剥离 BRDF）**
为了避免在实时渲染中对波长进行连续积分，引擎做了一个近似：假设材质对红、绿、蓝三个波段的反射率是常数（即粗糙的 $\mathbf{BRDF}_{rgb}$ 向量），并把它从内层波长积分中提取出来：
$$ \mathbf{RGB}_{out} \approx \int_{\Omega} \mathbf{BRDF}_{rgb} \otimes \left[ \int_{\lambda} L_{e,in}(\lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] (\omega_i \cdot \mathbf{n}) \, d\omega_i $$

**第 6 步：坍缩为普通 RGB 输入**
中括号里的这一坨 $\int_{\lambda} L_{e,in}(\lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda$，正是直接把**入射光谱提前转换成的普通 RGB 颜色**（我们叫它 $\mathbf{RGB}_{in}$）。
代入后得到最终公式：
$$ \mathbf{RGB}_{out} \approx \int_{\Omega} \mathbf{BRDF}_{rgb} \otimes \mathbf{RGB}_{in} \cdot (\omega_i \cdot \mathbf{n}) \, d\omega_i $$

**【第一部分结论】：**通过严谨的积分次序交换，我们证明了：**先把入射光谱转成 RGB，再做半球积分，与先做半球积分再转 RGB，数学上是完全等价的**（前提是接受 BRDF 分量化的近似）。

---

### 第二部分：光度学量纲的突变（带入 683 的完整推导）

现在，我们要把美术输入的**目标 Lux（光度学单位）**揉进去了。我们从最源头的物理交互开始，一步都不跳。

**第 1 步：纯物理光线交互依然发生**
光子交互的客观规律是不受人类观察者影响的。所以出射光谱依然是那个辐射度公式：
$$ L_{e,out}(\lambda) = \int_{\Omega} f_r(\lambda) \cdot L_{e,in}(\lambda) \cdot (\omega_i \cdot \mathbf{n}) \, d\omega_i $$

**第 2 步：光度学摄像机的转换（引入 683）**
当我们试图用**光度学单位（Nits）**去衡量这个出射光时，我们必须在感光积分外面乘以 683 常数。得到的输出我们称之为 $\mathbf{PhotoRGB}_{out}$：
$$ \mathbf{PhotoRGB}_{out} = \mathbf{683} \int_{\lambda} \mathbf{L_{e,out}(\lambda)} \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda $$

**第 3 步：嵌套的双重积分（此时 683 在最外层）**
将第 1 步代入第 2 步，得到带光度学常数的绝对物理公式：
$$ \mathbf{PhotoRGB}_{out} = \mathbf{683} \int_{\lambda} \left[ \int_{\Omega} f_r(\lambda) \cdot L_{e,in}(\lambda) \cdot (\omega_i \cdot \mathbf{n}) \, d\omega_i \right] \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda $$
*(此时，严格遵循：先在半球上算完连续波长的物理碰撞，然后再用 683 和光谱积分把它转化为人类能看懂的光度学数据。)*

**第 4 步：交换积分次序（连带常数一起塞进去）**
现在，我们执行核心魔法！把半球积分 $\int_{\Omega}$ 提出来，把波长积分 $\int_{\lambda}$ 连同常数 $\mathbf{683}$，**一起塞进**半球积分的内部：
$$ \mathbf{PhotoRGB}_{out} = \int_{\Omega} \left[ \mathbf{683} \int_{\lambda} f_r(\lambda) \cdot L_{e,in}(\lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] (\omega_i \cdot \mathbf{n}) \, d\omega_i $$

**第 5 步：离散化近似（再次剥离 BRDF）**
和第一部分一样，引擎将 $f_r(\lambda)$ 近似为离散的常量向量 $\mathbf{BRDF}_{rgb}$，并从内部的波长积分中提取出来：
$$ \mathbf{PhotoRGB}_{out} \approx \int_{\Omega} \mathbf{BRDF}_{rgb} \otimes \left[ \mathbf{683} \int_{\lambda} L_{e,in}(\lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] (\omega_i \cdot \mathbf{n}) \, d\omega_i $$

**第 6 步：见证光度学的坍缩（引擎中 _LightColor 的诞生）**
死死盯住当前公式中括号里的部分：
$$ \left[ \mathbf{683} \int_{\lambda} L_{e,in}(\lambda) \cdot \mathbf{C}_{rgb}(\lambda) \, d\lambda \right] $$
这段数学语言翻译成人类语言就是：**“对入射光谱进行 RGB 感光转换，并乘以 683 使其拥有光度学单位。”**
这，完完全全就是引擎给到 Shader 的那个携带了目标 Lux 的光度学输入参数 $\mathbf{PhotoRGB}_{in}$！

把这个结果代回第 5 步，我们得到了最终的引擎渲染方程：
$$ \mathbf{PhotoRGB}_{out} \approx \int_{\Omega} \mathbf{BRDF}_{rgb} \otimes \mathbf{PhotoRGB}_{in} \cdot (\omega_i \cdot \mathbf{n}) \, d\omega_i $$

---

### 最终总结与对齐

这样一步不跳地推导下来，你的推论被展现得淋漓尽致：

1.  **在第一部分（普通 RGB）中**，我们证明了**单纯的光谱颜色转换**是可以和半球渲染方程交换顺序的。
2.  **在第二部分（光度学 RGB）中**，我们证明了，因为 683 只是一个常数乘数，它完全可以随着波长积分一起，通过**积分交换**进入到半球渲染方程的最深处，并和入射光谱提前抱团，形成 $\mathbf{PhotoRGB}_{in}$。

所以，这根本不是什么破坏物理学定律的歪门邪道，而是**极度硬核的微积分交换律与常数提取**。引擎没有丢弃物理，引擎只是把“必须在物理碰撞最后才能测量的光度学尺度”，通过纯数学手段，等效地转移到了“光线出发之前的光源数据”上！
