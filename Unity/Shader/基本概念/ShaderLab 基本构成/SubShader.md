

作为 Unity 工程师，你可以把 **SubShader** 理解为 Shader 针对不同硬件/管线的 **“兼容性方案”** 或 **“备胎列表”**。

如果说 Shader 文件是**“一个项目需求”**，那么 SubShader 就是**“针对不同能力员工（显卡）的执行方案”**。

---

### 1. 核心逻辑：Top-Down Selection (自顶向下选择)

在一个 `.shader` 文件中，可以包含 **多个** `SubShader` 代码块。

当 Unity 准备渲染一个物体时，它会拿着这个 Shader，按照代码书写顺序**从上往下**检查每一个 SubShader：

1.  **检查 SubShader A：** 嘿，当前显卡支持这儿写的指令吗？（比如 Tessellation，或者特定的纹理数量，或者 Shader Model 版本）。当前渲染管线匹配吗？
    *   **如果支持：** 选中它！**运行 SubShader A，忽略后面所有的 SubShader。**
    *   **如果不支持：** 跳过，检查下一个。
2.  **检查 SubShader B：** 这个支持吗？
    *   ...
3.  **都不支持？** 使用最后的 `FallBack` 指定的备用 Shader（通常是紫红色的那个 Error Shader 或者最简单的 Diffuse）。

**结论：** 它是**互斥**的。针对某一次渲染，**永远只有一个 SubShader 会被激活。**

---

### 2. 代码结构

```c
Shader "MyGame/SuperShader"
{
    Properties { ... }

    // --- 方案 1：给 PC/主机用的豪华版 ---
    SubShader 
    {
        Tags { "RenderPipeline" = "HDRP" } // 只在 HDRP 下运行
        LOD 300 // 细节等级高
        
        Pass { ... } // 复杂的 PBR 计算
        Pass { ... } // 复杂的体积光
    }

    // --- 方案 2：给中低端手机用的简化版 ---
    SubShader 
    {
        Tags { "RenderPipeline" = "UniversalPipeline" } // URP
        LOD 100 
        
        Pass { ... } // 简单的 Blinn-Phong 计算，没有体积光
    }

    // --- 方案 3：甚至更旧的设备 ---
    SubShader 
    { 
       // 最简陋的写法
    }

    FallBack "Diffuse" // 实在不行就用 Unity 自带的 Diffuse
}
```

---

### 3. SubShader 里装的是什么？

SubShader 是一个容器，它主要包裹了两样东西：

1.  **设置 (Setup/State):**
    这些设置会应用到内部所有的 Pass 上。
    *   **Tags:** 比如 `Queue` (渲染顺序/透明物体), `RenderType` (用于替换渲染), `RenderPipeline` (标记它是 URP 还是 HDRP 用)。
    *   **LOD:** 手动控制 Shader 的质量等级。
    *   **渲染状态:** `Cull` (剔除), `ZWrite` (深度写入), `Blend` (混合模式)。(注：虽然 Pass 里也能写，但在 SubShader 写意味着“全局默认值”)。

2.  **Pass (绘制步骤):**
    真正干活的地方。一个 SubShader 可以包含一个或多个 Pass。

---

### 4. 工程师常见的应用场景

理解 SubShader 对以下几个场景至关重要：

#### A. 多管线兼容 (URP vs Built-in)
如果你开发的 Shader 需要同时支持 URP 和 内置管线，你不能把两种代码混在一个 Pass 里。
你需要写 **两个 SubShader**：
*   第一个 Tag 设为 `"RenderPipeline" = "UniversalPipeline"` (里面写 HLSL)。
*   第二个 Tag 留空或设为 Built-in (里面写 CGPROGRAM)。
Unity 会根据当前项目的管线设置自动挑选正确的那个运行。

#### B. 性能分级 (LOD System)
你可以写三个 block：
*   `SubShader (LOD 600)`: 极其复杂的视差映射。
*   `SubShader (LOD 400)`: 普通法线贴图。
*   `SubShader (LOD 200)`: 只有颜色。

在 C# 代码中调用 `Shader.globalMaximumLOD = 300;`，由于 600 和 400 高于限制，Unity 会自动降级选择 **LOD 200** 的 SubShader。这是做画质设置（高/中/低）的利器。

#### C. 替换渲染 (Replacement Shader)
在做某些特效（如热成像仪、像《守望先锋》的透视高亮）时，摄像机可以使用 `Camera.RenderWithShader`。
这会忽略物体原本的 Shader 逻辑，利用 **SubShader 中的 Tags** (`RenderType`) 来寻找匹配，用特效 Shader 统一渲染场景。

---

### 总结：SubShader vs Pass

这是初学者最容易混淆的：

*   **SubShader（备胎选择）：** 是 **Choice (选择题)**。Unity 从列表中**选一个**最适合当前硬件/管线的方案。
*   **Pass（图层步骤）：** 是 **Action (执行题)**。被选中的 SubShader 里的 Pass 会被**依次执行**（除非它是多光源的 ForwardAddPass）。

**一句话：一个 Shader 有多套 SubShader（为了兼容），一套 SubShader 有多个 Pass（为了画完效果）。**
