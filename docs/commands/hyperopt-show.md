用法：freqtrade hyperopt-show [-h] [-v] [--no-color] [--logfile FILE] [-V]
                               [-c PATH] [-d PATH] [--userdir PATH] [--best]
                               [--profitable] [-n INT] [--print-json]
                               [--hyperopt-filename FILENAME] [--no-header]
                               [--disable-param-export]
                               [--breakdown {day,week,month,year} [{day,week,month,year} ...]]

选项：
  -h, --help            显示此帮助信息并退出
  --best                仅选择表现最优的轮次。
  --profitable          仅选择盈利的轮次。
  -n INT, --index INT   指定要打印详细信息的轮次索引。
  --print-json          以JSON格式输出结果。
  --hyperopt-filename FILENAME
                        Hyperopt结果文件名。例如：`--hyperopt-filename=hyperopt_results_2020-09-27_16-20-48.pickle`
  --no-header           不打印轮次详情表头。
  --disable-param-export
                        禁用自动导出hyperopt参数。
  --breakdown {day,week,month,year} [{day,week,month,year} ...]
                        按[天、周、月、年]显示回测细节拆分。

常用参数：
  -v, --verbose         详细输出模式（-vv获得更多信息，-vvv显示所有消息）。
  --no-color            禁用高亮显示hyperopt结果。若将输出重定向到文件，可能会有用。
  --logfile FILE, --log-file FILE
                        日志输出到指定文件。特殊值包括：
                        'syslog', 'journald'。详细信息请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json`或`config.json`，取决于哪个存在）。可以使用多个--config选项。也可以设为`-`从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定存放历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径。