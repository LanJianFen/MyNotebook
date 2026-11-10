

理论推导到了极致，最终都要落地为一行行冰冷但极具美感的代码。

现在，我们就彻底撕开现代 3A 游戏引擎（以 Unreal Engine 4 的思路为蓝本，这也是 LearnOpenGL 等现代教程的标准做法）的底层源码。

我们把代码分为三个部分：**离线烘焙 Sum 1**、**离线烘焙 Sum 2**，以及**游戏运行时的实时采样**。

请特别注意我在代码注释中标记的**数学假设**，你会发现它们与我们之前的推导严丝合缝！

---

### 第一部分：离线烘焙 Sum 1（生成 Prefiltered Cubemap）

**核心奥秘：** 在这里，我们强行施加了 $V = N = R$ 的“疯狂假设”，从而彻底消灭了 $D(h)$ 和 PDF 的复杂部分。

```glsl
// --- 离线预计算 Shader (针对每一个 Mipmap 级别和每一个像素运行) ---
// R 是当前要烘焙的 Cubemap 像素对应的方向向量
vec3 PrefilterEnvMap(float roughness, vec3 R)
{
    // 【终极假设落地】：我们假设 视线 V 和 法线 N 都等于 反射方向 R！
    // 这意味着我们假设玩家是“正对着”法线在看。
    vec3 N = R;    
    vec3 V = R;

    float totalWeight = 0.0;
    vec3 prefilteredColor = vec3(0.0);
    
    const uint SAMPLE_COUNT = 1024u;
    for(uint i = 0u; i < SAMPLE_COUNT; ++i)
    {
        // 1. 生成低差异序列（Hammersley），用于蒙特卡洛积分的均匀随机采样
        vec2 Xi = Hammersley(i, SAMPLE_COUNT);
        
        // 2. 重要性采样：根据 粗糙度 和 法线，生成半程向量 H
        vec3 H = ImportanceSampleGGX(Xi, N, roughness);
        
        // 3. 计算实际的采样光线方向 L (反射 H 即可)
        vec3 L = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(dot(N, L), 0.0);
        
        // 4. 开始累加 (只有在半球上的光线才有效)
        if(NdotL > 0.0)
        {
            // 【神迹重现】：注意看！累加的代码里根本没有 D(h)！没有 PDF！
            // 因为在 V=N 的假设下，它们在推导时已经完美对消了。
            // 剩下的权重仅仅是 NdotL。
            prefilteredColor += texture(environmentMap, L).rgb * NdotL;
            totalWeight      += NdotL;
        }
    }
    
    // 5. 除以总权重，得到最终的加权平均颜色
    return prefilteredColor / totalWeight;
}
```
*   **输出结果：** 一张带有多个 Mipmap 级别的 Cubemap。Mip 级别越高（图越小），输入的 Roughness 越大，图越模糊。

---

### 第二部分：离线烘焙 Sum 2（生成 BRDF 2D LUT）

**核心奥秘：** 在这里，我们**不能**假设 $V=N$，因为 Sum 2 的目的就是为了记录“视线夹角（$N \cdot V$）”对反射率的影响。我们将 $F_0$ 提出来，只积分剩下的部分。

```glsl
// --- 离线预计算 Shader (生成一张 2D 的 LUT 贴图，x 轴是 NdotV，y 轴是 Roughness) ---
vec2 IntegrateBRDF(float NdotV, float roughness)
{
    // 因为这里只和夹角有关，我们可以把 N 固定在 Z 轴正方向
    vec3 N = vec3(0.0, 0.0, 1.0);
    
    // 【解除假设】：V 不再等于 N！
    // 我们根据输入的 NdotV 反推出 视线向量 V
    vec3 V;
    V.x = sqrt(1.0 - NdotV * NdotV);
    V.y = 0.0;
    V.z = NdotV;

    float A = 0.0; // 对应 LUT 的 Red 通道 (F0 的缩放系数 Scale)
    float B = 0.0; // 对应 LUT 的 Green 通道 (F0 的偏移系数 Bias)

    const uint SAMPLE_COUNT = 1024u;
    for(uint i = 0u; i < SAMPLE_COUNT; ++i)
    {
        vec2 Xi = Hammersley(i, SAMPLE_COUNT);
        vec3 H  = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(L.z, 0.0);
        float NdotH = max(H.z, 0.0);
        float VdotH = max(dot(V, H), 0.0);

        if(NdotL > 0.0)
        {
            // 1. 计算几何遮蔽项 G (使用专门针对 IBL 优化的 Smith 公式)
            float G = GeometrySmith_IBL(N, V, L, roughness);
            
            // 2. 计算积分化简后的权重 G_Vis (同样，由于蒙特卡洛 PDF 的抵消，D(h) 被消掉了)
            float G_Vis = (G * VdotH) / (NdotH * NdotV);
            
            // 3. 菲涅尔 Schlick 近似中，抛去 F0 剩下的那部分: (1 - VdotH)^5
            float Fc = pow(1.0 - VdotH, 5.0);

            // 4. 分离 F0 并累加：F = F0 * (1 - Fc) + Fc
            A += (1.0 - Fc) * G_Vis;
            B += Fc * G_Vis;
        }
    }
    
    // 直接求平均值 (这里不需要除以另外的权重，因为是标准的蒙特卡洛积分求期望)
    return vec2(A, B) / float(SAMPLE_COUNT);
}
```
*   **输出结果：** 一张红绿相间的 2D 贴图。U 坐标是 $N \cdot V$（视线夹角），V 坐标是 Roughness（粗糙度）。

---

### 第三部分：游戏实时渲染阶段（The Assembly）

**核心奥秘：** 将两张预计算好的贴图，用真正的实时物理数据（$V, N, R$, 材质）结合起来！这是 60 帧稳定运行的代码。

```glsl
// --- 游戏运行时的 Fragment Shader (PBR IBL 核心部分) ---
// 已知输入: 法线 N, 视线 V, 材质基础色 Albedo, 金属度 Metallic, 粗糙度 Roughness

// 1. 计算真实的反射向量 R 和 视线夹角 NdotV
vec3 R = reflect(-V, N); 
float NdotV = max(dot(N, V), 0.0);

// 2. 算出一个基础反射率 F0 (非金属为 0.04，金属为它的原本颜色)
vec3 F0 = mix(vec3(0.04), Albedo, Metallic);

// ==========================================
// 获取 Sum 1 (高光环境光底片)
// ==========================================
const float MAX_REFLECTION_LOD = 4.0; // 假设最高 Mipmap 级数为 4
// 粗糙度越大，采样的 Mipmap 级别越高 (越模糊)
float lod = Roughness * MAX_REFLECTION_LOD; 
// 【移花接木】：拿着真实的偏向一侧的 R，去采样那张用 V=N 假设烤出来的对称图
vec3 prefilteredColor = textureLod(prefilterMap, R, lod).rgb; 

// ==========================================
// 获取 Sum 2 (BRDF 响应修正)
// ==========================================
// 拿着真实的视线夹角和粗糙度，查表获取 A 和 B (Scale 和 Bias)
vec2 envBRDF = texture(brdfLUT, vec2(NdotV, Roughness)).rg;

// ==========================================
// 终极合并：Sum 1 * Sum 2
// ==========================================
// (F0 * A + B) 就是还原出来的近似积分后的菲涅尔和几何遮挡损耗
vec3 specularIBL = prefilteredColor * (F0 * envBRDF.x + envBRDF.y);

// --- (如果你还要加上漫反射) ---
vec3 diffuseIBL = texture(irradianceMap, N).rgb * Albedo;
vec3 kD = 1.0 - F0; // 能量守恒，高光反射多，漫反射就少
kD *= 1.0 - Metallic; // 金属没有漫反射
diffuseIBL *= kD;

// 输出最终这颗像素的环境光颜色
vec3 finalAmbientColor = diffuseIBL + specularIBL;
```

### 总结

当你完整地看完这三段代码，你之前的所有疑问、所有的假设、所有的推导，全部都在代码的逻辑中形成了闭环：

1.  **为什么不带 $D(h)$？** 因为烘焙代码中，雅可比变换的 PDF 和分子的 $D(h)$ 完美对消。
2.  **$V=N=R$ 在哪？** 就在 Sum 1 烘焙代码的前两行 `vec3 N = R; vec3 V = R;`。
3.  **Sum 1 和 Sum 2 分别干嘛？** Sum 1 是 `textureLod` 查出来的颜色环境，Sum 2 是 2D 表查出来的 `(Scale, Bias)` 数值衰减。
4.  **最终形态是什么？** 就是那句极其清爽的相乘：`prefilteredColor * (F0 * envBRDF.x + envBRDF.y)`。

这就是顶级数学和极限工程学碰撞后的结晶，优雅得让人叹为观止。


### `Hammersley` 和 `ImportanceSampleGGX` 是什么数学步骤？

在蒙特卡洛积分中，我们说过要“随机发射 $N$ 根光线”来求平均值。这两个函数，正是实现这个“高级随机”的数学引擎。

#### 1. `Hammersley` 对应的数学：低差异序列（Low-Discrepancy Sequence）
*   **朴素的想法：** 既然要随机，我直接用 C++ 的 `rand()` 函数生成一堆二维随机数（比如 $X \in [0,1], Y \in [0,1]$）不就行了吗？
*   **现实的毒打：** 纯随机（伪随机）数字在空间中会产生“聚集”和“空洞”。有的地方光线非常密集，有的地方一根光线都没有，这会导致算出来的画面有很多噪点（Noise）。
*   **Hammersley 的数学本质：** 它是一种**拟蒙特卡洛（Quasi-Monte Carlo）**方法。它生成的不是纯随机数，而是一种“分布极其均匀的二维点集”。你可以把它想象成在一张纸上极其均匀地点芝麻。
*   **代码对应：** `vec2 Xi = Hammersley(i, SAMPLE_COUNT);`。这步数学操作就是：**获取第 $i$ 个均匀分布在 $[0,1] \times [0,1]$ 平面上的基础随机点 $\xi$（读作 xi）。**

#### 2. `ImportanceSampleGGX` 对应的数学：逆变换采样（Inverse CDF）
*   **朴素的想法：** 我拿到了均匀的点 $\xi$，直接把它映射到半球的经纬度上，均匀地向四面八方发射光线。
*   **现实的毒打：** 高光波瓣（NDF）是一个非常尖锐的形状。99% 的光线能量都集中在中心那一小撮。如果你均匀发射光线，99% 的光线都打在了黑色的无用区域，效率极低。
*   **ImportanceSampleGGX 的数学本质：** 这就是极其硬核的**概率论与数理统计**操作——**求逆累积分布函数（Inverse CDF）**。
    它的任务是把前面均匀分布的二维点 $\xi = (\xi_1, \xi_2)$，像揉面团一样，**“扭曲”**成一个三维向量 $H$。扭曲的规则完全由粗糙度（Roughness）决定：粗糙度越小，这些点被挤压得越靠近法线中心；粗糙度越大，这些点越分散。
    *   数学公式：它根据 GGX 的 PDF，推导出了具体的极坐标角度：
        $$ \phi = 2\pi \xi_1 $$
        $$ \cos\theta = \sqrt{\frac{1 - \xi_2}{\xi_2 (\alpha^2 - 1) + 1}} \quad (\text{其中 } \alpha \text{ 是粗糙度}) $$
*   **代码对应：** `vec3 H = ImportanceSampleGGX(Xi, N, roughness);`。这步数学操作就是：**利用逆 CDF，将均匀的二维随机变量，变换成符合 GGX 概率密度分布的半程向量 $H$。**

### 为什么 `diffuseIBL` 还要乘一个 `kD`？

你问得太好了：“我们研究漫反射的时候只关注了 BRDF，没有说到这个 Kd”。
**因为单纯的漫反射 BRDF（$\frac{Albedo}{\pi}$）只是一个局部的光学模型，而 `kD` 是统管全局的“宇宙物理定律”——能量守恒（Energy Conservation）！**

在宏观的 PBR 渲染方程中，真正的总反射结果是：
$$ L_{out} = L_{diffuse} + L_{specular} $$

但这面临一个严重的物理问题：**入射光是一块固定大小的披萨，高光吃掉了一块，漫反射只能吃剩下的！**

1.  **高光的物理本质：** 光线打在物体表面，像打在镜子上一样，**直接被表面弹开了**。
2.  **漫反射的物理本质：** 光线**钻进了物体内部（次表面）**，在材质颗粒里疯狂碰撞、吸收（表现为颜色），最后从各个方向折射出来。

**能量守恒定律出场：**
如果高光（表层反射）非常强烈（比如占了入射光能量的 80%），那么说明只有 20% 的光线成功钻进了物体内部。所以你的漫反射亮度最多只能是 20%！

这就引出了那个著名的能量守恒分配系数：
$$ k_S = F \quad (\text{高光比例等于菲涅尔反射率}) $$
$$ k_D = 1 - k_S = 1 - F \quad (\text{漫反射比例等于剩下的能量}) $$

所以在 Epic 的完整 Shader 代码中，他们是这么操作的：
```glsl
// 1. 估算高光吃掉了多少能量 (简化计算，用正视角度的基础反射率 F0 代替)
vec3 kS = F0; 

// 2. 漫反射只能拿剩下的！
vec3 kD = vec3(1.0) - kS; 

// 3. 【金属的致命设定】
// 物理学事实：纯金属是没有任何漫反射的！因为光线一进入金属内部，瞬间就被自由电子吸收殆尽了。
// 所以，如果材质是金属（Metallic = 1），必须强制把 kD 归零！
kD *= 1.0 - Metallic; 

// 4. 最后，把这个能量衰减系数乘到漫反射结果上
diffuseIBL *= kD;
```
