

原来如此！如果你问的是 **`SV_Target1` (以及 `SV_Target2`, `SV_Target3` 等系列)**，那这就涉及到了图形学中一个非常强大、非常核心的高级技术：

### **MRT (Multiple Render Targets / 多重渲染目标)**

简单来说：**它允许你的片元着色器（Fragment Shader）在同一时刻，把不同的颜色输出到“多张不同的画布（纹理）”上。**

为了让你彻底弄懂，我们用一个比喻和几个实战场景来拆解。

---

### 1. 核心概念：分身术画画

在最基础的 Shader 里，你的片元着色器只返回一个 `float4`（颜色），这个颜色被画到了玩家的屏幕上。这对应的是 **`SV_Target0`**（通常简写为 `SV_Target`）。

但是，有时候我们不仅想画出最终的模样，还想**悄悄留下一些副产物**。
想象你是一个画家，面前叠着 3 张玻璃画布：
*   **`SV_Target0` (第一张画布)**：画角色的**真实长相**（发给玩家看）。
*   **`SV_Target1` (第二张画布)**：只画角色的**发光部分**（纯黑白，留给后期做泛光 Bloom 用）。
*   **`SV_Target2` (第三张画布)**：画角色的**世界空间法线**（看似花花绿绿的图，留给后期做屏幕空间反射 SSR 用）。

当你画一笔画时，你的笔能够穿透这三层玻璃，在每一层留下不同的痕迹。这就是 MRT。

---

### 2. 代码长什么样？

在 Shader 里，原本 `frag` 函数只需返回一个 `half4`。用了 MRT 后，你需要返回一个**结构体 (Struct)**：

```hlsl
// 定义 MRT 的输出结构
struct FragmentOutput 
{
    half4 MainColor : SV_Target0; // 画布 0：主画面
    half4 GlowMask  : SV_Target1; // 画布 1：发光遮罩
    half4 NormalData: SV_Target2; // 画布 2：法线数据
};

FragmentOutput frag(v2f i) 
{
    FragmentOutput o;
    
    // 1. 正常计算颜色输出给屏幕
    o.MainColor = tex2D(_MainTex, i.uv) * _Color;
    
    // 2. 提取需要发光的部分，输出到 Target1
    o.GlowMask = tex2D(_EmissionMap, i.uv);
    
    // 3. 把法线存起来，输出到 Target2
    o.NormalData = half4(i.worldNormal * 0.5 + 0.5, 1.0);
    
    return o;
}
```

---

### 3. 这些变量到底有什么用？（三大实战场景）

为什么我们要费劲存这么多图？主要有以下几个极其重要的用途：

#### 场景 A：延迟渲染 (Deferred Rendering) 🌟最经典
现代 3A 大作（和 Unity HDRP、URP 的延迟管线）几乎全部依赖这个技术。
如果在同屏有 1000 盏灯，传统的“前向渲染 (Forward)”会让显卡瞬间爆炸。
延迟渲染的思路是：在画模型的时候**先不算光照**，而是把模型的属性当作颜色的样子，统统存进 `SV_Target0 ~ SV_Target3` 里。这些产出的图叫做 **G-Buffer (几何缓冲区)**：
*   `SV_Target0`：存 BaseColor (漫反射颜色) + 粗糙度
*   `SV_Target1`：存 Specular (高光颜色) + 金属度
*   `SV_Target2`：存 World Normal (世界法线)
*   `SV_Target3`：存 Emission (自发光)

等所有模型都画完、这 4 张图都存好之后，再用这 4 张图作为原料，统一算一次满屏的光照。极其省性能！

#### 场景 B：高级后处理 (Post-Processing 遮罩)
你想做一个效果：**主角在墙后被遮挡时，显示红色的描边轮廓。（透视眼效果）**
*   你可以让主角的 Shader 输出正常颜色到 `SV_Target0`，同时输出一个纯白色的轮廓 ID 到 `SV_Target1`。
*   屏幕后处理的 Shader 拿到 `SV_Target1` 这张黑白图，就知道屏幕上哪个像素属于主角，从而针对性地画描边，而不用全屏找。

#### 场景 C：交互式草地 / 积雪压痕
*   角色的脚底或者车轮的 Shader，不往屏幕(`Target0`)画任何东西。
*   把它画进一张看不见的 RenderTexture (`Target1`) 中，作为“压痕数据”。
*   草地的 Shader 读取这张图，发现哪个像素变白了，就把那里的草压弯。

---

### 4. 配合 C# 食用更佳

单独在 Shader 里写 `SV_Target1` 是没用的，如果你不告诉显卡这张图该存到哪里，它就会因为找不到对应的内存而报错或失效。

在 Unity 脚本（C#）里，你需要准备好几张 RenderTexture，并把它们**绑定 (Bind)** 到这些 Target 上：

```csharp
// 伪代码：命令显卡把接下来的渲染输出到这几张图上
RenderTexture[] myTargets = new RenderTexture[3] { rt0, rt1, rt2 };

// 对应 Shader 里的 Target0, Target1, Target2
Graphics.SetRenderTarget(myTargets, depthBuffer);
```
*(在 URP 管线里，这通常是在 `ScriptableRenderPass` 中使用 `ConfigureTarget` 或 `Blit` API 来完成的。)*

### 总结

*   `SV_Target0`（或 `SV_Target`）：默认的主力输出，通常是画给玩家看的**最终画面**。
*   `SV_Target1` ~ `SV_Target7`：**分身输出**，用于把额外的数据（法线、深度、遮罩、材质ID）写入其他隐形的内存纹理中，供后面的特效或计算使用。现代手机和 PC 显卡通常支持同时输出 4 到 8 个 Target。
