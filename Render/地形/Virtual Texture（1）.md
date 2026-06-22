

好的，我们把镜头从宏观拉近到 **微观的管线级别**。

这就好比我们刚才在聊“物流系统”，现在我们要聊“快递员怎么把这一单快递送到你手上”。

你提出的 **四个核心问题** 问得非常专业，这正是实现 Virtual Texture (VT) 的最痛点。我们一步步拆解由于 VT 带来的 **Shader 内部的革命**。

---

### Phase 1: 数据的准备 (The Giant Texture)

#### Q1: 原始的超大贴图是已经混合好的材质吗？
**是的，通常是烘焙好的最终结果（Baked Final）。**

在开发阶段（比如 Substance / Mari / UE5 World Partition）：
你可能在这个巨型地形上涂了草、石头、泥土，这由很多层 Layer 混合而成。
但在 **导出 VT 数据 (Cooking)** 的时候，引擎会把这一切 **“压扁 (Flatten)”**。
*   它不存 "Grass_Material" 或 "Rock_Material"。
*   它存的是 **一张巨大的 Albedo 图**，一张巨大的 Normal 图，一张巨大的 Roughness 图。
*   这些图在硬盘上被切成了无数个 **Page (小块，通常 128x128 或 256x256 px)**。

**一定要记住：** VT 系统不关心“这是草还是石头”，它只关心“这是第 (X,Y) 号的颜色块”。

---

### Phase 2: Shader 里的魔法 (Process inside Pixel Shader)

现在，Pixel Shader 开始运行了。

#### Q2: 它的 UV 是在原始的那个超大贴图上采吗？
**是，也不是。**

*   **输入 UV (`v_uv`):** 确实是那张**虚拟的无限大贴图**的 UV（范围 0.0 ~ 1.0）。
    *   比如 UV = `(0.5001, 0.5001)`，代表正中心稍微偏一点点。
*   **动作：** 我们**绝对不能**直接用这个 UV 去采贴图。因为那张图在显存里根本不存在！显存里只有一张乱七八糟拼起来的 **物理缓存 (Physical Atlas)**。

我们需要做的是 **地址转换 (Address Translation)**：把 `虚拟 UV` 变成 `物理 UV`。

#### Q3: 怎么去找对应 Mip 级别的小块？(Mip Calculations)

这是最难的一步。在普通纹理采样中，硬件自动计算 MipLevel。但在 VT 里，我们需要**手动计算**或者**辅助计算**。

**Step A: 计算需要的 Mip Level**
Shader 利用屏幕空间的导数（Derivatives）来判断当前像素覆盖了多大的纹理区域。
```hlsl
// 计算 UV 在屏幕上的变化率
float2 dx = ddx(v_uv * VirtualTextureSize);
float2 dy = ddy(v_uv * VirtualTextureSize);
// 算出当前像素到底需要第几级 Mip
float mipLevel = 0.5 * log2(max(dot(dx, dx), dot(dy, dy))); 
mipLevel = round(mipLevel); // 取整，确定我们要找第几层的 Page
```
假设算出 `mipLevel = 2`。

**Step B: 查页表 (Indirection Lookup)**
现在我们知道：**“我需要第 2 级 Mip 上，位置在 (0.5, 0.5) 的那个 Page。”**

我们要去查 **页表 (Page Table / Indirection Texture)**。
页表本身也是一张特殊的贴图，而且它也有 Mipmap！
*   我们去采 **页表的 Mip 2**。
*   采样坐标还是原始的 `v_uv`。

```hlsl
// 查页表（注意：这里不是采颜色，而是采坐标信息！）
float4 pageInfo = tex2Dlod(_PageTable, float4(v_uv, 0, mipLevel));
```
这个 `pageInfo` 里的 RGBA 存的是什么？这可是藏宝图：
*   **PageX, PageY:** 这个 Page 在**物理缓存 (Atlas)** 里的起始角落坐标（比如放在了第 3 行第 4 列）。
*   **Scale:** 缩放比例（因为 Page 在 Atlas 里只占一小块）。
*   **Bias:** 偏移量。

**Step C: 假如页表说“没人” (Page Miss)**
如果 `pageInfo` 是空的（比如 Alpha=0），说明这个 Page 还没加载进显存。
*   **Fallback:** Shader 会自动降级去采更低级（更模糊）的 MipLevel，或者显示调试色（紫黑色）。
*   **Feedback:** 同时，显卡会将这个“缺页请求”写入一个特殊的 Buffer，告诉 CPU：“快去硬盘读这张图！”

#### Q4: 怎么用这个小块绘制？(Coordinate Transformation)

如果页表说“有货”，我们拿到了 `pageInfo`。现在进行最后的**坐标变换**。

原本的 `v_uv` 是针对整个地球的 0~1。
我们需要把它映射到 **物理缓存 (Atlas 4096 x 4096)** 里的某一个小格子 (128x128) 内部。

这就好比：你是全中国的坐标，我要把你换算成“某省某市某小区第几栋楼”。

**公式（原理）：**
```hlsl
// 1. 取出 Page 内的小数部分（Fract）
float2 pageUV = frac(v_uv * VirtualTextureSize / (1 << mipLevel));

// 2. 加上 Page 在 Atlas 里的物理偏移量
float2 physicalUV = pageUV * PageScale + PageOffset; // 这些数据来自 pageInfo
```

**最终采样：**
现在我们终于有了一个合法的 UV，指向那个乱七八糟拼图里的某一个小方块。

```hlsl
// 采样真正的物理纹理（Atlas）
float4 finalColor = tex2D(_PhysicalTexture, physicalUV);
```

---

### 总结全过程 (The Pipeline)

想象你在玩《赛博朋克 2077》，你正盯着墙上的一张海报（VT）。

1.  **Pixel Shader 启动。** 拿到原始 UV。
2.  **算 Mip:** 计算 `ddx/ddy`，发现你需要 **Mip 0 (最高清)**。
3.  **查页表:** 用原始 UV 去读 `IndirectionTexture` 的第 0 层。
    *   **结果:** 读到了！数据说：“这块图在 **物理缓存图** 的右下角。”
4.  **算坐标:** 把原始 UV 通过数学公式，变成指向 **物理缓存图右下角** 的 UV。
5.  **采颜色:** `tex2D(_PhysicalCache, transformedUV)`。
6.  **输出颜色。**

**如果这是你第一次看这张海报：**
*   第 3 步会失败（页表说：空）。
*   Shader 会降级去采 Mip 5（很模糊，但常驻内存）。
*   后台发信号给 CPU。
*   几帧后，硬盘把高清图读出来，塞进物理缓存，更新页表。
*   你看到海报突然“啪”地一下变清晰了。这就是 **Texture Streaming Pop-in**。
