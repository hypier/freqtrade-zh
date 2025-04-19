# Freqtrade 基础知识

本页面为您介绍 Freqtrade 的一些基本概念，帮助您理解其工作与运行机制。

## Freqtrade 术语

* **Strategy（策略）**：您的交易策略，指导机器人该采取的操作。
* **Trade（交易）**：已开仓的持仓。
* **Open Order（挂单）**：当前已挂在交易所的订单，尚未完成。
* **Pair（交易对）**：可交易的货币对，通常格式为 Base/Quote（例如，现货市场的 `XRP/USDT`，期货市场的 `XRP/USDT:USDT`）。
* **Timeframe（时间框架）**：蜡烛图周期长度（例如 `"5m"`、`"1h"` 等）。
* **Indicators（技术指标）**：技术分析指标（如 SMA、EMA、RSI 等）。
* **Limit order（限价单）**：在定义的限价或更优价格执行的订单。
* **Market order（市价单）**：保证成交的订单，成交价格可能因订单量而变动。
* **Current Profit（当前盈利）**：当前未实现的盈利。这一信息主要在机器人和UI中使用。
* **Realized Profit（已实现盈利）**：已结算的盈利，只有在结合 [部分退出](strategy-callbacks.md#adjust-trade-position) 时才相关，同时也包含计算逻辑的说明。
* **Total Profit（总盈利）**：已实现和未实现盈利的总和。百分比（%）相对于总投资计算。

## 手续费处理

Freqtrade 的所有盈利计算都包含手续费。在回测、超参数优化（Hyperopt）或模拟运行（Dry-run）模式下，使用交易所的默认费率（交易所最低费率）。在实盘操作中，手续费按照交易所实际收取，包括 BNB 抵扣等。

## 交易对命名

Freqtrade 遵循 [ccxt 的命名规范](https://docs.ccxt.com/#/README?id=consistency-of-base-and-quote-currencies) 来命名货币对。
在不同市场使用错误的命名规范通常会导致机器人无法识别交易对，表现为“此交易对不可用”等错误。

### 现货交易对命名

现货市场的交易对命名为 `base/quote`（例如：`ETH/USDT`）。

### 期货交易对命名

期货市场的交易对命名为 `base/quote:settle`（例如：`ETH/USDT:USDT`）。

## 机器人执行逻辑

启动 freqtrade（无论是模拟模式还是实盘模式，使用 `freqtrade trade` 命令）后，机器人将开始运行，并进入循环执行流程。
此流程启动时会执行 `bot_start()` 回调。

默认情况下，机器人循环每隔几秒（`internals.process_throttle_secs`）运行一次，执行以下操作：

* 从持久化存储中获取未平仓交易。
* 计算当前可交易的货币对列表。
* 下载所有【信息性交易对】的OHLCV数据（包括非交易对，详见 [策略定制](strategy-customization.md#get-data-for-non-tradeable-pairs)），此步骤每蜡烛线只执行一次以减少网络请求。
* 调用 `bot_loop_start()` 策略回调。
* 针对每个交易对进行策略分析。
  * 调用 `populate_indicators()`
  * 调用 `populate_entry_trend()`
  * 调用 `populate_exit_trend()`
* 从交易所更新未平仓订单状态。
  * 对已成交订单调用 `order_filled()` 回调。
  * 检查未成交订单的超时情况。
    * 对未成交的挂单调用 `check_entry_timeout()`。
    * 对已开放的退出订单调用 `check_exit_timeout()`。
    * 调用 `adjust_order_price()` 调整未成交订单的价格。
      * 调用 `adjust_entry_price()`（仅在 `adjust_order_price()` 未实现时调用）
      * 调用 `adjust_exit_price()`（仅在 `adjust_order_price()` 未实现时调用）
* 核查已有仓位，并在必要时放置退出订单。
  * 考虑止损、ROI、退出信号、`custom_exit()` 和 `custom_stoploss()`。
  * 根据 `exit_pricing` 配置或 `custom_exit_price()` 计算退出价格。
  * 提交退出订单前，调用 `confirm_trade_exit()` 策略回调确认。
* 调整未平仓交易的一些参数（如仓位调整），如果启用，调用 `adjust_trade_position()`，并根据需要加入额外订单。
* 检查是否还存在空闲交易位置（如果已达到 `max_open_trades` 上限）。
* 审查进入新仓位的交易信号。
  * 根据 `entry_pricing` 配置或 `custom_entry_price()` 计算入场价格。
  * 在保证金和期货模式中，调用 `leverage()` 策略回调以确定期望的杠杆倍数。
  * 通过调用 `custom_stake_amount()` 获取持仓金额。
  * 提交入场订单前，调用 `confirm_trade_entry()` 策略回调。

该循环将不断重复运行，直到机器人被停止。

## 回测 / 超参数优化的执行逻辑

[回测](backtesting.md) 和 [超参数优化](hyperopt.md) 仅执行上述部分逻辑，因为大部分交易操作为完全模拟。

* 加载配置的交易对历史数据。
* 仅调用一次 `bot_start()`。
* 计算指标（每个交易对调用一次 `populate_indicators()`）。
* 计算入场/离场信号（每个交易对调用一次 `populate_entry_trend()` 和 `populate_exit_trend()`）。
* 按蜡烛线模拟入场与离场点。
  * 调用 `bot_loop_start()` 策略回调。
  * 检查订单超时（通过 `unfilledtimeout` 配置或 `check_entry_timeout()` / `check_exit_timeout()` 回调）。
  * 调用 `adjust_order_price()` 调整未成交订单价格。
    * 调用 `adjust_entry_price()`（仅在未实现 `adjust_order_price()` 时调用）
    * 调用 `adjust_exit_price()`（仅在未实现 `adjust_order_price()` 时调用）
  * 检查交易信号（`enter_long` / `enter_short` 列）。
  * 确认入场/离场（调用 `confirm_trade_entry()` 和 `confirm_trade_exit()`，如果策略中已实现）。
  * 调用 `custom_entry_price()`（如策略中实现）以确定入场价格（价格会调整至开盘蜡烛内）。
  * 在保证金和期货模式中，调用 `leverage()` 策略回调以确定期望的杠杆。
  * 调用 `custom_stake_amount()` 获取持仓金额。
  * 如果启用，检查未平仓交易的仓位调整，并调用 `adjust_trade_position()` 判断是否需要额外订单。
  * 对已成交的入场订单调用 `order_filled()`。
  * 调用 `custom_stoploss()` 和 `custom_exit()` 获取自定义离场点。
  * 对基于退出信号、部分退出或自定义退出的情况：调用 `custom_exit_price()` 获取离场价格（价格调整至收盘蜡烛内）。
  * 对已成交的离场订单调用 `order_filled()`。
* 输出回测报告。

!!! 注意
    回测和超参数优化都包含交易所默认手续费的计算。可以通过指定 `--fee` 参数传递自定义手续费。  

!!! 警告 “回调调用频率”
    回测将最多每蜡烛调用一次每个回调（也可通过 `--timeframe-detail` 将调用频率细化到每个详细蜡烛）。
    绝大多数回调在实盘中每次循环（通常每5秒左右）调用一次，这可能导致回测结果与实盘不一致。