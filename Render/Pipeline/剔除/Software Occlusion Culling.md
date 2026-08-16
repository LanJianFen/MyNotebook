

这是一个非常前沿且在现代引擎（如 Unreal Engine 4/5 和一些自研引擎）中极其重要的技术点。

**Software Occlusion Culling (SOC / 软件遮挡剔除)**，简单来说，就是**利用 CPU 的算力，手动写代码模拟 GPU 的深度测试（Depth Test），在把物体提交给 GPU 渲染之前，先在 CPU 这边把被挡住的物体剔除掉。**

如果说 PVS 是“离线查表”，Portal 是“几何剪裁”，那么 SOC 就是**“CPU 模拟显卡”**。

下面我从原理、实现细节、工业界标准库以及优缺点详细介绍。

---

### 1. 为什么需要 SOC？

通常我们将遮挡剔除分为两种思路：
*   **PVS（预计算）**：快，但不支持动态物体（卡车挡不住人），且内存大。
*   **Hardware Occlusion Queries (GPU查询)**：由 GPU 告诉 CPU 某个物体是否可见。
    *   *缺点*：有严重的**回读延迟（Readback Latency）**。CPU 发出查询 -> GPU 渲染 -> GPU 返回结果 -> CPU 读结果，这通常需要等到下一帧甚至下下帧才能拿到结果，导致物体会出现“闪烁”或剔除不及时。

**SOC 的出现就是为了解决上面两个问题：**
1.  支持**动态场景**（不需要烘焙）。
2.  **无延迟**（在当帧的渲染命令提交前就能知道结果）。

---

### 2. 核心原理与流程

SOC 的核心思想是：在 CPU 内存里维护一个**非常低分辨率的深度缓冲（Software Z-Buffer）**，比如 256x128 像素。

#### 第一步：筛选大遮挡体（Select Occluders）
每一帧开始时，引擎选出场景里几个**最大的、不透明**的物体（如大墙壁、山丘、大卡车）。这些叫 Occluders。

#### 第二步：软件光栅化（Software Rasterization）
这是最硬核的一步。CPU 使用极度优化的指令集（SIMD, AVX, NEON）将这些大遮挡体的**低模（LOD）**或**包围盒（AABB）**绘制到那个小小的深度缓冲里。
*   *注意*：这里不处理颜色、贴图、光照，**只画深度（Depth）**。所以速度非常快。

#### 第三步：遮挡查询（Occlusion Query）
对于剩下的几千个小物体（Occludees）：
1.  计算它们在屏幕上的**屏幕空间包围框（Screen Space Bounding Box）**。
2.  拿着这个框，去那个低分辨率的深度缓冲（Z-Buffer）里比对。
3.  **逻辑判断**：如果这个框的最浅深度（最近点），比深度缓冲里对应区域的最深深度（最远点）还要远 -> 说明完全被挡住了。
4.  **结果**：被挡住的物体直接标记为不可见，不生成 Draw Call。

---

### 3. 技术难点：SIMD 加速

你会问：*“CPU 画图不是特别慢吗？以前玩游戏还要显卡加速呢。”*

是的，普通的 C++ 代码在 CPU 上画图确实慢。所以 SOC 的实现完全依赖于 **SIMD (Single Instruction, Multiple Data)** 技术。
*   **Intel SSE/AVX 指令集**（PC端）：一次能处理 4 到 8 个浮点数。
*   **ARM NEON 指令集**（手机端）：一次处理 4 个浮点数。

通过向量化编程，CPU 可以一次性计算多个像素的深度值。Intel 开源了一个非常著名的库叫 **Masked Occlusion Culling (MOC)**，它就是把 CPU 软光栅化性能压榨到极致的典范。很多 3A 游戏（包括 Unreal Engine）底层都集成了类似的算法。

---

### 4. 优缺点对比

#### 优点（Pros）
1.  **支持动态遮挡**：这是最大的优势。一辆移动的大卡车，在 SOC 里可以实时被画进深度缓冲，走在它后面的人就能被剔除。这是 PVS 做不到的。
2.  **没有帧延迟**：CPU 这一帧算完，马上决定这一帧剔除谁，直接提交 GPU，所见即所得，不会闪烁。
3.  **节省 GPU 压力**：把原本要 GPU 做的 Early-Z 测试提前到了 CPU 做，大幅减少了 Vertex Shader 和 Polygon 的处理量。
4.  **无需烘焙**：不需要像 PVS 那样等几个小时烘焙，也不占用内存和包体大小。

#### 缺点（Cons）
1.  **CPU 开销（CPU Overhead）**：这是最大的代价。如果你的游戏逻辑（AI、物理）已经把 CPU 占满了，再跑 SOC 会导致掉帧（CPU瓶颈）。
2.  **分辨率限制（False Positives）**：为了快，深度缓冲通常很小（如 256x128）。这意味着很多细小的缝隙会被“糊住”，导致本来能看见的东西被错误地剔除（不过通常 SOC 算法设计是保守的，宁可多画，不可少画，所以更多情况是：本来该剔除的没剔除）。
3.  **实现极其复杂**：写一个高效的、利用 AVX/NEON 指令集的软光栅化器，门槛非常高。

---

### 5. 在游戏引擎中的现状

#### Unreal Engine (UE4/UE5)
UE 非常依赖 SOC。在 Project Settings -> Rendering -> Culling 里，有一个 `Occlusion Culling` 选项，默认开启。
*   它使用的正是类似 Intel MOC 的技术。
*   UE 会把所有的 Static Mesh 中较大的当作 Occluder，画在 CPU 的 Z-Buffer 上，然后剔除由于动态变化产生的遮挡。

#### Unity
Unity 原生的遮挡剔除（Occlusion Culling）主要还是 **Umbra (PVS)**。
*   但在 **SRP (URP/HDRP)** 以及随着 **DOTS (Data-Oriented Technology Stack)** 的推进，Unity 正在转向更现代的方案。
*   很多高阶的优化插件（Asset Store上的）或者大厂的自研 Unity 版本，会手动接入 Intel 的 MOC 库来实现动态遮挡剔除。
*   Unity 现在更推崇 **GPU-Driven Culling**（利用 Compute Shader 在 GPU 端做剔除，但这需要较新的硬件支持）。

---

### 6. 总结：什么时候该用 SOC？

如果你是技术负责人，在做性能决策时，以下情况最适合引入 Software Occlusion Culling：

1.  **场景是动态的**：比如战场类游戏，很多可破坏建筑，或者巨型的移动载具（战舰、攻城车）。 PVS 对此无能为力。
2.  **GPU 瓶颈（GPU Bound）**：你的场景美术画质极高，Vertex/Fillrate 压力巨大，但 CPU 相对空闲。这时候用空闲的 CPU 换取 GPU 的喘息空间是非常划算的。
3.  **不是大平原**：SOC 依然基于遮挡。如果是一望无际的草原，谁也挡不住谁，SOC 也是白白浪费 CPU 算力（通常会有 AABB 检测，如果都没挡住就不光栅化了，但还是有开销）。

简单说：**SOC 就是让 CPU 充当“保安”，拿着一张模糊的照片（低分深度图），在数据进入 GPU “舞池”之前，把没资格入场的（被挡住的）统统拦下来。**
