

太棒了！为了让你直观地感受这套“极限拉扯”的精妙之处，我们来模拟一个具体的游戏场景：**角色状态异常系统**。

假设你的角色有 3 种可以通过 C# 动态附加的状态特效：
1. **冰冻 (`STATUS_FROZEN`)**：角色变蓝。
2. **燃烧 (`STATUS_FIRE`)**：角色变红。
3. **隐身 (`STATUS_STEALTH`)**：角色变半透明。

**逻辑分析（核心重点）：**
理论上有 $2 \times 2 \times 2 = 8$ 种变体组合。
但是策划定了一个硬规则：**“冰冻”和“燃烧”是互斥的！** 一个角色绝不可能同时处于冰冻和燃烧状态。
所以，包含 `STATUS_FROZEN` + `STATUS_FIRE` 的 2 个变体（无论隐不隐身）是**绝对废弃**的。我们真正需要的只有 **6 个组合**。

下面是标准工作流的代码和配置演示：

---

### 第一步：编写 Shader（全部用 shader_feature）

新建一个 `HeroStatus.shader` 文件。

```glsl
Shader "Custom/HeroStatus"
{
    Properties
    {
        _MainTex ("Base Texture", 2D) = "white" {}
    }
    SubShader
    {
        // 因为有隐身，所以渲染队列设为透明
        Tags { "RenderType"="Transparent" "Queue"="Transparent" }
        Blend SrcAlpha OneMinusSrcAlpha

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            // 【核心法则：全部使用 shader_feature】
            // 坚决不用 multi_compile，把生杀大权交给后面的 SVC 白名单
            #pragma shader_feature _ STATUS_FROZEN
            #pragma shader_feature _ STATUS_FIRE
            #pragma shader_feature _ STATUS_STEALTH

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
                fixed4 col = tex2D(_MainTex, i.uv);

                // 状态 1：冰冻（偏蓝）
                #if defined(STATUS_FROZEN)
                    col.rgb = col.rgb * fixed3(0.5, 0.8, 1.0);
                #endif

                // 状态 2：燃烧（偏红）
                #if defined(STATUS_FIRE)
                    col.rgb = col.rgb * fixed3(1.0, 0.4, 0.2);
                #endif

                // 状态 3：隐身（半透明）
                #if defined(STATUS_STEALTH)
                    col.a *= 0.3; 
                #endif

                return col;
            }
            ENDCG
        }
    }
}
```

---

### 第二步：配置 SVC 白名单（Unity 编辑器操作）

既然代码里写的是 `shader_feature`，如果场景里没有挂载特定材质，打包时这些变体就会被全删了。为了让 C# 能动态切，我们必须**给这 6 个合理组合发“免死金牌”**。

1. 在项目窗口右键 -> `Create` -> `Shader` -> `Shader Variant Collection`，命名为 `HeroSVC`。
2. 双击打开 `HeroSVC`。
3. 点击 **Add Shader**，把刚才写的 `HeroStatus.shader` 加进去。
4. 在下面的列表中，**精确勾选你需要的这 6 种组合**：

    *   [ ] (什么都不勾) -> **基础状态**
    *   [√] `STATUS_FROZEN` -> **纯冰冻**
    *   [√] `STATUS_FIRE` -> **纯燃烧**
    *   [√] `STATUS_STEALTH` -> **纯隐身**
    *   [√] `STATUS_FROZEN`, `STATUS_STEALTH` -> **冰冻+隐身**
    *   [√] `STATUS_FIRE`, `STATUS_STEALTH` -> **燃烧+隐身**

*(注意看：我们刻意**没有**添加 `STATUS_FROZEN` + `STATUS_FIRE` 的组合。这样打包时，Unity 就会把那两个非法组合彻底剔除！)*

5. 最后，打开 `Edit` -> `Project Settings` -> `Graphics`。把 `HeroSVC` 文件拖进底部的 **Preloaded Shaders** 数组中。

---

### 第三步：C# 运行时动态切换（安全调用）

现在，就算没有任何材质球在面板上打勾，因为有 SVC 保底，我们也可以在 C# 里肆无忌惮地动态切换这 6 种状态了。

新建 `HeroController.cs` 挂载在角色上：

```csharp
using UnityEngine;

public class HeroController : MonoBehaviour
{
    private Material heroMaterial;

    void Start()
    {
        // 获取当前角色的材质实例 (注意：这会生成一个Material实例)
        // 这样修改只会影响当前角色，不会影响全场的其他角色
        heroMaterial = GetComponent<Renderer>().material;
    }

    // 受到冰霜攻击
    public void OnHitByIce()
    {
        // 逻辑互斥：如果之前在燃烧，先关掉燃烧
        heroMaterial.DisableKeyword("STATUS_FIRE");
        
        // 开启冰冻
        heroMaterial.EnableKeyword("STATUS_FROZEN");
    }

    // 受到火焰攻击
    public void OnHitByFire()
    {
        // 逻辑互斥：如果之前在冰冻，先关掉冰冻
        heroMaterial.DisableKeyword("STATUS_FROZEN");
        
        // 开启燃烧
        heroMaterial.EnableKeyword("STATUS_FIRE");
    }

    // 开启/关闭隐身 (隐身可以和冰冻/燃烧共存)
    public void ToggleStealth(bool isStealth)
    {
        if (isStealth)
            heroMaterial.EnableKeyword("STATUS_STEALTH");
        else
            heroMaterial.DisableKeyword("STATUS_STEALTH");
    }
    
    // 清除所有异常状态
    public void ClearStatus()
    {
        heroMaterial.DisableKeyword("STATUS_FROZEN");
        heroMaterial.DisableKeyword("STATUS_FIRE");
        heroMaterial.DisableKeyword("STATUS_STEALTH");
        // 此时 Shader 运行在 "_" (基础状态) 变体下
    }
}
```

### 总结算账：

在这个例子中：
*   **如果不做这套操作（全用 `multi_compile`）**：打包 8 个变体。虽然 C# 逻辑保证了冰+火不会同时出现，但多余的 2 个变体依然傻傻地躺在安装包里占用空间，消耗编译时间。
*   **按照你顿悟的这套高级操作**：精准打包 6 个变体！如果配合底层的阴影、雾效宏，节省的变体数量可能是成百上千个。既保证了 C# 代码运行绝不紫屏，又实现了对底层资源的极致掌控。
