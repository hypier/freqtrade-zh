# 参数表

下表列出了所有可用的FreqAI配置参数。其中部分参数在`config_examples/config_freqai.example.json`中已有示例。

必填参数标记为**Required**，必须通过建议的方式之一进行设置。

### 通用配置参数

| 参数 | 描述 |
|-----|-------|
|  |  **`config.freqai`树中包含的通用配置参数** |
| `freqai` | **必填。** <br> 控制FreqAI的所有参数的父字典。<br> **数据类型：** 字典。 |
| `train_period_days` | **必填。** <br> 用于训练数据的天数（滑动窗口的宽度）。<br> **数据类型：** 正整数。 |
| `backtest_period_days` | **必填。** <br> 在滑动窗口`train_period_days`定义的时间段之前，从训练好的模型中推断的天数。背测（backtesting）期间会进行多次训练（详细信息[这里](freqai-running.md#backtesting)）。可以是分数天，但请注意，提供的`timerange`将会被此数字除以，以得到完成后台测试所需的训练次数。<br> **数据类型：** 浮点数。 |
| `identifier` | **必填。** <br> 当前模型的唯一ID。如果模型存储在磁盘上，`identifier`方便重新加载特定的预训练模型或数据。<br> **数据类型：** 字符串。 |
| `live_retrain_hours` |  Dry/实时运行中模型的重训练频率。<br> **数据类型：** 浮点数 > 0。<br> 默认值：`0`（模型尽可能频繁重训练）。 |
| `expiration_hours` | 避免对超过`expiration_hours`时间的模型进行预测。<br> **数据类型：** 正整数。<br> 默认值：`0`（模型永不过期）。 |
| `purge_old_models` | 保留在磁盘上的模型数量（不影响背测）。默认为2，即Dry/实时运行会保存最新的两个模型。设置为0时，保存所有模型。此参数也可以接受布尔值以兼容旧版本。<br> **数据类型：** 整数。<br> 默认值：`2`。 |
| `save_backtest_models` |  进行背测时，将模型保存到磁盘。背测通过保存预测数据并在后续运行中重用，提高效率（例如调优入场/出场参数）。保存的模型文件还可以用于以相同`identifier`启动Dry/实时实例。<br> **数据类型：** 布尔值。<br> 默认值：`False`（不保存模型）。 |
| `fit_live_predictions_candles` | 用于计算目标（标签）统计的历史蜡烛数，优先于训练集中的数据。（可参考[这里](freqai-configuration.md#creating-a-dynamic-target-threshold)）<br> **数据类型：** 正整数。 |
| `continual_learning` | 使用最近训练完的模型的最终状态作为新模型的起点，实现增量学习（详细资料[这里](freqai-running.md#continual-learning)）。注意，这是目前一种较为天真的增量学习方法，市场环境变化可能导致过拟合或陷入局部最优。我们主要出于实验目的提供此连接，以便未来引入更成熟的连续学习方法，适应加密货币等复杂市场。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `write_metrics_to_disk` | 将训练时间、推断时间和CPU使用情况收集到json文件中。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `data_kitchen_thread_count` | 指定数据处理（异常值检测、归一化等）所用的线程数。此参数对训练时的线程数没有影响。未设置时（默认）会使用最大线程数减去2（留出1个物理核心供Freqtrade机器人和FreqUI使用）<br> **数据类型：** 正整数。 |
| `activate_tensorboard` | 指示是否激活tensorboard，用于支持的模块（目前包括RL、XGBoost、Catboost和PyTorch）。Tensorboard需要安装Torch，意味着需使用Torch/RL Docker镜像，或在安装时选择“是”以安装Torch。<br> **数据类型：** 布尔值。<br> 默认值：`True`。 |
| `wait_for_training_iteration_on_reload` | 使用/reload或ctrl-c时，等待当前训练迭代完成后再关闭，以确保训练完整。在设置为`False`时，将中断当前训练迭代，加快关机速度，但可能会丢失当前训练状态。<br> **数据类型：** 布尔值。<br> 默认值：`True`。 |

### 特征参数

| 参数 | 描述 |
|-----|-------|
|  |  **`freqai.feature_parameters`子字典中的特征参数** |
| `feature_parameters` | 一个字典，包含用以生成特征集的参数，详情和示例见[这里](freqai-feature-engineering.md)。<br> **数据类型：** 字典。 |
| `include_timeframes` | 一个时间框架列表，所有在`feature_engineering_expand_*()`中创建的指标都将针对这些时间框架生成。这些时间框架会作为特征添加到基础指标数据集中。<br> **数据类型：** 时间框架字符串列表。 |
| `include_corr_pairlist` | 一个与币对相关的币列表，FreqAI会将其作为附加特征加入到所有`pair_whitelist`币对中。在特征工程过程中（详见[这里](freqai-feature-engineering.md)）会为每个相关币创建指标。这些相关币的指标会添加到基础指标数据中。<br> **数据类型：** 资产字符串列表。 |
| `label_period_candles` | 预测标签所基于的未来蜡烛数量。可在`set_freqai_targets()`中使用（详见`templates/FreqaiExampleStrategy.py`中的示例）此参数非必需，用户可自定义标签并决定是否使用此参数。详见`templates/FreqaiExampleStrategy.py`示例。<br> **数据类型：** 正整数。 |
| `include_shifted_candles` | 在后续蜡烛中添加来自前一蜡烛的特征，用于增加历史信息。如果启用，FreqAI会复制并移位所有`include_shifted_candles`指定的前一蜡烛特征，使后续蜡烛可以访问到。这主要用于某些特定应用场景。<br> **数据类型：** 正整数。 |
| `weight_factor` | 根据数据点的时间新旧，为训练数据赋予不同的权重（详见[这里](freqai-feature-engineering.md#weighting-features-for-temporal-importance)）。<br> **数据类型：** 正浮点数（通常<1）。 |
| `indicator_max_period_candles` | **已废弃（#7325）**。被`startup_candle_count`取代，该参数在策略配置中设置（详见[策略配置部分](freqai-configuration.md#building-a-freqai-strategy)）。`startup_candle_count`不依赖时间框架，定义在`feature_engineering_*()`中用于指标创建的最大*周期*数。FreqAI会结合`include_time_frames`中的最大时间框架，计算下载的数据点个数，以保证第一个数据点非NaN。<br> **数据类型：** 正整数。 |
| `indicator_periods_candles` | 计算指标所用的时间段（蜡烛数），指标会添加到基础指标集。<br> **数据类型：** 正整数列表。 |
| `principal_component_analysis` | 自动应用主成分分析（PCA）降维。详细信息[这里](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `plot_feature_importances` | 为每个模型绘制前后`plot_feature_importances`个特征的重要性图。生成的图会存放在`user_data/models/<identifier>/sub-train-<COIN>_<timestamp>.html`。<br> **数据类型：** 整数。<br> 默认值：`0`。 |
| `DI_threshold` | 设置>0后启用不相似性指数（Dissimilarity Index）检测异常值的功能。详情[这里](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)。<br> **数据类型：** 正浮点数（通常<1）。 |
| `use_SVM_to_remove_outliers` | 训练支持向量机（SVM）检测并剔除训练数据和实时数据中的异常点。详细信息[这里](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)。<br> **数据类型：** 布尔值。 |
| `svm_params` | Sklearn中`SGDOneClassSVM()`的所有参数。详细参数信息[这里](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)。<br> **数据类型：** 字典。 |
| `use_DBSCAN_to_remove_outliers` | 使用DBSCAN算法对数据进行聚类，识别并剔除训练和预测中的异常值。详细内容[这里](freqai-feature-engineering.md#identifying-outliers-with-dbscan)。<br> **数据类型：** 布尔值。 |
| `noise_standard_deviation` | 如果设置，FreqAI会在训练特征中加入噪声，以防止过拟合。噪声由高斯分布生成，标准差为`noise_standard_deviation`，并加入到所有数据点中。由于FreqAI中的数据始终归一化为[-1,1]，此值应相对较小（例如0.05），以产生合理的噪声水平。<br> **数据类型：** 整数。<br> 默认值：`0`。 |
| `outlier_protection_percentage` | 启用后，若检测到超过设定百分比（`outlier_protection_percentage`）的点为异常点（通过SVM或DBSCAN），FreqAI会发出警告并忽略异常值检测，保持原始数据集完整。在此情况下，不会基于训练集进行预测。<br> **数据类型：** 浮点数。<br> 默认值：`30`。 |
| `reverse_train_test_order` | 将特征数据集拆分，使用最新的数据进行训练，历史数据用于测试（逆序）。可避免模型只训练到当前，但使用前须理解此参数的非传统性质。<br> **数据类型：** 布尔值。<br> 默认值：`False`（不反转）。 |
| `shuffle_after_split` | 将数据拆分为训练集和测试集后，再分别随机打乱。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `buffer_train_data_candles` | 在生成指标后，从训练数据的开头和结尾裁剪`buffer_train_data_candles`个蜡烛，以处理最大值/最小值预测时边界信息不足的问题。此功能特别适用于极值点检测。若目标是价格变动偏移，此缓冲可无须使用，因为偏移蜡烛在端部会被自动删除（NaN）。<br> **数据类型：** 整数。<br> 默认值：`0`。 |

### 数据拆分参数

| 参数 | 描述 |
|-----|-------|
|  |  **`freqai.data_split_parameters`子字典中的拆分参数** |
| `data_split_parameters` | 额外的scikit-learn `test_train_split()`参数，详见[此处](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html)（外部网页）。<br> **数据类型：** 字典。 |
| `test_size` | 用于测试的数据比例，非训练数据。<br> **数据类型：** 正浮点数<1。 |
| `shuffle` | 在训练过程中打乱数据点。通常为保持时间序列的时间顺序，设置为 `False`。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |

### 模型训练参数

| 参数 | 描述 |
|-----|-------|
|  |  **`freqai.model_training_parameters`子字典中的模型训练参数** |
| `model_training_parameters` | 一个灵活的字典，包含所选模型库的所有参数。例如，使用`LightGBMRegressor`时，可设置其参数（详见[这里](https://lightgbm.readthedocs.io/en/latest/pythonapi/lightgbm.LGBMRegressor.html)）。不同模型对应不同参数。当前支持的模型列表可见[这里](freqai-configuration.md#using-different-prediction-models)。<br> **数据类型：** 字典。 |
| `n_estimators` | 用于训练的提升树的数量。<br> **数据类型：** 整数。 |
| `learning_rate` | 训练中的提升学习率。<br> **数据类型：** 浮点数。 |
| `n_jobs`、`thread_count`、`task_type` | 设置用于并行处理的线程数，以及`task_type`（`gpu`或`cpu`）。不同模型库参数名可能不同。<br> **数据类型：** 浮点数。 |

### 强化学习参数

| 参数 | 描述 |
|-----|-------|
|  |  **`freqai.rl_config`子字典中的强化学习参数** |
| `rl_config` |  一个字典，包含强化学习模型的控制参数。<br> **数据类型：** 字典。 |
| `train_cycles` | 训练时间步数，根据`train_cycles * 训练数据点数量`设定。<br> **数据类型：** 整数。 |
| `max_trade_duration_candles` | 指导代理训练，控制交易长度。例如在`prediction_models/ReinforcementLearner.py`中的`calculate_reward()`函数中使用。<br> **数据类型：** 整数。 |
| `model_type` | 模型字符串，来自`stable_baselines3`或`SBcontrib`。支持`'TRPO', 'ARS', 'RecurrentPPO', 'MaskablePPO', 'PPO', 'A2C', 'DQN'`等，确保`model_training_parameters`与对应模型匹配（详见[文档](https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html)）。<br> **数据类型：** 字符串。 |
| `policy_type` | 支持的策略类型之一（来自stable_baselines3）。<br> **数据类型：** 字符串。 |
| `max_training_drawdown_pct` | 训练期间允许的最大回撤比例。<br> **数据类型：** 浮点数。<br> 默认值：0.8。 |
| `cpu_count` | 分配给RL训练的线程数（如果未选择`ReinforcementLearning_multiproc`）。建议保持默认，即总物理核心数减一。<br> **数据类型：** 整数。 |
| `model_reward_parameters` | 在`ReinforcementLearner.py`中自定义`calculate_reward()`函数使用的参数。<br> **数据类型：** 整数。 |
| `add_state_info` | 指示FreqAI是否在训练和推断的特征集加入状态信息（如交易持续时间、当前盈利、交易仓位等）。仅在Dry/实时运行中有效，背测中自动关闭。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `net_arch` | 神经网络架构，详见[此处](https://stable-baselines3.readthedocs.io/en/master/guide/custom_policy.html#examples)。默认`[128, 128]`，表示两个128单元的隐藏层。 |
| `randomize_starting_position` | 每轮随机起点，避免过拟合。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `drop_ohlc_from_features` | 不将归一化后的OHLC数据包含在特征集中（但仍用于环境模拟）。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `progress_bar` | 显示训练进度条，包括当前进度、耗时和剩余时间。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |

### PyTorch参数

#### 一般参数

| 参数 | 描述 |
|-----|-------|
|  |  **`freqai.model_training_parameters`子字典中的PyTorch模型训练参数** |
| `learning_rate` | 传给优化器的学习率。<br> **数据类型：** 浮点数。<br> 默认：`3e-4`。 |
| `model_kwargs` | 传给模型类的参数。<br> **数据类型：** 字典。<br> 默认：`{}`。 |
| `trainer_kwargs` | 传给训练器类的参数。<br> **数据类型：** 字典。<br> 默认：`{}`。 |

#### trainer_kwargs

| 参数 | 描述 |
|-------|-------|
|       |  **`freqai.model_training_parameters.model_kwargs`子字典中的模型训练参数** |
| `n_epochs` | 训练的总轮数，代表完整训练集在每次参数更新时被用到的次数。每次训练全过程为1个epoch。该参数会覆盖`n_steps`，必须设置其一。<br><br> **数据类型：** 整数（可选）。<br> 默认：`10`。 |
| `n_steps` | 用于替代`n_epochs`的训练步骤数，表示训练的总迭代次数。每次调用`optimizer.step()`为一次迭代。若已设置`n_epochs`则被忽略。简化版本的对应关系：<br><br> n_epochs = n_steps / (n_obs / batch_size) <br><br> 这样设计是为了让`n_steps`更易于调优且在不同数据规模中保持稳定。<br> **数据类型：** 整数（可选）。<br> 默认：`None`。 |
| `batch_size` | 训练中每个批次的大小。<br> **数据类型：** 整数。<br> 默认：`64`。 |

### 其他参数

| 参数 | 描述 |
|-----|-------|
|  |  **额外参数** |
| `freqai.keras` | 若所用模型依赖Keras（多用于TensorFlow模型），需开启此开关以确保模型的保存与加载符合Keras标准。<br> **数据类型：** 布尔值。<br> 默认：`False`。 |
| `freqai.conv_width` | 神经网络输入张量的宽度，用于替代移动蜡烛（`include_shifted_candles`），通过在第二维输入历史数据点。技术上也适用于回归器，但会增加计算开销。<br> **数据类型：** 整数。<br> 默认：`2`。 |
| `freqai.reduce_df_footprint` | 将所有数值列重编码为float32或int32，旨在减小内存/存储占用，缩短训练/推断时间。该参数在主配置文件中设置（不在FreqAI内部）。<br> **数据类型：** 布尔值。<br> 默认：`False`。 |