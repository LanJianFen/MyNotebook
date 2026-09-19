# Google C# 变量命名规范

## 核心原则

**存储修饰符（`static` / `const` / `readonly`）不影响命名，命名只由"访问权限"决定。**

- `public` → `PascalCase`
- `private` / `protected` / `internal` → `_camelCase`（下划线前缀 + 小驼峰）

## 快速对照表

| 元素 | 命名 | 例子 |
|---|---|---|
| 类、结构、枚举、委托 | `PascalCase` | `MyClass`, `RenderMode` |
| 接口 | `I` + `PascalCase` | `IDisposable` |
| 方法 | `PascalCase` | `CalculateValue` |
| 属性 | `PascalCase` | `Score` |
| 枚举成员 | `PascalCase` | `Yes`, `Running` |
| **`public` 字段** | `PascalCase` | `public int Foo;` |
| **非 `public` 字段** | `_camelCase` | `private int _hitCount;` |
| 局部变量 | `camelCase` | `var resultValue = 5;` |
| 参数 | `camelCase` | `int mulNumber` |
| 泛型参数 | `T` + `PascalCase` | `T`, `TResult`, `TKey` |
| 命名空间 | `PascalCase` | `MyNamespace` |

---

## 访问修饰符 × 存储修饰符组合矩阵

这是最容易搞混的地方。**记住一句话：命名只跟访问权限走，不看 static / const / readonly。**

| 组合 | 命名规则 | 例子 |
|---|---|---|
| `public` | `PascalCase` | `public int Foo;` |
| `public static` | `PascalCase` | `public static int NumTimesCalled;` |
| `public const` | `PascalCase` | `public const int MaxRetries = 3;` |
| `public static readonly` | `PascalCase` | `public static readonly int MagicNumber;` |
| `public readonly` | `PascalCase` | `public readonly int Id;` |
| `private` | `_camelCase` | `private int _hitCount;` |
| `private static` | `_camelCase` | `private static Random _rng;` |
| `private const` | `_camelCase` | `private const int _bar = 100;` |
| `private static readonly` | `_camelCase` | `private static readonly int _shaderId;` |
| `private readonly` | `_camelCase` | `private readonly List<int> _cache;` |
| `protected` / `internal` | `_camelCase`（同 `private`） | `protected int _state;` |

### 常见误区

❌ **误区 1：给 static 加 `s_` 前缀**
```csharp
private static int s_count;   // Google 风格不这么写
```
✅ 正确：
```csharp
private static int _count;
```

❌ **误区 2：给 const 加 `k_` 前缀或全大写**
```csharp
private const int k_MaxSize = 100;    // C++ 派做法
private const int MAX_SIZE = 100;      // Java 派做法
```
✅ 正确：
```csharp
private const int _maxSize = 100;      // 私有 const 就跟私有字段一样
public const int MaxSize = 100;        // 公开 const 就跟公开字段一样
```

❌ **误区 3：给 readonly 特殊命名**
```csharp
private readonly int mReadOnlyValue;   // Hungarian 风格
```
✅ 正确：
```csharp
private readonly int _readOnlyValue;
```

---

## 详细说明

### 1. 类、接口、枚举

- 类：`PascalCase`
- 嵌套类：`PascalCase`（不需要特殊前缀）
- 接口：`I` + `PascalCase`
- 枚举本身：`PascalCase`
- 枚举成员：`PascalCase`

```csharp
public interface IReadable { }

public enum State
{
    Idle,
    Running,
    Finished,
}

public class OuterClass
{
    private class InnerClass { }
}
```

### 2. 字段（Field）

字段命名的**唯一决定因素**是访问权限：

```csharp
public class MyClass
{
    // ── public 字段：PascalCase ────
    public int Foo;
    public bool NoCounting;
    public static int NumTimesCalled;
    public const int MaxSize = 100;
    public static readonly int GlobalCounter;

    // ── 非 public 字段：_camelCase ─
    private int _hitCount;
    protected string _name;
    internal float _speed;
    private static Random _rng;
    private const int _bar = 100;
    private static readonly int _shaderId;
    private readonly List<int> _cache;
}
```

### 3. 属性

属性一律 `PascalCase`，不管可见性：

```csharp
public int Score { get; set; }
private int InternalCounter { get; set; }
```

### 4. 方法

一律 `PascalCase`，不管可见性：

```csharp
public void DoSomething() { }
private void _doInternal() { }   // ❌ 错，方法不用下划线前缀
private void DoInternal() { }    // ✅ 对
```

### 5. 局部变量、参数

一律 `camelCase`：

```csharp
public int Calculate(int mulNumber, int addValue)   // 参数
{
    var resultValue = mulNumber * 2 + addValue;     // 局部变量
    return resultValue;
}
```

### 6. 泛型参数

- 单个通用参数：`T`
- 有明确语义：`T` + `PascalCase`

```csharp
public class List<T> { }
public interface IMap<TKey, TValue> { }
public Task<TResult> ExecuteAsync<TResult>() { }
```

---

## 完整示例

```csharp
using System;

namespace MyNamespace
{
    public interface IProcessor
    {
        int Process(float value);
    }

    public enum ProcessState
    {
        Idle,
        Running,
        Finished,
    }

    public class DataProcessor : IProcessor
    {
        // ── public 字段：PascalCase ─────
        public int MaxRetries = 3;
        public static int InstanceCount;
        public const int Version = 1;

        // ── 非 public 字段：_camelCase ──
        private int _currentRetry;
        private static Random _rng = new Random();
        private const int _bufferSize = 4096;
        private static readonly int _shaderId
            = Shader.PropertyToID("_MainTex");
        private readonly List<int> _cache = new();

        // ── 属性：PascalCase ────────────
        public ProcessState State { get; private set; }

        // ── 方法：PascalCase ────────────
        public int Process(float value)
        {
            var scaledValue = value * MaxRetries;   // 局部：camelCase
            _currentRetry++;
            return (int)scaledValue;
        }

        private void ResetInternal(int newValue)    // 参数：camelCase
        {
            _currentRetry = newValue;
        }
    }
}
```

---

## Unity 项目的特殊情况

Unity 有两条与 Google 冲突的现实：

### 1. Inspector 序列化字段的显示

`[SerializeField] private int _hitCount;` 在 Inspector 里会显示为 "Hit Count"（Unity 自动去掉下划线、驼峰转空格）。**Google 规则完全兼容**，无需妥协。

但**社区惯例**是 public 序列化字段用 `camelCase`（不是 PascalCase）：

```csharp
public int score;                       // Unity 社区常见
public int Score;                       // Google 规则
```

**推荐做法**：Unity 项目里 public 字段用 `camelCase` 保持社区一致，或坚持 Google 用 `PascalCase`。**统一即可**。

### 2. Unity 引擎自身代码的老风格

Unity 引擎源码用 `m_XxxYyy`（Hungarian），这是 C++ 遗风。**你自己的代码不必跟随**，Google 风格更清晰。

---

## 一句话总结

**访问修饰符决定命名，存储修饰符不参与。**

- `public` 系 → `PascalCase`
- 非 `public` 系 → `_camelCase`
- `static` / `const` / `readonly` 只影响语义，不影响名字
- 方法、属性、类型 → 一律 `PascalCase`
- 局部变量、参数 → 一律 `camelCase`