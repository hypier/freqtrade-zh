## 运行策略所需的导入

在创建策略时，你需要导入必要的模块和类。以下导入是策略所需的基本内容：

默认情况下，我们建议使用以下导入作为策略的基础：
这将涵盖所有使频繁交易（freqtrade）功能正常运行的导入。
当然，你可以根据策略需要添加更多的导入。

``` python
# flake8: noqa: F401
# isort: skip_file
# --- 请勿删除以下导入 ---
import numpy as np
import pandas as pd
from datetime import datetime, timedelta, timezone
from pandas import DataFrame
from typing import Dict, Optional, Union, Tuple

from freqtrade.strategy import (
    IStrategy,
    Trade, 
    Order,
    PairLocks,
    informative,  # @informative 装饰器
    # 超参数（Hyperopt）相关
    BooleanParameter,
    CategoricalParameter,
    DecimalParameter,
    IntParameter,
    RealParameter,
    # 时间框架辅助函数
    timeframe_to_minutes,
    timeframe_to_next_date,
    timeframe_to_prev_date,
    # 策略辅助函数
    merge_informative_pair,
    stoploss_from_absolute,
    stoploss_from_open,
)

# --------------------------------
# 在这里添加你的库导入
import talib.abstract as ta
from technical import qtpylib
```