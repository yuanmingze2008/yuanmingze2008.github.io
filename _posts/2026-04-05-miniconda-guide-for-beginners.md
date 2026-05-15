---
title: "零基础 Miniconda 指南"
date: 2026-04-05 20:14:28 +08:00
permalink: /posts/miniconda-guide-for-beginners/
tags:
  - python
  - conda
  - tooling
  - notes
source_note: "local source file, not uploaded"
---

# 零基础 Miniconda 指南



## 一、什么是 Miniconda？

Miniconda 可以看成一个环境管理工具和包管理器的组合。

它包含了最小化的 Python 环境和包管理工具 Conda。

它允许用户根据需要安装和管理各种 Python 包和环境，而不需要安装完整的 Anaconda “全家桶”。



## 二、为什么选择 Miniconda？

1. 轻量级：Miniconda 只包含最基本的组件，占用空间小，适合资源有限的系统。
2. 灵活性：用户可以根据项目需求安装所需的包和依赖，而不必安装不必要的软件。
3. 环境管理：Conda 提供了强大的环境管理功能，允许用户创建隔离的环境，避免包冲突。
4. 跨平台：Miniconda 支持 Windows、macOS 和 Linux 操作系统。

具体的：

> 项目 A 需要 Python 2.7 但是项目 B 需要 Python 3.13，如果来回卸载安装 Python 会非常麻烦，这时候就可以使用 Conda 创建两个独立的环境来分别运行项目 A 和项目 B。



## 三、安装 Miniconda

1. 访问 Miniconda 官方下载页面：https://docs.conda.io/en/latest/miniconda.html
2. 根据你的操作系统选择合适的安装包（Windows、macOS 或 Linux）
3. 注意安装过程中将 miniconda3 加入环境变量中（Add Miniconda3 to my PATH environment variable），使得系统可以通过环境变量直接定位 miniconda，以后在 cmd 和 PowerShell 里也能直接用 conda 命令
4. 在终端输入 conda --version 可以验证安装是否成功。、

也可以使用命令行**静默安装**：

```bash
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-Windows-x86_64.exe -o .\miniconda.exe
start /wait "" .\miniconda.exe /S
del .\miniconda.exe
```

最后一行表面安装后会自动删除引导文件。

静默安装默认不会将 Miniconda 添加到系统 PATH 中，如果需要手动添加，可以参考以下步骤：

1. 右键点击“此电脑”或“计算机”，选择“属性”。
2. 点击“高级系统设置”，然后点击“环境变量”按钮
3. 在“系统变量”部分，找到并选择“Path”变量，然后点击“编辑”
4. 点击“新建”，然后添加 Miniconda 的安装路径（例如 C:\Users\YourUsername\Miniconda3）
5. 点击“确定”保存更改



## 四、Miniconda 的配置

1. 安装过后，可使用 win 搜索找到 Anaconda PowerShell Prompt 与 Anaconda Prompt，我们使用前者，打开。

   可以认为前者是一个 PowerShell+Miniconda 的集成终端，你可以在你创建的环境中直接使用 PowerShell 的命令。

   > 可为其在桌面上创建快捷方式，方便以后使用。

2. 换源（不换源，后续的更新和创建都可能因为网络超时而失败）：

   Anaconda 的官方服务器位于美国，下载速度较慢。可以将默认源更换为国内镜像源，例如清华大学的镜像源。

      ```bash
      # 1. 添加清华源的三个核心仓库
      conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
      conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
      conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
      
      # 2. 设置搜索时显示通道地址（验证用）
      conda config --set show_channel_urls yes
      
      # 3. 设置频道优先级为严格（防止版本混用）
      conda config --set channel_priority strict
      ```

   - **原理**：修改 `.condarc`，指引 Conda 去北京的服务器（清华 TUNA）下载包，而不是去美国
   - `--add channels [url]`: 将清华的 URL 添加到频道列表的**最上方**（优先级最高）

   - `pkgs/main` & `free`: 对应官方的基础库

   - `cloud/conda-forge`: 对应社区版库（不使用官方的 `conda-forge`，而是使用清华镜像版的 `conda-forge`）

   - `show_channel_urls yes`: 下载时显示包的具体来源 URL，方便你确认是否真的走了清华源

   可采用 conda config --show channels 查看当前的频道列表，确保清华源在最上方。

3. Conda 自更新：确保管理器本身是最新版：

   （刚下载的安装包里的 Conda 版本可能是几个月前的）

   ```bash
   conda update -n base conda
   ```

   - **`-n base`**：指定操作对象是 base 环境（后续会讲）
   - **`conda`**：指定更新的对象是 conda 这个程序本身，而不是更新 Python 或其他库



## 五、管理环境

> Conda Environment Management
>
> Layer 1: Isolation
>
> - Base Environment
> - Virtual Environment
>
> Layer 2: Lifecycle
>
> - Create
> - Activate/Deactivate
> - Remove etc.
> - Infrastructure as Code
>
> Layer 3: Dependency Specification
>
> - Package Name & Version



1. **创建第一个环境（Layer 2: LifeCycle - Create）：**

   ```bash 
   conda create -n myenv python=3.13
   ```

   这会创建一个名为 `myenv` 的环境，包含 Python 3.13。

   **参数 `-n` (或 `--name`)**

   - **定义**：指定新环境的名称。
   - **底层原理**：Conda 会在你的安装目录（通常是 `miniconda3/envs/`）下创建一个以此名称命名的文件夹。
   - **适用场景**：必须参数。如果不提供 `-n` 或 `-p`（指定路径），Conda 不知道要把环境存在哪。

   **参数 `-y` (或 `--yes`)**

   - **定义**：自动确认（Automatic Yes）。
   - **底层原理**：在安装过程中，Conda 的依赖解析器（Solver）会计算出需要下载的包列表、总容量以及权限申请。默认情况下，它会停下来打印一个列表并询问 `Proceed ([y]/n)?`。
   - **参数全解**：使用 `-y` 后，Conda 会跳过交互确认环节，直接开始下载和链接文件的 IO 操作。
   - **边界情况**：在自动化脚本（如 Dockerfile 或 CI/CD）中是强制要求的。但在手动操作时，如果你担心磁盘空间不足或想确认包版本，建议**不要**加 `-y`，以便肉眼最后核对一遍。

   ```bash
   # 逻辑：创建一个名为 learn-ai 的环境，安装 Python 3.10，跳过手动确认
   conda create -n learn-ai python=3.10 -y
   ```

2. **激活环境（进入该房间）：**

   ```bash
   conda activate myenv
   ```

   - **原理：将该环境的 Python 和库路径注入到系统 PATH 中。**

   此时你的提示符会变：从 `(base)` 变为 `(torch_env)`。

   你现在所做的任何安装（pip install or conda install），都会被隔离在这个小房间里。

3. **退出环境（离开该房间）：**

   `conda deactivate` 或 `conda deavtivate -n myenv`

4. **删除环境（拆除房间）：**

   `conda remove -n myenv --all`：删除名为 `myenv` 的环境及其所有内容。

   能否删去？绝对不能。

   - 如果删去：命令变成 `conda remove -n myenv`。Conda 会以为你想在一个叫当前环境里，删除一个叫 `myenv` 的软件包。它会报错说“找不到叫 myenv 的包”。
   - 加上 `--all`：Conda 才会理解为“删除 `myenv` 这个容器本身”。

   在文件系统层面，加上 `--all` 后，Conda 会直接移除 `miniconda3/envs/myenv` 整个文件夹，并清理相关的注册信息。

5. **列出所有环境：**

   `conda env list` 或 `conda info --envs`

   - 作用：显示所有已创建的 Conda 环境及其路径。

   - 场景：忘了自己建过什么环境时用。

6. **查看当前环境信息：**

      `conda info`

   - 作用：显示当前 Conda 配置和环境信息，类似于体检报告（包含当前激活的环境、Conda 版本、操作系统平台、配置文件(`.condarc`)的位置、缓存路径、用户代理（User-Agent）等）。
   - 场景：当你遇到报错（比如网络连不上、环境冲突），去社区提问时，别人通常会让你贴出这个信息，以便诊断你的底层配置。

7. **导出环境配置：**

    `conda env export > environment.yml`

   - 作用：将当前环境的包和版本信息导出到 `environment.yml` 文件中，方便分享或备份

   - 场景：当你想把当前环境分享给同事或在另一台机器上重现时使用。

   - 详细分析：

     - `conda env export`: 输出一堆文本，包含所有包名和版本（如 `numpy=1.21.0`）

     - `>`: 操作系统的重定向符，把刚才打印在屏幕上的那些文本，抓取下来，写入到一个叫 `environment.yml` 的文件中

     - 本质上就是一个纯文本文件，你可以用记事本打开它

8. **从配置文件创建环境：**

   `conda env create -f environment.yml`

   - 作用：根据 `environment.yml` 文件创建一个新的 Conda 环境

   - `-f`: 指定 file（文件）路径

   - 你有两种方法让该命令生效：

     - 假设 `environment.yml` 在 `D:\Downloads` 目录下，先 `cd D:\Downloads` 走到那个文件夹，（使用 `ls` 或 `dir` 确认文件真的在），然后执行 `conda env create -f environment.yml`

     - 告诉 Conda 文件的绝对路径：

       ```bash
       conda env create -f D:\Downloads\environment.yml
       ```

       

# 六、工作流中

1. 环境也是终端，开启工作流直接 cd 定位工作台目录即可。

2. **安装库**：在环境中有两套安装命令（均可进行批量安装）：

   - 使用 Conda 安装：

     ```bash
     conda install numpy pandas
     ```

     - **底层原理**：它不仅安装 Python 包，还会安装非 Python 的依赖库（如 C/C++ 编译的数学库 MKL、深度学习加速库 cuDNN）。它会进行严格的依赖与版本兼容性检查（SAT Solver）。
     - **适用场景**：核心计算库（NumPy, Pandas, PyTorch, Scikit-Learn）。

   - 使用 pip 安装（在 Conda 环境中也可以使用 pip）：

     ```bash
     pip install requests flask
     ```

     - **底层原理**：它是 Python 官方的包管理器，只管 Python 代码。它通常不检查非 Python 的底层依赖冲突。
     - **适用场景**：Conda 仓库里没有的小众库，或者纯 Python 编写的工具库。

   - **原则：优先使用 conda 安装，如果不行，再使用 pip 安装。**

   - **注意**：不要混用 conda 和 pip 安装同一个包（如 numpy）。否则可能会导致版本冲突或环境损坏。

3. 清理垃圾：

   ``` bash
   conda clean --all -y
   ```

   - 作用：释放磁盘空间，删除未使用的包缓存、索引缓存和临时文件。
   - 原理：Conda 在 `pkgs/`(packages) 目录下存了压缩包和解压后的文件，这些文件会随着安装和更新操作不断累积，占用大量磁盘空间。`conda clean --all` 会删除这些不再需要的文件。





----

- 如果想要在 PS 里使用 conda：PowerShell 本身有没有初始化 Conda，在普通 PowerShell 里跑：`conda init powershell`，然后彻底关掉再重开 PowerShell。Conda 官方说明 conda init 是按 shell 分别配置，而且重启 shell 后才生效。

