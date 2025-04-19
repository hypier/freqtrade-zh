用法：freqtrade edge [-h] [-v] [--no-color] [--logfile FILE] [-V] [-c PATH]
                     [-d PATH] [--userdir PATH] [-s NAME]
                     [--strategy-path PATH] [--recursive-strategy-search]
                     [--freqaimodel NAME] [--freqaimodel-path PATH]
                     [-i TIMEFRAME] [--timerange TIMERANGE]
                     [--data-format-ohlcv {json,jsongz,feather,parquet}]
                     [--max-open-trades INT] [--stake-amount STAKE_AMOUNT]
                     [--fee FLOAT] [-p PAIRS [PAIRS ...]]
                     [--stoplosses STOPLOSS_RANGE]

选项：
  -h, --help            显示此帮助信息并退出
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间周期（`1m`、`5m`、`30m`、`1h`、`1d`）
  --timerange TIMERANGE
                        指定使用的数据时间范围
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载蜡烛数据（OHLCV）的存储格式。
                        （默认：`feather`）
  --max-open-trades INT
                        覆盖配置中的 `max_open_trades` 设置值
  --stake-amount STAKE_AMOUNT
                        覆盖配置中的 `stake_amount` 设置值
  --fee FLOAT           指定手续费比例。将在交易的开仓和平仓时应用两次
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        指定交易对，限制命令仅作用于这些交易对。交易对以空格分隔
  --stoplosses STOPLOSS_RANGE
                        定义止损值范围，策略将依据此范围评估。格式为
                        "min,max,step"（无空格）。示例：
                        `--stoplosses=-0.01,-0.1,-0.001`

常用参数：
  -v, --verbose         提升输出详细程度（-vv 获取更多信息，-vvv 获取所有信息）
  --no-color            禁用高亮显示结果，如果输出重定向到文件可能有用
  --logfile FILE, --log-file FILE
                        日志输出到指定文件。特殊值包括：
                        'syslog'、'journald'。详细信息请查阅文档
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json`或
                        `config.json`，以存在的文件为准）。可以使用多
                        个 --config 选项。也可以设置为 `-` 以从标准输入读取配置
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据目录路径
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径

策略参数：
  -s NAME, --strategy NAME
                        指定将由机器人使用的策略类名
  --strategy-path PATH  指定额外的策略搜索路径
  --recursive-strategy-search
                        在策略文件夹中递归搜索策略
  --freqaimodel NAME    指定自定义的 freqaimodel
  --freqaimodel-path PATH
                        指定 freqaimodel 的额外搜索路径
```