用法：freqtrade list-freqaimodels [-h] [-v] [--no-color] [--logfile FILE]
                                  [-V] [-c PATH] [-d PATH] [--userdir PATH]
                                  [--freqaimodel-path PATH] [-1]

选项：
  -h, --help            显示此帮助信息并退出
  --freqaimodel-path PATH
                        指定 freqaimodels 的额外查找路径。
  -1, --one-column      以单列方式输出结果。

常用参数：
  -v, --verbose         详细模式（-vv 输出更多信息，-vvv 获取所有信息）
  --no-color            禁用超参数优化结果的颜色显示。如果你将输出重定向到文件，这可能会有用。
  --logfile FILE, --log-file FILE
                        将日志记录到指定文件。特殊值包括：
                        'syslog', 'journald'。详细信息请参阅文档。
  -V, --version         显示程序的版本信息并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json`，以存在的文件为准）。
                        可以使用多个 --config 选项。也可以设置为 `-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        包含历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。

```