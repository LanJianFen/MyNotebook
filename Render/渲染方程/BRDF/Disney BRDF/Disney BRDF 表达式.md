

既然我们要聊 Disney BRDF，那我们就把视线从屏幕空间的经验近似（5S）拉回**局部光照模型与微表面理论（Microfacet Theory）**的严谨数学体系中。

作为渲染工程师，我必须首先纠正一个常见的观念偏差：**Disney BRDF 并不是一个由单一物理定律推导出的绝对完美的方程式**。相反，它是 Brent Burley 在 2012 年基于 MERL（三菱电机研究院）测量的真实材质数据库，经过大量**非线性回归拟合**，并结合美术直觉构建出的一个**“现象学与物理学折中”**的混合光照模型。

它在数学上是一个**分块的多波瓣（Multi-Lobe）线性组合**。Disney BRDF 的核心方程可以表示为漫反射（Diffuse）、高光（Specular）、清漆（Clearcoat）和光泽（Sheen）四个主要项的叠加：

$$
f(\mathbf{l}, \mathbf{v}) = (1 - metallic) \cdot f_{diffuse} + f_{specular} + f_{clearcoat} + (1 - metallic) \cdot f_{sheen}
$$

我们定义光照方向 $\mathbf{l}$，观察方向 $\mathbf{v}$，法线 $\mathbf{n}$，半角向量 $\mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|}$。
定义各夹角余弦值：$\cos\theta_l = (\mathbf{n} \cdot \mathbf{l})$，$\cos\theta_v = (\mathbf{n} \cdot \mathbf{v})$，$\cos\theta_h = (\mathbf{n} \cdot \mathbf{h})$，$\cos\theta_d = (\mathbf{l} \cdot \mathbf{h}) = (\mathbf{v} \cdot \mathbf{h})$。

下面我们逐一拆解这些项的数学表达。

---

### 1. 漫反射项 (Diffuse Lobe)
传统的 Lambert 漫反射是一个常数 $f_d = \frac{BaseColor}{\pi}$。但迪士尼发现真实材质（特别是粗糙表面）在掠射角（Grazing Angle）会产生**背向散射（Retro-reflection）**，即光线沿着视线方向反射回来。

因此，Burley 用 Fresnel 形式构造了一个经验公式：
$$
f_{diffuse} = \frac{BaseColor}{\pi} \left( 1 + (F_{D90} - 1)(1 - \cos\theta_l)^5 \right) \left( 1 + (F_{D90} - 1)(1 - \cos\theta_v)^5 \right)
$$
其中，掠射角反射率 $F_{D90}$ 由表面粗糙度（Roughness）决定：
$$
F_{D90} = 0.5 + 2 \cdot roughness \cdot \cos^2\theta_d
$$
*数学原理本质*：这是一个针对入射角和观察角的二维多项式拟合，粗糙度为 0 时掠射角变暗，粗糙度为 1 时掠射角提亮（模拟微表面的多重散射和遮挡）。

---

### 2. 高光项 (Specular Lobe)
这是基于标准 Torrance-Sparrow 微表面模型的，这也是现代 PBR 的核心微积分基础：
$$
f_{specular} = \frac{D(\mathbf{h}) \cdot F(\mathbf{v}, \mathbf{h}) \cdot G(\mathbf{l}, \mathbf{v}, \mathbf{h})}{4 \cos\theta_l \cos\theta_v}
$$

**A. 法线分布函数 NDF (D 项) - GTR2 / GGX**
Disney 提出了 GTR（Generalized Trowbridge-Reitz）分布，其参数 $\gamma=2$ 时就是大名鼎鼎的 GGX：
$$
D_{GGX} = \frac{\alpha^2}{\pi \left( (\alpha^2 - 1)\cos^2\theta_h + 1 \right)^2}
$$
其中 $\alpha = roughness^2$。采用平方是为了让粗糙度参数在视觉上的变化更加线性。

**B. 菲涅尔方程 (F 项) - Schlick 近似**
$$
F = F_0 + (1 - F_0)(1 - \cos\theta_d)^5
$$
这里的关键在于 $F_0$（基础反射率）的推导。Disney 将金属（Conductor）和绝缘体（Dielectric）结合到了同一个公式里：
$$
F_0 = \text{lerp}(0.08 \cdot specular \cdot C_{tint}, BaseColor, metallic)
$$
*注：$C_{tint}$ 是由 BaseColor 算出的颜色偏移项，0.08 对应 IOR 约等于 1.5 的非金属基底反射率 (4% $\times$ 2)。*

**C. 几何遮蔽函数 (G 项) - Smith Joint GGX**
为了保证能量守恒，高光项的积分必须满足 $\int f_s (\mathbf{n}\cdot\mathbf{l}) d\omega_l \le 1$。Disney 采用分离的 Smith 遮蔽阴影函数：
$$
G(\mathbf{l}, \mathbf{v}) = G_1(\mathbf{l}) \cdot G_1(\mathbf{v})
$$
$$
G_1(\mathbf{v}) = \frac{2 \cos\theta_v}{\cos\theta_v + \sqrt{\alpha_g^2 + (1 - \alpha_g^2)\cos^2\theta_v}}
$$
*渲染硬核细节*：为了减少解析光源（Analytical Lights）下的高光过曝，Burley 在 2012 年做了一个数学 Hack，把算 G 项的粗糙度做了重映射 $\alpha_g = \left( \frac{roughness + 1}{2} \right)^2$。这一点在后续的诸多引擎（如 UE4）中被广泛采纳。

---

### 3. 清漆项 (Clearcoat Lobe)
模拟车漆、烤漆等表面覆盖的一层各向同性的、固定 IOR 的透明薄膜。
它也是一个微表面模型，但 NDF 采用了 $\gamma=1$ 的 GTR1 分布（也叫 Berry 分布），因为它能产生比 GGX 更长的高光拖尾（Tail）：
$$
D_{clearcoat} = \frac{\alpha_c^2 - 1}{\pi \ln(\alpha_c^2)} \cdot \frac{1}{(\alpha_c^2 - 1)\cos^2\theta_h + 1}
$$
其中 $\alpha_c = \text{lerp}(0.1, 0.001, clearcoatGloss)$。
它的 F 项强制 $F_0 = 0.04$（代表 IOR=1.5 的聚氨酯漆面），G 项也是一个固定粗糙度 $\alpha_g = 0.25$ 的 Smith GGX。

---

### 4. 光泽项 (Sheen Lobe)
主要用来补偿布料（Cloth）等材质在边缘的逆向散射能量（比如天鹅绒表面的细小绒毛）。它是一个极为简单的经验公式，直接加在漫反射之上：
$$
f_{sheen} = sheen \cdot C_{sheen} \cdot (1 - \cos\theta_d)^5
$$
这里的权重完全是 Schlick 菲涅尔的掠射角部分。$C_{sheen}$ 是在白色和 BaseColor 之间进行插值的颜色。

---

### 渲染工程师视角的数学批判与修正

如果你精通微积分，把上面的方程拿去做半球积分（Hemispherical Integration），你会立刻发现 2012 版 Disney BRDF 的一个致命缺陷：**它并不严格保证能量守恒 (Energy Conservation)。**

1. **多重散射的缺失**：传统的微表面理论（Microfacet）假设光线只反弹一次（Single Scattering）。当粗糙度增加时，大量光线被微表面遮蔽（G项挡住的光），这些能量直接“丢失”了，导致材质在极其粗糙时视觉上偏暗。
2. **线性相加导致能量溢出**：Disney BRDF 将 Diffuse 和 Specular 直接相加（虽然用 metallic 做了插值，但在 Dielectric 下是 Diffuse + Specular）。高光 Fresnel 在边缘趋近于 1，此时漫反射如果不相应的减弱（即乘以 $1 - F$），整体能量就会大于 1。

为了解决这些数学上的瑕疵，后续的工业界（包括 Sony Imageworks, 甚至 Disney 自己在 2015 年提出的 BSDF 更新版）做了很多补充算法：
*   引入了 **Multiple-Scattering GGX** 来积分补偿 G 项丢失的能量（涉及到解微积分方程 $\int_{\Omega} \frac{DFG}{4(\mathbf{n}\cdot\mathbf{v})} d\omega$ 的能量损耗补偿）。
*   使用了更严格的能量守恒耦合（Energy Preserving Coupled BRDF），确保 $f_{diffuse} \times (1 - \int f_{specular} d\omega)$。

**总结**
Disney BRDF 并不是纯粹的第一性原理推导，而是一个**参数化空间映射极其优秀的数学模型**。它的伟大之处在于，用诸如 `BaseColor`, `Roughness`, `Metallic` 这样互相正交、且极具美术直觉的标量，通过数学拟合，精妙地操控了底层那些复杂的半球积分结果。

这就是它的数学全貌。你对哪一项的积分推导或者数学化简更感兴趣？我们可以深入推演一下 GGX 的雅可比行列式（Jacobian）是怎么影响半角向量分布的。
