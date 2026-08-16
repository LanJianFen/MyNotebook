

这 7 个指令，堪称图形程序员手里的**“七种武器”**。

如果把显卡比作一个高度自动化的超级工厂，这 7 个指令就是你越过厂长（Unity 引擎默认流程），直接给车间下达的微操命令。

为了让你彻底吃透它们，我把它们按**真实的渲染流水线顺序**分成了 4 大类。我们一个个来扒开它们的底裤：

---

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
