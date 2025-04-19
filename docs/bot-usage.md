# 启动机器人

本页说明机器人的不同参数以及如何运行它。

!!! 注意
    如果你已经使用了 `setup.sh`，在执行 `freqtrade` 命令之前，请不要忘了激活你的虚拟环境（`source .venv/bin/activate`）。

!!! 警告 “保持时钟同步”
    运行机器人的系统时间必须准确无误，并经常同步到 NTP 服务器，以避免与交易所通信时出现问题。

## 机器人命令

--8<-- "commands/main.md"

### 机器人交易相关命令

--8<-- "commands/trade.md"

### 如何指定使用哪个配置文件？

机器人允许你通过 `-c/--config` 命令行参数选择要使用的配置文件：

```bash
freqtrade trade -c path/far/far/away/config.json
```

默认情况下，机器人会从当前工作目录加载 `config.json` 配置文件。

### 如何使用多个配置文件？

机器人支持通过在命令行中指定多个 `-c/--config` 选项来使用多个配置文件。后面配置文件中定义的参数会覆盖前面配置文件中同名的参数。

例如，你可以为你用来交易的交易所创建一个单独的配置文件，包含你的 API Key 和 Secret，运行在 Dry Mode（不实际执行交易时）时，指定一个空的 Key 和 Secret 作为默认配置文件：

```bash
freqtrade trade -c ./config.json
```

在正常的实盘交易模式下，可以同时指定多个配置文件：

```bash
freqtrade trade -c ./config.json -c path/to/secrets/keys.config.json
```

这样可以通过为包含真实私密信息的文件设置合适的文件权限，保护你的私有交易所密钥和密钥秘密，避免在项目问题反馈或互联网发布配置示例时泄露敏感数据。

关于此技术的更多细节和示例，可以参考文档页面上的[配置](configuration.md)。

### 自定义数据存放位置

Freqtrade 允许你使用命令 `freqtrade create-userdir --userdir someDirectory` 创建用户数据目录。该目录结构如下：

```
user_data/
├── backtest_results
├── data
├── hyperopts
├── hyperopt_results
├── plot
└── strategies
```

你可以在配置中添加 `"user_data_dir"` ，让机器人始终指向这个目录。
或者在每个命令中传入 `--userdir` 参数。
如果目录不存在，机器人启动时会失败，但会自动创建必要的子目录。

该目录应包含你的自定义策略、自定义hyperopt和hyperopt损失函数、回测的历史数据（通过回测命令或下载脚本获取）以及绘图输出。

建议使用版本控制工具追踪策略的变更。

### 如何使用 **--strategy**？

此参数允许你加载自定义策略类。  
要测试机器人安装是否成功，可以使用由 `create-userdir` 子命令（通常为 `user_data/strategy/sample_strategy.py`）安装的 `SampleStrategy`。

机器人会在 `user_data/strategies` 目录下搜索你的策略文件。  
如果需要从其他目录加载策略，请阅读下一节关于 `--strategy-path` 的内容。

要加载某个策略，只需在此参数中传入策略类名（例如：`CustomStrategy`）。

**示例：**  
如果在 `user_data/strategies` 目录下有一个文件 `my_awesome_strategy.py`，其中定义了 `AwesomeStrategy` 类，要加载它：

```bash
freqtrade trade --strategy AwesomeStrategy
```

如果机器人找不到你的策略文件，会在错误信息中显示原因（文件未找到或代码中有错误）。

更多关于策略文件的内容，请参考
[策略定制](strategy-customization.md)。

### 如何使用 **--strategy-path**？

此参数允许你添加额外的策略搜索路径，优先于默认位置进行查找（传入的路径必须是目录！）：

```bash
freqtrade trade --strategy AwesomeStrategy --strategy-path /some/directory
```

#### 如何安装策略？

非常简单。将你的策略文件复制到 `user_data/strategies` 目录，或使用 `--strategy-path` 参数指定路径。  
然后，机器人即可使用该策略。

### 如何使用 **--db-url**？

在 Dry-run（模拟运行）模式下，默认不会将交易信息存储到数据库中。  
如果你希望将机器人操作存入数据库，可以使用 `--db-url` 参数。如果在生产环境中想指定自定义数据库，也可以使用此参数。例如：

```bash
freqtrade trade -c config.json --db-url sqlite:///tradesv3.dry_run.sqlite
```

## 下一步

机器人的最优策略会随着市场趋势的变化而调整。下一步建议参考
[策略定制](strategy-customization.md)。