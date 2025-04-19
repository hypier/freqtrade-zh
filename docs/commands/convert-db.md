用法：freqtrade convert-db [-h] [--db-url PATH] [--db-url-from PATH]

选项：
  -h, --help          显示此帮助信息并退出
  --db-url PATH       覆盖交易数据库的URL，这在自定义部署中很有用
                      （默认：Live运行模式下为 `sqlite:///tradesv3.sqlite`，
                       Dry Run模式下为 `sqlite:///tradesv3.dryrun.sqlite`）。
  --db-url-from PATH 迁移数据库时使用的数据源数据库URL。