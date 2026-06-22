

你好！作为 Unity 游戏开发工程师，**TBN 矩阵（Tangent, Bitangent, Normal Matrix）** 是特别是涉及 **法线贴图（Normal Mapping）** 和高级光照模型时，必须彻底掌握的核心概念。

简单来说，TBN 矩阵是一个 **坐标空间转换矩阵**，用于在 **切线空间（Tangent Space）** 和 **模型/世界空间（Object/World Space）** 之间转换向量。

以下是关于 TBN 矩阵的详细技术解析，涵盖定义、数学原理、Unity 中的特殊处理以及 Shader 实现。

---

### 1. 什么是 TBN？

TBN 构成了表面上每一点的一个 **局部坐标系**（也称为正交基）。

*   **T (Tangent，切线)**：
    *   方向：平行于表面。
    *   通常对应于纹理坐标的 **U (horizontal)** 方向。
    *   在 Unity 的 Mesh 数据中，以 `float4` 形式存储（xyz 为向量，w 为手性符号）。
*   **B (Bitangent/Binormal，副切线/副法线)**：
    *   方向：平行于表面，且垂直于 T 和 N。
    *   通常对应于纹理坐标的 **V (vertical)** 方向。
    *   在 Shader 中通常通过计算得出（很少直接存储在 Mesh 里以节省带宽）。
    *   *注：数学上应称为 Bitangent，但在某些图形 API（如旧版 DirectX）中被称为 Binormal，Unity 中通常混用，但现在倾向于使用 Bitangent。*
*   **N (Normal，法线)**：
    *   方向：垂直于表面。
    *   通常存储在 Mesh 数据中。

### 2. 为什么要用 TBN？（核心痛点）

#### 问题场景：法线贴图
法线贴图通常是**蓝紫色**的。为什么？因为法线贴图存储的是 **切线空间** 下的法线向量。
在切线空间中，由于表面被展平了，法线默认指向正上方 `(0, 0, 1)`，RGB 映射为 XYZ 后就是蓝色。

#### 矛盾点
*   **光照计算**（如点积 `dot(N, L)`）通常发生在 **世界空间（World Space）**。
*   **法线贴图采样** 得到的数据是在 **切线空间（Tangent Space）**。

#### 解决方案
你需要一个矩阵，把从贴图里透出来的法线（Tangent Space），转换到光照所在的空间（World Space）。这个矩阵就是 **TBN 矩阵**。

---

### 3. 数学原理

#### 3.1 矩阵构造
因为 T、B、N 三个向量本身就是两两垂直的单位向量（正交基），它们可以直接构成旋转矩阵。

如果我们想把一个向量从 **切线空间** 转换到 **世界空间**，矩阵构造如下（假设 T、B、N 已经是世界空间下的单位向量）：

$$
M_{TBN} = \begin{bmatrix}
T_x & B_x & N_x \\
T_y & B_y & N_y \\
T_z & B_z & N_z
\end{bmatrix}
$$

**变换公式：**
$$ N_{world} = M_{TBN} \times N_{tangent} $$

#### 3.2 逆变换（世界 -> 切线）
由于 TBN 矩阵通常是正交矩阵（Orthogonal Matrix），其**逆矩阵等于转置矩阵**。
如果我们要把光照方向（World Space）转到切线空间计算：

$$
M_{WorldToTangent} = M_{TBN}^T = \begin{bmatrix}
T_x & T_y & T_z \\
B_x & B_y & B_z \\
N_x & N_y & N_z
\end{bmatrix}
$$

---

### 4. Unity 开发中的关键细节

在 Unity 中手写 Shader 时，TBN 有几个非常容易踩坑的地方。

#### 4.1 手性（Handedness）与 Tangent.w
这是最重要的部分！
由于模型 UV 展开的方式不同，或者镜像模型的存在，TBN 坐标系可能是 **左手系** 也可能是 **右手系**。
*   Unity 的 `Mesh.tangents` 是 `Vector4`。
*   前三个分量 `xyz` 是切线方向。
*   **切线 `w` 分量**（值为 1 或 -1）用于决定副切线（Bitangent）的方向。

在 Vertex Shader 中计算副切线 B 的标准公式：
```hlsl
// N: 世界空间法线
// T: 世界空间切线
// T.w: 存储在Mesh中的符号
float3 B = cross(N, T.xyz) * T.w;
```
*如果不乘 `T.w`，镜像模型的法线贴图会是反的，看上去像凹进去或光照方向错误。*

#### 4.2 正交化（Re-orthogonalization）
由于光栅化插值（Vertex -> Fragment），在 Fragment Shader 中接收到的 T、B、N 往往不再垂直，甚至长度不是 1。为了高质量渲染，通常建议在 Fragment Shader 中进行 **格拉姆-施密特正交化（Gram-Schmidt process）**，但为了性能，通常只做简单的 Normalization，或者只在 Vertex Shader 算好传过去。

---

### 5. Shader 实现工作流（HLSL/URP 代码示例）

在现代 Unity 渲染（URP/HDRP）中，有两种主流处理方式。

#### 方式 A：在世界空间计算光照（主流，Pixel Shader 负担稍重）
将法线从切线空间转到世界空间。

**Vertex Shader:**
```hlsl
struct v2f {
    float4 pos : SV_POSITION;
    float2 uv : TEXCOORD0;
    // 传递 TBN 矩阵的三行，或者直接传三个向量
    float3 normalWS : TEXCOORD3;
    float3 tangentWS : TEXCOORD4;
    float3 bitangentWS : TEXCOORD5;
};

v2f vert (appdata v) {
    v2f o;
    // ... 坐标变换 ...
    
    // 1. 将法线和切线转到世界空间
    float3 worldNormal = TransformObjectToWorldNormal(v.normal);
    float3 worldTangent = TransformObjectToWorldDir(v.tangent.xyz);
    
    // 2. 计算副切线 (注意 T.w)
    float3 worldBitangent = cross(worldNormal, worldTangent) * v.tangent.w;

    o.normalWS = worldNormal;
    o.tangentWS = worldTangent;
    o.bitangentWS = worldBitangent;
    return o;
}
```

**Fragment Shader:**
```hlsl
half4 frag (v2f i) : SV_Target {
    // 1. 采样法线贴图
    float4 packedNormal = SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, i.uv);
    // UnpackNormal 会处理 DXT5nm 压缩等平台差异，并重映射到 [-1, 1]
    float3 tangentNormal = UnpackNormal(packedNormal);

    // 2. 构建 TBN 矩阵 (注意：必须重新归一化以防止插值导致的误差)
    float3 N = normalize(i.normalWS);
    float3 T = normalize(i.tangentWS);
    float3 B = normalize(i.bitangentWS);
    
    // 构建矩阵 (列向量)
    float3x3 TBN = float3x3(T, B, N);

    // 3. 转换法线到世界空间
    float3 worldNormal = mul(tangentNormal, TBN); // 注意 mul 的顺序: vector * matrix
    
    // ... 接下来的光照计算使用 worldNormal ...
}
```

#### 方式 B：在切线空间计算光照（旧技术，用于低端机优化）
将光的方向、视线方向从世界空间转到切线空间。
*   **优点**：Fragment Shader 省去了矩阵乘法，只做点积。所有的变换都在 Vertex Shader 完成。
*   **缺点**：不适合复杂的 PBR 环境反射（因为环境贴图通常是世界空间的）。

---

### 6. 计算 TBN 的一个常见误区：为什么不用几何着色器？
虽然几何着色器可以处理 TBN，但由于性能原因（GS 在许多硬件上效率极低），我们几乎总是在 Vertex Shader 计算 T 和 B，插值传给 Pixel Shader。

### 总结 TBN 矩阵
1.  **定义**：由切线(T)、副切线(B)、法线(N)组成的基底变换矩阵。
2.  **作用**：将切线空间下的法线（来自贴图）转换到世界空间，以便进行正确的光照计算。
3.  **Unity 关键点**：`B = cross(N, T) * T.w`。永远不要忘记 `T.w`。
4.  **性能**：构建 TBN 需要三个 `normalize` 和一次矩阵向量乘法，这在手机端是有开销的，但在现代 PBR 流程中是标准配置。

希望这个解释能帮你在 Unity 中从底层理解法线贴图的运作机制！如果有具体 Shader 代码报错，可以发给我看看。
