用法：freqtrade recursive-analysis [-h] [-v] [--no-color] [--logfile FILE]
                                        [-V] [-c PATH] [-d PATH] [--userdir PATH]
                                        [-s NAME] [--strategy-path PATH]
                                        [--recursive-strategy-search]
                                        [--freqaimodel NAME]
                                        [--freqaimodel-path PATH] [-i TIMEFRAME]
                                        [--timerange TIMERANGE]
                                        [--data-format-ohlcv {json,jsongz,feather,parquet}]
                                        [-p PAIRS [PAIRS ...]]
                                        [--startup-candle STARTUP_CANDLE [STARTUP_CANDLE ...]]

参数说明：
  -h, --help            显示帮助信息并退出
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间周期（`1m`、`5m`、`30m`、`1h`、`1d`）
  --timerange TIMERANGE
                        指定要使用的数据时间范围
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的蜡烛（OHLCV）数据存储格式
                        （默认：`feather`）
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制在这些交易对上，交易对用空格分隔
  --startup-candle STARTUP_CANDLE [STARTUP_CANDLE ...]
                        指定需要检查的启动蜡烛（如：`199`、`499`、
                        `999`、`1999`）

常用参数：
  -v, --verbose         输出详细信息（-vv 获取更多信息，-vvv 获取所有消息）
  --no-color            禁用超参数优化结果的颜色显示。如果你将输出重定向到文件，这可能会很有用
  --logfile FILE, --log-file FILE
                        指定日志文件。特殊值包括：
                        'syslog'、'journald'。详细信息请参阅文档。
  -V, --version         显示程序版本信息并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或`config.json`，以存在的文件为准）。可以使用多个--config参数。也可以设为`-`，从标准输入读取配置
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据目录路径
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径

策略相关参数：
  -s NAME, --strategy NAME
                        指定将由机器人使用的策略类名
  --strategy-path PATH  指定额外的策略查找路径
  --recursive-strategy-search
                        递归搜索策略文件夹中的策略
  --freqaimodel NAME    指定自定义的freqaimodel模型
  --freqaimodel-path PATH
                        指定freqaimodel模型的额外查找路径