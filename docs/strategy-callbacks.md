# 策略回调函数

虽然主要的策略函数（`populate_indicators()`、`populate_entry_trend()`、`populate_exit_trend()`）应以向量化的方式使用，并且仅在**回测时**调用 [backtesting](bot-basics.md#backtesting-hyperopt-execution-logic) 中调用一次，但回调函数则是在“需要时”调用。

因此，避免在回调函数中进行繁重的计算，以免影响操作的实时性。根据使用的回调类型不同，它们可能在进场/离场时调用，或在持仓期间持续调用。

目前已支持的回调函数包括：

* [`bot_start()`](#bot-start)
* [`bot_loop_start()`](#bot-loop-start)
* [`custom_stake_amount()`](#stake-size-management)
* [`custom_exit()`](#custom-exit-signal)
* [`custom_stoploss()`](#custom-stoploss)
* [`custom_entry_price()` 和 `custom_exit_price()`](#custom-order-price-rules)
* [`check_entry_timeout()` 和 `check_exit_timeout()`](#custom-order-timeout-rules)
* [`confirm_trade_entry()`](#trade-entry-buy-order-confirmation)
* [`confirm_trade_exit()`](#trade-exit-sell-order-confirmation)
* [`adjust_trade_position()`](#adjust-trade-position)
* [`adjust_entry_price()`](#adjust-entry-price)
* [`leverage()`](#leverage-callback)
* [`order_filled()`](#order-filled-callback)

!!! Tip "回调调用序列"  
    实际调用顺序可在 [bot-basics](bot-basics.md#bot-execution-logic) 中查看。

--8<-- "includes/strategy-imports.md"

## Bot 启动

当策略加载完成时，系统会调用一次这个简单的回调函数。  
它可用于执行仅需运行一次的操作，且在数据提供器（dataprovider）和钱包（wallet）设置完成之后执行。

```python
import requests

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    def bot_start(self, **kwargs) -> None:
        """
        仅在实例化后调用一次。
        :param **kwargs:**：确保保留此参数，避免更新策略时出错。
        """
        if self.config["runmode"].value in ("live", "dry_run"):
            # 绑定到类实例，以便在其他方法中使用
            # 在populate_*方法中也能用
            self.custom_remote_data = requests.get("https://some_remote_source.example.com")
```

在超参数优化（hyperopt）过程中，此函数在启动时只会调用一次。

## Bot 循环开始

每次在 dry/live 模式下（大致每 5 秒，除非配置不同），在每轮循环开始时调用；在回测/超参数优化模式下，每个蜡烷线（candle）开始时调用。  
此回调可以用来执行与配对无关的计算（比如加载外部数据）等。

```python
# 预定义导入
import requests

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    def bot_loop_start(self, current_time: datetime, **kwargs) -> None:
        """
        每轮循环（一次）开始时调用。
        可以用来执行与配对无关的任务
        （比如加载一些远程资源以进行比较）
        :param current_time: datetime对象，表示当前时间
        :param **kwargs:**：确保保留此参数，避免更新策略时出错。
        """
        if self.config["runmode"].value in ("live", "dry_run"):
            # 绑定到类实例，以便在其他方法中使用
            self.remote_data = requests.get("https://some_remote_source.example.com")
```

## 持仓规模管理

在开启新交易之前调用，便于在下单时管理仓位大小。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            leverage: float, entry_tag: str | None, side: str,
                            **kwargs) -> float:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
        current_candle = dataframe.iloc[-1].squeeze()

        if current_candle["fastk_rsi_1h"] > current_candle["fastd_rsi_1h"]:
            if self.config["stake_amount"] == "unlimited":
                # 在有利条件下，利用复利模式，全部可用资金
                return max_stake
            else:
                # 利润复投，静态仓位不变
                return self.wallets.get_total_stake_amount() / self.config["max_open_trades"]

        # 使用默认的仓位
        return proposed_stake
```

如果你的代码抛出异常，Freqtrade会回退到`proposed_stake`的值。异常会被记录。

!!! Tip  
    你无需确保 `min_stake <= 返回值 <= max_stake`，返回值会被限制在支持范围内，且日志会记录。

!!! Tip  
    返回 `0` 或 `None` 表示不要开仓（不下单）。

## 自定义离场信号

在每个调节周期（大约每 5 秒）检测已开的仓位，可调用该函数决定是否卖出。  
支持定义自定义的退出信号，表示应卖出对应仓位。特别适合根据不同条件自定义退出策略，或基于交易数据作出退出决策。

例如，使用此函数实现1:2的风险收益比ROI。

**注意：**用 `custom_exit()` 代替 stoploss 并不推荐。相较于 `custom_stoploss()` ，`custom_exit()`不支持在交易所托管止损，效果较差。  
它更像是“在蜡烛上”设置退出信号的方式——但实际上，`custom_stoploss()`能更好地实现此功能，同时还能在交易所设置止损。

!!! 备注  
    从此方法返回（非空的）`字符串`或`True`，等同于在指定时间的蜡烛上设定退出信号。  
    若已设定退出信号或退出信号已禁用（`use_exit_signal=False`），此方法不会被调用。  
    `字符串`最大长度为64字符，超出会被截断到64字符。  
    `custom_exit()`会忽略`exit_profit_only`参数，即使有新入场信号，仍会调用。

举例：依据当前利润判断，退出持仓（如持仓超过一天的交易）：

```python
# 预定义导入

class AwesomeStrategy(IStrategy):
    def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                    current_profit: float, **kwargs):
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()

        # 利润超过20%时，RSI<80即卖
        if current_profit > 0.2:
            if last_candle["rsi"] < 80:
                return "rsi_below_80"

        # 资金收益在2%到10%之间，当EMA长线在EMA短线之上时卖出
        if 0.02 < current_profit < 0.1:
            if last_candle["emalong"] > last_candle["emashort"]:
                return "ema_long_below_80"

        # 盈亏以亏损持仓超过一天时卖出
        if current_profit < 0.0 and (current_time - trade.open_date_utc).days >= 1:
            return "unclog"
```

更多关于DataFrame在策略回调中的用法，可以参考[Dataframe访问](strategy-advanced.md#dataframe-access)。

## 自定义止损

在每次持仓（大约每5秒）检测，直到仓位平掉。

启用方式：在策略配置中设置 `use_custom_stoploss=True`。

止损价只会向上移动——如果从 `custom_stoploss()` 返回的值导致止损价低于之前的值，则被忽略。  
传统止损（`stoploss`）值作为绝对底线，确保不会低于设定值（在第一次调用前设定），依然需要配置。

由于自定义止损和常规止损类似，行为也类似“追踪止损（trailing stop）”，因此退出时的`exit_reason`会标记为 `"trailing_stop_loss"`。

此函数返回值为比例类型（浮点数）：  
例如，当前价为 200 美元，返回 `0.02`，则止损价为 2%低于当前价，即196美元。

在回测中，`current_rate` 和 `current_profit`均基于蜡烛最高价（空头仓为最低价）进行计算，止损价格以蜡烛最低（多头）或最高（空头）为依据。

返回值的绝对值会被使用（符号忽略）：  
返回 `0.05` 和 `-0.05` 的效果相同，均代表止损在当前价格下方5%；  
返回`None`表示“不变”，这是在你不想改动止损时最安全的返回值。  
`NaN`或`inf`值视为无效，会被忽略。

挂单在交易所上的止损行为类似于`trailing_stop`，会根据`stoploss_on_exchange_interval`的配置定期更新（详见[交易所止损](stoploss.md#stop-loss-on-exchangefreq)）。

!!! 备注 “日期时间使用”
    所有与时间相关的计算应基于`current_time`，避免直接用`datetime.now()`或`datetime.utcnow()`，否则会影响回测支持。

!!! Tip “追踪止损”
    建议在使用自定义止损时关闭`trailing_stop`，两者可以同时工作，但可能导致追踪止损在某些情况下向上移动，与你的自定义函数目标不符而发生冲突。

### 位置调整后再调节止损

根据策略需求，在[调整持仓](#adjust-trade-position)后，可能需要对止损价进行调整。  
此时，freqtrade会在订单成交后再次调用`custom_stoploss()`，传入参数`after_fill=True`，允许策略在任意方向调整止损（也可扩大止损与当前价的距离，避免被限制）。

!!! 备注 “向后兼容”  
    只有在`custom_stoploss()`定义中包含`after_fill`参数，才会调用此额外的调节。  
    这不会影响现有已运行的策略。

### 自定义止损示例

以下示例介绍利用自定义止损函数实现多种策略，当然可以灵活组合。

#### 利用自定义止损实现追踪止损

模拟4%的追踪止损（最高价追踪）非常简单：

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(
        self,
        pair: str,
        trade: Trade,
        current_time: datetime,
        current_rate: float,
        current_profit: float,
        after_fill: bool,
        **kwargs
    ) -> float | None:
        """
        自定义止损逻辑，返回相对于当前报价的距离（比值）
        例如返回 -0.05 表示止损为当前价5%以下
        自定义止损不会低于 `self.stoploss`
        """
        return -0.04
```

#### 基于时间的追踪止损

前60分钟使用原始止损值，之后变为10%的追踪止损，2小时后（120分钟）变为5%的追踪止损。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        # 时间从长到短排序，先检测最大时间（120分钟）
        if current_time - timedelta(minutes=120) > trade.open_date_utc:
            return -0.05
        elif current_time - timedelta(minutes=60) > trade.open_date_utc:
            return -0.10
        return None
```

#### 基于时间且支持`after_fill`调整的止损

结合时间和多订单填充情况：

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        if after_fill:
            # 订单填充后，将止损设为低于新开仓价的10%
            return stoploss_from_open(0.10, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        # 时间判断
        if current_time - timedelta(minutes=120) > trade.open_date_utc:
            return -0.05
        elif current_time - timedelta(minutes=60) > trade.open_date_utc:
            return -0.10
        return None
```

#### 针对不同交易对设定不同止损

对不同交易对采用不同止损策略。例如，`ETH/BTC` 和 `XRP/BTC` 使用10%的追踪止损，`LTC/BTC` 使用5%，其他都用15%。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        if pair in ("ETH/BTC", "XRP/BTC"):
            return -0.10
        elif pair in ("LTC/BTC",):
            return -0.05
        return -0.15
```

#### 正偏移量追踪止损

当利润超过4%后开始追踪止损，为当前利润的50%，最小2.5%，最大5%。

注意：止损只允许变大（向上移动），低于当前止损的值会被忽略。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        if current_profit < 0.04:
            return None  # 继续用原止损
        # 达到利润偏移后，根据利润设定新的止损
        desired_stoploss = current_profit / 2
        # 最小2.5%，最大5%
        return max(min(desired_stoploss, 0.05), 0.025)
```

#### 阶梯式止损

避免连续追踪，设定固定的止损线，根据利润水平变化。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        # 从高到低判断，确保用最严格的止损条件
        if current_profit > 0.40:
            return stoploss_from_open(0.25, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        elif current_profit > 0.25:
            return stoploss_from_open(0.15, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        elif current_profit > 0.20:
            return stoploss_from_open(0.07, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        # 超出范围就保持当前止损不变
        return None
```

#### 利用指标作为绝对止损点（如Parabolic SAR示例）

可以依据某些指标的绝对值作为止损点，例如Parabolic SAR。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # ... 其他指标
        dataframe["sar"] = ta.SAR(dataframe)

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()

        # 使用parabolic SAR值作为绝对止损点
        stoploss_price = last_candle["sar"]

        # 转换为比例（相对当前价格）
        if stoploss_price < current_rate:
            return stoploss_from_absolute(stoploss_price, current_rate, is_short=trade.is_short)
        # 不变化
        return None
```

更多策略细节请参考[Dataframe访问](strategy-advanced.md#dataframe-access)。

---

## 常用止损计算辅助函数

### 相对于开仓价的止损值

`custom_stoploss()`返回值应是相对`current_rate`的比例值，但有时希望相对`入场价`设定止损。  
此时可以用`stoploss_from_open()`辅助函数，根据入场价与目标止损比例，计算出对应的相对比例（比值），由`custom_stoploss()`返回。

??? 举例：“返回相对于入场价的止损比例代码示例”

假设：

- 入场价为100美元（`open_price=100`）
- 当前价为121美元（`current_price=121`，`current_profit=0.21`）

如果想设置突破入场价7%的止损点，可调用：`stoploss_from_open(0.07, current_profit, False)`，返回值约为0.1157（11.57%）  
意味着止损位为121 * (1 - 0.1157) ≈ 107美元，正好是比100美元高7%。

此函数会考虑杠杆（`leverage`），例如10倍杠杆实际止损则为0.7%（0.7% * 10）。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        # 利润超过10%，保持止损在7%上方
        if current_profit > 0.10:
            return stoploss_from_open(0.07, current_profit, is_short=trade.is_short, leverage=trade.leverage)

        return 1  # 未超过则返回默认值

# 详细用法参考文档中 [Custom Stoploss](strategy-callbacks.md#custom-stoploss)
```

!!! 备注  
    输入非法参数可能导致“CustomStoploss未返回有效止损”警告。  
    比如，`current_profit`低于设置的`open_relative_stop`，会阻止平仓（阻塞条件由`confirm_trade_exit()`决定）。  
    避免此问题：不要阻挡止损卖出（在`confirm_trade_exit()`中检查`exit_reason`），或用`return stoploss_from_open(...) or 1`，可确保不改变止损。

### 绝对价格对应的止损百分比

`custom_stoploss()`返回值始终是相对于`current_rate`的比例。  
若需要以绝对价格设定止损，需用`stop_rate`辅助计算对应比例。

举例：  
想在当前价格下2倍ATR之下设止损，可调用：  
`stoploss_from_absolute(current_rate + (side * candle["atr"] * 2), current_rate=current_rate, is_short=trade.is_short, leverage=trade.leverage)`

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    use_custom_stoploss = True

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # ... 其他指标
        dataframe["atr"] = ta.ATR(dataframe, timeperiod=14)
        return dataframe

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        trade_date = timeframe_to_prev_date(self.timeframe, trade.open_date_utc)
        candle = dataframe.iloc[-1].squeeze()
        side = 1 if trade.is_short else -1
        return stoploss_from_absolute(current_rate + (side * candle["atr"] * 2), 
                                      current_rate=current_rate, 
                                      is_short=trade.is_short,
                                      leverage=trade.leverage)
```

---

## 自定义订单价格规则

默认情况下，Freqtrade会使用订单簿（orderbook）中的价格自动设定订单价格（详见[价格配置](configuration.md#prices-used-for-orders)）。  
但你也可以基于自己的策略定义自定义的挂单价格。

在策略中实现`custom_entry_price()` 和 `custom_exit_price()`两个方法，即可自定义入场/出场的订单价格。

这两个方法在挂单提交到交易所之前调用。

!!! 备注
    如果你的自定义价格函数返回`None`或无效值，系统会回退使用`proposed_rate`，即基于默认配置的价格。

!!! 备注
    使用`custom_entry_price()`时，首次关联此策略的订单生成后，`trade`参数值为`None`。

### 自定义订单入场和出场价格示例

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... 其他 populate_* 方法

    def custom_entry_price(self, pair: str, trade: Trade | None, current_time: datetime, proposed_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:
        dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
        new_entryprice = dataframe["bollinger_10_lowerband"].iat[-1]
        return new_entryprice

    def custom_exit_price(self, pair: str, trade: Trade,
                          current_time: datetime, proposed_rate: float,
                          current_profit: float, exit_tag: str | None, **kwargs) -> float:
        dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
        new_exitprice = dataframe["bollinger_10_upperband"].iat[-1]
        return new_exitprice
```

!!! 警告
    修改订单价格仅影响限价订单（limit orders），实际应用中可能导致很多不成交订单。  
    默认情况下，最大支持的价格偏差为当前价的2%，可以在配置中通过`custom_price_max_distance_ratio`参数修改。  
    **示例**：  
    假设 `new_entryprice` 为97，`proposed_rate` 为100，且`custom_price_max_distance_ratio`设为2%，  
    则实际采纳的自定义入场价格为98（比 proposal 低2%），即在允许范围内。

!!! 警告 "回测支持"  
    自定义价格在回测中支持（从2021.12开始），订单会在价格落在蜡烛的最低/最高值范围内成交。  
    但未立即成交的订单会受到常规超时机制的限制，每个蜡烛线检测一次。  
    `custom_exit_price()`只在出场（sell）订单（出场信号、主动退出、部分退出）时调用，其他出场类型会用默认价格。

---

## 自定义订单超时规则

可以通过时间设定订单超时，配置文件中的`unfilledtimeout`参数也支持，但Freqtrade还支持定义自定义的超时回调函数，基于自定义条件判断订单是否超时。

!!! 备注
    回测时，订单在价格落在蜡烛最低/最高范围内会自动成交。  
    未立即成交的订单会由超时机制（每蜡烛线检测一次）处理。  
    如订单未成交，定义的超时回调会被调用。

### 自定义订单超时示例

该示例为每个未成交订单调用，利用不同价格条件设置超时机制。  
适合高价资产设置较短超时，低价资产留更多时间。

返回`True`表示订单被取消，返回`False`表示订单继续待成交。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... 其他 populate_* 方法

    # 设置超时时间为25小时（最大超时时间为24小时，因此这里设置为更长时间）
    unfilledtimeout = {
        "entry": 60 * 25,
        "exit": 60 * 25
    }

    def check_entry_timeout(self, pair: str, trade: Trade, order: Order,
                            current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
            return True
        return False

    def check_exit_timeout(self, pair: str, trade: Trade, order: Order,
                           current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
            return True
        return False
```

!!! 备注  
    上述示例中，`unfilledtimeout`必须设置超时时间大于24小时，否则会优先触发。

---

### 使用附加数据的超时示例

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... 其他 populate_* 方法

    unfilledtimeout = {
        "entry": 60 * 25,
        "exit": 60 * 25
    }

    def check_entry_timeout(self, pair: str, trade: Trade, order: Order,
                            current_time: datetime, **kwargs) -> bool:
        ob = self.dp.orderbook(pair, 1)
        current_price = ob["bids"][0][0]
        # 若当前价格比订单价高出2%以上，则取消订单
        if current_price > order.price * 1.02:
            return True
        return False

    def check_exit_timeout(self, pair: str, trade: Trade, order: Order,
                           current_time: datetime, **kwargs) -> bool:
        ob = self.dp.orderbook(pair, 1)
        current_price = ob["asks"][0][0]
        # 若当前价格比订单价低出2%以上，则取消订单
        if current_price < order.price * 0.98:
            return True
        return False
```

---

## 交易确认

在订单下达之前，确认交易（买入或卖出）是否成立。

### 买入订单确认

`confirm_trade_entry()`在订单提交的最后时刻被调用，可用以提前撤销订单（比如预期价格变化不符合要求等）。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... 其他 populate_* 方法

    def confirm_trade_entry(self, pair: str, order_type: str, amount: float, rate: float,
                            time_in_force: str, current_time: datetime, entry_tag: str | None,
                            side: str, **kwargs) -> bool:
        """
        在下入场（买入、开多）订单之前调用。
        时间敏感，不要在此进行耗时积累或网络请求。

        详细文档参考：https://www.freqtrade.io/en/latest/strategy-advanced/

        若策略未实现此方法，默认返回True（始终确认）。
        :param pair: 即将买入/做空的交易对
        :param order_type: 订单类型（如限制(limit)或市价(market)）
        :param amount: 目标货币数量
        :param rate: 换算价格（限价订单时采用此价格，市价订单此参数无影响）
        :param time_in_force: 持续时间（如GTC，默认为“有效直到取消”）
        :param current_time: 当前时间（datetime对象）
        :param entry_tag: 可选的买入标签（buy_tag），有用于识别
        :param side: "long"或"short"，表示拟议交易方向
        :param **kwargs: 其他参数
        :return: True表示确认订单，False表示取消
        """
        return True
```

### 出场（卖出）订单确认

`confirm_trade_exit()`在订单提交的最后时刻调用，可用以撤销不合理的出场请求。

同一轮循环中，可能会多次调用此函数，以不同出场原因处理。

出场原因（如果有）会按下列顺序调用：  
`exit_signal` / `custom_exit` —> `stop_loss` —> `roi` —> `trailing_stop_loss`

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... 其他 populate_* 方法

    def confirm_trade_exit(self, pair: str, trade: Trade, order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: datetime, **kwargs) -> bool:
        """
        在提交平仓（卖出）订单之前调用。
        亦可多次调用，依据不同退出原因。

        详细文档参考：https://www.freqtrade.io/en/latest/strategy-advanced/

        若未实现，默认返回True。
        :param pair: 交易对
        :param trade: 交易对象
        :param order_type: 订单类型（限价或市价）
        :param amount: 订单交易量
        :param rate: 订单价格
        :param time_in_force: 持续时间
        :param exit_reason: 退出原因（可为：["roi", "stop_loss", "stoploss_on_exchange", "trailing_stop_loss", "exit_signal", "force_exit", "emergency_exit"]）
        :param current_time: 当前时间（datetime）
        :param **kwargs: 其他参数
        :return: True/False
        """
        # 拒绝在亏损条件下强制平仓
        if exit_reason == "force_exit" and trade.calc_profit_ratio(rate) < 0:
            return False
        return True
```

!!! 警告
    `confirm_trade_exit()`可能阻止止损操作，导致亏损扩大（被阻止的止损将无法执行）。  
    交易所的强制清仓（liquidation）不会调用此函数，因其由交易所强制执行，不可拒绝。

---

## 调整持仓

开启`position_adjustment_enable`策略属性后，策略可以使用`adjust_trade_position()`回调函数。  
（默认禁用，启用时系统会有警告。）  
此函数可用于在仓位内进行额外下单（如DCA、增减仓等），而不会影响最大开仓数（`max_open_trades`）。

此回调会在订单（买入或卖出）等待执行时被调用，若订单的数量、价格、方向变化，会取消原订单，重新下单，还会自动取消部分成交订单。

此函数会在持仓期间频繁调用，建议实现高效。

仓位调节始终沿持仓方向操作（正值增仓，负值减仓），但不支持调整杠杆。  
返回的仓位数值为“预期仓位数量（stake_amount）”，在调用时，仓位参数未考虑杠杆效果。

仓位在`trade.stake_amount`中实时更新，每次操作（新仓或部分平仓）都会更新。

!!! 警告 "松散逻辑"
    在 dry/live 测试时，此函数会每隔`throttle_process_secs`（默认5秒）调用一次。  
    某些策略（如“当最后蜡烛RSI<30时增仓”）可能导致无限循环（每5秒重复多次入仓，可能因资金不足或达到最大调仓限制停止）。

    同理，部分平仓时也可能频繁调用，需保证逻辑严谨，避免陷入无限循环。

!!! 警告 "性能考虑"
    频繁调仓可能会影响策略性能，尤其是在长时间运行且调仓频次高的情况下。  
    每次调仓会产生额外的内存和计算负担。建议合理规划调仓条件，避免过度频繁操作。

!!! 警告 "回测注意"
    在回测中，此函数会针对每个蜡烛调用，可能拖慢运行速度。  
    也可能导致实盘与回测结果不一致（回测每蜡烛调一次，实盘可以多次调仓）。

### 增仓示例
策略在满足条件时返回一个正值，表示增仓（买入或做多）。

```python
# 预定义导入

class DigDeeperStrategy(IStrategy):

    position_adjustment_enable = True

    # 高跌幅下采取DCA，设置较高止损
    stoploss = -0.30

    # 其他 populate_* 方法

    # 最大追加次数
    max_entry_position_adjustment = 3
    # 说明：该参数后续会解释

    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            leverage: float, entry_tag: str | None, side: str,
                            **kwargs) -> float:
        # 预留大部分资金应对后续DCA
        return proposed_stake / self.max_dca_multiplier

    def adjust_trade_position(self, trade: Trade, current_time: datetime,
                              current_rate: float, current_profit: float,
                              min_stake: float | None, max_stake: float,
                              current_entry_rate: float, current_exit_rate: float,
                              current_entry_profit: float, current_exit_profit: float,
                              **kwargs
                              ) -> float | None | tuple[float | None, str | None]:
        """
        根据条件调整仓位。返回值为：  
        - 正数：增仓（买入）  
        - 负数：减仓（卖出）  
        - None：不操作  
        - 也可返回元组：(调仓数量，调仓原因标签)
        """

        # 若订单未执行完毕，则不调仓
        if trade.has_open_orders:
            return

        # 利润超5%，无成功平仓，部分平仓一半
        if current_profit > 0.05 and trade.nr_of_successful_exits == 0:
            return -(trade.stake_amount / 2), "half_profit_5%"

        # 利润在-5%到0%之间，不操作
        if current_profit > -0.05:
            return None

        # 获取行情蜡烛数据，判断价格是否上涨
        dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        previous_candle = dataframe.iloc[-2].squeeze()
        if last_candle["close"] < previous_candle["close"]:
            return

        # 获取已成功的入场订单（已全额成交）
        filled_entries = trade.select_filled_orders(trade.entry_side)
        count_of_entries = trade.nr_of_successful_entries

        # 最多再追加3次逐步增仓（共4次）
        # 初始买入1倍仓
        # 利润跌至-5%，补仓1.25倍，目标平均利润为-2.2%
        # 再跌-5%，补仓1.5倍
        # 再跌-5%，补仓1.75倍
        # 最终仓位：1 + 1.25 + 1.5 + 1.75 = 5.5倍
        try:
            stake_amount = filled_entries[0].stake_amount_filled
            stake_amount = stake_amount * (1 + (count_of_entries * 0.25))
            return stake_amount, "1/3rd_increase"
        except Exception:
            return None

        return None
```

### 调整调仓数额的算法

- 开仓价：用加权平均计算
- 平仓：不影响开仓价
- 部分平仓：用公式 `部分平仓数量 = 负数仓位 * 交易量 / 仓位总值`，不考虑盈亏，只由`trade.amount`和`trade.stake_amount`决定。

示例：  
买入2个SHITCOIN/USDT，开仓价50美元，仓位值100 USDT。  
涨到200美元，想卖出一半，即50美元部分，返回-50（表示减仓为50 USDT），  
计算：`50 * 2 / 100 = 1`（卖出1个SHITCOIN），  
范围判断：负仓位不能超过总仓位，否则会发生错误。

再如，当前价200美元，仓位价值400美元，想产生部分平仓，取出100美元：  
部分平仓数量 = `-100 * trade.stake_amount / trade.amount`  
其中：  
- `trade.amount`：持仓数量  
- `trade.stake_amount`：当前仓位值  

注意：止损仍按开仓价计算，不会随盈亏变化。

!!! 警告 “止损计算”
    止损是基于开仓时的价格，**不考虑平均价格**。  
    常规止损规则（不能向下）依然有效。  
    位置调整（调仓）时，止损会随之调整，但会受到限制。

```python
# 预定义导入

class StoplossDCA(IStrategy):

    position_adjustment_enable = True

    stoploss = -0.30

    # ...其他populate_*方法

    max_entry_position_adjustment = 3
    max_dca_multiplier = 5.5

    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            leverage: float, entry_tag: str | None, side: str,
                            **kwargs) -> float:
        # 留出大部分资金应对后续DCA
        return proposed_stake / self.max_dca_multiplier

    def adjust_trade_position(self, trade: Trade, current_time: datetime,
                              current_rate: float, current_profit: float,
                              min_stake: float | None, max_stake: float,
                              current_entry_rate: float, current_exit_rate: float,
                              current_entry_profit: float, current_exit_profit: float,
                              **kwargs
                              ) -> float | None | tuple[float | None, str | None]:
        """
        调仓逻辑，返回调整仓位（正值增仓，负值减仓）。
        """
        if trade.has_open_orders:
            return

        # 利润超过5%，卖出一半
        if current_profit > 0.05 and trade.nr_of_successful_exits == 0:
            return -(trade.stake_amount / 2), "half_profit_5%"

        # 利润在-5%到零之间，不操作
        if current_profit > -0.05:
            return

        # 判断价格趋势
        dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        previous_candle = dataframe.iloc[-2].squeeze()
        if last_candle["close"] < previous_candle["close"]:
            return

        filled_entries = trade.select_filled_orders(trade.entry_side)
        count_of_entries = trade.nr_of_successful_entries

        # 允许最多3次逐步增仓
        try:
            stake_amount = filled_entries[0].stake_amount_filled
            stake_amount = stake_amount * (1 + (count_of_entries * 0.25))
            return stake_amount, "1/3rd_increase"
        except Exception:
            return None
        return None
```

---

## 调整订单价格

`adjust_order_price()`回调允许你在蜡烛到来时，动态调整挂单价格（若订单已有且状态未完全成交或未超时），  
每个订单在每次蜡烛检测中最多调整一次。  
快速执行时间点：通常在下一蜡烛开始时。

注意：`custom_entry_price()` 与 `custom_exit_price()` 依然决定了订单初始价格，此回调主要用来追踪价格变化。

返回值为：  
- `None`：取消订单（订单会被取消，价格不再调整）  
- 其他价格：更新订单价格为该值，订单不会被取消。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):

    # ... 其他 populate_* 方法

    def adjust_order_price(
        self,
        trade: Trade,
        order: Order | None,
        pair: str,
        current_time: datetime,
        proposed_rate: float,
        current_order_rate: float,
        entry_tag: str | None,
        side: str,
        is_entry: bool,
        **kwargs,
    ) -> float | None:
        """
        动态调整订单价格的逻辑。返回希望的新价格。
        仅在订单已挂出（未完全成交/未超时）且在蜡烛范围内时调用。

        详见文档：https://www.freqtrade.io/en/latest/strategy-callbacks/

        若未实现该方法，则返回`current_order_rate`，订单维持不变。
        返回`None`则取消订单，不再挂单。

        :param pair: 当前分析的交易对
        :param trade: 交易对象
        :param order: 订单对象
        :param current_time: 当前时间（datetime）
        :param proposed_rate: 根据定价策略计算的建议价格
        :param current_order_rate: 当前订单价格
        :param entry_tag: 订单标签（买入标签）
        :param side: 'long'或'short'，交易方向
        :param is_entry: 是否为入场订单，若为出场订单，值为False
        :param **kwargs: 其他参数
        :return: 返回新价格（float），或None取消订单
        """

        # 示例：针对BTC/USDT，前10分钟内，若订单标签为long_sma200，限制订单价格
        if (
            is_entry
            and pair == "BTC/USDT" 
            and entry_tag == "long_sma200" 
            and side == "long" 
            and (current_time - timedelta(minutes=10)) <= trade.open_date_utc
        ):
            # 如果订单已成交超过一半，取消订单
            if order.filled > order.remaining:
                return None
            else:
                dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
                current_candle = dataframe.iloc[-1].squeeze()
                return current_candle["sma_200"]
        # 默认：保持原有订单价格
        return current_order_rate
```

!!! 警告 “与`adjust_*_price()`不兼容”  
    若同时实现`adjust_order_price()`和`adjust_entry_price()`/`adjust_exit_price()`，系统只调用`adjust_order_price()`，  
    不支持同时使用，二者应二选一或拆分。  
    不能混用，否则启动时会报错。

### 调整入场价格

`adjust_entry_price()`专用于调节挂出挂单的入场限价订单，行为与`adjust_order_price()`类似，只在入场订单时调用。  
开仓时的价格由`custom_entry_price()`设定，`adjust_entry_price()`可在开仓期间动态调整。

开仓时间持续不变（`trade.open_date_utc`保持第一次挂单时间），需确保在其他回调中考虑到此点。

### 调整出场价格

`adjust_exit_price()`用于调整止盈止损订单，行为类似，只在出场订单时调用。

## 杠杆回调

在允许杠杆交易的市场（如期货）中，返回值即为杠杆倍数（默认为1），否则此方法会被忽略。

杠杆效果：用本金乘以杠杆倍数，控制仓位。如：本金为500 USDT，杠杆3倍，实际仓位为1500 USDT。

超出`max_leverage`的值会被限制。  

```python
# 预定义导入

class AwesomeStrategy(IStrategy):
    def leverage(self, pair: str, current_time: datetime, current_rate: float,
                 proposed_leverage: float, max_leverage: float, entry_tag: str | None, side: str,
                 **kwargs) -> float:
        """
        调整杠杆倍数，仅在期货市场有效。

        :param pair: 交易对
        :param current_time: 当前时间
        :param current_rate: 当前价格
        :param proposed_leverage: 系统建议的杠杆
        :param max_leverage: 上限
        :param entry_tag: 标签（可选）
        :param side: 交易方向（long/short）
        :return: 实际杠杆倍数（1.0~max_leverage）
        """
        return 1.0
```

利润和止损/ROI计算都包括杠杆倍数。例如10%的止损在10倍杠杆下，实际触发点为1%下跌。

## 订单已成交回调

`order_filled()`在订单成交时调用，用于执行一些后续操作（如记录蜡烛最高价等）。

可支持所有订单类型（入场、出场、止损、调仓）。

```python
# 预定义导入

class AwesomeStrategy(IStrategy):
    def order_filled(self, pair: str, trade: Trade, order: Order, current_time: datetime, **kwargs) -> None:
        """
        订单成交后调用。可做状态跟踪、数据存储等。
        :param pair: 交易对
        :param trade: 交易对象
        :param order: 订单对象
        :param current_time: 当前时间
        :param **kwargs: 其他参数
        """
        # 获取行情蜡烛数据
        dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        # 若为第一次成功入场订单，存储最高价信息
        if (trade.nr_of_successful_entries == 1) and (order.ft_order_side == trade.entry_side):
            trade.set_custom_data(key="entry_candle_high", value=last_candle["high"])
        return None
```