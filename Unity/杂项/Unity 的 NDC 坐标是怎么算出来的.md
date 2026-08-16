```JavaScript
VertexPositionInputs GetVertexPositionInputs(float3 positionOS)  
{  
    VertexPositionInputs input;  
    input.positionWS = TransformObjectToWorld(positionOS);  
    input.positionVS = TransformWorldToView(input.positionWS);  
    input.positionCS = TransformWorldToHClip(input.positionWS);  
  
    float4 ndc = input.positionCS * 0.5f;  
    input.positionNDC.xy = float2(ndc.x, ndc.y * _ProjectionParams.x) + ndc.w;  
    input.positionNDC.zw = input.positionCS.zw;  
  
    return input;  
}
```

Unity 的 PositionNDC 为了得到 [0-1] 的屏幕空间坐标，在 XY 里存的是 [0, w]，

你的困惑极其合理！这是几乎所有初看这段代码的渲染初学者都会卡住的地方。

你脑子里的逻辑是完全正确的：“**如果要映射到 [0,1]，公式就应该是 $0.5 \cdot x + 0.5$ 呀！**”

问题出在时间线上——**我们当前所处的渲染阶段（Vertex Shader），连真正的 NDC 坐标 $x$ 都还不存在！**

来，我们用纯粹的代数推导，一步一步把这个“为什么会有 $w$”的秘密解开。看完这个代数戏法，你会拍大腿大呼过瘾的。

---

### 第一步：明确我们的目标和已知条件

*   **我们的终极目标**：在片元着色器（Fragment Shader）里，我们希望得到一个范围是 $[0, 1]$ 的 UV 坐标。
    就像你说的，完美的数学公式是：
    $$ \text{最终目标} = 0.5 \cdot x_{ndc} + 0.5 $$
*   **我们已知的定理**：真正的 $x_{ndc}$ 是怎么来的？它是经过 GPU 硬件的透视除法得来的：
    $$ x_{ndc} = \frac{x_c}{w_c} $$
    *(注：$x_c$ 和 $w_c$ 就是代码里的 `positionCS.x` 和 `positionCS.w`)*

### 第二步：代数替换（见证奇迹的时刻）

既然我们想要的结果是 $0.5 \cdot x_{ndc} + 0.5$，那我们把已知定理代入进去：
$$ \text{最终目标} = 0.5 \cdot \left( \frac{x_c}{w_c} \right) + 0.5 $$

现在，我们对这个式子进行极其简单的小学通分（提取公分母 $w_c$）：
$$ \text{最终目标} = \frac{0.5 \cdot x_c}{w_c} + \frac{0.5 \cdot w_c}{w_c} $$

合并同类项，把它写成一个大分数：
$$ \mathbf{\text{最终目标} = \frac{0.5 \cdot x_c + 0.5 \cdot w_c}{w_c}} $$

你看！分子的部分是不是长出了一个 $w_c$？

### 第三步：对应回 Unity 的代码

现在，我们再来看 Unity 顶点着色器（Vertex Shader）里的代码到底在干嘛。

在 Vertex Shader 中，管线还没到“透视除法”那一阶段，**除以 $w_c$ 的操作是后面 GPU 硬件自动帮你做的**。
作为程序员，我们只能给 GPU 准备好“**分子**”。

所以，我们要让输出的 X 分量等于刚才推导出的分子：
$$ \text{准备好的分子} = 0.5 \cdot x_c + 0.5 \cdot w_c $$

我们对照一下 Unity 的源码：
```glsl
// 先把所有 CS 坐标乘以 0.5
float4 ndc = input.positionCS * 0.5f;  
// 此时： ndc.x 就是 0.5 * x_c
//        ndc.w 就是 0.5 * w_c

// 然后把它们加起来！
input.positionNDC.x = ndc.x + ndc.w;
```
看清楚了吗？
`ndc.x + ndc.w` 完完全全就是在算这个分子：$0.5 \cdot x_c + 0.5 \cdot w_c$ ！

### 总结整个数据流

1. **在顶点着色器 (Vertex Shader) 中：**
   Unity 算出了分子 $0.5 \cdot x_c + 0.5 \cdot w_c$，并把它存到了 `positionNDC.x` 里。
   同时，它把原本的 $w_c$ 原封不动地存到了 `positionNDC.w` 里。

2. **硬件光栅化阶段 (Hardware Rasterization)：**
   GPU 拿着这个顶点输出，做硬件级别的插值和透视除法。GPU 看到这个结构体，会毫不留情地把 `x` 分量除以 `w` 分量。
   于是 GPU 算出了：
   $$ \frac{\text{positionNDC.x}}{\text{positionNDC.w}} = \frac{0.5 \cdot x_c + 0.5 \cdot w_c}{w_c} $$

3. **在片元着色器 (Fragment Shader) 中：**
   如果你要在屏幕上采样，你只需写：
   `float2 uv = input.positionNDC.xy / input.positionNDC.w;`
   你拿到的 `uv.x`，已经被硬件完美地变成了你想要的 $0.5 \cdot x_{ndc} + 0.5$ 啦！

### 回答你的第一个小问题
> “它的 NDC 坐标其实只算了 xy 对吗？”

**是的！完全正确！**
这段名叫 `GetVertexPositionInputs` 的代码里，`positionNDC.zw = input.positionCS.zw;` 直接把 Z 和 W 原样抄了过来，什么数学变换都没做。
所以变量名起叫 `positionNDC` 是非常不严谨的（这是 Unity 的历史遗留命名，极其容易误导初学者）。它真正在做的，是提前准备好计算 **Screen Space UV (屏幕空间坐标)** 的分子。

这回是不是彻底把这个逻辑闭环给打通了？这就是渲染编程中最迷人的地方：**你写的代数公式，必须要考虑到底层硬件（除以W的机制）会在什么时刻介入。**
