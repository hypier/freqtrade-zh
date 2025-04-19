# 绘图

本页面将说明如何绘制价格、指标和利润。

!!! warning "已废弃"
    本页面介绍的命令（`plot-dataframe`、`plot-profit`）应视为已废弃，并处于维护状态。
    主要原因在于它们在性能方面存在问题，即使中等规模的图表也可能导致性能下降；另外，“存储文件后在浏览器中打开”从用户界面角度来说也不够直观。

    目前没有立即删除的计划，但它们不再积极维护——如果需要重大改变以确保其正常运行，可能会在短期内将其移除。
    
    建议使用 [FreqUI](freq-ui.md) 来满足绘图需求，后者不会遇到相同的性能问题。

## 安装 / 设置

绘图模块使用 Plotly 库。你可以通过运行以下命令进行安装或升级：

```bash
pip install -U -r requirements-plot.txt
```

## 绘制价格和指标

`freqtrade plot-dataframe` 子命令显示一个带有三个子图的交互式图表：

* 主要图表，显示蜡烛图和跟随价格的指标（如 sma/ema）
* 成交量柱状图
* 由 `--indicators2` 指定的其他指标

![plot-dataframe](assets/plot-dataframe.png)

可能的参数：

--8<-- "commands/plot-dataframe.md"

示例：

```bash
freqtrade plot-dataframe -p BTC/ETH --strategy AwesomeStrategy
```

`-p/--pairs` 参数可用来指定你希望绘制的交易对。

!!! note
    `freqtrade plot-dataframe` 子命令会为每个交易对生成一个图表文件。

可以自定义指标。  
使用 `--indicators1` 表示主图中的指标，使用 `--indicators2` 表示位于下方子图（如果值范围与价格不同）。

```bash
freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH --indicators1 sma ema --indicators2 macd
```

### 更多用例示例

绘制多个交易对，用空格分隔：

```bash
freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH XRP/ETH
```

缩放特定时间段（放大区间）：

```bash
freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH --timerange=20180801-20180805
```

如果你想绘制存储在数据库中的交易，可以结合 `--db-url` 和 `--trade-source DB` 使用：

```bash
freqtrade plot-dataframe --strategy AwesomeStrategy --db-url sqlite:///tradesv3.dry_run.sqlite -p BTC/ETH --trade-source DB
```

要绘制回测结果中的交易，用 `--export-filename <文件名>` 指定导出文件名：

```bash
freqtrade plot-dataframe --strategy AwesomeStrategy --export-filename user_data/backtest_results/backtest-result.json -p BTC/ETH
```

### 绘制数据框基础

![plot-dataframe2](assets/plot-dataframe2.png)

`plot-dataframe` 子命令需要提供回测数据、策略以及一个包含对应交易的回测结果文件或数据库。

生成的图表包括：

* 绿色三角：策略发出的买入信号。（注意：并非每个买入信号都会实际进行交易，参考青色圆圈。）
* 红色三角：策略发出的卖出信号。（同样，非每个卖出信号都终止一笔交易，参考红色和绿色方块。）
* 青色圆圈：交易入场点。
* 红色方块：亏损或利润为0%的交易退出点。
* 绿色方块：盈利交易的退出点。
* 以蜡烛图比例显示的指标（如 SMA/EMA），由 `--indicators1` 指定。
* 成交量（主图底部的柱状图）。
* 以不同尺度（如 MACD、RSI）显示的指标，位于成交量柱下方，由 `--indicators2` 指定。

!!! note "布林带"
    如果存在列 `bb_lowerband` 和 `bb_upperband`，则会自动在图中添加布林带，表现为从下轨到上轨的淡蓝色区域。

#### 高级图表配置

可以在策略中通过 `plot_config` 参数指定高级图表配置。

使用 `plot_config` 时的额外功能包括：

* 指定每个指标的颜色
* 指定额外的子图（子图）
* 指定指标对以填充中间区域

下面的示例配置为指标指定了固定颜色。否则，每次绘制可能会生成不同的配色方案，影响对比。
它还支持同时显示多个子图，例如同时显示 MACD 和 RSI。

图表类型可以通过 `type` 关键字配置。常用类型包括：

* `scatter` 对应 `plotly.graph_objects.Scatter`（默认）
* `bar` 对应 `plotly.graph_objects.Bar`

其他参数可在 `plotly` 字典中指定，用于传递给 `plotly.graph_objects.*` 的构造函数。

以下是带有内联注释的示例配置，说明过程：

```python
@property
def plot_config(self):
    """
        构建返回字典的方法有多种方案。
        唯一重要的是返回值。
        例子：
            plot_config = {'main_plot': {}, 'subplots': {}}
    """
    plot_config = {}
    # 主要图表的配置
    plot_config['main_plot'] = {
        # 主要指标的配置
        # 预设参数：emashort 和 emalong
        f'ema_{self.emashort.value}': {'color': 'red'},
        f'ema_{self.emalong.value}': {'color': '#CCCCCC'},
        # 省略 color 时会自动随机选取颜色。
        'sar': {},
        # 填充 senkou_a 和 senkou_b 之间的区域
        'senkou_a': {
            'color': 'green', # 可选
            'fill_to': 'senkou_b',
            'fill_label': 'Ichimoku Cloud', # 可选
            'fill_color': 'rgba(255,76,46,0.2)', # 可选
        },
        # 也绘制 senkou_b，不只填充区域
        'senkou_b': {}
    }
    # 子图配置
    plot_config['subplots'] = {
        # 创建 MACD 子图
        "MACD": {
            'macd': {'color': 'blue', 'fill_to': 'macdhist'},
            'macdsignal': {'color': 'orange'},
            'macdhist': {'type': 'bar', 'plotly': {'opacity': 0.9}}
        },
        # 额外的 RSI 子图
        "RSI": {
            'rsi': {'color': 'red'}
        }
    }

    return plot_config
```

??? note "作为属性（以前的方法）"
    也可以将 `plot_config` 作为属性（这是以前的默认方法）。
    但缺点是策略参数无法被访问，导致某些配置无法正常工作。

    ```python
        plot_config = {
            'main_plot': {
                # 主要指标的配置
                # 设定 `ema10` 为红色，`ema50` 为灰色
                'ema10': {'color': 'red'},
                'ema50': {'color': '#CCCCCC'},
                # 省略 color 时自动随机
                'sar': {},
            # 填充 senkou_a 和 senkou_b 之间的区域
            'senkou_a': {
                'color': 'green', #可选
                'fill_to': 'senkou_b',
                'fill_label': 'Ichimoku Cloud', #可选
                'fill_color': 'rgba(255,76,46,0.2)', #可选
            },
            # 也绘制 senkou_b，不只区间
            'senkou_b': {}
            },
            'subplots': {
                # 创建 MACD 子图
                "MACD": {
                    'macd': {'color': 'blue', 'fill_to': 'macdhist'},
                    'macdsignal': {'color': 'orange'},
                    'macdhist': {'type': 'bar', 'plotly': {'opacity': 0.9}}
                },
                # 额外的 RSI 子图
                "RSI": {
                    'rsi': {'color': 'red'}
                }
            }
        }
    ```

!!! note
    以上配置假设 `ema10`、`ema50`、`senkou_a`、`senkou_b`、
    `macd`、`macdsignal`、`macdhist` 和 `rsi` 都是策略生成的 DataFrame 列。

!!! warning
    `plotly` 参数仅支持使用 `plotly` 库时，不能在 `freq-ui` 中使用。

!!! note "交易仓位调整"
    如果启用 `position_adjustment_enable` / 调用 `adjust_trade_position()`，那么交易的起始买入价会在多个订单间进行平均，交易开始的价格可能会超出蜡烛图范围。

## 绘制利润

![plot-profit](assets/plot-profit.png)

`plot-profit` 子命令显示一个包含三个子图的交互式图表：

* 所有交易对的平均收盘价
* 回测获得的累计利润（注意：这不是实际利润，更像是估算值）
* 每个单独交易对的利润
* 交易的平行程度
* 最大回撤期间的资金变动（Underwater）

第一个图表帮助你了解整体市场的走势。

第二个图表用来验证你的算法是否有效。  
你可以希望算法稳定获利，或者少操作但收益大。  
此图还会显示最大回撤的起止时间。

第三个图表有助于识别异常值，即在某些交易对出现利润突增的事件。

第四个图表可以帮助分析交易的平行执行情况，显示最大同时开启的最大仓位数。

`freqtrade plot-profit` 子命令的可能参数：

--8<-- "commands/plot-profit.md"

`-p/--pairs` 参数可用来限制只考虑特定交易对。

示例：

使用自定义回测导出文件：

```bash
freqtrade plot-profit  -p LTC/BTC --export-filename user_data/backtest_results/backtest-result.json
```

使用自定义数据库：

```bash
freqtrade plot-profit  -p LTC/BTC --db-url sqlite:///tradesv3.sqlite --trade-source DB
```

```bash
freqtrade --datadir user_data/data/binance_save/ plot-profit -p LTC/BTC
```