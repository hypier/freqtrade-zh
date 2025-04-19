# 高级策略

本页介绍一些可用于策略的高级概念。
如果你刚开始使用，请先熟悉[Freqtrade基础](bot-basics.md)以及[策略自定义](strategy-customization.md)中描述的方法。

本文所述方法的调用顺序可在[bot执行逻辑](bot-basics.md#bot-execution-logic)中查阅。这些文档也有助于你判断哪些方法最适合你的自定义需求。

!!! 注意
    只有在策略使用回调方法时，才应实现相应的回调函数。

!!! 提示
    可以用以下命令，快速创建包含所有可用回调方法的策略模板：`freqtrade new-strategy --strategy MyAwesomeStrategy --template advanced`

## 存储信息（持久化）

Freqtrade允许在数据库中存储和检索与特定交易相关的用户自定义信息。

通过交易对象，可以使用`trade.set_custom_data(key='my_key', value=my_value)`存储信息，通过`trade.get_custom_data(key='my_key')`检索信息。每个数据项都关联一个交易和一个由用户提供的键（类型为`string`）。这意味着，只能在提供交易对象的回调中使用。

为了保证数据可以存入数据库，Freqtrade会对数据进行序列化。这是通过将数据转换为JSON格式字符串实现的。
Freqtrade会在取出数据时试图逆转此操作，从策略的角度来看，这一般不影响使用。

```python
from freqtrade.persistence import Trade
from datetime import timedelta

class AwesomeStrategy(IStrategy):

    def bot_loop_start(self, **kwargs) -> None:
        for trade in Trade.get_open_order_trades():
            fills = trade.select_filled_orders(trade.entry_side)
            if trade.pair == 'ETH/USDT':
                trade_entry_type = trade.get_custom_data(key='entry_type')
                if trade_entry_type is None:
                    trade_entry_type = 'breakout' if 'entry_1' in trade.enter_tag else 'dip'
                elif fills > 1:
                    trade_entry_type = 'buy_up'
                trade.set_custom_data(key='entry_type', value=trade_entry_type)
        return super().bot_loop_start(**kwargs)

    def adjust_entry_price(self, trade: Trade, order: Order | None, pair: str,
                           current_time: datetime, proposed_rate: float, current_order_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:
        # 限价单在BTC/USDT对进入触发后，前10分钟内使用并跟随SMA200作为价格目标。
        if (
            pair == 'BTC/USDT' 
            and entry_tag == 'long_sma200' 
            and side == 'long' 
            and (current_time - timedelta(minutes=10)) > trade.open_date_utc 
            and order.filled == 0.0
        ):
            dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
            current_candle = dataframe.iloc[-1].squeeze()
            # 存储入场调整信息
            existing_count = trade.get_custom_data('num_entry_adjustments', default=0)
            if not existing_count:
                existing_count = 1
            else:
                existing_count += 1
            trade.set_custom_data(key='num_entry_adjustments', value=existing_count)

            # 调整订单价格
            return current_candle['sma_200']

        # 默认保持原有订单
        return current_order_rate

    def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float, current_profit: float, **kwargs):

        entry_adjustment_count = trade.get_custom_data(key='num_entry_adjustments')
        trade_entry_type = trade.get_custom_data(key='entry_type')
        if entry_adjustment_count is None:
            if current_profit > 0.01 and (current_time - timedelta(minutes=100) > trade.open_date_utc):
                return True, 'exit_1'
        else:
            if entry_adjustment_count > 0 and current_profit > 0.05:
                return True, 'exit_2'
            if trade_entry_type == 'breakout' and current_profit > 0.1:
                return True, 'exit_3'

        return False, None
```

上述为一个简单示例——还有更简便的方法可以获取交易的入场调整信息。

!!! 注意
    建议使用基本数据类型（[bool, int, float, str]）以确保存储的数据不会出现序列化问题。
    存储大量数据可能引发意料之外的问题，比如数据库变大（导致性能变差甚至崩溃）。

!!! 警告 "不可序列化数据"
    如果提供的数据无法序列化，会记录警告，并且指定`key`的条目中将存入`None`。

??? 备注 "所有属性"
    自定义数据（custom-data）通过Trade对象提供以下访问器（以下假设对象为`trade`）：

    * `trade.get_custom_data(key='something', default=0)` - 返回指定键的值，类型为对应的类型。
    * `trade.get_custom_data_entry(key='something')` - 返回完整条目（包含元数据），值可通过`.value`属性访问。
    * `trade.set_custom_data(key='something', value={'some': 'value'})` - 设置或更新某个交易的指定键。值必须可序列化，建议存储数据尽量小。

    "值"可以是任何类型（在设置和读取时），但必须是JSON可序列化的。

## 存储信息（非持久化）

!!! 警告 "已弃用"
    此存储信息的方法已被废弃，我们不建议使用非持久化存储方式。
    请采用[持久化存储](#存储信息-持久化)方案。

    相关内容已折叠。

??? 摘要 "存储信息"
    通过在策略类中创建一个新的字典实现存储功能。

    变量名可以自定义，但应以`custom_`作为前缀，以避免与预定义策略变量冲突。

    ```python
    class AwesomeStrategy(IStrategy):
        # 创建自定义字典
        custom_info = {}

        def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            # 检查是否已存在该交易对的条目
            if not metadata["pair"] in self.custom_info:
                # 创建空条目
                self.custom_info[metadata["pair"]] = {}

            if "crosstime" in self.custom_info[metadata["pair"]]:
                self.custom_info[metadata["pair"]]["crosstime"] += 1
            else:
                self.custom_info[metadata["pair"]]["crosstime"] = 1
    ```

    !!! 警告
        数据在策略重启（或配置重新加载）后不会被保存。并且数据量应保持较小（不要使用DataFrame等大块数据），否则会占用大量内存，甚至导致内存耗尽崩溃。

    !!! 备注
        如果数据与交易对相关，应将交易对作为字典的键之一。

## Dataframe访问

你可以在各种策略函数中通过查询数据提供者获取DataFrame。

```python
from freqtrade.exchange import timeframe_to_prev_date

class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: 'Trade', order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: 'datetime', **kwargs) -> bool:
        # 获取交易对的数据框
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)

        # 获取最后一根K线。不要用current_time查最新K线，因为
        # current_time指向的是当前未完整的K线，该数据不可用。
        last_candle = dataframe.iloc[-1].squeeze()
        # <...>

        # 在 dry/live 交易中，交易开启时间可能与K线开盘时间不一致，因此需要四舍五入。
        trade_date = timeframe_to_prev_date(self.timeframe, trade.open_date_utc)
        # 查询交易的K线
        trade_candle = dataframe.loc[dataframe['date'] == trade_date]
        # 由于交易刚开启，数据可能尚未完整，trade_candle可能为空。
        if not trade_candle.empty:
            trade_candle = trade_candle.squeeze()
            # <...>
```

!!! 警告 "使用 .iloc[-1]"
    你可以在这里用`.iloc[-1]`，因为`get_analyzed_dataframe()`只返回允许进行回测的K线数据。
    但在`populate_*`方法中不要用`.iloc[]`，因为在这些方法里数据未必完整。
    并且，从版本2021.5开始，才支持这样用。

***

## 入场标签

当策略有多个买入信号时，你可以为触发的信号命名。
然后在`custom_exit`中可以访问这个买入信号的标签。

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] < 35) &
            (dataframe['volume'] > 0)
        ),
        ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')

    return dataframe

def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                current_profit: float, **kwargs):
    dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
    last_candle = dataframe.iloc[-1].squeeze()
    if trade.enter_tag == 'buy_signal_rsi' and last_candle['rsi'] > 80:
        return 'sell_signal_rsi'
    return None

```

!!! 注意
    `enter_tag`长度限制为100个字符，超出部分会被截断。

!!! 警告
    只有一个`enter_tag`列，且同时用于多头和空头交易。
    因此，该列应遵循“最后写入的优先”原则（毕竟它只是个DataFrame列）。
    在某些复杂场景中，如果多个信号冲突（或由不同条件再次禁用某信号），可能会导致标签被意外覆盖，显示错误的入口信号。
    这是因为策略会覆盖之前的标签——最后一个标签会“占用”该字段，Freqtrade会据此决策。

## 退出标签

类似于[入场标签](#enter-tag)，你也可以为退出策略指定标签。

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] > 70) &
            (dataframe['volume'] > 0)
        ),
        ['exit_long', 'exit_tag']] = (1, 'exit_rsi')

    return dataframe
```

指定的退出标签将作为卖出理由，并在回测结果中显示。

!!! 注意
    `exit_reason`长度限制为100个字符，超出部分会被截断。

## 策略版本

你可以通过实现`version`方法，自定义策略的版本信息，并返回所需的版本字符串。

```python
def version(self) -> str:
    """
    返回策略的版本号。
    """
    return "1.1"
```

!!! 备注
    建议配合版本控制系统（如git）使用，因Freqtrade不会保留策略的历史版本，用户需要自行管理版本回退。

## 派生策略

策略可以继承自其他策略。这可以避免重复编写策略代码。你可以通过这种方式只重写部分内容，其余部分保持不变：

```python title="user_data/strategies/myawesomestrategy.py"
class MyAwesomeStrategy(IStrategy):
    ...
    stoploss = 0.13
    trailing_stop = False
    # 其他所有属性和方法保持不变...
    ...
```

```python title="user_data/strategies/MyAwesomeStrategy2.py"
from myawesomestrategy import MyAwesomeStrategy
class MyAwesomeStrategy2(MyAwesomeStrategy):
    # 重写某些内容
    stoploss = 0.08
    trailing_stop = True
```

属性和方法都可以被重写，从而修改原策略的行为。

虽然在同一文件中定义子类是可行的，但可能会与参数优化（hyperopt）文件产生冲突。因此，建议将策略拆分到不同的文件，并像上面一样导入父策略。

## 嵌入策略

Freqtrade提供一种方便的方法，将策略嵌入到配置文件中。
通常通过Base64编码，并将生成的字符串在策略配置字段中指定，即在配置文件中填写。

### 将字符串编码为BASE64

以下是用Python快速生成BASE64字符串的示例：

```python
from base64 import urlsafe_b64encode

with open(file, 'r') as f:
    content = f.read()
content = urlsafe_b64encode(content.encode('utf-8'))
```

变量`content`即为策略文件内容的BASE64编码，可以在配置文件中这样使用：

```json
"strategy": "NameOfStrategy:BASE64String"
```

请确保`NameOfStrategy`与策略名称完全一致！

## 性能警告

在执行策略时，日志中可能会提示：

> PerformanceWarning: DataFrame is highly fragmented.

这是 [`pandas`](https://github.com/pandas-dev/pandas) 发出的警告，其建议是使用`pd.concat(axis=1)`。
这可能会带来轻微的性能影响，尤其在进行指标优化（hyperopt）时更明显。

例如：

```python
for val in self.buy_ema_short.range:
    dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)
```

应改写为：

```python
frames = [dataframe]
for val in self.buy_ema_short.range:
    frames.append(DataFrame({
        f'ema_short_{val}': ta.EMA(dataframe, timeperiod=val)
    }))

# 合并所有DataFrame，并将原始DataFrame赋值
dataframe = pd.concat(frames, axis=1)
```

不过，Freqtrade在调用`populate_indicators()`后，会在DataFrame上执行`dataframe.copy()`，因此性能影响基本可以忽略。