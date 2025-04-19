# 生产者 / 消费者 模式

freqtrade 提供了一种机制，允许一个实例（也称为 `consumer`）通过消息 WebSocket 监听来自上游 freqtrade 实例（也称为 `producer`）的消息。主要包括 `analyzed_df` 和 `whitelist` 消息。这使得在多个机器人中重复使用已计算的指标（和信号）成为可能，而无需多次计算。

详见 [Message Websocket](rest-api.md#message-websocket) in the Rest API 文档，了解如何设置用于你的消息 WebSocket 的 `api_server` 配置（这将作为你的 producer）。

!!! 注意
    强烈建议将 `ws_token` 设置为随机且只有你自己知道的值，以避免未授权访问你的机器人。

## 配置

通过在消费者的配置文件中添加 `external_message_consumer` 部分，启用订阅上游实例。

```json
{
    //...
   "external_message_consumer": {
        "enabled": true,
        "producers": [
            {
                "name": "default", // 你可以使用任何名称，默认是 "default"
                "host": "127.0.0.1", // 你的 producer 的 api_server 配置中的 host
                "port": 8080, // 你的 producer 的 api_server 配置中的端口
                "secure": false, // 使用安全的 websockets 连接，默认为 false
                "ws_token": "sercet_Ws_t0ken" // 你的 producer 配置中的 ws_token
            }
        ],
        // 以下配置为可选，通常不需要
        // "wait_timeout": 300,
        // "ping_timeout": 10,
        // "sleep_time": 10,
        // "remove_entry_exit_signals": false,
        // "message_size_limit": 8
    }
    //...
}
```

| 参数 | 描述 |
|-----|-------|
| `enabled` | **必填。** 启用消费者模式。若设置为 false，则此部分的所有其他设置将被忽略。<br>*默认值为 `false`。*<br> **数据类型：** boolean |
| `producers` | **必填。** 生产者列表<br> **数据类型：** 数组 |
| `producers.name` | **必填。** 该生产者的名称。在使用 `get_producer_pairs()` 和 `get_producer_df()` 时必须用到此名称（尤其在使用多个生产者时）。<br> **数据类型：** string |
| `producers.host` | **必填。** 你的生产者的主机名或 IP 地址。<br> **数据类型：** string |
| `producers.port` | **必填。** 与上述 host 配对使用的端口。<br>*默认值为 `8080`。*<br> **数据类型：** Integer |
| `producers.secure` | **可选。** 是否在 websockets 连接中使用 SSL。默认 false。<br> **数据类型：** boolean |
| `producers.ws_token` | **必填。** 生产者端配置的 `ws_token`。<br> **数据类型：** string |
| | **可选配置项** |
| `wait_timeout` | 如果未收到消息，则等待再次 ping 的超时时间。<br>*默认值为 `300`。*<br> **数据类型：**整数（秒） |
| `ping_timeout` | ping 超时时间<br>*默认值为 `10`。*<br> **数据类型：**整数（秒） |
| `sleep_time` | 重试连接前的等待时间。<br>*默认值为 `10`。*<br> **数据类型：**整数（秒） |
| `remove_entry_exit_signals` | 在接收数据框时，将信号列（signal columns）设为 0，从数据框中删除信号列。<br>*默认值为 `false`。*<br> **数据类型：** boolean |
| `message_size_limit` | 每条消息的大小限制<br>*默认值为 `8`（MB）。*<br> **数据类型：** integer（兆字节） |

在 `populate_indicators()` 中，除了（或代替）在本地计算指标外，还可以监听与生产者实例的连接，向生产者请求每对资产的最新分析数据框（`DataFrame`）。在高级配置中，还可以监听多个生产者实例。

消费者实例将拥有完整的分析数据框副本，无需自己计算。

## 示例

### 示例 - 生产者策略

一个包含多个指标的简单策略。策略本身不需要特殊考虑。

```py
class ProducerStrategy(IStrategy):
    #...
    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        按照标准 freqtrade 方法计算指标，然后广播给其他实例
        """
        dataframe['rsi'] = ta.RSI(dataframe)
        bollinger = qtpylib.bollinger_bands(qtpylib.typical_price(dataframe), window=20, stds=2)
        dataframe['bb_lowerband'] = bollinger['lower']
        dataframe['bb_middleband'] = bollinger['mid']
        dataframe['bb_upperband'] = bollinger['upper']
        dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        为给定 dataframe 填充入场信号
        """
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], self.buy_rsi.value)) &
                (dataframe['tema'] <= dataframe['bb_middleband']) &
                (dataframe['tema'] > dataframe['tema'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

!!! 提示 "FreqAI"
    你可以用这个在强大机器（如高配服务器）上设置 [FreqAI](freqai.md)，同时在简单的设备（如树莓派）上运行消费者，从生产者那里获取信号，并以不同方式解读。

### 示例 - 消费者策略

一个逻辑上等价的策略，不自行计算指标，而是基于生产者计算的指标，获得相同的分析数据框以进行交易决策。在此例中，消费者具有相同的入场条件，但这一点不是必须的。消费者也可以用不同的逻辑进行出入场操作，只使用指定的指标。

```py
class ConsumerStrategy(IStrategy):
    #...
    process_only_new_candles = False # 这是消费者必须的配置

    _columns_to_expect = ['rsi_default', 'tema_default', 'bb_middleband_default']

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        使用 websocket API 获取来自其他 freqtrade 实例的预填充指标。
        使用 `self.dp.get_producer_df(pair)` 获取数据框
        """
        pair = metadata['pair']
        timeframe = self.timeframe

        producer_pairs = self.dp.get_producer_pairs()
        # 你也可以通过以下方式指定特定生产者：
        # self.dp.get_producer_pairs("my_other_producer")

        # 返回分析过的数据框及其分析时间
        producer_dataframe, _ = self.dp.get_producer_df(pair)
        # 如果生产者提供了其他数据，也可以获取：
        # self.dp.get_producer_df(
        #   pair,
        #   timeframe="1h",
        #   candle_type=CandleType.SPOT,
        #   producer_name="my_other_producer"
        # )

        if not producer_dataframe.empty:
            # 如果你打算直接使用生产者的入场/退出信号，
            # 请指定 ffill=False，否则可能出现异常结果
            merged_dataframe = merge_informative_pair(dataframe, producer_dataframe,
                                                      timeframe, timeframe,
                                                      append_timeframe=False,
                                                      suffix="default")
            return merged_dataframe
        else:
            dataframe[self._columns_to_expect] = 0

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        填充给定数据框的入场信号
        """
        # 以自己计算的指标方式使用数据框列
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi_default'], self.buy_rsi.value)) &
                (dataframe['tema_default'] <= dataframe['bb_middleband_default']) &
                (dataframe['tema_default'] > dataframe['tema_default'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

!!! 提示 "使用上游信号"
    通过设置 `remove_entry_exit_signals=false`，你也可以直接使用生产者的信号。它们应作为 `enter_long_default`（假设使用了 `suffix="default"`）出现——可以作为信号直接使用，也可以作为附加指标。