# 高级回测分析

## 分析买/入场和卖/出场标记

理解策略根据使用的买入/入场标记（buy/entry tags）表现的方式可能很有帮助。这些标签用于标识不同的买入条件。你可能希望获得比默认回测输出更复杂的统计数据，以深入了解每个买卖条件。你也可能希望在触发交易开仓的信号蜡烛上，确定指标的具体数值。

!!! 注意
    以下买入原因分析仅在回测模式下可用，*不适用于 hyperopt*。

我们需要在运行回测时，将 `--export` 选项设置为 `signals`，以启用信号 **和** 交易的导出：

```bash
freqtrade backtesting -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange=<timerange> --export=signals
```

这将指示 freqtrade 输出一个存储策略、交易对以及导致开仓和出场信号的蜡烛的字典（pkl格式）。这个字典会包含对应的 DataFrame。根据你的策略下单次数不同，该文件可能会变得相当大，因此建议定期检查 `user_data/backtest_results` 文件夹，删除旧的导出文件。

在进行下一次回测之前，请确保你删除了旧的回测结果，或者在运行时使用 `--cache none` 选项，以确保不使用缓存结果。

如果一切顺利，你应该会在 `user_data/backtest_results` 文件夹中看到类似如下的文件：

- `backtest-result-{timestamp}_signals.pkl`
- `backtest-result-{timestamp}_exited.pkl`

要分析这些买卖标记（entry/exit tags），需要使用 `freqtrade backtesting-analysis` 命令，并提供 `--analysis-groups` 选项，后跟用空格分隔的参数：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 1 2 3 4 5
```

此命令会读取最近一次的回测结果。`--analysis-groups` 选项用来设置不同的输出表格，从简单（0）到每对每个买卖标签的详细统计（4）：

* 0：整体胜率和按enter_tag的利润汇总
* 1：按enter_tag分组的利润汇总
* 2：按enter_tag和exit_tag分组的利润汇总
* 3：按交易对和enter_tag分组的利润汇总
* 4：按交易对、enter_tag和exit_tag分组的利润汇总（可能会很大）
* 5：按exit_tag分组的利润汇总

更多选项可使用 `-h` 查看。

### 使用 export-filename

通常情况下，`backtesting-analysis` 会自动使用最新的回测结果。但如果你想分析早期的某次回测输出，需要提供 `--export-filename` 选项。可以将此参数设置为你想分析的特定回测输出文件名，从而保存多个历史版本的回测结果，便于日后复查。

例如：

```bash
freqtrade backtesting -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange=<timerange> --export=signals --export-filename=/tmp/mystrat_backtest.json
```

在日志中，你会看到类似如下的输出，显示导出的文件名和时间戳：

```
2022-06-14 16:28:32,698 - freqtrade.misc - INFO - dumping json to "/tmp/mystrat_backtest-2022-06-14_16-28-32.json"
```

之后，在 `backtesting-analysis` 时，可以指定该文件名：

```bash
freqtrade backtesting-analysis -c <config.json> --export-filename=/tmp/mystrat_backtest-2022-06-14_16-28-32.json
```

### 调整显示的买卖标签（买卖原因）

可以通过以下两个选项只显示特定的买入或卖出原因：

```
--enter-reason-list : 用空格分隔的入场信号标签列表。默认值："all"
--exit-reason-list  : 用空格分隔的出场信号标签列表。默认值："all"
```

例如：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss
```

### 输出信号蜡烛的指标数值

`freqtrade backtesting-analysis` 最强大的功能之一，是可以打印出信号蜡烛上所有指标的数值，便于进行细粒度的分析和参数微调。你可以通过 `--indicator-list` 选项指定要显示的指标列：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss --indicator-list rsi rsi_1h bb_lowerband ema_9 macd macdsignal
```

注意，这些指标必须存在于策略的主 DataFrame（无论是主时间框架还是信息时间框架）中，否则它们不会在输出中显示。

!!! 注意 "指标列表"
    指标值会同时显示在入场和出场点。如果指定 `--indicator-list all`，则只会显示入场点的指标，以避免输出过长，这在策略比较复杂时尤为重要。

除了指标外，分析还会自动包括一些蜡烛和交易相关的字段，如：

- **open_date     ：** 交易开仓时间
- **close_date    ：** 交易平仓时间
- **min_rate      ：** 在持仓期间的最低价格
- **max_rate      ：** 在持仓期间的最高价格
- **open          ：** 信号蜡烛开盘价
- **close         ：** 信号蜡烛收盘价
- **high          ：** 信号蜡烛最高价
- **low           ：** 信号蜡烛最低价
- **volume        ：** 信号蜡烛成交量
- **profit_ratio  ：** 交易利润比
- **profit_abs    ：** 交易的绝对利润

#### 指标值示例输出

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen
```

例如，目标是显示每笔交易的 `chikou_span` 和 `tenkan_sen` 指标值在入场和出场的时刻。

示例输出可能如下：

| pair      | open_date                 | enter_reason | exit_reason | chikou_span (entry) | tenkan_sen (entry) | chikou_span (exit) | tenkan_sen (exit) |
|-----------|---------------------------|--------------|-------------|---------------------|--------------------|--------------------|-------------------|
| DOGE/USDT | 2024-07-06 00:35:00+00:00 |              | exit_signal | 0.105               | 0.106              | 0.105              | 0.107             |
| BTC/USDT  | 2024-08-05 14:20:00+00:00 |              | roi         | 54643.440           | 51696.400          | 54386.000          | 52072.010         |

在这个表格中，`chikou_span (entry)` 表示交易入场时的指标值，`chikou_span (exit)` 则代表出场时的值。这种详细的指标显示有助于更深入的分析。

指标名后缀中的 `(entry)` 和 `(exit)` 用于区分入场时和出场时的指标值。

!!! 注 "交易范围指标"
    某些交易范围的指标没有 `(entry)` 或 `(exit)` 后缀，包括：`pair`、`stake_amount`、`max_stake_amount`、`amount`、`open_date`、`close_date`、`open_rate`、`close_rate`、`fee_open`、`fee_close`、`trade_duration`、`profit_ratio`、`profit_abs`、`exit_reason`、`initial_stop_loss_abs`、`initial_stop_loss_ratio`、`stop_loss_abs`、`stop_loss_ratio`、`min_rate`、`max_rate`、`is_open`、`enter_tag`、`leverage`、`is_short`、`open_timestamp`、`close_timestamp` 和 `orders`。

#### 按入场或出场信号过滤指标

`--indicator-list` 默认会显示入场和出场信号的指标值。若只想显示入场信号的指标，可以使用 `--entry-only`；只想显示出场点的指标，则使用 `--exit-only`。

示例：只显示入场信号的指标值：

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --entry-only
```

示例：只显示出场信号的指标值：

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --exit-only
```

!!! 注意
    使用这些过滤参数时，指标名不会附加 `(entry)` 或 `(exit)` 后缀。

### 按日期过滤交易输出

若只想查看在回测时间范围内（timerange）的交易，可提供 `--timerange` 选项，格式为 `YYYYMMDD-[YYYYMMDD]`，其中开始日期包含，结束日期不包含。例如：

```
--timerange : 用于过滤输出交易的时间范围，起始日期（包含）-结束日期（不包含）。如：20220101-20221231
```

例如，如果回测时间范围为 `20220101-20221231`，但只想看一月的交易：

```bash
freqtrade backtesting-analysis -c <config.json> --timerange 20220101-20220201
```

### 打印被拒绝的信号

使用 `--rejected-signals` 选项，可以打印出被过滤掉的信号。

```bash
freqtrade backtesting-analysis -c <config.json> --rejected-signals
```

### 将分析表格写入CSV

部分统计表较大，不适合直接在终端输出。可以使用 `--analysis-to-csv` 选项，将输出结果保存为CSV文件而不在终端显示。

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv
```

默认情况下，每个输出表会生成对应的CSV文件，例如：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --rejected-signals --analysis-groups 0 1
```

会在 `user_data/backtest_results` 文件夹中生成：

- rejected_signals.csv
- group_0.csv
- group_1.csv

你也可以通过 `--analysis-csv-path` 选项自定义输出路径：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --analysis-csv-path another/data/path/
```