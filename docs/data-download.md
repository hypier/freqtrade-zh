# 数据下载

## 获取回测和超参数优化所需的数据

要下载用于回测和超优化的蜡烛图（OHLCV）数据，请使用 `freqtrade download-data` 命令。

如果未指定其他参数，freqtrade 将默认下载过去30天的 `"1m"` 和 `"5m"` 时间框架的数据。  
交易所和交易对信息将从 `config.json` 中读取（如果在参数中使用 `-c/--config` 指定）。  
未提供配置文件时，`--exchange` 参数将变得必填。

你可以使用相对时间范围（如 `--days 20`）或绝对起始日期（如 `--timerange 20200101-`）来指定数据时间段。  
对于增量下载，建议使用相对时间方式。

!!! Tip "提示：更新现有数据"
    如果你的数据目录中已经有回测数据，并且希望将数据刷新至今天，freqtrade 会自动计算已有交易对的缺失时间段，然后从最新数据点开始下载，直到“现在”。此操作无需额外指定 `--days` 或 `--timerange` 参数。  
    该方式只会下载缺失部分的数据，保留已有数据。  
    如果在插入新交易对后需要更新（且没有对应数据），可以使用 `--new-pairs-days xx` 参数，指定新交易对的天数。如此会为新交易对下载完整天数，而旧交易对只会补充缺失数据。

### 使用示例

--8<-- "commands/download-data.md"

!!! Tip "下载某个行情货币的所有数据"
    常常你会希望下载某个行情货币所有交易对的数据。可以使用下面的简写：
    `freqtrade download-data --exchange binance --pairs ".*/USDT" <...>`。  
    这会自动扩展为该交易所所有以USDT结尾的活跃交易对。  
    若还想下载已退市（非活跃）交易对的数据，可以加上 `--include-inactive-pairs`。

!!! Note "启动期"
    `download-data` 是一个策略无关的命令。其思想是在一次性下载大量数据后，逐步增加存储的数据量。

    因此，`download-data` 不考虑策略中定义的“启动期（startup-period）”。如果想在特定时间点开始回测，应额外下载对应时间的天数（同时考虑启动期）。

### 开始下载

假设已有 `config.json` 文件，以下是简单示例。

```bash
freqtrade download-data --exchange binance
```

此命令将下载配置中定义的所有交易对的历史蜡烛图（OHLCV）数据。

也可以直接指定交易对：

```bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT
```

或者用正则表达式（此例为下载所有活跃的USDT交易对）：

```bash
freqtrade download-data --exchange binance --pairs ".*/USDT"
```

### 其他注意事项

* 若想使用非默认的数据目录（例如：`user_data/data/some_directory`），请用 `--datadir` 参数：
  ```bash
  --datadir user_data/data/some_directory
  ```
* 若要更改下载数据的交易所，可以用 `--exchange <交易所>` ，或指定不同的配置文件。
* 若使用存放交易对信息的 `pairs.json` 文件，可用 `--pairs-file some_other_dir/pairs.json` 指定路径。
* 若只想下载10天的数据（默认30天）：
  ```bash
  --days 10
  ```
* 若要从固定起点开始下载（比如2020年1月1日）：
  ```bash
  --timerange 20200101-
  ```
  注意：如果已有对应时间段的数据，将只下载缺失部分。
* 通过 `--timeframes` 参数还可以指定希望下载的时间框架，默认值为 `--timeframes 1m 5m`。
* 要使用配置文件中的交易所、时间框架和交易对列表，使用 `-c/--config` 选项。这样脚本会用配置中的白名单，不必提供 `pairs.json`。

??? Note "权限问题"
    如果你的配置目录 `user_data` 是由 Docker 创建的，可能会遇到如下权限错误：
    ```
    cp: cannot create regular file 'user_data/data/binance/pairs.json': Permission denied
    ```
    你可以通过下面的命令修正权限：
    ```
    sudo chown -R $UID:$GID user_data
    ```

### 下载后续时间段的数据（覆盖新增）

假设你之前已下载了2022年的数据（`--timerange 20220101-`），现在想提前下载更早的数据进行回测，可以用 `--prepend` 参数配合 `--timerange` 指定结束日期。

```bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT --prepend --timerange 20210101-20220101
```

!!! Note
    在此模式下，如果已有数据，Freqtrade 会忽略结束日期，自动将结束日期调整为已有数据的起始点。

### 数据格式

Freqtrade 目前支持以下数据格式：

* `feather` — 基于 Apache Arrow 的数据格式
* `json` — 普通文本 json 文件
* `jsongz` — gzip 压缩版的 json 文件
* `parquet` — 列式存储格式（仅OHLCV）

默认情况下，OHLCV数据和交易数据都以 `feather` 格式存储。

可以通过命令行参数 `--data-format-ohlcv` 和 `--data-format-trades` 分别改变存储格式。  
若想持久化此设置，可在配置中添加如下内容，以免每次都指定参数：

``` jsonc
    // ...
    "dataformat_ohlcv": "feather",
    "dataformat_trades": "feather",
    // ...
```

如果在下载过程中改变了默认的数据格式，也需在配置文件中相应调整 `dataformat_ohlcv` 和 `dataformat_trades` 键值。

!!! Note
    你可以使用 [convert-data](#sub-command-convert-data) 和 [convert-trade-data](#sub-command-convert-trade-data) 方法在不同格式间转换。

#### 数据格式对比

以下比较基于该数据和使用 Linux `time` 命令测得的结果。

```
找到6组交易对/时间框架组合。
+----------+-------------+--------+---------------------+---------------------+
|     交易对 |   时间框架 |   类型 |                起点 |                  终点 |
|----------+-------------+--------+---------------------+---------------------|
| BTC/USDT |          5m |   现货 | 2017-08-17 04:00:00 | 2022-09-13 19:25:00 |
| ETH/USDT |          1m |   现货 | 2017-08-17 04:00:00 | 2022-09-13 19:26:00 |
| BTC/USDT |          1m |   现货 | 2017-08-17 04:00:00 | 2022-09-13 19:30:00 |
| XRP/USDT |          5m |   现货 | 2018-05-04 08:10:00 | 2022-09-13 19:15:00 |
| XRP/USDT |          1m |   现货 | 2018-05-04 08:11:00 | 2022-09-13 19:22:00 |
| ETH/USDT |          5m |   现货 | 2017-08-17 04:00:00 | 2022-09-13 19:20:00 |
+----------+-------------+--------+---------------------+---------------------+
```

测时采用了强制读入内存的命令，以确保比较一致性。

```bash
time freqtrade list-data --show-timerange --data-format-ohlcv <dataformat>
```

| 格式 | 文件大小 | 耗时 |
|--------|----------|-------|
| `feather` | 72MB | 3.5秒 |
| `json` | 149MB | 25.6秒 |
| `jsongz` | 39MB | 27秒 |
| `parquet` | 83MB | 3.8秒 |

大小数据来源为指定时间范围内BTC/USDT 1分钟现货交易对。  
建议使用默认的 `feather` 或 `parquet` 格式，以获得较优的性能和存储平衡。

### 交易对文件（Pairs File）

除了 `config.json` 中的白名单外，还可以使用 `pairs.json` 文件。

以 Binance 为例：

* 创建目录 `user_data/data/binance`，并将 `pairs.json` 文件放入其中。
* 修改 `pairs.json`，列出你感兴趣的交易对。

示例操作：

```bash
mkdir -p user_data/data/binance
touch user_data/data/binance/pairs.json
```

`pairs.json` 文件格式为一个简单的 json 数组。可以混合不同基础货币，但仅用于下载目的。

```json
[
    "ETH/BTC",
    "ETH/USDT",
    "BTC/USDT",
    "XRP/ETH"
]
```

!!! Note
    `pairs.json` 文件仅在未加载配置（通过命名或 `--config`）的情况下使用。  
    如果希望强制使用此文件，可加参数 `--pairs-file pairs.json`。  
    推荐做法仍是使用配置中的交易对列表（通过 `exchange.pair_whitelist` 或配置中的 `pairs` 设置）。

## 子命令：convert-data

--8<-- "commands/convert-data.md"

### 转换数据示例

以下命令将 `~/.freqtrade/data/binance` 目录下所有的蜡烛图（OHLCV）数据由 json 转换为 jsongz 格式，省空间。同时会删除原始 json 文件（`--erase` 参数）。

```bash
freqtrade convert-data --format-from json --format-to jsongz --datadir ~/.freqtrade/data/binance -t 5m 15m --erase
```

## 子命令：convert-trade-data

--8<-- "commands/convert-trade-data.md"

### 转换交易数据示例

以下命令将 `~/.freqtrade/data/kraken` 目录中所有可用交易数据由 jsongz 转为 json，并删除原有的 jsongz 文件（`--erase`）：

```bash
freqtrade convert-trade-data --format-from jsongz --format-to json --datadir ~/.freqtrade/data/kraken --erase
```

## 子命令：trades to ohlcv

当需要使用 `--dl-trades`（仅 Kraken 支持）来下载数据时，转换交易数据为 OHLCV 格式是最后一步。此命令可以多次为不同时间框架重复此步骤，无需重新下载数据。

--8<-- "commands/trades-to-ohlcv.md"

### 交易到 OHLCV 转换示例

```bash
freqtrade trades-to-ohlcv --exchange kraken -t 5m 1h 1d --pairs BTC/EUR ETH/EUR
```

## 子命令：list-data

可以用 `list-data` 子命令列出已下载的数据。

--8<-- "commands/list-data.md"

### 示例：列出所有数据

```bash
> freqtrade list-data --userdir ~/.freqtrade/user_data/

              发现33组交易对/时间框架组合。
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━┓
┃          交易对 ┃                                 时间框架 ┃ 类型 ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━┩
│       ADA/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ 现货 │
│       ADA/ETH │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ 现货 │
│       ETH/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ 现货 │
│      ETH/USDT │                  5m, 15m, 30m, 1h, 2h, 4h │ 现货 │
└───────────────┴───────────────────────────────────────────┴──────┘
```

显示所有交易数据，包括起止时间范围：

```bash
> freqtrade list-data --show --trades
                     发现1个交易对的交易数据。                     
┏━━━━━━━━━┳━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━┓
┃    交易对 ┃ 类型 ┃                起点 ┃                  终点 ┃ 交易次数 ┃
┡━━━━━━━━━╇━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━┤
│ XRP/ETH │ 现货 │ 2019-10-11 00:00:11 │ 2019-10-13 11:19:28 │  12477 │
└─────────┴──────┴─────────────────────┴─────────────────────┴────────┘
```

## 交易（tick）数据

默认情况下，`download-data` 子命令下载蜡烛图（OHLCV）数据。  
大部分交易所也通过API提供历史交易数据。  

此类数据在需要多时间框架时尤其有用，因为只需一次下载，后续本地采样即可。

由于此数据体积较大，默认采用 feather 格式存储，文件命名为 `<pair>-trades.feather`（如：`ETH_BTC-trades.feather`）。  
支持增量模式，类似历史OHLCV数据， weekly `--days 8` 下载一次即可建立增量存储库。

启用此模式只需添加 `--dl-trades` 参数。此操作会切换到下载交易数据。若同时使用 `--convert`，将自动进行重采样，并覆盖已有的OHLCV数据。

!!! Warning "请勿使用"
    除非你是 Kraken 用户（Kraken 不提供历史OHLCV数据），否则不建议使用。  
    许多其他交易所以充足的历史提供OHLCV数据，通过该方式下载多时间框架的速度会快很多。

!!! Note "Kraken用户"
    Kraken用户在下载前建议阅读 [这个](exchanges.md#historic-kraken-data)。

示例调用：

```bash
freqtrade download-data --exchange kraken --pairs XRP/EUR ETH/EUR --days 20 --dl-trades
```

!!! Note
    虽然此方法采用异步调用，但速度会较慢，因为每次都需等待前次请求完成后生成下一次请求。

## 下一步

恭喜，你已经下载了一些数据，可以开始 [回测](backtesting.md) 你的策略了。