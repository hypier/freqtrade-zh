# Orderflow 数据

本指南将引导你如何利用公开交易数据进行高级订单流分析，适用于Freqtrade。

!!! Warning "实验性功能"
    订单流功能目前处于测试阶段，可能在未来的版本中进行调整。请在 [Freqtrade GitHub 仓库](https://github.com/freqtrade/freqtrade/issues)报告任何问题或反馈。
    当前尚未与 freqAI 结合测试，也不建议在两者同时使用，相关集成超出本指南范围。

!!! Warning "性能"
    订单流需要原始交易数据。这些数据体积较大，会导致初始启动变慢，因为Freqtrade需要下载最近X根K线的交易数据。此外，启用此功能会增加内存使用。请确保系统资源充足。

## 快速入门

### 启用公开交易
在你的 `config.json` 文件中，在 `exchange` 部分将 `use_public_trades` 选项设置为 `true`。

```json
"exchange": {
   ...
   "use_public_trades": true,
}
```

### 配置订单流处理
在 `config.json` 的订单流部分定义你需要的设置。此处可以调整诸如：

- `cache_size`：缓存中保存的前几根订单流蜡烛数，而非每次新蜡烛重新计算
- `max_candles`：筛选你希望获取交易数据的蜡烛数量
- `scale`：控制脚印图的价格桶大小
- `stacked_imbalance_range`：定义连续价格失衡的最小范围数
- `imbalance_volume`：过滤掉成交量低于此阈值的失衡
- `imbalance_ratio`：过滤掉比率（买卖盘差异）低于此值的失衡

```json
"orderflow": {
    "cache_size": 1000, 
    "max_candles": 1500, 
    "scale": 0.5, 
    "stacked_imbalance_range": 3, // 需要连续出现的失衡数量
    "imbalance_volume": 1, // 低于此成交量的失衡将被过滤
    "imbalance_ratio": 3 // 比率低于此值的失衡将被过滤
  },
```

## 下载历史交易数据用于回测

使用 `--dl-trades` 标志搭配 `freqtrade download-data` 命令即可下载历史交易数据。

```bash
freqtrade download-data -p BTC/USDT:USDT --timerange 20230101- --trading-mode futures --timeframes 5m --dl-trades
```

!!! Warning "数据可用性"
    并非所有交易所都提供公开交易数据。支持的交易所在你使用 `--dl-trades` 下载数据时，freqtrade会提示是否提供公开交易数据。

## 访问订单流数据

启用后，你的DataFrame会新增一些列，提供更丰富的订单流信息：

``` python
dataframe["trades"] # 包含每笔交易的详细信息
dataframe["orderflow"] # 表示脚印图的字典（见下文）
dataframe["imbalances"] # 关于订单流失衡的相关信息
dataframe["bid"] # 买盘总成交量
dataframe["ask"] # 卖盘总成交量
dataframe["delta"] # 卖盘与买盘的成交量差
dataframe["min_delta"] # 涡轮蜡烛内的最小成交量差
dataframe["max_delta"] # 涡轮蜡烛内的最大成交量差
dataframe["total_trades"] # 总交易笔数
dataframe["stacked_imbalances_bid"] # 堆叠买盘失衡区域的价格水平列表
dataframe["stacked_imbalances_ask"] # 堆叠卖盘失衡区域的价格水平列表
```

可以在策略代码中访问这些列以进行进一步分析。例如：

``` python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 计算累计成交量差
    dataframe["cum_delta"] = cumulative_delta(dataframe["delta"])
    # 获取总交易数
    total_trades = dataframe["total_trades"]
    ...

def cumulative_delta(delta: Series):
    cumdelta = delta.cumsum()
    return cumdelta
```

### 脚印图（`dataframe["orderflow"]`）

该列详细显示不同价格水平的买卖订单，提供订单流动态的宝贵洞察。配置中的 `scale` 参数决定了这个表示的价格桶大小。

`orderflow` 列为一个字典，其结构如下：

``` output
{
    "price": {
        "bid_amount": 0.0,
        "ask_amount": 0.0,
        "bid": 0,
        "ask": 0,
        "delta": 0.0,
        "total_volume": 0.0,
        "total_trades": 0
    }
}
```

#### 订单流列说明

- key：价格桶——按照 `scale` 间隔进行分桶
- `bid_amount`：该价格水平的买入总成交量
- `ask_amount`：该价格水平的卖出总成交量
- `bid`：该价格水平的买单数
- `ask`：该价格水平的卖单数
- `delta`：该价格水平的卖盘与买盘成交量差
- `total_volume`：该价格水平的总成交量（`ask_amount` + `bid_amount`）
- `total_trades`：该价格水平的总交易次数（`ask` + `bid`）

利用这些信息，你可以深入洞察市场情绪和潜在的交易机会，辅以订单流分析。

### 原始交易数据（`dataframe["trades"]`）

此列表记录了蜡烛期内发生的所有单笔交易，可用于更细粒度的订单流动态分析。

每个交易条目为一个字典，包含：

- `timestamp`：交易时间戳
- `date`：交易日期
- `price`：成交价格
- `amount`：成交量
- `side`：买或卖
- `id`：交易的唯一标识
- `cost`：交易总成本（`price` * `amount`）

### 失衡信息（`dataframe["imbalances"]`）

该列提供一个字典，描述订单流中的失衡情况。当某一价格水平的买卖盘成交量差异显著时，会出现失衡。

每行样式如下（以价格为索引，包含相应的买卖失衡值）：

``` output
{
    "price": {
        "bid_imbalance": False,
        "ask_imbalance": False
    }
}
```