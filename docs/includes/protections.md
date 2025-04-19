## 保护措施

保护措施旨在通过在特定情况下临时停止交易，保护您的策略免受意外事件和市场条件的影响，可以针对单个交易对或全部交易对生效。所有保护措施的结束时间均向上取整至下一根K线，以避免在K线内部发生突发性、不可预料的买入行为。

!!! tip "使用提示"
    并非所有保护措施都适用于所有策略，参数需要根据您的策略进行调优，以提升表现。

    每个保护措施可以用不同参数多次配置，从而实现不同层级的保护（短期/长期）。

!!! note "回测"
    保护措施支持通过回测和超参数优化（hyperopt）进行测试，但必须通过使用 `--enable-protections` 参数显式启用。

### 可用的保护措施

* [`StoplossGuard`](#stoploss-guard) 若在一定时间范围内发生一定数量的止损，会停止交易。
* [`MaxDrawdown`](#maxdrawdown) 达到最大回撤时停止交易。
* [`LowProfitPairs`](#low-profit-pairs) 锁定利润较低的交易对。
* [`CooldownPeriod`](#cooldown-period) 在刚平仓后，不立即重新进入，设定冷却时间。

### 所有保护措施的通用设置

| 参数 | 描述 |
|--------|--------|
| `method` | 使用的保护措施名称。<br>**数据类型：** 字符串，从[可用的保护措施](#available-protections)中选择 |
| `stop_duration_candles` | 设定锁定的持续时间（以K线为单位）。<br>**数据类型：** 正整数（以K线为单位） |
| `stop_duration` | 保护措施的锁定时间（以分钟为单位）。<br>不能与 `stop_duration_candles` 同时使用。<br>**数据类型：** 浮点数（分钟） |
| `lookback_period_candles` | 只考虑在过去 `lookback_period_candles` 根K线内完成的交易。这一设置可能被某些保护措施忽略。<br>**数据类型：** 正整数（以K线为单位） |
| `lookback_period` | 只考虑在 `current_time - lookback_period` 之后完成的交易。<br>不能与 `lookback_period_candles` 同时使用。<br>此设置可能被某些保护措施忽略。<br>**数据类型：** 浮点数（分钟） |
| `trade_limit` | 发生的交易次数达到此数以上，触发保护（并非所有保护都使用此参数）。<br>**数据类型：** 正整数 |
| `unlock_at` | 设定恢复正常交易的时间（24小时制字符串，例如 "HH:MM"）。<br>**数据类型：** 字符串 |

!!! note "持时 durations"
    `stop_duration*` 及 `lookback_period*` 可定义为分钟或K线数。为了在测试不同时间框架时有更大灵活性，以下示例中都采用“K线”作为单位。

#### Stoploss Guard（止损保护）

`StoplossGuard` 会检测在 `lookback_period`（分钟或K线）内是否发生了指定数量（`trade_limit`）的止损交易。如果达到或超过此数量，将在 `stop_duration`（分钟或K线）内停止交易（或直到设定的 `unlock_at` 时间）。

此功能在所有交易对中有效，除非将 `only_per_pair` 设置为 `True`，此时只会检测单个交易对。

类似地，默认情况下会考虑所有类型的交易（多头和空头）。如果是在期货策略中，设置 `only_per_side` 可以让策略只考虑某一交易方向（如仅空头或仅多头），这样即使某一侧连续多次止损，另一侧仍可继续交易。

`required_profit` 用于设定止损的利润（或亏损）阈值，默认值为0.0，意味着所有亏损止损都会被触发门控。

以下示例在过去24根K线内，如果触发4次止损，将暂停所有交易对4根K线（即持续停止）：

``` python
@property
def protections(self):
    return [
        {
            "method": "StoplossGuard",
            "lookback_period_candles": 24,
            "trade_limit": 4,
            "stop_duration_candles": 4,
            "required_profit": 0.0,
            "only_per_pair": False,
            "only_per_side": False
        }
    ]
```

!!! note
    `StoplossGuard` 会统计所有结果为 `"stop_loss"`、`"stoploss_on_exchange"` 和 `"trailing_stop_loss"` 且利润为负的交易。
    `trade_limit` 和 `lookback_period` 需要根据您的策略调整。

#### MaxDrawdown（最大回撤）

`MaxDrawdown` 使用在 `lookback_period`（分钟或K线）内的所有交易，计算最大回撤比例。当最大回撤低于设定阈值 `max_allowed_drawdown` 时，将在 `stop_duration`（分钟或K线）内停止交易（假设市场需要一些时间恢复）。

比如，在过去48根K线（即两天）内，所有交易累计超过20%的最大回撤时，若满足最低交易数（`trade_limit`），则会停止交易12根K线。

示例：

``` python
@property
def protections(self):
    return  [
        {
            "method": "MaxDrawdown",
            "lookback_period_candles": 48,
            "trade_limit": 20,
            "stop_duration_candles": 12,
            "max_allowed_drawdown": 0.2
        },
    ]
```

#### LowProfitPairs（低利润交易对）

`LowProfitPairs` 会检测在 `lookback_period`（分钟或K线）范围内，某个交易对的所有交易的总体利润率。如果低于 `required_profit`，则该交易对会被锁定，停用指定时长（`stop_duration`或`stop_duration_candles`，或直到`unlock_at`时间）。

在期货策略中，设置 `only_per_side` 会让策略只考虑某一方向（买入或卖出），并只锁定该方向，从而允许另一方向继续交易。

以下示例指定，在过去6根K线（代表一小时内）中，如果某交易对的利润率低于2%（且最低4笔交易），则暂停交易60分钟：

``` python
@property
def protections(self):
    return [
        {
            "method": "LowProfitPairs",
            "lookback_period_candles": 6,
            "trade_limit": 2,
            "stop_duration": 60,
            "required_profit": 0.02,
            "only_per_pair": False,
        }
    ]
```

#### Cooldown Period（冷却时间）

`CooldownPeriod` 在交易对平仓后，锁定这个对（即暂停一定时间）以避免频繁交易。解除锁定后，此交易对可在 `stop_duration`（分钟或K线）后重新进入。

示例：平仓后，等待2根K线（即冷却2小时）再重新开仓：

``` python
@property
def protections(self):
    return  [
        {
            "method": "CooldownPeriod",
            "stop_duration_candles": 2
        }
    ]
```

!!! note
    该保护仅在交易对级别有效，不会全局锁定所有交易对。
    它不考虑 `lookback_period`，因为只关注最新一次交易。

### 完整保护措施示例

所有保护措施可以任意组合，也可以用不同参数组合，形成对表现不佳交易对的多层防护。所有保护措施按照定义的顺序依次评估。

以下示例假设时间周期为1小时：

- 在平仓后，为每个交易对设置额外的5根K线（`CooldownPeriod`），给予其他交易对机会入场。
- 如果在过去两天（48根K线）内发生了20次交易，且造成最大回撤超过20%，则暂停所有交易4小时（`MaxDrawdown`）。
- 在过去24根K线内，如果某交易对触发了超过4次止损，将暂停交易。
- 如果在过去6小时（6根K线）内，某交易对有2笔交易，且总体利润率低于2%（`LowProfitPairs`），则暂停该交易对。
- 在过去24小时（24根K线）内，某交易对的利润低于1%（`<1%`）且交易数至少为4笔，则暂停2根K线。

示例代码如下：

``` python
from freqtrade.strategy import IStrategy

class AwesomeStrategy(IStrategy):
    timeframe = '1h'
    
    @property
    def protections(self):
        return [
            {
                "method": "CooldownPeriod",
                "stop_duration_candles": 5
            },
            {
                "method": "MaxDrawdown",
                "lookback_period_candles": 48,
                "trade_limit": 20,
                "stop_duration_candles": 4,
                "max_allowed_drawdown": 0.2
            },
            {
                "method": "StoplossGuard",
                "lookback_period_candles": 24,
                "trade_limit": 4,
                "stop_duration_candles": 2,
                "only_per_pair": False
            },
            {
                "method": "LowProfitPairs",
                "lookback_period_candles": 6,
                "trade_limit": 2,
                "stop_duration_candles": 60,
                "required_profit": 0.02
            },
            {
                "method": "LowProfitPairs",
                "lookback_period_candles": 24,
                "trade_limit": 4,
                "stop_duration_candles": 2,
                "required_profit": 0.01
            }
        ]
    # ...
```