用法：freqtrade list-markets [-h] [-v] [--no-color] [--logfile FILE] [-V]
                              [-c PATH] [-d PATH] [--userdir PATH]
                              [--exchange EXCHANGE] [--print-list]
                              [--print-json] [-1] [--print-csv]
                              [--base BASE_CURRENCY [BASE_CURRENCY ...]]
                              [--quote QUOTE_CURRENCY [QUOTE_CURRENCY ...]]
                              [-a] [--trading-mode {spot,margin,futures}]

参数选项：
  -h, --help            显示此帮助信息并退出
  --exchange EXCHANGE   交易所名称。仅在未提供配置文件时有效。
  --print-list          输出交易对或市场符号列表。默认以表格格式显示数据。
  --print-json          以JSON格式输出交易对或市场符号列表。
  -1, --one-column      以单列格式输出结果。
  --print-csv           以csv格式输出交易所对或市场数据。
  --base BASE_CURRENCY [BASE_CURRENCY ...]
                        指定基础货币。空格分隔的列表。
  --quote QUOTE_CURRENCY [QUOTE_CURRENCY ...]
                        指定报价货币。空格分隔的列表。
  -a, --all             输出所有的交易对或市场符号。默认只显示活跃的。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式

常用参数：
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取所有消息）。
  --no-color            禁用高亮显示hyperopt结果。用于将输出重定向到文件时可能会有帮助。
  --logfile FILE, --log-file FILE
                        记录日志到指定文件。特殊值包括：
                        'syslog', 'journald'。详情请见相关文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或存在的`config.json`）。
                        可以使用多个 --config 选项。也可设为 `-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据存放目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。