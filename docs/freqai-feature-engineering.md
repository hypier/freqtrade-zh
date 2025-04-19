# 特征工程

## 定义特征

在用户策略中通过一组名为`feature_engineering_*`的函数执行低阶特征工程。这些函数设置“基础特征”，例如`RSI`、`MFI`、`EMA`、`SMA`、时间段、交易量等。`基础特征`可以是自定义指标，也可以从任何技术分析库中导入。FreqAI配备了一套函数以简化快速大规模特征工程：

| 函数                     | 描述                                                                                   |
|--------------------------|----------------------------------------------------------------------------------------|
| `feature_engineering_expand_all()` | 该可选函数会根据配置中的`indicator_periods_candles`、`include_timeframes`、`include_shifted_candles`和`include_corr_pairs`自动扩展定义的特征。|
| `feature_engineering_expand_basic()` | 该可选函数会根据配置中的`include_timeframes`、`include_shifted_candles`和`include_corr_pairs`自动扩展定义的特征。注意：此函数不跨`indicator_periods_candles`扩展。 |
| `feature_engineering_standard()` | 在基础时间框架的数据框上调用一次的可选函数。这是最后调用的函数，意味着输入该函数的数据框包含由其他`feature_engineering_expand`函数创建的所有特征和列。这也是进行自定义特殊特征提取（如tsfresh）以及包含不应自动扩展的特征（如星期几）的理想位置。 |
| `set_freqai_targets()` | 设置模型目标的必需函数。所有目标必须以`&`前缀，以被FreqAI内部识别。 |

与此同时，高级特征工程在FreqAI配置文件中的`"feature_parameters":{}`中处理。在此文件中，可以基于`base_features`决定大规模特征扩展，例如“包含相关配对”或“包含有信息量的时间框架”甚至“包含最近的蜡烛”。

建议从源码提供的示例策略中的`feature_engineering_*`模板函数开始（示例文件位于`templates/FreqaiExampleStrategy.py`），确保特征定义遵循正确规范。以下是在策略中设置指标和标签的示例：

```python
    def feature_engineering_expand_all(self, dataframe: DataFrame, period, metadata, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI策略时有效*
        该函数会根据配置中的`indicator_periods_candles`, `include_timeframes`, `include_shifted_candles`, 和`include_corr_pairs`自动扩展定义的特征。换句话说，在此函数中定义的单个特征，将自动扩展出
        `indicator_periods_candles` * `include_timeframes` * `include_shifted_candles` * `include_corr_pairs` 个特征添加到模型中。

        所有特征前必须加`%`以被FreqAI内部识别。

        通过以下方式访问元数据（如当前交易对/时间框架/周期）：

        `metadata["pair"]` `metadata["tf"]`  `metadata["period"]`

        :param df: 将接收特征的策略数据框
        :param period: 指标的周期 - 示例用法：
        :param metadata: 当前交易对的元数据
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)
        """

        dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
        dataframe["%-mfi-period"] = ta.MFI(dataframe, timeperiod=period)
        dataframe["%-adx-period"] = ta.ADX(dataframe, timeperiod=period)
        dataframe["%-sma-period"] = ta.SMA(dataframe, timeperiod=period)
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)

        bollinger = qtpylib.bollinger_bands(
            qtpylib.typical_price(dataframe), window=period, stds=2.2
        )
        dataframe["bb_lowerband-period"] = bollinger["lower"]
        dataframe["bb_middleband-period"] = bollinger["mid"]
        dataframe["bb_upperband-period"] = bollinger["upper"]

        dataframe["%-bb_width-period"] = (
            dataframe["bb_upperband-period"]
            - dataframe["bb_lowerband-period"]
        ) / dataframe["bb_middleband-period"]
        dataframe["%-close-bb_lower-period"] = (
            dataframe["close"] / dataframe["bb_lowerband-period"]
        )

        dataframe["%-roc-period"] = ta.ROC(dataframe, timeperiod=period)

        dataframe["%-relative_volume-period"] = (
            dataframe["volume"] / dataframe["volume"].rolling(period).mean()
        )

        return dataframe
```

```python
    def feature_engineering_expand_basic(self, dataframe: DataFrame, metadata, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI策略时有效*
        该函数会根据配置中的`include_timeframes`、`include_shifted_candles`和`include_corr_pairs`自动扩展定义的特征。
        换句话说，在此函数中定义的单个特征，将自动扩展出
        `include_timeframes` * `include_shifted_candles` * `include_corr_pairs` 个特征加入模型。

        此处定义的特征不会在用户自定义的`indicator_periods_candles`中自动重复。

        通过以下方式访问元数据（如当前交易对/时间框架）：

        `metadata["pair"]` `metadata["tf"]`

        所有特征前面必须加`%`以被FreqAI识别。

        :param df: 将接收特征的策略数据框
        :param metadata: 当前交易对的元数据
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-ema-200"] = ta.EMA(dataframe, timeperiod=200)
        """
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        return dataframe
```

```python
    def feature_engineering_standard(self, dataframe: DataFrame, metadata, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI策略时有效*
        该可选函数会针对基础时间框架的数据框调用一次。这是最后调用的函数，意味着进入此函数的数据框将包含所有其它`freqai_feature_engineering_*`函数创建的特征和列。

        这是进行自定义特殊特征提取（如tsfresh）的良好位置，也是适合不应自动扩展的特征（如星期几）的存放位置。

        通过以下方式访问元数据（如当前交易对）：

        `metadata["pair"]`

        所有特征前必须加`%`以被FreqAI内部识别。

        :param df: 将接收特征的策略数据框
        :param metadata: 当前交易对的元数据
        使用示例： dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        """
        dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        dataframe["%-hour_of_day"] = (dataframe["date"].dt.hour + 1) / 25
        return dataframe
```

```python
    def set_freqai_targets(self, dataframe: DataFrame, metadata, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI策略时有效*
        设置模型目标的必需函数。所有目标必须以`&`前缀，以被FreqAI内部识别。

        访问元数据（如当前交易对）：

        `metadata["pair"]`

        :param df: 将接收目标的策略数据框
        :param metadata: 当前交易对的元数据
        使用示例： dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
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

在上述示例中，用户不希望将`bb_lowerband`作为特征传入模型，因此没有在前面加`%`。而用户希望将`bb_width`传入模型用于训练/预测，因此在前面加了`%`。

定义好`基础特征`后，下一步就是利用配置文件中的强大`feature_parameters`进行扩展，例如：

```json
    "freqai": {
        //...
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
        //...
    }
```

这里的`include_timeframes`是策略中调用`feature_engineering_expand_*()`的时间框架（`tf`）。在示例中，用户请求包含`5m`、`15m`和`4h`的`rsi`、`mfi`、`roc`和`bb_width`特征。

同时，可以通过`include_corr_pairlist`为每个定义的特征集包含相关配对的特征。这意味着特征集将包含所有在所有`include_timeframes`中为每个在配置中定义的相关配对（如`ETH/USD`、`LINK/USD`、`BNB/USD`）扩展的特征。

`include_shifted_candles`指示包含的前多少期蜡烛（k线）。例如，`include_shifted_candles: 2`表示FreqAI会包含每个特征的过去两根蜡烛数据。

总的来说，用户通过示例策略中的配置产生的特征数量为：
`长度 of include_timeframes` * `feature_engineering_expand_*()`中的特征数 * `include_corr_pairlist`的长度 * `include_shifted_candles` * `indicator_periods_candles`  
即：`3 * 3 * 3 * 2 * 2 = 108`。

!!! note "了解更多创意特征工程的方法"
    查看我们的[Medium文章](https://emergentmethods.medium.com/freqai-from-price-to-prediction-6fadac18b665)，帮助用户学习如何创造性地工程特征。

### 利用`metadata`实现对`feature_engineering_*`函数的更细粒度控制

所有`feature_engineering_*`和`set_freqai_targets()`函数都接收一个`metadata`字典，包含关于`pair`、`tf`（时间框架）和`period`的信息。用户可以利用`metadata`作为条件，阻止或保留某些时间框架、周期或交易对的特征。

```python
def feature_engineering_expand_all(self, dataframe: DataFrame, period, metadata, **kwargs) -> DataFrame:
    if metadata["tf"] == "1h":
        dataframe["%-roc-period"] = ta.ROC(dataframe, timeperiod=period)
```

上例中，仅在时间框架为`"1h"`时，才会添加`ta.ROC()`特征，其他时间框架将被阻止。

### 训练结束后返回额外信息

重要指标可以在每次模型训练结束时返回，并通过在自定义预测模型类中将其赋值给`dk.data['extra_returns_per_train']['my_new_value'] = XYZ`来实现。  

FreqAI会将这个`my_new_value`展开，以适应返回给策略的数据框结构，策略中可以通过`dataframe['my_new_value']`访问。示例：用`&*_mean`和`&*_std`值动态设定目标阈值（详见[创建动态目标阈值](freqai-configuration.md#creating-a-dynamic-target-threshold)）。

另一个示例是使用交易数据库中的实时指标，例如：  

```json
    "freqai": {
        "extra_returns_per_train": {"total_profit": 4}
    }
```

你需要在配置中预设字典，使FreqAI能返回正确的数据框形状。这些值可能会被预测模型覆盖，但在模型未设置或需要默认值时，这些预设值会被返回。

### 权重因子用于时间重要性

FreqAI支持通过指数函数设置`weight_factor`，赋予近期数据更高权重：

$$ W_i = \exp(\frac{-i}{\alpha*n}) $$

其中 $W_i$ 是第 $i$ 个数据点的权重，$n$是数据点总数。以下图示不同权重因子对特征集数据点的影响。

![weight-factor](assets/freqai_weight-factor.jpg)

## 构建数据管道

默认情况下，FreqAI根据用户配置自动构建动态管道。默认包括`MinMaxScaler(-1,1)`和`VarianceThreshold`（去除方差为零的列）。用户可以通过配置启用其他步骤，例如：  

- 加入`use_SVM_to_remove_outliers: true`，FreqAI会自动添加`SVMOutlierExtractor`到管道（详见[使用支持向量机识别离群点](#identifying-outliers-using-a-support-vector-machine-svm)）  
- 添加`principal_component_analysis: true`，启用PCA  
- 配置`DI_threshold: 1`，启用异质性指标（Dissimilarity Index）  
- 添加噪声：`noise_standard_deviation: 0.1`  
- 启用DBSCAN离群点去除：`use_DBSCAN_to_remove_outliers: true`

!!! note "更多参数说明"
    请查阅 [参数表](freqai-parameter-table.md)，了解更多详情。

### 自定义数据管道

鼓励用户根据需要自定义数据管道。实现方式是将`dk.feature_pipeline`设为自定义的`Pipeline`对象，或者重载`define_data_pipeline`/`define_label_pipeline`函数（详见[迁移指南](strategy_migration.md#freqai---new-data-pipeline)）。  

FreqAI使用的是 [`DataSieve`](https://github.com/emergentmethods/datasieve) 管道，遵循sklearn管道API，并增加了数据点移除、一致性维护、特征名追踪等功能。  

示例：  

```python
from datasieve.transforms import SKLearnWrapper, DissimilarityIndex
from datasieve.pipeline import Pipeline
from sklearn.preprocessing import QuantileTransformer, StandardScaler
from freqai.base_models import BaseRegressionModel

class MyFreqaiModel(BaseRegressionModel):
    """
    一些酷炫的自定义模型
    """
    def fit(self, data_dictionary: Dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        """
        我的自定义训练函数
        """
        model = cool_model.fit()
        return model

    def define_data_pipeline(self) -> Pipeline:
        """
        用户可自定义特征管道
        """
        feature_pipeline = Pipeline([
            ('qt', SKLearnWrapper(QuantileTransformer(output_distribution='normal'))),
            ('di', ds.DissimilarityIndex(di_threshold=1))
        ])

        return feature_pipeline
    
    def define_label_pipeline(self) -> Pipeline:
        """
        用户可自定义标签管道
        """
        label_pipeline = Pipeline([
            ('qt', SKLearnWrapper(StandardScaler())),
        ])

        return label_pipeline
```

此示例中，定义了训练和预测时使用的特征管道，你可以用大部分sklearn变换步骤，包装在`SKLearnWrapper`中。还可以使用任何`DataSieve`库中的转换。

自定义转换只需通过继承`BaseTransform`并实现`fit()`、`transform()`和`inverse_transform()`方法，比如：  

```python
from datasieve.transforms.base_transform import BaseTransform
# 其他导入

class MyCoolTransform(BaseTransform):
    def __init__(self, **kwargs):
        self.param1 = kwargs.get('param1', 1)

    def fit(self, X, y=None, sample_weight=None, feature_list=None, **kwargs):
        # 训练逻辑
        return X, y, sample_weight, feature_list

    def transform(self, X, y=None, sample_weight=None,
                  feature_list=None, outlier_check=False, **kwargs):
        # 转换逻辑
        return X, y, sample_weight, feature_list

    def inverse_transform(self, X, y=None, sample_weight=None, feature_list=None, **kwargs):
        # 逆变换逻辑
        return X, y, sample_weight, feature_list
```

!!! note "提示"
    自定义类可以与`IFreqaiModel`写在同一文件中。

### 将自定义`IFreqaiModel`迁移到新管道

如果已有自定义的`IFreqaiModel`，且在`train()`/`predict()`中使用`data_cleaning_train/predict()`，则需要迁移到新管道。若不依赖其，迁移无关紧要。

迁移详情请查阅[这里](strategy_migration.md#freqai---new-data-pipeline)。

## 异常值检测

市场中存在大量无规律噪声和离群点，FreqAI引入多种方法识别并减轻风险。

### 使用异质性指标（DI）识别离群点

异质性指标（DI）度量模型预测的不确定性。

通过在配置中加入：  

```json
    "freqai": {
        "feature_parameters" : {
            "DI_threshold": 1
        }
    }
```

会在特征管道中加入`DissimilarityIndex`，阈值为1。DI允许识别为离群的预测（不在模型特征空间内），并舍弃以降低风险。其原理为：计算每个训练点$X_a$与其他所有点$X_b$的距离：  

$$ d_{ab} = \sqrt{\sum_{j=1}^p(X_{a,j}-X_{b,j})^2} $$

然后，计算训练集的平均距离：  

$$ \overline{d} = \sum_{a=1}^n(\sum_{b=1}^n(d_{ab}/n)/n) $$

新预测点$X_k$与训练集的距离：  

$$ d_k = \arg \min d_{k,i} $$

估算DI：  

$$ DI_k = d_k / \overline{d} $$

可以调节`DI_threshold`以控制离群点判定的敏感度。阈值越高，模型容许的预测越远，越宽松；阈值越低则越严格。

以下图示为3D数据集的DI示意：

![DI](assets/freqai_DI.jpg)

### 使用支持向量机（SVM）识别离群点

可在配置中加入：  

```json
    "freqai": {
        "feature_parameters" : {
            "use_SVM_to_remove_outliers": true
        }
    }
```

会自动在特征管道中加入`SVMOutlierExtractor`。SVM在训练后会判断数据点是否超出特征空间，超出的将被移除。

你也可以通过`feature_parameters.svm_params`设置参数，如：`shuffle`和`nu`。  

- `shuffle`默认值为`False`，确保结果一致。若设置为`True`，每次训练可能不同（因`max_iter`过低导致收敛不到`tol`），建议适当增大`max_iter`。  
- `nu`大致代表被判定为离群的比例，值在0到1之间。

### 使用DBSCAN识别离群点

你可以启用：  

```json
    "freqai": {
        "feature_parameters" : {
            "use_DBSCAN_to_remove_outliers": true
        }
    }
```

加入`DataSieveDBSCAN`。这是无监督聚类算法，不需预先知道簇的数量。

定义：给定点数$N$和距离阈值$\varepsilon$，DBSCAN会将每个点周围半径$\varepsilon$内至少有$N-1$个点的点标记为“核心点”；距离$\varepsilon$内有核心点但自己未达到要求的点为“边界点”。簇由核心点和边界点组成；没有邻居的点被视为离群点。

示意图：  

![dbscan](assets/freqai_dbscan.jpg)

采用`sklearn.cluster.DBSCAN`，其中`min_samples`取值为数据点数量的1/4（按蜡烛数计算），`eps`自动采用k距图的拐点。

### 使用主成分分析（PCA）进行降维

通过配置：  

```json
    "freqai": {
        "feature_parameters" : {
            "principal_component_analysis": true
        }
    }
```

启用PCA，将特征降至解释方差≥0.999的维度。这能加快训练速度，使模型更新更快。