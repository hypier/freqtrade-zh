# Windows 安装

我们**强烈建议**Windows用户使用 [Docker](docker_quickstart.md)，因为这样会更加易用且流畅（也更安全）。

如果不方便使用Docker，建议尝试使用Windows子系统Linux（WSL）——对应的Ubuntu安装说明应该也适用。  
否则，请按照下面的指引操作。

所有说明均假设已安装并可用Python 3.10+。

## 克隆git仓库

首先，通过运行以下命令克隆仓库：

```powershell
git clone https://github.com/freqtrade/freqtrade.git
```

接下来，选择你的安装方式，可以通过脚本自动安装（推荐）或手动按照对应的指南操作。

## 自动安装freqtrade

### 运行安装脚本

该脚本会询问你几个问题，以确定需要安装哪些部分。

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass
cd freqtrade
. .\setup.ps1
```

## 手动安装freqtrade

!!! Note "64位Python版本"  
    请确保使用64位Windows和64位Python，以避免因Windows下32位应用程序的内存限制导致回测或超参数优化时出现问题。  
    Windows不再支持32位Python版本。

!!! Hint  
    在Windows上使用[Anaconda发行版](https://www.anaconda.com/distribution/)可以大大简化安装过程。更多信息请参考文档中的[Anaconda安装部分](installation.md#installation-with-conda)。

### 安装ta-lib

根据[ta-lib官方文档](https://github.com/TA-Lib/ta-lib-python#windows)安装ta-lib。

由于在Windows上从源代码编译依赖繁重（需要部分Visual Studio安装），Freqtrade提供了这些依赖的二进制轮子（Wheel格式），适用于最新的3个Python版本（3.10、3.11和3.12）以及64位Windows系统。这些Wheel文件也由在Windows上运行的持续集成（CI）测试，与Freqtrade一同测试通过。

其他版本请从上述链接下载。

```powershell
cd \path\freqtrade
python -m venv .venv
.venv\Scripts\activate.ps1
# 可选：安装ta-lib wheel包
# 根据实际下载的Wheel文件名调整以下命令中的文件名
pip install --find-links build_helpers\ TA-Lib -U
pip install -r requirements.txt
pip install -e .
freqtrade
```

!!! Note "使用PowerShell"  
    上述安装脚本假设你在64位Windows上使用PowerShell。  
    在传统CMD（命令提示符）中的命令可能略有不同。

### 在Windows上安装时遇到错误

```bash
error: Microsoft Visual C++ 14.0 is required. Get it with "Microsoft Visual C++ Build Tools": http://landinghub.visualstudio.com/visual-cpp-build-tools
```

不幸的是，许多需要编译的包没有提供预编译的Wheel，因此必须在Python环境中安装可用的C/C++编译器。

你可以从[这里](https://visualstudio.microsoft.com/visual-cpp-build-tools/)下载Visual C++构建工具，安装“使用C++的桌面开发”（默认配置）。  
这是一份较大的下载和依赖，你也可以考虑优先使用WSL2或[docker compose](docker_quickstart.md)。  

![Windows installation](assets/windows_install.png)