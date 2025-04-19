# 配置交易机器人

Freqtrade拥有丰富的可配置功能与选项。  
默认情况下，这些设置都是通过配置文件进行（见下文）。

## Freqtrade的配置文件

在运行过程中，机器人会使用一组配置参数，这些参数共同构成了机器人配置。  
通常，机器人会从一个配置文件（即Freqtrade配置文件）中读取其配置。

默认情况下，机器人会加载位于当前工作目录中的 `config.json` 文件。

你也可以通过命令行参数 `-c/--config` 指定使用其他配置文件。

如果你采用[快速入门](docker_quickstart.md#docker-quick-start)的方法安装机器人，安装脚本应该已经帮你创建了默认配置文件(`config.json`)。

如果没有创建默认配置文件，建议使用 `freqtrade new-config --config user_data/config.json` 来生成一个基础的配置文件。

Freqtrade的配置文件必须采用JSON格式。

除了标准的JSON语法外，你还可以在配置文件中使用单行 `// ...` 和多行 `/* ... */` 注释，以及在参数列表中使用尾随逗号。

如果你对JSON格式不熟悉也不用担心——只需用自己喜欢的编辑器打开配置文件，修改所需参数，保存后重新启动机器人，或者在之前关闭的情况下再次运行，机器人会在启动时验证配置文件的语法，并在有错误时提醒你指出问题所在。

### 环境变量

可以通过设置环境变量来配置Freqtrade的参数。  
环境变量的优先级高于配置文件中的对应项或策略中的参数。

环境变量必须以 `FREQTRADE__` 为前缀，才能被加载到Freqtrade配置中。

`__` 作为层级分隔符，格式应为 `FREQTRADE__{section}__{key}`。  
例如，定义：
```bash
export FREQTRADE__STAKE_AMOUNT=200
```
将导致配置中 `stake_amount` 被设置为200。

更复杂的示例，例如用环境变量隐藏你的交易所密钥：
```bash
export FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>
```
这样会把值放到配置的 `exchange.key` 部分。  
通过这种方式，所有配置设置也都可以通过环境变量访问。

请注意：环境变量会覆盖配置文件中的对应设置，但命令行参数的优先级总是最高的。

常用示例：

```bash
FREQTRADE__TELEGRAM__CHAT_ID=<telegramchatid>
FREQTRADE__TELEGRAM__TOKEN=<telegramToken>
FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>
FREQTRADE__EXCHANGE__SECRET=<yourExchangeSecret>
```

JSON列表也支持通过环境变量设置，比如设置交易对白名单列表：

```bash
export FREQTRADE__EXCHANGE__PAIR_WHITELIST='["BTC/USDT", "ETH/USDT"]'
```

!!! 注意  
    在启动时，检测到的环境变量会被记录日志。如果你发现配置的某个值与你预期不符，可能是被环境变量覆盖，建议确认是否有相关环境变量被加载。

!!! 提示 "验证合并结果"  
    可以使用 [show-config 子命令](utils.md#show-config) 查看最终合并生成的完整配置。

??? 警告 "加载顺序"  
    环境变量是在最初配置加载后再进行载入的，因此你不能通过环境变量指定配置文件路径。请使用 `--config path/to/config.json` 的命令行参数来指定配置文件。  
    这也在一定程度上限制了 `user_dir` ，虽然可以通过环境变量设置用户目录，但配置文件不会从该位置加载。

### 多个配置文件

可以为机器人指定多个配置文件，或者让机器人从标准输入流读取配置参数。

在 `add_config_files` 参数中指定额外配置文件。这些文件会被加载并合并到初始配置文件中。  
文件路径相对于初始配置文件解析。

这种方式类似于多次使用 `--config` 参数，但更简便，无需为每个命令都指定所有配置文件。

!!! 提示 "验证合并结果"  
    使用 [show-config 子命令](utils.md#show-config) 可以查看最终合并的配置。

!!! 提示 "用多个配置文件保密"  
    可以用第二个配置文件来存放敏感信息（如API密钥），避免在公共配置中暴露。  
    第二个配置文件应只包含需要覆盖的内容。在合并时，最后指定的配置优先。  
    举例：`config-private.json` 是你的私密配置文件。

    执行命令示例：

```bash
freqtrade trade --config user_data/config.json --config user_data/config-private.json <...>
```

配置示例：

```json
"user_data/config.json"
{
    "add_config_files": [
        "config1.json",
        "config-private.json"
    ]
}
```

这样可以在保持配置的同时，更好地管理私密信息。

```bash
freqtrade trade --config user_data/config.json <...>
```

??? 注意 "配置冲突处理"  
    如果在 `config.json` 和 `config-import.json` 中都定义了相同参数，优先加载父配置。

例如：
```json
// user_data/config.json
{
    "max_open_trades": 3,
    "stake_currency": "USDT",
    "add_config_files": ["config-import.json"]
}
```

```json
// user_data/config-import.json
{
    "max_open_trades": 10,
    "stake_amount": "unlimited"
}
```

最终合并结果为：
```json
{
    "max_open_trades": 3,
    "stake_currency": "USDT",
    "stake_amount": "unlimited"
}
```

如果`add_config_files`中配置了多个文件，最后出现的配置会覆盖之前的（除非父配置已定义该参数）。

## 编辑器自动补全与验证

如果你使用支持JSON Schema的编辑器，可以在配置文件顶部添加以下内容，以获得自动补全与验证功能：

```json
{
    "$schema": "https://schema.freqtrade.io/schema.json"
}
```

??? 提示 "开发版"  
    开发中的Schema可用地址为 `https://schema.freqtrade.io/schema_dev.json`，但建议使用稳定版本以获得最佳体验。

## 配置参数

下表列出所有可用的配置参数。

Freqtrade也支持通过命令行参数（CLI）设置许多选项（详细信息请参考 `--help` 输出）。

### 配置选项的优先级

所有选项的优先级顺序为：

* 命令行参数优先
* [环境变量](#environment-variables)
* 配置文件（按顺序加载，后加载的覆盖前面的）  
* 策略中的参数（如果没有被配置或命令行覆盖）——在下表中标识为 [策略覆盖](#parameters-in-the-strategy)。

### 参数表

必要参数用 **Required** 标注，表示必须在某种方式中设置。

| 参数名 | 描述 |
|------------|------------------------------------------------|
| `max_open_trades` | **必需。** 允许的最大未平仓交易数量。每对仅允许一笔未平仓交易，因此对名单长度也是上限之一。若设置为-1，则代表无限制（受交易对名单限制）。[详情](#configuring-amount-per-trade)。[策略覆盖](#parameters-in-the-strategy)。<br>**数据类型：** 正整数或 -1。|
| `stake_currency` | **必需。** 用于交易的加密货币。<br>**数据类型：** 字符串。|
| `stake_amount` | **必需。** 每笔交易使用的加密货币数量。设为 `"unlimited"` 表示机器人可用所有余额。 [详情](#configuring-amount-per-trade)。<br>**数据类型：** 正浮点数或 `"unlimited"`。|
| `tradable_balance_ratio` | 允许交易的总账户余额比例。 [详情](#configuring-amount-per-trade)。<br>**默认：** `0.99`（99%）。<br>**数据类型：** 介于 `0.1` 和 `1.0` 的正浮点数。|
| `available_capital` | 机器人可用的起始资金。适用于运行多个机器人在同一交易所账户的场景。 [详情](#configuring-amount-per-trade)。<br>**数据类型：** 正浮点数。|
| `amend_last_stake_amount` | 必要时使用缩减的上次投入金额。 [详情](#configuring-amount-per-trade)。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `last_stake_amount_min_ratio` | 定义剩余且必须执行的最小投入金额比。仅在 `amend_last_stake_amount` 设置为 `true` 时生效。 [详情](#configuring-amount-per-trade)。<br>**默认：** `0.5`。<br>**数据类型：** 浮点，比值形式。|
| `amount_reserve_percent` | 在最小交易对投入金额中保留一定比例，避免交易拒单。会在计算最小投入金额时，预留 `amount_reserve_percent` + 止损值（如止损偏移）对应的金额。 [详情](#configuring-amount-per-trade)。<br>**默认：** `0.05`（5%）。<br>**数据类型：** 正浮点数（比率形式）。|
| `timeframe` | 使用的时间周期（例如`1m`、`5m`、`15m`、`30m`、`1h`等），多在策略中指定。 [策略覆盖](#parameters-in-the-strategy)。<br>**数据类型：** 字符串。|
| `fiat_display_currency` | 用于显示盈利的法币单位。 [详情](#what-values-can-be-used-for-fiat_display_currency)。<br>**数据类型：** 字符串。|
| `dry_run` | **必需。** 指定机器人是否应在Dry Run模式（模拟交易）下运行。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| `dry_run_wallet` | 在Dry Run模式中，用于模拟钱包的起始资金。支持用数字（浮点）或字典定义每个货币的余额。示例：  
```json
"dry_run_wallet": { "BTC": 0.01, "ETH": 2, "USDT": 1000 }
```  
可以通过命令行参数覆盖数字值，但字典形式必须在配置文件中设置。  
**默认：** 1000。|
| `cancel_open_orders_on_exit` | 在收到 `/stop` RPC命令、按下 `Ctrl+C` 或机器人异常关闭时，取消所有未完成订单。启用后，利用 `/stop` 命令取消未完成和部分成交订单，但不影响持仓。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `process_only_new_candles` | 只在新K线到来时处理指标。否则每轮循环都会处理一次相同的K线，可能造成系统负载，但对于依赖逐笔数据的策略很有用。 [策略覆盖](#parameters-in-the-strategy)。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| `minimal_roi` | **必需。** 设定机器人在何种ROI阈值下退出交易的比例。 [详情](#understand-minimal_roi)。[策略覆盖](#parameters-in-the-strategy)。<br>**数据类型：** 字典。|
| `stoploss` | **必需。** 设定止损值（比例）。详细信息请参见 [止损文档](stoploss.md)。[策略覆盖](#parameters-in-the-strategy)。<br>**数据类型：** 浮点数（比例）。|
| `trailing_stop` | 激活追踪止损（基于配置或策略文件中的 `stoploss`）。详细见 [止损文档](stoploss.md#trailing-stop-loss)。[策略覆盖](#parameters-in-the-strategy)。<br>**数据类型：** 布尔值。|
| `trailing_stop_positive` | 在盈利达到一定水平后调整止损。详见 [止损文档](stoploss.md#trailing-stop-loss-custom-positive-loss)。[策略覆盖](#parameters-in-the-strategy)。<br>**数据类型：** 浮点数。|
| `trailing_stop_positive_offset` | 激活`trailing_stop_positive`的偏移比例。正数，比例值。详见 [止损文档](stoploss.md#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset)。[策略覆盖](#parameters-in-the-strategy)。<br>**默认：** `0.0`（无偏移）。<br>**数据类型：** 浮点数。|
| `trailing_only_offset_is_reached` | 仅在偏移值达到后才应用追踪止损。详见 [止损文档](stoploss.md)。[策略覆盖](#parameters-in-the-strategy)。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `fee` | 回测/模拟交易中使用的手续费。通常不配置，freqtrade会默认为交易所默认手续费。以比例方式设定（如0.001=0.1%），每笔交易开仓和平仓各收取一次。<br>**数据类型：** 浮点数（比例）。|
| `futures_funding_rate` | 用户自定义的资金费率（当交易所没有提供历史资金费率时使用）。不覆盖实际历史费率。建议设为0，除非在测试特定币种并理解资金费率对利润的影响。详见 [杠杆文档](leverage.md#unavailable-funding-rates)。<br>**默认：** `None`。<br>**数据类型：** 浮点数。|
| `trading_mode` | 指定交易方式：正常交易、杠杆交易或基于匹配加密货币价格的合约交易。详见 [杠杆交易文档](leverage.md)。<br>**默认：** `"spot"`。<br>**数据类型：** 字符串。|
| `margin_mode` | 使用杠杆交易时，决定保证金是共享还是隔离。详见 [杠杆交易](leverage.md)。<br>**数据类型：** 字符串。|
| `liquidation_buffer` | 设定一个比例，保证在强制平仓价格与止损价之间留出安全垫，防止触及平仓线。详见 [杠杆交易](leverage.md)。<br>**默认：** `0.05`。<br>**数据类型：** 浮点数。|
| | **未平仓超时设置** |
| `unfilledtimeout.entry` | **必需。** 机器人在未成交的入场订单等待的时间（分钟或秒），超时后订单会被取消。详见 [策略](#parameters-in-the-strategy)。<br>**数据类型：** 整数。|
| `unfilledtimeout.exit` | **必需。** 机器人在未成交的出场订单等待的时间（分钟或秒），超时后订单会被取消，并以当前（新）价格重新挂单，只要有信号。详见 [策略](#parameters-in-the-strategy)。<br>**数据类型：** 整数。|
| `unfilledtimeout.unit` | 设置未成交订单超时单位。注意：若设为 `"seconds"`，则 `internals.process_throttle_secs` 必须小于或等于此超时值。详见 [策略](#parameters-in-the-strategy)。<br>**默认：** `"minutes"`。<br>**数据类型：** 字符串。|
| `unfilledtimeout.exit_timeout_count` | 出场订单超时的最大次数。达到次数后触发紧急退出，设为0则不限制。详见 [策略](#parameters-in-the-strategy)。<br>**默认：** `0`。<br>**数据类型：** 整数。|
| | **价格设定** |
| `entry_pricing.price_side` | 选择机器人应关注的买卖差价侧（ask/bid）以获得入场价格。详见 [入场价](#entry-price)。<br>**默认：** `"same"`。<br>**数据类型：** 字符串（`ask`, `bid`, `same`, `other`）。|
| `entry_pricing.price_last_balance` | **必需。** 插值计算出缠入价格。详见 [无订单簿入场](#entry-price-without-orderbook-enabled)。|
| `entry_pricing.use_order_book` | 启用订单簿入场方式，使用订单簿中的价格。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| `entry_pricing.order_book_top` | 使用订单簿中“price_side”对应的前N个最优价格作为入场。比如值为2，则机器人可以选择订单簿中第二个报价。<br>**默认：** `1`。<br>**数据类型：** 正整数。|
| `entry_pricing.check_depth_of_market.enabled` | 若买卖盘差异超过设定值，则不入场。详见 [市场深度检查](#check-depth-of-market)。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `entry_pricing.check_depth_of_market.bids_to_ask_delta` | 订单簿中买卖盘差异比率（买卖笔数或量比）。小于1表示卖盘较大，大于1表示买盘较大。详见 [市场深度检查](#check-depth-of-market)。<br>**默认：** `0`。<br>**数据类型：** 浮点数（比率）。|
| `exit_pricing.price_side` | 选择关注的卖出差价侧（ask/bid）以获得出场价格。详见 [出场价](#exit-price-side)。<br>**默认：** `"same"`。<br>**数据类型：** 字符串（`ask`, `bid`, `same`, `other`）。|
| `exit_pricing.price_last_balance` | **必需。** 插值计算出出场价格。详见 [无订单簿出场](#exit-price-without-orderbook-enabled)。|
| `exit_pricing.use_order_book` | 启用订单簿出场，使用订单簿中的价格。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| `exit_pricing.order_book_top` | 使用订单簿中“price_side”横向的前N个报价作为出场。比如值为2，则用第二个ask价格。<br>**默认：** `1`。<br>**数据类型：** 正整数。|
| `custom_price_max_distance_ratio` | 配置当前价格与自定义入场/出场价格的最大距离比（比例）。<br>**默认：** `0.02`（2%）。<br>**数据类型：** 正浮点数。|
| | **订单/信号处理** |
| `use_exit_signal` | 使用策略提供的退出信号（除ROI外）。启用后，会禁用 `"exit_long"` 和 `"exit_short"` 这两列，其他退出方式不受影响。详见 [策略](#parameters-in-the-strategy)。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| `exit_profit_only` | 只有在盈利达到 `exit_profit_offset` 时才考虑退出信号。详见 [策略](#parameters-in-the-strategy)。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `exit_profit_offset` | 出场信号只在超过此比例时激活。仅在 `exit_profit_only=True` 时有效。详见 [策略](#parameters-in-the-strategy)。<br>**默认：** `0.0`。<br>**数据类型：** 浮点（比例值）。|
| `ignore_roi_if_entry_signal` | 若入场信号仍有效，则不退出。优先级高于 `minimal_roi` 和 `use_exit_signal`。详见 [策略](#parameters-in-the-strategy)。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `ignore_buying_expired_candle_after` | 定义在多少秒后买入信号失效。<br>**数据类型：** 整数。|
| `order_types` | 配置订单类型（包括 `"entry"`、`"exit"`、`"stoploss"`、`"stoploss_on_exchange"`等），以及止损在交易所的实现方式。详见 [订单类型](#understand-order_types)。<br>**数据类型：** 字典。|
| `order_time_in_force` | 配置订单的“生存策略”时间（GTC、FOK、IOC等）。详见 [订单时间策略](#understand-order_time_in_force)。<br>**数据类型：** 字典。|
| `position_adjustment_enable` | 允许策略进行仓位调整（加仓或减仓）。详见 [仓位调整](strategy-callbacks.md#adjust-trade-position)。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `max_entry_position_adjustment` | 每个开仓在第一笔仓位基础上的最大调整次数（数量），设为 `-1` 表示无限次调整。详见 [仓位调整](strategy-callbacks.md#adjust-trade-position)。<br>**默认：** `-1`。<br>**数据类型：** 正整数或 -1。|
| | **交易所设置** |
| `exchange.name` | **必需。** 要使用的交易所类名。<br>**数据类型：** 字符串。|
| `exchange.key` | API密钥，仅在生产模式下必需。请妥善保管，勿公开泄露。<br>**数据类型：** 字符串。|
| `exchange.secret` | API秘钥，仅在生产模式下必需。请妥善保管，勿公开泄露。<br>**数据类型：** 字符串。|
| `exchange.password` | API密码，仅在生产模式下需要（部分交易所用密码API）。请妥善保管，勿泄露。<br>**数据类型：** 字符串。|
| `exchange.uid` | API用户ID（uid），在部分交易所用。仅在生产模式下需提供。<br>**数据类型：** 字符串。|
| `exchange.pair_whitelist` | 机器人进行交易和回测时的白名单交易对。支持正则匹配，如 `.*/BTC`。不适用于 `VolumePairList`。详见 [交易对列表插件](plugins.md#pairlists-and-pairlist-handlers)。<br>**数据类型：** 列表。|
| `exchange.pair_blacklist` | 绝对禁用的交易对列表。详见 [插件](plugins.md#pairlists-and-pairlist-handlers)。<br>**数据类型：** 列表。|
| `exchange.ccxt_config` | 传递给ccxt实例的附加参数(同步&异步)。通常用于配置额外的ccxt参数。不同交易所支持的参数可能不同，详见 [ccxt文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation)。不要在此处放置敏感的API密钥，可能会出现在日志中。<br>**数据类型：** 字典。|
| `exchange.ccxt_sync_config` | 传递给同步ccxt实例的参数。详见 [ccxt文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation)。<br>**数据类型：** 字典。|
| `exchange.ccxt_async_config` | 传递给异步ccxt实例的参数。详见 [ccxt文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation)。<br>**数据类型：** 字典。|
| `exchange.enable_ws` | 是否启用Websocket。<br>[详情](#consuming-exchange-websockets)。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| `exchange.markets_refresh_interval` | 市场信息刷新间隔（分钟）。<br>**默认：** 60分钟。<br>**数据类型：** 正整数。|
| `exchange.skip_open_order_update` | 避免在启动时更新未成交订单（尤其在交易所不稳定时）<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `exchange.unknown_fee_rate` | 计算手续费的备用值。用于部分费率非以主要交易币计的交易所。系统会将此值乘以手续费成本。<br>**默认：** `None`。<br>**数据类型：** 浮点数。|
| `exchange.log_responses` | 记录交易所响应信息（调试用）。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `exchange.only_from_ccxt` | 禁止从 `data.binance.vision` 等第三方源下载数据。关闭此项能加快下载，但可能影响数据完整性。<br>**默认：** `false`。<br>**数据类型：** 布尔值。|
| `experimental.block_bad_exchanges` | 阻止已知不可用的交易所。除非你想测试，否则建议保持默认。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| | **插件** |
| `edge.*` | 详见 [edge配置文档](edge.md)，描述所有可选配置选项。|
| `pairlists` | 指定一个或多个交易对列表。详见 [插件](plugins.md#pairlists-and-pairlist-handlers)。<br>**默认：** `StaticPairList`。<br>**数据类型：** 列表（字典）。|
| | **Telegram通知** |
| `telegram.enabled` | 是否启用Telegram通知。<br>**数据类型：** 布尔值。|
| `telegram.token` | 你的Telegram机器人Token。仅在 `enabled` 为`true`时需要。请妥善保管，勿公开泄露。<br>**数据类型：** 字符串。|
| `telegram.chat_id` | 你的Telegram账号ID。仅在 `enabled` 为`true`时需要。请妥善保管，勿公开泄露。<br>**数据类型：** 字符串。|
| `telegram.balance_dust_level` | 微尘水平（用stake货币计）。余额低于该值时， `/balance` 不显示。<br>**数据类型：** 浮点数。|
| `telegram.reload` | 是否允许在Telegram消息中使用“重新加载”按钮。<br>**默认：** `true`。<br>**数据类型：** 布尔值。|
| `telegram.notification_settings.*` | 详细的通知设置。详见 [Telegram使用文档](telegram-usage.md)。<br>**数据类型：** 字典。|
| `telegram.allow_custom_messages` | 允许策略通过 `dataprovider.send_msg()` 发送自定义Telegram消息。<br>**数据类型：** 布尔值。|
| | **Webhook通知** |
| `webhook.enabled` | 是否启用Webhook通知。<br>**数据类型：** 布尔值。|
| `webhook.url` | Webhook的URL。仅在 `enabled` 为`true`时需要。详见 [Webhook文档](webhook-config.md)。<br>**数据类型：** 字符串。|
| `webhook.entry` | 始始化请求的负载（进入信号）。仅在启用时需要。详见 [Webhook文档](webhook-config.md)。<br>**数据类型：** 字符串。|
| `webhook.entry_cancel` | 取消入场订单时的负载。<br>**数据类型：** 字符串。|
| `webhook.entry_fill` | 入场订单成交时的负载。<br>**数据类型：** 字符串。|
| `webhook.exit` | 出场时的负载。<br>**数据类型：** 字符串。|
| `webhook.exit_cancel` | 取消出场订单时的负载。<br>**数据类型：** 字符串。|
| `webhook.exit_fill` | 出场订单成交时的负载。<br>**数据类型：** 字符串。|
| `webhook.status` | 状态请求的负载。<br>**数据类型：** 字符串。|
| `webhook.allow_custom_messages` | 允许策略通过 `dataprovider.send_msg()` 发送Webhook消息。<br>**数据类型：** 布尔值。|
| | **REST API / FreqUI / 生产者-消费者** |
| `api_server.enabled` | 允许启用API服务器。详见 [API服务器文档](rest-api.md)。<br>**数据类型：** 布尔值。|
| `api_server.listen_ip_address` | 绑定的IP地址。详见 [API服务器](rest-api.md)。<br>**数据类型：** IPv4地址。|
| `api_server.listen_port` | 绑定端口，范围1024-65535。详见 [API服务器](rest-api.md)。<br>**数据类型：** 整数。|
| `api_server.verbosity` | 日志详细程度。`info`会显示所有RPC调用；`error`只显示错误。详见 [API服务器](rest-api.md)。<br>**数据类型：** 枚举（`info`或`error`），默认`info`。|
| `api_server.username` | API服务器用户名（用于认证）。请保密不公开。<br>**数据类型：** 字符串。|
| `api_server.password` | API密码（用于认证）。请保密不公开。<br>**数据类型：** 字符串。|
| `api_server.ws_token` | WebSocket消息令牌（用以验证）。请妥善保管。<br>**数据类型：** 字符串。|
| `bot_name` | 机器人的名字，作为API参数传递。用于区分不同机器人。<br>**默认值：**`freqtrade`。<br>**数据类型：** 字符串。|
| `external_message_consumer` | 开启生产者-消费者模式（详见 [producer-consumer.md](producer-consumer.md)）。<br>**数据类型：** 字典。|
| | **其他** |
| `initial_state` | 定义应启动的初始状态（`running`、`paused`、`stopped`）。如设为`stopped`，需通过 `/start` RPC显式启动。<br>**默认：** `stopped`。<br>**数据类型：** 枚举。|
| `force_entry_enable` | 使能RPC强制开仓命令（`/forcelong`、`/forceshort`）。优化开仓。详见 [策略](#parameters-in-the-strategy)。<br>**数据类型：** 布尔值。|
| `disable_dataframe_checks` | 禁用策略返回的OHLCV数据的正确性检查。仅在明确知道自己在做什么时使用。详见 [参数](#parameters-in-the-strategy)。<br>**默认：** `False`。<br>**数据类型：** 布尔值。|
| `internals.process_throttle_secs` | 设置单次机器人循环的最小时间间隔（秒）。<br>**默认：** 5秒。<br>**数据类型：** 正整数。|
| `internals.heartbeat_interval` | 每隔N秒打印一次心跳消息，设为0则禁用。<br>**默认：** 60秒。<br>**数据类型：** 正整数或0。|
| `internals.sd_notify` | 支持使用 `sd_notify` 协议通知 `systemd` 服务管理器机器人状态变化和存活心跳。详见 [高级设置](advanced-setup.md#configure-the-bot-running-as-a-systemd-service)。<br>**数据类型：** 布尔值。|
| `strategy` | **必需。** 指定策略类名。推荐在启动参数中通过 `--strategy NAME` 指定。<br>**数据类型：** 类名字符串。|
| `strategy_path` | 指定额外的策略查找路径（目录）。<br>**数据类型：** 字符串。|
| `recursive_strategy_search` | 设置为`true`时，会递归搜索 `user_data/strategies` 子目录中的策略文件。<br>**数据类型：** 布尔值。|
| `user_data_dir` | 用户数据所在目录。<br>**默认：** `./user_data/`。<br>**数据类型：** 字符串。|
| `db_url` | 数据库连接URL。注意：若`dry_run`为`true`，默认为`sqlite:///tradesv3.dryrun.sqlite`；否则为`sqlite:///tradesv3.sqlite`（生产环境）。<br>**数据类型：** 字符串/SQLAlchemy连接字符串。|
| `logfile` | 指定日志文件名。采用滚动策略，最多10个文件，每个最大1MB。<br>**数据类型：** 字符串。|
| `add_config_files` | 其他配置文件。会被加载并与当前配置合并，路径相对初始配置文件。<br>**默认：** `[]`。<br>**数据类型：** 字符串列表。|
| `dataformat_ohlcv` | 存储历史K线（OHLCV）数据的格式（例如`feather`）。<br>**默认：** `feather`。<br>**数据类型：** 字符串。|
| `dataformat_trades` | 存储历史交易数据的格式（例如`feather`）。<br>**默认：** `feather`。<br>**数据类型：** 字符串。|
| `reduce_df_footprint` | 将所有数值列转成`float32`/`int32`，以降低内存和存储空间占用，减少回测和训练时间。详见 [参数](#parameters-in-the-strategy)。<br>**默认：** `False`。<br>**数据类型：** 布尔值。|
| `log_config` | 包含Python日志配置的字典。详见 [高级设置](advanced-setup.md#advanced-logging)。<br>**数据类型：** 字典。<br>**默认：** `FtRichHandler`。|

## 策略中的参数

以下参数可在配置文件或策略中设置。  
配置文件中的参数值会覆盖策略中的设置。

* `minimal_roi`
* `timeframe`
* `stoploss`
* `max_open_trades`
* `trailing_stop`
* `trailing_stop_positive`
* `trailing_stop_positive_offset`
* `trailing_only_offset_is_reached`
* `use_custom_stoploss`
* `process_only_new_candles`
* `order_types`
* `order_time_in_force`
* `unfilledtimeout`
* `disable_dataframe_checks`
* `use_exit_signal`
* `exit_profit_only`
* `exit_profit_offset`
* `ignore_roi_if_entry_signal`
* `ignore_buying_expired_candle_after`
* `position_adjustment_enable`
* `max_entry_position_adjustment`

## 配置每笔交易的投入金额

有多种方法配置机器人每次交易所用的投入额度。  
所有方法都遵循 [可用余额配置](#tradable-balance) 规则，详见下文。

### 最小交易额度

最小投入金额依赖于交易所及交易对，一般在交易所的支持页面列出。

举例：假设 XRP/USD 的最小可交易量是20 XRP（由交易所提供），且当前价格为0.6美元，  
则买入该交易对的最小投入为 `20 * 0.6 ≈ 12$`。  
该交易所对美元有最低订单限制（需大于10美元），但在此例中不适用。

为了保证安全执行，Freqtrade不会允许以10.1美元的投入进行买入，  
而是会确保有足够空间设立止损（包括由 `amount_reserve_percent` 设置的偏移，默认5%）比。

考虑预留5%，那么最小投入约为：`12 * (1 + 0.05) ≈ 12.6$`  
如果再考虑在此基础上设置10%的止损，实际投入会变为：`12.6 / (1 - 0.1) ≈ 14$`。

为了避免在较大止损时的计算偏差，实际允许的最小投入金额不会超过真实限制的50%。

!!! 警告  
    由于交易所的最大可用额度通常比较稳定且更新不频繁，某些交易对由于价格大幅上涨会显示较高的最小限制，该限制其实是价格变化带来的。  
    Freqtrade会调整投入额度以符合市场，但如果该值超过目标投入的30%，则会拒绝该交易。

### Dry-run钱包

在Dry-Run模式下，机器人会使用模拟钱包进行交易模拟。  
其起始余额由 `dry_run_wallet` 指定（默认为1000）。  
对于复杂场景，也可以用字典指定每个货币的起始余额，例如：

```json
"dry_run_wallet": {
    "BTC": 0.01,
    "ETH": 2,
    "USDT": 1000
}
```

可以通过命令行参数（`--dry-run-wallet`）覆盖单个浮点值，但不能直接覆盖字典。  
若用字典，必须在配置文件中设置。

!!! 注意  
    不在投入货币的余额将不会用于交易，但会显示在钱包余额中。  
    跨保证金交易所的余额也可能影响可用保证金的计算。

### 可交易余额

默认情况下，机器人认为“总余额的99%”（即 `1%` 预留用于手续费）是可用的。   
当启用[动态投入金额](#dynamic-stake-amount)时，会将总余额分配到 `max_open_trades` 个额度中。

机器人会留出1%的余额用于交易手续费，所以默认不会用尽全部余额。

你也可以通过设置 `tradable_balance_ratio` 来配置“未被动用的部分”。

例如：你在交易所钱包中有10 ETH，如果设置  
`tradable_balance_ratio=0.5`（50%），机器人将最多使用5 ETH进行交易，其余余额保持未动。

!!! 危险  
    在同一账户运行多个机器人时，不要使用此设置。请使用 [给机器人预留的资金](#assign-available-capital)。

!!! 警告  
    `tradable_balance_ratio` 作用在“当前余额”上（可用余额+在交易中的余额）。  
    举例：假设起始余额为1000，`tradable_balance_ratio=0.99`，那么不一定总会剩下10本币。例如余额因亏损或提币减少到500，则剩余可能只有5。

### 分配可用资金

为了充分利用多机器人在同一交易所的资金池，你可以为每个机器人设置单独的起始资本（`available_capital`），  
适配策略是将初始资金平均分配到 `max_open_trades` 额度中。

比如：你的账户有10000 USDT，想运行两个策略，将 `available_capital` 设置为5000，两者各得5000初始资金。  
机器人会将这笔资金平分成最大开仓数的份额，盈利后会增加投入规模，但不会影响其他机器人。

更改 `available_capital` 后需要重新加载配置，且变更部分会是之前价值与新值的差额。  
若减小资金，将不会强制平仓，收益会在交易结束后体现到余额中。

!!! 警告 "不兼容 `tradable_balance_ratio`"  
    设置 `available_capital` 会覆盖 `tradable_balance_ratio` 配置。

### 修改最后一次投入金额

假设你有可交易余额为1000 USDT，  
`stake_amount=400`，最大开仓数为3，  
当已经有800 USDT（两笔400）交易在进行时，如剩余额只有200 USDT，就无法再开第三笔。

此时，可以设置 `amend_last_stake_amount` 为 `True`，让机器人自动用剩余余额调整最后一笔的投入。

示例：  
- 交易1：400 USDT  
- 交易2：400 USDT  
- 交易3：用剩余余额200 USDT（调整为200 USDT投入）

!!! 注意  
    该功能仅在固定投入（Static stake）下有效。  
    在动态投入（Dynamic stake）模式下，余额会平均分配。

!!! 注意  
    最小投入金额可以通过 `last_stake_amount_min_ratio` 配置，默认为0.5，即最低为 `stake_amount * 0.5`，避免投入过少难以交易。

### 固定投入金额

`stake_amount` 固定配置每次交易的投入金额。  
最小值建议查阅交易所支持的最低交易额度。

此设置会和 `max_open_trades` 联合。如果最大开仓数为3，投入金额为0.05，则最大投入资金为0.15。如果配置为 `stake_amount=0.05` 及 `max_open_trades=3`。

### 动态投入金额

设置为 `"unlimited"`，机器人会根据账户余额自动平分到 `max_open_trades` 份。  
建议同时设置：  
`"stake_amount": "unlimited"` 和 `tradable_balance_ratio=0.99`，以保证账户余额有一定余量。

此时，单笔交易金额的计算公式为：  
```python
currency_balance / (max_open_trades - current_open_trades)
```

例如：在账户剩余余额为 `X`，允许最大 `max_open_trades`，则每笔交易金额为：  
`X / (max_open_trades - 当前已开仓数)`

要启用全部余额（减去保留比例），可以设置：  
```json
"stake_amount": "unlimited",
"tradable_balance_ratio": 0.99,
```

!!! 提示 "复利收益"  
    利用此配置，机器人会根据盈利情况自动调节投入规模（亏损时降低，盈利时升高），实现利润复利。

!!! 注意 "使用Dry-Run模式时"  
    在Dry-Run、回测或超参数调优时，`"stake_amount": "unlimited"`，余额是模拟的，会从 `dry_run_wallet` 初始余额开始演变。  
    推荐设置合理的 `dry_run_wallet`（例如：0.05、0.01BTC，或1000、100USDT），避免模拟交易大额资金或过小余额，导致结果偏差。

### 带仓位调整的动态投入金额

启用无限投入时，若需要仓位调整，还必须实现 `custom_stake_amount`，根据策略返回一个调整值。  
一般建议在25%-50%区间内，具体依据策略定义的缓冲空间。

例如：如果仓位调整允许额外两次买入，那么缓冲空间应为原始无限投入的66.7%；  
如果允许一次买入，额度为原始的25%；  
结果是：  
- `custom_stake_amount` 返回提议规模的25%，允许余额保留75%作后续调整。

--8<-- "includes/pricing.md"

## 其他配置细节

### 了解 minimal_roi

`minimal_roi` 是一个JSON对象，键是持续时间（分钟），值是最大ROI比例。  
示例：
```json
"minimal_roi": {
    "40": 0.0,    // 40分钟后无盈利限制
    "30": 0.01,   // 30分钟内利润≥1%则退出
    "20": 0.02,   // 20分钟内利润≥2%则退出
    "0":  0.04    // 任何时候利润≥4%则立即退出
}
```

大多数策略文件中已包含合理的 `minimal_roi` 设置。  
此参数可在策略或配置中设置，配置中优先。如果未设置，则默认 `{"0": 10}`（即风险允许最大收益为1000%，实际几乎永不退出）。

!!! 注意 "强制退出（force exit）时间限制"
    使用 `"<N>": -1` 表示无论利润正负，在N分钟后强制退出交易。

### 理解 force_entry_enable

`force_entry_enable` 开启后，可以通过Telegram或REST API使用 `/forcelong` `/forceshort` 命令强制开仓。  
默认禁用，启用时会有警告。  
此功能因策略不同可能有风险，请谨慎使用。

详细操作请参阅 [Telegram文档](telegram-usage.md)。

### 忽略过期K线信号

在使用较长时间周期（如1小时以上）时，尤其当 `max_open_trades` 设置较低，  
最后一根K线被处理时可能已过了用以产生买入信号的有效期。  
此时，可以设定 `ignore_buying_expired_candle_after`，表示超过多少秒后，买入信号作废。

示例：使用1小时周期，但只在前5分钟内买入，配置：
```json
{
  //...
  "ignore_buying_expired_candle_after": 300,
  //...
}
```

!!! 注意  
    该设置在每根新K线到来后会重置，不能阻止连续的多根K线触发信号。  
    最佳做法是用“触发”筛选买入信号，只在一个K线期间激活。

### 理解 order_types

`order_types` 用于定义订单操作的类型（`entry`、`exit`、`stoploss`、`emergency_exit`、`force_exit`、`force_entry`等），以及止损在交易所的执行方式（市场或限价订单）。  
还可以配置止损订单在交易所的“挂单”更新间隔。

允许使用限价和市价订单进行入场、出场、止损等操作。  
比如，止损可以在买入后立即用市价单挂出。

配置在配置文件中的 `order_types` 会覆盖策略中的设置，  
必须确保包含以下4个参数（否则机器人无法启动）：  
- `entry`  
- `exit`  
- `stoploss`  
- `stoploss_on_exchange`  

示例策略配置：
```python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": False,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99,
}
```

示例配置：
```json
"order_types": {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": false,
    "stoploss_on_exchange_interval": 60
}
```

!!! 警告 "支持市价单"
    并非所有交易所都支持市价订单。  
    若不支持，启动时会提示 `"Exchange <交易所名> does not support market orders."`，机器人将无法启动。

!!! 警告 "使用市价单"
    使用市价单时务必谨慎，详见 [市场订单价格](#market-order-pricing)部分。

!!! 注意 "交易所止损"
    `order_types.stoploss_on_exchange_interval`非强制。  
    若未明确了解操作，建议不要修改。

    若开启`stoploss_on_exchange`，且止损被交易所手动取消，机器人会自动挂出新的止损单。

!!! 警告 "order_types.stoploss_on_exchange 失败"
    若在交易所挂止损失败，机器人会启动应急退出（默认用市价单平仓）。  
    可以通过配置 `order_types` 中的 `emergency_exit` 指定退出订单的类型，不过不建议更改。

### 了解 order_time_in_force

`order_time_in_force` 定义订单在交易所的存活策略（GTC、FOK、IOC等），影响订单何时生效或自动取消。

常用时间策略：

**GTC（Good Till Canceled）**：  
订单会一直挂单，直到被取消或成交。

**FOK（Fill Or Kill）**：  
订单未能立即全部成交，则会被自动取消。

**IOC（Immediate Or Canceled）**：  
订单会尽快成交部分，剩余部分被取消。

**PO（Post Only）**：  
只挂入“maak”模式（只能作为挂单放在订单簿），若未挂入则取消。

#### `order_time_in_force` 配置示例

在配置或策略中，用字典指定入场和出场订单策略：
```python
"order_time_in_force": {
    "entry": "GTC",
    "exit": "GTC"
}
```

!!! 警告  
    目前只支持部分交易所（如 Binance、Gate、Kucoin），其他交易所暂不支持。  
    修改前请确认自己是否了解其影响。

### 法币转换

Freqtrade 使用 Coingecko API 将加密货币价值转换为对应的法币，通过 `fiat_display_currency` 参数设置。

如果完全移除 `fiat_display_currency`，则不会初始化Coingecko，也不会显示法币转换。这不会影响机器人正常运行。

#### forex_display_currency 可以取哪些值？

支持多种法币（包括美元、欧元等）：

```json
"AUD", "BRL", "CAD", "CHF", "CLP", "CNY", "CZK", "DKK", "EUR", "GBP", "HKD", "HUF", "IDR", "ILS", "INR", "JPY", "KRW", "MXN", "MYR", "NOK", "NZD", "PHP", "PKR", "PLN", "RUB", "SEK", "SGD", "THB", "TRY", "TWD", "ZAR", "USD"
```

也支持部分主流加密货币（如BTC、ETH、XRP等）：

```json
"BTC", "ETH", "XRP", "LTC", "BCH", "BNB"
```

#### Coingecko调用频率限制

部分IP段会实施频率限制，若遇到此问题可在配置中添加API密钥：

```json
{
    "fiat_display_currency": "USD",
    "coingecko": {
        "api_key": "your-api",
        "is_demo": true
    }
}
```

Freqtrade支持免费和专业版本的API密钥。

**注意**：API密钥非必要，仅用于交易报告中的币价转换。  
大多数场景无需提供API密钥。

## 通过Websocket消费交易所数据

Freqtrade可以通过 `ccxt.pro` 使用Websocket，确保数据实时可用。  
若连接失败或禁用，会自动退回到REST API。

默认启用Websocket `exchange.enable_ws`，如需禁用：

```jsonc
"exchange": {
    // ...
    "enable_ws": false,
    // ...
}
```

若需要使用代理（VPN等），请参照 [代理部分](#using-proxy-with-freqtrade)。

!!! 信息 "逐步推出"  
    功能逐步实现，目前只支持OHLCV数据流，且仅部分交易所支持，未来会逐步扩展。

## 使用Dry-Run（模拟）模式

建议初次使用时，用Dry-Run模式，观察机器人表现及策略效果，避免实际风险。

操作步骤：

1. 修改 `config.json`，将 `dry_run` 设为 `true`  
2. 指定 `db_url`（如 `sqlite:///tradesv3.dryrun.sqlite`）  
3. 移除API密钥信息（或用空值/伪造值）  

示例：
```json
"dry_run": true,
"db_url": "sqlite:///tradesv3.dryrun.sqlite"
```

```json
"exchange": {
    "name": "binance",
    "key": "",
    "secret": ""
}
```

满足条件后即可切换到正式交易。

!!! 注意  
    Dry-Run时，会模拟钱包余额，起始值由 `dry_run_wallet` 设定（默认为1000）。  

### Dry-Run的注意事项

* API密钥可以提供也可以不提供（仅模拟操作，不影响账户）。  
* 钱包余额 (`/balance`) 是模拟值。  
* 订单模拟：未提交到交易所，仅在本地模拟确认。  
* 市价订单按订单簿 volume 即时成交，最大滑点5%。  
* 限价订单成交依价格和超时时间决定。  
* 若订单价格偏离超过1%，系统会将限价单转成市价单立即成交。  
* 若启用 `stoploss_on_exchange`，假设止损已成交。  
* 未平仓订单（非已完成的订单）在重启后会保持，假设未被执行。

## 切换到正式交易

切换至真实交易须保证：

1. 修改 `config.json`，将 `dry_run` 设为 `false`。  
2. 填入真实的API密钥信息（在交易所后台获取），不要用伪造的值。  

示例：
```json
{
    "exchange": {
        "name": "binance",
        "key": "你的API Key",
        "secret": "你的API Secret"
    }
}
```

建议为安全起见，使用专用的API密钥和不同的配置文件。

**强烈建议：**  
不要与公共配置文件共享你的私密信息。

## 配置交易所账户

在交易所后台创建API密钥。  
通常只在正式交易模式（`production`）需要。

运行在Dry-Run模式下，API密钥可以留空或用假数据。

## 切换到正式模式

修改 `config.json`：

```json
"dry_run": false,
```

填入交易所API密钥（用真实的）：

```json
{
    "exchange": {
        "name": "binance",
        "key": "你的真实API Key",
        "secret": "你的真实API Secret"
    }
}
```

阅读 [交易所文档](exchanges.md)，了解可能的配置细节。

### 保密建议

推荐使用第二份配置文件存放私密信息，譬如 `config-private.json`，  
使用方法：

```bash
freqtrade trade --config user_data/config.json --config user_data/config-private.json <...>
```

重要：绝不要泄露你的私密配置或API密钥！

## 使用代理（Proxy）设置

可通过环境变量设置代理（`HTTP_PROXY` 和 `HTTPS_PROXY`）：

```bash
export HTTP_PROXY="http://addr:port"
export HTTPS_PROXY="http://addr:port"
freqtrade
```

注意：交易所请求不受代理影响。

### 为交易所请求配置代理

在 `ccxt` 配置中加入代理参数：

```json
{ 
  "exchange": {
    "ccxt_config": {
      "httpsProxy": "http://addr:port",
      "wsProxy": "http://addr:port"
    }
  }
}
```

详见 [ccxt代理文档](https://docs.ccxt.com/#/README?id=proxy)。