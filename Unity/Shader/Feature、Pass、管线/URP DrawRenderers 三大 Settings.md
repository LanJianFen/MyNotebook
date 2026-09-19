# URP DrawRenderers 三大 Settings

`context.DrawRenderers()` 是 URP RenderPass 里最常用的绘制入口，它需要三个 Settings 结构体协作：

```csharp
context.DrawRenderers(
    cullResults,
    ref drawingSettings,     // 画什么 Pass、怎么画
    ref filteringSettings,   // 画哪些 Renderer
    ref renderStateBlock     // 覆盖 render state（可选）
);
```

配合排序：

```csharp
SortingSettings sorting = new SortingSettings(camera) { criteria = ... };
DrawingSettings drawing = new DrawingSettings(tagId, sorting);
```

本文档按 `SortingSettings` → `DrawingSettings` → `FilteringSettings` 顺序介绍。

---

## 1. SortingSettings

控制 URP 怎么排序 Renderer。**值类型（struct）**。

### 构造函数

```csharp
// 推荐：传相机（排序需要相机位置/矩阵）
public SortingSettings(Camera camera);

// 不推荐：无参（排序结果不可靠）
public SortingSettings();
```

### 核心属性

#### `criteria`（最常用）⭐

告诉 URP 用什么规则排序。

```csharp
sorting.criteria = SortingCriteria.CommonOpaque;
```

**`SortingCriteria` 枚举**（可组合位标志）：

| 值 | 作用 |
|---|---|
| `None` | 不排序（按提交顺序） |
| `SortingLayer` | 按 SortingLayer 排 |
| `RenderQueue` | 按 Material 的 RenderQueue 排 |
| `BackToFront` | 从远到近（透明用） |
| `QuantizedFrontToBack` | 从近到远（分段量化，快） |
| `OptimizeStateChanges` | 按 material 分组，减少状态切换 |
| `CanvasOrder` | UI 用 |
| `RendererPriority` | 按 Renderer.rendererPriority 属性 |

**预设组合**：

```csharp
SortingCriteria.CommonOpaque      = SortingLayer | RenderQueue
                                  | QuantizedFrontToBack | OptimizeStateChanges

SortingCriteria.CommonTransparent = SortingLayer | RenderQueue
                                  | BackToFront | OptimizeStateChanges

SortingCriteria.Default = CommonTransparent
```

**如何选择**：

| 场景 | 推荐 |
|---|---|
| 不透明物体主渲染 | `CommonOpaque`（近到远，早 z-cull 剔除远物体） |
| 普通半透明物体 | `CommonTransparent`（远到近，正确 alpha blend） |
| 加性混合的效果（fog / bloom / god ray）| `CommonOpaque`（配合 stencil 或 early-z）|
| 需要精确控制 | 手动组合位 |
| 不在意顺序 | `None`（最快，但结果不可控） |

#### `customAxis`

自定义"距离"计算方向。

```csharp
sorting.customAxis = Vector3.up;   // 按 Y 轴排序（顶视角游戏）
```

**默认**：相机 forward。

**罕用**。

#### `worldToCameraMatrix` / `distanceMetric`

构造时传相机会自动设置，一般不手动改。

### 使用模式

```csharp
// 最常见：传相机 + 只覆盖 criteria
SortingSettings sorting = new SortingSettings(camera)
{
    criteria = SortingCriteria.CommonOpaque
};
```

### 属性使用频率

| 属性 | 频率 | 说明 |
|---|---|---|
| 构造函数传 camera | ⭐⭐⭐⭐⭐ | 必传 |
| `criteria` | ⭐⭐⭐⭐⭐ | 99% 只改这个 |
| `customAxis` | ⭐ | 特殊排序需求 |
| `distanceMetric` | ⭐ | 几乎不用 |
| `worldToCameraMatrix` | ⭐ | 几乎不用 |

---

## 2. DrawingSettings

告诉 URP "画哪个 shader Pass、怎么画"。**值类型（struct）**。

### 构造函数

```csharp
// 单 Pass（最常用）
public DrawingSettings(ShaderTagId shaderPassName, SortingSettings sortingSettings);

// 无参构造
public DrawingSettings();
```

**注意**：构造函数只接受一个 `ShaderTagId`，多 tag 用 `SetShaderPassName` 方法。

### 核心属性

#### `sortingSettings`

构造时传入的 SortingSettings，之后可通过属性再改。

#### `perObjectData`（性能敏感）⭐

告诉 URP 每个 Renderer 需要哪些"附加数据"。

```csharp
drawing.perObjectData = PerObjectData.None;
```

**`PerObjectData` 枚举**（可组合位标志）：

| 值 | 传的数据 | 用途 |
|---|---|---|
| `None` | 什么都不传 | 自发光、纯颜色物体 |
| `LightProbe` | Light probe 系数 | 静态间接光 |
| `ReflectionProbes` | 反射探针 cubemap | 金属反射 |
| `Lightmaps` | Lightmap UV + 贴图 | 静态光照 |
| `LightData` | 光源信息 | 动态光照 |
| `LightIndices` | 光源索引数组 | forward+ 光源循环 |
| `MotionVectors` | 运动向量 | TAA / motion blur |
| `ShadowMask` | Mixed 光 shadow mask | 混合光照 |
| `OcclusionProbe` | Occlusion probe | 静态遮蔽 |

**性能影响**：每加一个 flag，URP 每个 Renderer 就要多传一份数据（CPU 端 `SetPerObjectData` 开销 + GPU uniform 缓冲区更大）。

**原则**：只勾选真正需要的。

#### `enableInstancing`（性能优化）⭐

允许 GPU Instancing：

```csharp
drawing.enableInstancing = true;
```

**条件**：
- Material 勾选 "Enable GPU Instancing"
- Shader 支持 instancing（`#pragma multi_compile_instancing`）
- 多个物体用同一个 material

**效果**：同 material 的多个物体合并成 1 个 drawcall。**默认 false，一般都要开**。

#### `enableDynamicBatching`

老技术，被 SRP Batcher 取代。**URP 里通常关掉**（默认 false）。

#### `mainLightIndex`

指定主光源索引。默认 `-1`（URP 自动选），几乎不改。

#### `overrideMaterial` / `overrideShader`

**忽略 Renderer 上的原 material/shader，强制用指定的**：

```csharp
drawing.overrideMaterial = myMaterial;
drawing.overrideMaterialPassIndex = 0;

// 或
drawing.overrideShader = myShader;
drawing.overrideShaderPassIndex = 0;
```

**典型用途**：
- **Depth prepass**：所有物体用一个统一的 depth-only shader
- **Outline pass**：所有物体用 outline material 再画一遍
- **Normal prepass**：所有物体用统一 normal shader

**注意**：override 后原 material 的参数失效。

### 多 Pass 匹配：`SetShaderPassName`

构造函数只接受一个 tag，多 tag 用这个方法：

```csharp
var drawing = new DrawingSettings(new ShaderTagId("Pass1"), sorting);
drawing.SetShaderPassName(1, new ShaderTagId("Pass2"));
drawing.SetShaderPassName(2, new ShaderTagId("Pass3"));
```

- 第一个参数是 index（0-15）
- 最多支持 16 个 tag
- 匹配到任何一个 tag 的 Pass 都会被画

**极其罕用**。

### 属性使用频率

| 属性 | 频率 | 说明 |
|---|---|---|
| 构造函数传 tagId + sorting | ⭐⭐⭐⭐⭐ | 必传 |
| `perObjectData` | ⭐⭐⭐⭐⭐ | 每次都要设 |
| `enableInstancing` | ⭐⭐⭐⭐ | 一般都开 |
| `enableDynamicBatching` | ⭐ | 一般不用 |
| `mainLightIndex` | ⭐ | 几乎不改 |
| `overrideMaterial/Shader` | ⭐⭐ | 特殊 Pass（如 depth prepass） |
| `SetShaderPassName` | ⭐ | 多 tag 场景（罕见） |

### 典型模式

**普通 forward 渲染**：
```csharp
var drawing = new DrawingSettings(_forwardTagId, sorting)
{
    perObjectData    = PerObjectData.LightProbe
                     | PerObjectData.ReflectionProbes
                     | PerObjectData.Lightmaps,
    enableInstancing = true,
};
```

**自发光效果（fog / bloom / decal）**：
```csharp
var drawing = new DrawingSettings(_myTagId, sorting)
{
    perObjectData    = PerObjectData.None,
    enableInstancing = true,
};
```

**Depth prepass（override shader）**：
```csharp
var drawing = new DrawingSettings(_depthTagId, sorting)
{
    perObjectData           = PerObjectData.None,
    enableInstancing        = true,
    overrideShader          = _depthOnlyShader,
    overrideShaderPassIndex = 0,
};
```

**Outline（覆盖 material）**：
```csharp
var drawing = new DrawingSettings(_defaultTagId, sorting)
{
    perObjectData             = PerObjectData.None,
    overrideMaterial          = _outlineMaterial,
    overrideMaterialPassIndex = 0,
};
```

---

## 3. FilteringSettings

告诉 URP "画哪些 Renderer"（在 cull 结果里过滤）。**值类型（struct）**。

### 构造函数

```csharp
// 最常用：queue 范围 + layerMask
public FilteringSettings(RenderQueueRange? renderQueueRange, int layerMask = -1);

// 完整参数
public FilteringSettings(
    RenderQueueRange? renderQueueRange,
    int layerMask                   = -1,
    uint renderingLayerMask         = 0xFFFFFFFF,
    int excludeMotionVectorObjects  = 0
);
```

### 核心属性

#### `renderQueueRange`（必设）⭐

按 Material 的 RenderQueue 过滤。

```csharp
filtering.renderQueueRange = RenderQueueRange.opaque;      // 0 - 2500
filtering.renderQueueRange = RenderQueueRange.transparent; // 2501 - 5000
filtering.renderQueueRange = RenderQueueRange.all;         // 0 - 5000
```

**预设范围**：

| 常量 | 数值范围 | 语义 |
|---|---|---|
| `RenderQueueRange.opaque` | 0 - 2500 | 不透明 |
| `RenderQueueRange.transparent` | 2501 - 5000 | 透明 |
| `RenderQueueRange.all` | 0 - 5000 | 全部 |

**自定义范围**：
```csharp
filtering.renderQueueRange = new RenderQueueRange
{
    lowerBound = 3000,
    upperBound = 3500
};
```

**常见 queue 数值**：

| 数值 | 名称 |
|---|---|
| 1000 | Background |
| 2000 | Geometry（默认不透明） |
| 2450 | AlphaTest |
| 3000 | Transparent |
| 4000 | Overlay |

**Material 的 RenderQueue 来源**：
- Shader 的 `Tags { "Queue" = "Transparent" }`（默认值）
- Material 面板 `Advanced Options → Render Queue`（覆盖）
- 代码 `material.renderQueue = 3000`

#### `layerMask`（必设）⭐

按 Unity 的 GameObject Layer 过滤（32 个 layer）。

```csharp
filtering.layerMask = -1;                       // 所有 layer（默认）
filtering.layerMask = 1 << 8;                   // 只画 layer 8
filtering.layerMask = (1 << 8) | (1 << 9);      // layer 8 和 9
filtering.layerMask = ~(1 << 5);                // 除了 layer 5
```

**推荐用 API 转换**：
```csharp
int fogLayer = LayerMask.NameToLayer("Fog");
filtering.layerMask = 1 << fogLayer;

// 多个 layer
filtering.layerMask = LayerMask.GetMask("Fog", "Effects", "Volumes");
```

**性能**：Layer 过滤是 URP culling 的一部分，几乎零开销。

#### `renderingLayerMask`（URP 独有）

URP 独有的 32 位过滤 mask，比 layerMask 更精细。

```csharp
filtering.renderingLayerMask = 0x00000001;
```

**和 layerMask 的区别**：

| | `layerMask` | `renderingLayerMask` |
|---|---|---|
| 来源 | Unity Layer | URP Rendering Layer |
| 配置位置 | GameObject Layer | Renderer.renderingLayerMask |
| 可否重叠 | 一个物体一个 layer | 可属于多个 rendering layer |
| 用途 | 通用过滤 | 细粒度光照/影子过滤 |

**默认全 1，一般不改**。

#### `excludeMotionVectorObjects`

从渲染中排除有 motion vector 需求的物体。**罕用**。

#### `sortingLayerRange`

按 SortingLayer 过滤（SpriteRenderer 用）。**3D 场景一般不管**。

### 属性使用频率

| 属性 | 频率 | 说明 |
|---|---|---|
| 构造函数传 queueRange | ⭐⭐⭐⭐⭐ | 必传 |
| `layerMask` | ⭐⭐⭐⭐⭐ | 强烈建议设 |
| `renderingLayerMask` | ⭐⭐ | 特殊需求 |
| `excludeMotionVectorObjects` | ⭐ | 罕用 |
| `sortingLayerRange` | ⭐ | 2D / Sprite 用 |

### 典型模式

**画所有透明物体**：
```csharp
var filtering = new FilteringSettings(RenderQueueRange.transparent);
```

**画特定 Layer 的物体（推荐）**：
```csharp
var filtering = new FilteringSettings(
    RenderQueueRange.transparent,
    layerMask: LayerMask.GetMask("Fog")
);
```

**排除某些 Layer**：
```csharp
var filtering = new FilteringSettings(
    RenderQueueRange.opaque,
    layerMask: ~LayerMask.GetMask("SkipMe")
);
```

**多 Layer 组合**：
```csharp
var filtering = new FilteringSettings(
    RenderQueueRange.transparent,
    layerMask: LayerMask.GetMask("Fog", "GodRay", "Volumes")
);
```

### 常见陷阱

**陷阱 1：忘给物体分配 Layer**
- Feature 里 `layerMask: 1 << 8`
- 场景里的物体没设为 layer 8
- → 什么都不画

排查：Frame Debugger 里看 drawcall 数是不是 0。

**陷阱 2：Material 的 renderQueue 和 Filter 不匹配**
- Material queue = 2000（opaque）
- Filter 是 `RenderQueueRange.transparent`
- → 永远匹配不上

排查：检查 Material 面板的 `Advanced Options → Render Queue`。

**陷阱 3：`layerMask = 0`**
- 空 mask，什么都不画
- 应该用 `-1` 表示所有 layer

### 最佳实践

给每个自定义效果**分配独立 Layer**（Edit → Project Settings → Tags and Layers），然后 Feature 只画对应 Layer。

**好处**：
- 其他物体不会被误伤
- 关卡设计师改 Layer 即可控制
- Layer culling 性能好

---

## 4. RenderStateBlock（可选）

**运行时覆盖 Shader 里 render state 的机制**。不改 Shader，从 C# 端强制修改 Blend / ZTest / Stencil / Cull 等。**值类型（struct）**。

### 为什么需要

**常规做法**：Shader 里写死 `Blend One One / ZWrite Off / Stencil { ... }`
- 一个 shader 一套固定 state
- 不同 Feature 想用不同 state → 要复制 shader
- 业务逻辑（如"限制叠加次数"）污染 shader

**RenderStateBlock**：C# 端覆盖
- Shader 不动
- 业务逻辑在 C#
- 调试灵活，参数化控制

### 核心结构

```csharp
public struct RenderStateBlock
{
    public RenderStateMask mask;   // 覆盖哪些 state
    public BlendState blendState;
    public RasterState rasterState;
    public DepthState depthState;
    public StencilState stencilState;
    public int stencilReference;
}
```

**关键**：`mask` 决定覆盖什么，未 mask 的部分保持 Shader 原设置。

### RenderStateMask 枚举

```csharp
[Flags]
public enum RenderStateMask
{
    Nothing    = 0,
    Blend      = 1 << 0,  // 覆盖 Blend
    Raster     = 1 << 1,  // 覆盖 Cull / Offset
    Depth      = 1 << 2,  // 覆盖 ZTest / ZWrite
    Stencil    = 1 << 3,  // 覆盖 Stencil
    Everything = ~0,      // 全部覆盖
}
```

**可组合**：`mask = RenderStateMask.Blend | RenderStateMask.Depth`

### 四种 State

#### `blendState` — 混合状态

对应 Shader `Blend`：

```csharp
stateBlock.blendState = new BlendState
{
    blendState0 = new RenderTargetBlendState
    {
        writeMask                 = ColorWriteMask.All,
        sourceColorBlendMode      = BlendMode.SrcAlpha,
        destinationColorBlendMode = BlendMode.OneMinusSrcAlpha,
        colorBlendOperation       = BlendOp.Add,
    }
};
```

#### `rasterState` — 光栅化状态

对应 Shader `Cull` 和 `Offset`：

```csharp
stateBlock.rasterState = new RasterState
{
    cullingMode          = CullMode.Front,
    depthBias            = 0.5f,
    slopeScaledDepthBias = 1.0f,
};
```

#### `depthState` — 深度状态

对应 Shader `ZTest` 和 `ZWrite`：

```csharp
stateBlock.depthState = new DepthState
{
    writeEnabled    = false,
    compareFunction = CompareFunction.LessEqual,
};
```

#### `stencilState` + `stencilReference` — 模板状态

对应 Shader `Stencil`：

```csharp
stateBlock.stencilReference = 2;   // 注意：ref 是 RenderStateBlock 的字段
stateBlock.stencilState = new StencilState
{
    enabled         = true,
    readMask        = 255,
    writeMask       = 255,
    compareFunction = CompareFunction.Greater,
    passOperation   = StencilOp.IncrementSaturate,
    failOperation   = StencilOp.Keep,
    zFailOperation  = StencilOp.Keep,
};
```

**⚠️ 陷阱**：`stencilReference` 是 `RenderStateBlock` 的字段，**不在 `StencilState` 里**。

### 使用模式

**只覆盖 Stencil（最常用）**：
```csharp
var stateBlock = new RenderStateBlock(RenderStateMask.Stencil)
{
    stencilReference = 2,
    stencilState = new StencilState(...),
};
// Blend / ZTest / Cull 保持 shader 设置
```

**只覆盖 Depth**：
```csharp
var stateBlock = new RenderStateBlock(RenderStateMask.Depth)
{
    depthState = new DepthState
    {
        writeEnabled = false,
        compareFunction = CompareFunction.Always
    }
};
```

**全部覆盖**：
```csharp
var stateBlock = new RenderStateBlock(RenderStateMask.Everything)
{
    blendState   = new BlendState(...),
    rasterState  = new RasterState(...),
    depthState   = new DepthState(...),
    stencilState = new StencilState(...),
    stencilReference = 0,
};
```

### 典型场景

**场景 1：Stencil 限制叠加次数**
```csharp
// 每个像素最多叠加 2 个物体
var stateBlock = new RenderStateBlock(RenderStateMask.Stencil)
{
    stencilReference = 2,
    stencilState = new StencilState(
        enabled: true, readMask: 255, writeMask: 255,
        compareFunction: CompareFunction.Greater,
        passOperation: StencilOp.IncrementSaturate,
        failOperation: StencilOp.Keep,
        zFailOperation: StencilOp.Keep
    )
};
```

**场景 2：Depth Prepass 强制不写颜色**
```csharp
var stateBlock = new RenderStateBlock(RenderStateMask.Blend)
{
    blendState = new BlendState
    {
        blendState0 = new RenderTargetBlendState
        {
            writeMask = ColorWriteMask.None
        }
    }
};
```

**场景 3：强制 Cull Back**
```csharp
var stateBlock = new RenderStateBlock(RenderStateMask.Raster)
{
    rasterState = new RasterState { cullingMode = CullMode.Back }
};
```

**场景 4：Outline 只画未被标记的地方**
```csharp
// 前置 Pass 已把物体区域 stencil 写成 1
// Outline Pass 只画 stencil != 1（即物体外边缘）
var stateBlock = new RenderStateBlock(RenderStateMask.Stencil)
{
    stencilReference = 1,
    stencilState = new StencilState(
        enabled: true,
        compareFunction: CompareFunction.NotEqual,
        passOperation: StencilOp.Keep,
        failOperation: StencilOp.Keep,
        zFailOperation: StencilOp.Keep,
        readMask: 255, writeMask: 0
    )
};
```

### 传给 DrawRenderers

**两个重载**：

```csharp
// 不带 stateBlock
context.DrawRenderers(cullResults, ref drawing, ref filtering);

// 带 stateBlock
context.DrawRenderers(cullResults, ref drawing, ref filtering, ref stateBlock);
```

`stateBlock` 用 **`ref`** 传（避免值类型拷贝）。

### 属性使用频率

| 属性 | 频率 | 说明 |
|---|---|---|
| `RenderStateMask.Stencil` + `stencilState` | ⭐⭐⭐⭐ | 限制叠加、outline mask |
| `RenderStateMask.Depth` + `depthState` | ⭐⭐⭐ | 强制关 ZWrite / 改 ZTest |
| `RenderStateMask.Blend` + `blendState` | ⭐⭐ | 覆盖 blend 模式 |
| `RenderStateMask.Raster` + `rasterState` | ⭐⭐ | 覆盖 Cull / bias |
| `Everything` | ⭐ | 极少用 |

### 常见陷阱

**陷阱 1：忘设 mask**
```csharp
var stateBlock = new RenderStateBlock  // 没传 mask，默认 Nothing
{
    stencilState = ...   // 不生效！
};
```
必须显式传 mask：`new RenderStateBlock(RenderStateMask.Stencil)`。

**陷阱 2：`stencilReference` 放错位置**
```csharp
// ❌ stencilReference 不在 StencilState 里
new StencilState(stencilReference: 2, ...)

// ✅ 在 RenderStateBlock 里
new RenderStateBlock(RenderStateMask.Stencil)
{
    stencilReference = 2,
    stencilState = new StencilState(...)
}
```

**陷阱 3：`writeMask = 0`**
- 只读不写，stencil 永远不变
- 想更新 stencil 就必须 `writeMask > 0`

---

## 5. 筛选流程串讲

`DrawRenderers` 一共有 **四道筛子**，全部通过才会真正渲染：

```
所有 Renderer
    ↓
① Culling（视锥剔除 + 遮挡剔除）
    ↓ cullResults
② LayerMask（GameObject Layer 过滤）
    ↓
③ RenderQueue 范围（Material 的 queue）
    ↓
④ ShaderTagId（Material 的 Shader 有没有对应 Pass）
    ↓
真正画的 Renderer
```

**任何一步失败都不画**。

### 每道筛子由谁配置

| 筛子 | 配置来源 |
|---|---|
| ① Culling | URP 自动（cullResults） |
| ② LayerMask | FilteringSettings.layerMask |
| ③ Queue 范围 | FilteringSettings.renderQueueRange |
| ④ Shader Tag | DrawingSettings 构造函数的 ShaderTagId |

**FilteringSettings 和 DrawingSettings 是配合关系**：
- **Filtering** 先粗筛（layer + queue，culling 阶段就完成）
- **Drawing** 再精筛（shader 有没有对应 Pass）

### 示例：具体筛选过程

**假设场景里有 4 个物体**：

| 物体 | Layer | Material Queue | Shader 里有 `MyForward` Pass? |
|---|---|---|---|
| Fog Cylinder | Default | Transparent (3000) | ✅ |
| 玻璃 Cube | Default | Transparent (3000) | ❌ |
| 粒子系统 | Default | Transparent (3000) | ❌ |
| 一堵墙 | Default | Geometry (2000) | ✅（意外加了） |

**Feature 里的配置**：
```csharp
FilteringSettings filtering = new FilteringSettings(
    RenderQueueRange.transparent  // 2501 - 5000
);
DrawingSettings drawing = new DrawingSettings(
    new ShaderTagId("MyForward"),
    sorting
);
```

**筛选过程**：

| 物体 | ① Culling | ② LayerMask(-1) | ③ Queue(2501-5000) | ④ Tag("MyForward") | 结果 |
|---|---|---|---|---|---|
| Fog Cylinder | ✅ | ✅ 全通 | ✅ 3000 ∈ range | ✅ 有此 Pass | **画** ✅ |
| 玻璃 Cube | ✅ | ✅ | ✅ 3000 ∈ range | ❌ 没此 Pass | 跳过 |
| 粒子 | ✅ | ✅ | ✅ 3000 ∈ range | ❌ 没此 Pass | 跳过 |
| 墙 | ✅ | ✅ | ❌ 2000 ∉ range | -（未到第 4 步）| 跳过 |

**结论**：只有 Fog Cylinder 会被画。

### 不加 Layer 会怎样

**技术上能跑**（Shader Tag 已经过滤），但有三个隐患：

**隐患 1：不同 Feature 抢 tag**
- 未来加的其他 Feature 如果用了相同 tag，会互相干扰
- 加了 Layer 隔离后，每个 Feature 只画自己 Layer，永不串扰

**隐患 2：性能浪费**
- `layerMask = -1` 时 URP 遍历所有透明物体
- 每个都要查 material 的 shader 里有没有对应 Pass
- 加 Layer 后 culling 阶段就过滤掉，比 tag 匹配快

**隐患 3：Editor 里误操作**
- 设计师换 material 时可能意外触发匹配
- Layer 隔离能避免这类问题

### 何时可以不加 Layer

**满足所有条件才能省 Layer**：
- ✅ Shader tag 名字独一无二（不和 URP 内置冲突）
- ✅ 项目里只有你自己用这个 tag
- ✅ 场景里目标物体数量少（性能不敏感）
- ✅ 团队只有你自己开发（没人误操作）

**任何一条不满足 → 应该加 Layer**。