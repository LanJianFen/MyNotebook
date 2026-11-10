

你太敏锐了！在前面我们主要是在“审判”将它和 Specular 强行相加导致的灾难。但如果把聚光灯**单单打在 Diffuse 本身**上，它也经历了一场极其壮烈的进化史。

2012 年 Brent Burley 捏出来的那个带有 $F_{D90}$ 和 Schlick 5 次方的 **Disney Diffuse**，虽然极其惊艳地模拟了粗糙表面的“边缘提亮（Retro-reflection）”，但它本质上是一个**经验公式（Empirical Model）**。

在随后的 10 年里，随着离线渲染和实时渲染对物理真实的极致追求，这个经验公式被工业界“动了三次大手术”，并且每一次都成为了当今 3A 引擎的主流标准！

---

### 第一场手术：Disney 的自我否定与修正（2015 BSDF）

**【痛点】**
当年把 Disney Diffuse 拿回皮克斯做电影时，艺术家们很快发现了一个大 Bug：**它不能处理次表面散射（Subsurface Scattering, SSS）！**
真实的非金属（如皮肤、大理石、甚至树叶），光线钻进去之后，不是在极浅的表面瞬间弹出来，而是在内部“跑了一段距离”再出来。2012 年的 Diffuse 完全没考虑这种深层穿透。

**【改进：Disney 2015 修正版】**
解铃还须系铃人。2015 年，Brent Burley 自己发表了新论文，对 2012 年的模型进行了大规模升级（史称 **Disney BSDF 2015**）。
他引入了一个 `Subsurface` 参数，并重写了 Diffuse 的内部插值：
*   他把漫反射分成了两部分：极其微小的表面散射（Base Diffuse）和深层的次表面散射（Subsurface）。
*   **数学改进：** 他修正了原本边缘高光过曝的问题，让整体能量积分更加平滑。

**【行业现状】**
这个 2015 修正版直接成为了后来离线渲染器（如 Blender 的 Principled BSDF 第一版、Arnold 早期版本）的基石。

---

### 第二场手术：极致的能量守恒 —— 归一化（Renormalized Diffuse）

**【痛点】**
到了实时渲染界（特别是 UE 和 Frostbite 引擎），架构师们拿着 2012 和 2015 的 Disney Diffuse 去做**白炉测试**，结果全被骂了回来。
为什么？因为无论是哪一版，Disney Diffuse 在半球上的积分**根本不等于 1，而且随粗糙度剧烈波动！**
如果你给它强加粗糙度，它的 Directional Albedo（方向反射率）可能会超出 1.0 很多，直接凭空造出光来！

**【改进：Frostbite 与 Filament 的归一化魔法】**
既然你积分不等于 1，那我们就强行用微积分把你压扁！
著名引擎 **Frostbite (寒霜)** 的架构师 Sébastien Lagarde，以及 Google **Filament** 的架构师 Romain Guy，在底层强制引入了**归一化系数**。

他们通过预计算，算出了 Disney Diffuse 在不同粗糙度和视角下的能量溢出值，然后直接除以这个值。
在现代引擎的代码里，如果你开启了高质量的 Diffuse，你看到的不再是原始的 Disney 公式，而是带有能量守恒修正的缩放版：
**$$ f_{renormalized\_diffuse} = f_{disney} \times EnergyCompensation(Roughness, N \cdot V) $$**

**【意想不到的结局：Lambert 的文艺复兴】**
更好玩的是，Filament 引擎在白皮书中写下了一个暴论：**“经过极其严格的能量守恒和 GGX 透射修正后，我们发现，用最古老的 Lambert 加上物理遮蔽，和花里胡哨的 Disney Diffuse 视觉差异微乎其微！”**
所以，为了极致的性能，许多现代移动端和部分 3A 管线，干脆抛弃了 Disney Diffuse，回归了 **“物理正确的 Lambert”**（即 $Lambert \times (1 - E_{spec})$），这被证明是性价比最高的主流方案。

---

### 第三场手术：终极真理 —— 耦合微表面漫反射（Coupled Microfacet Diffuse）

**【痛点：最大的物理 Bug】**
如果我们要追求绝对的物理真实，Disney Diffuse 有一个底层的逻辑错误：
**光线是穿过 GGX 微表面（山脉）进入材质的，出来的时候，它还得再穿过一次同一片 GGX 山脉！**
但是 Disney Diffuse 在计算时，完全无视了外层的 GGX 形状，自己玩自己的一套插值。这就好比你在一个坑洼不平的玻璃球里倒满了牛奶，牛奶发出的光却假装玻璃球是绝对光滑的！

**【改进：Turquin (2018) 与现代工业界】**
2018 年，Autodesk 的 Emmanuel Turquin 发表了论文，彻底把 Diffuse 纳入了 GGX 的统辖之下。
他推导了光线在 GGX 表面发生的两次折射（进、出）的完整积分：
1. 光线 $L$ 砸在微表面上，没有被高光 $F$ 弹走，而是折射进了表面：折射率是 $1 - F_{in}$。
2. 光线在内部均匀散射。
3. 光线准备从 $V$ 方向射出，但又得经过一次微表面的菲涅尔折射：折射率是 $1 - F_{out}$。

但是，微表面是有遮挡的！所以这绝对不是简单的相乘。Turquin 用纯数学解出了这个**“GGX 透射耦合漫反射”**模型（基于多重散射）。

它的终极形态（经过简化）长这样：
**$$ f_{coupled\_diffuse} = \frac{BaseColor}{\pi} \times \frac{(1 - E(L)) \cdot (1 - E(V))}{1 - E_{avg}} $$**
*注意看：这里面再也没有什么 $F_{D90}$、也没有什么 5 次方插值了。完全用高光的能量积分 $E$ 来反向控制 Diffuse！*

**【行业现状】**
**这是当今图形学界毫无争议的最高标准（Ground Truth）。**
*   **Epic Games** 在 **UE5 的 Substrate 材质系统**中，底层的 Diffuse 已经全面切换到了这种基于微表面折射耦合的物理漫反射模型。
*   **Blender 4.0** 最近刚刚完全重写了 Principled BSDF，它的 Diffuse 也已经抛弃了原本的 Disney 模型，转而使用了基于这个理论的改进版（如 Multiscattering Lambert / Oren-Nayar 耦合）。
*   **企业级 PBR (Enterprise PBR)** 和各大离线渲染器，都已经将这个公式作为标配。

---

### 架构师的总结

从 2012 年到今天，Diffuse 的进化史是一场**从“感性艺术”走向“理性物理”**的硬核革命。

*   **2012 年 (Disney 崛起)**：美术说边缘太暗了，程序员用经验插值硬生生造出了 Retro-reflection 提亮。
*   **2015 - 2017 年 (工程修补)**：渲染工程师发现能量爆炸，于是强行引入能量守恒除法，将其“压制”在白炉测试的及格线内。
*   **2018 年至今 (物理大一统)**：数学家接管战场，彻底废除经验公式，用**微表面透射率（Transmission）**和**能量互补（Energy Compensation）**，让 Diffuse 和 Specular 共享同一个 GGX 山脉。

如果你现在去写一个现代的 Renderer，如果你对性能要求高，你会用带有高光遮蔽的 Lambert；如果你追求次世代极限画质，你会毫不犹豫地写下 **Coupled Microfacet Diffuse**。而那个曾经开启大航海时代的 2012 版 Disney Diffuse 公式，如今已经功成身退，成为了教科书里伟大的历史遗迹了！
