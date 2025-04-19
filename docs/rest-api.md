# REST API

## FreqUI

FreqUI 现在拥有专属的 [文档部分](freq-ui.md) —— 请参考该部分了解有关 FreqUI 的所有信息。

## 配置

通过在配置文件中添加 `api_server` 部分并将 `api_server.enabled` 设置为 `true`，启用 rest API。

示例配置：

``` json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "127.0.0.1",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "somethingrandom",
        "CORS_origins": [],
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        "ws_token": "sercet_Ws_t0ken"
    }
}
```

!!! Danger "安全警告"
    默认情况下，配置仅监听本地主机（localhost）（因此无法被其他系统访问）。强烈建议不要将此 API 公开到互联网，并且应选择强大且唯一的密码，因为其他人可能会控制你的机器人。

??? Note "远程服务器上的 API/UI 访问"
    如果你在 VPS 上运行，应考虑使用 SSH 隧道，或设置 VPN（OpenVPN、WireGuard）连接到你的机器人。
    这样可以确保频UI不会直接暴露在互联网中，为安全起见（频UI默认不支持 HTTPS）。
    这些工具的设置不在本教程范围内，网络上有许多优质教程可以参考。

你可以通过在浏览器中访问 `http://127.0.0.1:8080/api/v1/ping` 来检查 API 是否正常运行。  
应返回以下响应：

``` output
{"status":"pong"}
```

其他所有端点返回敏感信息，且需要认证，因此不能通过网页浏览器访问。

### 安全性

为了生成安全的密码，建议使用密码管理器，或者使用以下代码：

``` python
import secrets
secrets.token_hex()
```

!!! Hint "JWT 令牌"
    同样的方法还可以用来生成 JWT 秘钥（`jwt_secret_key`）。

!!! Danger "密码选择"
    请确保选择一个非常强大且唯一的密码，以保护你的机器人免受未授权访问，同时将 `jwt_secret_key` 更换为随机的内容（不用记住，但它将用于加密你的会话，所以应确保其唯一性）！

### 与 Docker 配合的配置

如果你使用 Docker 运行你的机器人，需要让机器人监听传入连接，安全由 Docker 处理。

``` json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!"
        //...
    }
}
```

确保你的 `docker-compose.yml` 文件中包含以下两行：

``` yml
ports:
  - "127.0.0.1:8080:8080"
```

!!! Danger "安全警告"
    将端口映射配置为 `"8080:8080"`（或 `"0.0.0.0:8080:8080"`）会使 API 对连接到服务器的所有用户开放，其他人可能会控制你的机器人。
    如果你在安全环境（如家庭网络）中运行，这可能是安全的，但通常不建议将 API 公开到互联网。

## Rest API

### 调用 API

建议使用支持的 `freqtrade-client` 包（也可以使用 `scripts/rest_client.py` 脚本）调用 API。

无需运行频交易机器人，只需通过 `pip install freqtrade-client` 安装此包。

该模块设计轻量，仅依赖 `requests` 和 `python-rapidjson` 模块，避免加载频繁依赖的庞大库。

``` bash
freqtrade-client <命令> [可选参数]
```

默认情况下，脚本假定使用 `127.0.0.1`（localhost）和端口 `8080`，但可通过配置文件覆盖。

#### 最简客户端配置

``` json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!"
        //...
    }
}
```

``` bash
freqtrade-client --config rest_config.json <命令> [可选参数]
```

带有多个参数的命令可能需要使用关键词参数（为清晰起见）——如下示例：

``` bash
freqtrade-client --config rest_config.json forceenter BTC/USDT long enter_tag=GutFeeling
```

此方法适用于所有参数——可通过 `show` 命令查看所有支持参数。

??? Note "编程调用"
    `freqtrade-client` 包（可独立于频交易安装）可在你的脚本中使用，与频交易 API 交互。使用方法如下：

    ``` python
    from freqtrade_client import FtRestClient
    

    client = FtRestClient(server_url, username, password)

    # 获取机器人状态
    ping = client.ping()
    print(ping)
    # ... 
    ```

    有关全部支持的指令，请参考下方列表。

可以通过在 `rest-client` 脚本中使用 `help` 命令列出所有指令：

``` bash
freqtrade-client help
```

``` output
可能的命令：

available_pairs
    根据时间框架 / 资金币种选择，返回可用的交易对（回测数据）

balance
    获取账户余额。

blacklist
    显示当前黑名单。

cancel_open_order
    取消交易的未成交订单。

        :param trade_id: 取消此交易的未成交订单。

count
    返回未平仓交易的数量。

daily
    返回每日利润，以及交易次数。

delete_lock
    从数据库中删除（禁用）锁定。

        :param lock_id: 要删除的锁ID。

delete_trade
    从数据库中删除交易。
    尝试关闭未成交订单。需要在交易所手动处理该资产。

        :param trade_id: 要删除的交易ID。

edge
    返回关于 edge 的信息。

forcebuy
    立即购买某资产。

        :param pair: 交易对（如 ETH/BTC）
        :param price: 可选 - 购买价格

forceenter
    强制进入交易

        :param pair: 交易对（如 ETH/BTC）
        :param side: 'long' 或 'short'
        :param price: 可选 - 购买价格

forceexit
    强制退出交易。

        :param tradeid: 交易ID（可以通过状态命令获取）
        :param ordertype: 订单类型（market 或 limit）
        :param amount: 卖出数量。未提供则全部卖出。

health
    提供机器人运行的简要健康状态。

lock_add
    手动锁定特定交易对

        :param pair: 交易对
        :param until: 锁定至此日期（格式 "2024-03-30 16:00:00Z"）
        :param side: 锁定方向（long，short，*）
        :param reason: 锁定原因        

locks
    返回当前的锁定状态

logs
    显示最新日志。

        :param limit: 限制日志条数，为最后的 <limit> 条，无限制则显示全部。

pair_candles
    返回特定交易对 / 时间框架的实时数据。

        :param pair: 交易对
        :param timeframe: 只支持此时间框架。
        :param limit: 限制返回最后n根蜡烛。

pair_history
    返回指定时间范围内的历史分析数据。

        :param pair: 交易对
        :param timeframe: 只支持此时间框架。
        :param strategy: 用于分析的策略
        :param timerange: 时间范围（与 --timerange 参数格式相同）

performance
    返回不同币种的绩效。

ping
    简单的测试命令。

plot_config
    如果策略定义了绘图配置，返回该配置。

profit
    返回利润概览。

reload_config
    重新加载配置。

show_config
    仅显示与交易操作相关的配置部分。

start
    启动机器人（若已停止）。

pause
    暂停机器人（若在运行状态）；若在停止状态，则处理未平仓位。

stats
    返回统计报告（持续时间、退出原因等）。

status
    获取未平仓交易状态。

stop
    停止机器人。重启请用 `start`。

stopbuy
    停止新建交易（会优雅处理已存在的卖出）。用 `reload_config` 重置。

strategies
    列出可用的策略。

strategy
    获取策略详情。

        :param strategy: 策略类名。

sysinfo
    提供系统信息（CPU、内存使用情况）。

trade
    返回特定交易详情。

        :param trade_id: 指定交易ID。

trades
    返回交易历史，按ID排序。

        :param limit: 限制显示最近的交易数（最多500）。
        :param offset: 交易偏移量。

list_open_trades_custom_data
    返回包含开启交易的自定义数据的字典。

        :param key: str，可选——自定义数据的键。
        :param limit: 限制交易数。
        :param offset: 偏移量。

list_custom_data
    返回特定交易的自定义数据字典。

        :param trade_id: 交易ID
        :param key: str，可选——自定义数据的键。

version
    返回机器人版本。

whitelist
    显示当前白名单。

```

### 可用端点

如果希望通过其他方式手动调用 REST API，例如直接使用 `curl`，下表显示相关的 URL 端点及参数。  
所有端点都应以 API 的基础 URL 为前缀，例如 `http://127.0.0.1:8080/api/v1/`，这样完整命令为 `http://127.0.0.1:8080/api/v1/<command>`。

| 端点                    | 方法  | 描述 / 参数                               |
|------------------------|-------|-----------------------------------------|
| `/ping`               | GET   | 简单测试 API 是否就绪——无需认证。                 |
| `/start`              | POST  | 启动交易机器人。                              |
| `/pause`              | POST  | 暂停交易机器人。根据规则优雅处理未平仓交易。不开新仓。  |
| `/stop`               | POST  | 停止交易机器人。                              |
| `/stopbuy`            | POST  | 停止发布新订单。根据规则优雅关闭未平仓交易。          |
| `/reload_config`      | POST  | 重新加载配置文件。                              |
| `/trades`             | GET   | 获取最新交易列表。每次限制最多500条。                  |
| `/trade/<tradeid>`     | GET   | 获取指定交易。<br/>*参数:*<br/>- `tradeid`（`int`）  |
| `/trades/<tradeid>`    | DELETE| 从数据库中删除交易。尝试关闭未成交订单。需要在交易所手动处理该交易。<br/>*参数:*<br/>- `tradeid`（`int`） |
| `/trades/<tradeid>/open-order` | DELETE| 取消该交易的未成交订单。<br/>*参数:*<br/>- `tradeid`（`int`） |
| `/trades/<tradeid>/reload` | POST | 从交易所重新加载交易信息，仅在实盘中有效，可能帮助恢复手动卖出交易。<br/>*参数:*<br/>- `tradeid`（`int`） |
| `/show_config`        | GET   | 展示当前配置的部分内容，含操作关键设置。                     |
| `/logs`               | GET   | 获取最新日志。                                |
| `/status`             | GET   | 列出所有未平仓交易。                            |
| `/count`              | GET   | 查看已用和剩余的交易数量。                         |
| `/entries`            | GET   | 显示某交易对（或全部）不同进入标签的利润统计。<br/>*参数:*<br/>- `pair`（`str`） |
| `/exits`              | GET   | 显示某交易对（或全部）不同退出原因的利润统计。<br/>*参数:*<br/>- `pair`（`str`） |
| `/mix_tags`           | GET   | 显示某交易对（或全部）不同的进入标签与退出原因组合的利润统计。<br/>*参数:*<br/>- `pair`（`str`） |
| `/locks`              | GET   | 显示当前被锁定的交易对。                     |
| `/locks`              | POST  | 锁定某交易对直到“until”时间（会向上取整到最近的时间框架），方向可选（long、short，默认为long），原因可选。<br/>*参数:*<br/>- `<pair>`（`str`）<br/>- `<until>`（`datetime`）<br/>- `[side]`（`str`）<br/>- `[reason]`（`str`） |
| `/locks/<lockid>`     | DELETE| 删除（禁用）某个锁定，ID由参数提供。<br/>*参数:*<br/>- `lockid`（`int`） |
| `/profit`             | GET   | 展示你的盈亏总结及相关绩效统计。                   |
| `/forceexit`          | POST  | 立即退出指定交易（忽略 `minimum_roi`），使用所提供的订单类型（"market" 或 "limit"），未提供则为全部卖出，`tradeid` 可为 `all` 表示全部交易。<br/>*参数:*<br/>- `<tradeid>`（`int` 或 `str`）<br/>- `<ordertype>`（`str`）<br/>- `[amount]`（`float`） |
| `/forceenter`         | POST  | 立即进入某交易对。方向可选（long或 short，默认为long），价格可选（需要 `force_entry_enable` 设置为 True）。<br/>*参数:*<br/>- `<pair>`（`str`）<br/>- `<side>`（`str`）<br/>- `[rate]`（`float`） |
| `/performance`        | GET   | 按交易对显示已完成交易的绩效。                     |
| `/balance`            | GET   | 按币种显示账户余额。                            |
| `/daily`              | GET   | 显示过去 n 天的每日盈亏（默认为7天）。<br/>*参数:*<br/>- `<n>`（`int`） |
| `/weekly`             | GET   | 显示过去 n 天的每周盈亏（默认为4周）。<br/>*参数:*<br/>- `<n>`（`int`） |
| `/monthly`            | GET   | 显示过去 n 天的每月盈亏（默认为3个月）。<br/>*参数:*<br/>- `<n>`（`int`） |
| `/stats`              | GET   | 展示盈亏原因统计以及平均持仓时间。                   |
| `/whitelist`          | GET   | 查看当前白名单。                                |
| `/blacklist`          | GET   | 查看当前黑名单。                                |
| `/blacklist`          | POST  | 将指定交易对加入黑名单。<br/>*参数:*<br/>- `pair`（`str`） |
| `/blacklist`          | DELETE| 删除黑名单中的指定交易对。<br/>*参数:*<br/>- `[pair,pair]`（`list[str]`） |
| `/edge`               | GET   | 若启用，显示经过 Edge 验证的交易对。                   |
| `/pair_candles`       | GET   | 在机器人运行时返回交易对 / 时间框架的DataFrame。**Alpha** |
| `/pair_candles`       | POST  | 提供列列表，返回交易对 / 时间框架的 DataFrame（过滤后）。**Alpha**<br/>*参数:*<br/>- `<column_list>`（`list[str]`） |
| `/pair_history`       | GET   | 返回指定时间范围的分析数据 DataFrame。**Alpha** |
| `/pair_history`       | POST  | 提供列列表，返回分析后的 DataFrame（过滤后）。**Alpha**<br/>*参数:*<br/>- `<column_list>`（`list[str]`） |
| `/plot_config`        | GET   | 获取策略的绘图配置（如果有的话）。**Alpha** |
| `/strategies`         | GET   | 列出策略目录中的所有策略。**Alpha** |
| `/strategy/<strategy>`| GET   | 按策略类名获取特定策略内容。**Alpha**<br/>*参数:*<br/>- `<strategy>`（`str`） |
| `/available_pairs`     | GET   | 列出所有可用的回测数据。**Alpha** |
| `/version`           | GET   | 显示版本信息。                                |
| `/sysinfo`           | GET   | 显示系统负载信息。                            |
| `/health`            | GET   | 展示机器人健康状态（上一次循环状态）。             |

!!! Warning "Alpha 状态"
    上述带有 *Alpha 状态* 标签的端点可能会在任何时间更改，且不另行通知。

### 消息 WebSocket

API 服务器还提供一个 WebSocket 端点，用于订阅来自频交易机器人（freqtrade Bot）的 RPC 消息。  
可用于接收机器人传输的实时数据，比如进入/退出交易的通知、白名单变更、指标状态等。

这也是设置 [Producer/Consumer 模式](producer-consumer.md) 的方式之一。

假设你的 REST API 设置为 `127.0.0.1` 端口 `8080`，则 WebSocket 端点地址为：`http://localhost:8080/api/v1/message/ws`。

连接 WebSocket 时，需在 URL 中带上 `ws_token` 作为查询参数。

要生成安全的 `ws_token`，可以运行以下代码：

``` python
>>> import secrets
>>> secrets.token_urlsafe(25)
'hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q'
```

然后在配置文件中的 `api_server` 部分将该 token 填入 `ws_token` 字段。例如：

``` json
"api_server": {
    "enabled": true,
    "listen_ip_address": "127.0.0.1",
    "listen_port": 8080,
    "verbosity": "error",
    "enable_openapi": false,
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": [],
    "username": "Freqtrader",
    "password": "SuperSecret1!",
    "ws_token": "hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q" // <-----
}
```

连接地址示例：  
`http://localhost:8080/api/v1/message/ws?token=hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q`

!!! Danger "示例 Token 重用"
    请勿使用上述示例 token，为确保安全，应生成全新的 token。

#### 使用 WebSocket

连接成功后，机器人会向订阅者推送 RPC 消息。若要订阅某些消息类型，需通过 WebSocket 发送如下格式的 JSON 请求。  
`data` 字段须为消息类型字符串列表。

``` json
{
  "type": "subscribe",
  "data": ["whitelist", "analyzed_df"] // 消息类型字符串列表
}
```

消息类型列表可参考 `freqtrade/enums/rpcmessagetype.py` 中的 `RPCMessageType` 枚举。

只要机器人中有对应类型的 RPC 消息，连接保持活跃时就会收到。  
示例请求如下：

``` json
{
  "type": "analyzed_df",
  "data": {
      "key": ["NEO/BTC", "5m", "spot"],
      "df": {}, // DataFrame
      "la": "2022-09-08 22:14:41.457786+00:00"
  }
}
```

#### 反向代理配置

使用 [Nginx](https://nginx.org/en/docs/) 时，需进行如下配置以支持 WebSocket（此配置不完整，缺少部分信息，仅供参考）：

请将 `<freqtrade_listen_ip>` 和端口替换为你的实际配置。

```
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    #...

    server {
        #...

        location / {
            proxy_http_version 1.1;
            proxy_pass http://<freqtrade_listen_ip>:8080;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;
        }
    }
}
```

关于安全配置，请查阅对应代理软件的文档。

- **Traefik**：Traefik 支持 WebSocket ，详情见[官方文档](https://doc.traefik.io/traefik/)
- **Caddy**：Caddy v2 原生支持 WebSocket，详见[官方文档](https://caddyserver.com/docs/v2-upgrade#proxy)

!!! Tip "SSL证书"
    你可以使用 certbot 等工具，通过上述反向代理配置为你的机器人 UI 设置 SSL 证书，实现加密访问。  
但建议不要在公共网络直接运行频交易的 REST API，最好通过 VPN 或 SSH 隧道保护。

### OpenAPI 接口

在 `api_server` 配置中将 `"enable_openapi": true` 即可启用内置的 OpenAPI（Swagger UI）界面。  
配置后，Swagger UI 会在 `/docs` 端点提供。默认访问地址为：`http://localhost:8080/docs`，具体取决于你的配置。

### 高级 API 使用及 JWT 令牌

!!! Note
    下面内容应在应用（频交易 REST API 客户端）中完成，且不建议在日常操作中频繁使用。

频交易的 REST API 还支持 JWT（JSON Web Token）认证。  
你可以用以下命令登录，获取 `access_token`，随后在请求中使用。

``` bash
> curl -X POST --user Freqtrader http://localhost:8080/api/v1/token/login
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiZWQ1ZWI3YjAtYjMwMy00YzAyLTg2N2MtNWViMjIxNWQ2YTMxIiwiZXhwIjoxNTkxNzExNjgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJ0eXBlIjoicmVmcmVzaCJ9.d1AT_jYICyTAjD0fiQAr52rkRqtxCjUGEMwlNuuzgNQ"}
```

使用示例：  
将 `access_token` 赋值后，在请求头中加入授权信息：

``` bash
> access_token="eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g"
# 使用 access_token 进行认证
> curl -X GET --header "Authorization: Bearer ${access_token}" http://localhost:8080/api/v1/count
```

由于 `access_token` 有较短的有效期（15分钟），应定期使用 `token/refresh` 来刷新获取新令牌：

``` bash
> curl -X POST --header "Authorization: Bearer ${refresh_token}" http://localhost:8080/api/v1/token/refresh
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk5NzQsIm5iZiI6MTU4OTExOTk3NCwianRpIjoiMDBjNTlhMWUtMjBmYS00ZTk0LTliZjAtNWQwNTg2MTdiZDIyIiwiZXhwIjoxNTg5MTIwODc0LCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.1seHlII3WprjjclY6DpRhen0rqdF4j6jbvxIhUFaSbs"}
```