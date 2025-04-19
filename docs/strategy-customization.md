# 策略自定义

本页面说明如何自定义策略，添加新指标以及设置交易规则。

如果还没有了解过，请先熟悉以下内容：

- [Freqtrade策略入门101](strategy-101.md)，提供策略开发的快速入门指南
- [Freqtrade机器人基础](bot-basics.md)，介绍机器人总体的运行机制

## 开发你自己的策略

机器人包含一个默认策略文件。

此外，在[策略仓库](https://github.com/freqtrade/freqtrade-strategies)中还提供了多个其他策略。

不过，你很可能会有自己的策略想法。

本指南旨在帮助你将你的想法转化为可用的策略。

### 生成策略模板

你可以使用以下命令开始：

```bash
freqtrade new-strategy --strategy AwesomeStrategy
```

此命令将基于模板创建一个名为`AwesomeStrategy`的新策略，其文件位置为`user_data/strategies/AwesomeStrategy.py`。

!!! 注意
    策略的*名称*和文件名是不同的。在大多数命令中，Freqtrade使用的是策略的*名称*，而不是文件名。

!!! 注意
    `new-strategy`命令生成的示例模板在初始状态下不会盈利，需自行优化。

??? 提示 "不同模板级别"
    `freqtrade new-strategy`还有一个参数`--template`，可控制生成策略中的预设信息量。使用`--template minimal`可以得到空白策略（无指标示例），而`--template advanced`则会生成更复杂的功能模板。

### 策略组成结构

一个策略文件包含所有定义逻辑所需的信息：

- Candle数据（OHLCV格式）
- 指标
- 入场逻辑
  - 触发信号
- 出场逻辑
  - 触发信号
  - 最小ROI
  - 回调函数（“自定义函数”）
- 止损
  - 固定/绝对
  - 跟踪（Trailing）
  - 回调函数（“自定义函数”）
- 价格策略 [可选]
- 仓位调整 [可选]

策略示例配有一个叫`SampleStrategy`的样例策略，文件在`user_data/strategies/sample_strategy.py`。可以通过参数`--strategy SampleStrategy`测试，但请注意使用的是策略类名，而非文件名。

此外，策略类中还包含一个`INTERFACE_VERSION`属性，定义了策略接口的版本。当前版本为3，若未显式设置，默认为版本3，未来版本可能会要求更新。

不同版本的策略可能设置的`INTERFACE_VERSION`不同（如2版），需要随着未来版本的升级而更新为v3。

启动机器人（测试或正式交易）可用以下命令：

```bash
freqtrade trade --strategy AwesomeStrategy
```

### 机器人运行模式

Freqtrade支持五种主要模式：

- 回测（backtesting）
- 超参数优化（hyperopt）
- 数字“模拟”测试（dry，即“前向测试”）
- 实时交易（live）
- FreqAI（此处不涉及）

有关如何设置机器人为dry或live模式，请查阅[配置文档](configuration.md)。

**在测试时，必须始终使用dry模式，这样可以在不涉入资金的情况下模拟策略表现。**

## 深入学习

**以下内容，以 [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py) 文件为参考。**

!!! 注意 "策略与回测"
    为避免回测和dry/live模式之间出现问题或差异，提醒如下：
    - 在回测中，整个时间范围会一次性传递给`populate_*()`方法。
    - 因此，应优先使用向量化操作（对整个DataFrame操作，而非循环）
    - 避免使用索引引用（如`df.iloc[-1]`），应用`df.shift()`函数获取前一根K线，以避免未来数据泄露。

!!! 警告 "警惕未来数据"
    由于回测会一次性传入完整时间段数据，策略开发时必须确保不利用未来数据（“提前预知”）。
    常见违规模式可在[常见错误](#common-mistakes-when-developing-strategies)部分查阅。

??? 提示 "提前预判和递归分析"
    Freqtrade提供两个有用的命令帮助识别常见的“提前预判”（利用未来数据）和“递归偏差”（指标值波动）问题：
    - 运行干/正式模式前，务必首先执行这些命令。
    - 可查阅[提前预判分析](lookahead-analysis.md)和[递归分析](recursive-analysis.md)文档。

### DataFrame基础

Freqtrade使用[pandas](https://pandas.pydata.org/)来存储/提供蜡烛图（OHLCV）数据。
pandas是处理大量表格数据的优秀库。

每一行对应一根蜡烛，最新完成的蜡烛总在DataFrame的最后（按时间排序）。

例如，用pandas的`head()`函数查看前几行会得到：

```output
> dataframe.head()
                       date      open      high       low     close     volume
0 2021-11-09 23:25:00+00:00  67279.67  67321.84  67255.01  67300.97   44.62253
1 2021-11-09 23:30:00+00:00  67300.97  67301.34  67183.03  67187.01   61.38076
2 2021-11-09 23:35:00+00:00  67187.02  67187.02  67031.93  67123.81  113.42728
3 2021-11-09 23:40:00+00:00  67123.80  67222.40  67080.33  67160.48   78.96008
4 2021-11-09 23:45:00+00:00  67160.48  67160.48  66901.26  66943.37  111.39292
```

DataFrame是个表格，列不只单个值，而是一列列的数据。因此，像下面的Python比较表达式是无效的：

```python
    if dataframe['rsi'] > 30:
        dataframe['enter_long'] = 1
```

会抛出`The truth value of a Series is ambiguous [...]`错误。

正确的做法是用pandas的写法，进行向量化操作：

```python
    dataframe.loc[
        (dataframe['rsi'] > 30)
    , 'enter_long'] = 1
```

这样在整个DataFrame范围内，"enter_long"列在RSI>30时都为1。

利用pandas的向量化计算，可极大提高指标的运算速度。建议避免用循环，改用向量化方法。

??? 提示 "信号 vs 交易"
- 信号（Signals）由指标在蜡烛收盘时产生，表示入场意向。
- 交易（Trades）为实际订单（在实盘中在交易所执行），一个趋势可能在下一根K线开盘时立即成交。

!!! 警告 "交易订单假设"
    在回测中，信号在蜡烛收盘时产生，订单在下一根蜡烛开盘时立即执行。
    
    在dry和实盘模式，可能会有延迟，因为所有配对的DataFrame都需要分析，且每个订单都会处理。
    
    这意味着在dry/live模式下，应尽量降低计算延迟，减少同时处理的配对数，并确保CPU性能较好。

#### 为什么不能看到“实时”蜡烛数据？

Freqtrade不会存储不完整（未结束）的蜡烛。

不使用未来数据（“重绘”）做决策，避免策略“提前预判”。一些平台可能允许重绘，但Freqtrade不会。

只有已完全完成的蜡烛数据，才会存入DataFrame。

### 自定义指标

入场和出场信号都依赖指标。你可以在策略文件中的`populate_indicators()`方法里添加更多指标。

只应添加在`populate_entry_trend()`、`populate_exit_trend()`或其他指标计算中用到的指标，否则可能影响性能。

此外，必须确保这三个函数返回的DataFrame都不删除或修改`"open"`、`"high"`、`"low"`、`"close"`、`"volume"`列，否则可能导致后续计算出错。

示例：

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    在DataFrame中添加多种技术指标
    
    性能提示：为追求最好性能，应减少指标个数，仅使用策略中实际用到的指标。
    只启用你在策略或超参数优化配置中用到的指标，否则会浪费内存和CPU。
    :param dataframe: 来自交易所的数据
    :param metadata: 其他信息，如当前交易对
    :return: 包含所有必须指标的DataFrame
    """
    dataframe['sar'] = ta.SAR(dataframe)
    dataframe['adx'] = ta.ADX(dataframe)
    stoch = ta.STOCHF(dataframe)
    dataframe['fastd'] = stoch['fastd']
    dataframe['fastk'] = stoch['fastk']
    dataframe['bb_lower'] = ta.BBANDS(dataframe, nbdevup=2, nbdevdn=2)['lowerband']
    dataframe['sma'] = ta.SMA(dataframe, timeperiod=40)
    dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)
    dataframe['mfi'] = ta.MFI(dataframe)
    dataframe['rsi'] = ta.RSI(dataframe)
    dataframe['ema5'] = ta.EMA(dataframe, timeperiod=5)
    dataframe['ema10'] = ta.EMA(dataframe, timeperiod=10)
    dataframe['ema50'] = ta.EMA(dataframe, timeperiod=50)
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
    dataframe['ao'] = awesome_oscillator(dataframe)
    macd = ta.MACD(dataframe)
    dataframe['macd'] = macd['macd']
    dataframe['macdsignal'] = macd['macdsignal']
    dataframe['macdhist'] = macd['macdhist']
    hilbert = ta.HT_SINE(dataframe)
    dataframe['htsine'] = hilbert['sine']
    dataframe['htleadsine'] = hilbert['leadsine']
    dataframe['plus_dm'] = ta.PLUS_DM(dataframe)
    dataframe['plus_di'] = ta.PLUS_DI(dataframe)
    dataframe['minus_dm'] = ta.MINUS_DM(dataframe)
    dataframe['minus_di'] = ta.MINUS_DI(dataframe)

    # 记得始终返回DataFrame
    return dataframe
```

!!! 注意 "需要更多指标示例？"
    查看 [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py)，
    其中已包含多个指标示例，你可以根据需要取消注释启用。

#### 常用指标库

Freqtrade内置支持以下技术指标库：

- [ta-lib](https://ta-lib.github.io/ta-lib-python/)
- [pandas-ta](https://twopirllc.github.io/pandas-ta/)
- [technical](https://technical.freqtrade.io)

你也可以根据需要安装其他技术库或自行编写/发明指标。

### 策略启动期

某些指标在策略开始阶段（启动期）可能计算不稳定，导致值为NaN或计算错误。这会引起不一致，因为Freqtrade不知道不稳定期的长度，使用指标的值。

为此，可设置`startup_candle_count`属性，指明策略所需的最大蜡烛数（即需要多少蜡烛才能得到稳定指标值）。

如果在包含高时间框架与辅助配对的场景中，`startup_candle_count`通常不变，值为所有时间段中最长的那个。

你可以用[递归分析](recursive-analysis.md)检验并找到合适的`startup_candle_count`。当递归分析显示偏差为0%时，说明蜡烛数据已足够。

例如，以下策略示例中，`startup_candle_count`应设为400（即`startup_candle_count = 400`），因为计算`ema100`所需的最少蜡烛数即为400根。

```python
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
```

明确策略所需的蜡烛数，有助在回测和超参数优化时提前准备数据。

!!! 警告 "多次调用获取OHLCV数据"
    如果收到警告“WARNING - Using 3 calls to get OHLCV...” ，建议你重新考虑是否真需要这么多历史蜡烛的数据。
    频繁调用会导致交易机器人变慢甚至被交易所限制。

!!! 警告 "`startup_candle_count`应低于`ohlcv_candle_limit * 5`"
    这是因为在dry或实盘交易中，最多只会提供`ohlcv_candle_limit * 5`根蜡烛。

#### 示例

用示例策略对2019年1月的5分钟蜡烛进行回测，假设`startup_candle_count=400`：

```bash
freqtrade backtesting --timerange 20190101-20190201 --timeframe 5m
```

回测会在`2018-12-30 11:40:00`左右开始加载数据（考虑`400`蜡烛的历史长度）。如果数据足够，则指标会在此时间范围内计算，且在2019-01-01 00:00之前的“非稳定”启动期会被剔除。

!!! 注意 "启动期蜡烛数据不可用"
    如果无法获取到策略所需的启动期蜡烛数据，则会自动调整时间范围，从实际起点开始。

比如，回测可能从`2019-01-02 09:20:00`开始。

### 入场信号规则

修改`populate_entry_trend()`函数，实现自己的入场策略。

务必要始终返回未被删除或修改`"open"`、`"high"`、`"low"`、`"close"`、`"volume"`列的DataFrame，否则将导致出错。

此方法会新增`"enter_long"`（多头）和`"enter_short"`（空头）列，值为`1`代表入场，`0`为“无动作”。`enter_long`为必填列（即使只做空也要设置），而`"enter_tag"`列为标识标签，用于调试。

示例（`user_data/strategies/sample_strategy.py`）：

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    根据技术指标，设置入场信号
    :param dataframe: 包含指标的DataFrame
    :param metadata: 其他信息，如交易对
    :return: 包含买入信号的DataFrame
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # RSI 上穿30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # 保护措施
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # 趋势确认
            (dataframe['volume'] > 0)  # 确保有成交量
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

??? 提示 "空头交易入场"
- 您可以设置`enter_short`列实现空头入场（对应`enter_long`）。
- `enter_tag`列保持一致。
- 需确认交易所支持空头，并正确设置策略中的`can_short`。

示例（支持多空）：

```python
# 允许多空交易
can_short = True

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # RSI 上穿30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # 保护
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # 趋势确认
            (dataframe['volume'] > 0)  # 确保有成交量
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    dataframe.loc[
        (
            (qtpylib.crossed_below(dataframe['rsi'], 70)) &  # RSI下穿70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # 保护
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # 趋势确认
            (dataframe['volume'] > 0)  # 确保有成交量
        ),
        ['enter_short', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

??? 注意 "买入需要有卖单，确保成交量 > 0"
- 交易时，买卖都要求`volume > 0`，以确保在无成交量的空档期不发起买卖。

### 出场信号规则

修改`populate_exit_trend()`函数，定义你的出场逻辑。

可以通过在配置或策略中设置`use_exit_signal = False`来禁用出场信号。

`use_exit_signal`不影响信号碰撞规则（后文详述），即碰到冲突信号时仍会阻止入场。

务必返回未删减“open”、“high”、“low”、“close”、“volume”列，以免产生异常。

此方法会新增`"exit_long"`（多头平仓）和`"exit_short"`（空头平仓）列，值为`1`表示执行出场，否则为`0`。

`"exit_tag"`列为标签，用于后续调试。

示例（`user_data/strategies/sample_strategy.py`）：

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    根据技术指标，设置出场信号
    :param dataframe: 包含指标的DataFrame
    :param metadata: 其他信息，如交易对
    :return: 包含出场信号的DataFrame
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # RSI上穿70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # 保护
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # 趋势确认
            (dataframe['volume'] > 0)  # 确保有成交量
        ),
        ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
    return dataframe
```

??? 提示 "空头平仓"
- 空头平仓信号可以通过设置`exit_short`实现（对应`exit_long`）。
- `exit_tag`标签保持一致。
- 确认交易所支持空头，设置`can_short`。

示例（支持多空）：

```python
# 允许多空交易
can_short = True

def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # RSI上穿70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # 保护
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # 趋势确认
            (dataframe['volume'] > 0)  # 保证成交量
        ),
        ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
    dataframe.loc[
        (
            (qtpylib.crossed_below(dataframe['rsi'], 30)) &  # RSI下穿30
            (dataframe['tema'] < dataframe['bb_middleband']) &  # 保护
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # 趋势确认
            (dataframe['volume'] > 0)  # 保证成交量
        ),
        ['exit_short', 'exit_tag']] = (1, 'rsi_too_low')
    return dataframe
```

### 最小ROI（Minimal ROI）

`minimal_roi`是一个字典，定义在非信号触发的情况下，交易应达到的最低收益率（ROI）。

格式：`{minutes_since_open: ROI百分比}`

示例：

```python
minimal_roi = {
    "40": 0.0,
    "30": 0.01,
    "20": 0.02,
    "0": 0.04
}
```

含义：  
- 40分钟内，无盈利时落袋为安（ROI<4%）  
- 30分钟后盈利≥1%  
- 20分钟后盈利≥2%  
- 任何时间占用盈利≥4%都会触发出场

此配置会考虑手续费。

#### 关闭ROI策略

设置为空字典即可禁用：

```python
minimal_roi = {}
```

#### 用时间（基于蜡烛数）设置ROI

可以用策略中的`timeframe`和蜡烛数转换成时间点：

```python
from freqtrade.exchange import timeframe_to_minutes

class AwesomeStrategy(IStrategy):

    timeframe = "1d"
    timeframe_mins = timeframe_to_minutes(timeframe)
    minimal_roi = {
        "0": 0.05,                      # 之前3根蜡烛（默认）
        str(timeframe_mins * 3): 0.02, # 3根蜡烛后达成2%
        str(timeframe_mins * 6): 0.01, # 6根蜡烛后达成1%
    }
```

!!! 提示 "未立即成交的订单"
- `minimal_roi`会以`trade.open_date`（订单开启时间）为基准。
- 适用于限价单未立即成交时的场景（配合`custom_entry_price()`使用），以及订单被调价后。

### 止损（Stoploss）

设置止损可以有效保护资金。

示例：设置10%的止损

```python
stoploss = -0.10
```

完整功能详情请查看[止损页](stoploss.md)。

### 时间周期（Timeframe）

代表策略使用的蜡烛周期。

常用有`"1m"`、`"5m"`、`"15m"`、`"1h"`等，但支持所有交易所支持的周期。

注意：不同时间周期的入场/出场信号效果不同。可以在策略文件中通过`self.timeframe`访问。

### 支持空头（Can short）

在期货市场启用空头策略，需设定`can_short = True`。

启用后，仅支持期货市场（现货市场中此设置无效）。若`enter_short`列中有值，未启用`can_short`时，空头信号将被忽略。

### 元数据字典（metadata）

`metadata`包含额外信息（在`populate_*`函数中可用）：

- `pair`：交易对，例如`XRP/BTC`（或`XRP/BTC:BTC`用于期货市场）。

勿自行修改此字典。可参考[存储信息](strategy-advanced.md#storing-information-persistent)。

--8<-- "includes/strategy-imports.md"

## 策略文件加载方式

默认情况下，Freqtrade会加载所有`user_data/strategies`目录下的`.py`文件中的策略。

假设你的策略名为`AwesomeStrategy`，文件名为`user_data/strategies/AwesomeStrategy.py`，可以用：

```bash
freqtrade trade --strategy AwesomeStrategy
```

启动时用策略类名（非文件名）。

还可以用`freqtrade list-strategies`列出所有能加载的策略（即在策略文件夹中的策略），并显示状态。

??? 提示 "自定义策略目录"
    可用参数`--strategy-path`指定其他路径，例如`--strategy-path user_data/otherPath`，作用于所有涉及策略的命令。

## 信息配对（Informative Pairs）

### 获取非交易用配对数据

某些策略需要观察更长周期（如日线）数据，以辅助决策。

这些配对的OHLCV数据，会在常规的白名单刷新时下载，且可通过`DataProvider`访问。

这些配对**不会**直接交易，除非它们被加入白名单或经过动态白名单（如`VolumePairlist`）筛选。

配对格式：`("pair", "timeframe")`，`pair`为交易对，`timeframe`为时间周期。

示例：

```python
def informative_pairs(self):
    return [("ETH/USDT", "5m"),
            ("BTC/TUSD", "15m"),
            ]
```

详细示例可参见[DataProvider部分](#complete-dataprovider-sample)。

!!! 警告
    因为这些配对会随白名单刷新，建议限制列表长度。支持的平台必须在交易所中存在且激活。建议优先使用重采样（resampling）到长周期，以减少请求频次。

??? 提示 "替代蜡烛类型"
    informative_pairs还支持第三个参数，明示蜡烛类型（`candletype`）。不同交易模式与交易所支持不同：
    - 现货配对不能用于期货配对，反之亦然。
    具体支持情况可参考交易所文档。

```python
def informative_pairs(self):
    return [
        ("ETH/USDT", "5m", ""),       # 使用默认蜡烛类型（建议）
        ("ETH/USDT", "5m", "spot"),   # 强制使用现货蜡烛（仅支持现货）
        ("BTC/TUSD", "15m", "futures"),  # 期货蜡烛（需设置trading_mode=futures）
        ("BTC/TUSD", "15m", "mark"),     # 结算（标记）蜡烛（仅期货）
    ]
```

---

### 装饰器`@informative()`（Informative Pairs定义）

用`@informative`装饰器，可快速定义辅助配对。被装饰的`populate_indicators_*`方法只会在自身环境中运行，不会访问其他配对的数据信息。

但是，所有“辅助数据帧”会在每次调用`populate_indicators()`时合并后传入。

!!! 注意
    若需在生成某个辅助配对时引用另一个辅助配对的数据，不要用`@informative()`，应手动定义配对列表。

在超参数优化（hyperopt）时，不支持通过`.value`访问参数属性，应使用`.range`。

??? 信息 "完整示例"
```python
def informative(
    timeframe: str,
    asset: str = "",
    fmt: str | Callable[[Any], str] | None = None,
    *,
    candle_type: CandleType | str | None = None,
    ffill: bool = True,
) -> Callable[[PopulateIndicators], PopulateIndicators]:
    """
    装饰器，用于定义数据提供的辅助指标函数。

    示例：

        @informative('1h')
        def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

    参数说明：
    - timeframe: 辅助周期，必须大于或等于策略周期
    - asset: 辅助交易资产（如BTC、BTC/USDT），可留空使用当前对
    - fmt: 格式字符串或格式化函数，用于列名（详见官方文档）
    - ffill: 合并后是否进行前向填充
    - candle_type: 指定蜡烛类型（空、mark、index、premiumIndex、funding_rate）
    """
```

!!! 示例 "快速定义辅助配对"
- 一般无需使用`merge_informative_pair()`的强大功能，可用装饰器快速定义。

```python
from datetime import datetime
from freqtrade.persistence import Trade
from freqtrade.strategy import IStrategy, informative

class AwesomeStrategy(IStrategy):

    @informative('30m')
    @informative('1h')
    def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    @informative('1h', 'BTC/{stake}')
    def populate_indicators_btc_stake(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        # 访问其他配对辅助数据
        dataframe['rsi_less'] = dataframe['rsi'] < dataframe['rsi_1h']
        return dataframe
```

??? 注意 "跨配对访问，建议用字符串格式化"
- 在调用其他配对的辅助数据时，用字符串格式化访问列名，确保策略可动态切换主/辅配对。

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    stake = self.config['stake_currency']
    dataframe.loc[
        (
            (dataframe[f'btc_{stake}_rsi_1h'] < 35)
            &
            (dataframe['volume'] > 0)
        ),
        ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')

    return dataframe
```

也可以用列名重命名`@informative()`的`fmt`参数，避免引入持仓货币。

```python
@informative('1h', 'BTC/{stake}', fmt='{base}_{column}_{timeframe}')
```

!!! 警告 "方法名重复"
- 被`@informative()`装饰的方法必须唯一，否则会覆盖已有方法，导致指标无法被引用。
- 例如拷贝粘贴多个同名的`populate_indicators_*`，会出现此问题。
- 应确保方法名唯一，避免覆盖。

### *merge_informative_pair()*

此函数可将辅助配对与主数据安全合并，避免未来数据泄露（lookahead bias）。

功能：
- 改名列以确保唯一
- 不引入未来信息
- 可以选择前向填充

完整示例参见[完整DataProvider示例](#complete-dataprovider-sample)。

合并后，辅助配对的所有列会自动更名（例子见下）：

!!! 示例 "列名重命名"
假设`inf_tf='1d'`，合并后列名：

```python
'date', 'open', 'high', 'low', 'close', 'rsi'                     # 来自原始DataFrame
'date_1d', 'open_1d', 'high_1d', 'low_1d', 'close_1d', 'rsi_1d'   # 来自辅助配对
```

??? 示例 "1h配对的列名"  
假设`inf_tf='1h'`，合并后：

```python
'date', 'open', 'high', 'low', 'close', 'rsi'
'date_1h', 'open_1h', 'high_1h', 'low_1h', 'close_1h', 'rsi_1h'
```

??? 示例 "自定义实现"  
可以自己写合并逻辑，例如：

```python
# 以1根蜡烛的偏移模拟时间对齐（防未来信息泄露）
import pandas as pd
minutes = timeframe_to_minutes(inf_tf)
# 时间偏移1个周期
informative['date_merge'] = informative["date"] + pd.to_timedelta(minutes, 'm')

# 更改列名，确保唯一
informative.columns = [f"{col}_{inf_tf}" for col in informative.columns]

# 合并DataFrame
# 先确保所有指标在此之前都已计算
dataframe = pd.merge(dataframe, informative, left_on='date', right_on=f'date_merge_{inf_tf}', how='left')
# 使用ffill保证每日的辅助指标值在每个时间点都可用
dataframe = dataframe.ffill()
```

??? 警告 "辅助时间帧小于主时间帧"
- 使用辅助配对的时间周期小于主周期，会失去辅助信息的作用。建议用较长周期。

## 附加数据（DataProvider）

策略可以访问`DataProvider`，用于获取更多数据。

所有方法返回`None`表示失败（无异常抛出）。

请根据当前运行状态选择用对应的方法（如回测、dry或实盘）。

!!! 警告 "超参数优化中的限制"
- DataProvider在超参数优化时可用，但只能在`populate_indicators()`中使用（仅在策略中），不能在`populate_entry_trend()`或`populate_exit_trend()`中用。

### 可用方法

- [`available_pairs`](#available_pairs)：已缓存配对列表（pair, timeframe）
- [`current_whitelist()`](#current_whitelist)：动态白名单（如VolumePairlist）
- [`get_pair_dataframe(pair, timeframe)`](#get_pair_dataframepair-timeframe)：返回历史或缓存的实时数据
- [`get_analyzed_dataframe(pair, timeframe)`](#get_analyzed_dataframepair-timeframe)：返回指标分析后数据和上次分析时间
- `historic_ohlcv(pair, timeframe)`：磁盘存储的历史数据
- `market(pair)`：配对市场信息（手续费、限制、精度等）
- `ohlcv(pair, timeframe)`：当前缓存的K线数据（DataFrame）
- [`orderbook(pair, maximum)`](#orderbookpair-maximum)：最新订单簿（含bids/asks）
- [`ticker(pair)`](#tickerpair)：实时行情
- `runmode`：运行模式（回测/实盘/优化等）

### 使用示例

#### *available_pairs*

```python
for pair, timeframe in self.dp.available_pairs:
    print(f"可用配对：{pair}, 周期：{timeframe}")
```

#### *current_whitelist()*

例如，开发策略用1天周期（日线）配合10个交易对，用5分钟线信号判定买卖。

因数据有限，不能直接用`5m`转成日线。可以用`current_whitelist()`获取动态白单中的配对集合。

```python
def informative_pairs(self):
    # 获取白名单中的全部配对
    pairs = self.dp.current_whitelist()
    # 以1日周期下载缓存
    informative_pairs = [(pair, '1d') for pair in pairs]
    return informative_pairs
```

!!! 提示 "配对绘图建议"
- `plot-dataframe`目前不支持用白名单，结果可能误导。
- Web界面显示也不支持动态配对。

#### *get_pair_dataframe(pair, timeframe)*

```python
# 获取第一个配置的辅助配对数据
inf_pair, inf_timeframe = self.informative_pairs()[0]
informative = self.dp.get_pair_dataframe(pair=inf_pair, timeframe=inf_timeframe)
```

!!! 警告 "回测中的特殊情况"
- 在回测中，`get_pair_dataframe()`会返回全时间段数据，但要确保不“预知未来”。
- 在回调中会返回部分时间段（直到当前蜡烛）数据。

#### *get_analyzed_dataframe(pair, timeframe)*

用以获取已分析后（指标/买卖点）数据，常用于判定操作。

```python
# 获取当前数据
dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=metadata['pair'], timeframe=self.timeframe)
```

!!! 注意 "无数据"
- 若配对未缓存，会返回空DataFrame。
- 可用`if dataframe.empty:`判断处理。

#### *orderbook(pair, maximum)*

```python
if self.dp.runmode.value in ('live', 'dry_run'):
    ob = self.dp.orderbook(metadata['pair'], 1)
    dataframe['best_bid'] = ob['bids'][0][0]
    dataframe['best_ask'] = ob['asks'][0][0]
```

结构示意：

```js
{
    'bids': [
        [价格, 数量],  //例如[30000, 0.5]
        ...
    ],
    'asks': [
        [价格, 数量],
        ...
    ],
}
```

用`ob['bids'][0][0]`得到最高买价，`ob['bids'][0][1]`对应买量。

!!! 警告 "回测中不可用"
- 订单簿是实时数据，回测时此函数返回的可能与实际历史不符。

#### *ticker(pair)*

```python
if self.dp.runmode.value in ('live', 'dry_run'):
    ticker = self.dp.ticker(metadata['pair'])
    dataframe['last_price'] = ticker['last']
    dataframe['volume24h'] = ticker['quoteVolume']
    dataframe['vwap'] = ticker['vwap']
```

- 返回值结构随交易所不同，可能没有`vwap`或`last`字段等。用前应验证。

!!! 警告 "回测中不适用"
- 此方法返回实时行情数据，回测无限制使用会导致每行数据都出现相同值。

### 发送通知（Notification）

用`.send_msg()`可以在策略中发自定义通知。

参数`always_send=True`可以强制每根K线都发一次。

```python
self.dp.send_msg(f"{metadata['pair']}火热中！")
# 强制每根都发
self.dp.send_msg(f"{metadata['pair']}火热中！", always_send=True)
```

- 仅在交易中（实盘/模拟）有效。

!!! 警告 "刷屏"
- `always_send=True`时，会持续推送，请谨慎使用，避免频繁消息扰民。

## 完整示例：DataProvider用法

```python
from freqtrade.strategy import IStrategy, merge_informative_pair
from pandas import DataFrame

class SampleStrategy(IStrategy):
    # 初始化配置...

    timeframe = '5m'

    def informative_pairs(self):
        # 获取当前白名单中的所有配对
        pairs = self.dp.current_whitelist()
        # 为每对指定日线周期，下载缓存
        informative_pairs = [(pair, '1d') for pair in pairs]
        # 可加入静态配对
        informative_pairs += [("ETH/USDT", "5m"),
                              ("BTC/TUSD", "15m"),
                            ]
        return informative_pairs

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 确保DataProvider可用
        if not self.dp:
            return dataframe
        inf_tf = '1d'
        # 获取日线配对数据
        informative = self.dp.get_pair_dataframe(pair=metadata['pair'], timeframe=inf_tf)
        # 计算日线RSI
        informative['rsi'] = ta.RSI(informative, timeperiod=14)
        # 合并辅助日线指标到本次数据
        dataframe = merge_informative_pair(dataframe, informative, self.timeframe, inf_tf, ffill=True)

        # 计算本周期RSI
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        # 其他指标...
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 设定入场条件
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 30)) &
                (dataframe['rsi_1d'] < 30) &
                (dataframe['volume'] > 0)
            ),
            ['enter_long', 'enter_tag']
        ] = (1, 'rsi_cross')

        return dataframe
```

***  

## 其他数据（钱包信息）

策略还可访问`wallets`对象，获取账户余额。

!!! 注意 "回测/超参优化"
- 在`populate_*()`中，`wallets`返回的是配置的全钱包状态。
- 在callbacks中，则是模拟的实际钱包。

示例：

```python
if self.wallets:
    free_eth = self.wallets.get_free('ETH')
    used_eth = self.wallets.get_used('ETH')
    total_eth = self.wallets.get_total('ETH')
```

### Wallets常用方法

- `get_free(asset)`：当前可用余额
- `get_used(asset)`：暂用余额（挂单）
- `get_total(asset)`：总余额（“可用+挂单”）

---

## 其他数据（交易记录）

可以在策略中查询已完成的交易历史。

在文件开头导入：

```python
from freqtrade.persistence import Trade
```

示例：查询昨日的交易

```python
trades = Trade.get_trades_proxy(pair=metadata['pair'],
                                open_date=datetime.now(timezone.utc) - timedelta(days=1),
                                is_open=False)
# 统计利润
curdayprofit = sum(trade.close_profit for trade in trades)
```

完整方法细节请查阅[Trade对象文档](trade-object.md)。

!!! 警告
- 在回测或超参优化中，`populate_*()`无法访问交易历史，返回空。

## 阻止某配对交易

Freqtrade会在当前蜡烛完成前锁定某配对，避免重复操作。

显示消息：`Pair <pair> is currently locked.`

### 在策略中锁定配对

可以调用`self.lock_pair(pair, until, [reason])`实现。参数`until`为未来时间点，`reason`为说明。

解锁用`self.unlock_pair(pair)`或`self.unlock_reason(reason)`。

检查锁定状态用`self.is_pair_locked(pair)`。

!!! 注意
- 锁定要以蜡烛结束时间为准（向上取整）。
- 例如，5分钟周期，锁到10:18意味着10:15-10:20的蜡烛结束后才解锁。

!!! 警告
- 在回测中不能用手动锁定。只支持策略内部设置的保护机制。

示例代码：在交易后检查盈亏，决定锁定

```python
from freqtrade.persistence import Trade
from datetime import timedelta, datetime, timezone

# 在策略的`populate_*()`中
if self.config['runmode'].value in ('live', 'dry_run'):
    trades = Trade.get_trades_proxy(
        pair=metadata['pair'], is_open=False,
        open_date=datetime.now(timezone.utc) - timedelta(days=2))
    profit_sum = sum(trade.close_profit for trade in trades)
    if profit_sum < 0:
        # 亏损，锁定2小时
        self.lock_pair(metadata['pair'], until=datetime.now(timezone.utc) + timedelta(hours=2))
```

### 打印主DataFrame

可以在`populate_*()`中打印调试信息：

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 条件
    dataframe.loc[ ..., ['enter_long', 'enter_tag']] = (1, 'signal')

    # 打印当前配对信息
    print(f"Processing {metadata['pair']}")

    # 打印最后几行
    print(dataframe.tail())

    return dataframe
```

也可以输出全部内容（不过行数多，建议慎用）：

```python
print(dataframe)
```

## 常见开发错误

### 回测时“提前看未来”

- 回测会一次分析完全部数据，因性能考虑，不能用未来数据“作弊”。
- 常见错误：
  - 用`shift(-1)`或类似操作，造成未来数据泄露。
  - 用`iloc[-1]`在`populate_*()`方法中，可能在不同阶段值不一致。
  - 用全列统计（如`mean()`）会涵盖未来数据，应改用滚动窗口。
  - 不用`.resample()`，应用`.resample(..., label='right')`。
  - 不能用`merge()`直接合长短时间框架，必须用辅助配对（[informative pairs](#informative-pairs)）方案。

- 建议：定期执行[提前预判分析](lookahead-analysis.md)和[递归分析](recursive-analysis.md)，帮助排查潜在问题。

### 信号冲突（Colliding signals）

- 同一时间，若`'enter_long'`和`'exit_long'`同时为1，机器人会忽略入场，避免瞬间进出。
- 规则：
  - 若`enter_long`被触发，`exit_long`和`enter_short`会被忽略。
  - 若`enter_short`被触发，`exit_short`和`enter_long`会被忽略。

这样确保策略在冲突时不会立即反复买卖。

## 未来策略建议

- 结合不同指标，尝试不同激励方案。
- 使用[策略仓库](https://github.com/freqtrade/freqtrade-strategies)中的示例作为参考。
- 先在纸上或模拟环境中验证策略表现，然后逐步实盘运行。

欢迎提交Pull Requests，贡献新策略！

## 后续步骤

你已经拥有了一个策略，下一步可以学习[回测使用方法](backtesting.md)。