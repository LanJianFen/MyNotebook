

太对味了！作为工程师，抛弃那些“左手右手”、“照镜子”的具象化比喻，直接用**代数展开式**来证明，才是最硬核、最严谨的做法。

我们直接用纯粹的**向量代数**和**矩阵乘法**，来推演一遍“为什么不乘这个参数，等式的左右两边就会不成立”。

---

### 1. 设定纯数学初始条件

为了剔除干扰，只看缩放的正负号对等式的影响，我们假设缩放矩阵 $S$ 只有对角线元素，且元素 $a, b, c \in \{1, -1\}$：

$$
S = \begin{bmatrix} 
a & 0 & 0 \\ 
0 & b & 0 \\ 
0 & 0 & c 
\end{bmatrix}
$$
*(注：因为 $a, b, c$ 要么是1要么是-1，所以它们的平方 $a^2 = b^2 = c^2 = 1$，这个性质待会儿会立大功)*

假设模型空间中有：
*   法线向量：$N = (N_x, N_y, N_z)$
*   切线向量：$T = (T_x, T_y, T_z)$

模型空间中正确的副切线 $B$ 由标准叉乘定义：
$$ B = N \times T = (N_y T_z - N_z T_y, \;\; N_z T_x - N_x T_z, \;\; N_x T_y - N_y T_x) $$
令 $B = (B_x, B_y, B_z)$。

---

### 2. 我们“期望得到”的正确结果

副切线 $B$ 是一个存在于模型表面的向量，它跟着物体一起缩放。所以**世界空间下绝对正确的副切线 $B_{correct}$**，必须是将 $B$ 直接乘以缩放矩阵 $S$：

$$ B_{correct} = S \cdot B = \begin{bmatrix} a & 0 & 0 \\ 0 & b & 0 \\ 0 & 0 & c \end{bmatrix} \begin{bmatrix} B_x \\ B_y \\ B_z \end{bmatrix} = \mathbf{(a B_x, \;\; b B_y, \;\; c B_z)} $$

**这是我们的目标等式。**

---

### 3. Shader 里的“实际计算”过程展开

在 Shader 中，我们并没有先算 $B$ 再乘矩阵，而是**先用矩阵变换了 $N$ 和 $T$，然后再做叉乘**。我们来看看代数展开会发生什么。

经过矩阵 $S$ 变换后的世界空间 $N'$ 和 $T'$：
*   $N' = S \cdot N = (a N_x, \;\; b N_y, \;\; c N_z)$
*   $T' = S \cdot T = (a T_x, \;\; b T_y, \;\; c T_z)$

*(注：严格来说法线应该乘逆转置矩阵，但因为 $a,b,c \in \{1, -1\}$，矩阵 $S$ 的逆转置就是它自己，所以直接乘 $S$ 代数上是等价的)*

现在，Shader 执愣愣地对变换后的向量执行叉乘 $B_{wrong} = N' \times T'$：

我们逐个分量计算叉乘：
**X 分量：**
$$ (N' \times T')_x = (b N_y)(c T_z) - (c N_z)(b T_y) $$
$$ = bc(N_y T_z - N_z T_y) $$
$$ = \mathbf{bc \cdot B_x} $$

**Y 分量：**
$$ (N' \times T')_y = (c N_z)(a T_x) - (a N_x)(c T_z) $$
$$ = ac(N_z T_x - N_x T_z) $$
$$ = \mathbf{ac \cdot B_y} $$

**Z 分量：**
$$ (N' \times T')_z = (a N_x)(b T_y) - (b N_y)(a T_x) $$
$$ = ab(N_x T_y - N_y T_x) $$
$$ = \mathbf{ab \cdot B_z} $$

所以，Shader 算出来的实际结果是：
$$ B_{wrong} = \mathbf{(bc \cdot B_x, \;\; ac \cdot B_y, \;\; ab \cdot B_z)} $$

---

### 4. 矛盾爆发与代数修正

把“Shader算出来的”和“期望正确的”放在一起对比：
*   目标结果： $B_{correct} = \mathbf{(a \cdot B_x, \;\; b \cdot B_y, \;\; c \cdot B_z)}$
*   实际算得： $B_{wrong} = \mathbf{(bc \cdot B_x, \;\; ac \cdot B_y, \;\; ab \cdot B_z)}$

**看 X 分量：我们想要系数 $a$，但算出来的是 $bc$。它们等价吗？**
*   如果 $a=1, b=1, c=1$，想要的是 1，算出来的是 1。没问题。
*   如果 $a=-1, b=1, c=1$（单数负），想要的是 -1，但 $b \times c$ 算出来是 1！**系数错了！**

**怎么用数学把 $B_{wrong}$ 强行修正为 $B_{correct}$？**
很简单，把 $B_{wrong}$ 乘以一个标量 $k$，使得 $(k \cdot bc) = a$。

这个标量 $k$ 是什么？**恰好就是 $(a \cdot b \cdot c)$！**

我们把 $B_{wrong}$ 乘以 $(a \cdot b \cdot c)$ 来验证一下：

*   **修正 X：** $(a \cdot b \cdot c) \times (bc \cdot B_x) = a \cdot b^2 \cdot c^2 \cdot B_x$
    因为 $b^2 = 1, c^2 = 1$，结果等于 **$a \cdot B_x$** （完美等于目标！）
*   **修正 Y：** $(a \cdot b \cdot c) \times (ac \cdot B_y) = b \cdot a^2 \cdot c^2 \cdot B_y$
    因为 $a^2 = 1, c^2 = 1$，结果等于 **$b \cdot B_y$** （完美等于目标！）
*   **修正 Z：** $(a \cdot b \cdot c) \times (ab \cdot B_z) = c \cdot a^2 \cdot b^2 \cdot B_z$
    因为 $a^2 = 1, b^2 = 1$，结果等于 **$c \cdot B_z$** （完美等于目标！）

---

### 5. 纯代数结论

通过代数展开，我们证明了：
**$$ (S \cdot N) \times (S \cdot T) = (a \cdot b \cdot c) \times [ S \cdot (N \times T) ] $$**

为了让等号两边相等，Shader 算出来的结果必须乘以 $(a \cdot b \cdot c)$。

而 $(a \cdot b \cdot c)$ 是什么？
它就是矩阵 $S$ 主对角线元素的乘积，也就是**矩阵的行列式（Determinant）**。

*   当 $a, b, c$ 中有**奇数（单数）个负号**时，$(a \cdot b \cdot c) = -1$。
*   当 $a, b, c$ 中有**偶数（双数）个负号**时，$(a \cdot b \cdot c) = 1$。

**这就是 `unity_WorldTransformParams.w` 的底层代数真相。** 
它不是为了配合什么空间感知的特判逻辑，而纯粹是为了**填平“交叉相乘展开式”中因矩阵系数重组而引发的代数残缺**。这是一种极为纯粹的数学必然。
