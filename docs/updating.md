# 如何更新

要更新你的 freqtrade 安装，请使用以下方法之一，选择与你的安装方式相对应的方法。

!!! Note "跟踪变更"
    破坏性变更或行为变更将在每个版本发布时的更新日志中记录。
    对于开发分支，请关注 PR，以避免被突如其来的变更所惊吓。

## Docker

!!! Note "使用 `master` 镜像的旧版本安装"
    我们正在将发布镜像的基础从 `master` 迁移到 `stable`，请调整你的 Docker 文件，将 `freqtradeorg/freqtrade:master` 替换为 `freqtradeorg/freqtrade:stable`

``` bash
docker compose pull
docker compose up -d
```

## 通过安装脚本进行安装

``` bash
./setup.sh --update
```

!!! Note
    确保在运行此命令时你的虚拟环境已关闭！

## 纯本地原生安装

请确保你也在更新依赖项，否则可能会出现未被注意到的问题。

``` bash
git pull
pip install -U -r requirements.txt
pip install -e .

# 确保 freqUI 为最新版本
freqtrade install-ui 
```

### 更新中遇到的问题

更新问题通常由缺失的依赖导致（你没有按照上述指引操作）——或者是因为依赖更新后无法成功安装（例如 TA-lib）。
请参考相应的安装指南（以下链接中包括常见问题解答）

常见问题及解决方案：

* [ta-lib 在 Windows 上的更新](windows_installation.md#2-install-ta-lib)