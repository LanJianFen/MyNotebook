
我们刚才一直在赞美 LDLTLDLT 有多爽、多快。但是，这一切都建立在一个前提上：**定海神针（对角线上的主元 di）不能是 0！**  

甚至是，它不能是一个“极度接近 0 的数字”（比如 0.0000000010.000000001），否则一做除法，倍数瞬间爆炸，电脑的浮点数直接溢出（数值不稳定）。

如果矩阵是**正定**的，舒尔补遗传法则保证了主元永远 >0，这很完美。  
但是，如果矩阵仅仅是“实对称”，但“非正定”呢？


---

### 准备工作：设定变异模板

我们的恶魔靶子：
$$ A = \begin{bmatrix} \mathbf{0} & \mathbf{1} & 2 \\ \mathbf{1} & \mathbf{0} & 3 \\ 2 & 3 & 4 \end{bmatrix} $$

因为左上角是 $0$，单纯的 $d_1$ 撑不住了。所以我们**强行把左上角的 $2 \times 2$ 方块设定为“巨型神针”**。
这也意味着，$L$ 和 $D$ 的结构会发生变异。

**脑海里的模板长这样：**
1.  中间的 $D$：前两行前两列被整个保留，第三个还是单独的 $d_3$。
    $$ D = \begin{bmatrix} \mathbf{0} & \mathbf{1} & 0 \\ \mathbf{1} & \mathbf{0} & 0 \\ 0 & 0 & d_3 \end{bmatrix} $$
2.  下三角的 $L$：因为前两行/列被“打包”处理了，所以 $L$ 的左上角必须是一个纯净的 $2 \times 2$ 单位阵（不能再去掺和了），只有最下面那一行有未知数！
    $$ L = \begin{bmatrix} \mathbf{1} & \mathbf{0} & 0 \\ \mathbf{0} & \mathbf{1} & 0 \\ l_{31} & l_{32} & 1 \end{bmatrix} $$

好了，现在我们用 $A = L D L^T$ 来暴力破解 $l_{31}, l_{32}$ 和 $d_3$！

---

### 第一步：破解 $L$ 的未知数（联立方程，代替除法）

我们要算 $l_{31}$ 和 $l_{32}$。
在之前，我们是拿下面那个数直接除以最上面的数。现在最上面是一个 $2 \times 2$ 方块，我们用矩阵乘法的位置来对比。

我们盯着 $A$ 的第三行：$A_{31} = 2$, $A_{32} = 3$。
根据 $A = L D L^T$，算第三行的前两个数字，其实就是拿 $L$ 的第三行 $[l_{31}, l_{32}, 1]$ 去乘 $D$ 的列，再乘 $L^T$。

推导出来极其简单，它就是一个二元一次方程组：
$$ [l_{31}, \quad l_{32}] \times \begin{bmatrix} 0 & 1 \\ 1 & 0 \end{bmatrix} = [A_{31}, \quad A_{32}] $$

代入已知数字：
$$ [l_{31}, \quad l_{32}] \times \begin{bmatrix} 0 & 1 \\ 1 & 0 \end{bmatrix} = [2, \quad 3] $$

**纯手算解方程：**
*   第一个元素位置：$l_{31} \times 0 + l_{32} \times 1 = 2 \implies \mathbf{l_{32} = 2}$
*   第二个元素位置：$l_{31} \times 1 + l_{32} \times 0 = 3 \implies \mathbf{l_{31} = 3}$

你看！虽然不能做 $2 \div 0$ 的除法，但是因为那个 $2 \times 2$ 方块的作用，未知数直接被“交叉换位”解出来了！
我们成功得到了：$L = \begin{bmatrix} 1 & 0 & 0 \\ 0 & 1 & 0 \\ \mathbf{3} & \mathbf{2} & 1 \end{bmatrix}$

---

### 第二步：破解最后的 $d_3$（算残骸）

现在只剩最后一个未知数 $d_3$ 了。
我们盯住 $A$ 最右下角的数字：$A_{33} = 4$。

根据 $A = L D L^T$ 展开，算右下角元素的公式是：
$$ (\text{L 的第三行}) \times D \times (\text{L 的第三行转置}) = A_{33} $$

代入我们刚算出的数字：
$$ [3, 2, 1] \times \begin{bmatrix} 0 & 1 & 0 \\ 1 & 0 & 0 \\ 0 & 0 & d_3 \end{bmatrix} \times \begin{bmatrix} 3 \\ 2 \\ 1 \end{bmatrix} = 4 $$

**纯手算这堆乘法：**
先算前面两个：
$[3, 2, 1] \times D = [ (3\times0 + 2\times1), \quad (3\times1 + 2\times0), \quad d_3 ] = [2, 3, d_3]$

再和后面乘起来：
$[2, 3, d_3] \times \begin{bmatrix} 3 \\ 2 \\ 1 \end{bmatrix} = 2\times3 + 3\times2 + d_3\times1 = 6 + 6 + d_3 = \mathbf{12 + d_3}$

因为等式右边等于 $4$，所以：
$12 + d_3 = 4 \implies \mathbf{d_3 = -8}$

---
### 终极拼图与验算

结束了！我们只用了初中生级别的二元一次方程和加减乘除，就把这个“除以 0 会死”的矩阵分解出来了！

**最终形态：**
$$ L = \begin{bmatrix} 1 & 0 & 0 \\ 0 & 1 & 0 \\ 3 & 2 & 1 \end{bmatrix}, \quad D = \begin{bmatrix} 0 & 1 & 0 \\ 1 & 0 & 0 \\ 0 & 0 & -8 \end{bmatrix} $$

你不信的话，我们在草稿纸上反向乘一次 $L D L^T$：
$L \times D = \begin{bmatrix} 0 & 1 & 0 \\ 1 & 0 & 0 \\ 2 & 3 & -8 \end{bmatrix}$
$(L D) \times L^T = \begin{bmatrix} 0 & 1 & 0 \\ 1 & 0 & 0 \\ 2 & 3 & -8 \end{bmatrix} \begin{bmatrix} 1 & 0 & 3 \\ 0 & 1 & 2 \\ 0 & 0 & 1 \end{bmatrix} = \begin{bmatrix} 0 & 1 & 2 \\ 1 & 0 & 3 \\ 2 & 3 & 4 \end{bmatrix}$

**精精确确就是原矩阵 $A$！

---

### 准备靶子：4x4 对称恶魔矩阵

$$ A = \begin{bmatrix} \mathbf{0} & \mathbf{1} & 2 & 4 \\ \mathbf{1} & \mathbf{0} & 3 & 5 \\ 2 & 3 & 13 & 22 \\ 4 & 5 & 22 & 42 \end{bmatrix} $$
*你可以检查一下，完美对称。但左上角是 0，普通 $LDL^T$ 进来第一步就会死。*

我们要把它大卸八块，目标是抽出 $L$ 和 $D$。

---

### 第一回合：危机爆发，启动 $2\times2$ 分块消元

算法扫描左上角，发现 $A_{11} = 0$。警报拉响！
它立刻把左上角的 $2\times2$ 框起来，作为第一根**“巨型神针”**：
$$ D_1 = \begin{bmatrix} \mathbf{0} & \mathbf{1} \\ \mathbf{1} & \mathbf{0} \end{bmatrix} $$

#### 1. 算除法（算 $L$ 的左下角块）
我们要用 $D_1$ 这个整体，去消灭它下面那个 $2\times2$ 的目标块 $C = \begin{bmatrix} 2 & 3 \\ 4 & 5 \end{bmatrix}$。
怎么做除法？在矩阵世界里，除法就是**右乘逆矩阵**。
因为 $D_1^{-1} = \begin{bmatrix} 0 & 1 \\ 1 & 0 \end{bmatrix}$，所以我们算倍数（即 $L$ 的那部分）：
$$ L_{block} = C \times D_1^{-1} = \begin{bmatrix} 2 & 3 \\ 4 & 5 \end{bmatrix} \times \begin{bmatrix} 0 & 1 \\ 1 & 0 \end{bmatrix} = \begin{bmatrix} \mathbf{3} & \mathbf{2} \\ \mathbf{5} & \mathbf{4} \end{bmatrix} $$

**记账！** 赶紧把算出来的倍数写进 $L$ 的对应位置（左下角）：
$$ L = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ \mathbf{3} & \mathbf{2} & \dots & \dots \\ \mathbf{5} & \mathbf{4} & \dots & \dots \end{bmatrix} $$
*(注意看：前两行前两列的内部，我们直接认怂保留了单位阵的形态，这就是你说的“跳过不消元”)*。

#### 2. 更新右下角（算舒尔补）
现在，我们要算出消元后，右下角残余的那个 $2\times2$ 矩阵。
**口诀：原残局 - (倍数矩阵 $\times$ 被消掉的行)**
原残局 $D_{orig} = \begin{bmatrix} 13 & 22 \\ 22 & 42 \end{bmatrix}$。
被消掉的行（其实就是 $C$ 的转置） $= \begin{bmatrix} 2 & 4 \\ 3 & 5 \end{bmatrix}$。

扣减的残骸 $= \begin{bmatrix} 3 & 2 \\ 5 & 4 \end{bmatrix} \times \begin{bmatrix} 2 & 4 \\ 3 & 5 \end{bmatrix} = \begin{bmatrix} 12 & 22 \\ 22 & 40 \end{bmatrix}$。

更新舒尔补 $S$：
$$ S = \begin{bmatrix} 13 & 22 \\ 22 & 42 \end{bmatrix} - \begin{bmatrix} 12 & 22 \\ 22 & 40 \end{bmatrix} = \begin{bmatrix} \mathbf{1} & \mathbf{0} \\ \mathbf{0} & \mathbf{2} \end{bmatrix} $$

**第一回合结束！最危险的雷区被排除了，我们得到了一个极其干净的新残局！**

---

### 第二回合：安全降落，切回 $1\times1$ 普通消元

现在，算法的目光死死盯住我们刚算出来的舒尔补残局：
$$ S = \begin{bmatrix} \mathbf{1} & 0 \\ 0 & 2 \end{bmatrix} $$

算法再次扫描左上角的数字：是 $\mathbf{1}$！
警报解除！$\mathbf{1}$ 是一个非常安全、完美的数字。**算法瞬间把离合器切回普通的高斯消元法。**

*   **抄神针**：把 $1$ 抄进 $D$ 里，$d_3 = \mathbf{1}$。
*   **算 $L$**：用它下面的 $0$ 除以 $1$，$0 \div 1 = \mathbf{0}$。
*   **记账**：把 $0$ 写进 $L$ 的位置，$l_{43} = \mathbf{0}$。

更新最后一个数字：$2 - (0 \times 0 \times 1) = \mathbf{2}$。
这就是最后的 $d_4 = \mathbf{2}$。

**全剧终！矩阵被彻底拆解！**

---

### 终极拼图（检阅战利品）

来看看我们通过**“自动换挡”**得到的 $L$ 和 $D$ 到底长什么样：

中间的能量阵 $D$ 是一个**混合体（块对角阵）**：
$$ D = \begin{bmatrix} \mathbf{0} & \mathbf{1} & 0 & 0 \\ \mathbf{1} & \mathbf{0} & 0 & 0 \\ 0 & 0 & \mathbf{1} & 0 \\ 0 & 0 & 0 & \mathbf{2} \end{bmatrix} $$
*(你看它多漂亮：左上角是危急时刻被迫抱团的 $2\times2$ 方块，右下角是岁月静好的两个 $1\times1$ 独立数字)。*

而下三角阵 $L$ 记录了我们所有的开火倍数：
$$ L = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ \mathbf{3} & \mathbf{2} & 1 & 0 \\ \mathbf{5} & \mathbf{4} & \mathbf{0} & 1 \end{bmatrix} $$
*(左下角的四个数，是我们用分块矩阵除法一炮轰出来的；右下角的 0，是退化回标量除法算出来的)。*

> *你可以自己在草稿纸上速算一下 $L \times D \times L^T$，绝对一分不差地等于最开始那个恶魔靶子 $A$。*

---

### 手算的终极感悟

#### 1. 分块高斯消元法

本质就是把上面的小矩阵跳过不消元了，在矩阵代数里有一个极其威武的名字，叫做：**“分块高斯消元法 (Block Gaussian Elimination)

因为在那个 $2 \times 2$ 的小矩阵内部，对角线上是 0（或者极度接近 0 的危险数字）。
如果你强行进入它内部去搞消元，就是找死。

所以，算法做了一个绝顶聪明的妥协：
**“既然你们俩内部矛盾解不开，那我就把你们俩焊死在一个铁箱子里！对外，你们就是一个不可分割的原子（Block）！”**

这就是你说的“跳过不消元”。我们在 $L$ 矩阵的左上角放了一个 $2 \times 2$ 的单位阵 $\begin{bmatrix} 1 & 0 \\ 0 & 1 \end{bmatrix}$，意思就是：**这块区域，我认怂，我原封不动地保留。但我拿你们俩捆绑在一起的威力，去消灭下面的行！**


#### 2. Bunch 和 Kaufman 到底牛在哪里？

你可能会问：既然能切 $2 \times 2$，万一 $2 \times 2$ 也是个废物，我是不是要切 $3 \times 3$、$4 \times 4$？

**这就是这两位数学家名垂青史的原因！**
他们在 1977 年用极其复杂的数学证明了一个惊天的结论：
> **对于任何实对称矩阵，不管它长得多么扭曲、多么变态、包含了多少个 0，你永远只需要 $1 \times 1$ 和 $2 \times 2$ 这两种“子弹”，就绝对能把它完美分解！永远不需要 $3 \times 3$ 或更大的块！**

所以，最终分解出来的那个中间矩阵 $D$，就像一段摩斯密码：
$$ D = \begin{bmatrix} d_1 & & & & \\ & \mathbf{D_2} & \mathbf{D_2} & & \\ & \mathbf{D_2} & \mathbf{D_2} & & \\ & & & d_4 & \\ & & & & \dots \end{bmatrix} $$
有的是孤立的数字（$1 \times 1$ 步进留下的），有的是 $2 \times 2$ 的小方块（危险时刻跳过不消元留下的）。
