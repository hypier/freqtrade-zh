用法：freqtrade convert-data [-h] [-v] [--no-color] [--logfile FILE] [-V]
                              [-c PATH] [-d PATH] [--userdir PATH]
                              [-p PAIRS [PAIRS ...]] --format-from
                              {json,jsongz,feather,parquet} --format-to
                              {json,jsongz,feather,parquet} [--erase]
                              [--exchange EXCHANGE]
                              [-t TIMEFRAMES [TIMEFRAMES ...]]
                              [--trading-mode {spot,margin,futures}]
                              [--candle-types {spot,futures,mark,index,premiumIndex,funding_rate} [{spot,futures,mark,index,premiumIndex,funding_rate} ...]]

选项：
  -h, --help            显示此帮助信息并退出
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制在这些交易对。交易对以空格分隔。
  --format-from {json,jsongz,feather,parquet}
                        数据转换的源格式。
  --format-to {json,jsongz,feather,parquet}
                        数据转换的目标格式。
  --erase               清除所选交易所/交易对/时间框架的所有现有数据。
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。
  -t TIMEFRAMES [TIMEFRAMES ...], --timeframes TIMEFRAMES [TIMEFRAMES ...]
                        指定需要下载的行情时间周期。以空格分隔的列表。默认：`1m 5m`。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式
  --candle-types {spot,futures,mark,index,premiumIndex,funding_rate} [{spot,futures,mark,index,premiumIndex,funding_rate} ...]
                        选择要转换的K线类型。默认全部可用类型。

常用参数：
  -v, --verbose         显示详细信息模式（-vv 获取更多信息，-vvv 获取所有消息）。
  --no-color            禁用超参数优化结果的彩色显示。若将输出重定向到文件可能很有用。
  --logfile FILE, --log-file FILE
                        将日志记录到指定文件。特殊值包括：'syslog'，'journald'。详见文档。
  -V, --version         显示程序的版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或存在的`config.json`）。可以使用多个 --config 选项。也可以设置为`-`从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据所在目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。