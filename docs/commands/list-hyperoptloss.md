用法：freqtrade list-hyperoptloss [-h] [-v] [--no-color] [--logfile FILE]
                                      [-V] [-c PATH] [-d PATH] [--userdir PATH]
                                      [--hyperopt-path PATH] [-1]

选项：
  -h, --help            显示此帮助信息并退出
  --hyperopt-path PATH  指定 Hyperopt Loss 函数的额外查找路径
  -1, --one-column      以单列格式输出

常用参数：
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取全部消息）
  --no-color            禁用超参数优化结果的彩色显示。若将输出重定向到文件时可能会有用。
  --logfile FILE, --log-file FILE
                        记录日志到指定文件。特殊值包括：
                        'syslog', 'journald'。详细信息请查阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json`，以存在的文件为准）。可以使用多个 --config 选项。也可以设为 `-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。

```