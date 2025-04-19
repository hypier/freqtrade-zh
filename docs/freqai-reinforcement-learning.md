# 强化学习

!!! Note "安装大小"
    强化学习的依赖包括诸如 `torch` 等大型包，建议在执行 `./setup.sh -i` 时明确选择安装，并在问“是否还需要 freqai-rl 的依赖 (~700MB 额外空间) [y/N]?” 时回答“y”。
    喜欢使用 Docker 的用户应确保使用带有 `_freqairl` 后缀的 Docker 镜像。

## 背景与术语

### 什么是RL，FreqAI为何需要它？

强化学习涉及两个重要组成部分，*代理*（agent）和训练*环境*。在代理训练过程中，代理逐根蜡烛地遍历历史数据，不断选择一系列动作中的一个：多头入场、多头平仓、空头入场、空头平仓、中性（neutral）。在训练过程中，环境会根据这些动作的表现进行跟踪，并通过用户自定义的 `calculate_reward()`（这里提供了一个默认奖励，供用户在此基础上拓展【详细内容见#创建自定义奖励函数】）给予奖励。奖励用于训练神经网络中的权重。

FreqAI强化学习的第二个关键元素是*状态*信息。状态信息在每一步被输入到网络中，包括当前利润、当前持仓、当前交易持续时间等。这些信息既用于训练环境中的代理，也用于干/实盘操作中强化代理（此功能不可在回测中实现）。*FreqAI + Freqtrade*这是一个完美的结合，因为这些信息在实盘中是随时可用的。

强化学习是FreqAI的自然进化路径，因为它为模型加入了一个新的适应性层和市场反应机制，远胜于传统的分类器和回归器。然而，分类器和回归器具有强化学习所不具备的优势，比如预测的鲁棒性。不当训练的RL代理可能会找到“作弊”或“窍门”以最大化奖励，而不是真正实现盈利的交易。因此，RL的复杂度较高，理解难度也高于一般的分类器和回归器。

### RL接口

在当前框架中，我们旨在通过通用的“prediction model”文件暴露训练环境，该文件是用户继承自 `BaseReinforcementLearner` 的对象（例如：`freqai/prediction_models/ReinforcementLearner`）。在这个自定义类中，RL环境是可用且可配置的，通常通过 `MyRLEnv` 来实现【见下文#创建自定义奖励函数】。

大多数用户会专注于创造性的设计 `calculate_reward()` 函数【详见#创建自定义奖励函数】，而保持剩余环境配置不变。其他用户可以完全不触碰环境，只修改配置参数和FreqAI中已有的强大特征工程。而高级用户还可以自己定义完全独立的模型类。

该框架基于 stable_baselines3（torch）和 OpenAI gym，作为基础环境类。但整体上，模型类是相对隔离的 — 这样可以方便快速整合其他竞争库。环境继承自 `gym.Env`，因此如果需要切换到其他库，则必须重新编写环境。

### 重要注意事项

如前所述，代理在“人工交易环境”中“训练”。虽然这个环境看起来与Freqtrade的回测环境非常相似，但实际上*并不相同*。强化学习的训练环境显著更简化，没有包括复杂的策略逻辑，例如 `custom_exit`、`custom_stoploss`、杠杆控制等。RL环境是对真实市场的非常“原始”的表现形式，代理可以自主学习策略（如止损、止盈等），而这些策略由 `calculate_reward()` 强制执行。因此，必须理解训练环境与真实环境存在差异。

## 运行强化学习

设置与运行模型，与运行回归器或分类器相同。需要在命令行中指定两个参数 `--freqaimodel` 和 `--strategy`：

```bash
freqtrade trade --freqaimodel ReinforcementLearner --strategy MyRLStrategy --config config.json
```

其中 `ReinforcementLearner` 指使用 `freqai/prediction_models/ReinforcementLearner` 中的模板类（或用户自定义的类，存放在 `user_data/freqaimodels`），而策略参数则沿用一般【特征工程】的配置（即 `feature_engineering_*`），区别在于目标生成。强化学习不依赖目标（不需要标签），但FreqAI要求必须在动作列设置一个默认（中性）值：

```python
    def set_freqai_targets(self, dataframe, **kwargs) -> DataFrame:
        """
        *仅在启用FreqAI策略时生效*
        设置模型目标的必要函数。
        所有目标都必须以`&`作为前缀，以便被FreqAI内部识别。

        详细的特征工程信息请参阅：

        https://www.freqtrade.io/en/latest/freqai-feature-engineering

        :param df: 将接收目标的策略数据框
        例如用法：dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        # 对于RL，没有直接目标需要设置，这里填充（中性）
        dataframe["&-action"] = 0
        return dataframe
```

大部分功能与普通回归器相似，但以下示例展示了如何让策略将原始价格数据传递给代理，以便在训练环境中访问原始的OHLCV数据：

```python
    def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        # RL模型所需的特征
        dataframe[f"%-raw_close"] = dataframe["close"]
        dataframe[f"%-raw_open"] = dataframe["open"]
        dataframe[f"%-raw_high"] = dataframe["high"]
        dataframe[f"%-raw_low"] = dataframe["low"]
        return dataframe
```

最终，没有明确的“标签”字段 — 取而代之的是通过 `&-action` 列传递代理动作，该列在调用 `populate_entry_trend()` 和 `populate_exit_trend()` 时被用到。在示例中，中性动作设为0。这个值应与所用环境对应。FreqAI提供了两个环境，都采用0作为中性动作。

用户理解没有标签后，会发现代理自动做出“自主”的入场和出场决定。这使得策略设计变得相对简单。入场和出场信号由代理通过整数值直接传递，直接用于策略中的入场和出场判定：

```python
    def populate_entry_trend(self, df: DataFrame, metadata: dict) -> DataFrame:

        enter_long_conditions = [df["do_predict"] == 1, df["&-action"] == 1]

        if enter_long_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_long_conditions), ["enter_long", "enter_tag"]
            ] = (1, "long")

        enter_short_conditions = [df["do_predict"] == 1, df["&-action"] == 3]

        if enter_short_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_short_conditions), ["enter_short", "enter_tag"]
            ] = (1, "short")

        return df

    def populate_exit_trend(self, df: DataFrame, metadata: dict) -> DataFrame:
        exit_long_conditions = [df["do_predict"] == 1, df["&-action"] == 2]
        if exit_long_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_long_conditions), "exit_long"] = 1

        exit_short_conditions = [df["do_predict"] == 1, df["&-action"] == 4]
        if exit_short_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_short_conditions), "exit_short"] = 1

        return df
```

必须注意，`&-action`的具体数值与所用环境密切相关。上述示例中定义了5个动作：0为中性，1为多头入场，2为多头出场，3为空头入场，4为空头出场。

## 配置强化学习器

若要配置 `Reinforcement Learner`，需要在 `freqai` 配置文件中添加如下字典：

```json
        "rl_config": {
            "train_cycles": 25,
            "add_state_info": true,
            "max_trade_duration_candles": 300,
            "max_training_drawdown_pct": 0.02,
            "cpu_count": 8,
            "model_type": "PPO",
            "policy_type": "MlpPolicy",
            "model_reward_parameters": {
                "rr": 1,
                "profit_aim": 0.025
            }
        }
```

参数详情参见【频率表】【freqai-parameter-table.md】，但总体上，`train_cycles`决定代理在人工环境中遍历蜡烛数据的次数，用于训练模型的权重。`model_type`为字符串，选择 [stable_baselines](https://stable-baselines3.readthedocs.io/en/master/)(外部链接) 中的模型类型。

!!! Note
    如果希望启用 `continual_learning`，应在主配置字典中将其设置为`true`。这会让RL库在每次重新训练时，从上次模型的最终状态继续训练，而不是每次都重新从零开始。

!!! Note
    记得 `model_training_parameters` 字典应包含所选模型类型的所有超参数定制。例如，`PPO` 参数可以在【此链接】找到【链接：https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html】。

## 创建自定义奖励函数

!!! danger "非生产用途"
    警告！
    Freqtrade源码中提供的奖励函数仅作为功能展示，旨在展示尽可能多的环境控制特性，且运行速度快，适合小型计算机测试。这只是一个基准，不适合用于正式生产环境。请注意，用户需自行编写 `custom_reward()` 函数，或者使用其他用户在源码之外编写的模板，并存放在 `user_data/freqaimodels`。

当你开始修改策略和预测模型时，会很快意识到强化学习与回归器/分类器之间的几个关键区别。首先，策略没有设定目标值（无标签！），而是通过在 `MyRLEnv` 类中定义的 `calculate_reward()` 来实现奖励——示例默认实现位于 `prediction_models/ReinforcementLearner.py`，展示了创建奖励的基本要素，但绝非生产级。用户*必须*自行创建专用的强化学习模型类，或者使用外部预构建的模型，存放在 `user_data/freqaimodels`。在 `calculate_reward()` 中，可以实现对市场的创造性策略。例如，奖励获利交易，惩罚亏损交易；或者奖励入场，惩罚持仓过久。以下是一些常用奖励设计示例：

!!! note "提示"
    最佳的奖励函数应是连续可微且经过良好缩放的。比如，对罕见事件施加巨大负惩罚并不可取，因为神经网络无法学会这样的函数。相反，应对常见事件施以较小的负惩罚，帮助模型学习更快。此外，可以通过线性或指数函数，使奖励或惩罚的尺度随着事件的严重程度变化——比如，随着交易持续时间增加逐步加大惩罚，这比在某一时刻突然施加巨大惩罚要优。

```python
from freqtrade.freqai.prediction_models.ReinforcementLearner import ReinforcementLearner
from freqtrade.freqai.RL.Base5ActionRLEnv import Actions, Base5ActionRLEnv, Positions


class MyCoolRLModel(ReinforcementLearner):
    """
    用户自定义RL预测模型。

    将此文件保存至 `freqtrade/user_data/freqaimodels`

    然后在命令行中使用：

    freqtrade trade --freqaimodel MyCoolRLModel --config config.json --strategy SomeCoolStrat

    用户可以重载 `IFreqaiModel` 继承树中的任何函数。尤其对于RL，
    这是定义自定义 `MyRLEnv`（见下）和 `calculate_reward()`函数的关键位置，
    或者覆盖环境中的其他部分。

    此类还允许用户重载 `IFreqaiModel` 树中的任何其他部分。
    比如，用户可以重载 `def fit()`、`def train()` 或 `def predict()`，
    以实现更精细的控制。

    另一种常见的重载是 `def data_cleaning_predict()`，用户可以
    更精细地控制数据处理流程。
    """
    class MyRLEnv(Base5ActionRLEnv):
        """
        用户自定义环境。该类继承自BaseEnvironment和gym.Env。
        用户可以重载任何父类函数。这里是一个自定义`calculate_reward()`的示例。
        """
        def calculate_reward(self, action: int) -> float:
            # 首先，如果动作无效则惩罚
            if not self._is_valid(action):
                return -2
            pnl = self.get_unrealized_profit()

            factor = 100

            pair = self.pair.replace(':', '')

            # 可以从数据框中使用特征值
            # 假设策略中已生成了偏移的RSI指标
            rsi_now = self.raw_features[f"%-rsi-period_10_shift-1_{pair}_"
                            f"{self.config['timeframe']}"].iloc[self._current_tick]

            # 代理入场交易
            if (action in (Actions.Long_enter.value, Actions.Short_enter.value)
                    and self._position == Positions.Neutral):
                if rsi_now < 40:
                    factor = 40 / rsi_now
                else:
                    factor = 1
                return 25 * factor

            # 惩罚代理不入场
            if action == Actions.Neutral.value and self._position == Positions.Neutral:
                return -1
            max_trade_duration = self.rl_config.get('max_trade_duration_candles', 300)
            trade_duration = self._current_tick - self._last_trade_tick
            if trade_duration <= max_trade_duration:
                factor *= 1.5
            elif trade_duration > max_trade_duration:
                factor *= 0.5
            # 惩罚长时间持仓
            if self._position in (Positions.Short, Positions.Long) and \
            action == Actions.Neutral.value:
                return -1 * trade_duration / max_trade_duration
            # 平多
            if action == Actions.Long_exit.value and self._position == Positions.Long:
                if pnl > self.profit_aim * self.rr:
                    factor *= self.rl_config['model_reward_parameters'].get('win_reward_factor', 2)
                return float(pnl * factor)
            # 平空
            if action == Actions.Short_exit.value and self._position == Positions.Short:
                if pnl > self.profit_aim * self.rr:
                    factor *= self.rl_config['model_reward_parameters'].get('win_reward_factor', 2)
                return float(pnl * factor)
            return 0.
```

## 使用 Tensorboard

强化学习模型有训练指标的追踪需求。FreqAI已集成Tensorboard，方便用户查看所有币种及所有重新训练的训练效果。激活命令如下：

```bash
tensorboard --logdir user_data/models/唯一ID
```

其中 `唯一ID` 是在 `freqai` 配置文件中设置的 `identifier`。此命令在独立Shell中运行，浏览器访问地址为 `127.0.0.1:6006`（6006为默认端口）。

![tensorboard](assets/tensorboard.jpg)

## 自定义日志

FreqAI提供了内置的“剧集”摘要记录器 `self.tensorboard_log`，用于向Tensorboard写入自定义信息。默认情况下，该函数在环境每一步中会被调用一次，记录代理的动作。所有在单个episode中累计的值将在episode结束时报告，然后重置所有指标，迎接下一轮。

`self.tensorboard_log`也可以在环境中的任何位置调用，例如在`calculate_reward`中添加，用于收集更详细的奖励调用信息：

```python
    class MyRLEnv(Base5ActionRLEnv):
        """
        用户自定义环境。继承自BaseEnvironment和gym.Env。
        可以重载任何函数。这里是一个自定义`calculate_reward()`的示例。
        """
        def calculate_reward(self, action: int) -> float:
            if not self._is_valid(action):
                self.tensorboard_log("invalid")
                return -2
```

!!! Note
    `self.tensorboard_log()`函数仅适用于追踪累加对象（事件、动作等）。如果关注的事件是浮点数，则可以作为第二参数传入，例如：`self.tensorboard_log("float_metric1", 0.23)`。此时，指标值不会累加。

## 选择基础环境

FreqAI提供三种基础环境：`Base3ActionRLEnvironment`、`Base4ActionEnvironment` 和 `Base5ActionEnvironment`。正如名称所示，这些环境针对支持3、4或5个动作的代理。`Base3Action`最为简单，代理可选择持仓、做多或做空。这种环境可以用于仅做多策略（自动跟随`can_short`标志，由策略设定，做多为入场条件，平仓为退出条件）。而`Base4Action`环境下，代理可选择多头入场、空头入场、持仓中性、退出仓位。`Base5Action`环境与之类似，但拆分了退出多头和退出空头的动作。环境差异主要体现在：

* 在`calculate_reward`中可用的动作
* 策略中可用的动作

所有FreqAI的环境继承自一个无动作/仓位的基础环境 `BaseEnvironment`，包含通用逻辑。这个架构设计为自定义非常便利。最基本的修改是重载`calculate_reward()`【详见#创建自定义奖励函数】，但也可以修改环境内部的任何函数。方法是在策略模型文件中重载`MyRLEnv`中的对应函数。对于更复杂的定制，可以考虑完全重写继承自`BaseEnvironment`的环境。

!!! Note
    仅有`Base3ActionRLEnv`支持仅做多（长仓）训练（可以在策略中将`can_short = False`）。