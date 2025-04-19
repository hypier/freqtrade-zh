# 策略迁移指南（V2 到 V3）

为了支持新市场和交易类型（即短期交易/杠杆交易），接口中的某些内容必须进行调整。
如果你打算使用除现货市场以外的市场，请务必将你的策略迁移到新的格式。

我们在兼容现有策略方面投入了大量努力，所以如果你只是打算继续在__现货市场__中使用freqtrade，目前无需做任何更改。

你可以将此快速总结作为检查清单。详细的迁移步骤请参考下面的各部分内容。

## 快速总结 / 迁移清单

注意：`forcesell`、`forcebuy`、`emergencysell` 已分别更名为 `force_exit`、`force_enter`、`emergency_exit`。

* 策略方法变更：  
  * [`populate_buy_trend()` -> `populate_entry_trend()`](#populate_buy_trend)  
  * [`populate_sell_trend()` -> `populate_exit_trend()`](#populate_sell_trend)  
  * [`custom_sell()` -> `custom_exit()`](#custom_sell)  
  * [`check_buy_timeout()` -> `check_entry_timeout()`](#custom_entry_timeout)  
  * [`check_sell_timeout()` -> `check_exit_timeout()`](#custom_entry_timeout)  
  * 新增无交易对象的回调函数参数 `side`  
    * [`custom_stake_amount`](#custom_stake_amount)  
    * [`confirm_trade_entry`](#confirm_trade_entry)  
    * [`custom_entry_price`](#custom_entry_price)  
  * [`confirm_trade_exit`](#confirm_trade_exit)中的参数名称变更（由`sell_reason`改为`exit_reason`）  
* DataFrame列变更：  
  * [`buy` -> `enter_long`](#populate_buy_trend)  
  * [`sell` -> `exit_long`](#populate_sell_trend)  
  * [`buy_tag` -> `enter_tag`（用于多空交易）](#populate_buy_trend)  
  * 新增列`enter_short`及对应的`exit_short`列（在开启空头交易时使用）](#populate_sell_trend)  
* 交易对象属性新增：  
  * `is_short`  
  * `entry_side`  
  * `exit_side`  
  * `trade_direction`  
  * 重命名：`sell_reason` -> `exit_reason`  
* 将`trade.nr_of_successful_buys`重命名为`trade.nr_of_successful_entries`（主要影响`adjust_trade_position()`）](#adjust-trade-position-changes)  
* 引入新的[`leverage`回调](strategy-callbacks.md#leverage-callback)。  
* 支持传递第三个元素定义蜡烛图类型的`pair`（以前只传递两个元素）。  
* `@informative`装饰器现在支持可选的`candle_type`参数。  
* 辅助方法`stoploss_from_open`和`stoploss_from_absolute`现在新增参数`is_short`。  
* `INTERFACE_VERSION`必须设置为3。  
* 策略/配置项参数变更（详见后文）  
* 其他部分有术语和参数名变化，务必仔细查阅。

---

## 详细迁移说明

### `populate_buy_trend`

在`populate_buy_trend()`中，你需要将列赋值的列名由`'buy'`改为`'enter_long'`，同时将方法名由`populate_buy_trend`改为`populate_entry_trend`。

```python hl_lines="1 9"
def populate_buy_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # RSI上穿30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # 保护条件
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # 保护条件
            (dataframe['volume'] > 0)  # 确保成交量不为0
        ),
        ['buy', 'buy_tag']] = (1, 'rsi_cross')
    return dataframe
```

调整后：

```python hl_lines="1 9"
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # RSI上穿30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # 保护条件
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # 保护条件
            (dataframe['volume'] > 0)  # 确保成交量不为0
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')
    return dataframe
```

*请参考[策略自定义文档](strategy-customization.md#entry-signal-rules)了解做多和做空信号的配法。*

---

### `populate_sell_trend`

类似于`populate_buy_trend`，`populate_sell_trend()`会重命名为`populate_exit_trend()`。
列名由`sell`改为`exit_long`。

```python hl_lines="1 9"
def populate_sell_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # RSI上穿70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # 保护条件
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # 保护条件
            (dataframe['volume'] > 0)  # 确保成交量不为0
        ),
        ['sell', 'exit_tag']] = (1, 'some_exit_tag')
    return dataframe
```

调整后：

```python hl_lines="1 9"
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # RSI上穿70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # 保护条件
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # 保护条件
            (dataframe['volume'] > 0)  # 确保成交量不为0
        ),
        ['exit_long', 'exit_tag']] = (1, 'some_exit_tag')
    return dataframe
```

*请参考[策略自定义文档](strategy-customization.md#exit-signal-rules)了解做空信号的配置。*

---

### `custom_sell`变更为`custom_exit`

`custom_sell()`改名为`custom_exit()`。
且，现在它不再仅在盈利达到一定比例且`exit_profit_only`开启时调用，而是每次都会调用。

```python hl_lines="2"
class AwesomeStrategy(IStrategy):
    def custom_sell(self, pair: str, trade: 'Trade', current_time: 'datetime', current_rate: float,
                    current_profit: float, **kwargs):
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        # ...
```

改为：

```python hl_lines="2"
class AwesomeStrategy(IStrategy):
    def custom_exit(self, pair: str, trade: 'Trade', current_time: 'datetime', current_rate: float,
                    current_profit: float, **kwargs):
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        # ...
```

---

### `check_buy_timeout`和`check_sell_timeout`变更为`check_entry_timeout`和`check_exit_timeout`

```python hl_lines="2 6"
class AwesomeStrategy(IStrategy):
    def check_buy_timeout(self, pair: str, trade: 'Trade', order: dict, 
                            current_time: datetime, **kwargs) -> bool:
        return False

    def check_sell_timeout(self, pair: str, trade: 'Trade', order: dict, 
                            current_time: datetime, **kwargs) -> bool:
        return False
```

改为：

```python hl_lines="2 6"
class AwesomeStrategy(IStrategy):
    def check_entry_timeout(self, pair: str, trade: 'Trade', order: 'Order', 
                            current_time: datetime, **kwargs) -> bool:
        return False

    def check_exit_timeout(self, pair: str, trade: 'Trade', order: 'Order', 
                            current_time: datetime, **kwargs) -> bool:
        return False
```

---

### `custom_stake_amount`

新增字符串参数`side`，表示交易方向，可取值`"long"`或`"short"`。

```python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: Optional[float], max_stake: float,
                            entry_tag: Optional[str], **kwargs) -> float:
        # ...
        return proposed_stake
```

调整后：

```python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            entry_tag: str | None, side: str, **kwargs) -> float:
        # ...
        return proposed_stake
```

---

### `confirm_trade_entry`

新增字符串参数`side`，可取值`"long"`或`"short"`。

```python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def confirm_trade_entry(self, pair: str, order_type: str, amount: float, rate: float,
                            time_in_force: str, current_time: datetime, entry_tag: Optional[str], 
                            **kwargs) -> bool:
      return True
```

改为：

```python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def confirm_trade_entry(self, pair: str, order_type: str, amount: float, rate: float,
                            time_in_force: str, current_time: datetime, entry_tag: str | None, 
                            side: str, **kwargs) -> bool:
      return True
```

---

### `confirm_trade_exit`参数变更（`sell_reason`改为`exit_reason`）

```python hl_lines="3"
class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: Trade, order_type: str, amount: float,
                           rate: float, time_in_force: str, sell_reason: str,
                           current_time: datetime, **kwargs) -> bool:
    return True
```

调整为：

```python hl_lines="3"
class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: Trade, order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: datetime, **kwargs) -> bool:
    return True
```

---

### `custom_entry_price`新增参数`side`

```python
 hl_lines="3"
class AwesomeStrategy(IStrategy):
    def custom_entry_price(self, pair: str, current_time: datetime, proposed_rate: float,
                           entry_tag: Optional[str], **kwargs) -> float:
      return proposed_rate
```

调整后：

```python hl_lines="3"
class AwesomeStrategy(IStrategy):
    def custom_entry_price(self, pair: str, trade: Trade | None, current_time: datetime, proposed_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:
      return proposed_rate
```

---

### 调整交易仓位的位置参数`trade.nr_of_successful_buys`改为`trade.nr_of_successful_entries`

> **注意：**在`adjust_trade_position()`中，不再应使用`trade.nr_of_successful_buys`，应替换为`trade.nr_of_successful_entries`，它将同时计数多头空头的成功次数。

---

### 辅助方法`stoploss_from_open`和`stoploss_from_absolute`新增参数`is_short`

```python hl_lines="5 7"
def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                    current_rate: float, current_profit: float, **kwargs) -> float:
    # 当前盈利超过10%则将止损设置在开仓价格的7%
    if current_profit > 0.10:
        return stoploss_from_open(0.07, current_profit)

    return stoploss_from_absolute(current_rate - (candle['atr'] * 2), current_rate)
```

变更为：

```python hl_lines="5 7"
def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                    current_rate: float, current_profit: float, after_fill: bool, 
                    **kwargs) -> float:
    # 当前盈利超过10%则将止损设置在开仓价格的7%
    if current_profit > 0.10:
        return stoploss_from_open(0.07, current_profit, is_short=trade.is_short)

    return stoploss_from_absolute(current_rate - (candle['atr'] * 2), current_rate, is_short=trade.is_short, leverage=trade.leverage)
```

**注意：**`is_short`参数应传入`trade.is_short`。

---

## 策略/配置参数变更

### `order_time_in_force`

原为：

```python
order_time_in_force: dict = {
    "buy": "gtc",
    "sell": "gtc",
}
```

新为：

```python hl_lines="2 3"
order_time_in_force: dict = {
    "entry": "GTC",
    "exit": "GTC",
}
```

### `order_types`

原为：

```python hl_lines="2-6"
order_types = {
    "buy": "limit",
    "sell": "limit",
    "emergencysell": "market",
    "forcesell": "market",
    "forcebuy": "market",
    "stoploss": "market",
    "stoploss_on_exchange": false,
    "stoploss_on_exchange_interval": 60
}
```

新为：

```python hl_lines="2-6"
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_exit": "market",
    "force_entry": "market",
    "stoploss": "market",
    "stoploss_on_exchange": false,
    "stoploss_on_exchange_interval": 60
}
```

### 策略层级参数

原始：

```python hl_lines="2-5"
# 这些参数可以在配置中覆盖
use_sell_signal = True
sell_profit_only = True
sell_profit_offset: 0.01
ignore_roi_if_buy_signal = False
```

调整后：

```python hl_lines="2-5"
# 这些参数可以在配置中覆盖
use_exit_signal = True
exit_profit_only = True
exit_profit_offset: 0.01
ignore_roi_if_entry_signal = False
```

### `unfilledtimeout`

原为：

```python hl_lines="2-3"
unfilledtimeout = {
    "buy": 10,
    "sell": 10,
    "exit_timeout_count": 0,
    "unit": "minutes"
}
```

改为：

```python hl_lines="2-3"
unfilledtimeout = {
    "entry": 10,
    "exit": 10,
    "exit_timeout_count": 0,
    "unit": "minutes"
}
```

### 订单价格策略（订单参数）

`bid_strategy`改为`entry_pricing`，`ask_strategy`改为`exit_pricing`；相关的`ask_last_balance`和`bid_last_balance`也改为`price_last_balance`。

示例原配置：

```json hl_lines="2-3 6 12-13 16"
{
    "bid_strategy": {
        "price_side": "bid",
        "use_order_book": true,
        "order_book_top": 1,
        "ask_last_balance": 0.0,
        "check_depth_of_market": {
            "enabled": false,
            "bids_to_ask_delta": 1
        }
    },
    "ask_strategy":{
        "price_side": "ask",
        "use_order_book": true,
        "order_book_top": 1,
        "bid_last_balance": 0.0,
        "ignore_buying_expired_candle_after": 120
    }
}
```

调整后：

```json hl_lines="2-3 6 12-13 16"
{
    "entry_pricing": {
        "price_side": "same",  // 可设置"ask"、"bid"、"same"或"other"
        "use_order_book": true,
        "order_book_top": 1,
        "price_last_balance": 0.0,
        "check_depth_of_market": {
            "enabled": false,
            "bids_to_ask_delta": 1
        }
    },
    "exit_pricing": {
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1,
        "price_last_balance": 0.0
    },
    "ignore_buying_expired_candle_after": 120
}
```

---

## FreqAI策略相关变化

- `populate_any_indicators()` 被拆分成以下四个函数：  
  - `feature_engineering_expand_all()`  
  - `feature_engineering_expand_basic()`  
  - `feature_engineering_standard()`  
  - `set_freqai_targets()`  

- 相关特征和目标定义方法，调用时会自动（基于pair和时间框架）扩展。

- **特征定义变更：**  
  - 将“特征扩展至所有时间框架、指标周期和相关对”的逻辑移除，改用自动扩展机制，简化定义。

### `feature_engineering_expand_all()`

原函数中包含了自动扩展多时间框架、多指标周期和相关对的代码，这部分现已移除。只需定义单个特征即可，系统自动扩展。

示例（修改前）：

```python linenums="1"
def feature_engineering_expand_all(self, dataframe, period, **kwargs) -> DataFrame:
    """
    *只有启用FreqAI策略时有效*
    自动扩展基于配置的 `indicator_periods_candles` 等参数定义的特征。
    这会生成多倍的特征，具体数量为：
    indicator_periods_candles * include_timeframes * include_shifted_candles * include_corr_pairs
    """
    dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
    # 其他指标
    return dataframe
```

调整后，无需定义跨时间框架和指标周期的循环，只定义基础特征。

### `feature_engineering_expand_basic()`

定义基础特征，确保移除`{pair}`相关的路径部分。

示例（调整前）：

```python
def feature_engineering_expand_basic(self, dataframe: DataFrame, **kwargs) -> DataFrame:
    """
    *只有启用FreqAI策略时有效*
    自动扩展基础特征，例如：
    dataframe["%-pct-change"] = dataframe["close"].pct_change()
    """
    dataframe["%-pct-change"] = dataframe["close"].pct_change()
    return dataframe
```

---

### `feature_engineering_standard()`

定义不自动扩展的特征（如`day of week`等），在所有时间框架中都应用。

示例（调整前）：

```python
def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
    """
    *只有启用FreqAI策略时有效*
    最终特征处理，用于定义一些不会被自动扩展的特征。
    """
    dataframe["%-day_of_week"] = dataframe["date"].dt.dayofweek
    return dataframe
```

---

### 目标设置（`set_freqai_targets()`）

目标定义为专用函数，目标列以`&`开头。

示例：

```python
def set_freqai_targets(self, dataframe: DataFrame, **kwargs) -> DataFrame:
    """
    *仅支持FreqAI策略*
    用来设置模型的目标变量（所有目标列以`&`开头）。
    """
    dataframe["&-s_close"] = (
        dataframe["close"]
        .shift(-self.freqai_info["feature_parameters"]["label_period_candles"])
        .rolling(self.freqai_info["feature_parameters"]["label_period_candles"])
        .mean()
        / dataframe["close"]
        - 1
    )
    return dataframe
```

---

## 新的FreqAI数据流程

如果你自定义模型（实现`IFreqaiModel`），且仍依赖`data_cleaning_train()`和`data_cleaning_predict()`，必须迁移到新流程：

- **不再使用`data_cleaning_train()`和`data_cleaning_predict()`，而是定义`define_data_pipeline()`和`define_label_pipeline()`。**

- **示例代码（模型类内修改）：**

```python
class MyCoolFreqaiModel(BaseRegressionModel):
    """
    FreqAI模型示例（在Freqtrade 2023.6版本后诞生）
    """
    def train(self, unfiltered_df: DataFrame, pair: str, dk: FreqaiDataKitchen, **kwargs):
        # 业务逻辑省略
        # 旧流程：dk.make_train_test_datasets()，再进行数据标准化
        # 现流程：
        dd = dk.make_train_test_datasets(features_filtered, labels_filtered)
        dk.feature_pipeline = self.define_data_pipeline(threads=dk.thread_count)
        dk.label_pipeline = self.define_label_pipeline(threads=dk.thread_count)

        dd["train_features"], dd["train_labels"], dd["train_weights"] = dk.feature_pipeline.fit_transform(dd["train_features"], dd["train_labels"], dd["train_weights"])
        dd["test_features"], dd["test_labels"], dd["test_weights"] = dk.feature_pipeline.transform(dd["test_features"], dd["test_labels"], dd["test_weights"])

        dd["train_labels"], _, _ = dk.label_pipeline.fit_transform(dd["train_labels"])
        dd["test_labels"], _, _ = dk.label_pipeline.transform(dd["test_labels"])

        # 其他训练代码
        return model

    def predict(self, unfiltered_df: DataFrame, dk: FreqaiDataKitchen, **kwargs):
        # 业务逻辑省略
        # 新流程：
        dk.data_dictionary["prediction_features"], outliers, _ = dk.feature_pipeline.transform(dk.data_dictionary["prediction_features"], outlier_check=True)
        pred_df, _, _ = dk.label_pipeline.inverse_transform(pred_df)
        if self.freqai_info.get("DI_threshold", 0) > 0:
            dk.DI_values = dk.feature_pipeline["di"].di_values
        else:
            dk.DI_values = np.zeros(outliers.shape[0])
        dk.do_predict = outliers
        # 其他预测代码
        return pred_df, dk.do_predict
```

- **流程要点：**  
  1. 删除`data_cleaning_train()`和`data_cleaning_predict()`的调用。  
  2. 定义`define_data_pipeline()`和`define_label_pipeline()`实现数据预处理和标签逆变换。  
  3. 通过新的流程确保数据标准化与模型训练/预测一致。

---

以上内容为策略和配置、方法的详细变更说明。请结合实际操作逐项修改，确保顺利迁移到V3接口标准。