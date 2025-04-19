用法：freqtrade lookahead-analysis [-h] [-v] [--no-color] [--logfile FILE]
                                    [-V] [-c PATH] [-d PATH] [--userdir PATH]
                                    [-s NAME] [--strategy-path PATH]
                                    [--recursive-strategy-search]
                                    [--freqaimodel NAME]
                                    [--freqaimodel-path PATH] [-i TIMEFRAME]
                                    [--timerange TIMERANGE]
                                    [--data-format-ohlcv {json,jsongz,feather,parquet}]
                                    [--max-open-trades INT]
                                    [--stake-amount STAKE_AMOUNT]
                                    [--fee FLOAT] [-p PAIRS [PAIRS ...]]
                                    [--enable-protections]
                                    [--dry-run-wallet DRY_RUN_WALLET]
                                    [--timeframe-detail TIMEFRAME_DETAIL]
                                    [--strategy-list STRATEGY_LIST [STRATEGY_LIST ...]]
                                    [--export {none,trades,signals}]
                                    [--export-filename PATH]
                                    [--freqai-backtest-live-models]
                                    [--minimum-trade-amount INT]
                                    [--targeted-trade-amount INT]
                                    [--lookahead-analysis-exportfilename LOOKAHEAD_ANALYSIS_EXPORTFILENAME]

参数选项：
  -h, --help            显示此帮助信息并退出
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间周期（`1m`、`5m`、`30m`、`1h`、`1d`）。
  --timerange TIMERANGE
                        指定使用的数据时间范围。
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的蜡烛图（OHLCV）数据存储格式。
                        （默认：`feather`）。
  --max-open-trades INT
                        覆盖配置中的`max_open_trades`设置值。
  --stake-amount STAKE_AMOUNT
                        覆盖配置中的`stake_amount`设置值。
  --fee FLOAT           指定手续费比例。将在开仓和平仓时应用两次。
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        限制命令作用于这些交易对。交易对用空格分隔。
  --enable-protections, --enableprotections
                        为回测启用保护措施。会显著减慢回测速度，但会包含配置的保护策略。
  --dry-run-wallet DRY_RUN_WALLET, --starting-balance DRY_RUN_WALLET
                        起始余额，用于回测 / 超参数优化和模拟运行。
  --timeframe-detail TIMEFRAME_DETAIL
                        指定回测的详细时间范围（`1m`、`5m`、`30m`、`1h`、`1d`）。
  --strategy-list STRATEGY_LIST [STRATEGY_LIST ...]
                        提供一个用空格分隔的策略列表用于回测。请注意，时间周期（timeframe）需要在配置或命令行中设置。当与`--export trades`一起使用时，策略名会被注入到文件名中，例如`backtest-data.json`会变成`backtest-data-SampleStrategy.json`。
  --export {none,trades,signals}
                        导出回测结果（默认：交易信号）。
  --export-filename PATH, --backtest-filename PATH
                        使用此文件名保存回测结果。需要同时设置`--export`。示例：`--export-filename=user_data/backtest_results/backtest_today.json`
  --freqai-backtest-live-models
                        使用已准备好的模型进行回测。
  --minimum-trade-amount INT
                        进行前瞻分析的最低交易金额
  --targeted-trade-amount INT
                        前瞻分析的目标交易金额
  --lookahead-analysis-exportfilename LOOKAHEAD_ANALYSIS_EXPORTFILENAME
                        使用此CSV文件名存储前瞻分析结果

常用参数：
  -v, --verbose         详细显示模式（-vv显示更多，-vvv显示全部信息）。
  --no-color            禁用超参数优化结果的颜色显示。如果你将输出重定向到文件可能会有用。
  --logfile FILE, --log-file FILE
                        记录日志到指定文件。特殊值包括：
                        'syslog'、'journald'。详见文档。
  -V, --version         显示程序的版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json`或`config.json`，取决于文件是否存在）。
                        可以使用多个`--config`参数。也可设为`-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。

策略参数：
  -s NAME, --strategy NAME
                        指定将由机器人使用的策略类名。
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        递归搜索策略文件夹中的策略。
  --freqaimodel NAME    指定自定义的freqai模型。
  --freqaimodel-path PATH
                        指定freqai模型的额外查找路径。