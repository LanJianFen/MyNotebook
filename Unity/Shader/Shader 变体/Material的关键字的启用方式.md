

这是一个非常好的问题！答案是：**并不是。**

虽然用 C# 动态启用关键字在程序逻辑中非常常见，但实际上在 Unity 中，Shader 关键字的启用主要有 **三种完全不同的途径**。

我们可以按照“谁在控制它”分为以下三类：

---

### 途径一：通过材质面板勾选（美术/TA 常用，静态启用）

这是游戏开发中最常见的关键字启用方式。美术同学不需要写任何代码，只需要在材质（Material）面板上打个勾，Unity 底层就会自动启用对应的关键字，并把这个状态保存在 `.mat` 文件里。

**怎么实现的？**
这依赖于 ShaderLab 属性面板的特殊标签 `[Toggle]` 或 `[KeywordEnum]`。

**代码示例：**
```glsl
Properties
{
    // 在面板上生成一个打勾的选项。如果勾选，自动激活 USE_NORMAL_MAP 宏
    [Toggle(USE_NORMAL_MAP)] _UseNormal ("使用法线贴图", Float) = 0
    _NormalMap ("法线贴图", 2D) = "bump" {}
}
SubShader
{
    Pass
    {
        CGPROGRAM
        // 配合 shader_feature 使用
        #pragma shader_feature USE_NORMAL_MAP
        // ...
        ENDCG
    }
}
```
**特点：** 打包时，Unity 会读取场景中所有材质球的这个打勾状态。如果勾了，就打包该变体；没勾，就剔除。这属于**静态配置**。

---

### 途径二：通过 C# 代码动态控制（程序常用，运行时启用）

这就是我们上一轮聊到的方式，通常用于游戏运行时的逻辑状态切换（比如切换天气、切换画质、角色受击闪白等）。

这里需要特别注意，C# 启用关键字分为**全局**和**局部（单体）**两种级别：

**1. 全局级别（影响所有使用了该Shader的材质）**
*   **API:** `Shader.EnableKeyword("MACRO_NAME")` / `Shader.DisableKeyword("MACRO_NAME")`
*   **适用场景:** 全局天气、昼夜交替、全局画质设置。

**2. 局部/材质级别（只影响特定的那个材质球实例）**
*   **API:** `material.EnableKeyword("MACRO_NAME")` / `material.DisableKeyword("MACRO_NAME")`
*   **适用场景:** 比如有10个怪物，只有被玩家打中的那1个怪物身体变红发光。此时你只能获取那个特定怪物的 material，然后 `EnableKeyword`。

---

### 途径三：Unity 底层引擎自动接管（黑盒机制，最容易被忽视）

有大量极其重要的关键字，**既不需要美术打勾，也不需要程序写 C# 代码**，而是 Unity 引擎的渲染管线（C++底层）根据当前场景的状况**自动激活**的！

**典型的自动关键字：**
1.  **光照类型 (Lighting):** 
    当 Unity 准备渲染一个受光照影响的物体时，引擎会自动判断当前是什么灯光在照它。
    *   如果是平行光，自动启用 `DIRECTIONAL`。
    *   如果是点光源，自动启用 `POINT`。
2.  **阴影 (Shadows):**
    如果场景里的灯光开启了阴影，并且物体的 Mesh Renderer 允许接收阴影。
    *   引擎自动启用 `SHADOWS_SCREEN` 或 `SHADOWS_DEPTH`。
3.  **光照贴图 (Lightmapping):**
    如果这个物体被勾选了 Static 并且烘焙了光照贴图。
    *   引擎自动启用 `LIGHTMAP_ON` 宏。
4.  **雾效 (Fog):**
    你在 Lighting 面板（Window -> Rendering -> Lighting）里勾选了 Fog。
    *   引擎会根据你选的雾效类型，自动启用 `FOG_LINEAR` 或 `FOG_EXP2`。

**我们怎么配合引擎？**
为了让 Shader 能接收到这些引擎自动发送的关键字，我们通常不需要手写一个个 `#pragma multi_compile`，而是使用 Unity 提供好的**快捷打包指令**，比如最著名的：

```glsl
// 这一句话，等同于引入了平行光、阴影、光照贴图等十几个宏的组合！
#pragma multi_compile_fwdbase 
```

---

### 总结归纳

| 启用途径 | 控制者 | 使用场景 | 搭配的指令 |
| :--- | :--- | :--- | :--- |
| **材质面板勾选** | 美术 / 关卡设计 | 设置静态材质属性（开关法线、高光、发光） | `[Toggle]` + `shader_feature` |
| **C# 代码控制** | 程序员 | 运行时动态逻辑（画质切换、天气、受击特效） | C# API + `multi_compile` |
| **Unity引擎自动** | 渲染管线 (底层) | 系统级渲染特性（光照、阴影、雾效、烘焙） | `multi_compile_fwdbase` 等内置指令 |
