用法：freqtrade backtesting [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                 [-c PATH] [-d PATH] [--userdir PATH] [-s NAME]
                                 [--strategy-path PATH]
                                 [--recursive-strategy-search]
                                 [--freqaimodel NAME] [--freqaimodel-path PATH]
                                 [-i TIMEFRAME] [--timerange TIMERANGE]
                                 [--data-format-ohlcv {json,jsongz,feather,parquet}]
                                 [--max-open-trades INT]
                                 [--stake-amount STAKE_AMOUNT] [--fee FLOAT]
                                 [-p PAIRS [PAIRS ...]] [--eps]
                                 [--enable-protections]
                                 [--dry-run-wallet DRY_RUN_WALLET]
                                 [--timeframe-detail TIMEFRAME_DETAIL]
                                 [--strategy-list STRATEGY_LIST [STRATEGY_LIST ...]]
                                 [--export {none,trades,signals}]
                                 [--export-filename PATH]
                                 [--breakdown {day,week,month,year} [{day,week,month,year} ...]]
                                 [--cache {none,day,week,month}]
                                 [--freqai-backtest-live-models]

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
                        覆盖配置中的 `max_open_trades` 值。
  --stake-amount STAKE_AMOUNT
                        覆盖配置中的 `stake_amount` 值。
  --fee FLOAT           指定手续费比例。将在入场和出场时两次应用。
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        限定交易对，多个用空格分隔。
  --eps, --enable-position-stacking
                        允许多次买入同一交易对（仓位叠加）。
  --enable-protections, --enableprotections
                        在回测中启用保护措施。会显著降低回测速度，但会包括配置的保护策略。
  --dry-run-wallet DRY_RUN_WALLET, --starting-balance DRY_RUN_WALLET
                        初始资金，用于回测 / 超参数优化和干运行。
  --timeframe-detail TIMEFRAME_DETAIL
                        指定回测的详细时间周期（`1m`、`5m`、`30m`、`1h`、`1d`）。
  --strategy-list STRATEGY_LIST [STRATEGY_LIST ...]
                        提供一个用空格分隔的策略列表用于回测。请注意，时间周期必须在配置或命令行中设置。当与`--export trades`一起使用时，策略名会加入到文件名中（例如`backtest-data.json`变为`backtest-data-SampleStrategy.json`）
  --export {none,trades,signals}
                        导出回测结果（默认：trades）。
  --export-filename PATH, --backtest-filename PATH
                        使用此文件名保存回测结果。必须同时指定`--export`。示例：`--export-filename=user_data/backtest_results/backtest_today.json`
  --breakdown {day,week,month,year} [{day,week,month,year} ...]
                        按天、周、月、年显示回测细分结果。
  --cache {none,day,week,month}
                        加载不早于指定年龄的缓存回测结果（默认：day）。
  --freqai-backtest-live-models
                        使用已准备好的模型进行回测。

常用参数：
  -v, --verbose         详细输出模式（-vv获得更多信息，-vvv输出所有信息）。
  --no-color            禁用超参数优化结果的彩色显示。若将输出重定向到文件，此选项可能有用。
  --logfile FILE, --log-file FILE
                        将日志输出到指定文件。特殊值：
                        'syslog'、'journald'。详情请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或存在的`config.json`）。可以使用多个--config参数。也可以设为`-`从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指向存放历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        指向用户数据目录。

策略参数：
  -s NAME, --strategy NAME
                        指定将由机器人使用的策略类名。
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        在策略文件夹中递归搜索策略。
  --freqaimodel NAME    指定自定义的freqaimodel。
  --freqaimodel-path PATH
                        指定freqaimodel的额外查找路径。