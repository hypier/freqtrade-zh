用法：freqtrade list-exchanges [-h] [-v] [--no-color] [--logfile FILE] [-V] [-c PATH] [-d PATH] [--userdir PATH] [-1] [-a]

选项：
  -h, --help            显示此帮助信息并退出
  -1, --one-column      以一列的形式输出结果
  -a, --all             输出ccxt库已知的所有交易所信息

常用参数：
  -v, --verbose         详细模式（-vv显示更多信息，-vvv显示全部消息）
  --no-color            禁用超参数优化结果的彩色显示。如果你将输出重定向到文件，这可能会很有用
  --logfile FILE, --log-file FILE
                        记录日志到指定文件。特殊值包括：
                        'syslog'，'journald'。详情请参阅文档
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json`，以存在的为准）。可以多次使用 --config
                        选项。可以设置为 `-` ，从标准输入读取配置
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据所在目录路径
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径