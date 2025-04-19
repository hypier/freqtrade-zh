用法：freqtrade hyperopt-list [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                    [-c PATH] [-d PATH] [--userdir PATH] [--best]
                                    [--profitable] [--min-trades INT]
                                    [--max-trades INT] [--min-avg-time FLOAT]
                                    [--max-avg-time FLOAT] [--min-avg-profit FLOAT]
                                    [--max-avg-profit FLOAT]
                                    [--min-total-profit FLOAT]
                                    [--max-total-profit FLOAT]
                                    [--min-objective FLOAT] [--max-objective FLOAT]
                                    [--print-json] [--no-details]
                                    [--hyperopt-filename FILENAME]
                                    [--export-csv FILE]

参数选项：
  -h, --help            显示帮助信息并退出
  --best                只选择最佳轮次。
  --profitable          只选择盈利的轮次。
  --min-trades INT      选择交易次数多于 INT 次的轮次。
  --max-trades INT      选择交易次数少于 INT 次的轮次。
  --min-avg-time FLOAT  选择平均时间高于此值的轮次。
  --max-avg-time FLOAT  选择平均时间低于此值的轮次。
  --min-avg-profit FLOAT
                        选择平均利润高于此值的轮次。
  --max-avg-profit FLOAT
                        选择平均利润低于此值的轮次。
  --min-total-profit FLOAT
                        选择总利润高于此值的轮次。
  --max-total-profit FLOAT
                        选择总利润低于此值的轮次。
  --min-objective FLOAT
                        选择目标值高于此的轮次。
  --max-objective FLOAT
                        选择目标值低于此的轮次。
  --print-json          使用JSON格式输出。
  --no-details          不显示最佳轮次的详细信息。
  --hyperopt-filename FILENAME
                        超参数调优结果文件名。例如：`--hyperopt-filename=hyperopt_results_2020-09-27_16-20-48.pickle`
  --export-csv FILE     导出为CSV文件。此操作将禁用表格打印。例子：--export-csv hyperopt.csv

常用参数：
  -v, --verbose         详细模式（-vv显示更多信息，-vvv显示全部信息）。
  --no-color            禁用超参数调优结果的彩色显示。如果你将输出重定向到文件，这可能会有用。
  --logfile FILE, --log-file FILE
                        指定日志文件路径。常用值有：
                        'syslog', 'journald'。详见相关文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或
                        `config.json`，取决于存在哪个文件）。可以使用多个--config。
                        可以设置为`-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        指向存放历史回测数据的目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。