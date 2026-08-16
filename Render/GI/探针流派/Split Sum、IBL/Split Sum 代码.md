

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
