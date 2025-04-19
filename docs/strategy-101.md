# Freqtrade 策略入门：策略开发快速指南

在本快速指南中，我们假设你已经掌握了交易的基础知识，并阅读过 
[Freqtrade基础知识](bot-basics.md) 页面。

## 所需基础知识

Freqtrade中的策略是一个Python类，它定义了买入和卖出加密货币资产的逻辑。

资产被定义为`对`，代表`币种`和`质押币`。币种是你用另一种货币作为质押进行交易的资产。

数据由交易所提供，以`蜡烛图`的形式呈现，由六个数值组成：`日期`、`开盘价`、`最高价`、`最低价`、`收盘价`和`交易量`。

`技术分析`函数利用各种计算和统计公式分析蜡烛图数据，并生成次级值，称为`指标`。

指标会在资产对的蜡烛图上进行分析，以生成`信号`。

信号会转化为在加密货币交易所的`订单`，即`交易`。

我们用`入场`和`出场`这两个术语替代`买入`和`卖出`，因为Freqtrade支持`多头（long）`和`空头（short）`交易。

- **多头（long）**：你用质押币购买币，比如用USDT买BTC，并通过以更高的价格卖出获利。在多头交易中，利润来自币价相对于质押币的上涨。
- **空头（short）**：你从交易所借入币（质押币），以后偿还。空头交易的利润来自币价相对于质押币的下跌（即以较低的价格买回偿还贷款）。

虽然Freqtrade支持某些交易所的现货和期货市场，为简便起见，我们只聚焦于现货（多头）交易。

## 基础策略结构

### 主要数据框（Main dataframe）

Freqtrade策略使用一种带有行列的表格结构，称为`dataframe`，用以生成进场和出场信号。

每个在配置的`pairlist`中的交易对，都有自己对应的`dataframe`。数据框是以`date`列作为索引的，例如`2024-06-31 12:00`。

接下来的5列代表`开盘价`、`最高价`、`最低价`、`收盘价`和`交易量`（OHLCV）数据。

### 填充指标值

`populate_indicators`函数在数据框中添加代表技术指标值的列。

常见指标包括相对强弱指数（RSI）、布林带（Bollinger Bands）、资金流指标（MFI）、移动平均线（MA）以及平均真实范围（ATR）。

这些列是通过调用技术分析库的函数生成的，例如ta-lib的`RSI()`，并赋值到对应列名，如`rsi`。

```python
dataframe['rsi'] = ta.RSI(dataframe)
```

??? 提示 "技术分析库"
    不同的库在生成指标值的方法上会有所差异。请查阅每个库的文档以了解如何将其集成到你的策略中。此外，你也可以参考[Freqtrade示例策略](https://github.com/freqtrade/freqtrade-strategies)，获取更多灵感。

### 填充入场信号

`populate_entry_trend`函数定义了生成入场信号的条件。

在数据框中添加`enter_long`列，当该列值为`1`时，Freqtrade会识别为入场信号。

??? 提示 "做空"
    若要开启空头交易，可使用`enter_short`列。

### 填充出场信号

`populate_exit_trend`函数定义了生成出场信号的条件。

在数据框中添加`exit_long`列，当该列值为`1`时，Freqtrade会识别为出场信号。

??? 提示 "做空"
    若要平空头仓位，可使用`exit_short`列。

## 一个简易策略示例

以下是一个最简的Freqtrade策略示例：

```python
from freqtrade.strategy import IStrategy
from pandas import DataFrame
import talib.abstract as ta

class MyStrategy(IStrategy):

    timeframe = '15m'

    # 初始止损设为 -10%
    stoploss = -0.10

    # 当盈利超过1%时，任意时间退出盈利仓位
    minimal_roi = {"0": 0.01}

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 生成技术分析指标的数值
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 根据指标值生成入场信号
        dataframe.loc[
            (dataframe['rsi'] < 30),
            'enter_long'] = 1

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 根据指标值生成出场信号
        dataframe.loc[
            (dataframe['rsi'] > 70),
            'exit_long'] = 1

        return dataframe
```

## 交易执行

当检测到信号（入场或出场列中出现`1`）时，Freqtrade会尝试发出订单，即执行`交易`或`仓位`。

每个新开仓位置会占用一个`槽位`。槽位代表最大可同时开持的交易仓位数。

槽位数量由`max_open_trades`配置项决定。

不过，在实际操作中，产生信号并不一定总会生成交易订单，原因包括：

- 钱包中的剩余资金不足以买入资产或卖出资产（包括手续费）
- 没有足够的空闲槽位开启新仓位（你已开仓的仓位数等于`max_open_trades`）
- 已经存在某个交易对的未平仓交易（Freqtrade不能叠加仓位——但可以通过[调整仓位](strategy-callbacks.md#adjust-trade-position)进行管理）
- 如果在同一蜡烛上同时出现入场和出场信号，这些被视为[冲突](strategy-customization.md#colliding-signals)，不会执行订单
- 策略会根据你在[入场](strategy-callbacks.md#trade-entry-buy-order-confirmation)或[出场](strategy-callbacks.md#trade-exit-sell-order-confirmation)回调中指定的逻辑主动拒绝订单

更多详细信息请阅读[策略定制](strategy-customization.md)文档。

## 回测与正向测试

策略开发可能是一个漫长且充满挫折的过程，将人为的“直觉”转化为可运行的自动交易策略（“算法”策略）并非总是直观简单。

因此，建议对策略进行测试，以确认其预期表现。

Freqtrade支持两种测试模式：

- **回测**：利用从[交易所下载](data-download.md)的历史数据，快速评估策略性能。然而，结果可能被人为扭曲，使策略看起来比实际盈利能力更高。详情请查阅[回测文档](backtesting.md)。
- **模拟运行（Dry Run）**：也叫**前向测试（Forward Testing）**，使用来自交易所的实时数据。但是，任何会导致交易的信号都被Freqtrade追踪，但不会在交易所上实际开仓。前向测试在实时进行，虽然耗时更长，但比回测更可靠地反映潜在表现。

可以通过在[配置](configuration.md#using-dry-run-mode)中将`dry_run`设置为true，开启模拟运行。

!!! 警告 "回测可能非常不准确"
    由于多种原因，回测结果可能与实际情况存在偏差。请查阅[回测假设](backtesting.md#assumptions-made-by-backtesting)及[常见策略错误](strategy-customization.md#common-mistakes-when-developing-strategies)部分。某些网站会展示和排名策略的极佳回测结果，但请勿盲目相信这些结果的实际可达性或真实性。

??? 提示 "有用的命令"
    Freqtrade内置两条帮助检测策略基础缺陷的命令：[lookahead-analysis](lookahead-analysis.md) 和 [recursive-analysis](recursive-analysis.md)。

### 评估回测与模拟的结果

每次回测后，务必进行模拟运行，核验证两者结果是否足够一致。

如果存在明显差异，应检查入场和出场信号是否在同一蜡烛上出现，并确保逻辑一致。然而，模拟和回测结果总会存在差异：

- 回测假设所有订单都能成交。而在模拟运行中，如果使用限价单或交易所没有对应量级的交易量，订单可能无法全部成交。
- 在蜡烛收盘后触发入场信号时，回测假设交易在下一根蜡烛的开盘价成交（除非你自定义了价格回调）。在模拟中，信号到交易开仓可能存在延迟。
  每当新蜡烛到来（如每5分钟），Freqtrade需要时间分析所有交易对的DataFrame。因此，频繁的延迟会影响实际执行。
- 由于模拟和回测的入场价格不同，盈利统计也会有所不同。因此，ROI、止损、追踪止损和退出回调的数值不必完全一致。
- 在模拟中，"滞后"越大，价格预测的随机性越高，实际表现越难以准确预估。确保你的电脑性能足以在合理时间内处理所有交易对的数据。Freqtrade会在日志中警告你可能的数据处理延迟。

## 控制或监控运行中的机器人

一旦你的机器人在模拟或实盘模式下运行，Freqtrade提供五种机制以控制或监控：

- **[FreqUI](freq-ui.md)**：最易上手的界面，可通过网页实时观察和操作机器人。
- **[Telegram](telegram-usage.md)**：在移动端使用，通过Telegram可接收提醒并控制部分功能。
- **[FTUI](https://github.com/freqtrade/ftui)**：基于终端（命令行）的界面，仅支持监控机器人状态。
- **[REST API](rest-api.md)**：面向开发者的接口，可以自定义开发工具与Freqtrade交互。
- **[Webhooks](webhook-config.md)**：可向其他服务（如Discord）推送信息。

### 日志

Freqtrade会生成详细的调试日志，帮助你理解系统运行情况。请熟悉可能在日志中看到的各种提示信息和错误信息。

## 最后思考

算法交易非常复杂，大部分公开策略在盈利表现上并不理想。这主要是因为让策略在多种场景下都稳定盈利，涉及大量时间和精力。

因此，单纯依赖公共策略和回测结果进行评估存在较大风险。幸好，Freqtrade提供了多种工具帮助你做出明智决策并确保尽职调查。

盈利的方法多种多样，没有“万能秘籍”可以一键改进一个表现不佳的策略。

Freqtrade是一个开源平台，拥有庞大的活跃社区——欢迎加入我们的[Discord频道](https://discord.gg/p7nuUNVfP7)，与他人交流策略经验！

请始终只投入你愿意承担损失的资金。

## 结语

在Freqtrade中开发策略涉及定义基于技术指标的入场和出场信号。按照上述结构和方法，你可以创建并测试属于自己的交易策略。

常见问题及解答请查阅我们的[常见问答](faq.md)。

继续深入学习，建议阅读更详细的[策略定制指南](strategy-customization.md)。