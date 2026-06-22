

这是一个非常核心且进阶的问题！一旦你开始研究这三者的配合，说明你已经真正迈入了 **URP（通用渲染管线）底层自定义渲染** 的大门。

在 URP 中，如果你想做一些特殊效果（比如：**角色被遮挡时的透视X光效果、全屏描边、水面反射、残影**），单靠写 Shader 是做不到的，必须靠 `ForwardRenderer`（现在叫 `ScriptableRenderer`）、`Custom Pass` 和 `LightMode` 配合来完成。

为了让你快速理解，我们先建立一个**形象的思维模型（流水线工厂）**：

*   **`ForwardRenderer`（工厂流水线）：** URP 的主渲染器，它规定了渲染的先后顺序（先画不透明物体 -> 再画天空盒 -> 再画半透明物体 -> 后处理）。
*   **`Custom Pass`（自定义工人）：** 你自己写的一个 C# 脚本。你把这个工人安插在流水线的某个特定位置（比如“画完半透明物体之后”），让他去执行特定的任务。
*   **`LightMode`（货物标签）：** 写在 Shader 里的一个 Tag。工人（Pass）在干活时，只会在场景里寻找贴着特定 `LightMode` 标签的物体进行加工。

下面我们一步步来看在工程中**如何实操使用它们**。

---

### 第一步：千万不要直接修改 `ForwardRenderer.cs` 源码！

早期 SRP 测试版时，很多人直接改源码。现在 Unity 提供了**插件式**的扩展方法，也就是 `ScriptableRendererFeature`（渲染特性）。
我们要做的，是写一个 Feature 挂载到 URP 的 Renderer Data 资产上，由这个 Feature 将我们的 `Custom Pass` 注入到 `ForwardRenderer` 中。

### 第二步：Shader 中的 `LightMode` 怎么写？

假设我们要实现一个需求：**把场景里特定角色的外发光轮廓单独画出来**。

你需要在你的 Shader 中新增一个 Pass，并且起一个独一无二的 `LightMode` 名字：

```glsl
Shader "Custom/MyCharacter"
{
    SubShader
    {
        // 正常的渲染 Pass (被 URP 默认渲染器调用)
        Pass
        {
            Tags { "LightMode" = "UniversalForward" } 
            // ... 正常光照代码
        }

        // 我们的自定义 Pass！专门用来画特效/轮廓
        Pass
        {
            // 这个名字你可以随便起，但 C# 里要和它完全对应！
            Tags { "LightMode" = "MyCustomOutline" } 
            
            // ... 画轮廓的代码 (比如顶点沿法线外扩并输出纯色)
        }
    }
}
```

---

### 第三步：写 C# 脚本 (`ScriptableRenderPass` + `Feature`)

我们需要写一个 C# 脚本，定义我们的“工人”（Pass）和“工位”（Feature）。

创建一个 C# 脚本 `MyCustomOutlineFeature.cs`，代码结构通常如下：

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

// 1. 定义工位 (Renderer Feature)
public class MyCustomOutlineFeature : ScriptableRendererFeature
{
    // 2. 定义工人 (Custom Pass)
    class CustomOutlinePass : ScriptableRenderPass
    {
        // 相当于你定义的那个 LightMode 标签
        ShaderTagId shaderTagId = new ShaderTagId("MyCustomOutline");
        FilteringSettings filteringSettings;

        public CustomOutlinePass()
        {
            // 设置过滤条件：只渲染不透明物体，或者特定 Layer 的物体
            filteringSettings = new FilteringSettings(RenderQueueRange.opaque);
        }

        // 核心干活的方法！每帧都会执行
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            // 从命令池借用一个 CommandBuffer (用于在 FrameDebugger 里显示名字，方便调试)
            CommandBuffer cmd = CommandBufferPool.Get("Draw My Custom Outline");

            // 创建绘制设置，告诉管线：我要画场景里那些带有 "MyCustomOutline" 标签的 Shader Pass！
            DrawingSettings drawingSettings = CreateDrawingSettings(
                shaderTagId, 
                ref renderingData, 
                SortingCriteria.CommonOpaque // 按不透明物体顺序排序
            );

            // 让 context 真正去执行绘制
            context.DrawRenderers(renderingData.cullResults, ref drawingSettings, ref filteringSettings);

            // 执行并释放 CommandBuffer
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }

    // ------------------- Feature 层的逻辑 -------------------
    CustomOutlinePass m_ScriptablePass;

    // 初始化时调用 (比如修改了面板参数)
    public override void Create()
    {
        m_ScriptablePass = new CustomOutlinePass();
        
        // 决定这个 Pass 在流水线的哪个阶段执行？
        // 这里设置为：在渲染完所有不透明物体之后执行
        m_ScriptablePass.renderPassEvent = RenderPassEvent.AfterRenderingOpaques; 
    }

    // 将 Pass 注入到 ForwardRenderer 中
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        // 只有当需要渲染时（比如不是在反射探针里），才加入管线
        if (renderingData.cameraData.cameraType != CameraType.Reflection)
        {
            renderer.EnqueuePass(m_ScriptablePass);
        }
    }
}
```

---

### 第四步：在 Unity 编辑器里组装起来

1.  找到你项目中正在使用的 **URP Renderer Data** 资产（通常叫 `UniversalRendererData`，是个蓝色的图标）。
2.  拉到最下面，点击 **"Add Renderer Feature"**。
3.  在弹出的列表里，你会看到你刚才写的 `MyCustomOutlineFeature`，添加它。
4.  将带有刚才那个 Shader (`Custom/MyCharacter`) 的材质赋给场景里的物体。

**此时发生的化学反应：**
1. 每一帧，URP 的 `ForwardRenderer` 开始运行。
2. 它按照原本的逻辑，用 `UniversalForward` 标签把角色正常画出来了。
3. 当管线执行到 `AfterRenderingOpaques` 这个节点时，触发了你的 `CustomOutlinePass`。
4. 你的 Pass 拿着喇叭在场景里喊：“谁的身上有 `MyCustomOutline` 这个标签的 Pass？现在由我来画你们！”
5. 物体的 Shader 响应了，被再次绘制了一遍（执行了外发光轮廓的逻辑）。

---

### 工程师必知的进阶 Tips：

1.  **善用 Frame Debugger（帧调试器）：**
    这是写 Custom Pass 必备的神器！打开 `Window -> Analysis -> Frame Debugger`，开启后，你能清晰地看到 `ForwardRenderer` 每一步画了什么。你能看到你自定义的 `Draw My Custom Outline` 出现在事件树里，如果不生效，这里是排查问题的第一现场。
2.  **不仅能画物体，还能做后处理（Blit）：**
    上面的例子是 `DrawRenderers`（把场景物体用特定的 Pass 再画一遍）。如果你需要做屏幕雨滴、全屏泛光等后处理，你可以在 `Execute` 里使用 `cmd.Blit(source, destination, material)`，这完全是同一个技术栈。
3.  **LightMode 是可以被覆盖（Override）的：**
    除了写死在 Shader 里，你甚至可以在 Custom Pass 中通过 `drawingSettings.overrideMaterial` 强行用一个指定的材质去渲染场景里的所有物体（比如用来生成场景深度图、或者法线全屏扫描特效），这也是非常常见的高级技巧。

总结来说，`ForwardRenderer + Feature + Custom Pass + LightMode` 是 URP 高级渲染的核心四件套。掌握它们，你就能跳出基础的“贴图+材质”思维，站在“整个渲染管线”的高度去开发商业级的游戏特效。
