## Pairlists和Pairlist处理器

Pairlist处理器定义了机器人应该交易的交易对（pairlist）列表。它们在配置设置的`pairlists`部分进行配置。

在您的配置中，可以使用静态交易对列表（由 [`StaticPairList`](#static-pair-list) 处理器定义）和动态交易对列表（由 [`VolumePairList`](#volume-pair-list) 和 [`PercentChangePairList`](#percent-change-pair-list) 处理器定义）。

此外， [`AgeFilter`](#agefilter)、[`PrecisionFilter`](#precisionfilter)、[`PriceFilter`](#pricefilter)、[`ShuffleFilter`](#shufflefilter)、[`SpreadFilter`](#spreadfilter) 和 [`VolatilityFilter`](#volatilityfilter) 也作为交易对过滤器，移除某些交易对和/或调整它们在pairlist中的位置。

如果使用多个Pairlist处理器，它们会串联执行，所有处理器共同组成机器人用于交易和回测的最终pairlist。Pairlist处理器按配置顺序依次执行。您可以将`StaticPairList`、`VolumePairList`、`ProducerPairList`、`RemotePairList`、`MarketCapPairList`或`PercentChangePairList`作为起始的Pairlist处理器。

未激活的市场始终会从最终的pairlist中移除。同时，明确列入黑名单的交易对（在`pair_blacklist`配置项中）也会被永远移除。

### 交易对黑名单

交易对黑名单（配置在`exchange.pair_blacklist`中）禁止某些交易对参与交易。例如，仅排除`DOGE/BTC`这一对。

黑名单还支持通配符（正则表达式风格）——比如`BNB/.*`将排除所有以BNB开头的交易对。您也可以使用`.*DOWN/BTC`或`.*UP/BTC`等，排除杠杆型Token（请查看您的交易所的交易对命名规则！）

### 常用的交易对列表处理器

* [`StaticPairList`](#static-pair-list)（默认，未配置时使用）
* [`VolumePairList`](#volume-pair-list)
* [`PercentChangePairList`](#percent-change-pair-list)
* [`ProducerPairList`](#producerpairlist)
* [`RemotePairList`](#remotepairlist)
* [`MarketCapPairList`](#marketcappairlist)
* [`AgeFilter`](#agefilter)
* [`FullTradesFilter`](#fulltradesfilter)
* [`OffsetFilter`](#offsetfilter)
* [`PerformanceFilter`](#performancefilter)
* [`PrecisionFilter`](#precisionfilter)
* [`PriceFilter`](#pricefilter)
* [`ShuffleFilter`](#shufflefilter)
* [`SpreadFilter`](#spreadfilter)
* [`RangeStabilityFilter`](#rangestabilityfilter)
* [`VolatilityFilter`](#volatilityfilter)

!!! Tip "测试交易对列表"
    配置的pairlist可能比较复杂，不易调试。建议使用 [`test-pairlist`](utils.md#test-pairlist) 工具子命令快速验证配置效果。

#### 静态交易对列表（Static Pair List）

默认使用`StaticPairList`方法，它从配置中加载静态定义的交易对白名单。该pairlist也支持通配符（正则表达式风格），比如`.*/BTC`将包含所有以BTC为基础货币的交易对。

它通过`exchange.pair_whitelist`和`exchange.pair_blacklist`两个配置项控制。例如，下例中机器人会交易BTC/USDT和ETH/USDT，但不交易BNB/USDT。

两个`pair_*list`参数支持正则表达式——比如`.*/USDT`，将启用所有未在黑名单中的交易对。

```json
"exchange": {
    "name": "...",
    // ...
    "pair_whitelist": [
        "BTC/USDT",
        "ETH/USDT",
        // ...
    ],
    "pair_blacklist": [
        "BNB/USDT",
        // ...
    ]
},
"pairlists": [
    {"method": "StaticPairList"}
],
```

默认情况下，只允许已激活的交易对。若要跳过市场激活状态验证，可以在`StaticPairList`配置中设置`"allow_inactive": true`。这对于回测过期的交易对（如季度现货市场）非常有用。

在后续位置（比如VolumePairList之后）使用时，`pair_whitelist`中的所有交易对会被加入到pairlist末尾。

#### 按交易量排序的交易对列表（Volume Pair List）

`VolumePairList`根据交易量对交易对进行排序/筛选，选出前`number_assets`个交易量最高的交易对（排序依据`sort_key`，仅支持`quoteVolume`）。

在作为非首个处理器（如在静态pairlist和其他过滤器之后）使用时，它会考虑前面的处理器输出的交易对列表，将其排序和筛选以交易量为依据。

若作为链条中的首个处理器使用，配置中的`pair_whitelist`会被忽略，此时`VolumePairList`会从所有符合基础货币的市场中选出顶尖的交易对。

`refresh_period`定义pairlist每次刷新时间（秒），默认1800秒（30分钟）。缓存机制只针对用于生成pairlist的操作，过滤器（非第一位置）不使用缓存（除提前缓存蜡烛数据以加快稳定状态下的处理）。

`VolumePairList`默认依赖于ccxt库提供的交易所Ticker数据：

* `quoteVolume`为在过去24小时内被交易（买入或卖出）的报价（基础货币）总量。

```json
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume",
        "min_value": 0,
        "max_value": 8000000,
        "refresh_period": 1800
    }
],
```

可以用`min_value`定义最低交易量门槛，以过滤掉成交量低于此值的交易对。也可以用`max_value`定义最高交易量门槛，过滤掉成交量高于此值的交易对。

##### VolumePairList 进阶模式

`VolumePairList`还能以高级模式对一定时间范围（基于指定蜡烛大小）内的交易量进行加权计算。它会利用交易所的历史蜡烛数据，计算典型价格（`(open + high + low)/3`），再乘以蜡烛交易量，最后求和得出该范围内的`quoteVolume`。这实现了长时间跨度大蜡烛或短时间跨度小蜡烛的不同平滑度。

可以通过`lookback_days`参数设置，表示用几天的蜡烛数据进行回溯。例如，设置`lookback_days: 7`，则根据最近7天的交易数据构建pairlist：

```json
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume",
        "min_value": 0,
        "refresh_period": 86400,
        "lookback_days": 7
    }
],
```

!!! 警告 "范围回溯和刷新周期"
    若同时启用`lookback_days`和`lookback_timeframe`，`refresh_period`不能小于蜡烛的时间长度（秒），否则会频繁请求交易所API，造成效率问题。

!!! 警告 "使用范围回溯的性能影响"
    与`lookback`组合使用时，基于范围的交易量计算会消耗大量时间和资源，因为会下载所有可交易对的蜡烛数据，建议优先结合`VolumeFilter`缩小pairlist范围，再进行范围内的交易量计算。

??? Tip "不支持的交易所"
    某些交易所（如Gemini）不支持API直接提供24h交易量，此时可以通过用蜡烛数据模拟交易量。例如每次只统计过去1天的蜡烛，从而近似估算24h交易量，但每次刷新只会发生一次。

示例配置（使用1小时蜡烛，回溯7天）：

```json
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume",
        "min_value": 0,
        "refresh_period": 86400,
        "lookback_days": 7
    }
],
```

也可以使用`lookback_timeframe`和`lookback_period`参数，构建滚动期间的交易量，比如3天（72个1小时蜡烛）：

```json
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume",
        "min_value": 0,
        "refresh_period": 3600,
        "lookback_timeframe": "1h",
        "lookback_period": 72
    }
],
```

### 比例变动百分比（PercentChangePairList）

`PercentChangePairList`根据交易对价格在特定时间段的百分比变动进行筛选和排序。支持比24小时变动（或其他自定义时间段）更关注发生重大涨跌的资产。

配置选项包括：

* `number_assets`: 按百分比变动排序，选出前N个交易对。
* `min_value`:百分比变动最低阈值，小于此值的交易对会被排除。
* `max_value`:百分比变动最高阈值，大于此值的交易对会被排除。
* `sort_direction`: 排序方向，`asc`（升序）或`desc`（降序）。
* `refresh_period`: 刷新间隔（秒），默认1800秒（30分钟）。
* `lookback_days`: 回溯天数，配合`lookback_timeframe`使用，默认为1天。
* `lookback_timeframe`: 回溯时间段（如`1h`、`1d`等）。
* `lookback_period`: 具体的蜡烛周期数。

在作为链条中其他处理器之后使用，会作用于前一处理器输出，作为筛选所用。如果作为第一个处理器，则会从所有市场筛选符合基础货币的交易对。

`PercentChangePairList`利用ccxt提供的行情数据，计算方式为：

$$ Percent Change = (\frac{Current Close - Previous Close}{Previous Close}) * 100 $$

??? 警告 "不支持的交易所"
    某些交易所（如HTX）没有API直接提供24小时百分比变动，此时可以用蜡烛数据模拟其百分比变动。配置示例如下（每次仅刷新一次）：

```json
"pairlists": [
    {
        "method": "PercentChangePairList",
        "number_assets": 20,
        "min_value": 0,
        "refresh_period": 86400,
        "lookback_days": 1
    }
],
```

示例配置（从行情数据读取）：

```json
"pairlists": [
    {
        "method": "PercentChangePairList",
        "number_assets": 15,
        "min_value": -10,
        "max_value": 50
    }
],
```

此配置选择过去24小时内变动最大的前15个交易对，且其变动在-10%到50%之间。

另一示例（基于蜡烛数据）：

```json
"pairlists": [
    {
        "method": "PercentChangePairList",
        "number_assets": 15,
        "sort_key": "percentage",
        "min_value": 0,
        "refresh_period": 3600,
        "lookback_timeframe": "1h",
        "lookback_period": 72
    }
],
```

采用过去3天（72个1小时蜡烛）数据计算百分比变动。变动公式为：

$$ Percent Change = (\frac{Current Close - Previous Close}{Previous Close}) * 100 $$

!!! 警告 "范围回溯和刷新周期"
    若使用`lookback_days`和`lookback_timeframe`，`refresh_period`不能小于蜡烛时间长度（秒），否则会频繁请求API。

!!! 警告 "性能影响"
    使用范围回溯计算百分比变动会耗费大量资源，建议结合`PercentChangeFilter`缩小pairlist范围后再计算。

!!! 提示 "回测"
    `PercentChangePairList`不支持回测模式。

#### ProducerPairList（生产者交易对列表）

通过`ProducerPairList`可以复用某个[生产者](producer-consumer.md)的交易对列表，而无需在每个消费者配置中重复定义。

此功能需要在`consumer`模式下运行。

它会校验活跃交易对的状态，确保不会尝试交易尚不可用的市场。

可以通过`number_assets`限制输出交易对数，设为0或省略表示复用所有生产者当前有效的交易对。

```json
"pairlists": [
    {
        "method": "ProducerPairList",
        "number_assets": 5,
        "producer_name": "default"
    }
],
```

!!! Tip "组合使用"
    此pairlist可以结合其他pairlist和过滤器，进一步减小范围，也可以作为“附加”交易对列表。  
    `ProducerPairList`可以多次叠加，合并多个生产者的交易对。  
    复杂策略中，生产者可能不能提供所有交易对，需要根据具体情况匹配。

#### RemotePairList（远程交易对列表）

允许从远程服务器或本地存储的json文件中动态获取交易对列表，方便更新和定制。

定义在配置中的`pairlists`部分，使用如下参数：

```json
"pairlists": [
    {
        "method": "RemotePairList",
        "mode": "whitelist",
        "processing_mode": "filter",
        "pairlist_url": "https://example.com/pairlist",
        "number_assets": 10,
        "refresh_period": 1800,
        "keep_pairlist_on_failure": true,
        "read_timeout": 60,
        "bearer_token": "my-bearer-token",
        "save_to_file": "user_data/filename.json"
    }
]
```

* `mode`（可选）：指定是用作黑名单（`blacklist`）还是白名单（`whitelist`），默认为`whitelist`。
* `processing_mode`（可选）：控制pairlist处理方式，`filter`（默认）表示筛选交集，`append`表示合并两个列表。
* `pairlist_url`：远程服务器URL（或本地文件路径，file:///开头）。
* `save_to_file`：如果提供，都会把处理后pairlist保存到指定文件（JSON格式）。默认为不保存。

示例：多个机器人共享一个pairlist文件

```json
"pairlists": [
    {
        "method": "RemotePairList",
        "mode": "whitelist",
        "pairlist_url": "https://example.com/pairlist",
        "number_assets": 10,
        "refresh_period": 1800,
        "keep_pairlist_on_failure": true,
        "save_to_file": "user_data/filename.json"
    }
]
```

Bot1保存pairlist为文件，然后Bot2或其他机器人可以通过文件路径加载。

用户需要提供能返回如下格式的JSON文件（获得市场对列表）：

```json
{
    "pairs": ["XRP/USDT", "ETH/USDT", "LTC/USDT"],
    "refresh_period": 1800
}
```

`pairs`为交易对字符串数组，`refresh_period`是缓存时间（秒，选填）。  
`keep_pairlist_on_failure`默认为`true`，表示如果远程服务器不可用，则维持上次的pairlist。

`read_timeout`为请求超时时间（秒），默认为60秒。  
`bearer_token`会加入请求授权中。

??? 注意
    服务器发生错误时，若`keep_pairlist_on_failure`为`true`，会保持上次的pairlist；若为`false`，则返回空pairlist。

#### MarketCapPairList（市值排序的交易对列表）

根据CoinGecko的市值排名排序筛选交易对。

示例配置：

```json
"pairlists": [
    {
        "method": "MarketCapPairList",
        "number_assets": 20,
        "max_rank": 50,
        "refresh_period": 86400,
        "categories": ["layer-1"]
    }
]
```

* `number_assets`：返回的最大交易对数。
* `max_rank`：市场排名上限，只有排名在此范围内的币会被考虑（注意，某些币可能没有交易对或不在所选类别中）。
* `refresh_period`：排名数据的刷新间隔（秒）；默认为86400秒（1天）。
* `categories`：CoinGecko分类（网址示例：[https://www.coingecko.com/en/categories](https://www.coingecko.com/en/categories)），为空数组表示不过滤类别。

注意事项：
- `max_rank` > 250基本没必要，可能会引发频繁API调用限制。
- 使用`categories`时要确保类别名称正确，否则会出错并打印可用类别。

### AgeFilter

过滤列表中列出交易对的注册上市时间。默认`min_days_listed=10`，意味着只保留已上市至少10天的交易对；`max_days_listed`限制最大上市天数。

新上市交易对可能经历巨大波动，机器人在其价格稳定之前可能发生亏损。此过滤器可以让机器人忽略刚上市或已出现过长时间的交易对。

### FullTradesFilter

当交易槽已满（`max_open_trades`未设为`-1`），只保持在交易中的（实质上在用“交易中”状态过滤器）交易对列表，减少不必要的指标计算，加快处理速度。当交易槽空出（交易关闭或`max_open_trades`调整）后，过滤器恢复正常。

建议此过滤器放在第二位（紧跟主pairlist后），确保在满仓时不加载额外数据。

!!! Warning "回测"
    `FullTradesFilter`不支持回测模式。

### OffsetFilter（偏移过滤器）

用`offset`参数对pairlist进行偏移。例如，跳过前10个交易对，再返回接下来的20个（即第10到30项）：

```json
"pairlists": [
    // ...
    {
        "method": "OffsetFilter",
        "offset": 10,
        "number_assets": 20
    }
],
```

!!! Warning
    使用`OffsetFilter`时，可能会因“不同刷新时间间隔”而导致碰撞重复（即多个机器人可能会选中相同的交易对），不能保证完全不重叠。

!!! Note
    offset总长超过pairlist长度，会导致空pairlist。

### PerformanceFilter（性能表现过滤器）

基于过去交易性能排序，优先选择表现良好的交易对，规则如下：

1. 正向表现。
2. 尚无已平仓交易。
3. 负向表现。

用`minutes`参数限制仅考虑过去几分钟（滚动窗口）内的表现，未设置或设为0则考虑全部历史。

`min_profit`设置最小利润（比例，例如0.01代表1%），低于此利润的交易对会被屏蔽。建议配合`minutes`参数使用，否则可能导致空pairlist。

```json
"pairlists": [
    // ...
    {
        "method": "PerformanceFilter",
        "minutes": 1440,  // 最近24小时
        "min_profit": 0.01 // 最低利润1%
    }
],
```

此过滤器根据过去交易表现，可能有启动延迟，建议在机器人运行一段时间（几百笔交易后）再启用。

!!! Warning "回测"
    `PerformanceFilter`不支持回测模式。

### PrecisionFilter（精度过滤器）

过滤无法设置合理止损的低值币。具体规则为：如果某币的“止损价”四舍五入后（考虑到交易所精度）变动一百分比（例如1%）以上，视为不适宜（黑名单）。

意图避免高价值低波动币因价格舍入产生亏损或无法设置止损。

!!! Tip "针对期货交易，PrecisionFilter基本无用"
    期货短线交易通常不会遇到此问题，止损会被自动平仓。

!!! Warning "回测"
    `PrecisionFilter`不支持多策略回测。

### PriceFilter（价格过滤器）

用以按价格范围筛选交易对，目前支持的筛选条件有：

* `min_price`
* `max_price`
* `max_value`
* `low_price_ratio`

**`min_price`**：排除价格低于该值的交易对，避免超低价货币（例如小数点后很多位的小币）。

**`max_price`**：排除价格高于该值的交易对，便于只交易低价币。

**`max_value`**：排除价格变动区间较高的交易对，适用某些限价交易的场景，如某些货币步进较大（如币值较高的小币，其单次交易金额会受额度限制）。

**`low_price_ratio`**：若某个价格单位（pip）涨幅超过此比例，则过滤。例如价格变动1个单位（1pip）超过了`low_price_ratio`值，则会被排除。

使用条件：`min_price`或`max_price`或`low_price_ratio`中至少一个要设置，否则无效。

示例：假设某币价格为0.00000011，价格变动1个pip（即0.00000001），占该价格比例约9%。若想过滤掉此币，可以用`low_price_ratio`设为0.09（9%）或用`min_price`设为0.00000011。

！！！ 警告 "低价币"
    低价币“1 pip”变动可能意味着价格极易受到交易所的最小变动尺（price step）影响，难以合理设置止损或止盈，交易时风险较大。

### ShuffleFilter（洗牌过滤器）

随机打乱交易对列表顺序，用于避免机器人对某些交易对偏好过强（比如总交易频率不均）。默认每蜡烛周期（candle）洗牌一次。

若希望每轮都洗牌，请设置`"shuffle_frequency"`为`"iteration"`。

示例：

```json
{
    "method": "ShuffleFilter", 
    "shuffle_frequency": "candle",
    "seed": 42
}
```

建议设置`seed`值，以便在多次回测中获得一致的随机结果，方便调试。

### SpreadFilter（价差过滤器）

过滤掉价差过大的交易对。具体表现为：ask价格和bid价格的差异比例超过`max_spread_ratio`（默认0.005，即0.5%）。

示例：  
若`DOGE/BTC`最高买价为0.00000026，最低卖价为0.00000027，则价差比例：`1 - bid/ask ≈ 0.037`，大于0.005，交易对会被过滤。

### RangeStabilityFilter（区间稳定性过滤器）

过滤掉价格在过去`lookback_days`天内变动幅度极端的交易对，设定最小和最大变化率。  
比如：最近10天的价格区间变动小于1%或大于99%都将被过滤。

配置示例：

```json
"pairlists": [
    {
        "method": "RangeStabilityFilter",
        "lookback_days": 10,
        "min_rate_of_change": 0.01,
        "max_rate_of_change": 0.99,
        "refresh_period": 86400
    }
]
```

可以加入`"sort_direction": "asc"`或`"desc"`进行排序。

此过滤器适合自动筛除波动极低（稳定币）或极端波动的交易对，以避免难以获利的交易。

### VolatilityFilter（波动率过滤器）

衡量交易对价格的历史波动程度，基于日对数收益率的标准差，表现为波动率。

筛选条件：   
- 若过去`lookback_days`天的平均波动率低于`min_volatility`，则过滤；  
- 若高于`max_volatility`也会过滤。

适合筛除过于平稳或极端波动的交易对。

配置示例：

```json
"pairlists": [
    {
        "method": "VolatilityFilter",
        "lookback_days": 10,
        "min_volatility": 0.05,
        "max_volatility": 0.50,
        "refresh_period": 86400
    }
]
```

可以启用排序：

```json
"pairlists": [
    {
        "method": "VolatilityFilter",
        "lookback_days": 10,
        "min_volatility": 0.05,
        "max_volatility": 0.50,
        "refresh_period": 86400,
        "sort_direction": "asc"
    }
]
```

### 全段Pairlist处理器示例

以下示例黑名单`BNB/BTC`，使用`VolumePairList`获取20个交易量最高的资产，按`quoteVolume`排序，同时应用`PrecisionFilter`和`PriceFilter`（过滤所有价格变化超过1%的交易对），之后应用`SpreadFilter`和`VolatilityFilter`，最后随机洗牌，设置随机种子以确保可重复性。

```json
"exchange": {
    "pair_whitelist": [],
    "pair_blacklist": ["BNB/BTC"]
},
"pairlists": [
    {
        "method": "VolumePairList",
        "number_assets": 20,
        "sort_key": "quoteVolume"
    },
    {"method": "AgeFilter", "min_days_listed": 10},
    {"method": "PrecisionFilter"},
    {"method": "PriceFilter", "low_price_ratio": 0.01},
    {"method": "SpreadFilter", "max_spread_ratio": 0.005},
    {
        "method": "RangeStabilityFilter",
        "lookback_days": 10,
        "min_rate_of_change": 0.01,
        "refresh_period": 86400
    },
    {
        "method": "VolatilityFilter",
        "lookback_days": 10,
        "min_volatility": 0.05,
        "max_volatility": 0.50,
        "refresh_period": 86400
    },
    {"method": "ShuffleFilter", "seed": 42}
],
```