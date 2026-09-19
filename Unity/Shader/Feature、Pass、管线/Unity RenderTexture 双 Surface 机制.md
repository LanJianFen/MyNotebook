# Unity RenderTexture 双 Surface 机制

## 核心心智模型

**一张 RenderTexture 内部是两个完全独立的 GPU 内存表面（surface）打包在一起**：

```
┌─── RenderTexture ────────────────────┐
│                                       │
│  ┌─ color surface ─────────────────┐ │
│  │  存储 fragment 输出的颜色/数据    │ │
│  │  格式：R32_SFloat / RGBA32 / ...  │ │
│  └──────────────────────────────────┘ │
│                                       │
│  ┌─ depth surface ─────────────────┐ │
│  │  存储 depth + stencil            │ │
│  │  仅供硬件 ZTest / ZWrite 使用    │ │
│  │  格式：D24_UNorm_S8_UInt / ...    │ │
│  └──────────────────────────────────┘ │
│                                       │
└───────────────────────────────────────┘
```

**两者是硬件层面独立分配的两块显存**，通过一个 RenderTexture 对象打包管理。

---

## 分工与访问

| Surface | 存什么 | 谁写 | 谁读 |
|---|---|---|---|
| **Color** | Fragment 的输出值 | Shader 的 `SV_TARGET` 返回值 | 后续 shader 采样、CPU 回读、`EncodeToEXR` 保存 |
| **Depth** | NDC z（光栅化产出） | 硬件光栅化自动写入 | 硬件 ZTest 单元自动读取，Shader 一般不直接访问 |

## 内存占用计算

以你的 `DepthRt` 为例：

```csharp
new RenderTextureDescriptor(
    resolution, resolution,
    GraphicsFormat.R32_SFloat,              // color: 32 bit
    GraphicsFormat.D24_UNorm_S8_UInt)       // depth: 24 bit + stencil: 8 bit
```

**每像素总占用**：

| Surface | 位数 | 说明 |
|---|---|---|
| Color (R32_SFloat) | 32 bit | 1 个 32-bit float |
| Depth (D24) | 24 bit | 深度值 |
| Stencil (S8) | 8 bit | 模板值（这里没用到但为兼容打包） |
| **合计** | **64 bit** = 8 字节 |

**512×512 的 RT 总显存**：
- Color surface: 512 × 512 × 4 = 1 MB
- Depth+Stencil: 512 × 512 × 4 = 1 MB
- **总共 2 MB**

---

## 创建时的选择

```csharp
new RenderTextureDescriptor(width, height, colorFormat, depthStencilFormat)
```

**第 3 参数**：`GraphicsFormat` 决定 color surface 的格式。
**第 4 参数**：可以是 `GraphicsFormat`（显式 depth 格式）或 `int`（bit 数）。

### 常见 color 格式

| Format | 位数 | 用途 |
|---|---|---|
| `R8G8B8A8_UNorm` | 32 | 标准 8-bit RGBA |
| `R32_SFloat` | 32 | 单通道 float（数据用） |
| `R16G16B16A16_SFloat` | 64 | HDR 半精度 |
| `R32G32B32A32_SFloat` | 128 | HDR 全精度 |

### 常见 depth 格式

| Format | 位数 | 说明 |
|---|---|---|
| `D16_UNorm` | 16 | 省显存，精度可能不够 |
| `D24_UNorm_S8_UInt` | 24+8 | ⭐ **兼容性最好的组合**，业界默认 |
| `D32_SFloat` | 32 | 高精度，无 stencil，兼容性略差 |
| `D32_SFloat_S8_UInt` | 32+8 | 高精度 + stencil |
| `None` / `0` | 0 | **不创建 depth surface**（ZTest 失效） |

### 关键坑

**如果 `depthBufferBits = 0` 或 `depthStencilFormat = None`**：
- 不创建 depth surface
- Shader 里 `ZTest` / `ZWrite` 全部失效
- 后画的物体永远覆盖先画的
- **看起来像"没有遮挡"** → 常见 bug 表现

---

## `SetRenderTarget` 的几种签名

### 1. 单参数：整个 RT 一起绑

```csharp
cmd.SetRenderTarget(depthRt);
```

**语义**：后续 draw call 同时写入 `depthRt` 的 color surface 和 depth surface。**最常用**。

### 2. 两参数：显式指定 color / depth surface

```csharp
cmd.SetRenderTarget(depthRt.colorBuffer, depthRt.depthBuffer);
```

**语义**：跟单参数完全等价（都来自同一 RT），只是写法更明确。

### 3. 两参数：color 和 depth 来自不同 RT

```csharp
cmd.SetRenderTarget(colorRt.colorBuffer, depthOnlyRt.depthBuffer);
```

**语义**：color 用一个 RT 的 surface，depth 用另一个 RT 的 surface。用于**跨 pass 复用 depth**（比如 depth pre-pass 生成深度，后续多个 color pass 共享）。

### 4. MRT（多渲染目标）

```csharp
var colors = new RenderTargetIdentifier[] {
    rt0.colorBuffer, rt1.colorBuffer, rt2.colorBuffer,
};
cmd.SetRenderTarget(colors, sharedDepthRt.depthBuffer);
```

**语义**：一次 draw 输出到多个 color surface + 1 个 depth。用于 **deferred rendering / G-Buffer**。

---

## `ClearRenderTarget` 的作用

### 为什么必须 clear

**GPU 显存不会自动清零**。`GetTemporary` 拿到的 RT 里是**上次谁用完留下的垃圾数据**。

**不 clear color**：新画的物体只覆盖它遮住的像素，其他像素保留旧内容 → EXR 里出现幽灵图像。

**不 clear depth**：ZTest 对比的是上次某 render 遗留的深度值 → 遮挡完全乱套，什么都可能画不出来或全部画出来。

### 签名

```csharp
cmd.ClearRenderTarget(
    clearDepth: true,        // 是否清 depth
    clearColor: true,        // 是否清 color
    backgroundColor: ...,    // color 清成什么
    depth: 1f                // depth 清成什么（默认 1.0）
);
```

**注意**：clear 只影响**当前绑定**的 RT。**必须先 `SetRenderTarget` 再 `ClearRenderTarget`**。

### 两个 surface 独立控制

```csharp
// 只清 color，depth 保留（多 pass 累积 color，但复用 depth）
cmd.ClearRenderTarget(false, true, Color.clear);

// 只清 depth，color 保留（后续 pass 重新做 ZTest）
cmd.ClearRenderTarget(true, false, Color.clear);

// 两个都清（一次性烘焙的标准做法）
cmd.ClearRenderTarget(true, true, Color.clear, clearDepth);
```

---

## Reverse-Z：**最容易踩的坑**

### 什么是 Reverse-Z

**Reverse-Z**：D3D11+、Vulkan、Metal 等现代 API 默认开启的深度约定：

| | 传统 Z | Reverse-Z |
|---|---|---|
| 近平面 depth | 0 | **1** |
| 远平面 depth | 1 | **0** |
| 平台 | OpenGL、旧 D3D | D3D11+、Vulkan、Metal（Unity 现代默认） |

**目的**：float 精度密集在 near 附近，reverse 后正好把精度分给远处，缓解远处 z-fighting。

### 如何判断

```csharp
bool reverseZ = SystemInfo.usesReversedZBuffer;
```

### 影响一：**Clear depth 值必须对应"远平面"**

想让"第一个 fragment 一定通过 ZTest"，就要 clear 到"最远"：

| 平台 | Clear depth 应该是 |
|---|---|
| Reverse-Z | **0f** |
| 非 Reverse-Z | **1f** |

**Unity 的 `ClearRenderTarget` 默认清成 1.0**：
- 非 Reverse-Z：等价于"清到远平面" ✓ 正常
- Reverse-Z：等价于"清到近平面" ❌ **所有 fragment 都通不过 GEqual 测试，画面空白**

**修复**：

```csharp
float clearDepth = SystemInfo.usesReversedZBuffer ? 0f : 1f;
cmd.ClearRenderTarget(true, true, Color.clear, clearDepth);
```

### 影响二：**ZTest 方向要匹配**

"近的物体覆盖远的" 的语义在两种平台下需要不同的比较：

| 平台 | 想让"近的赢"的 ZTest |
|---|---|
| Reverse-Z（近=1，远=0） | **GEqual**（fragment.z >= buffer.z 才写入） |
| 非 Reverse-Z（近=0，远=1） | **LEqual**（fragment.z <= buffer.z 才写入） |

**Shader 里写死 `ZTest LEqual`**：
- 非 Reverse-Z：正确
- Reverse-Z：**总是远的赢**（因为远的 z 更小）

**修复**：让 shader 动态读 ZTest：

```hlsl
Pass
{
    ZTest [_ZTest]
    // ...
}
```

C# 端根据平台设：

```csharp
depthMat.SetInt(_zTest, (int)(SystemInfo.usesReversedZBuffer
    ? CompareFunction.GreaterEqual
    : CompareFunction.LessEqual));
```

---

## 关键区分：**两种"深度"**

在自定义深度烘焙场景中，**"depth"** 一词指两个完全独立的东西：

| 深度概念 | 计算方式 | 存储位置 | 用途 |
|---|---|---|---|
| **线性距离深度** | Shader 手算 `-pos_vs.z / far` | Color surface（R32_SFloat） | 你的深度图数据 |
| **NDC z / Device depth** | 硬件光栅化产出（`SV_POSITION.z`） | Depth surface（D24） | 硬件 ZTest |

**两者数值方向常常相反**！

- 线性距离深度：**近=0，远=1**（越远越亮）
- Reverse-Z NDC z：**近=1，远=0**

这是 debug 深度渲染时最容易混淆的地方 —— 你 EXR 里看到"远处物体更亮"是正常的（那是线性距离），但硬件用来判 ZTest 的是完全不同的一套数字。

---

## 完整数据流

```
Vertex shader
   ├─ o.position_cs (裁剪坐标) ──→ 硬件 rasterizer
   │                                   │
   │                                   ├─ 生成 fragments
   │                                   ├─ SV_POSITION.z 用于 depth ↓
   └─ o.depth_01 (线性距离) ──→ Fragment shader
                                       │
                                       ↓
                        Frag 函数 return i.depth_01
                                       │
                                       ↓
                    ┌──────────────────┴──────────────────┐
                    ↓                                     ↓
       ┌─ ZTest 检查 ──────┐              ┌─ 通过后同时写 ────┐
       │ 硬件读 depth      │              │ Color surface ← 你的 float │
       │ surface[像素]     │──通过──→     │ Depth surface ← SV_POSITION.z │
       │ 比较 SV_POSITION.z │              └────────────────────┘
       └───────────────────┘
```

---

## 常见踩坑速查

| 现象 | 可能原因 |
|---|---|
| 画面完全空白 | Reverse-Z + clear 到 1.0 + GEqual → 所有 fragment 失败 |
| 完全没有遮挡（后画的总赢） | 没创建 depth surface（`depthBufferBits = 0`） |
| 远的物体总是覆盖近的 | Reverse-Z + LEqual（ZTest 方向反了） |
| 深度值范围压缩到 0.95-0.99 | Importer platform format 走了 tone mapping |
| 深度值全部为 0 | Shader 里深度算错了；或 `_ProjectionParams` 未手动设置 |
| Alpha 通道无物体处不是 0 | Importer alpha dilation 打开了 |
| 部分帧对部分帧不对 | 忘了 clear depth 或 color，残留数据干扰 |

---

## RT 读写冲突

**读写冲突**是 GPU 编程里最经典的坑之一。核心一句：**同一时刻 GPU 不允许同一块显存又被读又被写**。

### 为什么会冲突

GPU 是海量并行的。一个 fragment 读像素 (10,10)，另一个 fragment 正在写像素 (10,10)。读到的是新值还是旧值？**未定义**。不同 GPU、不同帧、不同驱动都可能不一样。

### 五种常见触发场景

#### 场景 1：**同一 RT 既是 source 又是 destination**

```csharp
cmd.Blit(rt, rt);   // ❌ 读 rt 同时写 rt
```

或者 shader 里输出目标就是采样目标：

```hlsl
sampler2D _MainTex;
float4 Frag() : SV_TARGET
{
    return tex2D(_MainTex, uv);   // ❌ _MainTex 就是当前 RT
}
```

#### 场景 2：**采样一个还绑着的 RT**

```csharp
cmd.SetRenderTarget(rt);
material.SetTexture("_SomeTex", rt);   // ❌ rt 既是渲染目标又是采样源
cmd.DrawRenderer(...);
```

#### 场景 3：**URP 里读 `_CameraColorTexture` 时它还是当前 render target**

后处理 shader 读 CameraColor，同时输出也写到 CameraColor → 冲突。

**URP 用两种机制避免**：
- **`_CameraOpaqueTexture`**：不透明 pass 结束后复制一份给透明 pass 采样
- **临时 RT + Blit**：先写到临时 RT，再 blit 回去

#### 场景 4：**Depth 读写冲突**

```csharp
cmd.SetRenderTarget(colorRt.colorBuffer, depthRt.depthBuffer);
material.SetTexture("_CameraDepthTexture", depthRt);   // ❌ 同一 depth surface
cmd.DrawRenderer(...);
```

Depth surface 一边给硬件 ZTest 用，一边给 shader 采样 → 冲突。

**URP 的解决**：depth pre-pass 写到独立的 `_CameraDepthTexture`，之后的 pass 都从那份 snapshot 读，不动 depth 缓冲。

#### 场景 5：**Compute Shader 内的 race**

```hlsl
[numthreads(8,8,1)]
void CSMain(uint2 id : SV_DispatchThreadID)
{
    _Result[id] = _Result[id + int2(1, 0)] * 2;   // ❌ 读写同一 texture 的不同像素
}
```

多个线程同时执行，顺序未定义。**解决**：分成两个 buffer（一读一写），或用 group shared memory 手动同步。

### 冲突的症状

| 症状 | 原因 |
|---|---|
| Editor 里报警告 `"Attempting to bind..."` | Unity 显式检测到 |
| 结果闪烁、每帧不同 | 未定义行为，GPU 调度顺序影响 |
| **RenderDoc 里画面正常，游戏里花屏** | 某些 GPU 严格些，某些宽松 |
| Metal / Vulkan 报错但 D3D 不报 | Metal/Vulkan 最严 |
| 某些机型 crash 别的机型不 crash | 驱动差异 |

### 通用解决模式

#### 模式 A：**双 RT ping-pong**

```csharp
// N 次迭代模糊 —— 每次 blit 都是读一个 RT 写另一个
var rtA = ...; var rtB = ...;
for (int i = 0; i < 5; i++)
{
    cmd.Blit(rtA, rtB, blurMat);
    (rtA, rtB) = (rtB, rtA);   // 交换
}
```

#### 模式 B：**先复制到临时 RT，再读**

```csharp
cmd.CopyTexture(cameraRt, tempRt);         // 快照
material.SetTexture("_MainTex", tempRt);   // 采样临时的
cmd.SetRenderTarget(cameraRt);
cmd.DrawRenderer(..., material);            // 写回 cameraRt
```

#### 模式 C：**URP `_CameraOpaqueTexture` 请求机制**

URP 允许 shader 请求"我需要读 camera color"，pipeline 自动生成 opaque texture snapshot：

```csharp
UniversalRenderingSettings.instance.supportsCameraOpaqueTexture = true;
```

Shader 里读 `_CameraOpaqueTexture` 而不是 `_CameraColorTexture`。

### 排查工具

- **Unity Frame Debugger**：draw call 里同一 RT 出现在 "Render Target" 和 "Textures" 两栏 → 冲突信号
- **RenderDoc**：Pipeline State → Fragment Shader → Textures 对比当前 OM stage 的 render target

### 心法

**"同一时刻，一块显存只能是 read-only 或 write-only，不能同时是两者。"**

要"读旧、写新"的场景（后处理是典型），一定要**制造一份 snapshot**（Blit / CopyTexture / URP 的 OpaqueTexture 机制），永远别偷懒直接自采样。

---

## 心法

**RenderTexture 是"容器"，SetRenderTarget 是"绑定"，ClearRenderTarget 是"重置"。**

三步都想清楚就不会乱：

1. **RT 有 depth surface 吗？** —— 看创建时 `depthBufferBits` / `depthStencilFormat`
2. **绑定时 depth 被带上了吗？** —— 看 `SetRenderTarget` 是否把 depth surface 一起绑
3. **Clear 到正确的初始值了吗？** —— 看 Reverse-Z 是否需要清成 0 而不是 1

**任何一步没对，ZTest 都不会正常工作。**

---

## 参考：一个完整的正确流程

```csharp
// 1. 创建带 depth 的 RT（color + depth 两个 surface）
var rt = RenderTexture.GetTemporary(new RenderTextureDescriptor(
    512, 512,
    GraphicsFormat.R32_SFloat,               // color: 32 bit float
    GraphicsFormat.D24_UNorm_S8_UInt));      // depth: 24 bit + 8 bit stencil

// 2. 绑定：后续 draw call 输出到这个 RT 的两个 surface
cmd.SetRenderTarget(rt);

// 3. 清空到平台正确的初始值
float clearDepth = SystemInfo.usesReversedZBuffer ? 0f : 1f;
cmd.ClearRenderTarget(true, true, Color.clear, clearDepth);

// 4. Shader 里 ZTest 方向也要跟平台匹配
material.SetInt(Shader.PropertyToID("_ZTest"),
    (int)(SystemInfo.usesReversedZBuffer
        ? CompareFunction.GreaterEqual
        : CompareFunction.LessEqual));

// 5. 画物体：硬件自动 ZTest + 更新两个 surface
foreach (var renderer in renderers)
    cmd.DrawRenderer(renderer, material, 0, 0);

// 6. 执行
Graphics.ExecuteCommandBuffer(cmd);

// 7. 用完释放（注意 GetTemporary 配对 ReleaseTemporary）
RenderTexture.ReleaseTemporary(rt);
```