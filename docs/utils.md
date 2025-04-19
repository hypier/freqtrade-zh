# 实用子命令

除了 Live-Trade 和 Dry-Run 运行模式之外，`backtesting`、`edge` 和 `hyperopt` 优化子命令，以及准备历史数据的 `download-data` 子命令之外，机器人还包含多个实用子命令。本节将对此进行介绍。

## 创建用户目录

创建存放你的文件的文件夹结构，用于频繁交易（freqtrade）。
同时会为你生成策略和 hyperopt 示例，帮助你快速入门。
可以多次使用—使用 `--reset` 将会重置示例策略和 hyperopt 文件到默认状态。

--8<-- "commands/create-userdir.md"

!!! 警告
    使用 `--reset` 可能会导致数据丢失，因为这将会覆盖所有示例文件，且不会再次确认。

```
├── backtest_results
├── data
├── hyperopt_results
├── hyperopts
│   ├── sample_hyperopt_loss.py
├── notebooks
│   └── strategy_analysis_example.ipynb
├── plot
└── strategies
    └── sample_strategy.py
```

## 创建新配置

生成新的配置文件，并会询问一些对配置选择至关重要的问题。

--8<-- "commands/new-config.md"

!!! 警告
    仅会询问必要的问题。Freqtrade 提供了更多的配置选项，详细内容请参阅 [配置文档](configuration.md#configuration-parameters)。

### 创建配置示例

```
$ freqtrade new-config --config user_data/config_binance.json

? 是否启用 Dry-run（模拟交易）？  是
? 请输入你的权益币：BTC
? 请输入你的权益金额：0.05
? 请输入最大未平仓交易数（整数或 -1 表示无限制）：3
? 请输入你希望的时间周期（如 5m）：5m
? 请输入用来显示的货币（用于报告）：USD
? 选择交易所  binance
? 是否启用 Telegram？  否
```

## 查看配置

显示配置文件（默认为敏感信息会被隐藏）。
对于[拆分配置文件](configuration.md#multiple-configuration-files)或[环境变量](configuration.md#environment-variables)特别有用，此命令会显示合并后的配置结果。

![显示配置输出](assets/show-config-output.png)

--8<-- "commands/show-config.md"

``` output
您的合并配置如下：
{
  "exit_pricing": {
    "price_side": "other",
    "use_order_book": true,
    "order_book_top": 1
  },
  "stake_currency": "USDT",
  "exchange": {
    "name": "binance",
    "key": "REDACTED",
    "secret": "REDACTED",
    "ccxt_config": {},
    "ccxt_async_config": {},
  }
  // ...
}
```

!!! 警告 "分享此命令返回的信息"
    我们会尽量在默认输出中去除所有已知的敏感信息（不使用 `--show-sensitive`）。
    但请务必再次检查输出，确保没有无意中泄露私人信息。

## 创建新策略

从模板（类似 SampleStrategy）创建新策略文件。
文件名将与类名保持一致，不会覆盖已有文件。

生成结果将存放在 `user_data/strategies/<策略类名>.py`。

--8<-- "commands/new-strategy.md"

### 使用新策略模板的示例

```bash
freqtrade new-strategy --strategy AwesomeStrategy
```

使用自定义用户目录

```bash
freqtrade new-strategy --userdir ~/.freqtrade/ --strategy AwesomeStrategy
```

使用高级模板（填充所有可选函数和方法）

```bash
freqtrade new-strategy --strategy AwesomeStrategy --template advanced
```

## 列出策略

使用 `list-strategies` 子命令可以列出某个特定目录中的所有策略。

此子命令非常适合检查加载策略是否出现问题：如果加载失败的模块会以红色显示（LOAD FAILED），重复命名的策略会显示为黄色（DUPLICATE NAME）。

--8<-- "commands/list-strategies.md"

!!! 警告
    使用这些命令会尝试加载某个目录下的所有 Python 文件。如果该目录中存在不可信文件，执行会存在安全风险，因为所有模块级代码都会被执行。

示例：搜索默认的策略目录（在默认用户目录内）：

``` bash
freqtrade list-strategies
```

示例：搜索用户目录中的策略目录：

``` bash
freqtrade list-strategies --userdir ~/.freqtrade/
```

示例：搜索专用的策略路径：

``` bash
freqtrade list-strategies --strategy-path ~/.freqtrade/strategies/
```

## 列出 Hyperopt 损失函数

使用 `list-hyperoptloss` 子命令可以查看所有可用的 hyperopt 损失函数。

此命令会快速列出你环境中所有支持的损失函数。

它对于发现加载损失函数问题非常有用：如出现包含错误且加载失败的模块（以红色显示，LOAD FAILED）、重名的损失函数（以黄色显示，DUPLICATE NAME）会被标记出来。

--8<-- "commands/list-hyperoptloss.md"

## 列出 freqAI 模型

使用 `list-freqaimodels` 子命令可以列出所有可用的 freqAI 模型。

此子命令有助于检测加载模型时的潜在问题：支持不良或加载失败的模型会以红色显示（LOAD FAILED），重复命名的模型会以黄色显示（DUPLICATE NAME）。

--8<-- "commands/list-freqaimodels.md"

## 列出交易所

使用 `list-exchanges` 子命令可以查看机器人支持的所有交易所。

--8<-- "commands/list-exchanges.md"

示例：查看机器人支持的交易所：

```
$ freqtrade list-exchanges
Freqtrade支持的交易所：
交易所名称       支持状态    市场                 备注
------------------  -----------  ----------------------  ------------------------------------------------------------------------
binance             官方      现货、隔离期货
bitmart             官方      现货
bybit                             现货、隔离期货
gate                官方      现货、隔离期货
htx                 官方      现货
huobi                             现货
kraken              官方      现货
okx                 官方      现货、隔离期货
```

!!! 说明 ""
    出于清晰起见，输出内容已缩减。支持的交易所和功能可能会随时间变化。

!!! 注意 "缺少可选交易所"
    带有 "missing opt:" 的值可能需要特殊配置（例如在 `fetchTickers` 缺失的情况下使用 orderbook），但理论上是可以的（虽然不能保证一定成功）。

示例：查看 ccxt 库支持的所有交易所（包括已知不支持的交易所）：

```
$ freqtrade list-exchanges -a
ccxt 库支持的所有交易所：
交易所名称       有效支持   支持状态    市场                    备注
------------------  -------  -----------  ----------------------  ---------------------------------------------------------------------------------
binance             True     官方      现货、隔离期货
bitflyer            False                 现货                    缺少：fetchOrder。缺少可选：fetchTickers。
bitmart             True     官方      现货
bybit               True                  现货、隔离期货
gate                True     官方      现货、隔离期货
htx                 True     官方      现货
kraken              True     官方      现货
okx                 True     官方      现货、隔离期货
```

!!! 说明 ""
    输出已缩减。支持和可用的交易所可能会随时间变化。

## 列出时间周期

使用 `list-timeframes` 子命令可以查看交易所支持的时间周期列表。

--8<-- "commands/list-timeframes.md"

示例：查看配置文件中 `binance` 交易所支持的时间周期：

```
$ freqtrade list-timeframes -c config_binance.json
...
交易所 `binance` 支持的时间周期：1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d, 3d, 1w, 1M
```

示例：列出 Freqtrade 支持的所有交易所及其支持的时间周期：

```
$ for i in `freqtrade list-exchanges -1`; do freqtrade list-timeframes --exchange $i; done
```

## 列出交易对／市场

`list-pairs` 和 `list-markets` 子命令可以用来查看交易所中的交易对／市场。

交易对是在市场符号中，基础货币和报价货币之间用 `/` 连接，例如 `ETH/BTC` 中，`ETH` 是基础货币，`BTC` 是报价货币。

在支持的交易所中，Freqtrade 使用配置中的 `stake_currency` 来定义交易的报价货币。

可以用这些子命令打印任何交易对／市场的信息，也可以通过 `--quote BTC` 过滤报价货币，或通过 `--base ETH` 过滤基础货币。

这些子命令的用法和选项完全相同：

--8<-- "commands/list-pairs.md"

默认只显示活跃的交易对／市场。活跃交易对是在当前交易所可以交易的那些。
你可以加上 `-a`/`-all` 选项，显示所有交易对／市场（包括非活跃的）。
交易对可能显示为不可交易，如果市场的最小可交易价格非常小（小于 `1e-11` 或 `0.00000000001`）。

交易对／市场按符号字符串排序输出。

### 示例

* 输出在默认配置文件中指定的交易所（如 Binance）上的活跃对，报价货币为 USD，格式为 JSON：

```
$ freqtrade list-pairs --quote USD --print-json
```

* 输出在配置文件 `config_binance.json` 中的所有交易对（在 Binance 交易所），基础货币为 BTC 或 ETH，报价货币为 USDT 或 USD，以人类可读的列表和摘要形式显示：

```
$ freqtrade list-pairs -c config_binance.json --all --base BTC ETH --quote USDT USD --print-list
```

* 输出交易所 "Kraken" 上的所有市场，采用表格显示：

```
$ freqtrade list-markets --exchange kraken --all
```

## 测试交易对列表

使用 `test-pairlist` 子命令测试 [动态交易对列表](plugins.md#pairlists) 的配置。

要求配置中包含 `pairlists` 属性。
可以用来生成静态交易对列表，用于回测／hyperopt。

--8<-- "commands/test-pairlist.md"

### 示例

显示启用了 [动态交易对列表](plugins.md#pairlists) 时的白名单：

```
freqtrade test-pairlist --config config.json --quote USDT BTC
```

## 转换数据库

`freqtrade convert-db` 可以将你的数据库从一种系统转换到另一种系统（sqlite -> postgres，postgres -> 其他 postgres），迁移所有的交易、订单以及 Pairlocks。

请参考 [相关文档](advanced-setup.md#use-a-different-database-system)，了解不同数据库系统的要求。

--8<-- "commands/convert-db.md"

!!! 警告
    仅应在目标数据库为空时使用此命令。Freqtrade 会进行常规迁移，但如果目标已有数据，可能会失败。

## Web服务器模式

!!! 警告 "实验性"
    Web服务器模式是一个实验性功能，用于提升回测与策略开发的效率。
    可能仍存在 bug — 如遇问题，请在 GitHub 提交 Issue，谢谢。

运行 freqtrade 进入 Web 服务器模式。
Freqtrade 会启动 Web 服务器，允许 FreqUI 启动并控制回测流程。
此模式的优点是，在相同的时间周期和时间范围下，数据不会重复加载。
FreqUI 还会显示回测结果。

--8<-- "commands/webserver.md"

### Web服务器模式 - Docker

也可以通过 Docker 使用 Web 服务器模式。
启动一次性容器时需要明确配置端口，因为端口默认未映射。
可以使用 `docker compose run --rm -p 127.0.0.1:8080:8080 freqtrade webserver` 来启动一个临时容器，停止后会自动删除。前提是端口 8080 还未被占用，且没有其他机器人在使用。

或者，你可以修改 `docker-compose` 文件，更新启动命令为：

``` yml
    command: >
      webserver
      --config /freqtrade/user_data/config.json
```

之后可以使用 `docker compose up` 启动 Web 服务器。
此配置假设已启用 Web 服务器，并将端口配置为 `0.0.0.0`。

!!! 提示
    开始使用前别忘了将命令恢复为交易命令，以便启动实盘或 Dry-Run 机器人。

## 查看之前的回测结果

可以查看之前的回测结果。
添加 `--show-pair-list` 会输出一个排序后的交易对列表，方便复制粘贴到配置中（过滤掉表现差的交易对）。

??? 警告 "策略过拟合"
    只使用获胜的交易对可能导致策略过拟合，未来可能无法表现良好。强烈建议在实盘前进行充分的 Dry-Run 测试。

--8<-- "commands/backtesting-show.md"

## 详细回测分析

高级的回测结果分析。

更多细节请参阅 [回测分析](advanced-backtesting.md#analyze-the-buyentry-and-sellexit-tags) 部分。

--8<-- "commands/backtesting-analysis.md"

## 列出 Hyperopt 结果

可以用 `hyperopt-list` 子命令列出 Hyperopt 模块之前评估的所有超参数优化轮次。

--8<-- "commands/hyperopt-list.md"

!!! 备注
    `hyperopt-list` 会自动使用最新的 hyperopt 结果文件。
    你也可以通过 `--hyperopt-filename` 参数指定其他文件（无需路径）。

### 示例

列出所有结果，最后显示最佳结果的详细信息：
```
freqtrade hyperopt-list
```

只列出盈利的轮次，不显示详细信息，以便在脚本中循环处理：
```
freqtrade hyperopt-list --profitable --no-details
```

## 查看 Hyperopt 结果详情

可以用 `hyperopt-show` 子命令查看之前由 Hyperopt 模块评估过的某个超参数优化轮次的详细信息。

--8<-- "commands/hyperopt-show.md"

!!! 备注
    `hyperopt-show` 会自动使用最新的 hyperopt 结果文件。
    也可以用 `--hyperopt-filename` 指定其他文件（无需路径）。

### 示例

打印第168轮的详细信息（轮次编号由 `hyperopt-list` 或 Hyperopt 在运行时显示）：

```
freqtrade hyperopt-show -n 168
```

打印包括所有轮次中最佳轮次的 JSON 详细信息：

```
freqtrade hyperopt-show --best -n -1 --print-json --no-header
```

## 查看交易

输出数据库中的（全部或指定的）交易信息到终端。

--8<-- "commands/show-trades.md"

### 示例

输出交易ID为2和3的交易，格式为 JSON：

``` bash
freqtrade show-trades --db-url sqlite:///tradesv3.sqlite --trade-ids 2 3 --print-json
```

## 策略更新工具（Strategy-Updater）

更新列出的策略或策略文件夹中的所有策略，使其符合 v3 标准。
如果不指定 `--strategy-list`，则会自动转换策略文件夹中的所有策略。
原始策略文件会被保留在 `user_data/strategies_orig_updater/` 目录中。

!!! 警告 "转换结果"
    策略更新器采用“尽最大努力”的方式工作。请务必自行核查转换结果。
    推荐运行一次 Python 格式化工具（如 `black`）以确保格式规范。