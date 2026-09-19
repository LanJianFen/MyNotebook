

太能理解了！URP 的 `RenderFeature` 之所以让人抓狂，是因为它**是一个缝合怪**：
它既有 `ScriptableObject`（数据资产）的超长生命周期，又被强行绑定在了**每帧、每相机**的实时渲染大循环里。而且 Editor（编辑器）和 Runtime（运行时）的表现还有微妙的区别。

为了让你以后写 URP 扩展时拥有“上帝视角”，我们把 Feature 和它手下的 Pass 的生命周期彻底扒光。

我们将它分为**三大阶段**：

---

### 🐣 第一阶段：诞生与装配（初始化阶段）

这个阶段的函数**不是每帧执行的**，它们只在特定时刻触发，专门用来“造东西”和“预计算”。

#### 1. `OnEnable()` / `OnDisable()` (极其边缘，别踩坑)
*   **什么时候调用**：因为 Feature 本质是 `ScriptableObject`，所以只有在**这个资产被加载到内存**（比如刚打开 Unity 工程），或者**从内存卸载**时才会调用。
*   **最大坑点**：你在 Inspector 面板里“取消勾选” Active 框，**绝对不会**触发 `OnDisable()`！
*   **使用建议**：通常留空。除非你要注册一些全局事件（比如 `RenderPipelineManager.beginCameraRendering`），否则别碰它们。

#### 2. `OnValidate()` (Editor 专属神技)
*   **什么时候调用**：**只在 Unity 编辑器下有效**。只要你在 Inspector 面板里修改了哪怕一个数字、切了一个下拉框，或者刚写完代码编译完，它就会瞬间触发。
*   **核心职责**：参数的容错校验、重度数学预计算（比如你的 Jimenez 积分）。
*   **注意**：打包出 APK/EXE 后，这个函数**直接灰飞烟灭**。

#### 3. `Create()` (整个 Feature 的绝对核心)
*   **什么时候调用**：
    1. 游戏刚启动，URP 管线初始化时。
    2. **只要面板上的参数发生任何改变（包括勾选/取消勾选），URP 会立刻重新调用 `Create()`**（底层逻辑是 URP 会销毁旧的渲染管线图并重建）。
*   **核心职责**：**当工厂！** 在这里 `new` 你的 RenderPass，实例化 Material (`CoreUtils.CreateEngineMaterial`)。
*   **黄金准则**：不要在这里做任何跟帧渲染相关的操作（比如申请 RT、绑全局变量），这里只负责把“打仗的兵（Pass）”和“武器（Material）”造出来。

---

### ⚔️ 第二阶段：每帧战斗（渲染循环阶段）

这个阶段的函数是**每帧、每个相机**都会执行的！你的游戏跑 60 帧，同屏有 1 个主相机和 1 个 UI 相机，那下面这些函数每秒就会执行 120 次！**性能极度敏感！**

#### 4. `AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)`
*   **什么时候调用**：每帧、每个相机在决定“今天我要画哪些东西”时调用。
*   **核心职责**：**当调度员！** 
    1. 判断当前帧要不要渲染这个特效？（比如查一下 `globalMaximumLOD`，或者查一下 `renderingData.cameraData.cameraType` 是不是 SceneView）。
    2. 如果条件允许，调用 `renderer.EnqueuePass(_pass)` 把造好的 Pass 推入渲染队列。
*   **黄金准则**：千万**不要**在这里 `new` 任何东西！只做 `if` 判断和 `EnqueuePass`。代码越少越好，就像你最后那个精简版一样。

*(注：URP 14+ 新增了一个 `SetupRenderPasses`，用来在入队后做一些 RT 依赖的预配置，普通特效用不到，了解即可)*

---

### 💀 第三阶段：销毁与清理（死亡阶段）

#### 5. `Dispose(bool disposing)`
*   **什么时候调用**：
    1. 你把这个 Feature 从 Renderer 面板的列表里点 `-` 号**彻底删掉**时。
    2. 切换了 URP 管线资产（高低画质管线切换）时。
    3. 退出游戏关闭引擎时。
*   **核心职责**：**擦屁股！**
    你用 `CoreUtils.CreateEngineMaterial` 借来的材质、你在 Pass 里生成的那些一直没释放的持久 RT，统统都要在这里 `Destroy()` 和 `Release()`。
*   **最大坑点**：如果没有正确清理，材质会一直挂在内存里，导致**内存泄漏（Memory Leak）**，在 Unity 编辑器里表现为你退出 Play 模式后，依然能看到某些报错或者内存没降下去。

---

### 🔄 附赠：Render Pass 的生命周期小抄

既然 Feature 把 Pass 推入了队列（`EnqueuePass`），那 Pass 自己在这一帧里是怎么干活的？
顺序严格如下：

1.  **`OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)`**
    *   **时机**：在这颗镜头真正开始画几何体之前。
    *   **干嘛**：申请你要用的 RenderTarget（`ReAllocateIfNeeded`），配置你要渲染到哪里（`ConfigureTarget`）。
2.  **`Execute(ScriptableRenderContext context, ref RenderingData renderingData)`**
    *   **时机**：到达了你配置的 `renderPassEvent` 时机（比如 Opaque 之后）。
    *   **干嘛**：纯纯的 GPU 调教。写 `CommandBuffer`，调 `Blit`，调 `DrawFullScreen`，最后 `context.ExecuteCommandBuffer(cmd)`。
3.  **`OnCameraCleanup(CommandBuffer cmd)`**
    *   **时机**：这颗镜头画完收工了。
    *   **干嘛**：如果你在 `OnCameraSetup` 里申请的是 `GetTemporaryRT`（临时 RT），必须在这里 `ReleaseTemporaryRT` 把它还给池子。*(因为你用的是持久化的 `RTHandle`，所以你不需要写这个函数，这也是 `RTHandle` 架构更先进的原因)*。

---

### 🗺️ 终极全景图（一张图印在脑子里）

假设你现在在编辑器里，**点下了 Play 按钮，运行了一帧，然后退出：**

```text
【游戏启动/数据加载】
 ├── OnEnable()       <-- 资产被加载
 └── Create()         <-- 造出 Pass 和 Material

【第 1 帧开始】 (主相机)
 ├── Feature.AddRenderPasses()   <-- 判断并入队 EnqueuePass
 │
 │   (URP 渲染大车轮滚到你设定的 Event 时机)
 ├── Pass.OnCameraSetup()        <-- 申请 RT
 ├── Pass.Execute()              <-- 真正向 GPU 发出 DrawCall
 └── Pass.OnCameraCleanup()      <-- 归还临时 RT

【第 2 帧开始】... 重复上述过程

【退出游戏/销毁】
 ├── Feature.Dispose()           <-- 释放材质和 Pass 里的 RTHandle
 └── OnDisable()                 <-- 资产被卸载
```

你之前之所以纠缠，是因为把 **“资产的生命周期（Create/Dispose）”** 和 **“帧的生命周期（Add/Execute）”** 混在了一起，还要加上 Editor 面板操作的干扰。

现在把这个全景图对照着你写的那份“终极版代码”看，是不是感觉每一行代码为什么待在那个位置，都有了极其严密的逻辑闭环？
