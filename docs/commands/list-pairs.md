用法: freqtrade list-pairs [-h] [-v] [--no-color] [--logfile FILE] [-V]
                            [-c PATH] [-d PATH] [--userdir PATH]
                            [--exchange EXCHANGE] [--print-list]
                            [--print-json] [-1] [--print-csv]
                            [--base BASE_CURRENCY [BASE_CURRENCY ...]]
                            [--quote QUOTE_CURRENCY [QUOTE_CURRENCY ...]] [-a]
                            [--trading-mode {spot,margin,futures}]

参数选项：
  -h, --help            显示帮助信息并退出
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。
  --print-list          打印交易对或市场符号列表。默认以表格格式输出数据。
  --print-json          以JSON格式打印交易对或市场符号列表。
  -1, --one-column      以单列形式输出。
  --print-csv           以csv格式输出交易所对或市场数据。
  --base BASE_CURRENCY [BASE_CURRENCY ...]
                        指定基础币种。用空格分隔多个币种。
  --quote QUOTE_CURRENCY [QUOTE_CURRENCY ...]
                        指定报价币种。用空格分隔多个币种。
  -a, --all             打印所有交易对或市场符号。默认只显示活跃的。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式

常用参数：
  -v, --verbose         使用详细模式 (-vv 获取更多信息，-vvv 获取全部消息)。
  --no-color            禁用超时结果的颜色化。如果你将输出重定向到文件，可能会有用。
  --logfile FILE, --log-file FILE
                        将日志输出到指定文件。特殊值包括：
                        'syslog', 'journald'。详细信息请参阅相关文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json` ，取决于实际存在的文件）。
                        可以使用多个 --config 选项。也可设置为 `-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定存放历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。