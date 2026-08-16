
### 一. 从均匀采样点到非均匀采样点
```csharp
// Calculate the SSS_Offset_UV:  
     float step = 2.0f * RANGE / (nSamples - 1);  
     for (int i = 0; i < nSamples; i++)  
     {  
         float o = -RANGE + i * step;  
         float sign = o < 0.0f ? -1.0f : 1.0f;  
         float w = RANGE * sign * Mathf.Abs(Mathf.Pow(o, EXPONENT)) / Mathf.Pow(RANGE, EXPONENT);//通过Power将更多的分段集中在中间  
         kernel[i + startIndex] = new Vector4(0, 0, 0, w);  
     }
```
### 1. 生成线性距离
```csharp
float step = 2.0f * RANGE / (nSamples - 1);  
// 循环中：
float o = -RANGE + i * step;  
```
*   `RANGE = 3.0f` 代表我们最远要在屏幕上往左/往右采 3 个单位距离。
*   这就像一把直尺。假设我们采 7 个点，那算出来的 `o`（线性偏移量）就是：
    **`-3.0,  -2.0,  -1.0,  0.0,  1.0,  2.0,  3.0`**
*   这叫**均匀分段**。但问题是，高斯曲线在中心 `0` 处是一个极其尖锐的山峰，在 `3` 处是极其平缓的山谷。如果我们均匀撒点，中心那座“尖峰”的细节就会被严重漏掉！

### 2. 非线性空间扭曲（核心公式）
```csharp
float sign = o < 0.0f ? -1.0f : 1.0f;  
float w = RANGE * sign * Mathf.Abs(Mathf.Pow(o, EXPONENT)) / Mathf.Pow(RANGE, EXPONENT);
```
这行代码看起来长，其实它在数学上做了一套完美的“归一化 $\to$ 弯曲 $\to$ 还原”操作。已知 `EXPONENT = 2.0f`：

1.  **弯曲（Pow）**：`Mathf.Pow(o, EXPONENT)` 也就是 $o^2$。它把数字给“扭曲”了。
2.  **归一化（除法）**：`/ Mathf.Pow(RANGE, EXPONENT)` 也就是除以 $3^2 = 9$。这一步把上一步扭曲后的数值，重新映射回 $0.0 \sim 1.0$ 之间。
3.  **还原缩放和方向（乘法）**：`RANGE * sign`，把它重新拉伸回 $0 \sim 3$ 的长度，并赋予它原来的正负号（向左还是向右）。

**见证奇迹的时刻！**
经过这行代码扭曲后，原本均匀的 **`-3, -2, -1, 0, 1, 2, 3`** 变成了（大致数值）：
**`-3.0,  -1.33,  -0.33,  0.0,  0.33,  1.33,  3.0`**

你看出了什么？
*   在边缘（两端）：跨度极大（从 1.33 一下子跨到了 3.0）。
*   在中心（0附近）：**点被极其密集地挤在了一起（-0.33, 0, 0.33）！**

### 3. 打包数据
```csharp
kernel[i + startIndex] = new Vector4(0, 0, 0, w);
```
最后，作者把算好的偏移距离 `w`，装进了 `Vector4` 的第 4 个通道（`.w`）。
前面的 `x, y, z` 为什么是 `0`？因为这三个通道是用来装“红绿蓝颜色权重”的，这正是接下来的代码要算的事情。

---
### 二. 黎曼积分求面积（计算每个采样点的权重）
```csharp
// Calculate the SSS_Scale:  
     for (int i = 0; i < nSamples; i++)  
     {  
         float w0 = i > 0 ? Mathf.Abs(kernel[i + startIndex].w - kernel[i - 1 + startIndex].w) : 0.0f;  
         float w1 = i < nSamples - 1 ? Mathf.Abs(kernel[i + startIndex].w - kernel[i + 1 + startIndex].w) : 0.0f;  
         float area = (w0 + w1) / 2.0f;  
         Vector3 temp = profile(kernel[i + startIndex].w, falloff);  
         Vector4 tt = new Vector4(area * temp.x, area * temp.y, area * temp.z, kernel[i + startIndex].w);  
         kernel[i + startIndex] = tt;  
     }
```
### 1. 寻找相邻兄弟的距离 (`w0` 和 `w1`)
```csharp
float w0 = i > 0 ? Mathf.Abs(kernel[i].w - kernel[i - 1].w) : 0.0f;  
float w1 = i < nSamples - 1 ? Mathf.Abs(kernel[i].w - kernel[i + 1].w) : 0.0f;  
```
*   上一节我们知道，`kernel[i].w` 存的是当前采样点到中心的距离（偏移量）。
*   `w0` 算的是：当前点，离**左边那个点**有多远。
*   `w1` 算的是：当前点，离**右边那个点**有多远。

### 2. 计算代表的“领土宽度” (`area`)
```csharp
float area = (w0 + w1) / 2.0f;  
```
*   这一步算出了当前这个点，在整条坐标轴上“管辖”多宽的区域。
*   **为什么宽度会不一样？** 还记得上一节的结论吗？我们在中心挤了很多点，在边缘只放了几个点。
    *   **在中心**：点和点挨得极近，算出来的 `area` 可能只有 `0.1`。
    *   **在边缘**：点和点离得极远，算出来的 `area` 可能高达 `1.5`。

### 3. 读取高斯曲线的“高度” (`temp`)
```csharp
Vector3 temp = profile(kernel[i + startIndex].w, falloff);  
```
*   把距离 `w` 塞进之前那个由 5 个高斯函数组成的 `profile` 函数里。
*   它返回一个 `Vector3`，分别代表在这段距离下，红绿蓝三根高斯曲线的**函数值（也就是曲线在这点的高度）**。

### 4. 终极积分公式：权重 = 高度 × 宽度
```csharp
Vector4 tt = new Vector4(area * temp.x, area * temp.y, area * temp.z, kernel[i].w);  
kernel[i + startIndex] = tt;  
```
*   作者把函数高度（`temp`）乘上了代表宽度（`area`）！
*   实际的公式是  $∫ \left[ \sum_{k=1}^6 w_k G_{1D}(x, \sigma_k) \right] \cdot E_{in} dx$ , 这里算的是 $G_{1D}(x, \sigma_k) dx$ , 而 $w_k$在分离 (1) 里用常量算好了
* $\int E_{in} \cdot G(x) dx \approx \sum \Big( E_{in}(x_i) \cdot \underbrace{[G(x_i) \cdot \Delta x_i]}_{\text{这就是 这一步 算出来的值}} \Big)，G(x_i) = \sum_{k=1}^6 w_k G_{1D}(x_i, \sigma_k)$

---

### 三.权重归一化
```csharp
Vector4 t = kernel[nSamples / 2 + startIndex];  
     for (int i = nSamples / 2; i > 0; i--)  
         kernel[i + startIndex] = kernel[i + startIndex - 1];  
     kernel[0 + startIndex] = t;  
     Vector4 sum = Vector4.zero;  
  
     for (int i = 0; i < nSamples; i++)  
     {  
         sum.x += kernel[i + startIndex].x;  
         sum.y += kernel[i + startIndex].y;  
         sum.z += kernel[i + startIndex].z;  
     }  
  
     for (int i = 0; i < nSamples; i++)  
     {  
         Vector4 vecx = kernel[i + startIndex];  
         vecx.x /= sum.x;  
         vecx.y /= sum.y;  
         vecx.z /= sum.z;  
         kernel[i + startIndex] = vecx;  
     }
```
### 1.数组重排（中心点提取术）
看这前三行代码：
```csharp
Vector4 t = kernel[nSamples / 2 + startIndex];  
for (int i = nSamples / 2; i > 0; i--)  
    kernel[i + startIndex] = kernel[i + startIndex - 1];  
kernel[0 + startIndex] = t;  
```

**它在干什么？**
假设我们采 7 个点，原本数组里的偏移量从小到大排是这样的：
`[-3, -2, -1, 0, 1, 2, 3]`

代码里 `nSamples / 2` 刚好指向最中间的那个 `0`（中心点）。
它把这个 `0` 抽出来，把前面的元素往后挤，然后把 `0` 塞到了数组的**第 0 号位置**！
排完序后，数组变成了这样：
`[0, -3, -2, -1, 1, 2, 3]`

**为什么要这么干？**
这就必须联动 `SSSSS_Core` 的Shader 代码了！
如果在 Shader 要获得 中心点的权重和 irradiance，就得多写一个 `if (offset == 0)` 的判断（因为中心像素本身不需要偏移，甚至可能不需要算深度防溢出）。而在 GPU 的 Shader 里写 `if` 分支是非常伤性能的！

所以，作者在 C# 端提前把中心点移到了 `kernel[0]`，在 Shader 里就可以直接获取

### 2.暴力归一化（能量守恒的最后一道防线）

看后面的求和与除法：
```csharp
// 1. 把所有 RGB 的权重各自加起来
for (int i = 0; i < nSamples; i++) {
    sum.x += kernel[i].x;  
    sum.y += kernel[i].y; ...
}

// 2. 除以总和
for (int i = 0; i < nSamples; i++) {
    kernel[i].x /= sum.x; ...
}
```

**为什么必须这么做？**
不论你前面的微积分和高斯公式写得多完美，由于我们只取了 13 或者 17 个离散的点，你把这十几个点的梯形面积加起来，绝对不可能刚好等于 `1.0`。
*   如果加起来是 `1.2`，那画面模糊完之后，皮肤就会**发光（过曝）**。
*   如果加起来是 `0.8`，那画面模糊完之后，皮肤就会**发黑（能量丢失）**。

---

### 四.补全第六个高斯核
```csharp
Vector4 vec = kernel[0 + startIndex];  
     vec.x = (1.0f - strength.x) * 1.0f + strength.x * vec.x;  
     vec.y = (1.0f - strength.y) * 1.0f + strength.y * vec.y;  
     vec.z = (1.0f - strength.z) * 1.0f + strength.z * vec.z;  
     kernel[0 + startIndex] = vec;  
  
     for (int i = 1; i < nSamples; i++)  
     {  
         var vect = kernel[i + startIndex];  
         vect.x *= strength.x;  
         vect.y *= strength.y;  
         vect.z *= strength.z;  
         kernel[i + startIndex] = vect;  
     }
```
#### 1. 发现被掩盖的真相
你看代码里那个由 5 根高斯组成的 `profile` 函数，如果你把它们前面的系数加起来：
$0.100 + 0.118 + 0.113 + 0.358 + 0.078 = 0.767$
也就是说，**这 5 根用来算散射的曲线，它们自带的总能量只有 76.7%！** 剩下的 23.3% 属于那根被注释掉的“极窄细刺”。

在之前的代码里，作者做了一件极其残暴的事：
```csharp
// 强制归一化
for (...) kernel[i] /= sum;
```
这一步操作，把原本总能量只有 76.7% 的那 5 根曲线，**强行拉伸（放大）到了 100%（1.0）！**

#### 2. Strength 的真正使命：重新切分蛋糕
如果我们不注释那个比较尖的高斯函数能量是100%。但是我们注释掉了，能量就变少了。对于中心点来说，它少掉的最多，其他采样点少掉的少一点。

大家的能量加起来只有0.767，缺少的高斯函数占了0.233的能量，它的能量占比还是比较多的。

作者计算了每个采样点分别计算了自己占这0.767的百分之几，接下来要想办法把这0.233的能量补充回去。

这0.233的能量几乎都集中在中心点，所以全部补充给中心点。对于中心点来说，normalizedWeight其实本该就是0.767，所以是0.767 * normalizedWeight + 0.233。
其他采样点，缺少的高斯函数损失不多，所以就是 0.767 * normalizedWeight。

我们要向这个系统引入 `Strength`（假设 $S = 0.767$）。

    $$ W_{surrounding} = S \times \text{NormalizedWeight} $$

*   **对于中心点：**
    中心点是一个特殊的“双重身份”兼职者。
    它不仅分到了那 76.7% 预算里属于自己的一小块散射权重，它还**独吞了所有那 23.3% 根本没跑出去的光（也就是那根刺）**！
    所以它的最终权重，是两笔账的叠加：
    $$ W_{center} = \underbrace{(1 - S)}_{\text{没跑出去的 23.3\%}} + \underbrace{S \times \text{NormalizedWeight}_{center}}_{\text{跑出去但在中心点的部分}} $$

这就是 `vec.x = (1.0f - strength.x) * 1.0f + strength.x * vec.x` 这行神仙代码的终极物理奥义！