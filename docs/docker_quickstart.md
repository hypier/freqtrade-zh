# 使用 Docker 运行 Freqtrade

本页面说明了如何使用 Docker 来运行交易机器人。此方法并非开箱即用，你仍需阅读相关文档并理解正确的配置方式。

## 安装 Docker

首先在你的平台上下载并安装 Docker / Docker Desktop：

* [Mac](https://docs.docker.com/docker-for-mac/install/)
* [Windows](https://docs.docker.com/docker-for-windows/install/)
* [Linux](https://docs.docker.com/install/)

!!! 信息 "Docker compose 安装"
    Freqtrade 文档默认假设使用 Docker Desktop（或 docker compose 插件）。  
    虽然独立的 `docker-compose` 安装仍然可行，但需要将所有 `docker compose` 命令中的空格改为中横线，即从 `docker compose` 改为 `docker-compose` 才能正常使用（例如，`docker compose up -d` 变为 `docker-compose up -d`）。

??? 警告 "Windows 上的 Docker"
    如果你刚在 Windows 系统上安装了 Docker，确保重启系统，否则可能会遇到与网络连接相关的莫名问题。

## 使用 Docker 运行 Freqtrade

Freqtrade 在 [Dockerhub](https://hub.docker.com/r/freqtradeorg/freqtrade/) 上提供了官方 Docker 镜像，以及一个[示例docker compose文件](https://github.com/freqtrade/freqtrade/blob/stable/docker-compose.yml)，可以直接使用。

!!! 注意
    - 以下内容假设你已安装并可以使用 `docker` 命令。
    - 所有以下命令都基于相对路径，必须在包含 `docker-compose.yml` 文件的目录中执行。

### Docker 快速入门

创建一个新目录，将[docker-compose 文件](https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml)放入此目录。

``` bash
mkdir ft_userdata
cd ft_userdata/
# 从仓库下载 docker-compose 文件
curl https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml -o docker-compose.yml

# 拉取 freqtrade 镜像
docker compose pull

# 创建用户目录结构
docker compose run --rm freqtrade create-userdir --userdir user_data

# 创建配置 - 需要回答交互式问题
docker compose run --rm freqtrade new-config --config user_data/config.json
```

上述操作会创建名为 `ft_userdata` 的目录，下载最新的 compose 文件，并拉取 freqtrade 镜像。  
最后两步会在 `user_data` 目录下创建用户目录，并根据你的选择交互式生成默认配置。

!!! 问题 "如何编辑机器人配置？"
    你可以随时编辑配置文件 `user_data/config.json`（在 `ft_userdata` 目录内）。  
    你也可以通过编辑 `docker-compose.yml` 文件中的命令部分，修改策略和指令。

#### 添加自定义策略

1. 配置文件已保存为 `user_data/config.json`
2. 将自定义策略复制到 `user_data/strategies/` 目录
3. 在 `docker-compose.yml` 文件中添加该策略的类名

默认运行的是 `SampleStrategy`。

!!! 危险 "`SampleStrategy` 仅为示例!"
    `SampleStrategy` 仅供参考，帮助你了解策略结构。  
    请务必在实际交易前进行回测，并使用模拟交易（dry-run）一段时间，确保策略稳定性！  
    关于策略开发的详细信息，请参考 [Strategy 文档](strategy-customization.md)。

完成上述操作后，即可启动交易机器人（选择模拟或实盘，取决于你之前的选择）。

``` bash
docker compose up -d
```

!!! 警告 "默认配置"
    尽管生成的配置大致可用，但启动机器人之前，仍需确认所有选项是否符合你的要求（如定价、配对名单等）。

#### 访问 UI

如果在 `new-config` 步骤中启用了 FreqUI，你可以在端口 `localhost:8080` 访问频繁交易界面。

在浏览器中输入：`localhost:8080`

??? 提示 "远程服务器上的 UI 访问"
    如果你在 VPS 上运行，应考虑使用 SSH 隧道或设置 VPN（如 openVPN、wireguard）连接到你的机器人。  
    这样可以避免频繁交易界面直接暴露在公网，出于安全考虑（freqUI 默认不支持 HTTPS）。  
    相关工具的设置不在本教程范围，但网络上有许多详细教程。  
    另外，建议阅读 [通过 Docker 配置 API](rest-api.md#configuration-with-docker) 部分，了解相关配置。

#### 监控机器人

可以用 `docker compose ps` 查看运行中的实例。  
正常情况下，会显示 `freqtrade` 服务处于 `running` 状态。如果没有，建议检查日志（见下一部分）。

#### 查看 Docker compose 日志

日志会写入：`user_data/logs/freqtrade.log`。  
也可以用以下命令实时查看最新日志：

``` bash
docker compose logs -f
```

#### 数据库

交易数据存放在：`user_data/tradesv3.sqlite`

#### 使用 Docker 升级 freqtrade

升级频率策略的步骤如下：

``` bash
# 拉取最新镜像
docker compose pull
# 重新启动容器
docker compose up -d
```

此操作会拉取最新版本的镜像，并用新镜像重启容器。

!!! 警告 "查看变更日志"
    升级前请务必检查变更日志，确认是否存在破坏性变更或需要手动调整的事项，确保机器人正常启动。

### 编辑 docker-compose 文件

高级用户可以进一步编辑 `docker-compose.yml` 文件，添加所有可能的参数或选项。

所有 `freqtrade` 参数都可以通过运行以下命令获得：

`docker compose run --rm freqtrade <command> <optional arguments>`

!!! 警告 "`docker compose` 用于交易指令"
    交易指令（如 `freqtrade trade <...>`）请不要通过 `docker compose run` 执行，而应使用 `docker compose up -d`。  
    这样可以确保容器正常启动（包括端口转发），并且系统重启后容器会自动重启。  
    如果计划使用 freqUI，也请相应调整配置（详见 [配置部分](rest-api.md#configuration-with-docker)），否则界面无法访问。

!!! 注意 "`docker compose run --rm`"
    加上 `--rm` 会在命令结束后自动删除容器，强烈建议在非交易模式（如回测、数据下载）使用，交易时不要加。

??? 说明 "不使用 Docker compose 也可以"
    "`docker compose run --rm`" 需要提供 compose 文件。  
    一些无需认证的命令（如 `list-pairs`）可以用 `docker run --rm` 执行，例如：  
    `docker run --rm freqtradeorg/freqtrade:stable list-pairs --exchange binance --quote BTC --print-json`。  
    这样可以在不影响运行中的容器的情况下获取交易所信息，便于添加到配置文件。

#### 示例：使用 Docker 下载数据

下载 Binance 上 ETH/BTC 对的过去 5 天的回测数据，时间周期为 1 小时，存入 `user_data/data/` 目录。

``` bash
docker compose run --rm freqtrade download-data --pairs ETH/BTC --exchange binance --days 5 -t 1h
```

详细请参考 [数据下载文档](data-download.md)。

#### 示例：使用 Docker 进行回测

用 Docker 容器对 `SampleStrategy` 进行回测，时间范围为 2019年8月1日至2019年10月1日，时间周期为 5 分钟：

``` bash
docker compose run --rm freqtrade backtesting --config user_data/config.json --strategy SampleStrategy --timerange 20190801-20191001 -i 5m
```

详细信息请查阅 [回测文档](backtesting.md)。

### 使用 Docker 时的其他依赖

如果你的策略需要默认镜像中未包含的依赖，需自行在主机上构建镜像。  
可在项目中创建一个 Dockerfile，写入额外依赖的安装步骤（可参考 [docker/Dockerfile.custom](https://github.com/freqtrade/freqtrade/blob/develop/docker/Dockerfile.custom)）。

修改 `docker-compose.yml`，取消注释 build 配置，并将镜像命名为避免冲突（如下所示）：

``` yaml
    image: freqtrade_custom
    build:
      context: .
      dockerfile: "./Dockerfile.<你自己的扩展名>"
```

完成后，运行：

``` bash
docker compose build --pull
```

即可构建自定义镜像，然后按照之前的方法启动。

### 使用 Docker 进行绘图

可以通过修改镜像名称为 `*_plot`，在 `docker-compose.yml` 文件中调用 `freqtrade plot-profit` 和 `freqtrade plot-dataframe`（详见 [绘图指南](plotting.md)）。

例如：

``` bash
docker compose run --rm freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH --timerange=20180801-20180805
```

绘图结果会存放在 `user_data/plot` 目录，可用任意现代浏览器打开。

### 使用 Docker 进行数据分析

Freqtrade 提供了一个启动 Jupyter Lab 服务器的 docker-compose 文件。  
可以运行以下命令启动：

``` bash
docker compose -f docker/docker-compose-jupyter.yml up
```

会创建一个运行 Jupyter Lab 的容器，可以通过 `https://127.0.0.1:8888/lab` 访问。  
启动后，控制台会显示一个访问链接，用于快速登录。

建议定期重建镜像，保持频繁交易（freqtrade）及相关依赖的最新状态：

``` bash
docker compose -f docker/docker-compose-jupyter.yml build --no-cache
```

## 故障排查

### Windows 上的 Docker

* 错误：“`Timestamp for this request is outside of the recvWindow.`”  
  交易所 API 请求需要时间同步，但 docker 容器中的时钟会随着时间逐渐偏移。  
  临时解决方案：运行 `wsl --shutdown`，然后重启 Docker（在 Windows 10 上会弹出提示）。  
  长期方案：在 Linux 主机上运行 Docker，或定期重启 WSL（使用调度任务）。

  操作示例：
  
``` bash
taskkill /IM "Docker Desktop.exe" /F
wsl --shutdown
start "" "C:\Program Files\Docker\Docker\Docker Desktop.exe"
```

* 无法连接 API（Windows）  
  在 Windows 上新安装 Docker 后，确保重启系统。否则可能会出现网络连接问题。  
  同时确认你的 [设置](#accessing-the-ui) 已正确配置。

!!! 警告
    基于上述情况，不建议在 Windows 环境用于正式生产，仅推荐用于测试、数据下载和回测。  
    更稳定的方案是使用 Linux VPS 托管频繁交易机器人。