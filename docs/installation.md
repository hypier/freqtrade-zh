# 安装

本页面介绍如何准备环境以运行交易机器人。

Freqtrade 的文档中描述了多种安装Freqtrade的方法

* [Docker镜像](docker_quickstart.md)（单独页面）
* [脚本安装](#script-installation)
* [手动安装](#manual-installation)
* [Conda环境安装](#installation-with-conda)

在开始评估Freqtrade的工作方式时，建议使用预构建的 [docker镜像](docker_quickstart.md)，以快速上手。

------

## 相关信息

Windows系统的安装请参考 [Windows安装指南](windows_installation.md)。

最简便的安装和运行Freqtrade的方法是克隆该机器人的Github仓库，然后运行 `./setup.sh` 脚本（如果你的平台支持的话）。

!!! Note "版本注意事项"
    克隆仓库时，默认工作分支名称为 `develop`。此分支包含所有最新特性（可以视为相对稳定，得益于自动化测试）。
    `stable` 分支包含最近一次发布的代码（通常每月一次，从距离最新`develop`分支大约一周的快照中创建，以避免打包时出现的Bug，所以潜在上更稳定）。

!!! Note
    假设已安装Python3.10或更高版本，以及对应的`pip`。如果未满足条件，安装脚本会发出警告并停止。还需要`git`软件以克隆Freqtrade仓库。
    此外，Python头文件（`python<你的版本>-dev` / `python<你的版本>-devel`）必须可用，才能确保安装过程顺利完成。

!!! Warning "保持时间同步"
    运行机器上的系统时间必须准确，并且要频繁同步到NTP服务器，以避免与交易所通信出现问题。

------

## 需求条件

这些需求适用于 [脚本安装](#script-installation) 和 [手动安装](#manual-installation)。

!!! Note "ARM64系统"
    如果你使用的是 ARM64 架构（如Mac M1或Oracle VM），请使用 [docker](docker_quickstart.md) 运行Freqtrade。
    虽然可以通过一些手动操作实现本地原生安装，但目前尚不支持。

### 安装指南

* [Python >= 3.10](http://docs.python-guide.org/en/latest/starting/installation/)
* [pip](https://pip.pypa.io/en/stable/installing/)
* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [virtualenv](https://virtualenv.pypa.io/en/stable/installation.html)（推荐）
* [TA-Lib](https://ta-lib.github.io/ta-lib-python/)（安装说明见下文[安装TA-Lib](#install-ta-lib)）

### 安装软件

我们已整理/收集了适用于Ubuntu、macOS和Windows的安装说明。这些只是指导方针，其他发行版可能略有差异。  
首先列出特定操作系统的步骤，以下通用部分适用于所有系统。

!!! Note
    预设Python3.10或更高版本及对应的pip已安装。

=== "Debian/Ubuntu"
    #### 安装必要依赖

    ```bash
    # 更新仓库列表
    sudo apt-get update

    # 安装所需包
    sudo apt install -y python3-pip python3-venv python3-dev python3-pandas git curl
    ```

=== "macOS"
    #### 安装必要依赖

    如果尚未安装[Homebrew](https://brew.sh/)，请先安装。

    ```bash
    # 安装包
    brew install gettext libomp
    ```
    !!! Note
        `setup.sh`脚本会帮你安装这些依赖——前提是你的系统已安装brew。

=== "Raspberry Pi/Raspbian"
    以下假设你使用的是最新版的 [Raspbian Buster lite 镜像](https://www.raspberrypi.org/downloads/raspbian/)，
    该镜像已预装Python3.11，便于快速部署Freqtrade。

    测试环境为：Raspberry Pi 3，使用Raspbian Buster lite镜像，已全部更新。

    ```bash
    sudo apt-get install python3-venv libatlas-base-dev cmake curl
    # 使用 piwheels.org 来加快安装速度
    sudo echo "[global]\nextra-index-url=https://www.piwheels.org/simple" > tee /etc/pip.conf

    git clone https://github.com/freqtrade/freqtrade.git
    cd freqtrade

    bash setup.sh -i
    ```
    !!! Note "安装时长"
        由于网络速度和Pi型号不同，安装可能耗时数小时。
        建议使用[Docker快速入门指南](docker_quickstart.md)中的预建docker镜像，提高效率。

    !!! Note
        上述步骤未安装hyperopt依赖。如需安装，请执行：
        ```bash
        python3 -m pip install -e .[hyperopt]
        ```
        不建议在Raspberry Pi上运行hyperopt，因为这是一项资源密集型操作，应在性能更强的计算机上进行。

------

## Freqtrade仓库

Freqtrade是一个开源的加密货币交易机器人，其代码托管于 `github.com`

```bash
# 下载`develop`分支
git clone https://github.com/freqtrade/freqtrade.git

# 进入下载的目录
cd freqtrade

# 你的选择（1）：新手用户
git checkout stable

# 你的选择（2）：高级用户
git checkout develop
```

(1) 此命令会将仓库切换到`stable`分支。如果你希望保持在`develop`分支，可以跳过此步骤。

之后可以随时使用 `git checkout stable` / `git checkout develop` 在分支间切换。

??? Note "从PyPI安装"
    另一种安装Freqtrade的方法是通过 [pypi](https://pypi.org/project/freqtrade/)。但缺点是需要提前正确安装好ta-lib，目前不推荐此方法。

    ```bash
    pip install freqtrade
    ```

------

## 脚本安装

Freqtrade的第一种安装方式是使用Linux/MacOS的 `./setup.sh` 脚本，自动安装所有依赖，并帮助你配置机器人。

确保已满足[需求条件](#requirements)，并已下载[Freqtrade仓库](#freqtrade-repository)。

### 使用 /setup.sh -install（Linux/MacOS）

如果你使用的是Debian、Ubuntu或macOS，Freqtrade提供了对应的安装脚本。

```bash
# --install，从头安装Freqtrade
./setup.sh -i
```

### 激活虚拟环境

每次打开新终端，都需要执行 `source .venv/bin/activate` 来激活虚拟环境。

```bash
# 激活虚拟环境
source ./.venv/bin/activate
```

[你已准备好](#you-are-ready)开始运行机器人。

### 其他 /setup.sh 脚本选项

你还可以使用 `./setup.sh` 更新、配置或重置机器人代码库。

```bash
# --update，拉取最新代码
./setup.sh -u
# --reset，硬重置分支
./setup.sh -r
```

```
** --install **

此选项将自动安装机器人及大部分依赖：
前提是已提前安装git和python3.10以上版本。

* 关键软件：`ta-lib`
* 在 `.venv/` 下配置虚拟环境

此选项会结合安装任务与`--reset`。

** --update **

此选项会拉取当前分支的最新版本，并更新虚拟环境。建议定期运行，以保持机器人最新。

** --reset **

此选项会硬重置当前分支（前提是你在`stable`或`develop`），并重新建立虚拟环境。
```

-----

## 手动安装

确保已满足[需求条件](#requirements)，并已下载[Freqtrade仓库](#freqtrade-repository)。

### 安装TA-Lib

#### TA-Lib脚本安装

```bash
sudo ./build_helpers/install_ta-lib.sh
```

!!! Note
    这会使用本仓库内包含的ta-lib tar.gz包。

##### 手动安装TA-Lib

[官方安装指南](https://ta-lib.github.io/ta-lib-python/install.html)

```bash
wget http://prdownloads.sourceforge.net/ta-lib/ta-lib-0.4.0-src.tar.gz
tar xvzf ta-lib-0.4.0-src.tar.gz
cd ta-lib
sed -i.bak "s|0.00000001|0.000000000000000001 |g" src/ta_func/ta_utility.h
./configure --prefix=/usr/local
make
sudo make install
# 在Debian系统（如Debian、Ubuntu等）可能需要运行ldconfig
sudo ldconfig  
cd ..
rm -rf ./ta-lib*
```

### 配置Python虚拟环境（virtualenv）

在隔离环境中运行Freqtrade。

```bash
# 创建虚拟环境在 /freqtrade/.venv
python3 -m venv .venv

# 激活虚拟环境
source .venv/bin/activate
```

### 安装Python依赖

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
# 安装freqtrade
python3 -m pip install -e .
```

[你已准备好](#you-are-ready)开始运行机器人。

### （可选）安装后任务

!!! Note 
    如果在服务器上运行机器人，建议使用 [Docker](docker_quickstart.md) 或类似 `screen`、[`tmux`](https://en.wikipedia.org/wiki/Tmux) 的终端多路复用工具，以避免退出后机器人停止。

在基于 `systemd` 的Linux系统中，作为一项可选的后续配置，还可以将机器人设置为 `systemd 服务`，或配置其将日志输出到 `syslog`/`rsyslog`或 `journald`，详见 [高级日志配置](advanced-setup.md#advanced-logging)。

------

## Conda环境安装

Freqtrade也可通过 Miniconda 或 Anaconda 安装。建议使用Miniconda，因其占用空间更小。Conda会自动准备并管理Freqtrade所需的庞大依赖库。

### 什么是Conda？

Conda是一个多语言的包、依赖和环境管理工具：[conda文档](https://docs.conda.io/projects/conda/en/latest/index.html)

### 使用Conda安装

#### 安装Conda

[在Linux上安装](https://conda.io/projects/conda/en/latest/user-guide/install/linux.html#install-linux-silent)

[在Windows上安装](https://conda.io/projects/conda/en/latest/user-guide/install/windows.html)

安装过程中请答复所有提示。安装完成后，必须重新打开终端。

#### 下载Freqtrade

```bash
# 克隆仓库
git clone https://github.com/freqtrade/freqtrade.git

# 进入仓库目录
cd freqtrade
```

#### 在Conda环境中安装Freqtrade

```bash
conda create --name freqtrade python=3.12
```

!!! Note "创建Conda环境"
    `conda create -n` 命令会自动安装所选库的所有依赖，安装命令结构如下：

```bash
# 选择你需要的包
conda env create -n [你的环境名] [python版本] [包列表]
```

#### 进入/退出freqtrade环境

查看所有环境：

```bash
conda env list
```

激活环境：

```bash
# 激活环境
conda activate freqtrade
```

退出环境（此处暂不执行）：

```bash
# 退出任何conda环境
conda deactivate
```

用pip安装最新依赖：

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
python3 -m pip install -e .
```

在Linux上修补conda的libta-lib（仅适用）：

```bash
# 确保环境已激活！
conda activate freqtrade

cd build_helpers
bash install_ta-lib.sh ${CONDA_PREFIX} nosudo
```

[你已准备好](#you-are-ready)开始运行机器人。

### 常用快捷命令

```bash
# 列出已安装的conda环境
conda env list

# 激活base环境
conda activate

# 激活freqtrade环境
conda activate freqtrade

# 退出所有conda环境
conda deactivate
```

### 其他关于Anaconda的说明

!!! Info "新出现的重量级包"
    有时，创建新的Conda环境（包含特定包）所用时间，比在已建环境中安装大包或应用所需时间更短。

!!! Warning "在conda中使用pip"
    conda文档指出，不应在conda环境中频繁使用`pip`，避免潜在问题，但实际上问题较少。[关于在conda中使用pip的博客](https://www.anaconda.com/blog/using-pip-in-a-conda-environment)

    更推荐使用`conda-forge`渠道：
    
    * 存在更多的库（减少对`pip`的依赖）
    * `conda-forge` 与`pip`兼容性更好
    * 相关库更新更快

祝您交易愉快！

-----

## 你已准备好

你已成功安装Freqtrade，接下来可以进行配置。

### 初始化配置

```bash
# 第一步 - 初始化用户目录
freqtrade create-userdir --userdir user_data

# 第二步 - 创建配置文件
freqtrade new-config --config user_data/config.json
```

配置完成后，即可开始运行，建议阅读 [交易机器人配置](configuration.md)，初次建议设为 `dry_run: True`，确保一切正常。

有关配置的详细设置方法，请参考 [交易机器人配置](configuration.md) 页面。

### 启动交易机器人

```bash
freqtrade trade --config user_data/config.json --strategy SampleStrategy
```

!!! Warning
    在正式开交易之前，务必仔细阅读剩余文档，回测策略，使用模拟交易（dry-run）验证一切无误后再开启实盘交易。

-----

## 常见问题排查

### 常见问题：“command not found”

如果你使用了（1）`脚本`或（2）`手动`安装方式，必须在虚拟环境中运行机器人。若出现如下错误，请确保虚拟环境已激活。

```bash
# 如果：
bash: freqtrade: command not found

# 需要激活虚拟环境
source ./.venv/bin/activate
```

### macOS安装错误

新版macOS可能出现类似`error: command 'g++' failed with exit status 1`的错误。这可能是因为未安装SDK头文件。

在macOS 10.14上，可以执行：

```bash
open /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg
```

若此文件不存在，说明你的macOS版本不同，请咨询网上相关解决方案。