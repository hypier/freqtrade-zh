# FreqUI

Freqtrade 内置了一个网页服务器，可提供 [FreqUI](https://github.com/freqtrade/frequi)，即 freqtrade 的前端界面。

默认情况下，UI 会在安装过程中自动安装（脚本安装、Docker 镜像）。  
你也可以通过手动执行 `freqtrade install-ui` 命令来安装 freqUI。  
同一个命令还可以用来将 freqUI 更新到最新版本。

一旦在交易或模拟交易模式下启动机器人（使用 `freqtrade trade`）——界面将在配置好的 API 端口上提供（默认为 `http://127.0.0.1:8080`）。

??? 提示 "想为 freqUI 做贡献？"  
开发者不应使用此方法，而应克隆相应的仓库，使用 [freqUI 仓库](https://github.com/freqtrade/frequi) 中描述的方法获取源码。  
构建前端界面时需要安装 Node.js。

!!! 提示 "运行 freqtrade 并不需要 freqUI"  
freqUI 是 freqtrade 的一个可选组件，并非运行机器人所必需。  
它是一个前端界面，用于监控和与机器人交互——但没有它，freqtrade 依然可以正常运行。

## 配置

FreqUI 没有自己的配置文件——但假设已配置好 [rest-api](rest-api.md) 的正常运行。  
请参考相应的文档页面以完成 freqUI 的配置。

## 界面

FreqUI 是一个现代、响应式的网页应用，可以用来监控和操作你的机器人。

FreqUI 支持浅色主题和深色主题。  
主题可以通过页面顶部的显著按钮轻松切换。  
本文档中的截图主题会根据所选的文档主题自动调整，因此要查看深色（或浅色）版本，请切换文档的主题。

### 登录界面

下方截图显示了 freqUI 的登录页面。

![FreqUI - login](assets/frequi-login-CORS.png#only-dark)  
![FreqUI - login](assets/frequi-login-CORS-light.png#only-light)

!!! 提示 "CORS"  
此截图中显示的 CORS 错误，是因为界面运行的端口与 API 不同，且 [CORS](#cors) 配置尚未正确设置。

### 交易视图

交易视图允许你可视化机器人执行的交易，并与机器人交互。  
在此页面，你还可以通过启动和停止机器人，以及在配置允许的情况下，强制进行交易入场和退出操作。

![FreqUI - trade view](assets/freqUI-trade-pane-dark.png#only-dark)  
![FreqUI - trade view](assets/freqUI-trade-pane-light.png#only-light)

### 绘图配置器

FreqUI 的绘图可以通过策略中的 `plot_config` 配置对象（支持通过“从策略加载”按钮加载）或通过界面进行配置。  
可以创建和切换多个绘图配置，以实现对图表的不同视角。

绘图配置可以通过交易视图右上角的“绘图配置器”按钮（齿轮图标）打开。

![FreqUI - plot configuration](assets/freqUI-plot-configurator-dark.png#only-dark)  
![FreqUI - plot configuration](assets/freqUI-plot-configurator-light.png#only-light)

### 设置

可以通过访问设置页面，修改多项界面相关的设置。

你可以调整的内容包括但不限于：

* 界面所用的时区
* 浏览器标签页 favicon 上的未平仓交易显示
* K线颜色（涨/跌 — 红色/绿色）
* 开启或关闭应用内通知类型

![FreqUI - Settings view](assets/frequi-settings-dark.png#only-dark)  
![FreqUI - Settings view](assets/frequi-settings-light.png#only-light)

## 回测

当在 [网页服务器模式](utils.md#webserver-mode)下启动 freqtrade（使用 `freqtrade webserver` 命令）时，回测界面将变得可用。  
该界面允许你进行策略回测并可视化结果。

你还可以加载并分析历史回测数据，或者将多次回测结果进行对比。

![FreqUI - Backtesting](assets/freqUI-backtesting-dark.png#only-dark)  
![FreqUI - Backtesting](assets/freqUI-backtesting-light.png#only-light)