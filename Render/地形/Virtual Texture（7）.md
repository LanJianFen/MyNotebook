那万一中间问页表的时候发现没加载level1的这一块tile怎么办，fallback和feedback怎么进行后续的处理

这是一个非常关键的工程问题！处理好“未命中（Cache Miss）”，才是 VT 技术能否真正落地的分水岭。

当 Shader 拿着 `Level 1` 的 UV 去查页表，发现页表说：“不好意思，这块数据（Status）是空的 / 未加载。”

这时候，显卡会兵分两路：
1.  **左手画画（Fallback）：** 为了不让屏幕变黑，赶紧找个替补画上去。
2.  **右手写信（Feedback）：** 赶紧写个条子告诉 CPU：“缺货了！速补！”

我们详细拆解这两个过程。

---

### 第一路：Fallback（降级渲染 / 备胎计划）

**目标：** 玩家绝对不能看到黑色块或者紫红色的错误块。
**策略：** **“儿子不在找爸爸”**。

如果 `Level 1` 的块没加载，那它的上一级 `Level 2`（更模糊的一层）大概率是加载了的。
如果 `Level 2` 也没加载，那就找 `Level 3`...
直到找到 **根节点（Level N）**，那个最模糊的全图预览，通常是**常驻显存**的，永远都在。

#### Shader 里的循环逻辑（伪代码）：

```hlsl
// 1. 起始请求：Level 1
int targetMip = 1;
float4 pageInfo;

// 2. 循环查找（Loop）
// 最多找几层？比如往下找 4 层，或者直到找到为止
for (int i = 0; i < 4; i++) 
{
    // 去查页表的 targetMip 层
    pageInfo = tex2Dlod(_PageTable, float4(v_uv, 0, targetMip));
    
    // 检查页表里的状态位（通常存在 Alpha 通道或者专门的位标记）
    // 假设 pageInfo.a > 0 代表“已加载”
    if (pageInfo.a > 0) 
    {
        // 找到了！跳出循环
        break; 
    }
    
    // 没找到？降级！
    targetMip++; // 去找 Level 2
}

// 3. 此时 targetMip 变成了找到的那一层（比如 Level 3）
// 重新计算物理坐标（用新的 Scale 和 Bias）
float2 physicalUV = v_uv * pageInfo.z + pageInfo.xy;

// 4. 采样物理图集
float4 finalColor = tex2D(_PhysicalAtlas, physicalUV);
```

**视觉效果：**
原本应该清晰的地方，因为被迫用了 `Level 3`，看起来会**由于分辨率不足而变模糊**。这就解释了为什么有时你快速转头，物体会先模糊一下，然后变清晰。

---

### 第二路：Feedback（写入反馈 / 求救信号）

**目标：** 通知 CPU 加载 `Level 1` 的数据，以便几帧后我们能不用备胎。
**难点：** Pixel Shader 是并行的（几百万个像素同时跑），我们不能让几百万个线程同时给 CPU 发中断。

**策略：** **反馈缓冲区（Feedback Buffer）**。

#### 1. 建立一个小缓冲区
我们需要一张专门的、分辨率很小的**渲染目标（Render Target）**，通常只有屏幕的 **1/16** 甚至更小（比如 64x64 或者 128x128 像素）。
这张图不给人看，是给 CPU 看的。

#### 2. Shader 写入请求
在刚才 Shader 发现 `Level 1` 缺失的时候，除了做 Fallback，它还会把这个**“原本想要的 Page ID”** 写到这个小缓冲区里。

*   **写入什么？** 通常是一个 `uint`（无符号整数），包含了 `(PageX, PageY, MipLevel)` 的编码。
*   **因为缓冲区很小：** 很多屏幕像素会映射到同一个反馈像素上。但这没关系！反正它们要是都在同一个 Tile 里，缺的也是同一个 Page。显卡的 **深度测试（Depth Test）** 或 **原子操作（InterlockedMax）** 会自动去重。

```hlsl
// 如果发现缺页
if (missing) 
{
    // 计算当前需要哪个 Page ID
    uint pageID = EncodePageID(v_uv, originalMipLevel);
    
    // 写入反馈 Buffer（这是 Pixel Shader 的一个额外输出）
    // 下一帧 CPU 会来读这张图
    OutFeedback = float4(Unpack(pageID), 1.0);
}
```

#### 3. 现代硬件加速：Sparse Texture (Tiled Resources)
早期的 VT 需要自己写 Buffer。
现在 DirectX 12 / Vulkan 有硬件支持（**Residency Map**）。
显卡会自动统计：“在这个 DrawCall 里，有哪些 Tile 被访问了但是没数据（Page Fault）”，并直接生成列表给 CPU。

---

### 后续处理：CPU 接盘（异步加载）

这是发生在**下一帧（甚至几帧后）**的事情：

1.  **回读（Readback）：**
    CPU 从显存里把那个 **Feedback Buffer** 读回来（这是比较慢的 PCIe 操作，所以通常会延迟一两帧）。

2.  **解析与去重（Parse & Deduplicate）：**
    Feedback Buffer 里可能写满了“我要第 100 号块！我要第 100 号块！”
    CPU 整理一下名单：OK，这一帧总共缺了：`Page 100`，`Page 205`，`Page 88`。

3.  **IO 调度（Streaming）：**
    CPU 告诉硬盘（文件系统）：把这 3 个块的数据读出来！
    *   *优化点：* 按优先级排序。离摄像机近的先读。

4.  **上传与更新（Upload & Update）：**
    *   硬盘读完后，CPU 把数据解压。
    *   在 **物理图集 (Physical Atlas)** 里找几个空的停车位，把图塞进去。
    *   **最关键的一步：** 修改 **页表贴图**！
        *   把 `Page 100` 对应的页表像素，从“空”改成“指向物理车位 (X, Y)”。

5.  **闭环完成：**
    再过一帧，显卡渲染时：
    *   Shader 查页表 -> `Level 1` 有数据了！
    *   不需要 Fallback 了。
    *   物体表面瞬间变清晰（Pop-in）。

### 总结

*   **Fallback (备选):** Shader 里做 `while` 循环，找不到高清的就找低清的，确保画面永远有东西。
*   **Feedback (反馈):** Shader 把“原本想要的那个块”的 ID，偷偷写到一张极小的 **Feedback Texture** 上。
*   **Resolve (解决):** CPU 每一帧检查这张小图，像接到订单一样去硬盘进货，然后更新页表，完成闭环。
