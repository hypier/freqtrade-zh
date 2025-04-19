用法：freqtrade plot-profit [-h] [-v] [--no-color] [--logfile FILE] [-V]
                              [-c PATH] [-d PATH] [--userdir PATH] [-s NAME]
                              [--strategy-path PATH]
                              [--recursive-strategy-search]
                              [--freqaimodel NAME] [--freqaimodel-path PATH]
                              [-p PAIRS [PAIRS ...]] [--timerange TIMERANGE]
                              [--export {none,trades,signals}]
                              [--export-filename PATH] [--db-url PATH]
                              [--trade-source {DB,file}] [-i TIMEFRAME]
                              [--auto-open]

选项：
  -h, --help            显示此帮助信息并退出
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制在这些交易对。交易对之间用空格分隔。
  --timerange TIMERANGE
                        指定使用的数据时间范围。
  --export {none,trades,signals}
                        导出回测结果（默认：trades）。
  --export-filename PATH, --backtest-filename PATH
                        使用此文件名保存回测结果。需要同时设置`--export`。例如：`--export-filename=user_data/backtest_results/backtest_today.json`
  --db-url PATH         覆盖交易日志数据库URL，在自定义部署中非常有用（默认：`sqlite:///tradesv3.sqlite` 用于实时运行，`sqlite:///tradesv3.dryrun.sqlite` 用于模拟运行）。
  --trade-source {DB,file}
                        指定交易数据来源（可以是数据库或文件（回测文件））。默认：file
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间周期（`1m`、`5m`、`30m`、`1h`、`1d`）。
  --auto-open           自动打开生成的图表。

常用参数：
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取全部消息）。
  --no-color            禁用高亮显示结果的颜色，适合将输出重定向到文件。
  --logfile FILE, --log-file FILE
                        将日志信息写入指定文件。特殊值包括：
                        'syslog'、'journald'。详细信息请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json` 或 `config.json` ，根据实际存在的文件而定）。可以多次使用此参数。也可以设为`-`以从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定存放历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径。

策略相关参数：
  -s NAME, --strategy NAME
                        指定将被机器人使用的策略类名。
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        在策略文件夹中递归搜索策略。
  --freqaimodel NAME    指定自定义的频率模型。
  --freqaimodel-path PATH
                        指定额外的频率模型查找路径。