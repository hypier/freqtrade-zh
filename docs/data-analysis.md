# 使用 Jupyter 笔记本分析机器人数据

你可以轻松地使用 Jupyter 笔记本分析回测和交易历史的结果。示例笔记本位于 `user_data/notebooks/`，在通过 `freqtrade create-userdir --userdir user_data` 初始化用户目录后即可访问。

## 使用 Docker 快速入门

Freqtrade 提供了一个 docker-compose 文件，可以启动一个 Jupyter Lab 服务器。你可以使用以下命令运行此服务器：`docker compose -f docker/docker-compose-jupyter.yml up`

这样会创建一个运行 Jupyter Lab 的 Docker 容器，可通过 `https://127.0.0.1:8888/lab` 访问。启动后会在控制台打印出一个链接，建议使用该链接进行简便登录。

更多信息，请访问 [Data analysis with Docker](docker_quickstart.md#data-analysis-using-docker-compose) 部分。

### 技巧分享

* 参阅 [jupyter.org](https://jupyter.org/documentation) 获取使用说明。
* 切记在你的 conda 或 venv 环境中启动 Jupyter 笔记本服务器，或者使用 [nb_conda_kernels](https://github.com/Anaconda-Platform/nb_conda_kernels)。
* 在使用前复制示例笔记本，避免你的修改在下一次 `freqtrade` 更新时被覆盖。

### 使用虚拟环境结合系统范围的 Jupyter 安装

有时希望使用系统范围的 Jupyter Notebook 安装，同时使用虚拟环境中的 Jupyter 内核。这可以避免在每台系统上多次安装完整的 Jupyter 套件，并方便在任务（如 freqtrade / 其他分析任务）之间切换。

操作步骤如下，先激活虚拟环境，然后运行：

```bash
# 激活虚拟环境
source .venv/bin/activate

pip install ipykernel
ipython kernel install --user --name=freqtrade
# 重启 Jupyter（Lab / Notebook）
# 在笔记本中选择内核 "freqtrade"
```

!!! 注意
    本节内容旨在补充说明，Freqtrade 团队不会对此设置过程中遇到的问题提供全面支持。建议直接在虚拟环境中安装 Jupyter，以最快速地启动和运行笔记本。如需帮助，请参考 [Project Jupyter](https://jupyter.org/) 的 [文档](https://jupyter.org/documentation) 或 [帮助渠道](https://jupyter.org/community)。

!!! 警告
    有些任务在笔记本中运行效果不佳。例如，所有使用异步执行的操作在 Jupyter 中都可能遇到问题。此外，freqtrade 的主要入口点是命令行界面（CLI），在笔记本中纯粹使用 Python 可能会绕过传递必要参数和对象的命令行参数。你可能需要手动设置这些参数或创建所需的对象。

## 推荐的工作流程

| 任务 | 工具 |
|---|---|
| 机器人操作 | CLI |
| 重复性任务 | Shell 脚本 |
| 数据分析与可视化 | 笔记本 |

1. 使用 CLI 完成以下操作：

    * 下载历史数据
    * 运行回测
    * 使用实时数据进行交易
    * 导出结果

2. 将这些操作封装在 Shell 脚本中：

    * 保存带参数的复杂命令
    * 执行多步操作
    * 自动化测试策略和准备分析用的数据

3. 使用笔记本进行：

    * 数据可视化
    * 数据处理与绘图，生成洞见

## 示例工具片段

### 更改目录到项目根目录

Jupyter 笔记本默认在笔记本目录执行。以下代码片段会搜索项目根目录，确保相对路径保持一致。

```python
import os
from pathlib import Path

# 更改目录
# 修改此单元以确保输出显示正确路径
# 所有路径相对于在输出中显示的项目根目录定义
project_root = "somedir/freqtrade"
i=0
try:
    os.chdir(project_root)
    assert Path('LICENSE').is_file()
except:
    while i<4 and (not Path('LICENSE').is_file()):
        os.chdir(Path(Path.cwd(), '../'))
        i+=1
    project_root = Path.cwd()
print(Path.cwd())
```

### 加载多个配置文件

此选项便于检查传入多个配置文件的结果。它会完整地运行配置初始化流程，使配置完全被初始化后，可传递给其他方法。

```python
import json
from freqtrade.configuration import Configuration

# 从多个文件加载配置
config = Configuration.from_files(["config1.json", "config2.json"])

# 展示内存中的配置
print(json.dumps(config['original_config'], indent=2))
```

对于交互式环境，可以设置额外的配置项 `user_data_dir`，最后传入，这样运行机器人时就不需要更改目录。

```json
{
    "user_data_dir": "~/.freqtrade/"
}
```

## 进一步的数据分析文档

* [策略调试](strategy_analysis_example.md) — 也可在 Jupyter 笔记本中查看（`user_data/notebooks/strategy_analysis_example.ipynb`）
* [绘图](plotting.md)
* [标签分析](advanced-backtesting.md)

如果你有任何关于如何更好分析数据的想法，欢迎提交 Issue 或 Pull Request 来丰富完善本文档。