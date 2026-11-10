

现代次时代着色流水线（PBR Pipeline）将片元求值严格划分为 **5 个单向递进的阶段**。这种划分的核心原则是 **“视线（$V$）与光源（$L$）的正交依赖法则”**，确保计算不冗余、结构不耦合。

---
### 一、5 个阶段装什么数据与如何区分

#### 阶段 1：几何与空间上下文 (`FSkinPixelParameters`)
* **包含的数据**：
  * **空间与视线**：世界坐标 `position_ws`、世界视线向量 `view_dir_ws`
  * **几何骨架 (TBN)**：顶点世界法线 `geom_normal_ws`、切线 `tangent_ws`、副切线 `bitangent_ws`
  * **纹理与屏幕坐标**：多套 UV（`uv.xy`, `uv.zw`）、屏幕归一化坐标 `screen_uv`
  * **屏幕级遮挡**：屏幕阴影遮罩 `shadow_mask`（主光/特写光阴影、SSAO）
* **区分依据**：**“模型自带的几何空间事实”**。只要是从顶点插值来的、或者只靠模型和相机位置算出来的，不含任何贴图采样值的，全在这里。

---

#### 阶段 2：美术材质输入 (`FSkinMaterialInputs`)
* **包含的数据**：
  * **颜色原材料**：美术固有色 `albedo`
  * **切线空间法线**：贴图解包并混合后的切线法线 `normal_ts`
  * **遮罩与参数**：美术粗糙度 `roughness`、高光度 `specular`、次表面散射强度 `subsurface_intensity`
* **区分依据**：**“纯美术原材料”**。直接从贴图采样解包出来的数据。法线还在**切线空间**，数值没有经过物理换算（比如粗糙度还没平方），没有任何世界坐标系概念。

---

#### 阶段 3：物理基质 / BSDF 数据 (`FSubstrateBSDF`)
* **包含的数据**：
  * **微表面世界法线**：经过 TBN 矩阵旋转后的最终着色法线 `WorldNormal`
  * **物理漫反射色**：能量守恒剥离后的 `DiffuseColor`
  * **物理高光反射率**：非金属基础反射率 $F_0$
  * **物理微表面粗糙度**：$\alpha = \text{roughness}^2$
  * **物理介质属性**：次表面散射强度 `SubsurfaceIntensity`
* **区分依据**：**“黑暗中静止存在的物理常数”**。只要**相机转动或灯光移动时，数值绝对不会发生改变**，并且已经把法线转到了**世界空间**，就必须放在这里。

---

#### 阶段 4：视线与光源数据 (`FBxDFContext/PreLightData`)
* **包含基本的数据**：
  * **视线夹角**：视线与世界法线的夹角余弦 $N \cdot V$
  * **视线几何遮蔽项**：Smith 几何项中只依赖视线的部分 $\mathbf{\Lambda}_V$（或 $G_1(V)$）
  * **反射向量**：世界反射向量 $R = \text{reflect}(-V, N)$
  * **环境光查找参数**：IBL 采样使用的 Mip 级别
* 普通的 PreLightData：“依赖相机视线 $V$，但不依赖具体某盏灯 $L$”**。只要摄像机一动它就会变，但不管场景里有 1 盏灯还是 10 盏灯，它只在进入灯光循环前**预先计算 1 次**。
- 而UE 的`FBxDFContext` 是在每盏灯的循环内部初始化的，而不是在灯光循环之前，**“依赖相机视线 $V$，依赖具体某盏灯 $L$”，所以还包含**光源夹角**：$N \cdot L$、$N \cdot H$、$V \cdot H$
---

#### 阶段 5：逐光源光照总线 (`EvaluateLighting`)
* **包含的数据**：
  * **微表面三项**：法线分布 $D$、几何遮挡 $G$、菲涅尔 $F$
  * **输出能量**：当前光源贡献的最终漫反射与高光 Radiance
* **区分依据**：**“依赖具体光源 $L$”**。场景里每多一盏灯，这个阶段的数据就要重新跑一遍。

---

### 二、各阶段核心依赖与数学正交法则表格

| 阶段编号与名称                                         | 数学依赖关系                                              | 依赖视线 $V$ 吗？    | 依赖具体光源 $L$ 吗？                 | 依赖空间域           | 核心职责判据                                          |
| :---------------------------------------------- | :-------------------------------------------------- | :------------- | :---------------------------- | :-------------- | :---------------------------------------------- |
| **阶段 1：几何上下文**<br>`PixelParameters`             | $f(\text{Vertex}, \text{CameraPos})$                | **是** (提取 $V$) | **否**                         | 世界 / NDC / 顶点拓扑 | 承接 GPU 插值事实，消除后续逻辑对底层 `Varyings` 寄存器的依赖。        |
| **阶段 2：材质解包**<br>`MaterialInputs`               | $f(\text{UV}, \text{PixelParameters})$              | **否**          | **否**                         | UV / 切线空间 (TS)  | 纯美术贴图解包，严禁引入世界变换与视线计算。                          |
| **阶段 3：基质构建**<br>`SubstrateBSDF`                | $f(\text{MaterialInputs}, \text{PixelParameters})$  | **否**          | **否**                         | 世界物理空间 (WS)     | 将 `normal_ts` 变换为世界法线，完成能量守恒转换。**黑暗静止中依然恒定不变**。 |
| **阶段 4：视线与光源数据**<br>`PreLightData/FBxDFContext` | $f(\text{SubstrateBSDF}, \text{PixelParamters})$    | **是**          | **Unity不依赖;UE的在每盏灯之前计算，UE依赖** | 视线光学空间          | **视线相关。                                         |
| **阶段 5：光照积分**<br>`LightingLoop`                 | $f(\text{SubstrateBSDF}, \text{FBxDFContext}, L_i)$ | **是**          | **是**                         | 逐光源积分总线         | 结合光源入射方向 $L$，求值微表面光照反射方程。                       |
|                                                 |                                                     |                |                               |                 |                                                 |

---

### 三、工业界四大顶级引擎命名对照表

| 逻辑阶段 | 语义本质 | Unreal Engine 5 (Substrate) | Google Filament | Unity HDRP | 推荐标准化命名 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **阶段 1** | 片元几何上下文 | `FMaterialPixelParameters` | `MaterialVertexInputs` | `PositionInputs` / `BuiltinData` | `FSkinPixelParameters` |
| **阶段 2** | 材质美术输入 | `FPixelMaterialInputs` | `MaterialInputs` | `SurfaceData` | `FSkinMaterialInputs` |
| **阶段 3** | 物理光学基质 | `FSubstrateBSDF` / `FGBufferData` | `PixelParams` | `BSDFData` | `FSubstrateBSDF` |
| **阶段 4** | 视线预计算缓存 | `FBxDFContext` | `ShadingContext` | `PreLightData` | `FBxDFContext` / `FSkinPreLightData` |
| **阶段 5** | 光照求值器 | `EvaluateBxDF` / `FDirectLighting` | `surfaceShading` | `EvaluateBSDF` | `EvaluateSkinLighting` |

---

### 四、结构体归属快速决策准则

遇到一个变量或计算公式不知如何归类时，依次代入以下三条判据：

1. **若变量由视线 $V$ 和粗糙度/法线共同求出**（例如 $N \cdot V$、Smith $\mathbf{\Lambda}_V$、反射向量 $R$、IBL 预积分）：
   * **必须放入【阶段 4：`FBxDFContext` / `PreLightData`】**。
2. **若变量是材质固有常数或世界微表面状态，不随视线 $V$ 和光源 $L$ 变动**（例如 $F_0$、$\alpha$、世界法线 `WorldNormal`、次表面散射强度）：
   * **必须放入【阶段 3：`FSubstrateBSDF`】**。
3. **若变量仅来自纹理采样，依然处于局部 UV 或切线空间**（例如切线法线 `normal_ts`、美术 Albedo、遮罩 Mask）：
   * **必须放入【阶段 2：`FSkinMaterialInputs`】**。
