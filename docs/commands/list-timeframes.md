用法: freqtrade list-timeframes [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                     [-c PATH] [-d PATH] [--userdir PATH]
                                     [--exchange EXCHANGE] [-1]

选项：
  -h, --help            显示此帮助信息并退出
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。
  -1, --one-column      以单列方式打印输出。

常用参数：
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取全部消息）
  --no-color            禁用超参数搜索结果的彩色显示。若将输出重定向到文件，使用此选项可能更合适。
  --logfile FILE, --log-file FILE
                        将日志写入指定文件。特殊值包括：
                        'syslog' 和 'journald'。详细信息请参阅文档。
  -V, --version         显示程序版本信息并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json` 或 `config.json`，以存在的文件为准）。可以使用多个 --config 选项。也可设置为 `-` 从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定包含历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径。

```