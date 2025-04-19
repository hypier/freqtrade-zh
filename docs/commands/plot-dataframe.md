用法：freqtrade plot-dataframe [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                [-c PATH] [-d PATH] [--userdir PATH] [-s NAME]
                                [--strategy-path PATH]
                                [--recursive-strategy-search]
                                [--freqaimodel NAME] [--freqaimodel-path PATH]
                                [-p PAIRS [PAIRS ...]]
                                [--indicators1 INDICATORS1 [INDICATORS1 ...]]
                                [--indicators2 INDICATORS2 [INDICATORS2 ...]]
                                [--plot-limit INT] [--db-url PATH]
                                [--trade-source {DB,file}]
                                [--export {none,trades,signals}]
                                [--export-filename PATH]
                                [--timerange TIMERANGE] [-i TIMEFRAME]
                                [--no-trades]

参数选项：
  -h, --help            显示帮助信息并退出
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        将命令限制在这些交易对。交易对用空格分隔。
  --indicators1 INDICATORS1 [INDICATORS1 ...]
                        设置策略中希望在第一行显示的指标。用空格分隔的列表。示例：
                        `ema3 ema5`。默认值：`['sma', 'ema3', 'ema5']`。
  --indicators2 INDICATORS2 [INDICATORS2 ...]
                        设置策略中希望在第三行显示的指标。用空格分隔的列表。示例：
                        `fastd fastk`。默认值：`['macd', 'macdsignal']`。
  --plot-limit INT      指定绘图的最大刻度数。注意：数值过大会生成巨大的文件。默认值：750。
  --db-url PATH         覆盖交易数据库的URL。这在自定义部署中非常有用（默认：`sqlite:///tradesv3.sqlite`为实时运行模式，`sqlite:///tradesv3.dryrun.sqlite`为模拟交易）。
  --trade-source {DB,file}
                        指定交易数据来源（可以是数据库或文件（回测文件））。默认：file
  --export {none,trades,signals}
                        导出回测结果（默认：trades）。
  --export-filename PATH, --backtest-filename PATH
                        使用此文件名保存回测结果。需同时设置`--export`。示例：`--export-filename=user_data/backtest_results/backtest_today.json`
  --timerange TIMERANGE
                        指定使用的数据时间范围。
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间框架（`1m`、`5m`、`30m`、`1h`、`1d`）。
  --no-trades           跳过使用回测文件和数据库中的交易数据。

常用参数：
  -v, --verbose         显示详细信息（-vv获得更多信息，-vvv显示所有信息）。
  --no-color            禁用高亮结果的颜色化。如果将输出重定向到文件，可能会很有用。
  --logfile FILE, --log-file FILE
                        将日志写入指定文件。特殊值包括：
                        'syslog'、'journald'。详情请参考文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或存在的`config.json`）。可以使用多个--config选项。也可以设置为`-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指定包含历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        指定用户数据目录路径。

策略相关参数：
  -s NAME, --strategy NAME
                        指定将由机器人使用的策略类名。
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        在策略文件夹中递归搜索策略。
  --freqaimodel NAME    指定自定义的频率模型。
  --freqaimodel-path PATH
                        指定额外的频率模型查找路径。