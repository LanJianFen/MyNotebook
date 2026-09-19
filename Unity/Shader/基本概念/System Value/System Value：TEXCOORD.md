

这是一个极其经典、直指Shader管线核心的问题！

如果用一句话来概括它们的区别：
*   **`TEXCOORD0/1/2...` 是内部的“快递员”**，负责在 Shader 管线的**不同阶段之间**搬运数据。
*   **`SV_Target0/1/2...` 是最终的“集装箱”**，负责把 Shader 计算完的最终结果**打包输出**到显存（屏幕或纹理）里。

它们发生在渲染管线的**完全不同的阶段**。我们用一张流水线图和比喻来彻底搞懂它们：

---

### 🎨 渲染管线的故事：从模型到屏幕

想象 Shader 是一条汽车组装流水线。

#### 1. `TEXCOORD` （快递员 / 传送带）
它的全称是 Texture Coordinate（纹理坐标），但**在现代 Shader 里，它早就不仅仅是用来传 UV 贴图坐标的了**。它是一个**通用的数据寄存器（盛放数据的槽位）**。

它在流水线中出现**两次**：

*   **第一次出现（从模型拿到数据）：**
    顶点着色器（Vertex Shader）对前台大喊：“给我这个模型的原始数据！”
    系统就会通过 `TEXCOORD0` 把模型的第1套UV传进来，通过 `TEXCOORD1` 把第2套UV（比如光照贴图UV）传进来。
    ```hlsl
    struct appdata {
        float4 vertex : POSITION;
        float2 uv1 : TEXCOORD0; // 快递员0：拿着模型的第1套UV
        float2 uv2 : TEXCOORD1; // 快递员1：拿着模型的第2套UV
    };
    ```

*   **第二次出现（从顶点着色器传给片元着色器）：**
    顶点算完了坐标，要把数据**传给下一道工序**（片元着色器 Fragment Shader）。
    但是，能传的数据槽位是有限的。于是我们继续借用 `TEXCOORD` 这个名字装东西。**这时候，你想装什么都可以！**
    ```hlsl
    struct v2f {
        float4 pos : SV_POSITION; 
        float2 uv : TEXCOORD0;        // 槽位0：我装的是UV
        float3 worldPos : TEXCOORD1;  // 槽位1：我装的是世界坐标
        float3 viewDir : TEXCOORD2;   // 槽位2：我装的是视角方向
        float4 color : TEXCOORD3;     // 槽位3：我甚至可以装颜色！
    };
    ```
    GPU 硬件在光栅化阶段，会自动对 `TEXCOORD` 里的数据进行**插值（平滑过渡）**。

#### 2. `SV_Target` （集装箱 / 最终画布）
到了片元着色器（Fragment Shader），所有的计算都做完了，颜色算好了，光照也加上了。现在到了流水线的最后一步：**交货**。

`SV_Target` 就是收货的卡车。
它**只出现在片元着色器的 `return`（输出）阶段**。

*   把算好的主颜色装进 `SV_Target0`（送去屏幕显示）。
*   把算好的高光遮罩装进 `SV_Target1`（送去给后处理做 Bloom）。

```hlsl
// 片元着色器收到 TEXCOORD 快递员送来的数据包 (v2f i)
FragmentOutput frag(v2f i) 
{
    // 利用 TEXCOORD1 里的 worldPos 算光照
    // 利用 TEXCOORD0 里的 uv 贴图
    
    FragmentOutput o;
    // 算完了！交货！把结果塞进 SV_Target 里扔出 Shader！
    o.color0 = 算好的最终颜色 : SV_Target0; 
    o.color1 = 算好的法线数据 : SV_Target1; 
    
    return o;
}
```

---

### 📊 对比总结表

| 特性 | `TEXCOORD0/1/2...` (纹理坐标/寄存器) | `SV_Target0/1/2...` (系统值目标) |
| :--- | :--- | :--- |
| **发生位置** | **管线内部传递** (模型 $\to$ VS $\to$ FS) | **管线最终输出** (FS $\to$ 显存/屏幕) |
| **主要功能** | 充当输入/输出的数据槽位，传递变量 | 将最终计算的颜色/数值写入多张 RenderTexture 或屏幕 |
| **里面装什么** | UV、世界坐标、法线、视角方向、颜色...**任何你需要的数据**。 | 必须是 **颜色或将被当成颜色的像素数据** (用于 MRT)。 |
| **硬件行为** | GPU 会在顶点到片元之间对它进行**线性插值**。 | GPU 会直接把它**写入对应的目标纹理内存**中。 |
| **时代含义** | 旧时代的遗物，当年只能传UV，现在变成了“通用数据背包”。 | 现代大作 MRT（延迟渲染）的核心支撑。 |
| **数量限制** | 通常可以传 8~16 个 (TEXCOORD0 ~ TEXCOORD15) | 通常最多只能输出 4~8 个 (SV_Target0 ~ SV_Target7) |

### 💡 一句话记忆
*   **TEXCOORD**：我的 Shader 计算**需要**什么数据？（我带着数据在 Shader 里面跑）
*   **SV_Target**：我的 Shader 计算完**产出**了什么画面？（我把数据踢出 Shader 存进内存）
