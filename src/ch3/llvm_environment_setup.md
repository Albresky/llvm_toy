## Linux server 配置 VSCode LLVM 编译环境

**前提：通过 apt 包管理安装了 llvm。**

### 整体思路

我们的目标是告诉 VS Code 的 C++ 编译器（如 g++ 或 clang++）在哪里找到 LLVM 的头文件 (`.h`) 和库文件 (`.so` 或 `.a`)。我们将通过以下步骤实现：

1.  在你的服务器上，使用 `llvm-config` 工具获取编译和链接所需的所有参数。
2.  在 VS Code 中，配置两个核心文件：
      * `c_cpp_properties.json`：用于 **IntelliSense**，也就是代码补全、定义跳转和错误提示。
      * `tasks.json`：用于**定义编译任务**，也就是你按下 `Ctrl+Shift+B` 时真正执行的编译命令。

### 准备工作

请确保已经在 VS Code 中安装了官方的 **C/C++ 扩展** (来自 Microsoft)。如果你是通过 SSH 连接到服务器进行开发的，请确保这个扩展已经安装在 "SSH: [你的服务器名]" 上。

-----

### 第 1 步：在服务器上找到并使用 `llvm-config`

首先，通过 SSH 登录到你的服务器终端。

#### 1.1 找到 `llvm-config`

`apt` 安装的 `llvm-config` 通常会带有版本号。你可以通过以下命令找到它：

```bash
# 尝试直接执行
llvm-config --version

# 如果上面找不到，很可能它带有版本号，比如 14, 15, 16...
# 用 which 和通配符来找
which llvm-config-*
```

这个命令可能会输出类似 `/usr/bin/llvm-config-15` 的路径。**记下这个名字**，比如 `llvm-config-15`。在后续的所有命令中，都使用你找到的这个带有版本号的名字。为了方便，我们下面统一使用 `llvm-config-15` 作为例子。

#### 1.2 获取编译和链接参数

`llvm-config-15` 可以为我们生成所有需要的参数。在终端里尝试运行以下命令，看看它们的输出：

  * **获取 C++ 编译器参数 (主要是头文件路径)**:

    ```bash
    llvm-config-15 --cxxflags
    ```

    输出可能类似：`-I/usr/lib/llvm-15/include -D_GNU_SOURCE -D__STDC_CONSTANT_MACROS ...`

  * **获取链接器参数 (库文件路径和要链接的库)**:
    教程第三章及以后需要多个 LLVM 组件。一个比较全的组件列表如下（基本能满足整个教程的需求）：

    ```bash
    llvm-config-15 --libs core executionengine mcjit interpreter analysis native support transformutils
    ```

    输出可能类似：`-L/usr/lib/llvm-15/lib -lLLVM-15` （实际输出会长得多，包含很多 `-lLLVM...`）

  * **获取链接时需要的所有参数（推荐）**
    一个更简单的方法是使用 `--ldflags` 选项，它包含了库路径。

    ```bash
    llvm-config-15 --ldflags
    ```

#### 1.3 组合成一个完整的测试命令

现在，我们可以把这些参数组合起来，尝试手动编译一下你的代码（比如叫 `main.cpp`）。这可以验证我们的参数是否正确。

```bash
# `...` 是 shell 的命令替换语法，它会执行括号内的命令并把输出结果插入到当前位置
g++ main.cpp `llvm-config-15 --cxxflags --ldflags --libs core executionengine mcjit support` -o myapp
```

或者使用 `clang++` (推荐，因为和 LLVM 更搭):

```bash
clang++ main.cpp `llvm-config-15 --cxxflags --ldflags --libs core executionengine mcjit support` -o myapp
```

如果这个命令能够成功执行并生成 `myapp` 可执行文件，那么恭喜你，你已经获得了所有需要的信息！接下来就是把这个命令搬进 VS Code。

-----

### 第 2 步：配置 VS Code

在你的项目文件夹中，创建一个 `.vscode` 文件夹（如果不存在的话）。我们将在里面创建 `c_cpp_properties.json` 和 `tasks.json`。

#### 2.1 配置 IntelliSense (`c_cpp_properties.json`)

1.  在 VS Code 中，按 `Ctrl+Shift+P` 打开命令面板，输入 `C/C++: Edit Configurations (JSON)` 并回车。
2.  VS Code 会自动生成一个 `c_cpp_properties.json` 文件。
3.  你需要修改 `includePath` 字段，把 LLVM 的头文件路径加进去。

首先，在你的服务器终端上运行：

```bash
llvm-config-15 --includedir
```

它会输出一个路径，比如 `/usr/lib/llvm-15/include`。

然后，把这个路径添加到 `c_cpp_properties.json` 的 `includePath` 中，确保路径后面加上 `/**` 以递归包含所有子目录。

下面是一个配置示例：

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "/usr/lib/llvm-15/include/**" // <--- 把你上面命令得到的路径粘贴到这里
            ],
            "defines": [],
            "compilerPath": "/usr/bin/clang", // 或者 /usr/bin/gcc
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "linux-clang-x64"
        }
    ],
    "version": 4
}
```

**保存这个文件。** 现在 VS Code 应该能正确识别 `#include <llvm/...>` 这样的头文件，代码补全和跳转功能也应该正常了。

#### 2.2 配置编译任务 (`tasks.json`)

这是最关键的一步，它定义了如何编译你的项目。

1.  在 VS Code 中，按 `Ctrl+Shift+P` 打开命令面板，输入 `Tasks: Configure Default Build Task` 并回车。
2.  选择 `C/C++: clang++ build active file` 或 `g++ build active file`。
3.  VS Code 会生成一个 `tasks.json` 文件。你需要修改它的 `args` 部分。

将 `args` 替换为我们之前在终端测试成功的编译参数。

这是一个完整的 `tasks.json` 示例，你可以直接复制粘贴并根据你的情况修改：

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "shell",
            "label": "C/C++: clang++ build active file with LLVM",
            "command": "/usr/bin/clang++",
            "args": [
                // C++ 标准和调试信息
                "-std=c++17",
                "-g",
                // 要编译的当前文件
                "${file}",
                // 输出文件名
                "-o",
                "${fileDirname}/${fileBasenameNoExtension}",
                
                // --- 以下是 LLVM 的关键参数 ---
                // 使用 shell 命令替换来自动获取所有 LLVM 参数
                // 注意：这种写法需要你的 shell 支持 `...` 或 $(...)
                "`llvm-config-15 --cxxflags --ldflags --libs core executionengine mcjit native support`",
                // 链接时可能需要的额外库
                "-lpthread",
                "-ldl",
                "-lm"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

**关键点解释**：

  * `"label"`: 任务的名字，可以随便起。
  * `"command"`: 使用的编译器，推荐 `clang++`。
  * `"args"`:
      * `${file}`: VS Code 变量，代表你当前打开并激活的文件。
      * `${fileDirname}/${fileBasenameNoExtension}`: 指定输出文件，比如你正在编辑 `main.cpp`，输出就是 `main`。
      * **`"`llvm-config-15 ...`"`**: 这是最核心的部分。我们直接把之前测试成功的 `llvm-config` 命令作为一个参数放进去。`shell` 类型的任务会先解析这个命令，把它展开成所有需要的 `-I`, `-L`, `-l` 参数，然后再传递给 `clang++`。
  * `"group"`: `"isDefault": true` 意味着你按下 `Ctrl+Shift+B` 就会默认执行这个任务。

-----

### 第 3 步：编译和运行

1.  打开你的 `toy.cpp` 或者教程第三章的 C++ 代码文件。
2.  按下 `Ctrl+Shift+B` (或者 `Cmd+Shift+B` on Mac)。
3.  VS Code 的终端面板会显示编译过程。如果没有错误，你会在当前文件目录下看到一个同名的可执行文件。
4.  在 VS Code 的终端里，直接运行它，例如：`./toy`。
