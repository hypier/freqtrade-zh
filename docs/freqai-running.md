# 运行 FreqAI

有两种方法可以训练和部署自适应机器学习模型 —— 实时部署和历史回测。在这两种情况中，FreqAI都会定期运行/模拟模型的重新训练，示意如下图所示：

![freqai-window](assets/freqai_moving-window.jpg)

## 实时部署

可以使用以下命令运行FreqAI进行干跑/实时部署：

```bash
freqtrade trade --strategy FreqaiExampleStrategy --config config_freqai.example.json --freqaimodel LightGBMRegressor
```

启动后，FreqAI会根据配置设置开始训练一个新的模型，并赋予一个新的`identifier`。训练完成后，该模型将用于对传入的蜡烛数据进行预测，直到有新模型可用。新模型通常会尽可能频繁地生成，FreqAI会管理一个内部队列以维护要尝试的币对，确保所有模型保持同步更新。FreqAI始终会使用最新训练的模型对实时传入的数据进行预测。如果你不希望FreqAI如此频繁地重新训练新模型，可以设置`live_retrain_hours`参数，让FreqAI在训练新模型之前等待至少那多个小时。此外，还可以设置`expired_hours`，以让FreqAI避免对老于该小时数的模型进行预测。

训练完成的模型默认会保存到磁盘，便于在回测或崩溃后重用。你可以通过在配置中设置`"purge_old_models": true`来[清除旧模型](#purging-old-model-data)，节省存储空间。

若要从已保存的回测模型（或之前崩溃的干跑/实时会话）开始，只需指定特定模型的`identifier`：

```json
    "freqai": {
        "identifier": "example",
        "live_retrain_hours": 0.5
    }
```

此时，虽然FreqAI会用预训练模型启动，但它仍会检查距离模型上次训练经过的时间。如果超过了设定的`live_retrain_hours`小时数，FreqAI会开始训练一个新模型。

### 自动数据下载

FreqAI会自动下载确保模型训练所需的适量数据，依据的是`train_period_days`和`startup_candle_count`参数（详细说明请参考 [参数表](freqai-parameter-table.md)）。

### 保存预测数据

在特定`identifier`模型的生命周期中所有的预测结果都会存储在`historic_predictions.pkl`文件中，以便在崩溃或配置更改后重新加载。

### 清除旧模型数据

FreqAI会在每次成功训练后存储新模型文件。随着新模型的生成，为适应市场变化，旧模型会变得过时。若你计划长时间高频率运行FreqAI，建议在配置中启用`purge_old_models`：

```json
    "freqai": {
        "purge_old_models": 4,
    }
```

这会自动清除除最近训练的4个模型之外的所有模型，以节省空间。输入`0`则表示永不清除任何模型。

## 回测

可以使用如下命令执行FreqAI的回测模块：

```bash
freqtrade backtesting --strategy FreqaiExampleStrategy --strategy-path freqtrade/templates --config config_examples/config_freqai.example.json --freqaimodel LightGBMRegressor --timerange 20210501-20210701
```

如果该命令未曾在现有配置下运行过，FreqAI会为每个币对和每个回测窗口训练新模型（在扩展的`--timerange`范围内）。

回测模式在部署前需要提前[下载所需数据](#downloading-data-to-cover-the-full-backtest-period)（不同于干跑/实时模式，FreqAI不会自动下载数据）。请注意，下载的数据时间范围应覆盖比回测时间范围更长的时期。也就是说，需要将起始日期向前推`train_period_days`天，加上`startup_candle_count`（详细参数说明请参考 [参数表](freqai-parameter-table.md)），以确保模型能在回测开始的第一根蜡烛上做出预测。

详细计算可参考 [这里](#deciding-the-size-of-the-sliding-training-window-and-backtesting-duration)。

!!! 注意 "模型重用"
    训练完成后，可以在相同配置文件下再次执行回测，FreqAI会找到已训练好的模型并加载，而不会重新训练。这对于调节（甚至超参数优化）策略中的买卖条件非常有用。如果你希望用相同配置重新训练新模型，只需更改`identifier`，即可随时切换任何模型的使用。

!!! 注意
    每个回测窗口调用一次`set_freqai_targets()`（窗口总数为完整回测时间范围除以`backtest_period_days`参数），使目标模拟干跑/实时行为，不带先见偏差（look-ahead bias）。但在`feature_engineering_*()`中定义特征的方式是在整个训练时间段内一次性完成的，务必确保特征不包含未来信息。关于先见偏差（look-ahead bias）更多内容，参考 [常见错误](strategy-customization.md#common-mistakes-when-developing-strategies)。

---

### 保存回测预测数据

为了便于调整你的策略（**不是**特征！），FreqAI会自动在回测过程中保存预测结果，用于未来相同`identifier`模型的回测和实盘运行。这可以提升性能，特别是用于**高阶超参数调优**的入口/退出条件。

在`unique-id`文件夹中会新建一个名为`backtesting_predictions`的目录，存放所有预测结果（ feather 格式）。

如果需要更换特征（**必须**设置新的`identifier`），以提醒FreqAI训练新模型。

若希望从特定回测中保存模型，以便你后续可以直接用模型进行实盘部署，无需重新训练，则必须在配置中设置`save_backtest_models`为`True`。

!!! 注意
    为确保模型可重用，freqAI会用一个长度为1的DataFrame调用你的策略。如果你的策略需要更多数据以生成相同的特征，则不能直接使用回测预测进行实盘部署，必须为每次新回测更新`identifier`。

### 回测中实时使用历史预测

FreqAI支持通过回测参数`--freqai-backtest-live-models`重用实时历史预测。这在你需要对比或分析调试时非常有用。

需注意，`--timerange`参数无需传入，会由历史预测文件中的数据自动计算得出。

### 下载覆盖完整回测期间的数据

在实时/干跑部署中，FreqAI会自动下载所需数据。但在使用回测功能时，必须使用`download-data`命令（详情请参考 [数据下载](data-download.md#data-downloading)）。特别要注意，除了需要在回测期间范围内的数据外，还要确保前面有足够的历史数据用于模型训练。这个“额外”数据的数量，大致上可以通过将起始日期提前`train_period_days`天，再加上`startup_candle_count`（详情见参数表）来估算，例如：

假设`--timerange 20210501-20210701`，使用[示例配置](freqai-configuration.md#setting-up-the-configuration-file)，`train_period_days`为30，`startup_candle_count`为40，最大`include_timeframes`为1小时，计算如下：  
下载数据起点应为：`20210501` - 30天 - 40小时/24 = 20210330（大约提前31.7天）。

### 决定滑动训练窗口大小和回测时长

回测时间范围由配置文件中的`--timerange`参数定义。滑动训练窗口的长度由`train_period_days`设定，而`backtest_period_days`表示滑动回测窗口（小时模式下可以是浮点数，表示子日频率训练）。在[示例配置](freqai-configuration.md#setting-up-the-configuration-file)（路径：`config_examples/config_freqai.example.json`）中，用户设定模型训练30天，然后对随后的7天进行回测。执行完模型训练后，会对接下来的7天进行回测。之后，“滑动窗口”向前滚动一周（模拟FreqAI每周重新训练一次），新模型使用之前的30天（包括前一模型回测的那7天）数据进行训练。此过程一直重复，直到覆盖完整的`--timerange`。例如，若设为`20210501-20210701`，结束时会训练出8个模型（全范围跨越8周）。

!!! 注意
    虽然`backtest_period_days`可以是浮点数，但应注意：`--timerange`会除以此值，决定需要训练多少模型。例如，`--timerange 10`天，`backtest_period_days`为0.1，意味着每个币对需要训练100个模型，整个回测将非常耗时。实现完全的FreqAI自适应训练的回测时间会非常长。最稳妥的方法是运行干跑，让模型持续训练，回测时间与干跑相同。

## 定义模型失效时间

在干跑/实时模式下，FreqAI会在不同线程/GPU上依次训练各币对模型，模型之间难免存在年龄差。当你用50个币对，每个模型训练耗时为5分钟，最老模型也会超过4小时。这在某些策略的特征时间尺度（交易周期）少于4小时时，可能不太合适。你可以通过在配置中设置`expiration_hours`，限制模型的最大年龄（小时）：

```json
    "freqai": {
        "expiration_hours": 0.5,
    }
```

在示例配置中，用户只会允许调用年龄低于1/2小时的模型做出预测。

## 控制模型学习过程

模型训练参数由所用机器学习库决定。FreqAI允许你在配置中通过`model_training_parameters`字典设置任何参数（示例请参考 `config_examples/config_freqai.example.json`），如`Catboost`和`LightGBM`的参数。你也可以添加任何在所用库中支持的参数。

数据划分参数在`data_split_parameters`中定义，内容为scikit-learn的`train_test_split()`函数支持的参数。`train_test_split()`中的`shuffle`参数可以控制是否打乱数据，避免时间相关性带来的偏差。详细信息请查阅 [scikit-learn官网](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html)。

FreqAI的特定参数`label_period_candles`定义了用作`labels`的偏移（未来的蜡烛数）。在[示例配置](freqai-configuration.md#setting-up-the-configuration-file)中，定义标签未来24根蜡烛。

## 持续学习

你可以将`"continual_learning": true`添加到配置中，启用持续学习方案。开启后，首次从零开始训练模型，后续训练会基于前一次训练的最终模型状态继续进行，模型会“记住”之前的状态。默认值为`False`，意味着每次都从零训练模型。

???+ 危险 "持续学习会强制参数空间不变"
    由于`continual_learning`意味着模型参数空间**不能**变化，开启后`principal_component_analysis`会自动禁用。提示：PCA会改变参数空间和特征数，详情见 [特征工程](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)。

???+ 危险 "实验性功能"
    注意，这目前是一种简单的增量学习方法，存在过拟合或陷入局部最优的高风险，而市场走势偏离模型时表现可能更差。FreqAI中的机制主要是为实验设计，以便未来支持更成熟的连续学习方法，尤其是在加密货币等混沌系统中。

## 超参数优化（Hyperopt）

可使用与[典型的Freqtrade超参数调优](hyperopt.md)类似的命令：

```bash
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLoss --strategy FreqaiExampleStrategy --freqaimodel LightGBMRegressor --strategy-path freqtrade/templates --config config_examples/config_freqai.example.json --timerange 20220428-20220507
```

`hyperopt`要求提前按回测方式准备好数据（详见[数据下载](#downloading-data-to-cover-the-full-backtest-period)）。另外，在进行FreqAI策略超参数调优时，应注意以下限制：

- `--analyze-per-epoch`参数与FreqAI不兼容。
- 不能对`feature_engineering_*()`和`set_freqai_targets()`中的指标进行超参数调优，即不能用hyperopt优化模型参数。除此之外，其他所有[搜索空间](hyperopt.md#running-hyperopt-with-smaller-search-space)都可以调优。
- 回测指令同样适用。

最有效的超参数调优策略是专注于出入场的阈值或条件，不要调节特征中使用的参数。例如，不要试图超参数调节滑动窗口长度或任何改变预测的配置参数。FreqAI会存储预测结果（DataFrame形式）并复用，从而只需调节出入阈值。

一个示例超参数为[Dissimilarity Index（DI）](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)的阈值`DI_values`，超出此值的数据点被认定为异常值：  

```python
di_max = IntParameter(low=1, high=20, default=10, space='buy', optimize=True, load=True)
dataframe['outlier'] = np.where(dataframe['DI_values'] > self.di_max.value/10, 1, 0)
```

这个超参数可以帮助你找到最适合你参数空间的`DI_values`阈值。

## 使用Tensorboard

!!! 提示 "可用性"
    FreqAI支持Tensorboard，用于XGBoost、所有PyTorch模型、强化学习模型以及Catboost。如果希望支持其他模型类型的Tensorboard，可在[Freqtrade Github](https://github.com/freqtrade/freqtrade/issues)提交issue请求。

!!! 危险 "需求"
    需要FreqAI的PyTorch版本安装或容器镜像中包含Tensorboard。

最简便的方式是确保配置文件中的`freqai.activate_tensorboard`设为`True`（默认值），运行FreqAI后，在别的终端执行：

```bash
cd freqtrade
tensorboard --logdir user_data/models/unique-id
```

其中`unique-id`为`freqai`配置中的`identifier`。这个命令应在新终端中运行，才能在浏览器中访问`127.0.0.1:6060`（默认端口）查看。

![tensorboard](assets/tensorboard.jpg)

!!! 提示 "禁用以提升性能"
    Tensorboard记录会减慢训练速度，生产环境建议关闭。