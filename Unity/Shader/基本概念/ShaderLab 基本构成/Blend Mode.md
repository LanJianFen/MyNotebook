

在 Unity Shader 中，**Blend Mode (混合模式)** 决定了 **“当前在这个像素上计算出的颜色” (Source)** 如何与 **“屏幕上已经存在的颜色” (Destination)** 进行混合。

简单来说，就是你在画布上画新的一笔时，颜料是**覆盖**上去？还是**叠加变亮**？还是像**水墨一样晕染**？

---

### 1. 核心公式 (The Formula)

GPU 硬件执行混合的公式通常是固定的：

$$ \text{最终颜色} = (\text{SrcColor} \times \mathbf{SrcFactor}) \, \mathbf{Op} \, (\text{DstColor} \times \mathbf{DstFactor}) $$

*   **SrcColor (源颜色)**：Shader 当前输出的颜色（你正在画的）。
*   **DstColor (目标颜色)**：屏幕缓冲区里已经有的颜色（背景）。
*   **Op (操作符)**：通常是 **加法 (+)** `Add`，也可以是减法、取最小值等。
*   **Factor (因子)**：这是我们在 Shader 里定义的重点，决定谁占多少比例。

在 Unity ShaderLab 中，语法长这样：
```shader
Blend [SrcFactor] [DstFactor]
```
默认操作符是加法。

---

### 2. 最常用的 4 种混合模式

#### A. 正常透明混合 (Alpha Blending)
这是最常见的透明效果（玻璃、UI、塑料）。

*   **语法**：`Blend SrcAlpha OneMinusSrcAlpha`
*   **公式**：
    $$ \text{Final} = (\text{新颜色} \times \alpha) + (\text{背景颜色} \times (1 - \alpha)) $$
*   **原理**：
    *   如果 Alpha 是 1.0（不透明）：完全显示新颜色，背景乘 0（没了）。
    *   如果 Alpha 是 0.5（半透明）：新颜色占 50%，背景占 50%。
    *   如果 Alpha 是 0.0（全透明）：完全显示背景。

#### B. 线性叠加 (Additive / Linear Dodge)
这是做 **发光特效**（火、光束、魔法、全息投影）的标准模式。

*   **语法**：`Blend One One`
*   **公式**：
    $$ \text{Final} = \text{新颜色} + \text{背景颜色} $$
*   **原理**：
    *   颜色直接加在一起，越加越亮。
    *   **黑色 (0,0,0)** 变成了透明色（因为加 0 等于没变）。
    *   不需要 Alpha 通道参与，黑色就是透明。

#### C. 柔和叠加 (Soft Additive / Screen-like)
比上面的 Additive 更柔和一点，常用于粒子。

*   **语法**：`Blend SrcAlpha One`
*   **原理**：
    *   只有有 Alpha 的地方才会加亮背景。
    *   边缘会过度的更自然。

#### D. 正片叠底 (Multiply)
这是做 **阴影、染色玻璃、变暗特效** 的模式。

*   **语法**：`Blend DstColor Zero`
*   **公式**：
    $$ \text{Final} = (\text{新颜色} \times \text{背景颜色}) + 0 $$
*   **原理**：
    *   利用数学乘法：颜色值是 0~1。
    *   如果你乘以 0.5（灰色），背景就变暗一半。
    *   **白色 (1,1,1)** 变成了透明色（因为乘以 1 等于没变）。

---

### 3. 这些 Factor 到底是什么意思？

为了看懂代码，你需要这个对照表：

| 因子 Keyword | 对应数值 | 含义 |
| :--- | :--- | :--- |
| **One** | 1.0 | 完全保留这一项 |
| **Zero** | 0.0 | 完全丢弃这一项 |
| **SrcColor** | (R, G, B) | 用新颜色的亮度作为权重 |
| **SrcAlpha** | Alpha | 用新颜色的透明度作为权重 |
| **DstColor** | (R, G, B) | 用背景颜色的亮度作为权重 |
| **DstAlpha** | Alpha | 用背景颜色的透明度作为权重 |
| **OneMinus...** | 1.0 - ... | 上面的值取反 (反相) |

---

### 4. 只有加法吗？ (BlendOp)

默认是加法 `Add`，但有时候也会用到其他的。
语法：`BlendOp [Operation]`

*   **Add** (默认)：`Src + Dst` (变亮)
*   **Sub**：`Src - Dst` (扣除背景颜色)
*   **RevSub**：`Dst - Src` (背景减去新颜色，比如反色底片效果)
*   **Min**：`min(Src, Dst)` (取两者较暗的那个，适合做水渍、污垢)
*   **Max**：`max(Src, Dst)` (取两者较亮的那个)

---

### 总结

当你看到 Shader 里写：
```shader
Blend SrcAlpha OneMinusSrcAlpha // -> 正常透明
Blend One One                   // -> 霸道发光 (黑色=透明)
Blend DstColor Zero             // -> 阴影叠加 (白色=透明)
Blend Off                       // -> 不混合 (最快，实体物体用)
```

这些就是告诉显卡：**“不要扔掉旧像素，把新像素按这个比例揉进去！”**
