用法：freqtrade new-strategy [-h] [--userdir PATH] [-s NAME]
                              [--strategy-path PATH]
                              [--template {full,minimal,advanced}]

选项：
  -h, --help            显示帮助信息并退出
  --userdir PATH, --user-data-dir PATH
                        用户数据目录路径。
  -s NAME, --strategy NAME
                        指定策略类名称，机器人将使用该策略。
  --strategy-path PATH  指定额外的策略查找路径。
  --template {full,minimal,advanced}
                        使用模板，可选 `minimal`（极简）、`full`（完整，包含多个示例指标）或 `advanced`（高级）。
                        默认为：`full`。