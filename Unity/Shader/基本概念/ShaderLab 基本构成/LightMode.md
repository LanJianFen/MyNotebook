

在 Unity 的 ShaderLab 中，写在 `Pass` 里的 `Tags { "LightMode" = "..." }` 是一个极其关键的**“通信暗号”**。

它的核心作用是：**告诉 Unity 的渲染管线（C# 端），我这个 Pass 具体是干什么活的，请你在正确的时机调用我！**

为了让你彻底吃透它，我们需要把它和 Unity 的**渲染管线（Render Pipeline）**结合起来看。

---

### 1. 核心比喻：车间流水线上的工人

想象 Unity 的渲染管线是一个汽车制造车间。
你的 Shader 里的每一个 `Pass` 就是一个身怀绝技的工人。
`LightMode` 就是贴在他们脑门上的**“工牌 / 职位名称”**。

当车间主任（渲染管线）开始造车时：
1. 主任喊：“**现在开始画阴影映射图（Shadow Map）！请对应的人上！**”
   * 此时，脑门上贴着 `Tags {"LightMode" = "ShadowCaster"}` 的 Pass 就会站出来被执行。
2. 主任喊：“**现在开始画物体受主光照的真正颜色！**”
   * 此时，脑门上贴着 `Tags {"LightMode" = "UniversalForward"}`（如果是 URP 管线）的 Pass 就会站出来被执行。

**如果你不写 `LightMode` 或者写错了会怎样？**
车间主任点名时找不到你，**你的这个 Pass 就永远不会被执行！** 物体会变成粉色，或者完全没有阴影，或者一片死黑。

---

### 2. 衔接上文的“顿悟时刻”：摄像机的深度贴图是怎么来的？

还记得上一问我们讲的 `_CameraDepthTexture`（摄像机深度图）吗？
你肯定好奇过：显卡是怎么在不画颜色的情况下，单独把深度的黑白图画出来的？

**答案就在 `LightMode` 里！**

在 URP 管线中，标准 Shader 里通常会有这样一个隐藏在下面的 Pass：
```shader
Pass
{
    Name "DepthOnly"
    Tags { "LightMode" = "DepthOnly" } // 关键暗号！

    // 这个 Pass 没有繁杂的光照计算，不输出颜色 (ColorMask 0)
    // 它只做一件事：计算顶点位置，然后被 Early-Z 写入深度缓冲区！
    // ...
}
```
当你勾选了摄像机的“开启深度图”选项。
Unity 底层就会在画完真正的彩色画面**之前**，先单独搞一次额外的渲染流程（Draw Call）。在这波流程里，管线会大喊：“所有带有 `DepthOnly` 标签的 Pass 出列！” 
于是，全屏幕的物体都用极低的性能消耗跑了一遍这个特殊的 Pass，拼出了你心心念念的那张 `_CameraDepthTexture`！

---

### 3. 速查表：目前最常用的 LightMode 暗号大全

不同的渲染管线（老版本 Built-in 和新版本 URP）暗号是**不兼容的**。这也是为什么你把网上的老 Shader 放到 URP 里会变粉色的根本原因。

#### 🔥 URP 管线（重点掌握）
| LightMode 名称 | 它是用来干嘛的？ (时机与作用) |
| :--- | :--- |
| **`UniversalForward`** | **主菜**。渲染受光照的彩色画面。管线会把主光源、附加点光源的信息都塞给这个 Pass。 |
| **`ShadowCaster`** | **投影**。画物体在灯光视角的深度图，从而计算它投射在别人身上的阴影。没它物体就没有影子。 |
| **`DepthOnly`** | **测距**。用来生成我们前面说的摄像机深度图。 |
| **`DepthNormals`** | **法线深度图**。除了测距还输出法线方向。常用来做屏幕后期的环境光遮蔽（SSAO）。 |
| **`SRPDefaultUnlit`** | **无光照**。如果你的材质是个纯纯的 UI 或者是毫不受光的特效，用这个。 |

#### 🏚️ 传统内置管线 Built-in (老旧，了解即可)
| LightMode 名称 | 它是用来干嘛的？ (时机与作用) |
| :--- | :--- |
| **`ForwardBase`** | 渲染环境光（GI）、最亮的那一盏主平行光。 |
| **`ForwardAdd`** | 当物体被多个点光源照亮时用到。它是“叠加”上去的，有 3 个点光源，这个 Pass 就会被多跑 3 遍（非常耗性能，所以 URP 把它砍了改成了单 Pass 循环）。 |
| **`ShadowCaster`** | 投影（和 URP 一样）。 |

---

### 4. 代码结构位置（防踩坑）

**千万不要把 Pass 里的 `LightMode` 和 SubShader 里的 Tags 搞混了！**

```shader
Shader "Custom/MyShader"
{
    SubShader
    {
        // 这里的 Tags 是给整个材质用的 (告诉引擎我是透明还是不透明)
        Tags { "RenderType"="Opaque" "Queue"="Geometry" } 

        Pass
        {
            // 这里的 Tags 是给流水线点名用的！
            Tags { "LightMode" = "UniversalForward" } 
            
            CGPROGRAM
            // ...
            ENDCG
        }
        
        Pass
        {
            // 它是这个物体的影子分身
            Tags { "LightMode" = "ShadowCaster" } 
            // ...
        }
    }
}
```

### 总结
`LightMode` 赋予了 Multi-Pass（多通道）Shader 真正的意义。它让一个 Shader 能够身兼数职：既能画物体本身的漂亮颜色（`UniversalForward`），又能帮整个系统生成完美的影子（`ShadowCaster`），还能帮特效师测算屏幕距离（`DepthOnly`）。
