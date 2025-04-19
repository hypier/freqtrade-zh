# 交易对象

## 交易

频繁交易（freqtrade）进行仓位操作时，会将相关信息存储在一个 `Trade` 对象中——该对象会被持久化到数据库中。  
这是 freqtrade 的核心概念之一，也是你在许多文档章节中会遇到的内容，通常这些部分都会引导你到该位置。

这个对象在许多[策略回调](strategy-callbacks.md)中会被传递给策略。  
传递给策略的对象不能被直接修改。根据回调函数的结果，可能会进行间接修改。

## Trade - 可用属性

每个交易实例都拥有以下属性/字段，可以通过 `trade.<property>` 访问（例如 `trade.pair`）。

|  属性名 | 数据类型 | 说明 |
|------------|-------------|-------------|
| `pair` | string | 该交易的货币对。 |
| `is_open` | boolean | 交易当前是否为开启状态，或已结束。 |
| `open_rate` | float | 进入交易的价格（若有交易调整，则为平均入场价）。 |
| `close_rate` | float | 平仓价格——当 `is_open = False` 时才有值。 |
| `stake_amount` | float | 以 Stake（或报价货币）单位的金额。 |
| `amount` | float | 目前持有的资产/基础货币的数量。在初始订单成交前为 0.0。 |
| `open_date` | datetime | 开仓时间（**应使用 `open_date_utc` 替代**） |
| `open_date_utc` | datetime | 开仓时间（UTC） |
| `close_date` | datetime | 平仓时间（**应使用 `close_date_utc` 替代**） |
| `close_date_utc` | datetime | 平仓时间（UTC） |
| `close_profit` | float | 平仓时的相对利润，0.01 表示 1%。 |
| `close_profit_abs` | float | 平仓时的绝对利润（以 Stake 货币计）。 |
| `leverage` | float | 此交易所使用的杠杆，现货市场默认为 1.0。 |
| `enter_tag` | string | 通过 `enter_tag` 列在数据框中提供的标签。 |
| `is_short` | boolean | 是否为空头交易，True 表示空头，False 表示多头。 |
| `orders` | Order[] | 附加到此交易的订单列表（包括已成交和已取消的订单）。 |
| `date_last_filled_utc` | datetime | 最后一次成交订单时间。 |
| `entry_side` | "buy" / "sell" | 进入交易的订单方向。 |
| `exit_side` | "buy" / "sell" | 将导致平仓或减仓的订单方向。 |
| `trade_direction` | "long" / "short" | 交易方向的文字描述——多头或空头。 |
| `nr_of_successful_entries` | int | 成功（已成交）入仓订单的数量。 |
| `nr_of_successful_exits` | int | 成功（已成交）平仓订单的数量。 |
| `has_open_orders` | boolean | 交易是否有未平仓订单（不包括止损订单）。 |

## 类方法

以下为类方法，返回常规信息，通常会涉及对数据库的显式查询。  
可以用 `Trade.<method>` 调用，例如：  
`open_trades = Trade.get_open_trade_count()`

！！！警告 "回测/超参数调优"  
大多数方法在回测/超参数调优和实时/模拟操作中均可使用。  
在回测中仅限于在[策略回调](strategy-callbacks.md)中使用，  
在 `populate_*()` 方法中使用不被支持，可能会得出错误结果。

### get_trades_proxy

当策略需要获取某些已存在（开仓或平仓）交易的信息时，建议使用 `Trade.get_trades_proxy()`。

用法示例：

``` python
from freqtrade.persistence import Trade
from datetime import timedelta

# ...
trade_hist = Trade.get_trades_proxy(pair='ETH/USDT', is_open=False, open_date=current_date - timedelta(days=2))
```

`get_trades_proxy()` 支持以下关键字参数。所有参数都是可选的，调用时不传参将返回数据库中的所有交易列表：

* `pair` 例如：`pair='ETH/USDT'`
* `is_open` 例如：`is_open=False`
* `open_date` 例如：`open_date=current_date - timedelta(days=2)`
* `close_date` 例如：`close_date=current_date - timedelta(days=5)`

### get_open_trade_count

获取当前未平仓交易的数量。

``` python
from freqtrade.persistence import Trade
# ...
open_trades = Trade.get_open_trade_count()
```

### get_total_closed_profit

获取到目前为止策略所有已平仓交易累计产生的利润总和。  
会汇总所有已平仓交易的 `close_profit_abs`。

``` python
from freqtrade.persistence import Trade

# ...
profit = Trade.get_total_closed_profit()
```

### total_open_trades_stakes

获取目前交易中的总 stake_amount。

``` python
from freqtrade.persistence import Trade

# ...
stake_sum = Trade.total_open_trades_stakes()
```

### get_overall_performance

获取整体表现，类似于 Telegram `/performance` 命令。

``` python
from freqtrade.persistence import Trade

# ...
if self.config['runmode'].value in ('live', 'dry_run'):
    performance = Trade.get_overall_performance()
```

示例返回值：假设 ETH/BTC 有 5 笔交易，总利润为 1.5%（比例为 0.015）。

``` json
{"pair": "ETH/BTC", "profit": 0.015, "count": 5}
```

## Order 对象

`Order` 对象表示交易所中的订单（或在模拟模式下的模拟订单）。  
一个 `Order` 始终关联其对应的 [`Trade`](#trade-object)，且在交易上下文中才有意义。

### Order - 可用属性

一个订单对象通常会绑定到某个交易中。  
大部分属性可能会是 `None`，因为它们依赖于交易所的响应。

|  属性名 | 数据类型 | 说明 |
|------------|-------------|-------------|
| `trade` | Trade | 此订单关联的交易对象 |
| `ft_pair` | string | 订单对应的货币对 |
| `ft_is_open` | boolean | 订单是否已成交（已填充）？ |
| `order_type` | string | 订单类型，通常为 market、limit 或 stoploss（由交易所定义） |
| `status` | string | 订单状态（由 ccxt 定义），常见的有 open、closed、expired 或 canceled |
| `side` | string | 买（buy）或卖（sell） |
| `price` | float | 订单挂单价格 |
| `average` | float | 订单成交的平均价格 |
| `amount` | float | 基础货币的数量 |
| `filled` | float | 已成交的数量（基础货币） |
| `remaining` | float | 剩余未成交的数量 |
| `cost` | float | 订单成本（通常为 average * filled，期货交易可能包含杠杆，可能以合约数量显示） |
| `stake_amount` | float | 本订单使用的 stake 数量（*在 2023.7 版本中添加*） |
| `stake_amount_filled` | float | 已成交的 stake 数量（*在 2024.11 版本中添加*） |
| `order_date` | datetime | 订单创建时间（**应使用 `order_date_utc` 替代**） |
| `order_date_utc` | datetime | 订单创建时间（UTC） |
| `order_fill_date` | datetime | 订单成交时间（**应使用 `order_fill_utc` 替代**） |
| `order_fill_utc` | datetime | 订单成交时间（UTC） |