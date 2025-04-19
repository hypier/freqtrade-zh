# Freqtrade 常见问题解答

## 支持的市场

Freqtrade 支持现货交易，以及部分精选交易所的（隔离）期货交易。请参考 [文档首页](index.md#supported-futures-exchanges-experimental) 以获得最新的支持交易所列表。

### 我的机器人可以开空仓吗？

Freqtrade 支持在期货市场开空仓。  
这需要策略专门为此设计 —— 且配置中设置 `"trading_mode": "futures"`。  
请务必先阅读 [相关文档页面](leverage.md)。

在现货市场，有时也可以使用带杠杆的现货代币（如 BTCUP/USD、BTCDOWN/USD、ETHBULL/USD、ETHBEAR/USD 等），这些反映反向交易对，也可以通过 Freqtrade 进行交易。

### 我的机器人可以交易期权或期货吗？

部分交易所支持期货交易。请参考 [文档首页](index.md#supported-futures-exchanges-experimental) 以获得最新支持交易所名单。

## 初学者小贴士

* 当你使用策略和 hyperopt 文件时，建议使用像 VSCode 或 PyCharm 这样的专业代码编辑器。好的编辑器会提供语法高亮和行号，方便你快速找到语法错误（这些错误通常会在启动时被 Freqtrade 指出）。

## 常见问题解答

### Freqtrade 会同时对同一交易对开多个仓位吗？

不。Freqtrade 一次只会在每个交易对上开一个仓位。  
不过，你可以使用 [`adjust_trade_position()` 回调](strategy-callbacks.md#adjust-trade-position) 来调整已持有仓位。

回测时提供了 `--eps` 选项以支持此功能，但此仅用于突出“隐藏”信号，并不适用于实时交易。

### 机器人无法启动

运行命令 `freqtrade trade --config config.json` 后出现 `freqtrade: command not found` 提示。

可能原因包括：

* 虚拟环境未激活。  
  * 运行 `source .venv/bin/activate` 来激活虚拟环境。
* 安装未成功完成。  
  * 请查看 [安装指南](installation.md)。

### 机器人启动，但处于 STOPPED（停止）状态

请确保在 config.json 中设置 `initial_state` 配置项为 `"running"`。

### 为什么等待了5分钟，机器人还没有进行任何交易？

* 根据入场策略、白名单币种数量、市场行情等因素，可能需要几小时甚至几天才能找到合适的入场点。请耐心等待！

* 回测结果大致可以告诉你预期的交易次数，但不能保证交易会平均分布在每一天——比如一天可能会有20笔交易，其他天则没有。

* 可能存在配置错误。建议查看日志，它们通常会显示机器人是否没有收到买入信号（只显示心跳信息），或者是否存在其他问题（错误/异常信息）。

### 我已经完成了12笔交易，为什么总利润还是负的？

理解你的失望，但仅仅12笔交易不足以作出结论。  
回测结果显示你的策略总体仍在盈利，但这是在数千笔交易之后的情况。即使如此，有些币种你可能也会亏损，因为你曾在这些币上频繁交易（十几次甚至上百次）。我们不断努力优化机器人，但交易仍包含一定的风险。  
你应当以适度的利润为目标，不能单凭少量交易得出结论。

### 我想修改配置，可以不用停止机器人吗？

可以。你可以编辑配置文件，然后使用 `/reload_config` 命令让机器人重新加载配置。  
机器人会停止、加载新配置与策略，然后重启，应用新的设置。

### 为什么机器人没有卖出它买入的所有币？

这叫“零碎币（coin dust）”，在所有交易所都可能发生。  
这是因为许多交易所会在“接收币”环节扣除手续费，所以你可能买入100个币，但实际只收到99.9个币。  
由于币是以整手（1个币为单位）交易的，你不能卖出0.9个币（即99.9个币），只能卖出99个。  
这不是机器人问题，而是在手动交易中也会出现。

Freqtrade 可以处理这种情况（会卖出99个币），但手续费通常低于最小交易单位（只能交易整币）。  
留下“零碎币”通常也是合理的，下次Freqtrade买入币时，会逐步消耗掉剩余的少量余额，最终卖出所有已购币，慢慢减少“零碎币”。当然，它可能永远无法达到完全为0的状态。

在某些交易所（如Binance），使用平台专用的手续费币（如BNB）可以解决这个问题。只需在账户中持有BNB，并在设置中启用“用BNB支付手续费”。你的BNB余额会随着手续费的使用逐渐减少（手续费计入利润），无需担心“零碎币”。

其他交易所则没有此功能，你只能接受或选择切换至支持此方案的交易所。

### 我向交易所充值了更多资金，机器人没有识别到

Freqtrade 会在需要时（下单前）更新交易所余额。  
通过 RPC 调用（Telegram `/balance`，API `/balance`）最多每小时刷新一次余额。

若启用 `adjust_trade_position`（且存在符合条件的未平仓交易）——钱包余额也会每小时刷新一次。  
想立即刷新余额，则可以使用 `/reload_config`，这会重启机器人并更新余额。

### 我想使用未完整的K线数据

Freqtrade 不会提供未完整的K线数据给策略。使用不完整的K线会导致“重绘”问题，从而出现“鬼买”现象，这既无法进行有效回测，也无法验证未来的交易。

你可以通过 [dataprovider](strategy-customization.md#orderbookpair-maximum) 的 orderbook 或 ticker 方法获取“当前”市场数据，但这在回测时不可用。

### 有没有设置可以只退出持有的仓位，而不进行新的入场操作？

可以使用 Telegram 的 `/stopentry` 命令阻止未来的入场交易，之后再用 `/forceexit all` 卖出所有持仓。

### 我想在同一台机器上运行多个机器人

请参考 [高级设置指南](advanced-setup.md#running-multiple-instances-of-freqtrade)。

### 启动机器人时显示“无法加载策略”怎么办？

此错误提示机器人未能加载策略。  
你可以运行 `freqtrade list-strategies` 查看所有可用策略。  
该命令输出中会有状态列，显示策略是否可以成功加载。

请确认：

* 你使用的策略名称是否正确？策略名区分大小写，必须与策略类名一致（不是文件名！）。
* 策略文件是否放在 `user_data/strategies` 目录下，并以 `.py` 结尾？
* 运行前是否有其他警告？可能缺少某些依赖，日志会指出。
* 若使用 Docker，策略目录是否正确挂载（检查 `docker-compose` 文件中的挂载卷部分）？

### 日志中出现“Missing data fillup”消息怎么办？

这是个警告，表示最新的K线中有缺失的蜡烛。  
不同交易所表现不同，可能意味着所用时间段内没有交易，交易所只返回带成交量的K线（交易量为0的K线会被省略）。低成交量的币种这是常见情况。

如果所有币种都出现此问题，可能是交易所暂时宕机或其他问题。建议查看交易所的公告或状态页面。

不管原因如何，Freqtrade会用“空白”蜡烛填充缺失的部分，开盘价、最高价、最低价和收盘价都和前一根蜡烛一致，成交量为空。这在图表上就像一个 `_` 的符号，符合交易所通常表现0成交量蜡烛的方式。

### 警告：“Price jump between 2 candles detected”

警告表示两根蜡烛之间的价格跳跃超过30%。  
可能意味着交易对停止交易，或者发生了某种异常价格变化（例如2021年的 COCOS，价格从0.0000154突升到0.01621）。  
此类警示通常伴随“Missing data fillup”——说明交易暂停了一段时间。

### 如何重置机器人数据库？

可以删除数据库（默认为 `tradesv3.sqlite` 或 `tradesv3.dryrun.sqlite`），或通过 `--db-url` 指定新数据库（如 `sqlite:///mynewdatabase.sqlite`）。

### 日志中出现“Outdated history for pair xxx”怎么办？

这是提醒你获取到了不最新的蜡烛数据（非完整的最新蜡烛）。  
因此，Freqtrade不会对该交易对进行交易，因为用旧信息交易风险较大。

可能原因包括：

* 交易所宕机——请检查交易所状态页面或公告。
* 系统时间不正确——确保你的系统时间正确。
* 交易对少量交易——在交易所网页上检查该交易对的成交量。若某些蜡烛无成交（显示为“成交量0”或用“_”标记），说明某段时间内没有交易。建议避免在此类交易对上操作，以免订单无法成交。
* API 出错——API 返回数据错误（这纯粹作为补充，支持的交易所通常不会发生）。

### 出现“Exchange XXX does not support market orders.” 提示，不能运行策略怎么办？

提示说明你的交易所不支持市价单，而策略中定义的订单类型包含“market”。  
你的策略可能是为其他支持市价单的交易所设计的，且在设置中默认为“market”订单（如用于止损单），这是大部分支持市价单的交易所（比如 Binance）是正确的，但在不支持的交易所（如 Gate.io）会出错。

解决办法：  
在策略中将订单类型改为“limit”：

```python
    order_types = {
        ...
        "stoploss": "limit",
        ...
    }
```

如果你的自定义配置中定义了订单类型，也需要同步修改。

### 我想以实时方式运行，但遇到API权限错误

错误如 `Invalid API-key, IP, or permissions for action` 表示API密钥无效、过期或被限制。  
核查：  
* API 密钥是否正确（注意复制粘贴时是否有空格）  
* 密钥是否已过期  
* 机器人运行所在 IP 是否已在交易所API设置中启用权限（通常需要启用“现货交易”权限，期货可能也要特别启用）  

### 如何搜索机器人日志中的内容？

默认情况下，机器人将日志写入标准错误（stderr）。这是为了方便你用 grep 等工具筛选。  
你可以通过将 stderr 重定向到 stdout，并进行过滤。

* 在Unix shell中，常用以下格式：
```shell
$ freqtrade --some-options 2>&1 >/dev/null | grep 'something'
```
（注意 `2>&1` 和 `>/dev/null` 的顺序）

* Bash 支持进程替代语法，你也可以用：
```shell
$ freqtrade --some-options 2> >(grep 'something') >/dev/null
```
或
```shell
$ freqtrade --some-options 2> >(grep -v 'something' 1>&2)
```

* 你还可以将日志输出到文件，然后用 grep 搜索：
```shell
$ freqtrade --logfile /path/to/mylogfile.log --some-options
```
再用：
```shell
$ cat /path/to/mylogfile.log | grep 'something'
```
或者实时监控并过滤：
```shell
$ tail -f /path/to/mylogfile.log | grep 'something'
```
（在另一个终端窗口中操作）

Windows 系统中，`--logfile` 也支持，你可以用 `findstr` 来搜索：
```
> type \path\to\mylogfile.log | findstr "something"
```

## Hyperopt 模块

### 为什么 freqtrade 不支持GPU？

首要原因是大部分指标库都不支持GPU，因此指标计算没有明显加速效果。  
GPU的优化主要适用于pandas原生计算，或者你自己实现的部分。

在 hyperopt 方面，Freqtrade 使用的是 scikit-optimize，这是建立在 scikit-learn 之上的。  
它们关于GPU支持的声明非常明确：[FAQ](https://scikit-learn.org/stable/faq.html#will-you-add-gpu-support)。

GPU主要擅长浮点运算（如数字计算），而 hyperopt 既要进行参数搜索（找下一个参数组合），也要进行回测执行（运行Python代码）。  
因此，GPU对大部分 hyperopt 任务的提升有限，且引入的复杂性不值得。

当然，你可以在自己的策略中使用GPU加速的指标（如果你确定需要），但大概率付出的大部分努力得不到显著回报（相较于复杂性）。

### 为获得良好的 hyperopt 结果，需要多少轮评估？

不加 `-e` / `--epochs` 参数，默认只运行 100 次评估（100次触发条件、守卫参数的测试），这个数量很少，难以找到最优解（除非非常幸运）。  
通常建议运行 10000 次甚至更多，但耗时会非常长。

由于 hyperopt 使用贝叶斯搜索，评估轮次过多不一定能带来更佳结果。  
通常建议多次运行，反复累计总轮次达到至少 10000 次，然后判断结果。

示例命令：
```bash
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --strategy SampleStrategy -e 1000
```

### 为什么 hyperopt 运行时间很长？

* 这是因为策略优化涉及大量参数组合的评估。  
建议多参加社区交流（[Freqtrade官网](https://www.freqtrade.io)，文档，Discord 社区），等待找到适合你的“黄金策略”。

* 1000轮hyperopt的耗时取决于多个因素，例如：
  - 你的CPU性能
  - 硬盘速度
  - RAM容量
  - 选用的时间范围（`--timerange`）
  - 指标和参数的设置复杂度
  - 交易对数量
  - 交易总次数（对应利润潜力，可能是几百次，也可能是几万次交易）

举例：
4%的利润，650次交易，与0.3%的利润，每次交易1万次，总耗时差别巨大。如果设定 `--timerange` 为 365天，耗时会相应增加。

示例命令：
```bash
freqtrade --config config.json --strategy SampleStrategy --hyperopt SampleHyperopt -e 1000 --timerange 20190601-20200601
```

## Edge 模块

### Edge 利用了一种控制仓位的有趣方法，有什么理论依据吗？

Edge 模块主要是由 [@mishaker](https://github.com/mishaker) 和 [@creslinux](https://github.com/creslinux) 频率交易团队成员的脑力激荡结果。  

你可以参考以下资源了解期望值、胜率、风险管理和仓位控制的相关理论：  

- https://www.tradeciety.com/ultimate-math-guide-for-traders/  
- https://samuraitradingacademy.com/trading-expectancy/  
- https://www.learningmarkets.com/determining-expectancy-in-your-trading/  
- https://www.lonestocktrader.com/make-money-trading-positive-expectancy/  
- https://www.babypips.com/trading/trade-expectancy-matter

## 官方渠道

Freqtrade 仅通过以下官方渠道沟通和发布信息：

* [Freqtrade Discord 服务器](https://discord.gg/p7nuUNVfP7)  
* [Freqtrade 官方文档 (https://freqtrade.io)](https://freqtrade.io)  
* [Freqtrade Github 组织](https://github.com/freqtrade)  

没有任何关联Freqtrade项目的人员会索要你的交易所密钥或其他可能造成资金泄露的信息。  
如果有人要求你泄露密钥或转账到陌生钱包，请勿理会。

请勿相信或遵从此类不实请求，否则后果由你自己承担。

## "Freqtrade 代币"

Freqtrade 暂无加密货币代币发行计划。

网络上出现的与 Freqtrade、FreqAI 或 freqUI 相关的“代币”项目，均可判断为诈骗，试图借助Freqtrade的知名度牟取不法利益。