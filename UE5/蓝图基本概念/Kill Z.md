

Kill Z 是在 **World Settings**（世界场景设置）里查看和修改的。

### 查找步骤：

1.  **打开面板**：
    *   看编辑器界面的右下角（默认通常和 Details 面板在一起，有一个标签页叫 `World Settings`）。
    *   如果你没找到，看最上方菜单栏：点击 **Window**（窗口） -> 勾选 **World Settings**。

2.  **搜索 Kill Z**：
    *   在 World Settings 面板顶部的搜索框里，输入 **`Kill`**。

3.  **找到设置**：
    *   你会看到一项叫 **`Kill Z`**（在老版本 UE4 里）。
    *   **⚠️ 注意：在 UE5 中，这个名字改了！**
    *   在 **UE5** 里，它叫 **`World Bounds Checks`** 下面的 **`Kill Z`** 或者直接被重构进了 **`Enable World Bounds Checks`**。
    *   **UE5 新名字**：直接搜 **`Kill`** 应该能看到 **`Kill Z`** 属性，默认值通常是 **`-1048575.0`** 或者 **`-1000.0`**（取决于模板）。
    *   *注：在 UE5.1+ 的部分模板中，这个设置可能被移动到了 Project Settings -> Engine -> Physics 里，或者作为关卡默认属性存在。*

    **最稳妥的确认当前关卡 Kill Z 的方法（通用）：**
    看 World Settings 里 **Search** -> 输入 **`Kill Z`**。
    如果没搜到，搜 **`Bounds`**。

### 数值含义
*   默认值通常是 **-1000.0** cm（也就是地板下 10 米）。
*   任何 Actor 的坐标如果 **Z < -1000**，就会被引擎强制 Destroy。

### 如何快速测试是不是 Kill Z 的锅？
你可以临时把这个值改成一个超级大的负数，比如 **-999999**。如果改完后，你在空中 Jump 被撞就不销毁了，那就铁定是它干的。（测试完记得改回来，或者用我上面说的 `Set Movement Mode: None` 来治本）。
