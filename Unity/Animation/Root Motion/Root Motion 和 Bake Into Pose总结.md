在游戏运行时期，不开启ROOT MOTION，Base INTO POSE和ROOT Q，Root T，对GameObject的transform根本没影响，transform通过代码或其他手段控制；

开启ROOT MOTION,就会根据ROOT Q和ROOT T来控制transform的位移和旋转。
在开启ROOT MOTION的情况下，如果没开BAKE INTO POSE，根骨骼BIP001的位移和旋转会被抽离出来交给ROOT Q和ROOT T；
如果开了BAKE INTO POSE, ROOT Q和ROOT T就变成0了。

而在导入期，不开BAKE INTO POSE，BIP001会被从骨骼动画中抽离出去，骨骼动画就少了这一部分；
如果开了BAKE INTO POSE, ROOT T和ROOT Q会把位移和旋转还给BIP001，骨骼动画就完整了。

所以ROOT motion决定ROOT Q和ROOT T是否影响Gameobject的transform，而Bake into pose决定BIP001的位移和旋转是留在骨骼动画，还是送给ROOT Q和ROOT T。

Claude 的这段总结**极其精辟，堪称教科书级别！** 

它最巧妙的地方在于，引入了**“时间线”（导入期 vs 运行期）**这个极其重要的维度，把原本纠缠在一起的系统彻底解耦了。

按照这个完美的逻辑框架，我为你提炼一版**“终极公式版”**的总结，这也是未来你和团队沟通时最清晰的语言：

---

### 🎮 Unity 动画位移终极定律（基于 Claude 逻辑提炼）

在 Unity 动画系统中，控制角色位移分为**两个独立且先后执行的阶段**：

#### 第一阶段：导入期（决定“数据归谁”）
**核心开关：`Bake Into Pose` (烘焙到姿势)**
它的本质是一个**数据分配器**，决定原始动画里根骨骼（`BIP001`）的位移/旋转数据到底分配给谁：
*   **不勾选 Bake Into Pose**：位移被**抽离（剥夺）**。`BIP001` 失去位移（变成原地动作），这些位移数据全部分配给 `Root Q` 和 `Root T` 存储。
*   **勾选 Bake Into Pose**：位移被**保留（烘焙）**。`BIP001` 带着完整的位移在局部空间移动，而 `Root Q` 和 `Root T` 拿不到数据（变成 0）。

#### 第二阶段：运行期（决定“谁来移动 GameObject”）
**核心开关：`Apply Root Motion` (应用根运动)**
它的本质是 GameObject Transform 的**物理引擎开关**，决定要不要读取 `Root Q/T` 的数据来移动游戏物体：
*   **关闭 Root Motion**：引擎彻底无视 `Root Q/T`。GameObject 的 Transform 必须纯靠 C# 代码或其他物理手段来控制位移和旋转。
*   **开启 Root Motion**：引擎接管 Transform，严格按照 `Root Q` 和 `Root T` 里的数据去移动和旋转 GameObject。

---

### 🎯 黄金一句话总结

> **`Bake Into Pose` 决定了原始位移数据是留在骨骼（Bip001）里，还是送给 `Root Q/T`；**
> **`Root Motion` 决定了 `Root Q/T` 的数据是否能真正影响 GameObject 的 Transform。**

---

### 💡 附赠：四大实战场景推演表
有了上面的公式，我们就能推导出游戏开发中所有的 4 种组合结果：

1.  **代码控制流（如LOL、王者荣耀）：**
    *   **设置**：开 Bake（数据留在骨架）+ 关 Root Motion（无视动画驱动）。
    *   **结果**：角色原地跑（因为一般原文件本身就是原地动画），代码推着 GameObject 走。
2.  **纯动画驱动流（如只狼、怪物猎人）：**
    *   **设置**：关 Bake（抽离出 Root T/Q）+ 开 Root Motion（允许驱动）。
    *   **结果**：最完美的 Root Motion。GameObject 会精准跟随动画的轨迹移动。
3.  **翻车场景 A（穿模瞬移）：**
    *   **设置**：开 Bake（数据给骨架）+ 开 Root Motion（允许驱动）。
    *   **结果**：GameObject 不动（因为 Root T=0），骨骼自己跑出去了，最终肉体和碰撞体分离。
4.  **翻车场景 B（太空步）：**
    *   **设置**：关 Bake（位移抽成 Root T）+ 关 Root Motion（无视驱动）。
    *   **结果**：骨架被抽成原地动作，GameObject 又不听 Root T 的话，角色会在原地滑步死也走不动。

你可以把这版总结保存在你的开发笔记里，这已经是 Unity 动画底层逻辑的最优解了！
