

你说到了一个非常关键的词：**“捕获列表（Capture List）”**。

一听到这个词，我就知道你大概率接触过 **C++**。
在 C++ 里，Lambda 表达式长这样：`[捕获列表](参数) -> 返回值 { 函数体 }`，一切都分得清清楚楚。

但 **C# 的 Lambda 语法哲学完全不同**。C# 追求的是“极简主义和自然推断”。在 C# 中，Lambda 主要由 **`=>`（Lambda 运算符）** 分割成左右两半。

抛开底层的委托类型，我们单刀直入，纯粹从**“怎么写好一个 Lambda”**的角度，为你扒光它的语法细节：

---

### 👈 左半边：参数列表 (Parameters)

C# 的参数列表极其灵活，主打一个“能省则省”。

#### 1. 零参数与单参数（最简形态）
*   **零参数**：必须写空括号 `()`。
    ```csharp
    () => Debug.Log("Hello");
    ```
*   **单参数**：连括号都可以省掉！这是 C# 独有的优雅。
    ```csharp
    // 假设传进来一个 player 对象
    player => player.Health > 0;
    ```

#### 2. 多参数与明确类型（标准形态）
*   **多参数**：必须加括号，用逗号隔开。
    ```csharp
    (x, y) => x + y;
    ```
*   **显式声明类型**：如果编译器推断不出类型，或者为了提高代码可读性，你可以手动把类型加上（一旦加了类型，哪怕只有一个参数也必须加括号）。
    ```csharp
    (int x, float y) => x * y;
    ```

#### 3. 2026 年现代 C# 的高级参数玩法
*   **弃元参数 (`_`)**：当回调传给你参数，但你**根本不用**时，用下划线占位（非常适合规避命名冲突和内存占用）。
    ```csharp
    // 比如网络请求回调传了 (ErrorCode, Response)，但我只管成功逻辑
    (_, response) => ProcessData(response);
    ```
*   **带修饰符的参数**：Lambda 的参数同样支持 `ref`、`in`、`out`，非常适合写高性能的计算代码。
    ```csharp
    (ref Vector3 pos, in Matrix4x4 mat) => pos = mat.MultiplyPoint(pos);
    ```
*   **默认参数 (C# 12+)**：参数可以自带默认值了。
    ```csharp
    (int baseDamage, float multiplier = 1.5f) => baseDamage * multiplier;
    ```

---

### 👉 右半边：返回值与函数体 (Return & Body)

右边决定了这个 Lambda 到底要干嘛，以及返回什么结果。

#### 1. 表达式体 (Expression Body)
如果你的逻辑**只有一行代码**，直接写在 `=>` 后面。
*   **神奇之处**：**不需要写 `return` 关键字，不需要写大括号 `{}`，也不需要写分号 `;`**。C# 会自动把这行代码的计算结果作为返回值！
    ```csharp
    (x, y) => x * y + 10; // 直接返回计算结果
    ```

#### 2. 语句块体 (Statement Body)
如果逻辑很复杂，超过了一行，那就必须老老实实用大括号 `{}` 包起来。
*   **注意**：一旦用了大括号，如果有返回值，你**必须手动写 `return` 关键字**，并且每行结尾要加分号 `;`。
    ```csharp
    (x, y) => 
    {
        int temp = x * 2;
        if (temp > y) 
            return temp;
        return y;
    }
    ```

#### 3. 显式指定返回类型 (C# 10+)
C# 的 Lambda 以前是没有像 C++ 那样 `-> ReturnType` 的语法的，全靠编译器自己猜。但在现代 C# 里，如果你遇到三元运算符推断不出类型（比如返回不同的子类），你可以**在参数括号前面强制声明返回类型**：
```csharp
// 强制告诉编译器：这个 Lambda 返回的是 object，不是 string 或 null
object (bool flag) => flag ? "Success" : null; 
```

---

### 🧲 灵魂所在：捕获列表 (Capture / Closure)

这是 C# 和 C++ 最大的区别：
**C++ 的捕获是显式的（`[=]` 或 `[&]`），而 C# 的捕获是隐式的、全自动的，且全是“引用捕获”！**

在 C# 中，**根本没有捕获列表的语法**。你在 Lambda 大括号里面直接敲出了外面的变量名，C# 编译器就会默默地把它“捕获”。

#### 1. 隐式捕获（默认行为，危险）
```csharp
int multiplier = 2;

// 你没有写任何捕获列表，但你直接用了 multiplier，它就被捕获了！
var calc = (int x) => x * multiplier; 
```

#### 2. C# 版本的 `[]`（空捕获列表）—— `static` 关键字
正因为隐式捕获太容易一不小心把外面的变量吸进来（导致 GC Alloc），所以现代 C# 提供了一个极其暴力的关键字：**`static`**。

在 Lambda 最前面加上 `static`，就等同于 C++ 里的 **空捕获列表 `[]`**。它警告编译器：“这个 Lambda 绝对不允许捕获外面的任何东西！”

```csharp
int multiplier = 2;

// ❌ 报错！编译器会拦住你：静态 Lambda 不能捕获外部变量 'multiplier'。
var calc = static (int x) => x * multiplier; 

// ✅ 正确写法：只能用传进来的参数，或者类内部的静态变量。0 GC 保证！
var safeCalc = static (int x, int mult) => x * mult;
```

---

### 🎨 总结：看图默写

给你一个现代 C# (2026) 包含所有语法的终极 Lambda 表达式结构：

```csharp
// [返回类型] ( [修饰符] 参数类型 参数名 [= 默认值] ) => { 逻辑体; return 结果; }
  
var magicLambda = string (ref int count, bool force = false) => 
{
    if (force) count = 0;
    count++;
    return $"Count is {count}";
};
```

**写 Lambda 的心法口诀：**
1.  **左边能省则省**：单参数不写括号，能推断不写类型。
2.  **右边能短则短**：一行代码不写大括号，不写 `return`。
3.  **高频执行防爆炸**：只要是放在 `Update` 或者循环里，左边必定加 `static`！
