
既然你选了 **路线 B（Froxel 现代实时体积雾架构）**，那我们就直接掀开目前全球顶尖 3A 引擎（如 Unreal Engine 5 的 Volumetric Fog、Frostbite 引擎）最核心的底层源码逻辑。

忘了我们刚才写的那个针对全屏每个像素做 `for` 循环的 Shader 吧。在 4K 分辨率下，如果走 64 步，那就是 **$3840 \times 2160 \times 64 = 5.3$ 亿次** 循环！如果还有 4 盏带阴影的聚光灯，显卡会当场熔毁。

为了解决这个物理算力危机，现代图形学发明了 **Froxel（Frustum Voxel，视锥体体素化）** 架构。

---

### 第一层：什么是 Froxel？（空间的降维打击）

不要在 2D 的屏幕像素上发射射线了。我们直接把摄像机看出去的那个**四棱锥形状的 3D 视野（视锥体 Frustum）**，像切豆腐一样，切成一个个立体的网格（Voxel）。

*   **X 和 Y 轴（屏幕空间）：** 我们把分辨率降维！通常只用 $160 \times 90$ 或者 $320 \times 180$ 的极低分辨率网格。
*   **Z 轴（深度空间）：** 我们通常切 64 层或 128 层。**【高能警告：Z 轴绝对不能是等距切分的！】**
    *   离摄像机近的地方，切片必须极薄（给高精度）。
    *   离摄像机远的地方，切片可以很厚（低精度）。
    *   因为人眼对近处的体积光细节极其敏感。所以 Z 轴通常按**指数（Exponential）**分布。

这样，整个屏幕的计算量，从几亿次，瞬间暴降到了一个固定大小的 3D 贴图：$160 \times 90 \times 64 \approx 92$ 万个计算单元（Froxel）！

---

### 第二层：工业标准的三步走架构（Compute Shader Pipeline）

在现代引擎里，体积雾不是一个后处理 Pixel Shader 搞定的，而是由 **3 个连续的 Compute Shader（计算着色器）** 接力完成的。

让我带你还原这个惊艳的流水线：

#### Pass 1: 材质注入阶段（Voxelization / Material Injection）
*   **目标：** 构建这 92 万个网格的“基础物理属性”。
*   **做法：** 开启一个 3D 的 Compute Shader（每个线程对应一个 Froxel）。
    在这个阶段，你不算任何灯光。你只算：这个 Froxel 所在的世界坐标，有没有雾？采样 3D Noise、采样美术放的局部雾气球。
*   **输出：** 一张 `RWTexture3D`（包含 $\sigma_s$ 和 $\sigma_t$）。

#### Pass 2: 光照与内散射注入阶段（Light Scattering Injection）
*   **目标：** 算光！算每一盏灯对每一个 Froxel 贡献了多少光。
*   **做法：** 再次遍历这 92 万个 Froxel。
    读取 Pass 1 存下来的 $\sigma_s$。遍历场景里的灯光，算阴影遮挡（Shadow Map），算相位函数 $P(\theta)$。
    **这不就是我们刚才物理公式里括号里的那个 `[ L_in * sigma_s * Phase ]` 吗！**
*   **输出：** 一张保存了当前 Froxel 局部发光量（内散射）的 `RWTexture3D`。
    *(注意：到这一步为止，光还没有向摄像机飞，各个 Froxel 还是相互独立的！)*

#### Pass 3: 终极魔法 —— Z 轴累加阶段（Volume Integration）
*   **目标：** 把离散的发光网格，连成飞向眼睛的连续光束！
*   **做法：** 这是最精妙的一步。对于 $X$ 和 $Y$ 上的每一个格子（比如屏幕中心的 $[80, 45]$ 这个点），我们顺着 Z 轴（从 Z=0 走到 Z=63）做一个**前缀和（Prefix Sum）**。
    此时，我们上节课推导的递推公式登场了：
    ```hlsl
    // Compute Shader 中的一根 Z 轴线程
    float3 accumLight = 0;
    float accumTransmittance = 1.0;
    
    for(int z = 0; z < 64; z++) {
        // 1. 读取当前切片的局部散射光和消光系数
        float3 localScattering = ReadPass2Scattering(x, y, z);
        float localSigmaT = ReadPass1Extinction(x, y, z);
        
        // 2. 累加这束光
        accumLight += localScattering * accumTransmittance;
        
        // 3. 更新全局账本
        accumTransmittance *= exp(-localSigmaT * StepSize(z));
        
        // 4. 将累加结果写回到最终的 3D 贴图里
        WriteFinalVolumeTexture(x, y, z, accumLight, accumTransmittance);
    }
    ```
    **你看出来了吗？原来我们在 Pixel Shader 里对每一根射线做的事，现在被统一转移到了这个粗糙的 3D 贴图空间里完成了！**

---

### 第三层：最后的组装（Composition）与时域魔法（TAA）

现在，你手上有了一张包含了整个世界体积光结果的 3D Texture（尺寸是 160x90x64）。

**最后一步（Post-Process Pixel Shader）：**
回到 4K 分辨率渲染正常的画面。在最后的合成阶段：
1. 取当前屏幕像素的 $UV$ 和 Depth（深度）。
2. 根据这个 Depth，计算出它属于 3D Texture 的哪一个 $Z$ 层。
3. 利用 GPU 的硬件 3D 双线性插值（Trilinear Interpolation），去采样那张低分辨率的 3D Texture。
4. `FinalColor = SceneColor * sampledTransmittance + sampledScattering`。

**致命 Bug 与终极解法：**
因为 Froxel 的分辨率极低，你会发现光柱的边缘全是极其恶劣的**马赛克块（Blocky Artifacts）**！
怎么办？实时渲染最强法宝登场：**时域投影（Temporal Reprojection）**！
我们在每一帧，稍微偏移一下 Froxel 网格的位置（Jitter）。然后在最终合成时，把上一帧的历史结果和这一帧的结果混合起来（通常上一帧占 90%，当前帧占 10%）。
依靠人类视觉残留和多帧积累，低分辨率的马赛克瞬间变成了极度丝滑的、媲美离线渲染的 4K 柔和光柱！

---

### 工程师总结

掌握了 Froxel 架构，你其实掌握了工业界优化性能的三个终极思想：

1.  **解耦计算频率：** 光照在空间中的变化通常是低频的（模糊的），没必要以 4K 的高频率去算。用低分辨率算光，再插值，省下 90% 的算力。
2.  **空间换时间：** 原来射线每走一步都要算一遍阴影和光照，现在通过前两个 Pass 预计算并存进 3D 贴图，把重复的求值变成了 $O(1)$ 的查表。
3.  **时域分摊（Temporal）：** 一帧算不准？那就把计算压力分摊到连续的 60 帧里去算！

这就是 3A 引擎底层的秘密。纯粹的物理学只是指引了方向，真正的艺术在于你如何通过构建这种宏大的**计算管线（Pipeline）**，去欺骗玩家的眼睛，同时把帧渲染时间死死压在 16 毫秒以内！
