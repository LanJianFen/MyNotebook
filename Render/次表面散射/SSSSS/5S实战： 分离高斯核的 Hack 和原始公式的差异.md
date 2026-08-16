

#### 1. 原始的 12 Pass（物理正确的 6 层高斯分离）
每一次高斯模糊，都是先横向糊一遍，再纵向把横向的结果糊一遍。然后把这 6 层的结果**加**起来：
$$ M_{True} = \sum_{k=1}^6 w_k \cdot \underbrace{\sum_y G_{1D}(y, \sigma_k) \Big( \sum_x G_{1D}(x, \sigma_k) \cdot E_{in} \Big)}_{\text{先 X 后 Y 的可分离卷积}} $$

#### 2. 本代码作者的 2 Pass（暴力合并的 1D 近似）
作者在 C# 预计算里，定义了一个**无视物理规律的“混合 1D 数组”**（也就是 `kernel` 数组）：
$$ \mathbf{K}_{1D}(t) = \sum_{k=1}^6 w_k \cdot G_{1D}(t, \sigma_k) $$

然后，他在 GPU 里跑的 Shader 公式，是把这个综合的 $\mathbf{K}_{1D}$ 当作一个普通的滤波器，**先跑 X（求出临时变量），再跑 Y（对临时变量求积分）**：
$$ M_{Author} = \sum_y \mathbf{K}_{1D}(y) \Big( \sum_x \mathbf{K}_{1D}(x) \cdot E_{in} \Big) $$

如果我们把 $\mathbf{K}_{1D}$ 展开塞回去，作者真正在算的公式是：
$$ M_{Author} = \sum_y \left[ \sum_{k=1}^6 w_k G_{1D}(y, \sigma_k) \right] \times \Big( \sum_x \left[ \sum_{k=1}^6 w_k G_{1D}(x, \sigma_k) \right] \cdot E_{in} \Big) $$

**你可以仔细对比一下 $M_{True}$ 和 $M_{Author}$ 这两个公式！**
这就是我们上一层楼里讨论的：**作者强行把求和符号 $\sum_k$ 从乘法里面“提”到了外面！**

---

### 1. 案发现场：违背结合律的“强行提取”

你写出的推导过程极其致命。
真正的 12 Pass（物理正确）的混合 2D 卷积，也就是 6 层高斯的叠加，它的数学展开是这样的：
$$ M_{True} = \sum_{k=1}^6 \Big[ w_k \cdot \big( G_{1D\_x}(\sigma_k) \otimes G_{1D\_y}(\sigma_k) \big) \Big] $$

但是，作者在 C# 里把一维的高斯先加起来了，然后分别在 X-Pass 和 Y-Pass 执行。他真正在 GPU 里跑的公式，其实是：
$$ M_{Jimenez} = \Big( \sum_{k=1}^6 w_k \cdot G_{1D\_x}(\sigma_k) \Big) \otimes \Big( \sum_{k=1}^6 w_k \cdot G_{1D\_y}(\sigma_k) \Big) $$

**数学稍微好点的人一眼就能看出来，这两者绝对不相等！**
$\sum (A_k \cdot B_k) \neq (\sum A_k) \cdot (\sum B_k)$

为了让你看清后果，我们假设只有两层高斯，把它展开（多项式乘法）：
$$ (A_1 + A_2) \otimes (B_1 + B_2) = A_1 B_1 + A_2 B_2 + \underbrace{A_1 B_2 + A_2 B_1}_{\text{罪魁祸首！交叉项！}} $$

真正的次表面散射只需要 $A_1 B_1$（第一层高斯的完美圆）和 $A_2 B_2$（第二层高斯的完美圆）。
但是作者的公式里，多出了可怕的 **交叉项（Cross Terms）**！比如 $A_1 B_2$ 意味着：**在 X 方向上只糊了 1 毫米，但在 Y 方向上却糊了 10 毫米！**

### 2. 这个错误在视觉上是什么？（著名的十字伪影）

如果你用作者的这个算法，去渲染屏幕上的一个极亮的发光白点（Impulse Response），你会发现原本应该散开成一个**完美的红色圆形光晕（Disk）**，竟然变成了一个**沿着 X 轴和 Y 轴拉伸的红色十字星（Cross / Star Artifact）**！

因为那些“交叉项”破坏了各向同性（Isotropy），导致 X 轴和 Y 轴上的能量分布和其他对角线方向不一样。