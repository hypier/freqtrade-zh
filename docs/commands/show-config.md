用法: freqtrade show-config [-h] [--userdir PATH] [-c PATH]
                                 [--show-sensitive]

选项：
  -h, --help            显示此帮助信息并退出
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。
  -c PATH, --config PATH
                        指定配置文件（默认：
                        `userdir/config.json` 或 `config.json`，以存在的为准）。
                        可以使用多个 --config 选项。也可以设为 `-` 从标准输入读取配置。
  --show-sensitive      在输出中显示敏感信息。