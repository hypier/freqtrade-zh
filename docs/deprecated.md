# 已弃用功能

本页描述了开发团队宣告为已弃用且不再支持的命令行参数、配置参数及机器人功能。请避免在配置中使用它们。

## 被移除的功能

### `--refresh-pairs-cached` 命令行选项

在回测、超参数优化和边缘检测的场景中，`--refresh-pairs-cached` 用于刷新回测的蜡烛数据。由于此选项可能引起混淆且会降低回测速度（而且并非严格必要），因此被单独作为一个频率子命令 `freqtrade download-data` 提供。

该命令行参数于 2019.7-dev（开发分支）被弃用，并在 2019.9 版本中移除。

### **--dynamic-whitelist** 命令行选项

此命令行选项于 2018 年被弃用，并于 `freqtrade 2019.6-dev`（开发分支）和 2019.7 版中被移除。

请参考 [pairlists](plugins.md#pairlists-and-pairlist-handlers)。

### `--live` 命令行选项

在回测场景中，`--live` 用于下载最新的逐笔数据以便回测。该选项只下载了最新的 500 根蜡烛，因此获取优质回测数据的效果有限。此功能在 2019-7-dev（开发分支）和 `freqtrade 2019.8` 中被移除。

### `ticker_interval`（现改为 `timeframe`）

`ticker_interval` 相关支持于 2020.6 被弃用，建议使用 `timeframe`。兼容性代码于 2022.3 版本中被删除。

### 支持顺序运行多个 pairlist

之前配置中的 `"pairlist"` 段已被移除，取而代之的是 `"pairlists"`，这是一个列表，用于指定一系列的 pairlist。

旧的配置参数段（`"pairlist"`）于 2019.11 被废弃，并在 2020.4 版本中彻底移除。

### volume-pairlist 中 bidVolume 和 askVolume 的弃用

由于只有 quoteVolume 可以在不同资产间比较，`bidVolume` 和 `askVolume` 两项在 2020.4 被弃用，并于 2020.9 被移除。

### 使用订单簿步长作为退出价格

之前可以使用 `order_book_min` 和 `order_book_max` 来逐步调整订单簿，尝试找到下一个 ROI 区间，以提前挂卖单。然而，这种做法风险较高且没有明显好处，因此在 2021.7 版本中为保持代码简洁而被移除。

### 旧版 Hyperopt 模式

使用单独的超参数优化文件的方式于 2021.4 被弃用，并在 2021.9 版本中移除。建议切换到新的 [Parametrized Strategies](hyperopt.md)，以便体验新版超参数优化接口。

## V2 与 V3 策略之间的变化

2022.4 版本引入了单独的期货合约/短线交易（Isolated Futures / short trading），这对配置、策略接口等方面提出了重大修改。

我们努力保持与现有策略的兼容性，因此如果你只是继续使用现货市场的频率交易，不需要做任何更改。未来我们可能会停止支持当前接口，但会提前公告并提供过渡期。

请参阅 [策略迁移指南](strategy_migration.md)，以将你的策略迁移到新格式，启用新功能。

### Webhook - 2022.4 版本的变更

#### `buy_tag` 已更名为 `enter_tag`

此更改应仅影响你的策略及可能涉及的 Webhook 配置。我们会保留一两个版本的兼容层（即 `buy_tag` 和 `enter_tag` 同时支持）以确保平滑过渡，但之后会逐步废弃对旧名称的支持。

#### 命名变更

Webhook 术语从 "sell" 改为 "exit"，从 "buy" 改为 "entry"，同时移除 "webhook" 字样。

* `webhookbuy`、`webhookentry` -> `entry`
* `webhookbuyfill`、`webhookentryfill` -> `entry_fill`
* `webhookbuycancel`、`webhookentrycancel` -> `entry_cancel`
* `webhooksell`、`webhookexit` -> `exit`
* `webhooksellfill`、`webhookexitfill` -> `exit_fill`
* `webhooksellcancel`、`webhookexitcancel` -> `exit_cancel`

## 移除 `populate_any_indicators`

2023.3 版本中，`populate_any_indicators` 被移除，改为使用拆分的特征工程和目标函数方法。请查阅 [迁移文档](strategy_migration.md#freqai-strategy) 获取详细信息。

## 从配置中移除 `protections`

在 2024.10 版本中，取消了通过 `"protections": []` 在配置中设置防护措施的功能，此前已连续提醒弃用超过 3 年。

## hdf5 数据存储

2024.12 版本开始不推荐使用 hdf5 作为数据存储，2025.1 版本中已完全移除。建议切换到 feather 格式。

在升级之前，请使用 [`convert-data` 子命令](data-download.md#sub-command-convert-data) 将现有数据转换为支持的格式。

## 通过配置文件设置高级日志

在 2025.3 版本中，使用 `--logfile systemd` 和 `--logfile journald` 配置 syslog 和 journald 方式的日志已被弃用。请改用基于配置的 [日志设置](advanced-setup.md#advanced-logging) 方案。