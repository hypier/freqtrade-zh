用法：freqtrade [-h] [-V]
                 {trade,create-userdir,new-config,show-config,new-strategy,download-data,convert-data,convert-trade-data,trades-to-ohlcv,list-data,backtesting,backtesting-show,backtesting-analysis,edge,hyperopt,hyperopt-list,hyperopt-show,list-exchanges,list-markets,list-pairs,list-strategies,list-hyperoptloss,list-freqaimodels,list-timeframes,show-trades,test-pairlist,convert-db,install-ui,plot-dataframe,plot-profit,webserver,strategy-updater,lookahead-analysis,recursive-analysis}
                 ...

免费、开源的加密货币交易机器人

位置参数：
  {trade,create-userdir,new-config,show-config,new-strategy,download-data,convert-data,convert-trade-data,trades-to-ohlcv,list-data,backtesting,backtesting-show,backtesting-analysis,edge,hyperopt,hyperopt-list,hyperopt-show,list-exchanges,list-markets,list-pairs,list-strategies,list-hyperoptloss,list-freqaimodels,list-timeframes,show-trades,test-pairlist,convert-db,install-ui,plot-dataframe,plot-profit,webserver,strategy-updater,lookahead-analysis,recursive-analysis}
    trade               交易模块。
    create-userdir      创建用户数据目录。
    new-config          创建新配置。
    show-config         显示解析后的配置。
    new-strategy        创建新策略。
    download-data       下载回测数据。
    convert-data        将蜡烛图（OHLCV）数据转换为不同格式。
    convert-trade-data  将交易数据转换为不同格式。
    trades-to-ohlcv     将交易数据转换为OHLCV数据。
    list-data           列出已下载的数据。
    backtesting         回测模块。
    backtesting-show    显示过去的回测结果。
    backtesting-analysis
                        回测分析模块。
    edge                边际模块。
    hyperopt            超参数优化模块。
    hyperopt-list       列出超参数优化结果。
    hyperopt-show       显示超参数优化结果详情。
    list-exchanges      列出支持的交易所。
    list-markets        列出交易所中的市场。
    list-pairs          列出交易所中的交易对。
    list-strategies     列出可用策略。
    list-hyperoptloss 列出可用的超参数优化损失函数。
    list-freqaimodels   列出可用的 freqAI 模型。
    list-timeframes     列出交易所支持的时间框架。
    show-trades         展示交易信息。
    test-pairlist       测试你的交易对列表配置。
    convert-db          将数据库迁移到不同的系统。
    install-ui          安装 FreqUI。
    plot-dataframe      绘制带有指标的蜡烛图。
    plot-profit         生成利润示意图。
    webserver           Web服务器模块。
    strategy-updater    更新过时的策略文件到当前版本。
    lookahead-analysis  检查潜在的“前瞻偏差”问题。
    recursive-analysis  检查潜在的递归公式问题。

选项：
  -h, --help            显示帮助信息并退出
  -V, --version         显示程序版本号并退出