用法: freqtrade trades-to-ohlcv [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                   [-c PATH] [-d PATH] [--userdir PATH]
                                   [-p PAIRS [PAIRS ...]]
                                   [-t TIMEFRAMES [TIMEFRAMES ...]]
                                   [--exchange EXCHANGE]
                                   [--data-format-ohlcv {json,jsongz,feather,parquet}]
                                   [--data-format-trades {json,jsongz,feather,parquet}]
                                   [--trading-mode {spot,margin,futures}]

选项:
  -h, --help            显示此帮助信息并退出
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制在这些交易对。交易对用空格分隔。
  -t TIMEFRAMES [TIMEFRAMES ...], --timeframes TIMEFRAMES [TIMEFRAMES ...]
                        指定要下载的币种时间框架。空格分隔的列表。默认：`1m 5m`。
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的K线（OHLCV）数据的存储格式。
                        （默认：`feather`）。
  --data-format-trades {json,jsongz,feather,parquet}
                        下载的交易数据的存储格式。（默认：`feather`）。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式

常用参数:
  -v, --verbose         详细模式（-vv表示更多信息，-vvv显示所有信息）
  --no-color            禁用超频优化结果的彩色显示。如果将输出重定向到文件，这可能会有用。
  --logfile FILE, --log-file FILE
                        将日志记录到指定文件。特殊值为：
                        'syslog'、'journald'。更多详情请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或`config.json`（如果存在））。可以使用多个 --config 选项。也可以设置为 `-` 从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。