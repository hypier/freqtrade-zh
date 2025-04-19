# 配置

FreqAI的配置方式采用典型的[Freqtrade配置文件](configuration.md)和标准的[Freqtrade策略](strategy-customization.md)。示例的FreqAI配置和策略文件可以在`config_examples/config_freqai.example.json`和`freqtrade/templates/FreqaiExampleStrategy.py`中找到。

## 配置文件的搭建

虽然还有许多额外参数可以选择，正如[参数表](freqai-parameter-table.md#parameter-table)所强调的，一个FreqAI配置至少必须包含以下参数（参数值仅为示例）：

```json
    "freqai": {
        "enabled": true,
        "purge_old_models": 2,
        "train_period_days": 30,
        "backtest_period_days": 7,
        "identifier" : "unique-id",
        "feature_parameters" : {
            "include_timeframes": ["5m","15m","4h"],
            "include_corr_pairlist": [
                "ETH/USD",
                "LINK/USD",
                "BNB/USD"
            ],
            "label_period_candles": 24,
            "include_shifted_candles": 2,
            "indicator_periods_candles": [10, 20]
        },
        "data_split_parameters" : {
            "test_size": 0.25
        }
    }
```

完整配置示例请参见`config_examples/config_freqai.example.json`。

!!! 注意
    `identifier`参数常被新手忽略，但该值在你的配置中起着重要作用。这个值是你用来描述某次运行的唯一标识ID。保持其不变，可以增加系统的抗崩溃能力，并加快回测速度。每当尝试新的运行（加入新特性、新模型等）时，应修改此值（或删除`user_data/models/唯一ID`文件夹）。更多细节请参考[参数表](freqai-parameter-table.md#feature-parameters)。

## 构建FreqAI策略

FreqAI策略需要在标准的[Freqtrade策略](strategy-customization.md)中加入以下代码片段：

```python
    # 用户应定义最大初始化蜡烛数量（传递给任何单一指标的最大蜡烛数）
    startup_candle_count: int = 20

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        # model将返回`set_freqai_targets()`中创建的所有标签(& appended targets)，
        # 以及是否接受预测的指示，和每个训练周期中用户在
        # `set_freqai_targets()`中创建标签的平均值/标准值。

        dataframe = self.freqai.start(dataframe, metadata, self)

        return dataframe

    def feature_engineering_expand_all(self, dataframe: DataFrame, period, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI的策略中有效*
        该函数会自动扩展配置中定义的特征，包括
        `indicator_periods_candles`、`include_timeframes`、`include_shifted_candles`和
        `include_corr_pairs`。也就是说，定义在此函数中的单一特征会自动扩展为
        `indicator_periods_candles` * `include_timeframes` * `include_shifted_candles` *
        `include_corr_pairs`个特征，加入模型。

        所有特征必须以`%`开头，内核才能识别。

        :param df: 要添加特征的策略数据框
        :param period: 指标的周期 - 示例用法：
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)
        """

        dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
        dataframe["%-mfi-period"] = ta.MFI(dataframe, timeperiod=period)
        dataframe["%-adx-period"] = ta.ADX(dataframe, timeperiod=period)
        dataframe["%-sma-period"] = ta.SMA(dataframe, timeperiod=period)
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)

        return dataframe

    def feature_engineering_expand_basic(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI的策略中有效*
        该函数会自动扩展配置中定义的特征，包括
        `include_timeframes`、`include_shifted_candles`和`include_corr_pairs`。
        也就是说，定义在此函数中的单一特征会自动扩展为
        `include_timeframes` * `include_shifted_candles` * `include_corr_pairs`个特征，加入模型。

        这里定义的特征不会自动在`indicator_periods_candles`中复制。

        所有特征必须以`%`开头，内核才能识别。

        :param df: 要添加特征的策略数据框
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        """
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        return dataframe

    def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI的策略中有效*
        该可选函数在基础时间周期的数据框上调用一次。
        这是最后一层调用的函数，意味着传入的dataframe将包含所有其他
        `freqai_feature_engineering_*`函数创建的特征和列。

        这是进行定制特殊特征提取（如tsfresh）或不自动扩展的特征（如星期几）的理想地点。

        所有特征必须以`%`开头，内核才能识别。

        :param df: 方案数据框，接收特征
        使用示例：dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        """
        dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        dataframe["%-hour_of_day"] = (dataframe["date"].dt.hour + 1) / 25
        return dataframe

    def set_freqai_targets(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI的策略中有效*
        设置模型目标的必备函数。
        所有目标必须以`&`开头，内核才能识别。

        :param df: 目标数据框
        使用示例：dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        dataframe["&-s_close"] = (
            dataframe["close"]
            .shift(-self.freqai_info["feature_parameters"]["label_period_candles"])
            .rolling(self.freqai_info["feature_parameters"]["label_period_candles"])
            .mean()
            / dataframe["close"]
            - 1
            )
        return dataframe
```

注意，`feature_engineering_*()`函数负责[特征工程](freqai-feature-engineering.md#feature-engineering)，而`set_freqai_targets()`负责添加标签/目标。完整示例策略可以在`templates/FreqaiExampleStrategy.py`中找到。

!!! 注意
    `self.freqai.start()`函数不能在`populate_indicators()`之外调用。

!!! 注意
    只有在`feature_engineering_*()`中定义的特征才会被FreqAI识别。将FreqAI特征写在`populate_indicators()`中会导致模型在实盘或模拟中的运行失败。若要增加与特定币对或时间周期无关的通用特征，建议使用`feature_engineering_standard()`（正如在`freqtrade/templates/FreqaiExampleStrategy.py`中的示范所示）。

## 常用数据框关键模式

以下是在策略数据框（`df[]`）中常见的关键字和值：

|  DataFrame Key | 描述 |
|------------|-------------|
| `df['&*']` | 在`set_freqai_targets()`中以`&`开头的列被视为FreqAI中的训练目标（标签），通常命名为`&-s*`。例如，为预测未来40根蜡烛收盘价，可设：
```python
df['&-s_close'] = df['close'].shift(-40)
```
如果在配置中将`label_period_candles`设置为40，则模型会做出预测，并返回到`df['&-s_close']`，用于在`populate_entry/exit_trend()`中调用。 <br> **数据类型：** 取决于模型输出。|
| `df['&*_std/mean']` | 在训练（或实时监控`fit_live_predictions_candles`）中定义的标签的标准差和平均值。常用于评估预测的稀有度（例如用z-score，详见[此](#creating-a-dynamic-target-threshold)和[模板](freqai-feature-engineering.md#returning-additional-info-from-training)中的用法）。<br> **数据类型：** 浮点数。|
| `df['do_predict']` | 异常点指示器。返回值为-2到2之间的整数，表明预测是否可信。`do_predict==1` 表示可信。若输入点的Dissimilarity Index（DI，详见[此](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)和[详细](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)）超过配置阈值，FreqAI会减1，变为`do_predict==0`。若启用`use_SVM_to_remove_outliers`，支持向量机（SVM，详见[此](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)）也会检测出异常点，并减1。如果SVM和DI都判定为异常，则结果为`do_predict==-1`。若`use_DBSCAN_to_remove_outliers`启用，DBSCAN（详见[此](freqai-feature-engineering.md#identifying-outliers-with-dbscan)）也会检测离群点，减1。若SVM和DBSCAN均检测到某点为离群点，结果为`do_predict==-2`。特别地，若`do_predict==2`，表示模型因超出`expired_hours`而过期。<br> **数据类型：** -2到2的整数。|
| `df['DI_values']` | Dissimilarity Index（DI）值，代表FreqAI对预测的置信度。DI越低，表明预测接近训练数据，置信度越高。详见[此](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)。<br> **数据类型：** 浮点数。|
| `df['%*']` | 在`feature_engineering_*()`中以`%`开头的列为训练特征。例如，添加RSI到特征集中，可设：
```python
df['%-rsi']
```
详见[此](freqai-feature-engineering.md)。注意，由于以`%`开头的特征数在大量组合时可能迅速膨胀（数万甚至更多特征，使用`include_shifted_candles`和`include_timeframes`等多重组合），FreqAI会自动在返回的数据框中剔除这些以`%`开头的特征（除非你特殊定义保留它们，详见下方`%%`）。若希望保留某个特征用于绘图，可在其前加`%%`（见下文详细说明）。<br> **数据类型：** 用户定义的特征类型。|
| `df['%%*']` | 在`feature_engineering_*()`中以`%%`开头的列也视为训练特征，但这些特征会返回到策略数据框，供FreqUI/绘图/监控（模拟/实盘/回测）使用。<br> **数据类型：** 用户定义的特征类型。注意，`feature_engineering_expand()`中定义的特征，将会根据你配置的扩展（如`include_timeframes`、`include_corr_pairlist`、`indicators_periods_candles`、`include_shifted_candles`）自动生成命名规范。例如，如果你在`feature_engineering_expand_all()`中定义了`%%-rsi`，最终图形配置中的命名可能会是：`%%-rsi-period_10_ETH/USDT:USDT_1h`，代表周期10、时间框架1小时、币对ETH/USDT：USDT（若使用期货合约，可能会带`:USDT`后缀）。建议在`populate_indicators()`中调用`print(dataframe.columns)`，以查看所有实际存入策略的数据框特征的完整列表。|

## 设置`startup_candle_count`

在FreqAI策略中，`startup_candle_count`需要像标准Freqtrade策略一样设定（详见[此](strategy-customization.md#strategy-startup-period)）。该值用于确保调用`dataprovider`时提供足够数据，避免首次训练出现NaN。建议选取传入指标创建函数（如TA-Lib函数）的最大蜡烛周期值。如示例中，`startup_candle_count`设为20，因为这是`indicators_periods_candles`中的最大值。

!!! 注意
    有些TA-Lib函数实际上需要比传入的`period`更多的数据，否则特征会被NaN填充。经验总结显示，将`startup_candle_count`乘以2，能确保训练数据完全无NaN。请留意如下日志确认：
    
    ```
    2022-08-31 15:14:04 - freqtrade.freqai.data_kitchen - INFO - dropped 0 training points due to NaNs in populated dataset 4319.
    ```

## 创建动态目标阈值

可以以动态方式，根据市场状况调整入场或离场策略。FreqAI允许在模型训练中返回额外信息（详细[此](freqai-feature-engineering.md#returning-additional-info-from-training)），例如`&*_std/mean`等值，描述最近一次训练中标签（目标）的统计分布。用这些值与模型预测进行比较，可以理解预测的稀有程度（如用z-score，详见[此](#creating-a-dynamic-target-threshold)和示例中）。比如在`FreqaiExampleStrategy.py`中，设置目标ROI和卖出ROI如下，可以过滤掉靠近均值的预测：

```python
dataframe["target_roi"] = dataframe["&-s_close_mean"] + dataframe["&-s_close_std"] * 1.25
dataframe["sell_roi"] = dataframe["&-s_close_mean"] - dataframe["&-s_close_std"] * 1.25
```

若希望根据**历史预测的总体**数据生成动态目标，而非只依赖训练统计信息，可以在配置中设置`fit_live_predictions_candles`为所需的历史预测蜡烛数量：

```json
    "freqai": {
        "fit_live_predictions_candles": 300,
    }
```
启用后，FreqAI开始使用训练数据中的预测值，逐步引入真实预测数据。模型关闭后再重启，频AI会加载保存的历史预测，并存储以备下次使用。

## 使用不同的预测模型

FreqAI提供多个示例预测模型库，支持用`--freqaimodel`参数调用。包括`CatBoost`、`LightGBM`和`XGBoost`的回归、分类和多目标模型，位于`freqai/prediction_models/`中。

回归和分类模型的区别在于预测目标不同：回归模型输出连续值（比如下一天的比特币价格），分类模型输出离散类别（比如涨或跌）。因此，设置目标时必须对应选择（详见[此](#setting-model-targets)）。

全部模型都基于梯度提升决策树算法，采用集成学习思想，将多个基础学习器（决策树）结合以获得更稳定、更通用的预测。模型通过序列训练逐步改进误差。

详细信息请浏览：

* CatBoost：https://catboost.ai/en/docs/
* LightGBM：https://lightgbm.readthedocs.io/en/v3.3.2/
* XGBoost：https://xgboost.readthedocs.io/en/stable/

此外，还有许多相关文章对比和分析这些算法，比如：
- [CatBoost vs. LightGBM vs. XGBoost — 哪个算法更优？](https://towardsdatascience.com/catboost-vs-lightgbm-vs-xgboost-c80f40662924#:~:text=In%20CatBoost%2C%20symmetric%20trees%2C%20or,the%20same%20depth%20can%20differ.)
- [XGBoost、LightGBM还是CatBoost——我该用哪个？](https://medium.com/riskified-technology/xgboost-lightgbm-or-catboost-which-boosting-algorithm-should-i-use-e7fda7bb36bc)

请注意，不同模型的性能严重依赖具体应用，报告的指标未必适用于你的场景。

除现有模型外，你还可以通过继承`IFreqaiModel`类，自定义创建模型。在`user_data/freqaimodels`目录下存放你的自定义模型，`--freqaimodel`参数对应的应该是模型的类名，注意取唯一名称避免覆盖。

### 设置模型目标

#### 回归模型（regressors）

若用回归模型，目标应为连续值。例如，预测未来100根蜡烛的收盘价，可设：

```python
df['&s-close_price'] = df['close'].shift(-100)
```

要预测多个目标，只需定义多个标签（值习惯保持一致，详见示例）。

#### 分类模型（classifiers）

若用分类模型，目标应为类别标签（离散值）。例如，预测未来100根蜡烛涨跌，可设：

```python
df['&s-up_or_down'] = np.where(df["close"].shift(-100) > df["close"], 'up', 'down')
```

若要预测多类别或多目标，可以在单一标签列中并列多个类别，比如若还想标注价格未变的情况，可加入'same'类别：

```python
df['&s-up_or_down'] = np.where(df["close"].shift(-100) > df["close"], 'up', 'down')
df['&s-up_or_down'] = np.where(df["close"].shift(-100) == df["close"], 'same', df['&s-up_or_down'])
```

## PyTorch模块

### 快速入门

最简单地运行PyTorch模型（回归任务）示例命令：

```bash
freqtrade trade --config config_examples/config_freqai.example.json --strategy FreqaiExampleStrategy --freqaimodel PyTorchMLPRegressor --strategy-path freqtrade/templates 
```

!!! 注意 "安装/docker"
    PyTorch模块需要较大包（如`torch`），在`./setup.sh -i`时答“y”显式安装，答“y”以安装`freqai-rl`或`PyTorch`的依赖（大约多需700MB空间）。喜欢用Docker的用户，应选择带 `_freqaitorch`后缀的镜像版本。相关`docker-compose`文件在`docker/docker-compose-freqai.yml`，可通过
`docker compose -f docker/docker-compose-freqai.yml run ...`运行，或者复制替代原Docker配置。此文件还提供禁用/启用GPU资源的配置（系统须配有GPU支持）。

注意，PyTorch自版本2.3起不再支持macOS x64（苹果Intel设备），Freqtrade也因此停止支持。

### 结构

#### 模型定义

可在自定义的`[IFreqaiModel](#using-different-prediction-models)`文件中定义`nn.Module`模型结构，之后在`train()`函数中调用。例如，以下为用PyTorch实现的逻辑回归模型（需配合`nn.BCELoss`）示例。

```python
class LogisticRegression(nn.Module):
    def __init__(self, input_size: int):
        super().__init__()
        # 定义网络层
        self.linear = nn.Linear(input_size, 1)
        self.activation = nn.Sigmoid()

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # 前向传播
        out = self.linear(x)
        out = self.activation(out)
        return out

class MyCoolPyTorchClassifier(BasePyTorchClassifier):
    """
    这是一个自定义的IFreqaiModel，展示用户如何自定义
    神经网络结构进行训练。
    """

    @property
    def data_convertor(self) -> PyTorchDataConvertor:
        return DefaultPyTorchDataConvertor(target_tensor_type=torch.float)

    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        config = self.freqai_info.get("model_training_parameters", {})
        self.learning_rate: float = config.get("learning_rate",  3e-4)
        self.model_kwargs: dict[str, Any] = config.get("model_kwargs",  {})
        self.trainer_kwargs: dict[str, Any] = config.get("trainer_kwargs",  {})

    def fit(self, data_dictionary: dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        """
        用户在此处设置训练和测试数据，定义模型。
        :param data_dictionary: 包含所有训练、测试、标签、权重等数据的字典
        :param dk: 当前币种/模型的DataKitchen对象
        """

        class_names = self.get_class_names()
        self.convert_label_column_to_int(data_dictionary, dk, class_names)
        n_features = data_dictionary["train_features"].shape[-1]
        model = LogisticRegression(
            input_size=n_features
        )
        model.to(self.device)
        optimizer = torch.optim.AdamW(model.parameters(), lr=self.learning_rate)
        criterion = torch.nn.CrossEntropyLoss()
        init_model = self.get_init_model(dk.pair)
        trainer = PyTorchModelTrainer(
            model=model,
            optimizer=optimizer,
            criterion=criterion,
            model_meta_data={"class_names": class_names},
            device=self.device,
            init_model=init_model,
            data_convertor=self.data_convertor,
            **self.trainer_kwargs,
        )
        trainer.fit(data_dictionary, self.splits)
        return trainer
```

#### 训练器（Trainer）

`PyTorchModelTrainer`实现了典型的PyTorch训练循环。定义模型、损失函数、优化器后，将模型转到GPU或CPU，循环中逐批次读取数据、预测、计算损失、反向传播、参数更新。

此外，它还负责：
- 保存和加载模型
- 将`pandas.DataFrame`转换成`torch.Tensor`

#### 与Freqai模块的集成

所有freqai模型都继承`IFreqaiModel`，后者定义了`train`、`fit`和`predict`三个抽象方法。实现时分三个层级：

1. `BasePyTorchModel`——实现`train()`，所有`BasePyTorch*`类都继承它，负责数据预处理（如标准化）和调用`fit()`，并定义`device`和`model_type`属性。
2. `BasePyTorch*`（*指算法类别，如分类器或回归器）——实现`predict()`，负责数据预处理、预测和后处理。
3. `PyTorch*Classifier` / `PyTorch*Regressor`——实现`fit()`，进行训练流程，包括初始化trainer和模型。

![模型架构图](assets/freqai_pytorch-diagram.png)

### 完整示例

用MLP（多层感知机）+MSELoss、AdamW训练器构建PyTorch回归模型示例：

```python
class PyTorchMLPRegressor(BasePyTorchRegressor):
    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        config = self.freqai_info.get("model_training_parameters", {})
        self.learning_rate: float = config.get("learning_rate",  3e-4)
        self.model_kwargs: dict[str, Any] = config.get("model_kwargs",  {})
        self.trainer_kwargs: dict[str, Any] = config.get("trainer_kwargs",  {})

    def fit(self, data_dictionary: dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        n_features = data_dictionary["train_features"].shape[-1]
        model = PyTorchMLPModel(
            input_dim=n_features,
            output_dim=1,
            **self.model_kwargs
        )
        model.to(self.device)
        optimizer = torch.optim.AdamW(model.parameters(), lr=self.learning_rate)
        criterion = torch.nn.MSELoss()
        init_model = self.get_init_model(dk.pair)
        trainer = PyTorchModelTrainer(
            model=model,
            optimizer=optimizer,
            criterion=criterion,
            device=self.device,
            init_model=init_model,
            target_tensor_type=torch.float,
            **self.trainer_kwargs,
        )
        trainer.fit(data_dictionary)
        return trainer
```

这段代码定义了`PyTorchMLPRegressor`类，实现了`fit()`方法，指定模型、优化器、损失函数和训练器，它继承自`BasePyTorchRegressor`（定义预测逻辑）和`BasePyTorchModel`（定义训练流程）。

??? 提示 "如何为分类器设置类别名"
当使用分类器时，用户必须在`set_freqai_targets()`中通过`self.freqai.class_names`声明类别名。例如，用二分类预测涨跌：

```python
def set_freqai_targets(self, dataframe: DataFrame, metadata: dict, **kwargs) -> DataFrame:
    self.freqai.class_names = ["down", "up"]
    dataframe['&s-up_or_down'] = np.where(dataframe["close"].shift(-100) > dataframe["close"], 'up', 'down')
    return dataframe
```

完整示例请参考[分类器测试策略](https://github.com/freqtrade/freqtrade/blob/develop/tests/strategy/strats/freqai_test_classifier.py)。

### 利用`torch.compile()`优化性能

PyTorch自1.13起提供`torch.compile()`函数，用于提升GPU性能（详见[此](https://pytorch.org/tutorials/intermediate/torch_compile_tutorial.html)）。用法非常简单：将模型包裹在`torch.compile()`中即可：

```python
model = PyTorchMLPModel(
    input_dim=n_features,
    output_dim=1,
    **self.model_kwargs
)
model.to(self.device)
model = torch.compile(model)
```

之后正常使用模型即可。注意，此操作会关闭eager模式，出错信息可能不够详细。