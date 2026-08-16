

在 URP 的 `DrawRenderers` 指令中，`FilteringSettings` 和 `DrawingSettings` 堪称是**“左膀右臂”**。

如果把 `context.DrawRenderers` 比作一次**“全军出击”**：
*   **`FilteringSettings` (过滤器)**：就是**“征兵办”**。它负责制定规则，筛选出场景里**哪些物体**有资格出战（比如：只挑不透明的、只挑“皮肤”图层的）。
*   **`DrawingSettings` (绘制设置)**：就是**“战术教官”**。它负责告诉被选中的物体，你们上阵时要穿**哪套制服**（Shader Pass），并且按**什么队形**排队（排序规则）。

在编写自定义 Render Feature 时，这两个参数的设置有着极度标准的**“黄金模板”**。我们来逐一拆解：

---

### 一、 `FilteringSettings` （征兵办：筛选谁来画）

它的核心任务是两件事：**卡渲染队列（Render Queue）** 和 **卡图层（Layer Mask）**。

#### 1. 核心参数剖析
```csharp
// 构造函数签名：
public FilteringSettings(RenderQueueRange renderQueueRange, int layerMask = -1, uint renderingLayerMask = 4294967295);
```

*   **参数 1：`RenderQueueRange` (渲染队列 - 极其重要)**
    *   `RenderQueueRange.opaque`：只抓取不透明物体（Queue 0 ~ 2500）。**（画 3D 模型 99% 选这个）**
    *   `RenderQueueRange.transparent`：只抓取半透明物体（Queue 2501 ~ 5000）。（画玻璃、水面、粒子选这个）
    *   `RenderQueueRange.all`：全都要。
*   **参数 2：`layerMask` (物理图层 - 极其重要)**
    *   对应 Unity Editor 里右上角的 Layer（比如 Default, Water, UI）。
    *   如果你写 `-1`，代表 `Everything`（什么图层都画）。
    *   在你的 SSS 皮肤管线里，通常会在 C# 面板暴露一个 `public LayerMask skinLayer`，然后把这个变量传进来，这样就**只画带“皮肤”标签的模型**。
*   **参数 3：`renderingLayerMask` (灯光/贴花图层 - 进阶)**
    *   这是 URP 的高级功能，用来控制哪些物体能被特定的光照亮或被特定的贴花印上。平时不用管，默认值即可。

#### 2. 标准创建代码：
```csharp
// 只要不透明物体，且只画我们指定的 skinLayerMask 图层
FilteringSettings filterSettings = new FilteringSettings(RenderQueueRange.opaque, skinLayerMask);
```

---

### 二、 `DrawingSettings` （战术教官：怎么画、怎么排队）

它的配置相对复杂一点，但在 `ScriptableRenderPass` 基类里，Unity 官方贴心地提供了一个叫 **`CreateDrawingSettings`** 的魔法函数，帮我们省去了大量底层设置。

#### 1. 核心参数剖析

要创建一个 `DrawingSettings`，你需要准备两个最核心的材料：

*   **材料 A：`ShaderTagId` (Shader 的身份证)**
    *   **原理**：你打开任何一个 Shader 代码，找到 `Pass` 块，里面通常有一句 `Tags { "LightMode" = "UniversalForward" }`。这里的 `UniversalForward` 就是身份证号。
    *   **作用**：告诉 GPU：“等下画这些模型时，别管它们本来长什么样，**强行去它们身上的 Shader 里，找到名字叫 `XXX` 的那个 Pass 来执行！**”
    *   *作者的 SSS 案例*：他用的是 `new ShaderTagId("SkinDiffuse")`。这意味着所有的皮肤材质里，必须写了一个 `LightMode = SkinDiffuse` 的 Pass，否则画出来就是透明的（找不到代码）。

*   **材料 B：`SortingCriteria` (排队/排序规则)**
    *   **原理**：3D 模型画的先后顺序，直接决定了性能和画面正确性。
    *   `SortingCriteria.CommonOpaque` (不透明物体排序)：**从前向后画（Front-to-Back）**。这是为了性能！前面的物体先画，后面被挡住的物体在 GPU 深度测试时就会直接被丢弃（Early-Z），省下大量算力。
    *   `SortingCriteria.CommonTransparent` (半透明物体排序)：**从后向前画（Back-to-Front）**。画玻璃和水面必须从远往近叠加上去，否则 Alpha 混合绝对会出错！

#### 2. 标准创建代码：
```csharp
// 1. 指定要执行 Shader 里的哪个 Pass？
ShaderTagId shaderTagId = new ShaderTagId("UniversalForward");

// 2. 使用 URP 基类提供的魔法函数，一键生成 DrawingSettings
// (它会自动帮你把相机的 sorting 规则、灯光数据等脏活累活全配置好)
DrawingSettings drawSettings = CreateDrawingSettings(
    shaderTagId, 
    ref renderingData, 
    SortingCriteria.CommonOpaque
);

// 3. (进阶高级操作) 如果你想让这个 Pass 同时兼容两个不同的 LightMode：
// 比如既画 "UniversalForward"，又画 "SRPDefaultUnlit"
drawSettings.SetShaderPassName(1, new ShaderTagId("SRPDefaultUnlit")); 
```

---

### 三、 终极实战：把它们拼在一起！

现在，假设我们要写一个 Pass：**在场景中，把所有属于 `Enemy` 图层的不透明怪物，使用它们 Shader 中叫做 `MyOutline` 的 Pass，渲染成一堆红色的描边轮廓。**

这段标准的 `Execute` 函数代码如下，请欣赏这份属于管线架构师的极致优雅：

```csharp
public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
{
    // 1. 获取命令清单
    CommandBuffer cmd = CommandBufferPool.Get("Draw Enemy Outlines");

    // ==========================================
    // 第一步：配置“征兵办” (Filtering)
    // ==========================================
    // 规则：只要不透明队列，只要 Enemy 图层 (假设在外面已经配好了 m_EnemyLayerMask)
    FilteringSettings filterSettings = new FilteringSettings(RenderQueueRange.opaque, m_EnemyLayerMask);

    // ==========================================
    // 第二步：配置“战术教官” (Drawing)
    // ==========================================
    // 规则：强行调用 Shader 里的 "MyOutline" Pass，且按不透明物体从前向后排队
    ShaderTagId tagId = new ShaderTagId("MyOutline");
    DrawingSettings drawSettings = CreateDrawingSettings(tagId, ref renderingData, SortingCriteria.CommonOpaque);

    // 【可选拓展】：开启物体的全局光照探针支持 (如果描边不需要光照，可以不写)
    // drawSettings.perObjectData = PerObjectData.LightProbes;

    // ==========================================
    // 第三步：发兵出征！(执行绘制)
    // ==========================================
    // 先把 cmd 里的准备工作（比如绑画板）提交了
    context.ExecuteCommandBuffer(cmd);
    cmd.Clear();

    // 核心大招：按上面的规则，画 3D 模型！
    // 传入“幸存者名单”(cullResults)，传入教官(draw)，传入征兵办(filter)
    context.DrawRenderers(renderingData.cullResults, ref drawSettings, ref filterSettings);

    // 收尾
    context.ExecuteCommandBuffer(cmd);
    CommandBufferPool.Release(cmd);
}
```

**总结一下：**
当你掌握了 `FilteringSettings` 和 `DrawingSettings`，你就掌握了对场景中万事万物的**“生杀大权”和“换装大权”**。你可以把场景剥离出无数个图层，针对不同的物体使用截然不同的渲染策略，这正是《原神》这类游戏实现复杂角色二次元渲染（衣服、头发、脸部分层渲染）的最底层基石！
