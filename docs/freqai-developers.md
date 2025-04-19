# 开发

## 项目架构

FreqAI的架构和功能设计旨在鼓励开发具有特色的功能、模型和算法等。

类结构和详细算法概述如下图所示：

![image](assets/freqai_algorithm-diagram.jpg)

如图所示，FreqAI由三个不同的对象组成：

* **IFreqaiModel** - 一个单例的持久对象，包含收集、存储和处理数据、特征工程、训练和推理模型所需的全部逻辑。
* **FreqaiDataKitchen** - 非持久对象，为每个唯一资产/模型单独创建。除了元数据外，还包含各种数据处理工具。
* **FreqaiDataDrawer** - 一个单例的持久对象，存储所有历史预测、模型，以及保存/加载方法。

FreqAI内置多种[预测模型](freqai-configuration.md#using-different-prediction-models)，它们直接继承自`IFreqaiModel`。每个模型都可以完全访问`IFreqaiModel`中的所有方法，并在需要时覆盖任何函数。然而，高级用户通常会主要覆盖`fit()`、`train()`、`predict()`和`data_cleaning_train/predict()`等方法。

## 数据处理

FreqAI旨在以简化后处理和增强崩溃恢复能力的方式，组织模型文件、预测数据和元数据。所有数据存放在文件结构`user_data_dir/models/`中，该结构包含与训练和回测相关的所有数据。`FreqaiDataKitchen()`高度依赖此文件结构以确保正常训练和推理，因此不应手动修改。

### 文件结构

文件结构根据在[配置](freqai-configuration.md#setting-up-the-configuration-file)中设置的模型`identifier`自动生成。以下结构展示了后处理数据存储位置：

| 结构 | 描述 |
|-----------|---------|
| `config_*.json` | 模型特定配置文件的副本。 |
| `historic_predictions.pkl` | 一个文件，包含在模型`identifier`的整个生命周期内生成的所有历史预测。在发生崩溃或配置变更后用于重新加载模型。备份文件始终存在，以防主文件损坏。FreqAI**会自动**检测损坏并用备份文件替换。 |
| `pair_dictionary.json` | 包含训练队列以及最近训练完的模型在磁盘上的位置。 |
| `sub-train-*_TIMESTAMP` | 存储某个模型所有相关文件的文件夹，如：<br>
|| `*_metadata.json` - 模型的元数据，例如归一化的最大最小值、预期训练特征列表等。<br>
|| `*_model.*` - 保存到磁盘的模型文件，用于崩溃后重载。可能格式有`joblib`（常用的增强库）、`zip`（stable_baselines）、`hd5`（Keras类型）等。<br>
|| `*_pca_object.pkl` - [主成分分析（PCA）](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)变换（如果配置中`principal_component_analysis: True`），用于转换未见过的预测特征。<br>
|| `*_svm_model.pkl` - [支持向量机（SVM）](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)模型（如果配置中`use_SVM_to_remove_outliers: True`），用于检测未见预测特征中的异常值。<br>
|| `*_trained_df.pkl` - 包含用于训练`identifier`模型的所有训练特征的数据集，用于计算[不相似指数（DI）](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)，也可用于后处理。<br>
|| `*_trained_dates.df.pkl` - 与`trained_df.pkl`对应的日期信息，有助于后处理。 |

例如的文件结构如下所示：

```
├── models
│   └── unique-id
│       ├── config_freqai.example.json
│       ├── historic_predictions.backup.pkl
│       ├── historic_predictions.pkl
│       ├── pair_dictionary.json
│       ├── sub-train-1INCH_1662821319
│       │   ├── cb_1inch_1662821319_metadata.json
│       │   ├── cb_1inch_1662821319_model.joblib
│       │   ├── cb_1inch_1662821319_pca_object.pkl
│       │   ├── cb_1inch_1662821319_svm_model.joblib
│       │   ├── cb_1inch_1662821319_trained_dates_df.pkl
│       │   └── cb_1inch_1662821319_trained_df.pkl
│       ├── sub-train-1INCH_1662821371
│       │   ├── cb_1inch_1662821371_metadata.json
│       │   ├── cb_1inch_1662821371_model.joblib
│       │   ├── cb_1inch_1662821371_pca_object.pkl
│       │   ├── cb_1inch_1662821371_svm_model.joblib
│       │   ├── cb_1inch_1662821371_trained_dates_df.pkl
│       │   └── cb_1inch_1662821371_trained_df.pkl
│       ├── sub-train-ADA_1662821344
│       │   ├── cb_ada_1662821344_metadata.json
│       │   ├── cb_ada_1662821344_model.joblib
│       │   ├── cb_ada_1662821344_pca_object.pkl
│       │   ├── cb_ada_1662821344_svm_model.joblib
│       │   ├── cb_ada_1662821344_trained_dates_df.pkl
│       │   └── cb_ada_1662821344_trained_df.pkl
│       └── sub-train-ADA_1662821399
│           ├── cb_ada_1662821399_metadata.json
│           ├── cb_ada_1662821399_model.joblib
│           ├── cb_ada_1662821399_pca_object.pkl
│           ├── cb_ada_1662821399_svm_model.joblib
│           ├── cb_ada_1662821399_trained_dates_df.pkl
│           └── cb_ada_1662821399_trained_df.pkl

```