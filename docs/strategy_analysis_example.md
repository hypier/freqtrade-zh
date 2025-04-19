# 策略分析示例

调试策略可能非常耗时。Freqtrade 提供了一些辅助函数，用于可视化原始数据。
以下假设你使用的是 SampleStrategy，数据来自 Binance 的 5 分钟时间框架，并已下载到默认位置的数据目录中。
更多详细信息，请参考[文档](https://www.freqtrade.io/en/stable/data-download/)。

## 环境设置

### 将工作目录切换到仓库根目录

```python
import os
from pathlib import Path

# 切换目录
# 修改此单元格以确保输出显示正确的路径。
# 所有路径相对于在单元格输出中显示的项目根目录定义
project_root = "somedir/freqtrade"
i = 0
try:
    os.chdir(project_root)
    if not Path("LICENSE").is_file():
        i = 0
        while i < 4 and (not Path("LICENSE").is_file()):
            os.chdir(Path(Path.cwd(), "../"))
            i += 1
        project_root = Path.cwd()
except FileNotFoundError:
    print("请将项目根目录相对于当前目录进行定义。")
print(Path.cwd())
```

### 配置 Freqtrade 环境

```python
from freqtrade.configuration import Configuration

# 根据需要自定义以下参数。

# 初始化空的配置对象
config = Configuration.from_files([])
# 可选（推荐）使用现有配置文件
# config = Configuration.from_files(["user_data/config.json"])

# 定义一些常量
config["timeframe"] = "5m"
# 策略类名称
config["strategy"] = "SampleStrategy"
# 数据存放路径
data_location = config["datadir"]
# 要分析的交易对——这里只用一个交易对
pair = "BTC/USDT"
```


```python
# 使用上述设置的值加载数据
from freqtrade.data.history import load_pair_history
from freqtrade.enums import CandleType

candles = load_pair_history(
    datadir=data_location,
    timeframe=config["timeframe"],
    pair=pair,
    data_format="json",  # 确保根据你的数据更新此项
    candle_type=CandleType.SPOT,
)

# 确认加载成功
print(f"已加载 {len(candles)} 行数据，交易对：{pair}，位置：{data_location}")
candles.head()
```

## 加载并运行策略
* 每次修改策略文件后，请重新运行

```python
# 使用上述设置的值加载策略
from freqtrade.data.dataprovider import DataProvider
from freqtrade.resolvers import StrategyResolver

strategy = StrategyResolver.load_strategy(config)
strategy.dp = DataProvider(config, None, None)
strategy.ft_bot_start()

# 使用策略生成买卖信号
df = strategy.analyze_ticker(candles, {"pair": pair})
df.tail()
```

### 显示交易详情

* 注意，使用 `data.head()` 也可以，但大多数指标在数据框顶部有一些“启动”数据。
* 一些可能的问题：
    * 数据框末尾有NaN值的列
    * 在 `crossed*()` 函数中用到的列，单位完全不同
* 与完整回测的比较
    * `analyze_ticker()` 输出200个买入信号，并不一定意味着在回测期间会执行200笔交易。
    * 比如，只用条件 `df['rsi'] < 30` 作为买入条件会连续生成多个“买入”信号（直到 rsi 返回 > 29）。机器人只会在第一个信号（且交易槽“max_open_trades”仍有空位）时买入，或者在中间某个信号出现“槽位”空出时买入。

```python
# 输出分析结果
print(f"生成了 {df['enter_long'].sum()} 个入场信号")
data = df.set_index("date", drop=False)
data.tail()
```

## 在 Jupyter Notebook 中加载已有对象

以下单元假设你已通过 CLI 生成了数据。  
它们可以帮助你深入分析结果，否则由于信息过载，输出会非常难以理解。

### 加载回测结果到 pandas DataFrame

分析一个交易数据框（也用于后续绘图）

```python
from freqtrade.data.btanalysis import load_backtest_data, load_backtest_stats

# 如果 backtest_dir 指向一个目录，会自动加载最新的回测文件
backtest_dir = config["user_data_dir"] / "backtest_results"
# backtest_dir 也可以指向一个具体文件
# backtest_dir = (
#   config["user_data_dir"] / "backtest_results/backtest-result-2020-07-01_20-04-22.json"
# )
```

```python
# 可以通过以下命令获取完整的回测统计信息。
# 这包含了生成回测结果所用的所有信息。
stats = load_backtest_stats(backtest_dir)

strategy = "SampleStrategy"
# 所有统计信息都可以按策略查看，如果回测时使用了 `--strategy-list`，这里也会反映
# 示例用法：
print(stats["strategy"][strategy]["results_per_pair"])
# 获取此次回测用到的交易对列表
print(stats["strategy"][strategy]["pairlist"])
# 获取市场变化（所有交易对从开始到结束的平均变动）
print(stats["strategy"][strategy]["market_change"])
# 最大回撤（绝对值）
print(stats["strategy"][strategy]["max_drawdown_abs"])
# 最大回撤的起止时间
print(stats["strategy"][strategy]["drawdown_start"])
print(stats["strategy"][strategy]["drawdown_end"])

# 获取策略比较（仅在比较多个策略时相关）
print(stats["strategy_comparison"])
```

```python
# 以DataFrame形式加载回测交易数据
trades = load_backtest_data(backtest_dir)

# 显示每个交易对的成交次数
trades.groupby("pair")["exit_reason"].value_counts()
```

## 绘制每日利润/权益曲线

```python
# 绘制权益线（以第1天0点开始，逐日累计每日利润）

import pandas as pd
import plotly.express as px

from freqtrade.configuration import Configuration
from freqtrade.data.btanalysis import load_backtest_stats

# 假设策略名为 'SampleStrategy'
# config = Configuration.from_files(["user_data/config.json"])
# backtest_dir = config["user_data_dir"] / "backtest_results"

stats = load_backtest_stats(backtest_dir)
strategy_stats = stats["strategy"][strategy]

df = pd.DataFrame(columns=["dates", "equity"], data=strategy_stats["daily_profit"])
df["equity_daily"] = df["equity"].cumsum()

fig = px.line(df, x="dates", y="equity_daily")
fig.show()
```

### 将实盘交易结果加载到 pandas DataFrame

如果你已经进行了一些实盘交易，并希望分析你的表现

```python
from freqtrade.data.btanalysis import load_trades_from_db

# 从数据库加载交易
trades = load_trades_from_db("sqlite:///tradesv3.sqlite")

# 展示每个交易对的退出原因统计
trades.groupby("pair")["exit_reason"].value_counts()
```

## 分析已加载交易的平行交易情况
这对于在使用高 `max_open_trades` 设置的回测中，找到最佳 `max_open_trades` 参数非常有用。

`analyze_trade_parallelism()` 返回一个时间序列数据框，包含“open_trades”列，表示每个K线的未平仓交易笔数。

```python
from freqtrade.data.btanalysis import analyze_trade_parallelism

# 分析上述交易数据
parallel_trades = analyze_trade_parallelism(trades, "5m")

parallel_trades.plot()
```

## 绘制分析结果

Freqtrade 提供基于 plotly 的交互式绘图功能。

```python
from freqtrade.plot.plotting import generate_candlestick_graph

# 限制图表周期以确保 plotly 反应迅速

# 过滤某个交易对的数据
trades_red = trades.loc[trades["pair"] == pair]

data_red = data["2019-06-01":"2019-06-10"]
# 生成K线图
graph = generate_candlestick_graph(
    pair=pair,
    data=data_red,
    trades=trades_red,
    indicators1=["sma20", "ema50", "ema55"],
    indicators2=["rsi", "macd", "macdsignal", "macdhist"],
)
```

```python
# 以内联方式显示图表
# graph.show()

# 在新窗口显示图表
graph.show(renderer="browser")
```

## 绘制平均每笔交易利润的分布图

```python
import plotly.figure_factory as ff

hist_data = [trades.profit_ratio]
group_labels = ["profit_ratio"]  # 数据集名称

fig = ff.create_distplot(hist_data, group_labels, bin_size=0.01)
fig.show()
```

如果你有任何建议或想分享如何更好地分析数据的想法，欢迎提交 issue 或 Pull Request。