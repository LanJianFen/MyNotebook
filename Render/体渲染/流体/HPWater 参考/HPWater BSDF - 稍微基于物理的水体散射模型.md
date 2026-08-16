

<div align="center">
<br>
<b>🎬 点击图片观看 HPWater BSDF演示视频</b>
</div>

<div align="center">
<a href="https://www.bilibili.com/video/BV1QTAYzqEGx/">
  <img width="400"  alt="image" src="https://github.com/user-attachments/assets/f8884efa-3174-42ce-ae3f-e2670056bb2b" alt="点击观看 Bilibili 演示视频"/>
</a>
</div>

<br>
<br>


> 实时水体渲染 BSDF 模型，支持体积散射、薄层 SSS 和背光透射。

直射光5°恒定厚度
<img width="1440"  alt="image" src="https://github.com/user-attachments/assets/0c81ce98-0cd3-4eff-ab6c-d62ab572d2f3" />

直射光15°恒定厚度
<img width="1440"  alt="image" src="https://github.com/user-attachments/assets/c0c962fc-9b46-4405-b75a-62c7927d5c4d" />

直射光30°恒定厚度
<img width="1440"  alt="image" src="https://github.com/user-attachments/assets/dd4f534f-108c-4dcb-9606-89f79baeca37" />

第一张(厚度近似)  后三张（不同直射光角度）
<img width="1440"  alt="image" src="https://github.com/user-attachments/assets/26639838-7f3c-4715-a79c-b18972824cb8" />

不同光向下的厚度近似模拟
<img width="1440"  alt="image" src="https://github.com/user-attachments/assets/4b802002-37f5-4bc5-b018-3a247948f276" />

---

## 与 GGX BRDF 的类比

本模型的结构设计参考了经典的 **GGX 微表面 BRDF**，两者都将光照分解为几个物理项的乘积：

| GGX BRDF (镜面反射) | 水体 BSDF (漫反射/透射) | 物理含义 |
|---------------------|-------------------------|----------|
| **D** (法线分布) | **S** (散射光) | 光的分布/散射量 |
| **G** (几何遮蔽) | **G** (入射几何项) | 入射光的有效投影 |
| **F** (菲涅尔) | **T** (菲涅尔透射) | 界面反射/透射比例 |

**GGX 镜面公式：**
$$f_{spec} = \frac{D \cdot G \cdot F}{4 \cdot (N \cdot L) \cdot (N \cdot V)}$$

**水体散射公式：**
$$diffR = G_{entry} \cdot T_{entry} \cdot S_{volume}$$
$$diffT = G_{sss} \cdot S_{sss} + G_{backlit} \cdot T_{backlit} \cdot P_{backlit}$$

两者的共同点：
- 都用**几何项 G** 表示入射光的有效投影面积
- 都用**菲涅尔项 F/T** 处理界面的能量分配
- 都将复杂光照分解为可独立计算的物理项相乘

主要区别：
- GGX 处理的是**表面微观几何**的反射
- 水体 BSDF 处理的是**体积介质**的散射和透射
- 水体需要区分**反射方向 (diffR)** 和**透射方向 (diffT)**

---

## 输入参数

相比 GGX BRDF（只需要 roughness 和 fresnel0），水体 BSDF 需要额外的体积散射参数：

**材质参数（BSDFData）：**

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `scatterColor` | float3 | [0, ∞) | 散射系数 μs，决定光被散射的强度和颜色 |
| `absorptionColor` | float3 | [0, ∞) | 吸收系数 μa，决定光被吸收的强度和颜色 |
| `thickness` | float | [0, 1] | 归一化厚度，0=极薄（波峰），1=最厚（深水边界） |
| `fresnel0` | float | ~0.02 | 水的菲涅尔基础反射率 |
| `_PhaseG` | float | [-1, 1] | HG 相位函数的 g 值，水体典型值 0.8（前向散射） |

**体积散射参数（WaterLightLoopData）：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `RayStart` | float3 | 射线起点（水面位置） |
| `RayEnd` | float3 | 射线终点（折射后的水下位置） |
| `SceneColor` | float3 | 水下场景颜色（用于场景散射，可选） |
| `shadowValue` | float | 阴影值（可选） |

**派生量：**
- 消光系数： $\mu_t = \mu_a + \mu_s$ （absorptionColor + scatterColor）
- 散射反照率： $\omega = \mu_s / \mu_t$ （决定吸收/散射比例）
- 光学深度： $\tau = \mu_t \cdot d$ （决定介质的"不透明度"）

---

## 整体概述

水体的光照与普通材质不同。光进入水中后会被吸收和散射，部分光在水体内部多次弹射后从表面出射回到摄像机。这个过程涉及：入射折射、体积散射、出射透射三个阶段。

本模型将水体散射分解为两个输出通道（遵循 HDRP 的 CBSDF 结构）：

**关于 diffR / diffT 的命名：**

在 HDRP 的 CBSDF 中，`R` = Reflection（反射方向），`T` = Transmission（透射方向）：
- **diffR**：光从表面的"同一侧"出射（入射和出射在法线同侧）
- **diffT**：光从表面的"另一侧"穿过来（入射和出射在法线两侧）

对于水体：
- **diffR (宏观体积散射)**：光从正面入射 → 水下散射 → 正面出射。光路在法线同侧，属于"反射方向"。适用于深水区域，通过 ray marching 累加散射。

- **diffT (薄层散射)**：光从背面/侧面穿过薄层出射。光路穿过表面，属于"透射方向"。包含：
  - **thinLayerSSS**：近似深水散射在薄层区域的延续，侧面和背面可见
  - **backlitTransmission**：光从背面直接穿透薄水层，看向光源时产生辉光

三个组件通过入射几何项自然分离，避免能量重复计算。

---

## 渲染流程总结

**Step 1: 入射计算**
计算三个入射几何项（G_entry、G_sss、G_backlit）和入射菲涅尔透射（T_entry），决定多少光进入水体、进入哪个散射路径。

**Step 2: 体积散射 (diffR)**
对深水区域进行少量 ray marching，累加多次散射的光量，输出 S_volume。

**Step 3: 薄层 SSS (diffT 的一部分)**
对薄层区域计算散射光量 S_sss，使用非线性光程修正处理不同厚度，并与深水散射混合。

**Step 4: 背光透射 (diffT 的一部分)**
计算光从背面穿透薄层的透射光，使用极强前向相位函数。

**Step 5: 出射透射**
所有散射光经过出射菲涅尔透射 T_exit 后到达摄像机。

**最终输出**

```
diffR = G_entry × T_entry × S_volume
diffT = (G_sss × S_sss) + (G_backlit × T_backlit × P_backlit)
output = (diffR + diffT) × T_exit
```

---

## Step 1: 入射计算

### 1.1 菲涅尔透射 (Fresnel Transmission)

光在界面上一部分被反射，剩余部分透射进入水中。透射比例由菲涅尔方程决定。

**数学公式：**

$$T_{entry} = 1 - F_{Schlick}(f_0, N \cdot L)$$

$$F_{Schlick}(f_0, \cos\theta) = f_0 + (1-f_0)(1-\cos\theta)^5$$

水的 $f_0 \approx 0.02$，即垂直入射时只有 2% 的光被反射。

**代码实现：**

```hlsl
// HPWaterBSDFLibary.hlsl: Line 200
float3 T_entry = 1.0 - F_Schlick(bsdfData.fresnel0, clampedNdotL);
```

---

### 1.2 入射几何项 (Incident Geometry Terms)

三个组件各自有独立的入射几何项，根据 NdotL 的正负自然分配能量，避免重复计算。

**数学公式：**

$$G_{entry} = \text{clamp}(N \cdot L, 0, 1)$$

$$G_{sss} = 1 - G_{entry}$$

$$G_{backlit} = \text{clamp}(-N \cdot L, 0, 1)$$

**代码实现：**

```hlsl
// HPWaterBSDFLibary.hlsl
float G_entry = clampedNdotL;      // Line 204
float G_sss = 1.0 - G_entry;       // Line 309
float G_backlit = saturate(-NdotL); // Line 375
```

**分工表：**

| 组件 | 几何项 | 正面 (NdotL=1) | 侧面 (NdotL=0) | 背面 (NdotL=-1) |
|------|--------|----------------|----------------|-----------------|
| diffR | G_entry | 1 | 0 | 0 |
| thinLayerSSS | G_sss | 0 | 1 | 1 |
| backlitTransmission | G_backlit | 0 | 0 | 1 |

**解释：**
- **正面**（NdotL > 0）：大部分光穿过薄层进入深水，由 diffR 处理
- **侧面**（NdotL ≈ 0）：diffR 无贡献，薄层散射 thinLayerSSS 主导
- **背面**（NdotL < 0）：光从背后入射，thinLayerSSS 和 backlitTransmission 共同作用

---

## Step 2: 体积散射 (Volume Scattering)

深水区域使用 ray marching 沿视线采样，累加每一步的散射光量。循环次数默认6次

**物理过程：**
1. 光进入水后沿视线传播
2. 每走一小步，部分光被吸收、部分被散射
3. 散射的光中，朝向摄像机的部分被累加
4. 重复直到到达水底或视线结束

---

### 2.1 Beer-Lambert 透射

光在介质中传播时按指数衰减。

**公式：**

$$T = \frac{I(d)}{I_0} = e^{-\mu_t \cdot d}$$

$$\mu_t = \mu_a + \mu_s \quad \text{(消光系数 = 吸收 + 散射)}$$

**代码实现：**

```hlsl
// HPWaterVolumetrics.hlsl: Line 202-205
float3 extinctionCoeff = absorptionCoeff + scatteringCoeff;
transmittance = exp(-extinctionCoeff * crossDistance);
```

---

### 2.2 散射光计算

被消光的光中，一部分被吸收（变成热量），另一部分被散射（改变方向）。


**公式：**

$$S = L \cdot (1 - T) \cdot \frac{\mu_s}{\mu_t} \cdot P(\theta)$$

- $L$: 入射光
- $(1 - T)$: 被消光的光的比例（积分结果）
- $\mu_s / \mu_t$: 散射反照率（被消光的光中有多少是散射而非吸收）
- $P(\theta)$: 相位函数（散射光中朝向摄像机的比例）

**代码实现：**

```hlsl
// HPWaterVolumetrics.hlsl: Line 199-218
float3 CaculateScatteredLight(
    float3 originLight,        // L: 入射光
    float3 absorptionCoeff,    // μa: 吸收系数
    float3 scatteringCoeff,    // μs: 散射系数
    float crossDistance,       // d: 光程
    float3 phase,              // P(θ): 相位函数
    out float3 transmittance)
{
    // μt = μa + μs
    float3 extinctionCoeff = absorptionCoeff + scatteringCoeff;

    // T = exp(-μt × d)
    transmittance = exp(-extinctionCoeff * crossDistance); 

    // (1 - T): 被消光的光量
    float3 extinguishedLight = originLight * (1.0 - transmittance);

    // μs / μt: 散射反照率
    float3 scatteringAlbedo = scatteringCoeff * rcp(extinctionCoeff);

    // S = L × (1-T) × (μs/μt) × P(θ)
    float3 scatteredLight = extinguishedLight * scatteringAlbedo * phase;
    return scatteredLight;
}
```

---

### 2.3 Henyey-Greenstein 相位函数

相位函数描述光散射后的方向分布。水体主要是前向散射。

**数学公式：**

$$P_{HG}(\cos\theta) = \frac{1 - g^2}{(1 + g^2 - 2g \cdot \cos\theta)^{1.5}}$$

- $g = 0.8$：强前向散射（水体典型值）
- $g = 0$：各向同性
- $g < 0$：后向散射

**代码实现：**

```hlsl
// HPWaterVolumetrics.hlsl: Line 171-177
float HenyeyPhase(float cosTheta, float phaseG)
{
    float g2 = phaseG * phaseG;
    float denom = 1.0 + g2 - 2.0 * phaseG * cosTheta;
    float mieScatter = (1.0 - g2) * rcp(pow(abs(denom), 1.5));
    return mieScatter;
}
```

---

### 2.4 混合相位函数 (瑞利 + 米氏)

真实水体同时存在瑞利散射（分子级，波长相关，产生蓝色）和米氏散射（颗粒级，前向散射）。

**数学公式：**

$$P_{Rayleigh}(\cos\theta) = \frac{3}{16\pi}(1 + \cos^2\theta)$$

$$P_{total} = 0.05 \cdot P_{Rayleigh} + 0.95 \cdot P_{HG}$$

**代码实现：**

```hlsl
// HPWaterVolumetrics.hlsl: Line 187-196
float3 CaculateScatterPhase(float cosTheta, float phaseG)
{
    // 瑞利散射（5%）：βR ∝ 1/λ⁴，短波长（蓝光）散射更强
    static const float3 betaRayleigh = float3(5.8e-6, 13.5e-6, 33.1e-6);
    float rayleighPhase = (1.0 + cosTheta * cosTheta) * (3 / (16 * PI));
    float3 rayleighScatter = betaRayleigh * rayleighPhase * 1e6;

    // 米氏散射（95%）
    float mieScatter = HenyeyPhase(cosTheta, phaseG);
    
    // 混合
    float3 scatterPhase = rayleighScatter * 0.05 + mieScatter * 0.95;
    return scatterPhase;
}
```

---

### 2.5 Ray Marching 主循环

**辐射传输方程的数值积分：**

理论上，沿视线累加的散射光是一个积分：

$$L_{scatter} = \int_0^D S(x) \cdot T(x \to \text{eye}) \, dx$$

其中 $S(x)$ 是位置 x 处的散射光源项，$T(x \to \text{eye})$ 是从 x 到眼睛的透射率。

这个积分没有解析解，所以用 Ray Marching 做数值近似（黎曼求和）：

$$L_{scatter} \approx \sum_{i=0}^{N-1} S(x_i) \cdot T_i \cdot \Delta x_i$$

**指数步进的数学推导：**

近处对视觉贡献大（细节可见），远处贡献小（被衰减）。用指数步进可以在采样数相同时获得更好的精度。

设 $t \in [0, 1]$ 是归一化采样索引（ $t = i/N$ ）， $k = \ln(\text{EXP\\_FACTOR})$ ：

$$d(t) = \frac{e^{k \cdot t} - 1}{e^k - 1}$$

当 $t=0$ 时 $d=0$ ，当 $t=1$ 时 $d=1$ 。步长为导数：

$$\Delta d = \frac{dd}{dt} \cdot \Delta t = \frac{k \cdot e^{k \cdot t}}{e^k - 1} \cdot \frac{1}{N}$$

**代码实现优化：**

令 `currentExp = e^(k·t) = EXP_FACTOR^t`，则：
- `d = (currentExp - 1) * kDenom`
- `dd = currentExp * kDD`
- 每次迭代 `currentExp *= expStep`（其中 `expStep = EXP_FACTOR^(1/N)`）

```hlsl
// HPWaterVolumetrics.hlsl: Line 294-345

// 指数步进参数预计算
float rcpCount = rcp(float(WATER_SAMPLE_COUNT));      // 1/N
float kDenom = rcp(EXP_FACTOR - 1.0);                 // 1/(e^k - 1)
float kDD = log(EXP_FACTOR) * rcpCount * kDenom;      // k/(N·(e^k-1))
float expStep = pow(EXP_FACTOR, rcpCount);            // e^(k/N)
float currentExp = pow(EXP_FACTOR, Dither * rcpCount); // 抖动起始点

//WATER_SAMPLE_COUNT = 6
for (int i = 0; i < WATER_SAMPLE_COUNT; i++)
{
    // d: 归一化采样位置 [0,1]，dd: 当前步的归一化步长
    float d = (currentExp - 1.0) * kDenom;
    float dd = currentExp * kDD;   
    float3 samplePos = RayStart + NoLinearRayDirection * d;  
    float3 samplePosDynamic = RayStart + DynamicRayDirection * d;
    
    // 计算阴影
    shadowValue = ComputeShadowValue(samplePosDynamic, featureFlags, lightLoopContext, posInput, bsdfData);
    if(i == 0) lastShadowValue = shadowValue;
    
    // ===== 一次散射：直接光照 =====
    float cosTheta = dot(safeNormalize(samplePos), safeNormalize(LightDir));
    float3 scatterPhase = CaculateScatterPhase(cosTheta, _PhaseG);
    float3 directLighting = LightColor * shadowValue;
    float directCrossDistance = dd * NoLinearRayLength + SunDepth * dd;        

    // 先计算不带相位的基础散射（phase = 1）
    float3 baseScatter = CaculateScatteredLight(
        directLighting, AbsorptionCoefficient, ScatterCoefficient,
        directCrossDistance, 1, transmittance, scatteringAlbedo).rgb; 
    
    // 多次散射效果：根据散射反照率决定相位的各向同性程度
    // scatterAlbedo 高 -> 多次散射充分 -> 相位趋于各向同性
    // scatterAlbedo 低 -> 单次散射主导 -> 保留方向性相位
    float scatterAlbedo = Luminance(scatteringAlbedo) * GetInverseCurrentExposureMultiplier();
    float3 effectivePhase = lerp(scatterPhase, 1.0, saturate(smoothstep(0, 0.5, scatterAlbedo)));
    
    float3 directScatteredLight = baseScatter * effectivePhase;
    
    // 累加
    accumTransmittance *= transmittance;
    scatteredLight += directScatteredLight;
    currentExp *= expStep;
}

// ===== 水下场景散射（Scene In-Scattering）=====
// 水下物体反射的光，穿过水体时也会被散射，产生"雾化"效果
float3 sceneLight = SceneColor * lerp(shadowValue, 1, 0.3);
float lightIntensity = Luminance(LightColor);
float3 sceneScatteredLight = sceneLight * accumTransmittance * ScatterCoefficient * NoLinearRayLength * lightIntensity;
scatteredLight += sceneScatteredLight;
```

**解释：**
- `EXP_FACTOR` 控制指数步进的疏密程度，近处采样密集以捕捉细节
- `Dither` 用于抖动起始点，减少带状 artifact
- `effectivePhase` 在厚介质中趋于各向同性，模拟多次散射后方向性丢失

**水下场景散射公式：**

$$S_{scene} = L_{scene} \cdot T_{accum} \cdot \mu_s \cdot d \cdot I_{sun}$$

这是一个简化近似（避免对场景光再做完整 ray marching）：
- $L_{scene}$：水下场景光（SceneColor，考虑 30% 阴影穿透）
- $T_{accum}$：累积透射率（场景光穿过水体的衰减）
- $\mu_s$：散射系数（决定多少光被散射出来）
- $d$：光程长度（路径越长散射越多）
- $I_{sun}$：光源亮度（调制系数，使场景散射与直接光散射亮度一致）

物理效果：让远处的水下物体看起来更模糊、更偏向水的散射色（雾化）。

---

## Step 3: 薄层 SSS (Thin Layer Subsurface Scattering)

薄层（如波峰、浪花）厚度无法直接得知，只能用高度或法线近似。关键是正确估算光程，并处理与深水散射的过渡。

**输入参数 thickness：**

`thickness` 是归一化的厚度值，范围 **[0, 1]**：
- `0` = 极薄（波峰顶部、浪花边缘）
- `1` = 最厚（薄层与深水区域的边界）

通常由高度场、法线或其他几何信息近似得到。代码中乘以 `HPWATER_SSS_PATH_SCALE`（默认 20 米）转换为等效光程。

**薄层与深水的几何关系：**

```
        波峰（薄层顶部）thickness ≈ 0
          /\
         /  \
        /    \      ← 薄层区域
       /      \
      /────────\    ← thickness = 1（薄层底部 = 深水顶部）
     │          │
     │  深水区  │   ← S_volume（ray marching 累积）
     │__________│
```

薄层的底部与深水区域连接。当 thickness 接近 1 时，薄层实际上"看到"的是深水区域累积的散射光。因此：
- **薄层 SSS 不是独立计算**，而是近似深水散射在薄层区域的延续
- **光程缩放（20 米）较大**，是因为要匹配深水 ray marching 的累积效果
- **Fallback 机制**：当 thickness 很小（透射率高）时，直接使用 S_volume 作为薄层的颜色

---

**为什么相机与光源角度相近时，水面背面会稍亮？**

典型场景：夕阳下，人以较低视角看水面。此时相机方向 V 和光源方向 L 角度相近（都是低角度），水面波峰的背面会呈现微微透亮的效果。

这是 **相位函数**、**入射几何项** 和 **出射菲涅尔透射** 三者共同作用的结果：

1. **相位函数 P(cosθ) 的前向散射**
   - cosθ = dot(-V, L)，V 和 L 角度越近，cosθ 越大
   - HG 相位函数随 cosθ 增大而增大（前向散射特性）
   - 散射光更多地朝向摄像机方向

2. **入射几何项 G_sss 在背面有值**
   - 背面入射时 NdotL < 0，导致 G_entry = 0
   - 因此 G_sss = 1 - G_entry = 1，薄层 SSS 获得入射能量
   - 正面入射时 G_sss 减小，薄层贡献减少

3. **出射菲涅尔透射 fresnelTransmissionExit**
   - T_exit = 1 - F_Schlick(f0, NdotV)
   - 法线朝向相机（NdotV 大）→ T_exit 大 → 压制小，光容易出射
   - 法线不朝向相机（NdotV 小）→ T_exit 小 → 压制大，光难以出射
   - 波峰背面的法线相对更朝向相机，所以 T_exit 较大，散射光能出射

**三者的平衡：**

```
夕阳低角度 + 人高视角看水面：

入射：G_sss ≈ 0.6~1（背面/侧面入射）
  ↓
散射：P_sss 中等偏高（V、L 角度相近但非正对）
  ↓
出射：T_exit ≈ 0.7~0.9（低视角有所衰减）

三者相乘 → 背面稍亮（自然的透光效果）
```

这就是为什么波峰薄处在这种视角下会呈现柔和的透光效果。

---

### 3.1 非线性光程修正 (Nonlinear Path Length)

薄层 SSS 用单次计算近似深水散射的延续。光程需要随 thickness 增加而增长，以匹配深水 ray marching 的累积效果。

**物理直觉：**
- 薄层（d → 0）：透射率高，fallback 到 S_volume，光程影响小
- 厚层（d → 1）：需要更长光程来匹配深水累积散射的视觉效果

**数学公式：**

$$L_{linear} = d \cdot scale$$

$$L_{nonlinear} = d^2 \cdot scale \cdot (1 + \mu_s)$$

$$\tau = \mu_t \cdot d \quad \text{(光学深度)}$$

$$L_{effective} = \text{lerp}(L_{linear}, L_{nonlinear}, \tau \cdot strength)$$

**代码实现：**

```hlsl
// HPWaterBSDFLibary.hlsl: Line 264-280

// 线性部分：薄层区域
float L_linear = thickness * HPWATER_SSS_PATH_SCALE;

// 非线性部分：厚层区域，d² 让光程在 thickness→1 时增长更快
float scatterStrength = Luminance(bsdfData.scatterColor);
float L_nonlinear = thickness * thickness * HPWATER_SSS_PATH_SCALE 
                  * (1.0 + scatterStrength);

// 光学深度决定混合比例，τ 大时使用非线性
float opticalDepth = sss_extinctionScalar * thickness * HPWATER_SSS_PATH_SCALE;
float nonlinearWeight = saturate(opticalDepth * HPWATER_SSS_NONLINEAR_STRENGTH);

// 有效光程
float sssPathLength = lerp(L_linear, L_nonlinear, nonlinearWeight);
```

---

### 3.2 薄层散射计算与深水混合

薄层 SSS 是深水散射在薄层区域的延续。根据透射率与深水散射混合：透射率高说明水很薄，应该直接看到深水的散射颜色。

**数学公式：**

$$S_{sss} = \text{CaculateScatteredLight}(L, \mu_a, \mu_s, L_{effective}, P_{sss})$$

$$sssWeight = 1 - \text{Luminance}(T_{sss})$$

$$thinLayerSSS = \text{lerp}(S_{volume} \cdot G_{sss}, S_{sss} \cdot G_{sss}, sssWeight)$$

**代码实现：**

```hlsl
// HPWaterBSDFLibary.hlsl: Line 282-296
// P_sss：相位函数
float sss_cosTheta = dot(-V, L);
float3 P_sss = CaculateScatterPhase(sss_cosTheta, _PhaseG);

// S_sss：散射光计算
float3 sss_transmittance;
float3 S_sss = WaterVolumeLightLoop::CaculateScatteredLight(
    LightColor, bsdfData.absorptionColor, bsdfData.scatterColor,
    sssPathLength, P_sss, sss_transmittance);

// HPWaterBSDFLibary.hlsl: Line 309-329
// G_sss：薄层入射项 = 1 - 正面入射项
float G_sss = 1.0 - G_entry;

// 薄层 SSS 输出 = 入射项 × 散射 × 阴影 × 补偿
float3 thinLayerSSS = S_sss * G_sss * lastShadowValue * HPWATER_SSS_SCATTER_BOOST;

// --------------------------------
// 薄层与深水的混合（Fallback 机制）
// --------------------------------
//
//   薄层区域（thickness 小）：
//     └─ transmittance 高 → sssWeight 低
//     └─ 直接使用深水散射颜色（S_volume）
//
//   厚层区域（thickness 大）：
//     └─ transmittance 低 → sssWeight 高
//     └─ 使用薄层 SSS（近似深水累积散射的延续）
//
//   注意：两个分支都乘以 G_sss，保持入射分工一致
//
float sssWeight = saturate(1.0 - Luminance(sss_transmittance));
thinLayerSSS = lerp(S_volume * G_sss, thinLayerSSS, sssWeight);
```

**解释：**
- `G_sss = 1 - G_entry` 确保正面入射时薄层 SSS 为 0，能量由 diffR 处理
- `HPWATER_SSS_SCATTER_BOOST` 补偿单次计算与深水 ray marching 的亮度差异
- Fallback 机制让波峰顶部（极薄）直接看到深水散射颜色
- 两个分支都乘以 `G_sss`，保证入射分工一致

---

## Step 4: 背光透射 (Backlit Transmission)

当对准光源观察时，光从背后穿透薄水层产生辉光。这是对准太阳时水面发亮的原因。

**与薄层 SSS 的区别：**
- 薄层 SSS：光散射后出射，方向较分散（g ≈ 0.8）
- 背光透射：光几乎直穿，极强方向性（g ≈ 0.998）

**数学公式：**

$$Backlit = LightColor \cdot G_{backlit} \cdot T_{backlit} \cdot P_{backlit}$$

其中：
- $G_{backlit} = \text{clamp}(-N \cdot L, 0, 1)$：背面入射投影
- $T_{backlit} = e^{-\mu_t \cdot d}$：Beer-Lambert 透射
- $P_{backlit} = P_{HG}(\cos\theta, 0.998)$：极强前向相位

**代码实现：**

```hlsl
// HPWaterBSDFLibary.hlsl: Line 370-392

// G_backlit：背面入射投影
float G_backlit = saturate(-NdotL);

// 背光专用光程（比 SSS 短，纯穿透路径）
float backlitPathLength = thickness * HPWATER_BACKLIT_PATH_SCALE;

// T_backlit：Beer-Lambert 透射
float3 T_backlit = exp(-sss_extinctionCoeff * backlitPathLength);

// P_backlit：极强前向相位（g = 0.998）
float backlit_cosTheta = dot(V, -L);
float P_backlit = HenyeyPhase(backlit_cosTheta, 0.998);

// 输出
float3 backlitTransmission = LightColor * G_backlit * T_backlit * P_backlit * lastShadowValue;
```

**解释：**
- `G_backlit` 只在背面（NdotL < 0）有值
- `backlit_cosTheta = dot(V, -L)` 是摄像机看向光源穿透方向的夹角
- g = 0.998 意味着只有几乎正对光源时才能看到背光透射
- `HPWATER_BACKLIT_PATH_SCALE` 比 SSS 的 scale 小，因为背光是纯穿透

---

## Step 5: 出射透射 (Exit Transmission)

散射光从水内出射时，再次经过菲涅尔透射。

**数学公式：**

$$T_{exit} = 1 - F_{Schlick}(f_0, N \cdot V)$$

**代码实现：**

```hlsl
// HPWaterBSDFLibary.hlsl: Line 98 (PostEvaluateBSDF)
float3 fresnelTransmissionExit = 1.0 - F_Schlick(bsdfData.fresnel0, clampedNdotV);
lightLoopOutput.diffuseLighting *= fresnelTransmissionExit;
```

---

## 最终输出组合

**数学公式：**

$$diffR = G_{entry} \cdot T_{entry} \cdot S_{volume}$$

$$diffT = (G_{sss} \cdot S_{sss}) + (G_{backlit} \cdot T_{backlit} \cdot P_{backlit})$$

$$output = (diffR + diffT) \cdot T_{exit}$$

**代码实现：**

```hlsl
// HPWaterBSDFLibary.hlsl: Line 207, 220
float3 underwaterLight = G_entry * T_entry;
float3 macroScattering = S_volume * underwaterLight;

// HPWaterBSDFLibary.hlsl: Line 312
float3 thinLayerSSS = S_sss * G_sss * lastShadowValue * HPWATER_SSS_SCATTER_BOOST;

// HPWaterBSDFLibary.hlsl: Line 392
float3 backlitTransmission = LightColor * G_backlit * T_backlit * P_backlit * lastShadowValue;

// HPWaterBSDFLibary.hlsl: Line 440-441
cbsdf.diffR = macroScattering;
cbsdf.diffT = thinLayerSSS + backlitTransmission;
```

---

## 参数说明

### 薄层 SSS 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `HPWATER_SSS_PATH_SCALE` | 20.0 | 光程缩放（米），较大是因为要匹配深水累积散射的效果 |
| `HPWATER_SSS_NONLINEAR_STRENGTH` | 0.2 | 非线性修正强度 |
| `HPWATER_SSS_SCATTER_BOOST` | 2.0 | 能量补偿系数 |

### 背光透射参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `HPWATER_BACKLIT_PATH_SCALE` | 20.0 | 背光光程缩放（米） |

### 体积散射参数

| 参数 | 说明 |
|------|------|
| `WATER_SAMPLE_COUNT` | Ray marching 采样次数 |
| `EXP_FACTOR` | 指数步进因子，值越大近处越密集 |
| `_PhaseG` | HG 相位函数的 g 值（0.8 = 前向散射） |

---

## 视觉效果分区

| 区域 | 主导组件 | 效果描述 |
|------|----------|----------|
| 正面（朝向光源） | diffR | 深水体积散射，颜色饱和 |
| 侧面 | thinLayerSSS | 薄层边缘散射，半透明 |
| 背面（背对光源） | thinLayerSSS + backlitTransmission | 背光透射 + 薄层散射 |
| 看向太阳 | backlitTransmission | 强烈的前向散射辉光 |

---

## 物理参考

- **Beer-Lambert 定律**：光在介质中的指数衰减
- **Henyey-Greenstein 相位函数**：描述散射的角度分布
- **菲涅尔方程**：界面反射/透射比例
- **辐射传输方程 (RTE)**：体积散射的理论基础

---
