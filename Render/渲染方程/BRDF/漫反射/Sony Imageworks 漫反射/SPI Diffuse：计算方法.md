

这个问题问到了从“纸面微积分”落地到“引擎着色器代码”最核心的工程枢纽！

我们已经知道了耦合漫反射的完美公式是：
$$ f_{\text{diff}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = \frac{\mathbf{c}}{\pi} \cdot \frac{\Big( 1 - E(\mu_i) \Big) \cdot \Big( 1 - E(\mu_o) \Big)}{1 - E_{\text{avg}}} $$

那么，在运行时的 Shader 里，**$E(\mu_i)$、$E(\mu_o)$ 和 $E_{\text{avg}}$ 这三个数到底是怎么算出来的？**

为什么我们上一局只手算了 $\mu = 0$（$90^\circ$ 边缘）的极限，却没有手算出任意角度 $\mu \in (0, 1)$ 的通用公式？
因为对于任意的 $\mu$ 和粗糙度 $\alpha$，高光积分：
$$ E(\mu, \alpha, F_0) = \int_{\Omega} D(\mathbf{h}; \alpha) \cdot V(\mu_i, \mu; \alpha) \cdot F(\boldsymbol{\omega}_o \cdot \mathbf{h}; F_0) \cdot \mu_i \, d\omega_i $$
是一个**没有初等函数解析解的超越椭圆积分**！你不可能在 Pixel Shader 里每个像素跑几百次循环去实时做定积分。

为了在着色器里瞬间拿到 $E(\mu_i)$、$E(\mu_o)$ 和 $E_{\text{avg}}$，图形学界发明了一套极其漂亮的 **“线性拆解 + 预计算查表 / 多项式拟合”** 方案。我们分三步把它彻底拆开！

---
##
### 一、剥离 $F_0$（把 3 维问题降维成 2 维！）

你看上面的积分 $E(\mu, \alpha, F_0)$，它受 **3 个变量** 控制：角度余弦 $\mu$、粗糙度 $\alpha$、以及材质的基础反射率 $F_0$（比如水是 $0.02$，塑料是 $0.04$，钻石是 $0.17$）。
如果直接预计算，你需要一张三维贴图（3D LUT），而且如果 $F_0$ 是带颜色的 RGB 向量，还得查三次！

怎么把 $F_0$ 从积分号里面揪出来？
**直接把 Schlick 菲涅尔公式代进去，利用微积分的线性分配律拆成两项！**

已知 Schlick 公式可以写成关于 $F_0$ 和 $F_{90}$ 的线性组合（令 $u = \boldsymbol{\omega}_o \cdot \mathbf{h}$）：
$$ F(u) = F_0 + (F_{90} - F_0)(1 - u)^5 = \mathbf{F_0 \cdot \Big[ 1 - (1 - u)^5 \Big] + F_{90} \cdot (1 - u)^5} $$

把它塞进 $E(\mu)$ 的积分号里，因為 $F_0$ 和 $F_{90}$ 与积分变量 $\boldsymbol{\omega}_i$ 无关，直接提到积分号外面：

**$$ E(\mu, \alpha) = F_0 \cdot \underbrace{\int_{\Omega} D \cdot V \cdot \Big[ 1 - (1 - u)^5 \Big] \mu_i \, d\omega_i}_{\text{记为 } A(\mu, \alpha) \text{ (Scale 缩放项)}} \ + \ F_{90} \cdot \underbrace{\int_{\Omega} D \cdot V \cdot (1 - u)^5 \mu_i \, d\omega_i}_{\text{记为 } B(\mu, \alpha) \text{ (Bias 偏置项)}} $$**

**🚨 停下来看这神级的一步：**
原本复杂的积分，变成了一个极其简单的初中学过的一次线性方程：
**$$ E(\mu, \alpha) = F_0 \cdot A(\mu, \alpha) + F_{90} \cdot B(\mu, \alpha) $$**

*   **$A(\mu, \alpha)$ 和 $B(\mu, \alpha)$ 是什么？**
    它俩里面已经没有任何材质参数 $F_0$ 了，只跟角度 $\mu \in [0, 1]$ 和粗糙度 $\alpha \in [0, 1]$ 有关！
    而且你一定觉得它俩极其眼熟——**没错！这正是 Epic 2013 年发明的、全宇宙所有 PBR 引擎里都已经自带的那张 2D 环境光预积分贴图（EnvBRDF Split-Sum LUT）的红通道（`.x`）和绿通道（`.y`）！**
*   只要拿到 $(A, B)$：
    *   代入 $\mu_i = \mathbf{n} \cdot \boldsymbol{\omega}_i$，一乘一加就得到了 **$E(\mu_i)$**；
    *   代入 $\mu_o = \mathbf{n} \cdot \boldsymbol{\omega}_o$，一乘一加就得到了 **$E(\mu_o)$**！

---

### 二、同理拆解半球平均值 $E_{\text{avg}}$（降维成 1 维！）

既然 $E(\mu, \alpha) = F_0 \cdot A(\mu, \alpha) + F_{90} \cdot B(\mu, \alpha)$，那么我们对它求整个半球的平均值 $E_{\text{avg}} = 2 \int_0^1 E(\mu, \alpha) \mu \, d\mu$ 时，$F_0$ 和 $F_{90}$ 同样可以直接提出来：

**$$ E_{\text{avg}}(\alpha) = F_0 \cdot \underbrace{\left( 2 \int_0^1 A(\mu, \alpha) \mu \, d\mu \right)}_{\text{记为 } A_{\text{avg}}(\alpha)} \ + \ F_{90} \cdot \underbrace{\left( 2 \int_0^1 B(\mu, \alpha) \mu \, d\mu \right)}_{\text{记为 } B_{\text{avg}}(\alpha)} $$**

你看，因为角度 $\mu$ 被定积分积掉了，$A_{\text{avg}}(\alpha)$ 和 $B_{\text{avg}}(\alpha)$ 甚至连 2D 都不是，它们只是**仅关于粗糙度 $\alpha$ 的一维单变量曲线**！

#### 💡 一个让你拍案叫绝的特殊情况（当顶层高光已做多重散射补偿时）：
我们在前面算过，当表面绝对光滑（$\alpha = 0$）时，$A(\mu, 0) = 1 - (1 - \mu)^5$，$B(\mu, 0) = (1 - \mu)^5$。
把它们代入上面的半球平均积分里：
*   $B_{\text{avg}}(0) = 2 \int_0^1 (1 - \mu)^5 \mu \, d\mu = 2 \times \frac{1}{42} = \mathbf{\frac{1}{21}}$！
*   $A_{\text{avg}}(0) = 2 \int_0^1 \big[1 - (1 - \mu)^5\big] \mu \, d\mu = 1 - \frac{1}{21} = \mathbf{\frac{20}{21}}$！

所以，在光滑表面（或者当顶层高光已经通过 Turquin / Kulla-Conty 补回了多次散射丢失的能量，使得 $A + B \approx 1$）时，半球平均高光反射率 $E_{\text{avg}}$ 甚至连粗糙度都不怎么依赖，它直接约等于我们前两局算出来的那个黄金分数（这也是 Kulla-Conty 2017 讲义第 19 页给出的官方精确公式）：
**$$ E_{\text{avg}} \approx F_{\text{avg}} = \frac{20}{21} F_0 + \frac{1}{21} F_{90} ！！ $$**

---

### 三、在工程代码里，到底怎么算出 $A(\mu, r)$ 和 $B(\mu, r)$？

在现代引擎中，获取 $A(\mu, r)$、$B(\mu, r)$ 以及 $E_{\text{avg}}$ 一共有 **两条路线**：

#### 路线 1：查表法（LUT Texture Lookup —— UE5 / Filament 标准做法）
离线用蒙特卡洛积分生成一张只有 $32 \times 32$ 大小的双通道贴图 `BRDFLut`（横轴是 $\mu$，纵轴是 `roughness`，R 通道存 $A$，G 通道存 $B$，B 通道存平均值）：
```hlsl
// 查两次 2D 贴图（分别传入 NdotL 和 NdotV）
half2 AB_i = tex2D(BRDFLut, half2(NdotL, roughness)).rg;
half2 AB_o = tex2D(BRDFLut, half2(NdotV, roughness)).rg;

half E_i = F0 * AB_i.x + F90 * AB_i.y;
half E_o = F0 * AB_o.x + F90 * AB_o.y;
half E_avg = (20.0h / 21.0h) * F0 + (1.0h / 21.0h) * F90; // 或从 LUT 第3通道读取
```

#### 路线 2：纯数学多项式拟合法（零贴图采样！最适合你的《BRDF 理论代码大合集》！）
既然你在写一个纯粹的、不依赖外部贴图资源的 HLSL 数学代码库，如果为了算个漫反射还得在 C++ 里绑一张纹理贴图，那就太破坏代码库的纯粹性了！

好消息是：图形学界的大神（如 Brian Karis 2014、Dimitar Lazarov 2013 以及 OpenPBR 2024 团队）早就用**有理多项式与指数函数**，把 $A(\mu, r)$ 和 $B(\mu, r)$ 这两张曲面以 **小于 $1\%$ 的极高精度的纯数学公式** 拟合出来了！

我们直接把这套**无需任何贴图、纯数学闭式计算的 $E(\mu)$ 和 $E_{\text{avg}}$ 求解器**写出来：

```hlsl
// ===================================================
// 纯数学解析拟合计算高光定向反照率 E(μ) 与 半球平均值 E_avg
// 无需绑定外部 2D LUT 贴图，直接供 SPI 2017 / UE5 / OpenPBR 耦合漫反射调用
// ===================================================

// 1. 计算二元积分基函数 (A(μ, r), B(μ, r)) 的经典解析拟合 (Karis 2014 / Lazarov 2013)
//    满足 E(μ) = F0 * AB.x + F90 * AB.y
//    验算边界：当 roughness = 0 时，AB.x ≈ 1 - (1 - μ)⁵, AB.y ≈ (1 - μ)⁵，与理论极限完全吻合！
half2 IntegrateBRDF_Analytic(half roughness, half mu)
{
    const half4 c0 = half4(-1.0h, -0.0275h, -0.572h, 0.022h);
    const half4 c1 = half4( 1.0h,  0.0425h,  1.040h, -0.040h);
    half4 r = roughness * c0 + c1;
    half a004 = min(r.x * r.x, exp2(-9.28h * mu)) * r.x + r.y;
    half2 AB = half2(-1.04h, 1.04h) * a004 + r.zw;
    return saturate(AB);
}

// 2. 计算特定角度余弦 mu (即 NdotL 或 NdotV) 下的高光定向反照率 E(mu)
half ComputeSpecularAlbedo_E(half F0_dielectric, half F90_dielectric, half roughness, half mu)
{
    half2 AB = IntegrateBRDF_Analytic(roughness, mu);
    return F0_dielectric * AB.x + F90_dielectric * AB.y;
}

// 3. 计算高光在整个半球上的平均反照率 E_avg = 2 * ∫_0^1 E(μ) * μ dμ
//    对上面的 AB(μ, r) 关于 2μ dμ 做解析积分拟合：
//    - 当顶层高光包含多重散射补偿时，总能量守恒，E_avg 恒等于 Kulla-Conty 精确解：(20/21)*F0 + (1/21)*F90
//    - 若顶层高光仅为单次散射，粗糙度增大时存在遮蔽能量衰减，用二次多项式精确拟合 A_avg(r) 与 B_avg(r)
half ComputeSpecularAlbedo_Eavg(half F0_dielectric, half F90_dielectric, half roughness)
{
    // Kulla-Conty 2017 官方半球平均菲涅尔精确分数 (2 * ∫_0^1 (1 - μ)⁵ μ dμ = 1/21)
    const half B_avg_smooth = 1.0h / 21.0h;  // ≈ 0.04762
    const half A_avg_smooth = 20.0h / 21.0h; // ≈ 0.95238
    
    // 随粗糙度变化的单次散射能量衰减修正 (当 roughness=0 时精确还原 20/21 与 1/21)
    half r2 = roughness * roughness;
    half A_avg = A_avg_smooth * (1.0h - 0.28h * r2);
    half B_avg = B_avg_smooth * (1.0h - 0.35h * roughness);
    
    return F0_dielectric * A_avg + F90_dielectric * B_avg;
}
```


---

### 四、微积分推导：如何从零写出预计算 $E(\mu_o, \alpha)$ 的代码
#### SPI 2017 用的是查表法吗？第二种拟合又是谁在用？
##### 1. SPI 2017（索尼影业）用的是 100% 的【预计算查表法（LUT）】！
在 Kulla & Conty 2017 的原版讲义（第 18～28 页）中，因为他们写的是电影离线渲染器（Arnold），内存极其充裕，所以他们直接预计算了查找表：
*   **针对金属多重散射（先把 $F$ 设为 $1$）**：他们预计算了一张极小的 **$32 \times 32$ 分辨率的 2D 表**存 $E(\mu, \alpha)$，以及一张 **$32 \times 1$ 的 1D 表**存 $E_{\text{avg}}(\alpha)$。
*   **针对非金属耦合漫反射（Coupled Diffuse）**：因为电影里用的是精确麦克斯韦菲涅尔（不仅有外部反射，还有内部向外的全反射 TIR），无法像 Schlick 那样简单把 $F_0$ 线性提出来，所以 SPI 直接预计算了一张 **三维查找表（3D LUT）：$E(\mu, \alpha, \text{ior})$**，三个维度分别是角度余弦 $\mu$、粗糙度 $\alpha$ 和折射率 $\text{ior}$！
*   **而在实时游戏引擎（如 UE5 / Filament）中**：因为游戏引擎使用的是 Schlick 菲涅尔，满足线性拆分 $E(\mu, \alpha) = F_0 \cdot A(\mu, \alpha) + F_{90} \cdot B(\mu, \alpha)$，所以实时引擎直接**复用了现成的 2D 环境光贴图（EnvBRDF Split-Sum LUT）**，连新表都不用建！

##### 2. 第二种“纯数学多项式拟合”是什么地方在使用？
上一局给你看的那几行常数拟合代码，在工业界有两大主力应用场景：
1. **移动端与前向渲染管线（UE4/UE5 的 `EnvBRDFApprox` 与使命召唤）**：
   那段拟合公式最早由 **Dimitar Lazarov（2013，《使命召唤：黑色行动2》）** 提出，随后被 **Brian Karis 收录进了虚幻引擎的 `BRDF.ush`（函数名叫 `EnvBRDFApprox`）**，以及 Google Filament 的 `DFGApprox`。在移动端或不想额外占用一个贴图采样器（Texture Sampler）的 Shader 里，直接调用这几行指令代替查表。
2. **跨软件材质标准库（MaterialX 与 OpenPBR 2024）**：
   像 Autodesk 和 Adobe 联合制定的 **OpenPBR / MaterialX**，它的材质节点需要在 Maya、Blender、Houdini、Unreal 等十几个不同软件之间到处传递。如果为了一小段漫反射公式还要随身携带一张外部 `.exr` 贴图文件，在资产管理上是极大的麻烦。因此，MaterialX 官方源码里专门写了一个纯数学有理分式拟合函数 `mx_ggx_dir_albedo_analytic()`，直接在代码里硬算出 $E(\mu, \alpha)$！

---

#### 如何从零写出预计算 $E(\mu_o, \alpha)$ 的代码？

现在进入最硬核、最过瘾的环节：**如果我们不用任何现成的表，也不背任何拟合魔数，我们自己怎么写一个预计算程序（Compute Shader 或离线烘焙器），把 $E(\mu_o, \alpha)$ 和 $E_{\text{avg}}(\alpha)$ 算出来？**

我们要计算的半球积分是（固定出射视线余弦 $\mu_o = \mathbf{n}\cdot\boldsymbol{\omega}_o$，对上半球所有入射方向 $\boldsymbol{\omega}_i$ 积分）：
$$ E(\mu_o, \alpha) = \int_{\Omega_i} \underbrace{D(\mathbf{h}) \cdot V(\mu_i, \mu_o) \cdot F(\boldsymbol{\omega}_o \cdot \mathbf{h})}_{f_{\text{spec}}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)} \cdot \mu_i \, d\omega_i $$

如果我们在半球上“均匀随机”生成光线 $\boldsymbol{\omega}_i$ 去算这个积分，当粗糙度 $\alpha$ 很小时，$D(\mathbf{h})$ 是一根极细的针，你随机射 10000 根光线可能有 9999 根都落在 $D(\mathbf{h}) \approx 0$ 的地方，算出来全是噪点。

所以，我们必须使用蒙特卡洛 **重要性采样（Importance Sampling）**！推导只需三步，而且中间会发生一次**美妙绝伦的代数大抵消**！

##### 第一步：用“逆变换采样（Inverse CDF）”生成符合 GGX 分布的半角向量 $\mathbf{h}$

还记得我们在前面证明过的“球面微表面投影面积守恒”吗？
虽然 $D(\mathbf{h})$ 本身不是概率密度函数（PDF），但乘上 $\cos\theta_h = (\mathbf{n}\cdot\mathbf{h})$ 之后：
$$ p_H(\mathbf{h}) = D(\mathbf{h}) (\mathbf{n} \cdot \mathbf{h}) $$
它的半球积分 $\int_{\Omega_h} D(\mathbf{h})(\mathbf{n}\cdot\mathbf{h}) \, d\omega_h \equiv 1$，是一个**完美合法的概率密度函数（PDF）**！

在球坐标系 $(\theta_h, \phi_h)$ 下，给你两个 $[0, 1)$ 之间的均匀随机数 $(\xi_1, \xi_2)$，怎么生成服从 $p_H(\mathbf{h})$ 的角度 $(\theta_h, \phi_h)$？
1. **方位角 $\phi_h$（各向同性均匀分布）**：
   $$ \phi_h = 2\pi \xi_1 $$
2. **极角 $\theta_h$（对极角概率密度求累积分布积分 CDF，并令其等于 $\xi_2$）**：
   $$ \text{CDF}(\theta_h) = \int_0^{2\pi} d\phi \int_0^{\theta_h} \frac{\alpha^2 \cos\theta \sin\theta}{\pi \big(\cos^2\theta(\alpha^2 - 1) + 1\big)^2} \, d\theta = \frac{1 - \cos^2\theta_h}{\cos^2\theta_h(\alpha^2 - 1) + 1} = \xi_2 $$
   解这个关于 $\cos^2\theta_h$ 的一元一次方程，直接开根号得到：
   **$$ \cos\theta_h = \sqrt{\frac{1 - \xi_2}{1 + (\alpha^2 - 1)\xi_2}}, \qquad \sin\theta_h = \sqrt{1 - \cos^2\theta_h} $$**

拿到 $(\theta_h, \phi_h)$ 后，拼出半角向量 $\mathbf{h}$，再把视线 $\boldsymbol{\omega}_o$ 绕着 $\mathbf{h}$ 做镜面反射，就得到了入射光线方向 $\boldsymbol{\omega}_i$：
$$ \boldsymbol{\omega}_i = 2(\boldsymbol{\omega}_o \cdot \mathbf{h})\mathbf{h} - \boldsymbol{\omega}_o $$

##### 第二步：代入雅可比矩阵，算出采样 $\boldsymbol{\omega}_i$ 的真实概率密度 $p_i(\boldsymbol{\omega}_i)$

注意：我们的积分是对入射光立体角 **$d\omega_i$** 积的，但我们的随机向量 $\mathbf{h}$ 是在半角立体角 **$d\omega_h$** 里按 $p_H(\mathbf{h}) = D(\mathbf{h})(\mathbf{n}\cdot\mathbf{h})$ 抽出来的！
根据我们早就推导熟透的**反射雅可比行列式** $d\omega_i = 4(\boldsymbol{\omega}_o \cdot \mathbf{h}) \, d\omega_h$，换算到 $\boldsymbol{\omega}_i$ 空间的概率密度 $p_i(\boldsymbol{\omega}_i)$ 是：

**$$ p_i(\boldsymbol{\omega}_i) = p_H(\mathbf{h}) \cdot \left| \frac{d\omega_h}{d\omega_i} \right| = \frac{D(\mathbf{h}) (\mathbf{n} \cdot \mathbf{h})}{4 (\boldsymbol{\omega}_o \cdot \mathbf{h})} $$**

##### 第三步：蒙特卡洛相除，见证 $D(\mathbf{h})$ 的完美抵消！

在蒙特卡洛积分 $\int_{\Omega_i} g(\boldsymbol{\omega}_i) \, d\omega_i \approx \frac{1}{N} \sum_{k=1}^N \frac{g(\boldsymbol{\omega}_{i,k})}{p_i(\boldsymbol{\omega}_{i,k})}$ 中，我们每次采样的权重就是拿**“被积函数 $f_{\text{spec}} \cdot \mu_i$”**除以**“概率密度 $p_i(\boldsymbol{\omega}_i)$”**：

$$ \text{单次采样权重 Weight} = \frac{D(\mathbf{h}) \cdot V(\mu_i, \mu_o) \cdot F(\boldsymbol{\omega}_o \cdot \mathbf{h}) \cdot \mu_i}{p_i(\boldsymbol{\omega}_i)} = \frac{\cancel{D(\mathbf{h})} \cdot V(\mu_i, \mu_o) \cdot F(\boldsymbol{\omega}_o \cdot \mathbf{h}) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_i)}{\frac{\cancel{D(\mathbf{h})} (\mathbf{n} \cdot \mathbf{h})}{4 (\boldsymbol{\omega}_o \cdot \mathbf{h})}} $$

**🔥 请看上面这个分式发生了什么奇迹：**
那个在光滑时会导致除以零爆炸的罪魁祸首 **$D(\mathbf{h})$**，在分子和分母上**被 100% 约分消掉了**！
把分母的 $4(\boldsymbol{\omega}_o \cdot \mathbf{h})$ 翻到分子上来，单次采样的权重变成了极其干净、绝对不会爆炸的式子：

**$$ \text{Weight} = F(\boldsymbol{\omega}_o \cdot \mathbf{h}) \cdot \underbrace{\left[ \frac{4 \cdot V(\mu_i, \mu_o) \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot (\boldsymbol{\omega}_o \cdot \mathbf{h})}{\mathbf{n} \cdot \mathbf{h}} \right]}_{\text{记为几何可见权重 } G_{\text{vis}}} $$**

*(注：如果用 $G(\mu_i, \mu_o) = 4 \mu_i \mu_o V(\mu_i, \mu_o)$ 来写，括号里的 $G_{\text{vis}}$ 就是经典的 $\frac{G(\mu_i, \mu_o) (\boldsymbol{\omega}_o \cdot \mathbf{h})}{(\mathbf{n} \cdot \boldsymbol{\omega}_o)(\mathbf{n} \cdot \mathbf{h})}$！)*

而且更妙的是：
*   如果我们把 $F = 1$ 代入，累加求平均算出来的就是 **Kulla-Conty 的纯几何高光反照率 $E_{F=1}(\mu_o, \alpha)$**；
*   如果我们把 Schlick 的基函数 $1 - (1 - \boldsymbol{\omega}_o\cdot\mathbf{h})^5$ 和 $(1 - \boldsymbol{\omega}_o\cdot\mathbf{h})^5$ 分别代入，算出来的就是 **Split-Sum 的两个通道 $A(\mu_o, \alpha)$ 和 $B(\mu_o, \alpha)$**（且满足 $A + B \equiv E_{F=1}$）！

---

### 五、 手写完整的 HLSL 预计算代码（含 $E(\mu, \alpha)$ 与 $E_{\text{avg}}(\alpha)$）

现在，我们把上面的推导直接写成一段可以在任何 Compute Shader 或 Pixel Shader 中运行的**标准烘焙代码**。它包含三个核心函数：

```hlsl
// =========================================================================
// 预计算烘焙模块：蒙特卡洛重要性采样计算 E(μ, α) 与 E_avg(α)
// =========================================================================

// 1. 生成 Hammersley 低差异准随机序列 (比普通伪随机数收敛快得多)
//    输入当前采样序号 i 与总采样数 N，输出 [0, 1)^2 的二维均匀采样点 (xi_1, xi_2)
float2 Hammersley2D(uint i, uint N)
{
    // Van der Corput 序列 (将二进制位镜像反转到小数点后)
    uint bits = (i << 16u) | (i >> 16u);
    bits = ((bits & 0x55555555u) << 1u) | ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u) | ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u) | ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u) | ((bits & 0xFF00FF00u) >> 8u);
    float radical_inverse = float(bits) * 2.3283064365386963e-10; // 除以 2^32
    return float2(float(i) / float(N), radical_inverse);
}

// 2. GGX 法线分布 (NDF) 逆变换重要性采样
//    在切线空间 (N = (0, 0, 1)) 下，根据概率密度 p(H) = D(H) * (N·H) 生成半角向量 H
float3 ImportanceSampleGGX_NDF(float2 xi, float roughness)
{
    float alpha = roughness * roughness;
    float alpha2 = alpha * alpha;

    // 方位角 phi = 2 * PI * xi.x
    float phi = 2.0 * PI * xi.x;

    // 极角余弦 cosTheta = sqrt((1 - xi.y) / (1 + (alpha^2 - 1) * xi.y))
    float cos_theta = sqrt((1.0 - xi.y) / (1.0 + (alpha2 - 1.0) * xi.y));
    float sin_theta = sqrt(saturate(1.0 - cos_theta * cos_theta));

    // 球坐标转切线空间直角坐标
    return float3(sin_theta * cos(phi), sin_theta * sin(phi), cos_theta);
}

// 3. 预计算特定视角 mu_o (即 N·V) 和粗糙度 roughness 下的定向反照率 E(mu_o, roughness)
//    返回值 float3:
//      .x = A(mu_o, r)  --> 对应 F0 的缩放权重: ∫ D * V * (1 - (1 - V·H)⁵) * (N·L) dω_i
//      .y = B(mu_o, r)  --> 对应 F90 的偏置权重: ∫ D * V * (1 - V·H)⁵ * (N·L) dω_i
//      .z = E_F1(mu_o, r) = A + B --> 对应 Kulla-Conty 白炉 (F=1) 的纯几何定向反照率！
float3 PrecomputeSpecularAlbedo_E(float mu_o, float roughness, uint num_samples = 1024u)
{
    // 在切线空间下，法线 N = (0, 0, 1)，设视线 V 落在 XZ 平面内：
    mu_o = max(mu_o, 1e-4);
    float3 V = float3(sqrt(1.0 - mu_o * mu_o), 0.0, mu_o);

    float sum_A = 0.0;
    float sum_B = 0.0;

    for (uint i = 0u; i < num_samples; ++i)
    {
        // Step A: 生成低差异随机点并采样出微表面半角向量 H
        float2 xi = Hammersley2D(i, num_samples);
        float3 H  = ImportanceSampleGGX_NDF(xi, roughness);

        // Step B: 将视线 V 绕 H 对称反射，得到入射光方向 L = 2 * (V·H) * H - V
        float VdotH = saturate(dot(V, H));
        float3 L    = 2.0 * VdotH * H - V;

        // 在切线空间下，N = (0, 0, 1)，因此 N·L 就是 L.z，N·H 就是 H.z
        float NdotL = saturate(L.z);
        float NdotH = saturate(H.z);

        // Step C: 只有当反射出的入射光 L 位于上半球 (NdotL > 0) 时才有效
        if (NdotL > 0.0)
        {
            // 调用我们之前写好的高度相关精确 Smith V 项: V = G / (4 * NdotL * NdotV)
            float V_smith = V_SmithJoint_GGX_Frostbite(roughness, NdotL, mu_o);

            // 代入蒙特卡洛相除抵消 D(H) 后的几何权重公式:
            // G_vis = 4 * V_smith * NdotL * VdotH / NdotH
            float G_vis = (4.0 * V_smith * NdotL * VdotH) / max(NdotH, 1e-5);

            // 拆分为 F0 项与 F90 项分别累加
            float Fc = pow(1.0 - VdotH, 5.0);
            sum_A += (1.0 - Fc) * G_vis;
            sum_B += Fc * G_vis;
        }
    }

    float A = sum_A / float(num_samples);
    float B = sum_B / float(num_samples);
    return float3(A, B, A + B); // .z 通道直接就是 Kulla-Conty 的 E(mu_o) (当 F=1 时)
}

// 4. 预计算半球平均反照率 E_avg(roughness) = 2 * ∫_0^1 E(μ, roughness) * μ dμ
//    使用中点黎曼求和 (Midpoint Riemann Sum) 对 μ ∈ [0, 1] 进行一维数值积分
float3 PrecomputeSpecularAlbedo_Eavg(float roughness, uint num_mu_steps = 64u, uint num_mc_samples = 512u)
{
    float3 sum_avg = float3(0.0, 0.0, 0.0);
    float delta_mu = 1.0 / float(num_mu_steps);

    for (uint j = 0u; j < num_mu_steps; ++j)
    {
        // 取每个积分小区间的中点 mu_j ∈ (0, 1)
        float mu_j = (float(j) + 0.5) * delta_mu;

        // 算出该角度下的 (A, B, E_F1)
        float3 E_mu = PrecomputeSpecularAlbedo_E(mu_j, roughness, num_mc_samples);

        // 累加被积函数: 2 * E(mu) * mu * d(mu)
        sum_avg += 2.0 * E_mu * mu_j * delta_mu;
    }

    // 返回值 .x = A_avg, .y = B_avg, .z = E_F1_avg
    // 对于任意材质 F0 和 F90，最终 E_avg = F0 * sum_avg.x + F90 * sum_avg.y
    return sum_avg;
}
```

---

### 六、 底层的漫反射本身会变成各向异性吗？

上一局我们之所以能把半球的二维积分 $\int_0^{2\pi} d\phi \int_0^1 d\mu$ 变成一维的 $2\int_0^1 d\mu$，全靠**“各向同性（Isotropic）”**——因为围着法线转一圈，每个方位角 $\phi$ 的高光反射率 $E(\mu)$ 都是一样的，所以 $\int_0^{2\pi} d\phi = 2\pi$ 被直接提了出来！

**那如果顶层高光是【各向异性（Anisotropic，$\alpha_x \neq \alpha_y$）】的呢？！**
比如一块拉丝塑料、黑胶唱片、或者涂了各向异性清漆的木头：
当你顺着划痕方向看（$\phi = 0^\circ$）和垂直于划痕方向看（$\phi = 90^\circ$），微表面的粗糙度完全不同，高光弹走的能量 $E$ 也完全不同！

这时候会发生什么？工业界（SPI 2017、UE5、OpenPBR 2024）又是怎么解决这个难题的？我们分三层彻底揭晓！

首先明确一个物理事实：
哪怕物体表面的微观划痕是各向异性的（$\alpha_x \neq \alpha_y$），但当光子穿透这层划痕、钻进底下的色素颗粒层之后，光子在色素之间经历了几十上百次随机碰撞，**它依然会彻底迷失方向（包括忘记自己是从哪个方位角 $\phi$ 进来的）**！

所以：
1. **底层色素的散射内核**：依然是各向同性的均匀散射 $\frac{\mathbf{c}}{\pi}$；
2. **发生变化的地方**：是头顶那两道**“进出透射门” $\big(1 - E(\boldsymbol{\omega}_i)\big)$ 和 $\big(1 - E(\boldsymbol{\omega}_o)\big)$**，以及天花板内壁的平均挡回率 **$E_{\text{avg}}$**！

#### 数学危机：各向异性引发的“4 维维度灾难”！

在各向异性下，高光反射走的总能量 $E$ 不再只依赖天顶角余弦 $\mu = \cos\theta$，它还依赖**方位角 $\phi$**，以及两个轴的粗糙度 **$\alpha_x, \alpha_y$**！

1. **进出大门变成了 4 维函数**：
   $$ E = E(\mu, \ \phi, \ \alpha_x, \ \alpha_y) $$
   在预计算中，一张 4 维的查找表（4D LUT）在内存和带宽上是绝对不可接受的！

2. **半球平均值 $E_{\text{avg}}$ 不能再直接把 $\phi$ 约成 $2\pi$ 了**：
   因为 $E(\mu, \phi)$ 随方位角 $\phi$ 变化，所以天花板内壁挡回来的平均比例 $E_{\text{avg}}$，必须老老实实保留对 $\phi \in [0, 2\pi]$ 的积分：
   **$$ E_{\text{avg}}(\alpha_x, \alpha_y) = \frac{1}{\pi} \int_0^{2\pi} d\phi \int_0^1 E(\mu, \phi, \alpha_x, \alpha_y) \cdot \mu \, d\mu $$**

面对这个无法直接查 2D 表的 4 维难题，**SPI 2017（Kulla-Conty）**、**UE5** 和 **OpenPBR 2024** 拿出了两套极其聪明的降维打击方案！


#### 工业界的破局之道（两套降维打击方案）

##### 方案 1：SPI 2017 / UE5 / OpenPBR 的标准做法 ——“平均粗糙度守恒”直接白嫖 2D 表！

**还记得我们在讲各向异性参数化时埋下的那个巨大伏笔吗？！**
当时我们问：为什么 **SPI 2017** 要把粗糙度设计成 $\alpha_x = \alpha(1+a), \ \alpha_y = \alpha(1-a)$（保持算术平均 $\frac{\alpha_x + \alpha_y}{2} = \alpha$ 不变）？为什么 **OpenPBR 2024** 要设计成保持平方平均 $\sqrt{\frac{\alpha_x^2 + \alpha_y^2}{2}} = \alpha$ 不变？

**答案就在这里兑现了！**

2017 年，Kulla 和 Conty 在论文里专门用蒙特卡洛积分测试了各向异性的 $E_{\text{avg}}(\alpha_x, \alpha_y)$。他们发现了一个惊人的数学巧合：
当你对上面那个二重积分里的方位角 $\phi \in [0, 2\pi]$ 转一圈求平均时：
*   在 $\phi = 0^\circ$（顺着 $\alpha_x$）方向上，粗糙度变大了，高光 $E$ 的变化趋势往一边偏；
*   在 $\phi = 90^\circ$（顺着 $\alpha_y$）方向上，粗糙度变小了，高光 $E$ 的变化趋势刚好往相反方向偏；
*   **一圈 $\int_0^{2\pi} d\phi$ 积分平均下来，长轴和短轴的偏差在半球上几乎完美互相抵消了！**

数值实验证明：各向异性的半球平均值 $E_{\text{avg}}(\alpha_x, \alpha_y)$，与直接使用**平均粗糙度 $\alpha_{\text{avg}}$** 算出来的各向同性 $E_{\text{avg}}(\alpha_{\text{avg}})$，误差小到肉眼完全无法分辨（通常小于 $1\% \sim 2\%$）！

因此，在 **SPI 2017、Filament、UE5 和 OpenPBR 2024** 的真实引擎代码里：
**哪怕顶层高光是各向异性的，在计算耦合漫反射的 $E(\mu_i)$、$E(\mu_o)$ 和 $E_{\text{avg}}$ 时，引擎根本不去建什么 4D 表，而是直接拿基础粗糙度 `roughness`（即平均粗糙度），原封不动地去查原来那张各向同性的 2D LUT 表（或调同一个解析拟合函数）！**

##### 方案 2：进阶高精度做法 —— 用“椭圆截面投影粗糙度 $\alpha_{\text{proj}}(\phi)$”查 2D 表！

如果你是一位有极致物理洁癖的架构师，你说：
*“好，分母的 $E_{\text{avg}}$ 因为绕了一整圈 $\phi \in [0, 2\pi]$ 求平均，用平均粗糙度 $\alpha$ 我认了；*
*但是分子上的出射门 $1 - E(\boldsymbol{\omega}_o)$ 只看视线这**一个固定方位角 $\phi_o$**！如果我顺着划痕看（极光滑，$\alpha_y \to 0$）和垂直着划痕看（极粗糙，$\alpha_x \to 1$），边缘的漫反射透射门难道不应该不一样吗？！”*

**完全正确！而且我们连 4D 表都不用建，直接用之前推导过的【椭圆投影有效粗糙度】就能在 2D 表里算出来！** 注意！在数学上依然不是相等的，只是近似

还记得我们在推导各向异性 Smith G 项的第二种解法时发现的规律吗？
当一根视线 $\boldsymbol{\omega}_o$ 从特定的方位角 $\phi_o$ 切过半轴为 $(\alpha_x, \alpha_y)$ 的椭圆山脉时，这根视线在它自己的平面上看到的**一维等效粗糙度 $\alpha(\boldsymbol{\omega}_o)$** 是：

$$ \alpha(\boldsymbol{\omega}_o) = \sqrt{\alpha_x^2 \cos^2\phi_o + \alpha_y^2 \sin^2\phi_o} = \sqrt{\frac{\alpha_x^2 (\mathbf{t}\cdot\boldsymbol{\omega}_o)^2 + \alpha_y^2 (\mathbf{b}\cdot\boldsymbol{\omega}_o)^2}{1 - (\mathbf{n}\cdot\boldsymbol{\omega}_o)^2}} $$

于是，我们只需要：
1. 对入射方向 $\boldsymbol{\omega}_i$ 算出它的投影粗糙度 $r_i = \sqrt{\alpha(\boldsymbol{\omega}_i)}$，去查原来的 **2D 表**得到 **$E(\mu_i, \ r_i)$**；
2. 对出射视线 $\boldsymbol{\omega}_o$ 算出它的投影粗糙度 $r_o = \sqrt{\alpha(\boldsymbol{\omega}_o)}$，去查原来的 **2D 表**得到 **$E(\mu_o, \ r_o)$**；
3. 对分母的半球平均值 $E_{\text{avg}}$，直接用平均粗糙度 $r_{\text{avg}} = \sqrt{\frac{\alpha_x + \alpha_y}{2}}$ 查原来的 **1D 表**！

```hlsl
// =========================================================================
// 各向异性高精度耦合漫反射：利用方向投影粗糙度直接复用 2D 各向同性拟合/查表
// =========================================================================

// 计算任意方向向量 W (即 L 或 V) 在椭圆切平面 (ax, ay) 上的等效感知粗糙度
half GetProjectedPerceptualRoughness(half3 W, half3 T, half3 B, half NdotW, half ax, half ay)
{
    half TdotW = dot(T, W);
    half BdotW = dot(B, W);
    half sin2_theta = max(1.0h - NdotW * NdotW, 1e-5h); // 即 TdotW² + BdotW²
    
    // 椭圆在该方位角切面上的等效物理粗糙度平方: alpha_proj² = (ax² * cos²φ + ay² * sin²φ)
    half alpha_proj = sqrt((ax * ax * TdotW * TdotW + ay * ay * BdotW * BdotW) / sin2_theta);
    
    // 开根号转回感知粗糙度 perceptualRoughness 供 2D LUT 或拟合函数使用
    return sqrt(saturate(alpha_proj));
}
```
你看：**顺着光滑轴看去（$\alpha_{\text{proj}} \to 0$），边缘高光变成 $1.0$，漫反射在边缘自动降为 $0$；顺着粗糙轴看去（$\alpha_{\text{proj}} \to 1$），边缘高光变弱，漫反射在边缘自动变亮！** 依然只用原来那张 2D 表，就完美实现了随方位角 $\phi$ 动态变化的各向异性漫反射透射门！

##### 如果真要在离线预计算代码里精确算各向异性的 $E$，该改哪一行？

最后，回到你上一局让我写的那个蒙特卡洛预计算代码 `PrecomputeSpecularAlbedo_E`：
为什么在蒙特卡洛预计算里，算各向异性根本不需要操心 $\int_0^{2\pi} d\phi$ 的问题？

因为蒙特卡洛采样本来就是在整个二维半球 $(\xi_1, \xi_2) \in [0, 1)^2$ 上撒点，其中 **$\phi_h = 2\pi \xi_1$ 已经天然把 $0 \sim 2\pi$ 的所有方位角全遍历积分了一遍**！
如果你想在预计算器里真刀真枪地算出精确的各向异性 $E(\mu_o, \phi_o, \alpha_x, \alpha_y)$，只需要把抽样半角向量 $\mathbf{h}$ 的函数改成椭圆拉伸版（3 行代码）：

```hlsl
// 各向异性 GGX NDF 精确逆变换重要性采样 (Heitz 2014)
float3 ImportanceSampleGGX_Aniso_NDF(float2 xi, float ax, float ay)
{
    float phi = 2.0 * PI * xi.x;
    
    // 在斜率空间直接按 ax 和 ay 对单位圆斜率进行椭圆拉伸，再归一化回球面向量 H！
    float r = sqrt(xi.y / (1.0 - xi.y)); // 各向同性单位圆斜率半径 tan(theta)
    float3 slope_H = float3(ax * r * cos(phi), ay * r * sin(phi), 1.0);
    return normalize(slope_H);
}
```

