用法：freqtrade show-trades [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                [-c PATH] [-d PATH] [--userdir PATH]
                                [--db-url PATH]
                                [--trade-ids TRADE_IDS [TRADE_IDS ...]]
                                [--print-json]

选项：
  -h, --help            显示此帮助信息并退出
  --db-url PATH         覆盖交易数据库的URL，这对于自定义部署非常有用
                        （默认：在实时运行模式下为 `sqlite:///tradesv3.sqlite`，
                        在干运行（Dry Run）模式下为 `sqlite:///tradesv3.dryrun.sqlite`）
  --trade-ids TRADE_IDS [TRADE_IDS ...]
                        指定交易ID列表。
  --print-json          以JSON格式输出结果。

常用参数：
  -v, --verbose         详细模式（使用 -vv 获取更多信息，-vvv 显示所有消息）
  --no-color            禁用超导优化结果的彩色显示。如果你将输出重定向到文件，可能会很有用
  --logfile FILE, --log-file FILE
                        将日志输出到指定文件。特殊值包括：
                        'syslog'、'journald'。详情请参阅文档。
  -V, --version         显示程序版本信息并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json`或`config.json`，以存在的文件为准）
                        可以使用多个 --config 选项。也可以设置为 `-` ，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录的路径。