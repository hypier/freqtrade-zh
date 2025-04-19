用法：freqtrade backtesting-show [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                  [-c PATH] [-d PATH] [--userdir PATH]
                                  [--export-filename PATH] [--show-pair-list]
                                  [--breakdown {day,week,month,year} [{day,week,month,year} ...]]

选项：
  -h, --help            显示此帮助信息并退出
  --export-filename PATH, --backtest-filename PATH
                        使用此文件名保存回测结果。需要同时设置`--export`。示例：`--export-filename=user_data/backtest_results/backtest_today.json`
  --show-pair-list      按利润排序显示回测交易对列表
  --breakdown {day,week,month,year} [{day,week,month,year} ...]
                        按[天、周、月、年]显示回测细分统计

常用参数：
  -v, --verbose         详细输出模式 (-vv 获取更多信息，-vvv 获取全部消息)
  --no-color            禁用超参数优化结果的彩色显示。如果你将输出重定向到文件，可能会很有用。
  --logfile FILE, --log-file FILE
                        日志输出到指定文件。特殊值包括：
                        'syslog', 'journald'。详情请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或存在的`config.json`）。可以使用多个--config选项。也可以设为`-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据存放目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。