

在 URP 的自定义渲染中，`FilteringSettings` 就像是一个**“筛子”**。

当你的自定义 Pass 在场景里大喊 *“谁有 MyCustomOutline 这个标签！”* 时，如果场景里有 100 个物体都有这个标签，`FilteringSettings` 就会帮你把这 100 个物体再过滤一遍：**“我只要不透明的（RenderQueue），并且只画 Player 层的（LayerMask）”**。

作为开发工程师，最标准的做法是**不要把这些参数写死在代码里，而是把它们暴露到 Renderer Feature 的 Inspector 面板上**，让美术或策划自己去调。

下面我手把手教你如何设置，并提供一个可以直接抄的“标准工业级模板”。

---

### 核心概念拆解

`FilteringSettings` 的构造函数主要接收两个非常关键的参数：

1.  **`RenderQueueRange`（渲染队列范围）：**
    决定是画不透明物体还是半透明物体。
    *   `RenderQueueRange.opaque`：只画不透明物体（Queue 0 ~ 2500）。
    *   `RenderQueueRange.transparent`：只画半透明物体（Queue 2501 ~ 5000）。
    *   `RenderQueueRange.all`：全画。
2.  **`LayerMask`（层级掩码）：**
    就是 Unity 场景中 GameObject 挂载的 Layer（比如 Default, TransparentFX, Water, UI 等）。默认值是 `-1`（对应 Inspector 里的 `Everything`）。

---

### 标准代码实现模板（极其推荐这种写法）

我们分两步：先在 `Feature` 中把面板暴露出来，再传给 `Pass` 去干活。

#### 第一步：编写完整的 C# 脚本

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class MyCustomOutlineFeature : ScriptableRendererFeature
{
    // ==========================================
    // 1. 定义一个配置类，暴露到 Inspector 面板上
    // ==========================================
    [System.Serializable]
    public class CustomSettings
    {
        [Tooltip("这个 Pass 在哪个阶段执行？")]
        public RenderPassEvent passEvent = RenderPassEvent.AfterRenderingOpaques;
        
        [Tooltip("Shader 中的 LightMode 名字")]
        public string lightModeName = "MyCustomOutline";
        
        [Tooltip("过滤条件：画不透明还是半透明？")]
        public RenderQueueType queueType = RenderQueueType.Opaque;
        
        [Tooltip("过滤条件：只画哪些 Layer 的物体？")]
        public LayerMask layerMask = -1; // -1 表示 Everything
    }

    // 方便给下拉菜单用的枚举
    public enum RenderQueueType { Opaque, Transparent, All }

    // 在面板上显示这个 settings
    public CustomSettings settings = new CustomSettings();

    // ==========================================
    // 2. 定义我们的自定义 Pass
    // ==========================================
    class CustomOutlinePass : ScriptableRenderPass
    {
        ShaderTagId m_ShaderTagId;
        FilteringSettings m_FilteringSettings; // 核心：过滤设置

        // 构造函数，接收从 Feature 传过来的配置
        public CustomOutlinePass(CustomSettings settings)
        {
            this.renderPassEvent = settings.passEvent;
            m_ShaderTagId = new ShaderTagId(settings.lightModeName);

            // 转换我们定义的枚举为 URP 底层的 RenderQueueRange
            RenderQueueRange queueRange = RenderQueueRange.opaque;
            if (settings.queueType == RenderQueueType.Transparent)
                queueRange = RenderQueueRange.transparent;
            else if (settings.queueType == RenderQueueType.All)
                queueRange = RenderQueueRange.all;

            // ⚠️ 在这里初始化 FilteringSettings！
            // 传入 QueueRange 和 LayerMask
            m_FilteringSettings = new FilteringSettings(queueRange, settings.layerMask);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            CommandBuffer cmd = CommandBufferPool.Get("Draw Custom Pass");

            DrawingSettings drawingSettings = CreateDrawingSettings(
                m_ShaderTagId, 
                ref renderingData, 
                SortingCriteria.CommonOpaque
            );

            // 执行绘制，把我们的 m_FilteringSettings 塞进去！
            context.DrawRenderers(renderingData.cullResults, ref drawingSettings, ref m_FilteringSettings);

            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }

    // ==========================================
    // 3. Feature 的生命周期
    // ==========================================
    CustomOutlinePass m_ScriptablePass;

    public override void Create()
    {
        // 每次面板参数改变，都会重新 new 一个 Pass，并把面板上的 settings 传进去
        m_ScriptablePass = new CustomOutlinePass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (renderingData.cameraData.cameraType != CameraType.Reflection)
        {
            renderer.EnqueuePass(m_ScriptablePass);
        }
    }
}
```

---

#### 第二步：在 Unity 编辑器中配置

你写完这段代码后，回到 Unity 编辑器，点开你的 `Renderer Data`（比如 `URP-HighQuality-Renderer`）。

此时你会发现，你添加的 `My Custom Outline Feature` 下面多了很多友好的设置选项：
*   **Pass Event:** (选择执行时机)
*   **Light Mode Name:** MyCustomOutline
*   **Queue Type:** `Opaque` (下拉菜单)
*   **Layer Mask:** `Nothing`, `Everything`, `Default`, `Player`, `Enemy` ... (下拉菜单)

**使用场景举例：**
假设你想做一个“敌人被墙遮挡时显示透视红边”的效果。
你可以把 `Layer Mask` 改成只勾选 `Enemy` 层。那么这个 Pass 在执行时，就会**无视场景里的石头、树木、玩家**，只去寻找 `Enemy` 层的物体，并且只渲染这些物体身上的 `MyCustomOutline` 这个 Shader Pass。

---

### 💡 给开发者的进阶 Tips（非常重要）

在实际商业项目中，使用 GameObject 的 `LayerMask` 做渲染过滤会遇到一个**痛点**：
*   GameObject 的 `Layer` 只有 32 个，而且通常被**物理系统（碰撞、射线检测）**高度占用！
*   美术说：“我想给这个特殊的宝箱加个描边特效”，如果你为了这个特效专门去占用一个 GameObject Layer，主程会打人的（因为这可能会影响物理射线检测逻辑）。

**URP 的高级解法：使用 `Rendering Layer Mask`**

从 URP 12（Unity 2021.2）开始，Unity 引入了专门用于渲染过滤的 `Rendering Layer Mask`。
它允许你在不改变物体 GameObject Layer（不影响物理）的情况下，单独给物体打上“渲染标签”。

如果你想用这种高级特性，只需修改上面代码中的一行：
```csharp
// 构造函数中：不仅可以传 GameObject Layer，还可以传 Rendering Layer！
// 比如只渲染 Rendering Layer 为 2 (即 1 << 1) 的物体
uint myRenderingLayerMask = 2; 
m_FilteringSettings = new FilteringSettings(queueRange, settings.layerMask, myRenderingLayerMask);
```
（在 Mesh Renderer 组件上，展开 `Additional Settings` 就能看到 `Rendering Layer Mask` 的设置）。这也是目前大型项目做特定特效筛选的最优解！
