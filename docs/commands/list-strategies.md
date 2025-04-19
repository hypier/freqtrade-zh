用法：freqtrade list-strategies [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                   [-c PATH] [-d PATH] [--userdir PATH]
                                   [--strategy-path PATH] [-1]
                                   [--recursive-strategy-search]

选项：
  -h, --help            显示此帮助信息并退出
  --strategy-path PATH  指定额外的策略查找路径
  -1, --one-column      以单列格式输出结果
  --recursive-strategy-search
                        递归搜索策略文件夹中的策略

常用参数：
  -v, --verbose         详细模式（-vv 表示更多信息，-vvv 获取全部消息）
  --no-color            禁用超参数优化结果的彩色显示。若将输出重定向到文件，可能会有用。
  --logfile FILE, --log-file FILE
                        将日志记录到指定文件。特殊值包括：
                        'syslog'、'journald'。详情请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json`，以存在的文件为准）。
                        可以使用多个 --config 选项。也可以设为 `-` 以从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定包含历史回测数据的目录路径
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径

```