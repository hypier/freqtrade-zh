用法: freqtrade trade [-h] [-v] [--no-color] [--logfile FILE] [-V] [-c PATH]
                          [-d PATH] [--userdir PATH] [-s NAME]
                          [--strategy-path PATH] [--recursive-strategy-search]
                          [--freqaimodel NAME] [--freqaimodel-path PATH]
                          [--db-url PATH] [--sd-notify] [--dry-run]
                          [--dry-run-wallet DRY_RUN_WALLET] [--fee FLOAT]

选项：
  -h, --help            显示此帮助信息并退出
  --db-url PATH         覆盖交易数据库的URL，这在自定义部署中非常有用
                        （默认：`sqlite:///tradesv3.sqlite` 用于实盘运行，
                        `sqlite:///tradesv3.dryrun.sqlite` 用于模拟交易）。
  --sd-notify           通知systemd服务管理器。
  --dry-run             强制进行模拟交易（移除交易所密钥并模拟交易）。
  --dry-run-wallet DRY_RUN_WALLET, --starting-balance DRY_RUN_WALLET
                        起始余额，用于回测 / 超参数优化和模拟交易。
  --fee FLOAT           指定手续费比例。将在买入和卖出时应用两次。

常用参数：
  -v, --verbose         启用详细模式（-vv 获取更多信息，-vvv 获取所有信息）。
  --no-color            禁用超参数优化结果的颜色显示。如将输出重定向至文件，可能会用到。
  --logfile FILE, --log-file FILE
                        指定日志文件。特殊值包括：
                        'syslog'、'journald'。详细信息请参见文档。
  -V, --version         显示程序版本信息并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json` 或
                        `config.json`，取决于哪个存在）。可以使用多个
                        --config 选项。也可设为 `-` 从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        历史回测数据所在目录路径。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。

策略参数：
  -s NAME, --strategy NAME
                        指定将被机器人使用的策略类名称。
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        在策略文件夹中递归搜索策略。
  --freqaimodel NAME    指定自定义的 freqai 模型。
  --freqaimodel-path PATH
                        指定额外的 freqaimodel 查找路径。