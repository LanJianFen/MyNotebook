
## Command Buffer API 
### 第一步：架设画板与清洗（铺场）

虽然在 URP 中我们推荐在 `OnCameraSetup` 里用 `ConfigureTarget` 架设目标，但在某些复杂情况（比如在一个 Pass 里要反复交替画好几张图，俗称 Ping-Pong 模糊），我们就必须在 `Execute` 里手动用 `cmd` 切换画板。

#### 1. `cmd.SetRenderTarget` (把画板搬上画架)
*   **作用**：强行扭转显卡的输出管道，告诉它：“接下来的画面，别往屏幕上画了，给我画到这张指定的 RT 上！”
*   **用法**：
    ```csharp
    // 把输出目标对准 myCustomRT，并且不需要深度图
    cmd.SetRenderTarget(myCustomRT); 
    
    // 或者同时绑定颜色和深度
    cmd.SetRenderTarget(colorRT, depthRT);
    ```
*   **内行提示**：一旦调用这个，之前的输出目标就被抛弃了。如果你想切回屏幕，得再次调用 `cmd.SetRenderTarget(cameraColorTarget)`。

#### 2. `cmd.ClearRenderTarget` (拿抹布擦画板)
*   **作用**：把刚刚绑定好的 RT 擦干净。
*   **用法**：
    ```csharp
    // 参数：(是否清空深度, 是否清空颜色, 清空成什么颜色)
    cmd.ClearRenderTarget(true, true, Color.black);
    ```
*   **内行提示**：**极其耗费性能！** 除非你真的需要一张纯净的背景（比如造一张全新的纯黑遮罩图），否则后处理叠加时千万别调用它，不然底图就没了。

---

### 第二步：篡改空间法则（黑魔法）

#### 3. `cmd.SetViewProjectionMatrices` (篡改摄像机)
*   **作用**：强行改写传给 Shader 的 `UNITY_MATRIX_V` (视图矩阵) 和 `UNITY_MATRIX_P` (投影矩阵)。
*   **什么场景用？**
    *   **做传送门/镜子**：你可以把矩阵改成“镜子背后的虚拟摄像机”，画出来的场景就是镜像世界。
    *   **全屏特效的“降维打击”**：你那个作者写了 `cmd.SetViewProjectionMatrices(Matrix4x4.identity, Matrix4x4.identity);`。`identity` 是单位矩阵（就是全部归零的初始状态）。这等于把 3D 摄像机直接砸烂，把 3D 世界强行降维成一个绝对平面的 2D 坐标系，这样画全屏三角形时，就不受摄像机距离、FOV 视角的任何干扰，严丝合缝地贴在屏幕上！
*   **用法**：
    ```csharp
    // 篡改矩阵
    cmd.SetViewProjectionMatrices(Matrix4x4.identity, Matrix4x4.identity);
    
    // 【警告】干完坏事必须恢复现场！否则后面的东西全画错！
    cmd.SetViewProjectionMatrices(camera.worldToCameraMatrix, camera.projectionMatrix);
    ```

---

### 第三步：疯狂作画（核心输出）

这里有三种画法，代表了三个不同的时代和用途：

#### 4. `cmd.DrawMesh` (老派硬核画法)
*   **作用**：把一个真实的 3D 模型网格，用指定的材质，画在指定的空间位置。
*   **什么场景用？** 不想让模型挂在场景里（不想建 GameObject），纯靠代码凭空生成。比如脚下的技能范围指示圈、鼠标点击的光标、或者像作者那样强行传一个面片（`RenderingUtils.fullscreenMesh`）来做全屏特效。
*   **用法**：
    ```csharp
    // 参数：(网格数据, 空间位置矩阵, 材质, submesh索引, Shader里第几个Pass)
    cmd.DrawMesh(myMesh, Matrix4x4.identity, myMaterial, 0, 0);
    ```

#### 5. `cmd.DrawProcedural` (现代魔法画法)
*   **作用**：**不传任何 Mesh 数据！** 直接命令 GPU：“去执行顶点着色器 3 次！” 顶点到底在哪，全靠 Shader 里的魔法函数凭空算（就是我教你的 `SV_VertexID` 算大三角形）。
*   **优势**：极度节省 CPU 到 GPU 的数据传输带宽。这是目前 URP 官方做全屏特效的最底层基石（`CoreUtils.DrawFullScreen` 的本体）。
*   **用法**：
    ```csharp
    // 参数：(位置, 材质, Pass索引, 画什么拓扑图形, 画几个顶点, 画几个实例)
    // MeshTopology.Triangles, 3 表示：画1个三角形（3个顶点）
    cmd.DrawProcedural(Matrix4x4.identity, myMaterial, 0, MeshTopology.Triangles, 3, 1);
    ```

#### 6. `cmd.Blit` (经典滤镜画法)
*   **作用**：图形学中最经典的词汇（Block Image Transfer）。意思是：把 A 图通过一个 Shader 滤镜，复印到 B 图上。
*   **用法**：
    ```csharp
    // 把 sourceRT 塞进 material，处理完输出到 destRT
    cmd.Blit(sourceRT, destRT, material, 0); 
    ```
*   **🔥 极其重要的现代 URP 避坑指南**：
    如果你现在还在用 `cmd.Blit`，在某些手机设备或者开启了抗锯齿时，**你会发现画面上下颠倒了！** 这是因为 Unity 跨平台底层 API 的历史遗留 Bug。
    **现代解法**：在 URP 中，官方强烈建议把 `cmd.Blit` 换成 URP 专属的：
    `Blitter.BlitCameraTexture(cmd, source, dest, material, pass);`
    它在底层完美修复了所有画面翻转的破事。

---

### 第四步：极速搬运（物流）

#### 7. `cmd.CopyTexture` (内存瞬移)
*   **作用**：纯粹的显存到显存的数据拷贝。**不经过任何 Shader，不经过任何顶点处理！**
*   **速度**：快到离谱！比 `Blit` 快几十倍。
*   **限制条件极其苛刻**：A 图和 B 图的长宽、色彩格式（Format）、甚至抗锯齿设置，**必须 100% 绝对一模一样**，否则直接报错。
*   **什么场景用？**
    *   **时间抗锯齿 (TAA) / 残影特效**：在这一帧结束前，把当前的完美画面 `CopyTexture` 到一张叫 `_HistoryRT` 的图里。下一帧的时候，拿出来跟新画面混合。
    *   **保存干净的底图**：在 UI 画上去之前，把场景拷贝下来，然后拿去做高斯模糊，作为 UI 的毛玻璃背景。
*   **用法**：
    ```csharp
    cmd.CopyTexture(sourceRT, destRT);
    ```

---

### 一套神级连招示范（实战模拟）

假设我们现在要做一个**“眩晕残影 + 屏幕模糊”**的特效，把这几个命令串起来就是这样：

```csharp
// 1. (准备) 绑定一个临时模糊画布，并擦干净
cmd.SetRenderTarget(blurRT);
cmd.ClearRenderTarget(false, true, Color.black);

// 2. (作画) 把当前相机屏幕，通过模糊材质，画到模糊画布上
// 使用现代版的 Blit 替代 cmd.Blit 防止翻转
Blitter.BlitCameraTexture(cmd, cameraColorTarget, blurRT, blurMaterial, 0);

// 3. (物流) 把模糊好的图，极速拷贝到“上一帧历史图”里，留给下一帧用
cmd.CopyTexture(blurRT, historyRT);

// 4. (复原) 把画板切回主屏幕
cmd.SetRenderTarget(cameraColorTarget);

// 5. (黑魔法+现代作画) 砸烂矩阵，用大三角形把混合了残影的最终画面，拍到屏幕上
cmd.SetViewProjectionMatrices(Matrix4x4.identity, Matrix4x4.identity);
cmd.DrawProcedural(Matrix4x4.identity, finalBlendMaterial, 0, MeshTopology.Triangles, 3, 1);

// 恢复相机的正常视角矩阵
cmd.SetViewProjectionMatrices(cameraView, cameraProj);
```

这就是图形学大牛写出来的流水线。当你把这 7 个 API 烂熟于心，你就能像搭乐高积木一样，组合出世界上任何一种牛逼的渲染管线架构！

---


## Set Render Target 的用法

当你开始关注 `SetRenderTarget` 后面的参数时，说明你已经**从单纯的“实现效果”，跨入到了“榨干显卡性能”的顶级架构师领域。**

在 Unity 的底层，`cmd.SetRenderTarget` 有很多重载版本，除了最基础的传入 `colorRT` 和 `depthRT` 之外，后面的参数主要分为**两大核弹级类别**：
一是**控制内存搬运的 `Load/Store Action`**（关乎手机会不会发热掉帧）；
二是**控制特殊维度的 `MipLevel/Cubemap/Slice`**（关乎高级渲染特效）。

我们把它们扒得干干净净：

---

### 第一类：移动端性能的“生死符” (Load/Store Action)

在现代重载中，你会看到这样长长的一串参数：
```csharp
cmd.SetRenderTarget(
    colorRT, 
    RenderBufferLoadAction.DontCare,  // 颜色的加载动作
    RenderBufferStoreAction.Store,    // 颜色的保存动作
    depthRT, 
    RenderBufferLoadAction.Clear,     // 深度的加载动作
    RenderBufferStoreAction.DontCare  // 深度的保存动作
);
```
**背景硬核知识**：现代手机 GPU 采用的是 **TBDR（基于图块的延迟渲染）** 架构。显存分为“慢速的主显存”和“极速的芯片内缓存（Tile Memory）”。把数据从主显存搬到极速缓存是非常耗电、极其卡顿的！

这些参数就是你命令 GPU **“要不要搬运数据”** 的圣旨。

#### 1. `RenderBufferLoadAction` (画板放上画架前，你要干嘛？)
*   **`Load` (加载)**：命令 GPU 把上一帧或上一个 Pass 画好的画面，从慢速显存**完整地搬到**高速缓存里。
    *   *何时用*：比如你在画半透明物体，你需要底下已经画好的背景图来做混合，必须用 Load。
*   **`Clear` (清空)**：命令 GPU 别搬了，直接在高速缓存里给我造一块纯黑/纯白的干净画板。
    *   *🔥 顶级避坑指南*：**千万别再用 `cmd.ClearRenderTarget()` 了！** 那是老掉牙的写法！在现代 URP 中，想要清空画板，直接在这里填 `Clear`！这样不走任何内存搬运，性能极高！
*   **`DontCare` (无所谓/丢弃)**：命令 GPU：“我不管高速缓存里现在是什么垃圾数据，直接给我用，反正我等下会用大三角形把整个屏幕全部重新涂满！”
    *   *何时用*：做全屏覆盖的特效（比如全屏泛光、复制贴图）。**这是性能最高的选项！**

#### 2. `RenderBufferStoreAction` (画完之后，你要干嘛？)
*   **`Store` (保存)**：命令 GPU 把在极速缓存里画好的成果，**写回到**慢速主显存中，留给下一个 Pass 用，或者展示在屏幕上。
    *   *何时用*：你需要这张图里的结果时（90% 的颜色图都需要 Store）。
*   **`DontCare` (无所谓/扔掉)**：画完直接扔进垃圾桶，绝对不写回主显存！
    *   *何时用（神级优化）*：**深度图（DepthRT）的绝佳归宿！** 比如你做了一个临时 Pass，你需要深度图来确保 3D 模型的前后遮挡正确，但画完之后，这张深度图你再也不需要了。如果你填 `Store`，手机会强行把这几 MB 的深度图写回显存，狂耗带宽；填 `DontCare`，直接省下一大半性能！

---

### 第二类：维度打击 (MipLevel / CubemapFace / DepthSlice)

如果你在做一些非常骚的高级特效（比如 Bloom泛光、焦散、反射探针），你会用到这类重载：
```csharp
cmd.SetRenderTarget(
    colorRT, 
    mipLevel: 2,           // 渲染到第几个 Mipmap 层级？
    cubemapFace: CubemapFace.PositiveX, // 渲染到天空盒的哪一个面？
    depthSlice: 0          // 渲染到纹理数组的第几层？
);
```

#### 1. `mipLevel` (多级渐远纹理层级)
*   **是什么**：一张 1024x1024 的图是 Mip 0；缩小一半，512x512 就是 Mip 1；256x256 是 Mip 2。
*   **怎么用**：做 **Bloom（泛光）** 或 **景深（DOF）** 时。
    *   如果不用 `mipLevel`，你需要手动创建五六张越来越小的 RenderTexture，极其浪费内存。
    *   **高阶做法**：只创建一张开启了 `useMipMap = true` 的大贴图。然后通过 `cmd.SetRenderTarget(myRT, mipLevel: 1)`，直接命令显卡**画到这张贴图的第二层（缩小版）上！** 这叫做降采样（Down-sampling）。

#### 2. `cubemapFace` (天空盒面)
*   **是什么**：一个 Cubemap（天空盒/反射球）有 6 个面（上下左右前后）。
*   **怎么用**：做**汽车后视镜**、**实时水面反射**，或者烘焙反射探针。
    *   你把摄像机摆在汽车中心，朝右看，然后 `cmd.SetRenderTarget(cubeRT, CubemapFace.PositiveX)`，画一帧。
    *   朝上看，画到 `PositiveY`... 重复 6 次，一张完美的 360 度实时反射图就诞生了。

#### 3. `depthSlice` (纹理数组的切片)
*   **是什么**：Texture2DArray 是一种把几十张一样大的图片像千层饼一样叠在一起的特殊贴图。
*   **怎么用**：做**阴影级联（Cascaded Shadows）**、**地形的多层地表纹理**。
    *   传入 `depthSlice: 0`，就是画千层饼的第一层。
    *   它最大的好处是，不管画多少层，Shader 里只需要绑定**一次**贴图变量，然后用三维坐标 `(x, y, 饼层数)` 就能去采样，极大地突破了移动端只能绑定 16 张贴图的硬件限制。

---

### 工程师总结：如何优雅地使用它们？

给你一个极其标准的、性能拉满的**“渲染临时 3D 模型”**的 `SetRenderTarget` 模板：

```csharp
// 需求：我在一张独立的 UIRT 上画一个 3D 角色展示，并且需要剔除他背后的遮挡物，画完之后深度图不需要了。

cmd.SetRenderTarget(
    m_UICharacterRT, 
    RenderBufferLoadAction.Clear,   // 颜色：不需要老数据，直接帮我清屏成纯透明色
    RenderBufferStoreAction.Store,  // 颜色：画完后必须要存下来，因为UI系统等下要拿它显示在屏幕上
    m_DepthRT, 
    RenderBufferLoadAction.Clear,   // 深度：不需要老深度，直接清空，给这个角色重新算深度
    RenderBufferStoreAction.DontCare // 深度：【神级优化】画完角色后，这张深度图就没用了，直接扔掉，不要占用主显存带宽！
);

// 然后再执行画模型的指令...
context.DrawRenderers(...);
```

当你把这些 `Load/Store` 参数写对的时候，哪怕画面看起来没有任何区别，但如果在移动端连上性能测试工具（如 Snapdragon Profiler），你会发现显卡的总线带宽消耗瞬间**暴降 30% 以上**！这就是主程序和技术美术不可替代的价值。
