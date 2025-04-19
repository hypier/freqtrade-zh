![freqtrade](assets/freqtrade_poweredby.svg)

[![Freqtrade CI](https://github.com/freqtrade/freqtrade/actions/workflows/ci.yml/badge.svg?branch=develop)](https://github.com/freqtrade/freqtrade/actions/)
[![DOI](https://joss.theoj.org/papers/10.21105/joss.04864/status.svg)](https://doi.org/10.21105/joss.04864)
[![Coverage Status](https://coveralls.io/repos/github/freqtrade/freqtrade/badge.svg?branch=develop&service=github)](https://coveralls.io/github/freqtrade/freqtrade?branch=develop)
[![Maintainability](https://api.codeclimate.com/v1/badges/5737e6d668200b7518ff/maintainability)](https://codeclimate.com/github/freqtrade/freqtrade/maintainability)

<!-- 在您希望按钮渲染的位置放置此标签。 -->
<a class="github-button" href="https://github.com/freqtrade/freqtrade" data-icon="octicon-star" data-size="large" aria-label="Star freqtrade/freqtrade on GitHub">Star</a>
<a class="github-button" href="https://github.com/freqtrade/freqtrade/fork" data-icon="octicon-repo-forked" data-size="large" aria-label="Fork freqtrade/freqtrade on GitHub">Fork</a>
<a class="github-button" href="https://github.com/freqtrade/freqtrade/archive/stable.zip" data-icon="octicon-cloud-download" data-size="large" aria-label="Download freqtrade/freqtrade on GitHub">Download</a>

## 介绍

Freqtrade是一个用Python编写的免费开源加密货币交易机器人。它旨在支持所有主要交易所，并通过Telegram或WebUI进行控制。它包含回测、绘图和资金管理工具，以及利用机器学习进行策略优化。

!!! 警告 "免责声明"
    本软件仅用于教育目的。请勿使用你害怕会失去的资金进行交易。请自行承担使用本软件的风险。作者及所有关联方不对你的交易结果承担任何责任。

    建议你首先在模拟交易（Dry-run）中运行机器人，确保理解其工作原理以及预期的利润/亏损情况后再投入资金。

    我们强烈建议你具备基础编码技能和Python知识。不要犹豫阅读源代码，理解该机器人所采用的机制、算法和技巧。

![freqtrade 截图](assets/freqtrade-screenshot.png)

## 特性

- 开发你的策略：用Python编写你的交易策略，支持[pandas](https://pandas.pydata.org/)。可在[策略仓库](https://github.com/freqtrade/freqtrade-strategies)中找到示例策略以供参考。
- 下载市场数据：下载你可能交易的交易所和市场的历史数据。
- 回测：在下载的历史数据上测试你的交易策略。
- 优化：使用机器学习方法的超参数优化，寻找你的策略的最佳参数。你可以优化买入、卖出、盈利（ROI）、止损和追踪止损等参数。
- 选择市场：创建静态列表或使用基于交易量和/或价格排名的自动列表（回测时不可用）。也可以明确排除不希望交易的市场。
- 运行：用模拟资金（Dry-Run模式）测试策略，或以真实资金（Live-Trade模式）部署。
- 通过边缘（Edge）模块运行（可选）：通过变动止损值，寻找市场的最佳历史[交易预期](edge.md#expectancy)，再决定是否允许交易。每笔交易的规模基于资本的百分比风险。
- 控制/监控：使用Telegram或WebUI（启动/停止机器人、显示盈利/亏损、每日总结、当前持仓交易结果等）。
- 分析：可在回测数据或Freqtrade交易历史（SQL数据库）上进行更深层次的分析，包括自动标准绘图和将数据导入[交互式环境](data-analysis.md)的方法。

## 支持的交易市场

请阅读[交易所相关说明](exchanges.md)，了解每个交易所可能需要的特殊配置。

- [X] [Binance](https://www.binance.com/)
- [X] [BingX](https://bingx.com/invite/0EM9RX)
- [X] [Bitmart](https://bitmart.com/)
- [X] [Bybit](https://bybit.com/)
- [X] [Gate.io](https://www.gate.io/ref/6266643)
- [X] [HTX](https://www.htx.com/)
- [X] [Hyperliquid](https://hyperliquid.xyz/)（去中心化交易所，简称DEX）
- [X] [Kraken](https://kraken.com/)
- [X] [OKX](https://okx.com/)
- [X] [MyOKX](https://okx.com/)（OKX EEA）
- [ ] [可能支持许多其他交易所，通过 <img alt="ccxt" width="30px" src="assets/ccxt-logo.svg" />](https://github.com/ccxt/ccxt/)。**（我们不能保证它们可用）**

### 支持的期货交易所（实验性功能）

- [X] [Binance](https://www.binance.com/)
- [X] [Bybit](https://bybit.com/)
- [X] [Gate.io](https://www.gate.io/ref/6266643)
- [X] [Hyperliquid](https://hyperliquid.xyz/)（去中心化交易所，简称DEX）
- [X] [OKX](https://okx.com/)

请确保在开始前阅读[交易所相关说明](exchanges.md)，以及[杠杆交易](leverage.md)相关文档。

### 社区验证

由社区确认支持的交易所：

- [X] [Bitvavo](https://bitvavo.com/)
- [X] [Kucoin](https://www.kucoin.com/)

## 社区展示

--8<-- "includes/showcase.md"

## 系统需求

### 硬件需求

建议运行为Linux云服务器，最低配置为：

- 2GB RAM
- 1GB存储空间
- 2个虚拟CPU（vCPU）

### 软件需求

- Docker（推荐）

或者

- Python 3.10+
- pip（pip3）
- git
- TA-Lib
- virtualenv（推荐）

## 支持方式

### 帮助 / Discord

如在文档中找不到答案，或想了解更多关于机器人的信息，或与志同道合者交流，欢迎加入Freqtrade的[Discord服务器](https://discord.gg/p7nuUNVfP7)。

## 准备试用？

首先阅读[Docker安装指南](docker_quickstart.md)（推荐），或选择[非Docker安装指南](installation.md)。