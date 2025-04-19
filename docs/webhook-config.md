# Webhook 使用方法

## 配置

通过在配置文件中添加 webhook 部分，并将 `webhook.enabled` 设置为 `true`，即可启用 webhook。

示例配置（经过 IFTTT 测试）如下：

```json
  "webhook": {
        "enabled": true,
        "url": "https://maker.ifttt.com/trigger/<YOUREVENT>/with/key/<YOURKEY>/",
        "entry": {
            "value1": "Buying {pair}",
            "value2": "limit {limit:8f}",
            "value3": "{stake_amount:8f} {stake_currency}"
        },
        "entry_cancel": {
            "value1": "Cancelling Open Buy Order for {pair}",
            "value2": "limit {limit:8f}",
            "value3": "{stake_amount:8f} {stake_currency}"
        },
        "entry_fill": {
            "value1": "Buy Order for {pair} filled",
            "value2": "at {open_rate:8f}",
            "value3": ""
        },
        "exit": {
            "value1": "Exiting {pair}",
            "value2": "limit {limit:8f}",
            "value3": "profit: {profit_amount:8f} {stake_currency} ({profit_ratio})"
        },
        "exit_cancel": {
            "value1": "Cancelling Open Exit Order for {pair}",
            "value2": "limit {limit:8f}",
            "value3": "profit: {profit_amount:8f} {stake_currency} ({profit_ratio})"
        },
        "exit_fill": {
            "value1": "Exit Order for {pair} filled",
            "value2": "at {close_rate:8f}.",
            "value3": ""
        },
        "status": {
            "value1": "Status: {status}",
            "value2": "",
            "value3": ""
        }
    },
```

`webhook.url` 中的链接应指向你对应的 webhook 地址。如果你使用 [IFTTT](https://ifttt.com)（如样例所示），请在 URL 中插入你的事件名和密钥。

你可以将 POST 请求体格式设置为表单编码（默认）、JSON 编码或原始数据。分别使用 `"format": "form"`、`"format": "json"` 或 `"format": "raw"`。以下为 Mattermost 云端集成的示例配置：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURSUBDOMAIN>.cloud.mattermost.com/hooks/<YOURHOOK>",
        "format": "json",
        "status": {
            "text": "Status: {status}"
        }
    },
```

这样会生成一个 POST 请求，内容如 `{"text":"Status: running"}`，请求头为 `Content-Type: application/json`，可在 Mattermost 频道中显示“Status: running”。

在使用表单编码或 JSON 编码配置时，你可以配置任意数量的有效负载字段，键和值都会在 POST 请求中输出。但在使用原始数据格式时，只能配置一个值，而且该值必须命名为 `"data"`。在这种情况下，POST 请求中不会输出 `data` 这个键，只会输出其值。例如：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "format": "raw",
        "webhookstatus": {
            "data": "Status: {status}"
        }
    },
```

此时会生成一个 POST 请求，内容如 `Status: running`，请求头为 `Content-Type: text/plain`。

## 其他配置

`webhook.retries` 参数可设置 webhook 请求在失败（即 HTTP 响应状态非 200）时的最大重试次数。默认为 `0`，代表不重试。还可以设置 `webhook.retry_delay`，指定重试间隔的秒数，默认为 `0.1`（即100毫秒）。请注意，增加重试次数或重试间隔可能会影响交易速度，尤其在 webhook 网络连接不畅时。

你还可以设置 `webhook.timeout`，定义请求等待响应的最长时间（默认10秒）。

重试示例配置如下：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "timeout": 10,
        "retries": 3,
        "retry_delay": 0.2,
        "status": {
            "status": "Status: {status}"
        }
    },
```

你可以在策略内通过调用 `self.dp.send_msg()` 向 Webhook 端点发送自定义消息。启用此功能请将 `allow_custom_messages` 设为 `true`：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "allow_custom_messages": true,
        "strategy_msg": {
            "status": "StrategyMessage: {msg}"
        }
    },
```

不同事件可以配置不同的负载内容。不是每个字段都必须配置，但至少应配置其中一个字典，否则 webhook 将不会被调用。

## Webhook 消息类型

### 交易入口（Entry）

`webhook.entry` 中的字段在机器人执行多空（long/short）交易时会被填充。参数通过 `string.format` 填充。可能的参数包括：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* ~~`limit`（已弃用，不再使用）~~
* `open_rate`
* `amount`
* `open_date`
* `stake_amount`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `order_type`
* `current_rate`
* `enter_tag`

### 交易取消（Entry cancel）

`webhook.entry_cancel` 中的字段在机器人取消多空订单时会被填充。参数通过 `string.format` 填充。可能参数如下：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `limit`
* `amount`
* `open_date`
* `stake_amount`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `order_type`
* `current_rate`
* `enter_tag`

### 交易已成交（Entry fill）

`webhook.entry_fill` 中的字段在机器人完成多空订单时会被填充。参数通过 `string.format` 填充。可能参数如下：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `open_rate`
* `amount`
* `open_date`
* `stake_amount`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `order_type`
* `current_rate`
* `enter_tag`

### 退出交易（Exit）

`webhook.exit` 中的字段在机器人退出交易时会被填充。参数通过 `string.format` 填充。可能参数如下：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `limit`
* `amount`
* `open_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`

### 退出订单已完成（Exit fill）

`webhook.exit_fill` 中的字段在机器人完成退出订单（平仓）后会被填充。参数通过 `string.format` 填充。可能参数如下：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `close_rate`
* `amount`
* `open_rate`
* `current_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`

### 退出订单取消（Exit cancel）

`webhook.exit_cancel` 中的字段在机器人取消退出订单时会被填充。参数通过 `string.format` 填充。可能参数如下：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `limit`
* `amount`
* `open_rate`
* `current_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`

### 状态消息（Status）

`webhook.status` 用于普通状态消息（如 Started / Stopped / ...），参数通过 `string.format` 填充。

唯一的值为 `{status}`。

## Discord

为 Discord 提供了特殊的 Webhook 支持。

你可以按如下配置：

```json
"discord": {
    "enabled": true,
    "webhook_url": "https://discord.com/api/webhooks/<Your webhook URL ...>",
    "exit_fill": [
        {"Trade ID": "{trade_id}"},
        {"Exchange": "{exchange}"},
        {"Pair": "{pair}"},
        {"Direction": "{direction}"},
        {"Open rate": "{open_rate}"},
        {"Close rate": "{close_rate}"},
        {"Amount": "{amount}"},
        {"Open date": "{open_date:%Y-%m-%d %H:%M:%S}"},
        {"Close date": "{close_date:%Y-%m-%d %H:%M:%S}"},
        {"Profit": "{profit_amount} {stake_currency}"},
        {"Profitability": "{profit_ratio:.2%}"},
        {"Enter tag": "{enter_tag}"},
        {"Exit Reason": "{exit_reason}"},
        {"Strategy": "{strategy}"},
        {"Timeframe": "{timeframe}"},
    ],
    "entry_fill": [
        {"Trade ID": "{trade_id}"},
        {"Exchange": "{exchange}"},
        {"Pair": "{pair}"},
        {"Direction": "{direction}"},
        {"Open rate": "{open_rate}"},
        {"Amount": "{amount}"},
        {"Open date": "{open_date:%Y-%m-%d %H:%M:%S}"},
        {"Enter tag": "{enter_tag}"},
        {"Strategy": "{strategy} {timeframe}"},
    ]
}
```

以上为默认设置（`exit_fill` 和 `entry_fill` 为可选，若不需要可将其置为空数组 `[]`），随意修改。要禁用某一默认值（`entry_fill` / `exit_fill`），可将其设置为空数组。

可用字段对应 webhook 中的字段，详细请参见相关 webhook 部分。

默认通知效果如下图所示。

![discord-notification](assets/discord_notification.png)

你也可以通过调用 `dataprovider.send_msg()` 函数，从策略向 Discord 端点发送自定义消息。启用此功能请将 `allow_custom_messages` 设为 `true`：

```json
  "discord": {
        "enabled": true,
        "webhook_url": "https://discord.com/api/webhooks/<Your webhook URL ...>",
        "allow_custom_messages": true,
    },
```