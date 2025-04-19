用法: freqtrade list-data [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                   [-c PATH] [-d PATH] [--userdir PATH]
                                   [--exchange EXCHANGE]
                                   [--data-format-ohlcv {json,jsongz,feather,parquet}]
                                   [--data-format-trades {json,jsongz,feather,parquet}]
                                   [--trades] [-p PAIRS [PAIRS ...]]
                                   [--trading-mode {spot,margin,futures}]
                                   [--show-timerange]

选项:
  -h, --help            显示帮助信息并退出
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的蜡烛图（OHLCV）数据存储格式。
                        （默认：`feather`）。
  --data-format-trades {json,jsongz,feather,parquet}
                        下载的交易数据存储格式。（默认：
                        `feather`）。
  --trades              处理交易数据而非OHLCV数据。
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制在指定的交易对上。交易对用空格分隔。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式
  --show-timerange      显示可用数据的时间范围。 （可能需要一些时间计算）。

常用参数:
  -v, --verbose         详细模式 (-vv 获取更多信息, -vvv 获取全部消息)。
  --no-color            禁用超参数优化结果的彩色显示。如果你将输出重定向到文件，可能会有用。
  --logfile FILE, --log-file FILE
                        记录日志到指定文件。特殊值包括：
                        'syslog'、'journald'。详情请参见相关文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json` 或 `config.json`，取决于存在的文件）。可以使用多个 --config 选项。也可以设置为 `-` 从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定存放历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径。