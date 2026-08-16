

你这个问题问到了点子上！**“一切文件全堆在同一个文件夹里”**，虽然省事，但如果你的项目做大了，有几十个第三方库、几十个子模块，EXE 同级目录全是被塞满的 DLL，确实非常不优雅。

作为一个有追求的架构师，你当然可以把 DLL 藏进 `libs` 或 `modules` 这样的子文件夹里。

要做到这一点，你必须打破 Windows 操作系统默认的**“DLL 查找规则”**。

### 📜 规则科普：Windows 默认的“寻人顺序”
当你双击运行 `Project2.exe` 时，Windows 系统底层（OS Loader）会按照极其严格的顺序去找 `Project1.dll`：
1. **EXE 所在的当前目录**（这就是我们上一节用的方法）。
2. System32 目录。
3. Windows 目录。
4. 当前工作目录（Current Working Directory）。
5. **系统的环境变量 `PATH` 中包含的目录。**

除了放在同级目录，你有 **4 种机制** 可以让 EXE 跨越目录找到 DLL。根据你的使用场景，我为你从简单到硬核依次列出：

---

### 🛡️ 机制一：IDE 内部调试特供（最简单的“欺骗”法）
如果你只是不想在 Visual Studio 编译时把它们弄到同一个文件夹，但又想按 `F5` 完美运行。你可以只在 VS 里配置环境变量。

*   **做法**：
    1. 右键 `Project2` -> **属性 (Properties)**。
    2. 左侧展开 **调试 (Debugging)**。
    3. 找到 **环境 (Environment)** 这一栏。
    4. 填入：`PATH=$(SolutionDir)bin\$(Configuration)-$(Platform)\Project1;%PATH%`
*   **原理**：按 F5 启动时，VS 会临时给这个程序注入一个环境变量，告诉它：“你去 Project1 的输出目录找找 DLL”。
*   **缺点**：一旦脱离了 VS，你直接去文件夹里双击 EXE，它依然会报错找不到 DLL。

---

### 📦 机制二：写一个“启动器” (Launcher)
这是很多大型商业软件（比如早期的很多游戏或工具软件）的经典做法。把所有的核心代码全塞进一个 `bin` 文件夹，然后外面放一个很干净的 `.bat` 脚本或者一个极小的启动器 `exe`。

*   **做法（比如用批处理）**：
    新建一个 `Start.bat`：
    ```cmd
    @echo off
    set PATH=%~dp0\my_dll_folder;%PATH%
    start Project2.exe
    ```
*   **原理**：先在脚本里把包含 DLL 的路径加到临时的 `PATH` 环境变量里，然后再启动 EXE。OS Loader 查到第 5 步时，就会在这个临时 PATH 里找到 DLL。

---

### 🧙‍♂️ 机制三：代码级的魔法——“延迟加载 (Delay Load)”
这是我**最推荐的 C++ 原生优雅做法**！
如果你想把 EXE 放在根目录，而把 DLL 全扔进一个叫 `DLLs` 的子文件夹里，你可以告诉链接器：“你先别急着在启动时找 DLL，等我代码跑起来了，我告诉你去哪找！”

*   **做法**：
    1. **改配置**：右键 `Project2` -> 属性 -> **链接器 (Linker)** -> **输入 (Input)**。在 **延迟加载的 DLL (Delay Loaded Dlls)** 里，填入 `Project1.dll`。
    2. **写代码**：在 Project2 的 `main` 函数的最开头，加一句代码，硬性指定搜寻路径：
       ```cpp
       #include <windows.h>
       
       int main() {
           // 告诉 Windows，除了 EXE 目录，还要去这个子目录找 DLL！
           SetDllDirectoryA("DLLs"); 
           
           // 直到这行代码执行时，系统才会真正去加载 DLL
           Hazel::Print(); 
       }
       ```
*   **原理**：由于设置了“延迟加载”，双击 EXE 时系统根本不找 `Project1.dll`。直到代码运行到了 `SetDllDirectoryA`，你强行把 `DLLs` 文件夹塞进了搜索路径的第 1.5 顺位。接着执行 `Print()`，系统这才会去加载，并且完美在 `DLLs` 文件夹里找到了它！

---

### 👑 机制四：终极杀器——“动态显式加载” (Plugin / 插件架构)
如果有一天，你的 Hazel 引擎需要支持**“插件”**——游戏运行到一半，玩家丢进文件夹一个 `Mod.dll`，引擎瞬间识别并加载它。这时候，你连 `.lib`（借书条）都不需要了！

*   **做法**：
    把 Reference（引用）彻底删掉！不需要 `.lib`！在 Project2 里直接手写代码去硬盘上生抠这个 DLL：
    ```cpp
    #include <windows.h>
    #include <stdio.h>

    // 定义一个和你要调用的函数一样格式的指针类型
    typedef void(*PrintFunc)();

    int main() {
        // 1. 运行时强行去硬盘任意路径加载 DLL
        HMODULE hDll = LoadLibraryA("C:\\Any\\Crazy\\Path\\Project1.dll");
        if (hDll != NULL) {
            // 2. 靠“火星文”名字（或者在 DLL 里用 extern "C" 导出纯净名字）把函数挖出来
            PrintFunc myPrint = (PrintFunc)GetProcAddress(hDll, "?Print@Hazel@@YAXXZ");
            if (myPrint) {
                myPrint(); // 3. 执行！
            }
            FreeLibrary(hDll); // 4. 卸载 DLL
        }
    }
    ```
*   **原理**：这是彻头彻尾的“动态链接”。EXE 启动时完全不需要这个 DLL 的存在。代码跑到哪里，就去哪个具体的绝对/相对路径强行把 DLL 拽进内存运行。这就是现代游戏引擎的模块加载、脚本热更、Mod 系统的底层核心机制。

### 💡 给你的架构建议
对于 `Hazel` 这种引擎架构，**Core（引擎核心 DLL）和 Sandbox（游戏 EXE）在开发阶段放在同一个文件夹里其实是业内标准**，虚幻引擎 (Unreal) 也是这么做的，因为最省事、最不容易踩坑。

但如果以后你想让最终打出来的发布版本目录变好看（比如一个清爽的 `Hazel.exe` 旁边只有一个 `Bin` 文件夹），你可以毫不犹豫地使用**机制三（延迟加载 + SetDllDirectory）**！
