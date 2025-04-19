用法：freqtrade hyperopt [-h] [-v] [--no-color] [--logfile FILE] [-V]
                          [-c PATH] [-d PATH] [--userdir PATH] [-s NAME]
                          [--strategy-path PATH] [--recursive-strategy-search]
                          [--freqaimodel NAME] [--freqaimodel-path PATH]
                          [-i TIMEFRAME] [--timerange TIMERANGE]
                          [--data-format-ohlcv {json,jsongz,feather,parquet}]
                          [--max-open-trades INT]
                          [--stake-amount STAKE_AMOUNT] [--fee FLOAT]
                          [-p PAIRS [PAIRS ...]] [--hyperopt-path PATH]
                          [--eps] [--enable-protections]
                          [--dry-run-wallet DRY_RUN_WALLET]
                          [--timeframe-detail TIMEFRAME_DETAIL] [-e INT]
                          [--spaces {all,buy,sell,roi,stoploss,trailing,protection,trades,default} [{all,buy,sell,roi,stoploss,trailing,protection,trades,default} ...]]
                          [--print-all] [--print-json] [-j JOBS]
                          [--random-state INT] [--min-trades INT]
                          [--hyperopt-loss NAME] [--disable-param-export]
                          [--ignore-missing-spaces] [--analyze-per-epoch]

选项：
  -h, --help            显示此帮助信息并退出
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间周期（`1m`, `5m`, `30m`, `1h`, `1d`）
  --timerange TIMERANGE
                        指定要使用的数据时间范围
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的蜡烛图数据存储格式。
                        默认值：`feather`
  --max-open-trades INT
                        覆盖配置中的`max_open_trades`值
  --stake-amount STAKE_AMOUNT
                        覆盖配置中的`stake_amount`值
  --fee FLOAT           指定交易手续费比例。将在买入和卖出时均应用。
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        限制命令仅对这些交易对执行。交易对用空格分隔。
  --hyperopt-path PATH  指定附加的超参数优化损失函数查找路径
  --eps, --enable-position-stacking
                        允许对同一交易对多次买入（仓位叠加）
  --enable-protections, --enableprotections
                        启用回测保护措施。会显著减慢回测速度，但会包含已配置的保护策略
  --dry-run-wallet DRY_RUN_WALLET, --starting-balance DRY_RUN_WALLET
                        起始余额，用于回测/超参数优化和Dry-Run
  --timeframe-detail TIMEFRAME_DETAIL
                        指定回测的详细时间周期（`1m`, `5m`, `30m`, `1h`, `1d`）
  -e INT, --epochs INT  指定训练轮次（默认：100）
  --spaces {all,buy,sell,roi,stoploss,trailing,protection,trades,default} [{all,buy,sell,roi,stoploss,trailing,protection,trades,default} ...]
                        指定进行超参数优化的参数空间。空格分隔的列表。
  --print-all           打印所有结果，而不仅仅是最优结果
  --print-json          以JSON格式输出
  -j JOBS, --job-workers JOBS
                        同时运行的超参数优化任务数（hyperopt工作进程数）。如果设为-1（默认），则使用所有CPU；-2表示使用除一个CPU外的所有CPU，以此类推。如果设为1，则不进行并行计算。
  --random-state INT    设置随机种子为某个正整数，以确保超参数优化结果可重现
  --min-trades INT      设置超参数优化中评估所需的最小交易次数（默认：1）
  --hyperopt-loss NAME, --hyperoptloss NAME
                        指定超参数优化的损失函数类名（IHyperOptLoss）。不同的函数会产生完全不同的优化目标。内置的超参数损失函数包括：
                        ShortTradeDurHyperOptLoss, OnlyProfitHyperOptLoss,
                        SharpeHyperOptLoss, SharpeHyperOptLossDaily,
                        SortinoHyperOptLoss, SortinoHyperOptLossDaily,
                        CalmarHyperOptLoss, MaxDrawDownHyperOptLoss,
                        MaxDrawDownRelativeHyperOptLoss,
                        MaxDrawDownPerPairHyperOptLoss,
                        ProfitDrawDownHyperOptLoss, MultiMetricHyperOptLoss
  --disable-param-export
                        禁用自动超参数导出
  --ignore-missing-spaces, --ignore-unparameterized-spaces
                        忽略请求的超参数空间中没有参数的错误
  --analyze-per-epoch   每个轮次运行一次`populate_indicators`

常用参数：
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取全部消息）
  --no-color            禁用超参数优化结果的色彩显示。如果你将结果重定向到文件，这可能会有用。
  --logfile FILE, --log-file FILE
                        将日志输出到指定文件。特殊值包括：
                        'syslog', 'journald'。详见相关文档。
  -V, --version         显示程序版本信息并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json`，以存在的文件为准）。可以使用多个--config选项。也可以设为`-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定包含历史回测数据的目录路径
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径

策略参数：
  -s NAME, --strategy NAME
                        指定将由交易机器人使用的策略类名
  --strategy-path PATH  指定额外的策略查找路径
  --recursive-strategy-search
                        递归搜索策略文件夹中的策略
  --freqaimodel NAME    指定自定义的freqaimodel
  --freqaimodel-path PATH
                        指定freqaimodel的额外查找路径