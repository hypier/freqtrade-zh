用法：freqtrade download-data [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                    [-c PATH] [-d PATH] [--userdir PATH]
                                    [-p PAIRS [PAIRS ...]] [--pairs-file FILE]
                                    [--days INT] [--new-pairs-days INT]
                                    [--include-inactive-pairs]
                                    [--timerange TIMERANGE] [--dl-trades]
                                    [--convert] [--exchange EXCHANGE]
                                    [-t TIMEFRAMES [TIMEFRAMES ...]] [--erase]
                                    [--data-format-ohlcv {json,jsongz,feather,parquet}]
                                    [--data-format-trades {json,jsongz,feather,parquet}]
                                    [--trading-mode {spot,margin,futures}]
                                    [--prepend]

参数选项：
  -h, --help            显示帮助信息并退出
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制在这些交易对。交易对用空格分隔。
  --pairs-file FILE     包含交易对列表的文件。优先于--pairs或配置中的交易对。
  --days INT            下载给定天数的数据。
  --new-pairs-days INT  下载新交易对的指定天数的数据。默认：`None`。
  --include-inactive-pairs
                        同时下载非活跃交易对的数据。
  --timerange TIMERANGE
                        指定使用的数据时间范围。
  --dl-trades           下载交易数据而非OHLCV数据。
  --convert             将已下载的交易数据转换为OHLCV数据。仅在结合`--dl-trades`使用时有效。对于没有历史OHLCV数据的交易所（例如Kraken），会自动转换。如果不提供此参数，将使用`trades-to-ohlcv`命令将交易数据转换为OHLCV数据。
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。
  -t TIMEFRAMES [TIMEFRAMES ...], --timeframes TIMEFRAMES [TIMEFRAMES ...]
                        指定要下载的时间周期。空格分隔。默认：`1m 5m`。
  --erase               清除所选交易所/交易对/时间周期的所有现有数据。
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的K线（OHLCV）数据存储格式。(默认：`feather`)。
  --data-format-trades {json,jsongz,feather,parquet}
                        下载的交易数据存储格式。(默认：`feather`)。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式。
  --prepend             允许数据前置添加（数据追加被禁用）。

常用参数：
  -v, --verbose         显示详细信息模式（-vv表示更多信息，-vvv表示显示所有消息）。
  --no-color            禁用彩色显示，可能在将输出重定向到文件时有用。
  --logfile FILE, --log-file FILE
                        将日志记录到指定文件。特殊值包括：
                        'syslog', 'journald'。详细信息请参阅文档。
  -V, --version         显示程序版本号并退出。
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或`config.json`，以存在的文件为准）。可以使用多个--config参数。也可以设置为`-`从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据所在目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。