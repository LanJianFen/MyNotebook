

没问题！为了让你最直观地感受到 `#pragma multi_compile` 的威力，我们来模拟一个游戏开发中极其常见的需求：**全局天气系统（Global Weather System）**。

假设你的游戏里有几百个建筑物和树木的材质，当玩家在游戏中呼出控制台，输入“下雨”或“下雪”指令时，整个世界的场景材质都要随之改变（下雨变暗变蓝，下雪变亮变白）。

既然是**运行时动态全局切换**，这就必须使用 `multi_compile`。下面是完整的代码演示，分为 **Shader端** 和 **C#端**。

### 第一步：Shader 代码（材质表现）

新建一个名为 `WeatherShader.shader` 的文件。为了代码精简易懂，我们以无光照（Unlit）为例，重点看宏的处理逻辑。

```glsl
Shader "Custom/WeatherShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            // 【核心代码在这里】
            // 声明全局宏：_ (默认无天气)、WEATHER_RAIN (下雨)、WEATHER_SNOW (下雪)
            // 打包时，Unity会强制为这个Pass编译出 3 份独立的二进制代码包！
            #pragma multi_compile _ WEATHER_RAIN WEATHER_SNOW

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

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // 基础采样
                fixed4 col = tex2D(_MainTex, i.uv);

                // 【变体逻辑分支】
                // 注意：这里的 #if / #elif 是预编译指令！
                // GPU执行时根本没有 if/else，而是直接执行打包好的纯线性代码
                #if defined(WEATHER_RAIN)
                    // 下雨变体代码：整体压暗，并带一点点冷色调（蓝色）
                    col.rgb *= fixed3(0.6, 0.7, 0.9);
                    // （甚至可以在这里加上雨水流下的UV扰动计算代码...）

                #elif defined(WEATHER_SNOW)
                    // 下雪变体代码：整体提亮，模拟积雪覆盖
                    col.rgb += fixed3(0.3, 0.3, 0.3);

                #else
                    // 默认变体代码（即前面的 "_"）：什么都不加，保持原样
                    
                #endif

                return col;
            }
            ENDCG
        }
    }
}
```

### 第二步：C# 控制代码（运行时动态切换）

在游戏中挂载一个脚本 `WeatherManager.cs`。因为我们用的是 `multi_compile`，所以可以直接使用静态方法 `Shader.EnableKeyword` 进行**全局广播**，不需要去获取每一个材质球。

```csharp
using UnityEngine;

public class WeatherManager : MonoBehaviour
{
    // 定义天气枚举
    public enum WeatherType
    {
        Normal,
        Rain,
        Snow
    }

    void Start()
    {
        // 游戏开始时，默认设置为晴天
        SetWeather(WeatherType.Normal);
    }

    void Update()
    {
        // 模拟玩家按键动态切换天气
        if (Input.GetKeyDown(KeyCode.Alpha1)) SetWeather(WeatherType.Normal);
        if (Input.GetKeyDown(KeyCode.Alpha2)) SetWeather(WeatherType.Rain);
        if (Input.GetKeyDown(KeyCode.Alpha3)) SetWeather(WeatherType.Snow);
    }

    // 全局切换天气的方法
    public void SetWeather(WeatherType weather)
    {
        // 第一步：先严谨地关闭所有天气相关的宏
        Shader.DisableKeyword("WEATHER_RAIN");
        Shader.DisableKeyword("WEATHER_SNOW");

        // 第二步：根据传入的状态，开启对应的宏
        switch (weather)
        {
            case WeatherType.Normal:
                Debug.Log("切换到：晴天 (使用默认变体 '_')");
                // 都不开启，Shader会自动走 "_" 的默认分支
                break;

            case WeatherType.Rain:
                Debug.Log("切换到：下雨 (使用 WEATHER_RAIN 变体)");
                Shader.EnableKeyword("WEATHER_RAIN");
                break;

            case WeatherType.Snow:
                Debug.Log("切换到：下雪 (使用 WEATHER_SNOW 变体)");
                Shader.EnableKeyword("WEATHER_SNOW");
                break;
        }
    }
}
```

### 总结：底层到底发生了什么？

当你点击 Build 打包这款游戏时：
1. Unity读取到 `#pragma multi_compile _ WEATHER_RAIN WEATHER_SNOW`。
2. 无论你当前的测试场景是什么天气，Unity都会乖乖地把这段Shader**强行拆分成三份指令**进行编译。
   * **变体A (晴天)**：`col.rgb *= fixed3(0.6, 0.7, 0.9);` 和 `col.rgb += fixed3(0.3...);` 这两行代码被**彻底删除**。
   * **变体B (下雨)**：保留压暗代码，剔除积雪代码。
   * **变体C (下雪)**：保留积雪代码，剔除压暗代码。
3. 把这三个变体全部打包进最终的 `.apk` 或 `.exe` 之中。
4. 游戏运行时，当你按下键盘上的 `2` 键，C# 执行 `Shader.EnableKeyword("WEATHER_RAIN")`，**GPU的渲染管线会瞬间将当前材质的执行程序，从变体A替换成变体B**。没有任何if/else判断阻塞，性能极高！
