# 高级Hyperopt（超参数优化）

本页介绍一些高级的Hyperopt（超参数优化）主题，这些内容可能需要比创建有序超参数空间类更高的编码技能和Python知识。

## 创建和使用自定义损失函数

要使用自定义的损失函数类，确保在你的自定义Hyperopt损失类中定义了函数`hyperopt_loss_function`。  
在下面的示例中，必须在你的hyperopt调用中添加命令行参数`--hyperopt-loss SuperDuperHyperOptLoss`，以确保使用该函数。

以下示例与默认的Hyperopt损失实现相同。完整示例可在[userdata/hyperopts](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_hyperopt_loss.py)找到。

```python
from datetime import datetime
from typing import Any, Dict

from pandas import DataFrame

from freqtrade.constants import Config
from freqtrade.optimize.hyperopt import IHyperOptLoss

TARGET_TRADES = 600
EXPECTED_MAX_PROFIT = 3.0
MAX_ACCEPTED_TRADE_DURATION = 300

class SuperDuperHyperOptLoss(IHyperOptLoss):
    """
    定义超参数优化的默认损失函数
    """

    @staticmethod
    def hyperopt_loss_function(
        *,
        results: DataFrame,
        trade_count: int,
        min_date: datetime,
        max_date: datetime,
        config: Config,
        processed: dict[str, DataFrame],
        backtest_stats: dict[str, Any],
        starting_balance: float,
        **kwargs,
    ) -> float:
        """
        目标函数，返回越小越好
        这是legacy（旧版）算法（在freqtrade中一直使用）。
        权重分布如下：
        * 0.4 分配给交易持续时间
        * 0.25：规避交易亏损
        * 1.0：总利润，与上面定义的`EXPECTED_MAX_PROFIT`值比较
        """
        total_profit = results['profit_ratio'].sum()
        trade_duration = results['trade_duration'].mean()

        trade_loss = 1 - 0.25 * exp(-(trade_count - TARGET_TRADES) ** 2 / 10 ** 5.8)
        profit_loss = max(0, 1 - total_profit / EXPECTED_MAX_PROFIT)
        duration_loss = 0.4 * min(trade_duration / MAX_ACCEPTED_TRADE_DURATION, 1)
        result = trade_loss + profit_loss + duration_loss
        return result
```

当前，参数如下：

* `results`: DataFrame，包含交易结果数据。  
  结果中包含以下列（对应使用`--export trades`导出后的回测输出文件）：  
  `pair, profit_ratio, profit_abs, open_date, open_rate, fee_open, close_date, close_rate, fee_close, amount, trade_duration, is_open, exit_reason, stake_amount, min_rate, max_rate, stop_loss_ratio, stop_loss_abs`
* `trade_count`: 交易次数（等同于`len(results)`）
* `min_date`: 使用的时间范围起始日期
* `max_date`: 使用的时间范围结束日期
* `config`: 使用的配置对象（注意：如果策略相关参数在超参数空间中，则这些参数不会在此更新）
* `processed`: 以交易对为键的DataFrame字典，包含回测所用数据
* `backtest_stats`: 回测统计信息，格式同"strategy"子结构中的内容。字段可在`optimize_reports.py`中的`generate_strategy_stats()`查看。
* `starting_balance`: 用于回测的起始余额

该函数需要返回一个浮点数（`float`）。数值越小代表结果越好。参数调节和权重分配由用户自行决定。

!!! 注意
    该函数在每个epoch调用一次——请确保优化此函数，使其尽可能高效，以避免不必要地减慢超参数优化的速度。

!!! 注意 "`*args` 和 `**kwargs`"
    为了未来扩展，建议在接口中保持`*args`和`**kwargs`参数的存在。

## 重定义预设空间

要重定义某个预定义空间（如`roi_space`、`generate_roi_table`、`stoploss_space`、`trailing_space`、`max_open_trades_space`），可以定义一个嵌套类`Hyperopt`，并在其中定义所需的空间，例如：

```python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal

class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        # 定义自定义的止损空间
        def stoploss_space():
            return [SKDecimal(-0.05, -0.01, decimals=3, name='stoploss')]

        # 定义自定义的ROI空间
        def roi_space() -> List[Dimension]:
            return [
                Integer(10, 120, name='roi_t1'),
                Integer(10, 60, name='roi_t2'),
                Integer(10, 40, name='roi_t3'),
                SKDecimal(0.01, 0.04, decimals=3, name='roi_p1'),
                SKDecimal(0.01, 0.07, decimals=3, name='roi_p2'),
                SKDecimal(0.01, 0.20, decimals=3, name='roi_p3'),
            ]

        def generate_roi_table(params: Dict) -> dict[int, float]:
            roi_table = {}
            roi_table[0] = params['roi_p1'] + params['roi_p2'] + params['roi_p3']
            roi_table[params['roi_t3']] = params['roi_p1'] + params['roi_p2']
            roi_table[params['roi_t3'] + params['roi_t2']] = params['roi_p1']
            roi_table[params['roi_t3'] + params['roi_t2'] + params['roi_t1']] = 0
            return roi_table

        def trailing_space() -> List[Dimension]:
            # 所有参数都是必需的，只能修改类型或范围
            return [
                # 因为假设持续追踪止损始终启用，因此固定为True
                Categorical([True], name='trailing_stop'),
                SKDecimal(0.01, 0.35, decimals=3, name='trailing_stop_positive'),
                # ‘trailing_stop_positive_offset’应大于‘trailing_stop_positive’，
                # 所以使用此中间参数作为二者差值的值。
                # ‘trailing_stop_positive_offset’的值在generate_trailing_params()中构造。
                SKDecimal(0.001, 0.1, decimals=3, name='trailing_stop_positive_offset_p1'),
                Categorical([True, False], name='trailing_only_offset_is_reached'),
        ]

        # 定义自定义最大开仓交易数空间
        def max_open_trades_space(self) -> List[Dimension]:
            return [
                Integer(-1, 10, name='max_open_trades'),
            ]
```

!!! 注意
    所有重定义部分为可选，可以根据需要混合使用。

### 动态参数

参数也可以动态定义，但必须在调用过[`bot_start()`回调](strategy-callbacks.md#bot-start)后，实例已可用。

```python
class MyAwesomeStrategy(IStrategy):

    def bot_start(self, **kwargs) -> None:
        self.buy_adx = IntParameter(20, 30, default=30, optimize=True)

    # ...
```

!!! 警告
    这种方式创建的参数不会出现在`list-strategies`参数计数中。

### 重定义基础估算器（Base estimator）

你可以通过在`HyperOpt`子类中实现`generate_estimator()`来自定义超参数优化的估算器。

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            return "RF"
```

可能的取值包括`"GP"`、`"RF"`、`"ET"`、`"GBRT"`（详情请见[scikit-optimize文档](https://scikit-optimize.github.io/)），或继承自`RegressorMixin`的类的实例（从`sklearn`导入），且其`predict`方法包含可选的`return_std`参数，用于返回`std(Y | x)`以及`E[Y | x]`。

需要自行调研以找到其他适用的回归器。

示例：使用`ExtraTreesRegressor`（对应`"ET"`）且带附加参数

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            from skopt.learning import ExtraTreesRegressor
            # 对应"ET"，支持额外参数
            return ExtraTreesRegressor(n_estimators=100)
```

`dimensions`参数为待优化参数的`skopt.space.Dimension`对象列表。可以用来为`skopt.learning.GaussianProcessRegressor`创建各向同性核函数。示例：

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            from skopt.utils import cook_estimator
            from skopt.learning.gaussian_process.kernels import (Matern, ConstantKernel)
            kernel_bounds = (0.0001, 10000)
            kernel = (
                ConstantKernel(1.0, kernel_bounds) * 
                Matern(length_scale=np.ones(len(dimensions)), length_scale_bounds=[kernel_bounds for d in dimensions], nu=2.5)
            )
            kernel += (
                ConstantKernel(1.0, kernel_bounds) * 
                Matern(length_scale=np.ones(len(dimensions)), length_scale_bounds=[kernel_bounds for d in dimensions], nu=1.5)
            )
            return cook_estimator("GP", space=dimensions, kernel=kernel, n_restarts_optimizer=2)
```

!!! 注意
    虽然可以自定义估算器，但用户需自行研究参数，理解哪些参数应当使用。  
    如果不确定，建议直接使用默认（`"ET"`）或其他预设，不带额外参数。

## 空间选项

对于额外空间类型，`scikit-optimize`（与Freqtrade结合）提供以下几种类型：

* `Categorical` — 从一组类别中选择（如：`Categorical(['a', 'b', 'c'], name="cat")`）
* `Integer` — 从一组整数范围中选择（如：`Integer(1, 10, name='rsi')`）
* `SKDecimal` — 从有限精度的小数范围中选择（如：`SKDecimal(0.1, 0.5, decimals=3, name='adx')`）。*仅在freqtrade中可用*。
* `Real` — 从全精度的小数范围中选择（如：`Real(0.1, 0.5, name='adx')`）

这些都可以从`freqtrade.optimize.space`导入，虽然`Categorical`、`Integer`和`Real`其实只是对应scikit-optimize空间的别名。`SKDecimal`由freqtrade提供，用于更快的优化。

```python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal, Real  # noqa
```

!!! 提示 "SKDecimal vs. Real"
    建议在几乎所有情况下都使用`SKDecimal`而不是`Real`空间。  
    因为`Real`空间提供完整的精度（约16位小数），但实际几乎用不到，反而会导致超参数优化时间过长。

    以定义一个较小的空间为例（`SKDecimal(0.10, 0.15, decimals=2, name='xxx')`）——  
    `SKDecimal`会有5个可能值（`[0.10, 0.11, 0.12, 0.13, 0.14, 0.15]`）。

    相应的`Real(0.10, 0.15, name='xxx')`空间则几乎无限（`[0.10, 0.010000000001, 0.010000000002, ... 0.014999999999, 0.01500000000]`）。