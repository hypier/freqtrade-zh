# Hyperopt（超参数优化）

本页面介绍如何通过寻找最优参数来调优你的策略，这个过程称为超参数优化。机器人使用包含在 `scikit-optimize` 包中的算法来实现。  
搜索过程会耗尽你的所有CPU核数，让你的笔记本电脑发出如战斗机般的声音，并且仍然需要较长时间。

一般来说，最佳参数的搜索从一些随机组合开始（详见[下面](#reproducible-results)的详细说明），然后使用带有ML回归器算法（目前为ExtraTreesRegressor）的贝叶斯搜索，以快速在搜索超空间中找到能最小化[损失函数](#loss-functions)值的参数组合。

Hyperopt需要历史数据支持，就像回测一样（hyperopt会用不同参数多次运行回测）。  
如果你想了解如何获取感兴趣的交易对和交易所的数据，可以查看文档中的[数据下载](data-download.md)章节。

!!! 警告
    当只使用1个CPU核时，Hyperopt可能会崩溃，正如在[问题#1133](https://github.com/freqtrade/freqtrade/issues/1133)中发现的那样。

!!! 注意
    自2021.4版本起，你不再需要单独编写hyperopt类，而可以直接在策略中配置参数。  
    旧的方法一直支持到2021.8版本，但在2021.9被移除。

## 安装hyperopt依赖

由于Hyperopt依赖与运行机器人本身无关，且依赖较大且在某些平台（如树莓派）上难以构建，因此默认未安装。  
在你运行hyperopt之前，需要安装对应的依赖，具体方法如下。

!!! 注意
    由于Hyperopt是资源密集型的操作，不建议也不支持在树莓派上运行。

### Docker

包含hyperopt依赖的docker镜像，无需额外操作。

### 简便安装脚本（setup.sh）/手动安装

```bash
source .venv/bin/activate
pip install -r requirements-hyperopt.txt
```

## Hyperopt命令参考

--8<-- "commands/hyperopt.md"

### Hyperopt基本检查列表

列出hyperopt所有任务／可能性清单

根据你想优化的空间（search space），以下只需部分功能：

* 定义参数为 `space='buy'` — 用于入场信号优化
* 定义参数为 `space='sell'` — 用于离场信号优化

!!! 注意
    `populate_indicators`需要创建所有可能用到的指标，否则hyperopt将无法正常工作。

很少情况下，可能还需要创建一个[嵌套类](advanced-hyperopt.md#overriding-pre-defined-spaces)，名为 `HyperOpt` 并实现：

* `roi_space` — 用于自定义ROI优化（如果你需要的ROI参数范围不同于默认）
* `generate_roi_table` — 用于自定义ROI表（如果你需要调整ROI值范围或ROI表的步数，默认为4步）
* `stoploss_space` — 用于自定义止损参数（范围不同于默认时）
* `trailing_space` — 用于自定义后续跟踪止损参数范围
* `max_open_trades_space` — 用于自定义最大开启交易数（范围不同于默认）

!!! 提示 "快速优化ROI、止损和跟踪止损"
    你可以无需更改策略内容，快速优化`roi`、`stoploss`和`trailing`空间。

    ```bash
    # 你已有一个正常工作的策略
    freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --spaces roi stoploss trailing --strategy MyWorkingStrategy --config config.json -e 100
    ```

### Hyperopt执行逻辑

Hyperopt首先会将你的数据加载到内存，然后对于每一个交易对调用一次`populate_indicators()`以生成所有指标，除非指定了`--analyze-per-epoch`。

之后，Hyperopt会启动多个进程（可通过`-j <n>`指定核数），不断进行回测，改变定义在 `--spaces` 中的参数。

每次参数集变化后，freqtrade会先运行`populate_entry_trend()`，再运行`populate_exit_trend()`，最后进行常规的回测，模拟交易。

回测结束后，结果会输入到[损失函数](#loss-functions)，用以评估此结果优劣。  
基于损失函数的结果，hyperopt会决定下一轮回测时要尝试的参数组合。

### 配置你的守护条件（Guards）和触发器（Triggers）

在策略文件中，需要更改两个地方来添加新的买入hyperopt：

* 在类层面定义参数，hyperopt会优化这些参数。
* 在`populate_entry_trend()`中，使用定义的参数值替代硬编码的常量。

你会遇到两种指标类型：1. `guards`（守护条件）和2. `triggers`（触发器）。

1. Guards是条件，比如"永不买入如果ADX<10"或"当前价格高于EMA10"。
2. Triggers是真正触发买入的条件，例如"EMA5上穿EMA10"或"收盘价触碰布林带下轨"。

!!! 提示 "Guards和Triggers"
    从技术角度，Guards和Triggers没有本质区别。  
    但为了避免混淆，本文会特别区分，以强调这些信号不应“黏着”在一起。  
    持续的信号意味着它们在多根蜡烛上都保持激活，可能导致迟进入信号（信号快消失时入场），成功概率较低。

Hyper-optimization在每一轮（epoch）会为每个trigger选一个trigger并可能多个guards。

#### 离场信号优化

类似入场信号，离场信号也可以优化，将相关设置放入以下位置：

* 在类层面定义参数，hyperopt会优化这些参数，参数名可以以`sell_*`开头，或显式定义`space='sell'`。
* 在`populate_exit_trend()`中，使用参数值替代硬编码。

规则和配置与买入信号类似。

## 解密一个谜题

假设你很犹豫：是否应该用MACD金叉还是布林带下轨触发多单入场？  
你还在考虑，是用RSI还是ADX辅助决策。  
如果选择使用RSI或ADX，应该用哪个值？  

那么，让我们用超参数优化来破解这个谜。

### 定义要用的指标

首先，计算你的策略将用到的所有指标。

```python
class MyAwesomeStrategy(IStrategy):

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        生成策略用到的所有指标
        """
        dataframe['adx'] = ta.ADX(dataframe)
        dataframe['rsi'] = ta.RSI(dataframe)
        macd = ta.MACD(dataframe)
        dataframe['macd'] = macd['macd']
        dataframe['macdsignal'] = macd['macdsignal']
        dataframe['macdhist'] = macd['macdhist']

        bollinger = ta.BBANDS(dataframe, timeperiod=20, nbdevup=2.0, nbdevdn=2.0)
        dataframe['bb_lowerband'] = bollinger['lowerband']
        dataframe['bb_middleband'] = bollinger['middleband']
        dataframe['bb_upperband'] = bollinger['upperband']
        return dataframe
```

### Hyperopt可调参数定义

接下来，定义可调整的超参数：

```python
class MyAwesomeStrategy(IStrategy):
    buy_adx = DecimalParameter(20, 40, decimals=1, default=30.1, space="buy")
    buy_rsi = IntParameter(20, 40, default=30, space="buy")
    buy_adx_enabled = BooleanParameter(default=True, space="buy")
    buy_rsi_enabled = CategoricalParameter([True, False], default=False, space="buy")
    buy_trigger = CategoricalParameter(["bb_lower", "macd_cross_signal"], default="bb_lower", space="buy")
```

以上定义表示：我有五个参数，希望随机组合以找到最佳配置。  
`buy_rsi`是一个整数参数，测试区间为20到40，空间大小为20。  
`buy_adx`是小数参数，评价范围为20到40，保留一位小数，组合数为200（20*10）。  
后面有三个类别变量，前两个为`True`或`False`，用来启用或禁用ADX和RSI守护条件。  
最后一个叫`trigger`，用以选择买入触发条件。

!!! 提示 "参数空间分配"
    参数要么命名为`buy_*`或`sell_*`，要么包含`space='buy'`|`space='sell'`，才能正确归属到空间内。  
    如果没有为某个空间定义参数，会出现找不到空间的错误。  
    不明确空间的参数（例如：`adx_period = IntParameter(4, 24, default=14)`，既无明确也无隐式空间）会被忽略。

用这些值写出买入策略示例：

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    conditions = []
    # 守护条件和趋势
    if self.buy_adx_enabled.value:
        conditions.append(dataframe['adx'] > self.buy_adx.value)
    if self.buy_rsi_enabled.value:
        conditions.append(dataframe['rsi'] < self.buy_rsi.value)

    # 触发条件
    if self.buy_trigger.value == 'bb_lower':
        conditions.append(dataframe['close'] < dataframe['bb_lowerband'])
    if self.buy_trigger.value == 'macd_cross_signal':
        conditions.append(qtpylib.crossed_above(
            dataframe['macd'], dataframe['macdsignal']
        ))

    # 确保成交量不为0
    conditions.append(dataframe['volume'] > 0)

    if conditions:
        dataframe.loc[
            reduce(lambda x, y: x & y, conditions),
            'enter_long'
        ] = 1

    return dataframe
```

Hyperopt将会多次（多轮`epochs`）调用`populate_entry_trend()`，每次用不同参数组合。  
它会用历史数据模拟买入信号，最后根据[损失函数](#loss-functions)给出最优参数。

!!! 注意
    上述设置假设指标`adx`、`rsi`和布林带在指标集里已被计算出来。  
    如果你想测试一个未在`populate_indicators()`中用到的指标，记得将它加入对应的指标生成函数。

## 参数类型

参数主要有四种，每种适合不同场景。

* `IntParameter` — 用于定义整型参数，范围有限。
* `DecimalParameter` — 用于定义浮点数参数，带有限定的小数位（默认3），在大部分情况下优于`RealParameter`。
* `RealParameter` — 浮点参数，无范围限制，极少用到，因其搜索空间几乎无限。
* `CategoricalParameter` — 预定义的类别选择。
* `BooleanParameter` — 简写，等同于 `CategoricalParameter([True, False])`，适合启用/禁用参数。

### 参数选项

有两个参数选项可以帮助你快速测试不同想法：

* `optimize` — 设置为`False`时，不会被纳入优化（默认：True）。
* `load` — 设置为`False`时，不会用之前hyperopt的结果作为起点（默认：True），而会使用参数的默认值。

!!! 提示 "设置`load=False`对回测的影响"
    如果设置`load=False`，回测时会使用参数的默认值，而非hyperopt找到的最优值，需注意此差异。

!!! 警告
    超参数不能用于`populate_indicators`，因为hyperopt不会每轮都重新计算指标，指标值会固定。

## 优化指标参数

假设你有一个简单策略：EMA交叉（两个移动平均线交叉），想找到最优参数。

默认假设止损为5%，盈利目标为10%（`minimal_roi` = 10%，意味着利润达到10%时卖出）。

```python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    minimal_roi = {
        "0":  0.10
    }
    # 定义参数空间
    buy_ema_short = IntParameter(3, 50, default=5)
    buy_ema_long = IntParameter(15, 200, default=50)

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """生成策略用到的所有指标"""
        # 计算所有短EMA
        for val in self.buy_ema_short.range:
            dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)
        # 计算所有长EMA
        for val in self.buy_ema_long.range:
            dataframe[f'ema_long_{val}'] = ta.EMA(dataframe, timeperiod=val)
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        conditions.append(qtpylib.crossed_above(
            dataframe[f'ema_short_{self.buy_ema_short.value}'], dataframe[f'ema_long_{self.buy_ema_long.value}']
        ))
        # 量必须大于0
        conditions.append(dataframe['volume'] > 0)
        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'enter_long'
            ] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        conditions.append(qtpylib.crossed_above(
            dataframe[f'ema_long_{self.buy_ema_long.value}'], dataframe[f'ema_short_{self.buy_ema_short.value}']
        ))
        # 量必须大于0
        conditions.append(dataframe['volume'] > 0)
        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'exit_long'
            ] = 1
        return dataframe
```

分析如下：  
`self.buy_ema_short.range`返回一个对象，包含从参数范围最低到最高的所有值。  
在这个例子（`IntParameter(3, 50, default=5)`）中，循环会覆盖所有从3到50的整数（`[3, 4, 5, ..., 49, 50]`）。  
利用这个特性，hyperopt会自动生成多达48个新列（如`['buy_ema_3', 'buy_ema_4', ..., 'buy_ema_50']`）。

hyperopt会用选中的值，来生成买入和卖出信号。

虽然此方法可能太简单，无法确保持续获利，但它作为示例说明了如何调优指标参数的思路。

!!! 注意
    `self.buy_ema_short.range`在hyperopt和其他模式下表现不同。在hyperopt中，它会生成多列（比如48列）；  
    而在回测或实盘模式下，只会使用选中的某个值对应的列或参数。所以避免在实际运行中用显式的列名或非对应的值。

!!! 注意
    `range`属性同时可以用于`DecimalParameter`和`CategoricalParameter`。  
    `RealParameter`由于搜索空间无限，因此没有`range`属性。

??? 提示 "性能优化建议"
    在正常hyperopt过程中，指标会被计算一次，传递给每轮（epoch），会随核心数增加而消耗更多内存（RAM）。出于性能考虑，有两种减小RAM占用的方案：

    * 将`ema_short`和`ema_long`的计算从`populate_indicators()`移动到`populate_entry_trend()`。因为每轮都会调用`populate_entry_trend()`，不再需要`.range`。
    * hyperopt提供`--analyze-per-epoch`参数，会把`populate_indicators()`的执行移到每轮时，只计算单一值，不用`.range`，节省内存。

    这些优化虽然会减少RAM占用，但会增加CPU负担。你可以权衡，避免大规模超参数调优导致OOM。

    无论是否使用`.range`或上述方案，建议尽量减小搜索空间，以提高效率。

## 优化保护措施

freqtrade也可以对保护措施进行优化。方法由你自行定义，以下仅作示意。

策略只需定义一个`protections`属性，返回保护配置列表。

```python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    # 参数空间定义
    cooldown_lookback = IntParameter(2, 48, default=5, space="protection", optimize=True)
    stop_duration = IntParameter(12, 200, default=5, space="protection", optimize=True)
    use_stop_protection = BooleanParameter(default=True, space="protection", optimize=True)

    @property
    def protections(self):
        prot = []

        prot.append({
            "method": "CooldownPeriod",
            "stop_duration_candles": self.cooldown_lookback.value
        })
        if self.use_stop_protection.value:
            prot.append({
                "method": "StoplossGuard",
                "lookback_period_candles": 24 * 3,
                "trade_limit": 4,
                "stop_duration_candles": self.stop_duration.value,
                "only_per_pair": False
            })

        return prot

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # ...
```

你可以用如下命令启动hyperopt：  
`freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --strategy MyAwesomeStrategy --spaces protection`

!!! 注意
    `protection`空间不是默认空间，只在参数hyperopt界面中可用，不是旧版hyperopt结构（需要单独文件）。  
    选择包含`protection`空间时，freqtrade会自动开启`--enable-protections`。

!!! 警告
    如果在配置中直接定义`protections`为属性，策略中的保护配置会被覆盖，建议不要在配置文件中定义。

### 迁移之前的`protections`配置

旧版本配置示例：

```python
class MyAwesomeStrategy(IStrategy):
    protections = [
        {
            "method": "CooldownPeriod",
            "stop_duration_candles": 4
        }
    ]
```

迁移后：

```python
class MyAwesomeStrategy(IStrategy):
    
    @property
    def protections(self):
        return [
            {
                "method": "CooldownPeriod",
                "stop_duration_candles": 4
            }
        ]
```

有意向的参数也可以改为参数（参数优化）。

### 优化 `max_entry_position_adjustment`

虽然没有专用空间，但可以用属性方式支持hyperopt。

```python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'

    # 参数空间定义
    max_epa = CategoricalParameter([-1, 0, 1, 3, 5, 10], default=1, space="buy", optimize=True)

    @property
    def max_entry_position_adjustment(self):
        return self.max_epa.value
```

??? 提示 "用`IntParameter`"
```python
max_epa = IntParameter(-1, 10, default=1, space="buy", optimize=True)

@property
def max_entry_position_adjustment(self):
    return int(self.max_epa.value)
```

## 损失函数（Loss-functions）

每次超参数调优都需要一个目标。通常用损失函数（有时称目标函数），在目标函数中，越优越值越低，差越差越高。

必须用`--hyperopt-loss <类名>`参数或配置`"hyperopt_loss"`键来指定。  
类文件需放在`user_data/hyperopts/`目录下。

当前内置支持以下损失函数：

* `ShortTradeDurHyperOptLoss` —（默认的旧频率调优损失）针对短交易时间和避免亏损  
* `OnlyProfitHyperOptLoss` — 只考虑利润
* `SharpeHyperOptLoss` — 根据交易收益的夏普比率（收益/标准差）优化
* `SharpeHyperOptLossDaily` — 以每日收益计算夏普比率
* `SortinoHyperOptLoss` — 根据下行标准差优化Sortino比率
* `SortinoHyperOptLossDaily` — 按日收益和下行标准差
* `MaxDrawDownHyperOptLoss` — 最大绝对回撤
* `MaxDrawDownRelativeHyperOptLoss` — 绝对回撤与相对回撤的结合
* `MaxDrawDownPerPairHyperOptLoss` — 每对的盈亏比，取最差的为目标，强制优化所有对
* `CalmarHyperOptLoss` — 以最大回撤为基础的Calmar比率
* `ProfitDrawDownHyperOptLoss` — 利润与回撤的权衡，参数`DRAWDOWN_MULT`可调节严苛程度
* `MultiMetricHyperOptLoss` — 多指标平衡优化，包括利润、回撤、盈利因子、期望值和赢率，还会惩罚交易较少的策略

自定义损失函数的技术细节请参见[高级Hyperopt](advanced-hyperopt.md)。

## 执行Hyperopt

更新配置后，可以运行hyperopt。  
由于hyperopt会尝试大量参数组合，耗时较长。建议使用`screen`或`tmux`避免中途断线。

```bash
freqtrade hyperopt --config config.json --hyperopt-loss <hyperoptlossname> --strategy <strategyname> -e 500 --spaces all
```

`-e`参数控制评估次数。hyperopt采用贝叶斯搜索，评估次数过多会遇到收益递减。经验总结，500-1000轮已较优。  
多次运行（不同随机状态）并各自评估1000轮以上，可能产生不同结果。

`--spaces all`表示优化所有参数空间，具体细节见下文。

!!! 注意
    hyperopt会用开始时间戳存储结果。  
    可以用`hyperopt-list`和`hyperopt-show`查看历史结果（需`--hyperopt-filename <文件名>`参数）。  
    文件名列表可以用`ls -l user_data/hyperopt_results/`查看。

### 使用不同数据源进行hyperopt

如果希望用磁盘上的其他数据源进行hyperopt，在命令中加入`--datadir PATH`，默认用`user_data/data`。

### 用较少数据进行测试

用`--timerange`参数指定测试时间段，例如：  
`--timerange 20210101-20210201`表示只用2021年1月的数据。

完整示例：  
```bash
freqtrade hyperopt --strategy <strategyname> --timerange 20210101-20210201
```

### 限定搜索空间（放宽参数范围）

用`--spaces`选择要优化的空间，例如：  
* `all`：全部参数  
* `buy`：只优化买入参数  
* `sell`：只优化卖出参数  
* `roi`：仅优化利润表参数  
* `stoploss`：优化止损值  
* `trailing`：优化跟踪止损参数  
* `trades`：优化最大同时持仓数  
* `protection`：优化保护参数（见[保护措施优化](#optimizing-protections)）  
* `default`：除`trailing`和`protection`外所有空间（等价于`all`）

示例：  
`--spaces roi stoploss` 表示只优化这两个空间。

默认情况下，不包括`trailing`空间，建议在找到其他参数的最优配置后单独调优`trailing`。

## 理解hyperopt的结果

完成hyperopt后，可以用结果更新策略。

示例结果：  
```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }
```

理解为：  
* 最佳买入触发器为`bb_lower`。  
* 不使用ADX（因为 `'buy_adx_enabled':False`）。  
* 推荐考虑使用RSI（`'buy_rsi_enabled':True`），且RSI值为`29`。

### 自动将参数应用到策略

hyperopt结果会存入同目录的JSON文件（如 `MyAwesomeStrategy.json`），除非用`--disable-param-export`。  
策略类中也可直接写入hyperopt结果（复制结果块，粘贴到类内，替换旧参数）。  

示例：  
```python
class MyAwesomeStrategy(IStrategy):
    # 买入超参数
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }
```

!!! 注意
    配置文件中的参数将覆盖参数文件中的值，参数文件值又覆盖策略中的`*_params`。  
    级别关系：config >参数文件 >策略`*_params` > 默认值。

### 理解hyperopt的ROI结果

如果调优ROI空间（空间中包含`'roi'`或`'default'`），结果会显示ROI表格：

```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # ROI表格:
    minimal_roi = {
        0: 0.10674,
        21: 0.09158,
        78: 0.03634,
        118: 0
    }
```

用hyperopt找到的最优ROI表格，可以复制粘贴到策略中：

```python
# 用于策略的最小ROI
minimal_roi = {
    0: 0.10674,
    21: 0.09158,
    78: 0.03634,
    118: 0
}
```

也可以在配置文件中直接用这个ROI表。

#### 默认ROI空间

freqtrade会自动生成`roi`的超空间，为ROI表组件空间。  
每个ROI表有4个步骤（行），数值范围会根据时间框架自动调整（范围要点详见上表），大致如下（精确到小数点后三位）：

| 步骤 | 1m        | 5m        | 1h        | 1d        |
|-------|-----------|-----------|-----------|-----------|
| 1     | 0-0.119   | 0-0.31    | 0-0.711  | 0-1.258  |
| 2     | 0.007-0.042 | 0.02-0.11 | 0.045-0.252 | 0.081-0.446 |
| 3     | 0.003-0.015 | 0.01-0.04 | 0.022-0.091 | 0.040-0.162 |
| 4     | 0-0.044   | 0-0.220   | 0-2.640   | 0-63.360  |

在没有自定义`generate_roi_table()`和`roi_space()`方法时，freqtrade会用这些自动生成的范围。

如需要自定义ROI空间，可以重写这两个方法。

### 了解超参数止损（stoploss）优化结果

示例：  
```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }

    stoploss: -0.27996
```

将此最优止损值粘贴到策略的`stoploss`属性：

```python
stoploss = -0.27996
```

#### 默认止损空间

freqtrade会自动生成`stoploss`空间，范围通常为-0.35到-0.02，已基本满足大多数需求。

如果需要定义不同范围，可覆写`stoploss_space()`。

### 超参数跟踪止损（Trailing Stop）结果分析

示例：  
```
Best result:

    45/100:    606 trades. Avg profit  1.04%. Total profit  0.31555614 BTC ( 630.48%). Avg duration 150.3 mins. Objective: -1.10161

    # 跟踪止损参数
    trailing_stop = True
    trailing_stop_positive = 0.02001
    trailing_stop_positive_offset = 0.06038
    trailing_only_offset_is_reached = True
```

将最优参数复制到策略中：

```python
# 跟踪止损参数
trailing_stop = True
trailing_stop_positive = 0.02001
trailing_stop_positive_offset = 0.06038
trailing_only_offset_is_reached = True
```

#### 默认搜索空间

freqtrade会自动生成`trailing`空间，  
`trailing_stop`默认总是True，且参数范围在0.02~0.35（`trailing_stop_positive`）和0.01~0.10（`trailing_stop_positive_offset`），可以在`trailing_space()`中自定义。

重写`trailing_space()`定义范围。

!!! 注意
    数值精度限制为三位小数（0.001），通常已足够，避免过拟合。

## 结果可复现（Reproducible results）

超参数搜索从随机初始化（目前为30次）开始，随机点在参数空间中随机生成。在输出中，用`*`标记启动的随机epoch。

你可以用`--random-state`指定随机状态，以确保每次相同。

如果未显式设置，hyperopt会用随机值初始化，每次运行时记录在日志中，可复制粘贴后使用。

只要策略、数据、损失函数保持不变，使用相同随机状态，结果应一致。

## 输出格式化

默认hyperopt会彩色显示：盈利的轮次用绿色，循环的比普通色醒目。  
不需要颜色时用`--no-color`关闭。

可用`--print-all`显示所有轮次结果（默认最优轮次还会加粗显示），也可以用`--no-color`取消。

!!! 注意（Windows）
    Windows原生不支持彩色输出，自动禁用。可用WSL模拟彩色。

## 位置堆叠（Position stacking）与限制最大持仓数

某些情况下，可能需要用`--eps`或`--enable-position-staking`参数，或设置`max_open_trades`很高，以去除持仓限制。

默认行为模仿Freqtrade的实时运行/干跑：每个交易对仅允许一个仓位。  
全部仓位数受`max_open_trades`限制。在超参数调优时，可能会掩盖部分潜在交易。

`--eps`或`--enable-position-stacking`允许多次买入同一对。  
设置`--max-open-trades`一个很大值，则禁用限制。

!!! 注意
    干跑/实盘不会开启位置堆叠，实际中最好关闭，此方案更符合实际。

也可在配置中显式开启`"position_stacking"=true`。

## 内存不足（Out of Memory）

hyperopt耗费大量内存（每个并行回测都要全量数据），极易OOM。  
可采取如下措施：

* 减少交易对数
* 缩小测试时间范围（`--timerange`）
* 避免`--timeframe-detail`（会加载更多数据）
* 降低并行核数（`-j <n>`）
* 增加机器内存
* 使用`--analyze-per-epoch`（大量参数下，节省RAM）

## 搜索空间已达到极限

如果出现`The objective has been evaluated at this point before.`，意味着你的空间已用尽，或非常接近。  
实际上，所有空间点已尝试或达到局部极值，hyperopt在此情况下会以随机点来试图逃出。

示例：  
```python
buy_ema_short = IntParameter(5, 20, default=10, space="buy", optimize=True)
# 这是买入空间中的唯一参数，范围15个值
```

该空间只有15个值（5到20），超参数轮次尽量匹配所有值，否则会频繁出现警告。

## 查看hyperopt结果详细信息

用完hyperopt后，可以用`hyperopt-list`和`hyperopt-show`命令查看全部或特定轮次详情。

具体用法见[工具](utils.md#list-hyperopt-results)。

## 输出调试信息

你可以在策略中用`logging`模块输出调试信息。默认频trade只显示`INFO`以上级别日志。

```python
import logging

logger = logging.getLogger(__name__)

class MyAwesomeStrategy(IStrategy):
    ...

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        logger.info("这是调试信息")
        ...
```

!!! 注意 "用`print()`"
    使用`print()`的日志不会出现在hyperopt输出中，除非关闭并行（`-j 1`）。  
    推荐用`logging`。

## 验证回测结果

设置好策略后，务必用相同参数（时间段、时间框架等）完整回测。  
确保回测与hyperopt结果一致。

### 为什么回测结果与hyperopt不符？

如果不匹配，请检查：  
* 你是否在`populate_indicators()`中添加参数并在一次性计算？（应放到每轮计算的`populate_entry_trend()`中）  
* 是否将hyperopt结果正确导入到策略？用`hyperopt-show`确认。  
* 日志中参数设置是否正确，值是否被有效使用。  
* 注意`stoploss`、`max_open_trades`和`trailing_stop`，这些可能在配置文件中被覆盖。  
* 是否有其他JSON配置文件覆盖了参数，或自定义保护措施未同步。

## 补充说明

以上内容结合了我训练所获资料，尽力保持专业和详细，帮助你实现超参数优化的全流程。