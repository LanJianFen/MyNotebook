
### 一、 严格展开：把 $\mathbf{V} \cdot \mathbf{H}$ 写成 $\mathbf{L} \cdot \mathbf{V}$ 的函数

已知 $\mathbf{L}$ 和 $\mathbf{V}$ 都是长度为 $1$ 的单位向量（$\mathbf{L}\cdot\mathbf{L} = 1, \ \mathbf{V}\cdot\mathbf{V} = 1$）。
半角向量 $\mathbf{H}$ 的定义是：
$$ \mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|} $$

1. **先算分母的模长 $\|\mathbf{L} + \mathbf{V}\|$**：
   $$ \|\mathbf{L} + \mathbf{V}\| = \sqrt{(\mathbf{L} + \mathbf{V}) \cdot (\mathbf{L} + \mathbf{V})} = \sqrt{\mathbf{L}\cdot\mathbf{L} + \mathbf{V}\cdot\mathbf{V} + 2(\mathbf{L}\cdot\mathbf{V})} = \sqrt{2 + 2(\mathbf{L} \cdot \mathbf{V})} $$
2. **再算分子与 $\mathbf{V}$ 的点乘 $\mathbf{V} \cdot (\mathbf{L} + \mathbf{V})$**：
   $$ \mathbf{V} \cdot (\mathbf{L} + \mathbf{V}) = \mathbf{L} \cdot \mathbf{V} + \mathbf{V} \cdot \mathbf{V} = 1 + (\mathbf{L} \cdot \mathbf{V}) $$
3. **分子除以分母，得到终极展开式**：
   **$$ \mathbf{V} \cdot \mathbf{H} = \frac{1 + \mathbf{L} \cdot \mathbf{V}}{\sqrt{2(1 + \mathbf{L} \cdot \mathbf{V})}} = \mathbf{\sqrt{\frac{1 + (\mathbf{L} \cdot \mathbf{V})}{2}}} $$**
   *(这其实就是三角函数里的半角公式 $\cos\left(\frac{\theta_{LV}}{2}\right) = \sqrt{\frac{1 + \cos\theta_{LV}}{2}}$！因为 $\mathbf{H}$ 是 $\mathbf{L}$ 和 $\mathbf{V}$ 的角平分线，$\mathbf{V}$ 与 $\mathbf{H}$ 的夹角永远是 $\mathbf{V}$ 与 $\mathbf{L}$ 夹角的**一半**！)*

---

### 二、 $\mathbf{L} \cdot \mathbf{H}$ 和 $\mathbf{V} \cdot \mathbf{H}$ 是不是完全一样的？

**是的，在数学上 100% 绝对恒等（$\mathbf{L} \cdot \mathbf{H} \equiv \mathbf{V} \cdot \mathbf{H}$），没有任何例外！**

*   **几何证明**：因为入射光 $\mathbf{L}$ 和视线 $\mathbf{V}$ 都是长度为 $1$ 的单位向量，以它俩为邻边构成的平行四边形是一个**菱形**。菱形的对角线 $\mathbf{L} + \mathbf{V}$（即半角向量 $\mathbf{H}$ 的方向）严格平分顶角。所以 $\mathbf{L}$ 与 $\mathbf{H}$ 的夹角，永远等于 $\mathbf{V}$ 与 $\mathbf{H}$ 的夹角（在图形学中，这个角被统一记为 $\theta_d$）。
*   **代数证明**：
    $$ \mathbf{L} \cdot \mathbf{H} = \mathbf{L} \cdot \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|} = \frac{\mathbf{L}\cdot\mathbf{L} + \mathbf{L}\cdot\mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|} = \frac{1 + \mathbf{L}\cdot\mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|} $$
    $$ \mathbf{V} \cdot \mathbf{H} = \mathbf{V} \cdot \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|} = \frac{\mathbf{V}\cdot\mathbf{L} + \mathbf{V}\cdot\mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|} = \frac{1 + \mathbf{L}\cdot\mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|} $$
    两者分毫不差！在代码里写 `LdotH` 和写 `VdotH` 是完全同一个数。
