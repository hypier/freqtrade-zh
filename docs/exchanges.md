# 交易所特定注意事项

本页面整合了常见陷阱和信息，这些内容是针对特定交易所的，可能不适用于其他交易所。

## 交易所配置

Freqtrade基于 [CCXT库](https://github.com/ccxt/ccxt)，它支持超过100个加密货币交易市场和交易API。最新完整列表可在 [CCXT仓库首页](https://github.com/ccxt/ccxt/tree/master/python) 查看。
然而，开发团队只在少数几个交易所上进行了测试。当前这些交易所的列表可在本文档的“首页”部分找到。

欢迎自行测试其他交易所，并提交反馈或PR，以改进机器人或确认哪些交易所运行顺畅。

部分交易所需要特殊配置，具体见下文。

### 示例交易所配置

举例“binance”交易所的配置如下：

```json
"exchange": {
    "name": "binance",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "ccxt_config": {},
    "ccxt_async_config": {},
    // ... 
}
```

### 设置速率限制

通常，CCXT设定的速率限制是可靠且效果良好的。
如果遇到与速率限制相关的问题（通常在日志中显示为DDOS异常），可以轻松将rateLimit参数调整到其他值。

```json
"exchange": {
    "name": "kraken",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "ccxt_config": {"enableRateLimit": true},
    "ccxt_async_config": {
        "enableRateLimit": true,
        "rateLimit": 3100
    },
}
```

该配置启用了kraken交易所的速率限制，以避免被封禁。
`"rateLimit": 3100`表示每次调用间隔3110毫秒（即3.1秒）。可以通过设置`"enableRateLimit"`为`false`来完全禁用。

!!! 注意
    最佳的速率限制设置依赖于具体交易所和白名单规模，因此理想参数会因其他设置不同而异。
    我们会在可能的情况下为每个交易所提供合理默认值，如果遇到封禁，请确保已启用`"enableRateLimit"`，并逐步增加`"rateLimit"`的值。

## Binance

!!! 警告 “服务器位置及地理IP限制”
    请注意，Binance根据服务器所在国家限制API访问。目前，非详尽列出的封禁国家包括加拿大、马来西亚、荷兰和美国。请访问 [binance条款 > b. 资格](https://www.binance.com/en/terms) 获取最新封禁国家列表。

Binance支持 [time_in_force](configuration.md#understand-order_time_in_force)。

!!! 提示 “交易所止损”
    Binance支持`stoploss_on_exchange`，并使用`stop-loss-limit`订单。此特性带来很大优势，建议启用交易所止损。
    在期货市场，Binance支持`stop-limit`和`stop-market`订单。可在`order_types.stoploss`配置中使用`"limit"`或`"market"`来选择使用的订单类型。

### Binance黑名单建议

建议将`"BNB/<STAKE>"`加入黑名单，以避免在交易过程中出现问题，除非你准备在账户中持有足够的额外BNB或打算禁用用BNB支付手续费。
Binance账户可以用BNB支付手续费，如果某次交易的手续费用BNB支付，后续交易可能会消耗这部分BNB，导致最初的BNB交易无法卖出（因为剩余数量不足）。

如果没有足够的BNB支付交易费，费用不会由BNB覆盖，也不会减少。Freqtrade绝不会主动用BNB支付手续费，BNB需自行手动购买和监控。

### Binance站点

Binance已拆分为两个站点，用户必须使用对应的ccxt交易所ID，否则API密钥将无法识别。

* [binance.com](https://www.binance.com/) — 国际用户。使用交易所ID：`binance`。
* [binance.us](https://www.binance.us/) — 美国用户。使用交易所ID：`binanceus`。

### Binance RSA密钥

Freqtrade支持Binance的RSA API密钥。

建议将其作为环境变量使用：

```bash
export FREQTRADE__EXCHANGE__SECRET="$(cat ./rsa_binance.private)"
```

也可以在配置文件中配置。由于json不支持多行字符串，需将所有换行符用`\n`替换，构成有效的json格式。

```json
// ...
"key": "<someapikey>",
"secret": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBABACAFQA<...>s8KX8=\n-----END PRIVATE KEY-----"
// ...
```

### Binance期货

Binance有特定（且较复杂的）[期货交易量化规则](https://www.binance.com/en/support/faq/4f462ebe6ff445d4a170be7d9e897272)，必须遵循，否则可能因委托金额太低等原因限制交易。
在Binance期货市场交易时，必须使用订单簿数据，因为没有期货的价格行情。

```jsonc
  "entry_pricing": {
      "use_order_book": true,
      "order_book_top": 1,
      "check_depth_of_market": {
          "enabled": false,
          "bids_to_ask_delta": 1
      }
  },
  "exit_pricing": {
      "use_order_book": true,
      "order_book_top": 1
  },
```

#### Binance隔离期货设置

用户还需将“仓位模式”设置为“One-way Mode”，将“资产模式”设置为“Single-Asset Mode”。
这些设置在启动时会被检查，若设置错误，Freqtrade会显示错误信息。

![](assets/binance_futures_settings.png)

Freqtrade不会自动修改这些设置。

#### Binance BNFCR期货

BNFCR模式是在Binance为规避欧洲监管问题的一种特殊期货模式。
使用BNFCR期货时，需满足以下配置组合：

```jsonc
{
    // ...
    "trading_mode": "futures",
    "margin_mode": "cross",
    "proxy_coin": "BNFCR",
    "stake_currency": "USDT" // 或"USDC"
    // ...
}
```

其中`stake_currency`定义了机器人操作的市场类别，此选择较为随意。

在交易所端，需使用“多资产模式”并将“仓位模式”设为“One-way Mode”。  
Freqtrade会在启动时检查这些设置，但不会自行调整。

## Bingx

BingX支持 [time_in_force](configuration.md#understand-order_time_in_force)，包括 "GTC"（持仓 until 取消）、"IOC"（立即成交或取消）和 "PO"（Post only）等设置。

!!! 提示 “交易所止损”
    Bingx支持`stoploss_on_exchange`，可以使用止损限价单或止损市价单。强烈推荐启用此功能，以享受交易所止损带来优势。

## Kraken

Kraken支持 [time_in_force](configuration.md#understand-order_time_in_force)，包括 "GTC"（持仓直到取消）、"IOC"（立即成交否则取消）和 "PO"（Post only）。

!!! 提示 “交易所止损”
    Kraken支持`stoploss_on_exchange`，可使用止损市价单和止损限价单。推荐启用交易所止损。
    在`order_types.stoploss`配置中可以选择`"limit"`或`"market"`。

### 历史Kraken数据

Kraken API仅提供最多720个历史蜡烛线，适合Freqtrade的干运行和实盘交易，但不适合回测。
若需下载Kraken交易所的数据，必须使用`--dl-trades`参数，否则机器人会反复下载相同的720根蜡烛线，导致数据不足。

为了加快下载速度，可下载Kraken提供的[交易zip文件](https://support.kraken.com/hc/en-us/articles/360047543791-Downloadable-historical-market-data-time-and-sales-)。
这些文件一般每季度更新一次。建议将它们放在`user_data/data/kraken/trades_csv`目录。

使用增量文件的结构如下（即将完整历史存放在一个目录中，增量文件存放在不同目录中）：

```
└── trades_csv
    ├── Kraken_full_history
    │   ├── BCHEUR.csv
    │   └── XBTEUR.csv
    ├── Kraken_Trading_History_Q1_2023
    │   ├── BCHEUR.csv
    │   └── XBTEUR.csv
    └── Kraken_Trading_History_Q2_2023
        ├── BCHEUR.csv
        └── XBTEUR.csv
```

可以将这些文件转换成Freqtrade识别的格式：

```bash
freqtrade convert-trade-data --exchange kraken --format-from kraken_csv --format-to feather
# 转换为不同时间周期的ohlcv交易数据
freqtrade trades-to-ohlcv -p BTC/EUR BCH/EUR --exchange kraken -t 1m 5m 15m 1h
```

转换后数据还能加快下载速度——会从最新的交易开始下载。

```bash
freqtrade download-data --exchange kraken --dl-trades -p BTC/EUR BCH/EUR
```

!!! 警告 “从Kraken下载数据”
    下载Kraken数据会占用比其他交易所更多的内存（RAM），因为需要在本地转换成蜡烛线格式。
    过程会比较耗时，因为Freqtrade必须下载每笔交易的详细信息，耗时较长，请耐心等待。

!!! 警告 “rateLimit调优”
    请注意，`rateLimit`配置项指的是请求之间的延迟（毫秒），而非请求速率（请求/秒）。
    为避免Kraken API的“超出请求限制”异常，建议调高此参数，不要调低。

## Kucoin

Kucoin的API密钥需包含“密码（passphrase）”，配置示例如下：

```json
"exchange": {
    "name": "kucoin",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

Kucoin支持 [time_in_force](configuration.md#understand-order_time_in_force)。

!!! 提示 “交易所止损”
    支持`stoploss_on_exchange`，可用止损限价和止损市价单。强烈建议启用交易所止损。
    在`order_types.stoploss`中使用`"limit"`或`"market"`来选择订单类型。

### Kucoin黑名单

建议将`"KCS/<STAKE>"`加入黑名单，以避免出现问题，除非你愿意在账户中持有足够的额外KCS或决定禁用KCS支付手续费。
Kucoin账户可以用KCS支付手续费，如果某交易用到KCS，后续交易可能会消耗这部分KCS，导致最初的KCS交易无法卖出（因为剩余不足以支付手续费）。

## HTX

!!! 提示 “交易所止损”
    HTX支持`stoploss_on_exchange`，使用`stop-limit`订单。强烈建议开启交易所止损功能以保护仓位。

## OKX

OKX的API密钥也需要包含“密码（passphrase）”，配置示例：

```json
"exchange": {
    "name": "okx",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

在`my.okx.com`注册的API，只能用`"myokx"`作为交易所ID，否则会出现“OKX Error 50119: API key doesn't exist”错误。

!!! 警告
    OKX每次API调用最多返回100个蜡烛线，因此在回测模式下，策略可用的数据较少。

!!! 警告 “期货”
    OKX期货支持“仓位模式”——可以是“买卖模式”或多空对冲（Hedge Mode）。
    Freqtrade支持这两种模式（推荐使用“买卖模式”），但在交易中途切换模式会引发异常和错误。
    OKX仅提供过去大约3个月的MARK蜡烛线数据，回测更早时期的期货数据可能会存在偏差；因为没有这些数据，funding费无法正确计算。

## Gate.io

!!! 提示 “交易所止损”
    Gate.io支持`stoploss_on_exchange`，可用`stop-loss-limit`订单。建议开启交易所止损以保护仓位。

Gate.io支持使用`POINT`支付手续费，因该币种不在常规市场中，自动手续费计算会失败，默认为0。
可以通过配置`exchange.unknown_fee_rate`参数设置Point与基础货币的比例关系。更换基础货币时也需调整此参数。

使用Gate API密钥时，需要授权以下权限（除市场权限外）：

* “现货交易” 或 “永续合约” （读写）
* “钱包” （只读）
* “账户” （只读）

没有这些权限，机器人无法正常启动，可能会提示“permission missing”。

## Bybit

目前支持Bybit的USDT市场期货交易，采用隔离期货模式。

启动时，Freqtrade会将仓位模式设为“单向仓位（One-way Mode）”，避免反复调用（减少延迟），但更改此设置会导致异常。

由于Bybit不提供资金费历史，实际交易也采用干运行的计算方式。

实盘交易API密钥必须具备以下权限：

* 读写权限
* 合约 - 订单
* 合约 - 仓位

强烈建议仅用来自信任IP的API密钥。

!!! 警告 “统一账户”
    Freqtrade假设账户专属于机器人使用。
    建议每个机器人使用单独的子账户，特别是在使用“统一账户（Unified Account）”时。
    不支持同时用多个机器人或在机器人账户进行人工交易，可能导致异常行为。

!!! 提示 “交易所止损”
    Bybit（仅限期货）支持`stoploss_on_exchange`，使用`stop-loss-limit`订单，强烈推荐开启以保护仓位。
    在期货中支持`stop-limit`和`stop-market`订单，可在`order_types.stoploss`选择。

## Bitmart

Bitmart要求API密钥附带Memo（即API密钥的自定义名称）以及账号UID。

```json
"exchange": {
    "name": "bitmart",
    "uid": "your_bitmart_api_key_memo",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

!!! 警告 “必需的验证”
    Bitmart要求API交易的验证等级至少达到Lvl2，即使通过UI交易仅需Lvl1验证。

## Hyperliquid

!!! 提示 “交易所止损”
    Hyperliquid支持`stoploss_on_exchange`，可用`stop-loss-limit`订单。建议启用以增强安全。

Hyperliquid是去中心化交易所（DEX），其交易机制与普通交易所不同。无需通过API密钥认证，而需用你的钱包私钥签名（建议使用API钱包，生成于Hyperliquid或你信任的钱包）。

配置示例：

```json
"exchange": {
    "name": "hyperliquid",
    "walletAddress": "your_eth_wallet_address",
    "privateKey": "your_api_private_key",
    // ...
}
```

* `walletAddress`（十六进制）：`0x<40个十六进制字符>`，可以从钱包中复制，确保是你的钱包地址而非API钱包地址。
* `privateKey`（十六进制）：`0x<64个十六进制字符>`，使用API钱包生成时显示的私钥。

Hyperliquid在Arbitrum One链上处理存取款，此链为以太坊的Layer 2扩展方案。存入USDC的操作需经过几个步骤，详情参见 [Hyperliquid入门指南](https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/how-to-start-trading)。

!!! 注意 “Hyperliquid常用注意事项”
    Hyperliquid不支持市价单，但ccxt会模拟市价单，方法是在限价单中设置最大滑点为5%。  
    由于Hyperliquid只提供最多5000个历史蜡烛线，回测时需自己逐步构建蜡烛线（等待并逐步下载数据），否则只能用到最后的5000根蜡烛线。

!!! 其他信息 “一些常用最佳实践（非详尽）”
    * 小心供应链攻击，比如pip包污染等问题。使用私钥时确保环境安全。
    * 不要用实际钱包私钥进行交易。可用Hyperliquid [API生成器](https://app.hyperliquid.xyz/API)创建单独的API钱包。
    * 不要在你用Freqtrade的服务器上存储实际钱包私钥，只用API钱包私钥，该密钥仅用于交易，不允许提取资金。
    * 必须妥善保管助记词和私钥。
    * 不要用在硬件钱包初始化时备份的助记词，否则会危及硬件钱包的安全性。
    * 创建不同的软钱包，将你要交易的资金转入该钱包，只用它进行Hyperliquid交易。
    * 若有部分资金不打算用于交易（比如盈利后），可转回硬件钱包。

### Hyperliquid历史数据

Hyperliquid API不会提供超出单次请求的历史数据，因此无法批量下载历史数据，下载的内容不能作为真正的历史数据。

---

## 所有交易所

如果你经常遇到Nonce相关的错误（如`InvalidNonce`），建议重新生成API密钥。重置Nonce较困难，通常直接重新生成密钥更为便捷。

## 其他交易所的随机注意事项

* Ocean（交易所ID：`theocean`）使用Web3功能，需安装`web3` python包：

```bash
pip3 install web3
```

### 获取最新价格/不完整蜡烛

多数交易所的OHLCV/klines接口会返回当前未完成的蜡烛线。
默认情况下，Freqtrade会假设最后一根蜡烛线是不完整的，并将其去除。

可用[贡献者文档中的辅助脚本](developer.md#incomplete-candles)检测你的交易所是否返回未完整蜡烛。

由于重绘风险，Freqtrade不会允许你用未完成蜡烛线。

但如果你需要使用最新价格信息（比如策略需依赖实时价格），可以通过策略内的[数据提供者](strategy-customization.md#possible-options-for-dataprovider)获取。

### 高级交易所配置

可以使用 `_ft_has_params` 设置覆盖默认值和交易所特定行为。

示例：在Kraken中测试`FOK`订单类型，并将蜡烛数量限制设为200（每次API调用最多返回200根蜡烛）：

```json
"exchange": {
    "name": "kraken",
    "_ft_has_params": {
        "order_time_in_force": ["GTC", "FOK"],
        "ohlcv_candle_limit": 200
        }
    //...
}
```

!!! 警告
    修改这些设置前，请确保完全理解其影响和风险。