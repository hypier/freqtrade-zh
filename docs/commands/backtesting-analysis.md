用法：freqtrade backtesting-analysis [-h] [-v] [--no-color] [--logfile FILE]
                                      [-V] [-c PATH] [-d PATH]
                                      [--userdir PATH]
                                      [--export-filename PATH]
                                      [--analysis-groups {0,1,2,3,4,5} [{0,1,2,3,4,5} ...]]
                                      [--enter-reason-list ENTER_REASON_LIST [ENTER_REASON_LIST ...]]
                                      [--exit-reason-list EXIT_REASON_LIST [EXIT_REASON_LIST ...]]
                                      [--indicator-list INDICATOR_LIST [INDICATOR_LIST ...]]
                                      [--entry-only] [--exit-only]
                                      [--timerange TIMERANGE]
                                      [--rejected-signals] [--analysis-to-csv]
                                      [--analysis-csv-path ANALYSIS_CSV_PATH]

参数选项：
  -h, --help            显示帮助信息并退出
  --export-filename PATH, --backtest-filename PATH
                        使用该文件名保存回测结果。需要同时设置 `--export`。示例：`--export-filename=user_data/backtest_results/backtest_today.json`
  --analysis-groups {0,1,2,3,4,5} [{0,1,2,3,4,5} ...]
                        分组输出 - 0：按入场标签的简单赢/亏情况，1：按入场标签，2：按入场和退出标签，3：按对和入场标签，4：按对、入场和退出标签（可能会很大），5：按退出标签
  --enter-reason-list ENTER_REASON_LIST [ENTER_REASON_LIST ...]
                        要分析的入场信号列表，用空格分隔。默认：全部。例如：'entry_tag_a entry_tag_b'
  --exit-reason-list EXIT_REASON_LIST [EXIT_REASON_LIST ...]
                        要分析的退出信号列表，用空格分隔。默认：全部。例如：'exit_tag_a roi stop_loss trailing_stop_loss'
  --indicator-list INDICATOR_LIST [INDICATOR_LIST ...]
                        要分析的指标列表，用空格分隔。例如：'close rsi bb_lowerband profit_abs'
  --entry-only          仅分析入场信号。
  --exit-only           仅分析退出信号。
  --timerange TIMERANGE
                        指定使用的数据时间范围。
  --rejected-signals    分析被拒绝的信号
  --analysis-to-csv     将选定的分析表保存为单独的CSV文件
  --analysis-csv-path ANALYSIS_CSV_PATH
                        指定保存分析CSV文件的路径，前提是启用--analysis-to-csv。默认为：
                        user_data/backtesting_results/

常用参数：
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取全部信息）。
  --no-color            禁用超越优化结果的颜色显示。如果你将输出重定向到文件，这可能会很有用。
  --logfile FILE, --log-file FILE
                        将日志输出到指定文件。特殊值包括：
                        'syslog', 'journald'。详情请参阅相关文档。
  -V, --version         显示程序版本号后退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或存在的`config.json`）。可以使用多个 --config 选项。也可以设置为 `-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定存放历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径。