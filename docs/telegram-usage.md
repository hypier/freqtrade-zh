# Telegram 使用指南

## 设置你的 Telegram 机器人

下面我们将介绍如何创建你的 Telegram 机器人，以及如何获取你的 Telegram 用户ID。

### 1. 创建你的 Telegram 机器人

与 [Telegram BotFather](https://telegram.me/BotFather) 开始聊天。

发送消息 `/newbot`。

*BotFather 回复：*

> 好的，新机器人。我们打算叫它什么名字？请为你的机器人选择一个名称。

选择你机器人的公共名称（例如：`Freqtrade bot`）

*BotFather 回复：*

> 很好。接下来为你的机器人选择一个用户名。它必须以 `bot` 结尾。例如：TetrisBot 或 tetris_bot。

选择你的机器人ID（如：`My_own_freqtrade_bot`），并发给 BotFather。

*BotFather 回复：*

> 完成！恭喜你创建了你的新机器人。你可以在 `t.me/yourbots_name_bot` 找到它。现在你可以为你的机器人添加描述、关于部分和头像，详见 /help 查看指令列表。顺便提一下，当你完成创建你的酷炫机器人后，如果想换个更好的用户名，可以联系我们的 Bot 支持。在你确保机器人完全正常运行后再进行此操作。

> 使用此令牌访问 HTTP API：`22222222:APITOKEN`

> 关于 Bot API 的详细信息，请参见此页面：[https://core.telegram.org/bots/api](https://core.telegram.org/bots/api)。Father bot 会返回你的 Token（API 密钥）。

复制 API Token（如上示例中的 `22222222:APITOKEN`），并在配置参数 `token` 中使用它。

别忘了点击 `/START` 按钮，开始与机器人对话。

### 2. Telegram 用户ID

#### 获取你的用户ID

与 [userinfobot](https://telegram.me/userinfobot) 对话。

获得你的“Id”，你将用它在配置参数 `chat_id` 中。

#### 使用群组ID

要获取群组ID，可以将机器人添加到群组中，启动 Freqtrade，并发出 `/tg_info` 命令。这会返回群组ID，无需使用随机机器人。

虽然“chat_id”仍然需要，但不必为此命令设置为特定的群组ID。

返回的响应还会包含必要的“topic_id”——格式已准备好供你复制粘贴到配置中。

``` json
 {
    "enabled": true,
    "token": "********",
    "chat_id": "-1001332619709",
    "topic_id": "122"
}
```

在 Freqtrade 配置中，可以使用完整的值（包括 `-`）作为字符串：

```json
   "chat_id": "-1001332619709"
```

!!! 警告 "使用 Telegram 群组"
    使用 Telegram 群组时，会让所有群组成员都能访问你的 Freqtrade 机器人以及所有通过 Telegram 能用的命令。请确保你信任群组中的每一位成员，以避免出现不愉快的意外。

##### 群组话题ID (topic_id)

如果你想在特定话题内使用机器人，可以在配置中使用 `topic_id` 参数。这允许你在群组的特定话题中使用机器人。  
没有这个参数时，如果群聊启用了话题，机器人将始终响应群组中的通用频道。

```json
   "chat_id": "-1001332619709",
   "topic_id": "3"
```

类似于群组ID，你可以通过 `/tg_info` 命令获取正确的 `topic_id`。

#### 授权用户

对群组来说，限制能给机器人发指令的用户很有用。

如果 `"authorized_users": []` 存在且为空，任何用户都无法控制机器人。
在下面的示例中，只有ID为 "1234567" 的用户可以控制机器人，其他用户只能接收消息。

```json
   "chat_id": "-1001332619709",
   "topic_id": "3",
   "authorized_users": ["1234567"]
```

## 控制 Telegram 通知噪音

Freqtrade 提供多种方式控制你的 Telegram 机器人的消息详尽程度。
每个设置的可能值如下：

* `on` - 消息会被发送，用户会收到通知。
* `silent` - 消息会被发送，但通知无声音/震动。
* `off` - 完全不发送此类消息。

示例配置展示不同设置：

``` json
"telegram": {
    "enabled": true,
    "token": "your_telegram_token",
    "chat_id": "your_telegram_chat_id",
    "allow_custom_messages": true,
    "notification_settings": {
        "status": "silent",
        "warning": "on",
        "startup": "off",
        "entry": "silent",
        "entry_fill": "on",
        "entry_cancel": "silent",
        "exit": {
            "roi": "silent",
            "emergency_exit": "on",
            "force_exit": "on",
            "exit_signal": "silent",
            "trailing_stop_loss": "on",
            "stop_loss": "on",
            "stoploss_on_exchange": "on",
            "custom_exit": "silent",  // 自定义退出原因无需特别明确
            "partial_exit": "on",
            // "custom_exit_message": "silent",  // 禁用个别自定义退出原因
            "*": "off"  // 禁用所有其他退出原因
        },
        // "exit": "off",  // 简单配置，禁用所有退出通知
        "exit_cancel": "on",
        "exit_fill": "off",
        "protection_trigger": "off",
        "protection_trigger_global": "on",
        "strategy_msg": "off",
        "show_candle": "off"
    },
    "reload": true,
    "balance_dust_level": 0.01
},
```

* `entry` 通知在订单下达时发出，`entry_fill` 在订单成交时发出。  
* `exit` 通知在订单下达时发出，`exit_fill` 在订单成交时发出。  
  退出消息（`exit` 和 `exit_fill`）也可以在单个退出理由层级进一步控制，使用退出原因作为键，默认全部为 `on`，也可以用特殊的 `*` 作为通配符，影响所有未明确指定的退出原因。  
* `*_fill` 通知默认关闭，需明确启用。  
* `protection_trigger` 在保护触发时发出通知，`protection_trigger_global` 在全局保护触发时发出通知。  
* `strategy_msg` - 来自策略的通知，使用策略中的 `self.dp.send_msg()` 发送[详见](strategy-customization.md#send-notification)。  
* `show_candle` - 在入场/离场消息中显示蜡烛图数值，值仅支持 `"ohlc"` 或 `"off"`。  
* `balance_dust_level` 定义 `/balance` 命令中“微尘”货币的最低余额，余额低于此值的货币会显示出来。  
* `allow_custom_messages` 完全禁用策略消息。  
* `reload` 可以禁用某些消息的重新载入按钮。

## 创建自定义快捷按钮（命令快捷键）

Telegram 允许我们创建带按钮的自定义键盘，用于快捷命令。

默认的自定义键盘如下：

```python
[
    ["/daily", "/profit", "/balance"], # 第一行，3个命令
    ["/status", "/status table", "/performance"], # 第二行，3个命令
    ["/count", "/start", "/stop", "/help"] # 第三行，4个命令
]
```

### 使用方法

你可以在 `config.json` 中自定义你的键盘：

``` json
"telegram": {
      "enabled": true,
      "token": "your_telegram_token",
      "chat_id": "your_telegram_chat_id",
      "keyboard": [
          ["/daily", "/stats", "/balance", "/profit"],
          ["/status table", "/performance"],
          ["/reload_config", "/count", "/logs"]
      ]
   },
```

!!! 提示 "支持的命令"
    只支持以下命令。命令参数不被支持！

    `/start`, `/pause`, `/stop`, `/status`, `/status table`, `/trades`, `/profit`, `/performance`, `/daily`, `/stats`, `/count`, `/locks`, `/balance`, `/stopentry`, `/reload_config`, `/show_config`, `/logs`, `/whitelist`, `/blacklist`, `/edge`, `/help`, `/version`, `/marketdir`

## Telegram 指令

默认情况下，Telegram 机器人会显示预定义的命令。有些命令
只能通过发送命令给机器人来使用。下表列出了官方指令。你可以随时通过 `/help` 索取帮助。

|  命令 | 描述 |
|----------|-------------|
| **系统指令** |
| `/start` | 启动交易机器人 |
| `/pause | /stopentry | /stopbuy` | 暂停交易机器人。会根据规则妥善处理未平仓交易。不会开启新仓位。 |
| `/stop` | 关闭交易机器人 |
| `/reload_config` | 重新加载配置文件 |
| `/show_config` | 展示部分当前配置及相关操作参数 |
| `/logs [limit]` | 查看最新的日志信息 |
| `/help` | 获取帮助信息 |
| `/version` | 查看版本号 |
| **状态相关** |
| `/status` | 列出所有未平仓交易 |
| `/status <trade_id>` | 查看一个或多个特定交易。多个 `<trade_id>` 用空格分隔。 |
| `/status table` | 以表格形式列出所有未平仓交易。待买订单用 `*` 标记，待卖订单用 `**` 标记。 |
| `/order <trade_id>` | 查看一个或多个特定交易的订单。多个 `<trade_id>` 用空格分隔。 |
| `/trades [limit]` | 以表格形式列出最近已关闭的交易。 |
| `/count` | 显示已用和可用的交易数 |
| `/locks` | 查看当前已锁定的交易对 |
| `/unlock <pair or lock_id>` | 解除该交易对（或指定锁ID）的锁定 |
| `/marketdir [long | short | even | none]` | 更新代表当前市场方向的用户管理变量。如果未提供方向，则显示当前设置。示例：`/marketdir long` |
| `/list_custom_data <trade_id> [key]` | 列出特定交易ID及键的自定义数据。如果未提供键，则列出所有相关的键值对。 |
| **修改交易状态** |
| `/forceexit <trade_id> | /fx <trade_id>` | 立即退出指定交易（忽略 `minimum_roi`） |
| `/forceexit all | /fx all` | 立即退出所有未平仓交易（忽略 `minimum_roi`） |
| `/fx` | `/forceexit` 的别名 |
| `/forcelong <pair> [rate]` | 立即买入指定交易对。Rate 为选填，仅限限价单（`force_entry_enable` 必须设为 True） |
| `/forceshort <pair> [rate]` | 立即做空指定交易对。Rate 为选填，仅限非现货市场（`force_entry_enable` 必须设为 True） |
| `/delete <trade_id>` | 从数据库中删除某笔交易，并尝试关闭其订单。需手动在交易所处理。 |
| `/reload_trade <trade_id>` | 从交易所重新加载该交易，只在实盘中有效，可能帮助恢复被手动卖出的交易。 |
| `/cancel_open_order <trade_id> | /coo <trade_id>` | 取消某笔交易的未平仓订单 |
| **指标信息** |
| `/profit [<n>]` | 统计已平仓交易的盈亏情况和绩效表现（最近 n 天，默认为全部） |
| `/performance` | 按交易对显示每笔已完成交易的绩效表现 |
| `/balance` | 显示每种币的机器人管理余额 |
| `/balance full` | 显示每种币的账户余额 |
| `/daily <n>` | 显示过去 n 天的每日盈亏（n 默认为7） |
| `/weekly <n>` | 显示过去 n 周的每周盈亏（从周一开始，n 默认为8） |
| `/monthly <n>` | 显示过去 n 个月的月度盈亏（n 默认为6） |
| `/stats` | 显示按退出原因划分的赢/亏情况，以及买入卖出的平均持有时间 |
| `/exits` | 同 `/stats`，按退出原因划分 |
| `/entries` | 同 `/stats`，显示买入/卖出的赢/亏状态 |
| `/whitelist [sorted] [baseonly]` | 查看当前白名单，支持按字母排序，或仅显示基础货币。 |
| `/blacklist [pair]` | 查看当前黑名单，或添加某个交易对到黑名单，支持多个交易对用空格隔开。使用 `/reload_config` 重置黑名单。 |
| `/edge` | 查看由 Edge 验证过的交易对及其相应的胜率、预期盈亏和止损值。 |

## Telegram 指令示例

以下是每个命令你会收到的示例消息。

### /start

> **状态：** `running`

### /pause | /stopentry | /stopbuy

> **状态：** `paused, no more entries will occur from now. Run /start to enable entries.`

暂停策略，阻止开启新仓位，已开启的仓位会根据规则继续管理（ROI/退出信号、止损等）。  
注意，仓位调整仍会进行，但只限于退出方面——意味着当机器人处于 `paused` 时，只能缩减未平仓仓位的规模。

暂停后，等待机器人关闭所有仓位（可用 `/status table` 查看），所有仓位关闭后，可发 `/stop` 完全停止机器人。

用 `/start` 恢复运行状态，允许开新仓。

!!! 警告
    暂停/停止入场信号仅在机器人运行时有效，且不会持久化，重启机器人后会重置。

### /stop

> `Stopping trader ...`  
> **状态：** `stopped`

### /status

机器人会为每个未平仓交易发送如下信息。策略中的入场标签（Enter Tag）可配置。

> **交易ID：** `123`（已持有 1 天）  
> **当前对：** CVC/BTC  
> **方向：** 多头  
> **杠杆：** 1.0  
> **金额：** `26.64180098`  
> **入场标签：** Awesome Long Signal  
> **开仓价格：** `0.00007489`  
> **当前价格：** `0.00007489`  
> **未实现盈亏：** `12.95%`  
> **止损：** `0.00007389 (-0.02%)`

### /status table

以表格形式返回所有未平仓交易的状态。

```
ID L/S    Pair     Since   Profit
----    --------  -------  --------
  67 L   SC/BTC    1 d      13.33%
 123 S   CVC/BTC   1 h      12.95%
```

### /count

返回已用和最大交易数。

```
current    max
---------  -----
     2     10
```

### /profit

返回你的盈亏总结以及绩效。

> **ROI：** 已平仓交易  
>   ∙ `0.00485701 BTC (2.2%) (15.2 Σ%)`  
>   ∙ `62.968 USD`  
> **ROI：** 所有交易  
>   ∙ `0.00255280 BTC (1.5%) (6.43 Σ%)`  
>   ∙ `33.095 EUR`  
>  
> **总交易数：** `138`  
> **机器人启动时间：** `2022-07-11 18:40:44`  
> **首笔交易开启时间：** `3 天前`  
> **最新交易开启时间：** `2 分钟前`  
> **平均持有时间：** `2:33:45`  
> **表现最佳：** `PAY/BTC：50.23%`  
> **交易量：** `0.5 BTC`  
> **盈利因子：** `1.04`  
> **赢/亏：** `102 / 36`  
> **胜率：** `73.91%`  
> **期望值（比率）：** `4.87 (1.66)`  
> **最大回撤：** `9.23% (0.01255 BTC)`

相对利润为 `1.2%`，为每笔交易的平均利润。  
相对利润的 `15.2 Σ%` 是基于起始资金计算的——即：`0.00485701 * 1.152 ≈ 0.00738 BTC`。  
**起始资金**（`starting capital`）可以取自 `available_capital` 设置，或通过当前钱包余额和利润计算得出。  
**利润因子**（Profit Factor）为总利润除以总亏损，作为策略的总体衡量指标。  
**期望值**代表每单位风险币的平均回报，即胜率和风险回报比（赢的交易平均收益与亏的交易平均损失之比）。  
**期望比率（Expectancy Ratio）**是基于所有历史交易的表现，预测下一笔交易的预期盈亏。  
**最大回撤**对应回测指标 `Absolute Drawdown (Account)`，计算公式为 `(Absolute Drawdown) / (DrawdownHigh + startingBalance)`。  
**机器人启动时间**指首次启动机器人的时间，对于较早的机器人，则默认为首次交易的开启时间。

### /forceexit <trade_id>

> **BINANCE:** Exiting BTC/LTC with limit `0.01650000 (profit: ~-4.07%, -0.00008168)`

!!! 提示
    可以调用 `/forceexit` 不带参数，列出所有未平仓交易的按钮，直接退出某笔交易。
    该命令亦有 `/fx` 的别名，功能相同，但在紧急情况下输入更快捷。

### /forcelong <pair> [rate] | /forceshort <pair> [rate]

支持 `/forcebuy <pair> [rate]` 用于多头，但建议逐步弃用。

> **BINANCE:** 以限价 `0.03400000` 买入 ETH/BTC（`1.000000 ETH`，USD 225.290）

省略交易对时，会弹出请求所需交易对（基于当前白名单）。  
通过 `/forcelong` 创建的订单会带有 `force_entry` 标签。

![Telegram 强制买入截图](assets/telegram_forcebuy.png)

请注意，要使用此功能，必须将 `force_entry_enable` 设置为 `true`。

[更多详情](configuration.md#understand-force_entry_enable)

### /performance

显示机器人已卖出的每个加密货币的绩效表现。
> 绩效：  
> 1. `RCN/BTC 0.003 BTC (57.77%) (1)`  
> 2. `PAY/BTC 0.0012 BTC (56.91%) (1)`  
> 3. `VIB/BTC 0.0011 BTC (47.07%) (1)`  
> 4. `SALT/BTC 0.0010 BTC (30.24%) (1)`  
> 5. `STORJ/BTC 0.0009 BTC (27.24%) (1)`  
> ...  

相对表现基于在该币种的总投资额，汇总所有已完成的交易。

### /balance

返回你在交易所中的所有加密货币余额。

> **币种：** BTC  
> **可用：** 3.05890234  
> **余额：** 3.05890234  
> **待确认：** 0.0  
>
> **币种：** CVC  
> **可用：** 86.64180098  
> **余额：** 86.64180098  
> **待确认：** 0.0  

### /daily <n>

默认返回最近 7 天的每日盈亏。以下为 `/daily 3` 的示例：

> **最近 3 天的每日盈亏：**

```
天数（计数）   USDT           USD           盈亏比例 %
--------------  ------------  ----------  ----------
2022-06-11 (1)  -0.746 USDT  -0.75 USD  -0.08%
2022-06-10 (0)   0 USDT       0.00 USD    0.00%
2022-06-09 (5)  20 USDT      20.10 USD   5.00%
```

### /weekly <n>

默认返回包含当周在内的最近 8 周的周度盈亏，周一为一周的起点。以下为 `/weekly 3` 示例：

> **最近 3 周的周盈亏（从周一开始）：**

```
星期一（计数）  盈亏 BTC      盈亏 USD    盈亏比例 %
--------------  --------------  ----------  ----------
2018-01-03 (5)  0.00224175 BTC  29142 USD  4.98%
2017-12-27 (1)  0.00033131 BTC  4307 USD   0.00%
2017-12-20 (4)  0.00269130 BTC  34986 USD  5.12%
```

### /monthly <n>

默认返回最近 6 个月（含本月）的月度盈亏。以下为 `/monthly 3` 示例：

> **最近 3 个月的月盈亏：**

```
月份（计数）  盈亏 BTC      盈亏 USD    盈亏比例 %
--------------  --------------  ----------  ----------
2018-01 (20)    0.00224175 BTC  29142 USD  4.98%
2017-12 (5)    0.00033131 BTC   4307 USD   0.00%
2017-11 (10)   0.00269130 BTC   34986 USD  5.10%
```

### /whitelist

展示当前白名单。

> 使用白名单 `StaticPairList`，包含 22 个交易对  
> `IOTA/BTC, NEO/BTC, TRX/BTC, VET/BTC, ADA/BTC, ETC/BTC, NCASH/BTC, DASH/BTC, XRP/BTC, XVG/BTC, EOS/BTC, LTC/BTC, OMG/BTC, BTG/BTC, LSK/BTC, ZEC/BTC, HOT/BTC, IOTX/BTC, XMR/BTC, AST/BTC, XLM/BTC, NANO/BTC`

### /blacklist [pair]

展示当前黑名单。  
如果指定交易对（pair），则将该对加入黑名单。支持多个交易对，用空格隔开。  
使用 `/reload_config` 重置黑名单。

> 使用黑名单 `StaticPairList`，包含 2 个交易对：  
> `DODGE/BTC`, `HOT/BTC`。

### /edge

显示由 Edge 验证的交易对及其对应的胜率、预期盈亏和止损值。

> **Edge验证的交易对：**
```
Pair        Winrate    Expectancy    Stoploss
--------  ---------  ------------  ----------
DOCK/ETH   0.522727      0.881821       -0.03
PHX/ETH    0.677419      0.560488       -0.03
HOT/ETH    0.733333      0.490492       -0.03
HC/ETH     0.588235      0.280988       -0.02
ARDR/ETH   0.366667      0.143059       -0.01
```

### /version

> **版本：** `0.14.3`

### /marketdir

如果提供市场方向参数，该命令会更新代表当前市场方向的用户管理变量。  
此变量在机器人启动时默认未设置，需由用户自行设定。示例：`/marketdir long`。

如果未提供参数，则输出当前设置：

```
当前市场方向：even
```

你可以在策略中通过 `self.market_direction` 使用该变量。

!!! 警告 "机器人重启"
    请注意，市场方向不进行持久化，重启或重新加载后会重置。

!!! 危险 "回测"
    此值/变量旨在在模拟/实盘交易中手动调整。  
    使用 `market_direction` 的策略可能无法产生可靠的、可复现的结果（更改不会影响回测结果）。请自行承担风险。