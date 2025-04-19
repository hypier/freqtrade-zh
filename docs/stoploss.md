# 止损

`stoploss` 配置参数是一个比例损失值，当亏损达到该值时触发卖出。例如，值 `-0.10` 将在某笔交易的盈利下降至 -10% 时立即卖出。此参数为可选项。  
止损计算中会包含手续费，因此设置为 -10% 时，实际止损点正好比入场点低10%。

大部分策略文件已包含了最优的 `stoploss` 设置。

!!! Info
    本文件中提及的所有止损属性可以在策略中设置，也可以在配置中设置。  
    <ins>配置中的值将覆盖策略中的设置。</ins>

## 交易所/频率制止损

这些止损模式可以 *在交易所上* 或 *离线（off exchange）*。

可以通过以下值配置这些模式：

``` python
    'emergency_exit': 'market',
    'stoploss_on_exchange': False,
    'stoploss_on_exchange_interval': 60,
    'stoploss_on_exchange_limit_ratio': 0.99
```

仅支持以下交易所支持在交易所上设置止损订单，且并非所有交易所支持止损限价和市价同时使用。  
若只支持其中一种模式，则订单类型将被忽略。

| 交易所 | 止损类型 |
|---------|---------|
| Binance | 限价 |
| Binance Futures | 市价、限价 |
| Bingx | 市价、限价 |
| HTX | 限价 |
| kraken | 市价、限价 |
| Gate | 限价 |
| Okx | 限价 |
| Kucoin | 止损限价、止损市价 |
| Hyperliquid（仅期货） | 限价 |

!!! Note "紧凑型止损"
    <ins>使用交易所止损时，请勿设置过低或过紧的止损值！</ins>  
    设置过低或过紧的止损可能会增加订单未成交的风险，导致止损失效。

### stoploss_on_exchange 和 stoploss_on_exchange_limit_ratio

启用或禁用交易所上的止损。  
如果启用 *在交易所上* 设置止损，意味着在买入订单成交后立即在交易所上挂出止损限价订单。这可以防止市场突发崩盘带来的损失，因为订单在交易所内完成，无需网络传输。

如果 `stoploss_on_exchange` 使用限价订单，交易所需要两个价格：`stoploss_price` 和限价价格（Limit Price）。  
`stoploss` 定义了止损价格，限价应略低于此价格。  
如果交易所支持限价和市价止损订单，`stoploss` 的值将决定所用的止损类型。

示例计算：  
以100美元购买资产。  
止损价设为95美元，限价为 `95 * 0.99 = 94.05美元`，止损订单可以在95美元到94.05美元之间成交。

比如，假设启用交易所止损，且启用追踪止损，市场上涨时，机器人会自动取消之前的止损订单，并挂出一个更高的止损价格订单。

!!! Note
  如果启用了 `stoploss_on_exchange` 并且在交易所手动取消了止损订单，机器人会重新挂出一个新的止损订单。

### stoploss_on_exchange_interval

对于交易所上的止损，还有一个配置参数 `stoploss_on_exchange_interval`，它设置机器人检查并更新止损订单的时间间隔（秒）。  
该值避免机器人每5秒（每次循环）都刷新订单，以防被交易所封禁。  
默认值为60秒（1分钟）。  
如果你不小心取消了止损订单，也会在这个间隔重新挂出订单。

### stoploss_price_type

!!! Warning "仅“期货”支持"
    `stoploss_price_type` 仅适用于期货市场（在支持的交易所上）。  
    Freqtrade会在启动时校验此设置，若设置不支持，将无法启动。  
    支持的价格类型因交易所而异，请查阅你所在交易所支持的价格类型。

交易所期货上的止损订单可以基于不同的价格类型触发。  
这些价格在交易所术语中叫法可能不同，常见如 "last"（或 "contract price"）、"mark" 和 "index"。

支持的取值有 `"last"`、`"mark"` 和 `"index"`，Freqtrade会自动将其转换为对应API的类型，并在挂单时使用【在交易所上设置止损订单】（[#stoploss_on_exchange-and-stoploss_on_exchange_limit_ratio](#stoploss_on_exchange-and-stoploss_on_exchange_limit_ratio)）。

示例：  
买入资产价格为100美元。  
止损价设为95美元，限价为 `95 * 0.99 = 94.05美元`，限价订单会在95美元到94.05美元范围内成交。

如果启用了交易所止损，且开启了追踪止损，且市场向上走，机器人会自动取消之前的止损订单，并挂出一个止损价格更高的新订单。

### force_exit

`force_exit` 是一个可选值，默认为与 `exit` 相同，通常在通过Telegram或REST API发出 `/forceexit` 命令时使用。

### force_entry

`force_entry` 是一个可选值，默认为与 `entry` 相同，通常在通过Telegram或REST API发出 `/forceentry` 命令时使用。

### emergency_exit

`emergency_exit` 是一个可选值，默认为 `market`，在创建交易所止损订单失败时使用。  
如果未更改策略或配置文件中的设置，默认即为 `market`。

策略文件示例：

``` python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": True,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99
}
```

## 止损类型

目前机器人支持以下几种止损方式：

1. 静态止损。
2. 追踪止损。
3. 带有正收益的追踪止损。
4. 只有达到一定偏移后才开始追踪的追踪止损。
5. [自定义止损函数](strategy-callbacks.md#custom-stoploss)

### 静态止损

非常简单，你定义一个比例x（即价格的x * 100%），一旦亏损超过此比例，资产就会被卖出。

示例：

``` python
    stoploss = -0.10
```

示意：  
假设机器人以100美元买入资产。  
设置止损为-10%，即一旦资产跌破90美元，止损触发。

### 追踪止损

此模式的初始值为 `stoploss`，就像定义静态止损一样。  
启用方法：

``` python
    stoploss = -0.10
    trailing_stop = True
```

开启后，机器人会自动调整止损点，每当资产价格上涨时，止损点也会跟着上移。

示意：  
买入价100美元，止损-10%（90美元）；资产涨到102美元时，止损调整为-10%即91.8美元（即102 * 0.9）；资产跌至101美元时，止损依然为91.8美元，不会下降。

总结：止损点始终维持在观察到的最高价格的-10%。

### 带正收益的追踪止损

你可以定义在亏损时的静态止损（如-10%），但当盈利达到某个正偏移（比如0.1%）后，系统将使用不同的追踪止损。

例如：默认止损为-10%，当盈利达到0.1%后，将启用新的追踪止损策略。

!!! Note
    如果你只希望在实现盈亏平衡（打平）时调整止损（大多数用户的需求），请参考下一节【达到一定偏移后才追踪】（#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset）。

两个参数都需要启用 `trailing_stop`，并设置 `trailing_stop_positive` 和相关偏移：

``` python
    stoploss = -0.10
    trailing_stop = True
    trailing_stop_positive = 0.02
    trailing_stop_positive_offset = 0.0
    trailing_only_offset_is_reached = False  # 默认为否，此处可不必设置
```

示意：  
买入价格100美元，止损-10%（90美元）；资产上涨到102美元后，止损调整为-2%即99.96美元（锁定在最高价的-2%范围内）；如果资产涨到101美元，止损依然为99.96美元，一旦资产价格跌破99.96美元，则触发卖出。

`0.02` 表示-2%的止损。

在此之前，`stoploss` 仍用于追踪止损。

!!! Tip "用偏移量设置止损"
    你可以用 `trailing_stop_positive_offset` 来确保新追踪止损在盈利状态。将 `trailing_stop_positive_offset` 设置得高于 `trailing_stop_positive`，这样，第一次调整的止损点就已锁定部分利润。

示例（简化）：

``` python
    stoploss = -0.10
    trailing_stop = True
    trailing_stop_positive = 0.02
    trailing_stop_positive_offset = 0.03
```

流程：  
以100美元买入，止损-10%（即90美元）；资产涨到102美元后，止损被提升到比最高价的-10%还高一点；如果涨到103.5美元（超过偏移），则止损调整为-2%，即101.43美元；资产跌到102美元时，止损仍为101.43美元。

### 仅在达到一定偏移后才追踪止损

你可以设定在资产价格达到一定偏移（如买入价的3%）之前，不启用追踪止损，直到偏移值达到后才开始跟随。

如果设置 `trailing_only_offset_is_reached = True`，则追踪止损只在达到偏移后生效，之前保持静态 `stoploss`。  
若设为 `False`，资产价格一升高到初始买入价以上，追踪止损即开始跟随。

此参数可以配合或不配合 `trailing_stop_positive` 使用，但必须设置 `trailing_stop_positive_offset` 作为偏移。

示例（偏移为买入价+3%）：

``` python
    stoploss = -0.10
    trailing_stop = True
    trailing_stop_positive = 0.02
    trailing_stop_positive_offset = 0.03
    trailing_only_offset_is_reached = True
```

示意：  
买入价100美元，止损-10%（90美元）；  
只有资产涨到103美元（偏移点）之后，追踪止损才激活，止损调整为-2%；  
如果涨到超过偏移（如103.5美元），止损会调整为-2%的比例（即102美元左右）；  
资产跌到102美元以下，止损触发卖出。

!!! Tip
    为避免频繁挂单导致高交易成本，建议将 `trailing_stop_positive_offset` 设置得低于预期最小ROI，否则会优先实现最小ROI。

## 止损与杠杆

止损代表“此笔交易的风险”，即设定一个亏损百分比，比如10%的止损意味着你愿意承受最多10美元（在100美元交易中）亏损，达到此亏损时会触发卖出。

使用杠杆时，原则相同——止损定义了交易的风险（你愿意亏损的最大额度）。  
例如：10倍杠杆下，10%的止损实际上等于1%的价格变动触发卖出。  
假设自己的本金为100美元，利用10倍杠杆，实际交易金额为1000美元。  
当价格下跌1%时，自己亏损了10美元，也就是说亏损额度达到最大风险，止损会触发。

请确保明白此点，避免设置过紧的止损（在10倍杠杆下，10%的风险可能太小，让交易“喘息”空间不足）。

## 修改开仓交易的止损

可以通过修改配置或策略中的值，以及使用 `/reload_config` 指令应用更改（也可完全停止并重启机器人）。

新设置的止损会立即应用到已开仓的仓位（系统会记录相关日志信息）。

### 限制

如果启用 `trailing_stop`，且止损已被调整过，或者启用了 [Edge](edge.md) 功能（会根据市场变动重新计算止损），则无法再修改止损值。