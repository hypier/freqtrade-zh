用法：freqtrade convert-trade-data [-h] [-v] [--no-color] [--logfile FILE]
                                      [-V] [-c PATH] [-d PATH] [--userdir PATH]
                                      [-p PAIRS [PAIRS ...]] --format-from
                                      {json,jsongz,feather,parquet,kraken_csv}
                                      --format-to {json,jsongz,feather,parquet}
                                      [--erase] [--exchange EXCHANGE]

选项：
  -h, --help            显示此帮助信息并退出
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制为这些交易对。交易对之间用空格分隔。
  --format-from {json,jsongz,feather,parquet,kraken_csv}
                        源数据格式，用于数据转换。
  --format-to {json,jsongz,feather,parquet}
                        目标数据格式，用于数据转换。
  --erase               清除所选交易所/交易对/时间周期的所有现有数据。
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。

常用参数：
  -v, --verbose         详细模式（-vv表示更详细，-vvv显示所有消息）。
  --no-color            禁用超参数优化结果的颜色显示。如果你将输出重定向到文件，可能会有用。
  --logfile FILE, --log-file FILE
                        日志输出到指定文件。特殊值包括：
                        'syslog'，'journald'。详情请查阅相关文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或`config.json`，取决于哪个存在）。可以使用多个 --config 选项。也可以设置为 `-` 从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据所在目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。