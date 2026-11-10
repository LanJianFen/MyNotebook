

太精彩了！你现在不仅是在做物理和数学的推导，你已经把目光死死盯在了**现代游戏引擎的 Shader 源码**上！

你脑海里浮现的，或者是你在各大 PBR 教程（比如 LearnOpenGL）和 UE 源码里看到的 GGX（D项），大概率长这个样子对不对？
**$$ D_{GGX} = \frac{\alpha^2}{\pi \left( (N \cdot H)^2 (\alpha^2 - 1) + 1 \right)^2} $$**

*(注：这里的 $H$ 就是我们前面说的微表面法线 $m$，所以 $(N \cdot H)$ 就是 $\cos\theta_m$)*

你盯着这个公式看：**分母里确实只有一个 $(N \cdot H)^2$（也就是 $\cos^2\theta_m$），而且没有 $\sin$，也没有 $\tan$，更没有那个极其暴力的 $\frac{1}{\cos^4\theta_m}$！**

它们到底去哪了？是被近似去掉了吗？
**绝对没有！这中间发生了一次堪称“GPU 性能救星”的殿堂级代数化简！** 

我们这就用 3 步魔术，把你看到的“只有一个 $\cos$ 的工程版公式”，和我们刚才推导的“理论版公式”完美缝合在一起！

---

### 第一步：把 $\cos^4\theta_m$ “塞”进括号里

我们刚才推导出来的理论版公式是这样的：
$$ D_{GGX} = \frac{1}{\cos^4\theta_m} \times \frac{\alpha^2}{\pi (\alpha^2 + \tan^2\theta_m)^2} $$

注意到后面那个括号是**平方**的。如果我们想把外面的 $\frac{1}{\cos^4\theta_m}$ 塞进括号里，它进去之后就会变成 $\frac{1}{\cos^2\theta_m}$。

好，我们把它塞进去：
$$ D_{GGX} = \frac{\alpha^2}{\pi \left[ \cos^2\theta_m \times (\alpha^2 + \tan^2\theta_m) \right]^2} $$

---

### 第二步：消灭 $\tan$（三角函数大屠杀）

现在，我们把括号里面的 $\cos^2\theta_m$ 乘进去展开。
别忘了，$\tan^2\theta_m = \frac{\sin^2\theta_m}{\cos^2\theta_m}$。

当 $\cos^2\theta_m$ 遇到 $\tan^2\theta_m$ 时，分母直接被干掉了：
$$ \cos^2\theta_m \times \alpha^2 = \alpha^2 \cos^2\theta_m $$
$$ \cos^2\theta_m \times \frac{\sin^2\theta_m}{\cos^2\theta_m} = \sin^2\theta_m $$

所以，公式变成了我们在上面一局刚开始提到的样子：
$$ D_{GGX} = \frac{\alpha^2}{\pi \left( \alpha^2 \cos^2\theta_m + \sin^2\theta_m \right)^2} $$

**你看！那个恐怖的 $\frac{1}{\cos^4\theta_m}$ 并没有消失，它只是“溶解”进了括号里，和 $\tan$ 同归于尽了！**

---

### 第三步：最后的伪装（把 $\sin$ 变成 $\cos$）

到上面这一步，公式里还有 $\sin^2\theta_m$。
在 GPU 里，算三角函数（`sin`, `cos`, `tan`）是非常昂贵的指令。但是，**算两个向量的点乘 $(N \cdot H)$ 却是硬件级别的单指令（极度便宜）！**

所以，渲染工程师绝不容忍公式里出现 $\sin$。
利用小学三角恒等式：$\sin^2\theta_m = 1 - \cos^2\theta_m$，代入公式：

$$ D_{GGX} = \frac{\alpha^2}{\pi \left( \alpha^2 \cos^2\theta_m + 1 - \cos^2\theta_m \right)^2} $$

提取公因式 $\cos^2\theta_m$：
$$ D_{GGX} = \frac{\alpha^2}{\pi \left( \cos^2\theta_m (\alpha^2 - 1) + 1 \right)^2} $$

最后，把数学符号 $\cos\theta_m$ 替换成 Shader 里的点乘代码 $(N \cdot H)$：
**$$ D_{GGX} = \frac{\alpha^2}{\pi \left( (N \cdot H)^2 (\alpha^2 - 1) + 1 \right)^2} $$**

**BOOM！破案了！**

---

### 架构师的工程美学

你能提出这个问题，恰恰触碰到了“学术界”和“工业界”最美妙的一道桥梁。

*   **物理学家和数学家**（如 Torrance, Sparrow, Heitz）：他们脑子里想的是雅可比矩阵、空间扭曲、斜率积分。所以他们写出来的公式，带着 $\tan$ 和 $\frac{1}{\cos^4}$，因为这样**物理直觉最清晰**。
*   **引擎架构师**（如 Epic 的 Brian Karis，也就是把 GGX 引入 UE4 的人）：他们脑子里想的是 ALU 周期、寄存器压力、指令吞吐。所以他们拿到物理学家的公式后，一顿疯狂的代数变形，把所有的三角函数全部干碎，最后榨汁成一个**“只包含一次硬件点乘，三次基础乘加（FMA）”**的极限优化版代码。

这就是为什么你看到的引擎代码（只有一个 cos）和理论推导长得完全不一样。**它们在数学上 100% 绝对等价，但在计算机执行效率上，却差了成百上千倍！**
