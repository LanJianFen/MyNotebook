

这是一个非常硬核且直击本质的问题！很多有经验的开发者都在这里栽过跟头。

直接回答你的问题：**既不是仅仅看项目里有没有 Material 文件，也不完全是看场景里有没有 GameObject 使用。**

真正的判定标准是：**这个 Material 文件是否被“打包进了最终的游戏包（Build）中”。**

为了让你彻底弄懂这个机制，我们必须理解 Unity 打包时的**依赖引用收集机制（Dependency Tracking）**。

---

### 深入解析：Unity 的打包与剔除逻辑

当你点击 Build 打包时，Unity **不会**把你 `Assets` 文件夹下的所有东西都塞进包里。它会像顺藤摸瓜一样，建立一个引用树：

1.  Unity 首先去看 **Build Settings** 里勾选了哪些 Scene（场景）。
2.  进入这些 Scene，检查里面所有的 GameObject。
3.  检查这些 GameObject 挂载了什么 Mesh、什么 Material。
4.  检查这些 Material 勾选了什么宏（Keywords）。
5.  **额外检查：** 检查 `Resources` 文件夹、`StreamingAssets` 文件夹，以及被打包进 AssetBundle / Addressables 的资源。

基于这个顺藤摸瓜的逻辑，我们来看看几种具体情况，到底会不会被剔除：

#### 情况一：材质文件只存在于项目中（不在场景，不在 Resources）
*   **操作：** 美术在 `Assets/Materials/` 下建了一个发光的 Boss 材质，打上了勾。但是策划说这个 Boss 这次测试不上，于是把 Boss 从所有的场景里**删掉**了。
*   **结果：变体被剔除！**
*   **原因：** 虽然项目里有这个 `.mat` 文件，但 Unity 顺藤摸瓜时摸不到它。Unity 认为这个材质是“废弃资产”，根本不会把它打进包里，自然也不会去读取它的发光宏。此时如果用 C# 强行开启发光，必定紫屏/黑屏。

#### 情况二：材质被场景中的 GameObject 使用（常规情况）
*   **操作：** 场景里的 Boss 模型挂载了这个发光材质，且这个场景被加入了 Build Settings。
*   **结果：变体保留！**
*   **原因：** Scene -> Boss (GameObject) -> Boss_Mat (Material) -> 读取到 `USE_EMISSION`。Unity 成功收集到这个变体需求，将其编译入包。

#### 情况三：材质没在场景里，但是放在了 `Resources` 文件夹中
*   **操作：** 场景里没有 Boss。但是你把发光材质放到了 `Assets/Resources/` 目录下。
*   **结果：变体保留！**
*   **原因：** Unity 的打包规则是：`Resources` 文件夹下的所有资源，**无视是否被引用**，强制全部打包。因此，Unity 也会强制读取它的宏，生成发光变体。

---

### 高级工程师的避坑与优化指南

基于上面的底层逻辑，如果你真的想在运行时用 C# 动态开启一个 `shader_feature` 宏（比如小兵半血发光），你应该怎么做来防止它被剔除呢？

这里有三种做法，按推荐程度从低到高排列：

#### 做法 1：建假材质并塞入 `Resources`（勉强能用，但不推荐）
像前面提到的，建一个发光材质扔进 `Resources`。
*   **缺点：** 污染了 `Resources` 文件夹，会导致游戏启动变慢，且不易管理。

#### 做法 2：建假材质并挂载在场景的隐藏物体上（新手常用）
建一个发光材质，在游戏的 Loading 场景里建一个空的 GameObject（或者藏在地下的方块），把材质挂给它。
*   **缺点：** 场景里会有莫名其妙的隐藏物体，团队里的其他人可能会误删它，导致难以排查的 Bug。

#### 做法 3：使用 Shader Variant Collection (SVC) （最专业、最标准的做法）
这是 Unity 专门为了解决这个问题提供的官方工具，**完全不需要建假材质**。

1.  在 Project 窗口右键 -> `Create` -> `Shader` -> `Shader Variant Collection`。
2.  双击打开它，点击 `+` 号，把你写的 `MechaShader` 加进去。
3.  在列表中手动添加你需要的变体组合，比如把 `USE_EMISSION` 选上。
4.  打开 `Project Settings` -> `Graphics`，拉到最下面找到 **Preloaded Shaders**。
5.  把你刚才创建的 SVC 文件拖进去。

**底层原理：**
一旦你把 SVC 放入 `Preloaded Shaders`，当你打包时，Unity 会直接去读取这个列表。它相当于拿到了一份**“保命白名单”**。Unity 会说：“不管场景里有没有材质用到这个发光宏，既然它在 SVC 名单里，我就**必须**把它编译进包里，绝对不剔除！”

### 总结
`shader_feature` 的剔除判定，本质上是看**带有该宏的材质球是否被卷入了最终的游戏包中**。

为了确保动态切换的宏不被意外剔除，专业的图形程序员不会依赖材质文件存不存在，而是通过 **Shader Variant Collection (SVC)** 来建立明确的变体白名单，牢牢把控编译结果。
