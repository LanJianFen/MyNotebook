CPU的四叉树节点里具体存什么

我们回想一下 VT 的层级结构：

- **Level 0 (根节点):** 整个世界（1 张图）。
- **Level 1:** 切成 2×2=4 块。
- **Level 2:** 每块再切成 2×2=4块（共 16 块）。

这是一个非常扎实的工程实现问题。

在 CPU 端，四叉树节点（Quadtree Node）绝不仅仅是一个“存在与否”的开关，它其实是一个 **复杂的元数据控制块（Control Block）**。

我们可以把每个节点想象成一个 **“物流订单单据”**。
为了管理海量的图块（Tiles），这个结构体必须极其精简（节省内存），但又必须包含所有必要的信息。

通常，一个工业级（如 UE5 / Frostbite）的 CPU VT 节点包含以下 **4 类核心数据**：

---

### 1. 物理显存映射 (Physical Mapping)
**核心作用：** 记录“如果我加载了，我被放在了 GPU 图集的哪个位置？”

*   **`PhysicalSlotID` (uint16 / uint32):**
    *   物理缓存（Physical Atlas）通常被划分为固定数量的槽位（Slots）。
    *   比如一张 `16k x 8k` 的物理图集，切成 `256 x 256` 的块，一共有 $2048$ 个槽位。
    *   这里只需要存一个整数 `Index` (0 ~ 2047)。
    *   通过 `Index` 可以迅速算出 UV 偏移：`float2(Index % stride, Index / stride)`。

---

### 2. 流式状态机 (Streaming State)
**核心作用：** 记录“我现在到底是个什么状态？”（异步加载必备）

*   **`State` (enum / bitfield):**
    *   **`Unloaded`:** 彻底未加载（空）。
    *   **`PendingDisk`:** 正在排队等硬盘读取（Disk I/O）。
    *   **`Reading`:** 硬盘正在读数据。
    *   **`PendingUpload`:** 读完了，内存里有数据了，正在等显卡空闲上传。
    *   **`Resident` (Active):** 已经在显存里了，随时可用。
    *   **`Evicting`:** 内存不够了，正准备被踢出去。

---

### 3. LRU 缓存标记 (LRU Timestamp)
**核心作用：** 决定“谁该死”。当显存满了，必须踢掉最近最少使用的块。

*   **`LastFrameUsed` (uint32 / uint16):**
    *   记录该节点最后一次被 Shader请求（Feedback）的帧号。
    *   比如当前是第 10000 帧。
    *   如果节点 A 的 `LastFrameUsed` 是 9999，节点 B 是 5000。
    *   **决策：** 显然踢掉 B，因为它已经 5000 帧没人看了。

---

### 4. 硬盘数据源索引 (Source Addressing)
**核心作用：** “如果我要加载，去硬盘的哪个文件的哪个位置读？”

*   **`ChunkID` / `FileOffset` (uint64):**
    *   通常 `.vt` 文件是按 Morton Order 或者线性顺序打包的。
    *   我们不需要存完整的文件路径，只需要存一个 **Chunk Index** 或者 **Byte Offset**。
    *   IO 系统拿到这个 ID，直接 `fseek` 到对应位置读取压缩数据。

---

### 5. 极简代码结构示例 (C++)

为了节省内存（因为可能有几十万个节点），我们会疯狂使用 **位域 (Bit-fields)** 压缩：

```cpp
struct VTNode 
{
    // --- 8 Bytes Total (非常紧凑) ---

    // 1. 物理槽位 ID (12位足够存 4096 个槽位)
    uint16_t PhysicalSlotID : 12; 

    // 2. 当前加载状态 (4位足够存 16 种状态)
    uint16_t State : 4;

    // 3. LRU 时间戳 (16位，可能存 "当前帧 - 上次使用帧" 的差值)
    uint16_t LastFrameDelta;

    // 4. 数据源索引 (32位，对应硬盘上的第几个 Tile)
    // 通常可以通过 Morton Code 隐式计算，不一定要存
    uint32_t SourceChunkindex; 

    // --- 树结构指针 ---
    // 为了省内存，我们通常不用指针 (Node* children[4])
    // 而是用 "数组索引" 或者 "隐式 Morton 顺序"
    // 如果必须存，可能是:
    // uint32_t FirstChildIndex; // 指向子节点在大数组里的起始位置
};
```

---

### 6. 两种流派：指针式 vs 线性化

#### A. 指针式四叉树 (Pointer-based)
*   **结构：** 经典的 `Node* children[4]`。
*   **优点：** 动态分配，灵活，适合稀疏数据。
*   **缺点：** 内存碎片化严重，指针占用 8 字节太浪费，缓存不友好（Cache Miss）。
*   **现状：** 现代高性能引擎 **基本不用**。

#### B. 线性四叉树 (Linear / Morton-based) —— **这是主流**
*   **结构：** 所有的节点存在一个巨大的 `std::vector<VTNode>` 数组里。
*   **索引：** 利用 **Morton Code (Z-order Curve)**。
    *   节点 `(Level, X, Y)` 可以通过位运算直接算出它在数组里的下标 `Index`。
    *   父子关系不需要指针，通过 `Index` 数学计算就能找到。
*   **优点：**
    *   **内存连续：** 极度利于 CPU 缓存。
    *   **省内存：** 不需要存指针。
    *   **GPU 对齐：** 和 GPU 页表的存储顺序一致，更新页表时可以直接 `memcpy` 一整块内存。

### 总结

CPU 里的 VT 节点，本质上是 **“物理显存的房产证”**。

它手里捏着：
1.  **房号** (PhysicalSlotID)
2.  **入住状态** (State)
3.  **上次打卡时间** (LRU Timestamp)
4.  **老家地址** (Disk Chunk ID)

引擎通过遍历这个数组，每一帧都在做无情的调度：**“你太久没来了，房号收回；那个新来的，你先去排队，等下一帧给你发房卡。”**
