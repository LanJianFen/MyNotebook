

好的！我们这就来剖析第二种产生变体的方式：**`#pragma shader_feature`**。

正如前面所说，它的核心理念是**“按需打包，节省空间”**。这种方式最经典的运用场景就是**TA（技术美术）为项目编写通用的 Uber Shader（超级着色器）**。

假设我们要写一个通用的“机甲着色器”。有的普通机甲只需要基础贴图，而精英机甲需要身上带有**发光（Emission）效果**。为了不在运行时浪费性能去计算全黑的发光颜色，我们决定用宏来控制它。

下面是完整的演示，重点在于**属性面板的关联**和**打包时的智能剔除**。

### 第一步：Shader 代码（自带发光开关）

新建一个名为 `MechaShader.shader` 的文件。

```glsl
Shader "Custom/MechaShader"
{
    Properties
    {
        _MainTex ("基础贴图 (RGB)", 2D) = "white" {}
        
        // 【核心机制 1：面板开关】
        // [Toggle(宏名称)] 会在Unity材质面板上生成一个勾选框。
        // 勾选时，Unity会自动给这个材质打上 USE_EMISSION 的标签；不勾则去掉。
        [Toggle(USE_EMISSION)] _UseEmission ("开启发光效果?", Float) = 0
        
        // 发光颜色和强度（仅在开启发光时生效）
        [HDR] _EmissionColor ("发光颜色", Color) = (0,0,0,0)
    }
    
    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            // 【核心机制 2：按需编译指令】
            // 告诉Unity：这里有两种变体组合（无发光、有发光）。
            // 但打包时，请去检查项目里的材质球，只把真正用到的组合打进包里！
            #pragma shader_feature _ USE_EMISSION

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _EmissionColor; // 接收发光颜色

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // 1. 采样基础颜色
                fixed4 albedo = tex2D(_MainTex, i.uv);
                
                fixed3 finalColor = albedo.rgb;

                // 【变体逻辑分支】
                // 同样是预编译指令，GPU执行时没有 if 消耗
                #if defined(USE_EMISSION)
                    // 如果材质勾选了开关，编译时会保留这行代码，加上发光颜色
                    finalColor += _EmissionColor.rgb;
                #endif

                return fixed4(finalColor, albedo.a);
            }
            ENDCG
        }
    }
}
```

### 第二步：在 Unity 编辑器中的使用流程（无需写C#）

这段代码写完后，程序的任务就结束了，接下来是美术的工作流：

1.  美术在 Unity 里创建了一个材质球，命名为 `Mat_Soldier`（小兵机甲），把 `MechaShader` 赋给它。**不要勾选** "开启发光效果?"。
2.  美术又创建了一个材质球，命名为 `Mat_Boss`（Boss机甲），赋上同样的 Shader。**勾选上** "开启发光效果?"，并把发光颜色调成耀眼的红色。
3.  把这两个材质分别挂在对应的机甲模型上，场景里表现正常。

### 第三步：见证奇迹的打包时刻（底层行为）

现在，你点击了 Build 准备打包游戏。Unity 的 Shader 编译器开始工作，它会执行以下“查户口”般的操作：

*   **检查：** “嗯？这个 `MechaShader` 里面写了 `#pragma shader_feature _ USE_EMISSION`，说明它可能有 2 个变体。”
*   **扫描项目：** “让我看看项目里谁用了这个 Shader……找到了两个材质球，`Mat_Soldier` 和 `Mat_Boss`。”
*   **分析小兵材质：** “`Mat_Soldier` 面板上**没有勾选**发光开关。好的，我需要把 **`_` (默认无发光)变体**编译出来。”
*   **分析Boss材质：** “`Mat_Boss` 面板上**勾选了**发光开关。好的，我需要把 **`USE_EMISSION` (有发光)变体**编译出来。”

**【如果发生变数】：**
假设你的游戏被策划砍掉了一部分，Boss 不出场了，你把 `Mat_Boss` 材质从项目里**删除了**。再次打包时：
*   Unity 扫描后发现，全项目只有 `Mat_Soldier` 在用这个 Shader，且没有勾选发光。
*   Unity 就会毫不留情地把 `USE_EMISSION` 这个变体的二进制代码**彻底丢弃（Strip）**。
*   **结果：** 最终的游戏包体积变小了，内存占用也变小了。这就是 `shader_feature` 最伟大的地方！

---

### 高手避坑指南（致命错误演示）

因为 `shader_feature` 具有这种“剔除”特性，很多新手程序员会踩进一个极其隐蔽的坑：

**错误操作：**
你觉得 `Mat_Soldier`（小兵）在半血的时候发红光很酷。于是你既**没有**在面板上给小兵打勾，也**没有**Boss材质存在于项目中。然后你在 C# 里写了如下代码：

```csharp
// 当小兵半血时触发
void OnHalfHealth()
{
    // 试图在运行时通过代码强行开启发光宏
    soldierMaterial.EnableKeyword("USE_EMISSION"); 
}
```

**灾难后果：**
在 Unity 编辑器里测试时，一切完美，小兵发光了。（因为编辑器环境下，为了方便调试，所有变体都是实时临时编译的）。
**但是！当你打出手机包（APK/IPA）在真机上运行时，小兵半血时，非但不会发光，整个模型还会变成一坨“粉红色”，或者直接隐形！**

**为什么？**
因为打包时，Unity 发现没有任何材质在静态面板上勾选发光，直接把发光变体的代码**删了**！真机运行时，C# 要求 GPU 切换到 `USE_EMISSION` 变体，GPU 找了一圈没找到这段代码，直接崩溃，返回了经典的错误粉红色（Error Magenta）。

**如何解决？**
1.  **方案A：** 将 Shader 里的 `shader_feature` 改成 `multi_compile`。（强行把变体打入包中，但会增加包体）。
2.  **方案B（推荐做法）：** 创建一个“假材质（Dummy Material）”，应用这个 Shader 并勾选发光选项。把这个材质扔到 `Resources` 文件夹，或者丢进 `Shader Variant Collection`（SVC）里，相当于告诉 Unity：“给我留着这个变体，我运行时要用！”。
