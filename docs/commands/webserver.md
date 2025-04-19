用法：freqtrade webserver [-h] [-v] [--no-color] [--logfile FILE] [-V]
                           [-c PATH] [-d PATH] [--userdir PATH]

选项：
  -h, --help            显示此帮助信息并退出

常用参数：
  -v, --verbose         详细模式（-vv 代表更多信息，-vvv 获取全部消息）。
  --no-color            禁用超优化结果的颜色显示。如果你将输出重定向到文件，这可能会很有用。
  --logfile FILE, --log-file FILE
                        记录日志到指定文件。特殊值包括：
                        'syslog', 'journald'。详细信息请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json`，取决于哪个存在）。可以使用多个 --config 选项。也可以设置为 `-`，以从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据所在目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。