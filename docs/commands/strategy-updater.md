用法：freqtrade strategy-updater [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                  [-c PATH] [-d PATH] [--userdir PATH]
                                  [--strategy-list STRATEGY_LIST [STRATEGY_LIST ...]]
                                  [--strategy-path PATH]
                                  [--recursive-strategy-search]

选项：
  -h, --help            显示帮助信息并退出
  --strategy-list STRATEGY_LIST [STRATEGY_LIST ...]
                        提供以空格分隔的策略列表进行回测。请注意，时间框架需要在配置中设置或通过命令行指定。当与 `--export trades` 同时使用时，策略名称会被注入到文件名中，例如 `backtest-data.json` 会变成 `backtest-data-SampleStrategy.json`
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        递归搜索策略文件夹中的策略。

常用参数：
  -v, --verbose         详细模式 (-vv 表示更详细，-vvv 获取所有信息)。
  --no-color            禁用 hyperopt 结果的颜色显示。如果你将输出重定向到文件，这个选项可能会有用。
  --logfile FILE, --log-file FILE
                        将日志写入指定文件。特殊值包括：
                        'syslog'、'journald'。详见相关文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认： `userdir/config.json` 或存在的 `config.json`）。
                        可以使用多个 --config 选项。也可设置为 `-`，从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据所在目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。