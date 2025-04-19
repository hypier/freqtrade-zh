## 订单使用的价格

普通订单的价格可以通过参数结构`entry_pricing`（建仓）和`exit_pricing`（平仓）进行控制。  
价格总是在下单前即刻获取，无论是通过查询交易所的行情（tickers）还是使用订单簿（orderbook）数据。

!!! 注意
    Freqtrade使用的订单簿数据是通过ccxt库的`fetch_order_book()`函数从交易所获取的，通常是来自L2聚合的订单簿数据，而行情数据是由ccxt库的`fetch_ticker()`或`fetch_tickers()`函数返回的结构。详情请参考ccxt库的[文档](https://github.com/ccxt/ccxt/wiki/Manual#market-data)。

!!! 警告 "使用市价单"
    使用市价单时，请务必阅读[市价单定价](#market-order-pricing)部分。

### 建仓价格

#### 价格方向设置

配置项`entry_pricing.price_side`定义了在买入时，机器人关注的订单簿一侧。

以下展示了一个订单簿示意：

``` explanation
...
103
102
101  # 卖价
------------- 当前价差
99   # 买价
98
97
...
```

如果`entry_pricing.price_side`设置为`"bid"`，则机器人会以99作为建仓价格。  
相反，如果设置为`"ask"`，则使用101作为建仓价格。

这取决于订单方向（_多头_/_空头_），会产生不同的效果。因此建议使用`"same"`或`"other"`作为此配置值。  
对应的定价矩阵如下所示：

| 方向   | 订单类型 | 设置       | 价格  | 是否穿越价差 |
|--------|----------|------------|-------|--------------|
| 多头   | 买入     | ask        | 101   | 是           |
| 多头   | 买入     | bid        | 99    | 否           |
| 多头   | 买入     | same       | 99    | 否           |
| 多头   | 买入     | other      | 101   | 是           |
| 空头   | 卖出     | ask        | 101   | 否           |
| 空头   | 卖出     | bid        | 99    | 是           |
| 空头   | 卖出     | same       | 101   | 否           |
| 空头   | 卖出     | other      | 99    | 是           |

使用订单簿的另一侧通常能保证订单更快成交，但机器人也可能付出不必要的额外价格。  
在使用限价买单时，很可能会发生taker费用而非maker费用。另外，订单簿的“另一侧”价格通常高于“买价”一侧的价格，因此此行为类似于市场价（但有最大价格限制）。

#### 开仓价格（启用订单簿时）

当开启订单簿（`entry_pricing.use_order_book=True`）进行建仓时，Freqtrade会从订单簿中提取`entry_pricing.order_book_top`条目，并以配置的订单簿**对应侧**（`entry_pricing.price_side`）的第`entry_pricing.order_book_top`个位置价格作为建仓价格。  
1代表订单簿的最顶端（最优挂单），2代表第二个挂单，以此类推。

#### 开仓价格（未启用订单簿）

以下内容假设`side`为配置的`entry_pricing.price_side`（默认为`"same"`）。

当未启用订单簿（`entry_pricing.use_order_book=False`）时，Freqtrade会选择行情中`side`方向的最佳价格，前提是该价格低于最近成交价（`last`）。  
如果`side`价格高于`last`，则会根据参数`entry_pricing.price_last_balance`在`side`价格和`last`价格之间插值，得出建仓价格。

参数`entry_pricing.price_last_balance`控制这个过程。  
值为`0.0`表示完全使用`side`价格，  
值为`1.0`表示完全使用`last`价格，  
介于两者之间的值则在挂单价和最近成交价之间插值。

#### 观察市场深度

当启用市场深度检测（`entry_pricing.check_depth_of_market.enabled=True`）时，建仓信号会根据订单簿深度过滤（订单簿各侧的所有挂单金额总和）。

订单簿的`bid`（买）一侧深度除以`ask`（卖）一侧深度得出一个比值，  
然后与参数`entry_pricing.check_depth_of_market.bids_to_ask_delta`的值比较。  
只有当订单簿深度差（delta）大于或等于设置值时，订单才会被执行。

!!! 注意
    如果delta值低于1，意味着`ask`（卖）一侧的深度大于`bid`（买）一侧的深度；  
    如果delta值大于1，则相反（买侧深度大于卖侧深度）。

### 平仓价格

#### 平仓价格方向

配置项`exit_pricing.price_side`定义了在平仓时，机器人关注的订单簿一侧。

订单簿示意如下：

``` explanation
...
103
102
101  # 卖价
------------- 当前价差
99   # 买价
98
97
...
```

如果`exit_pricing.price_side`设置为`"ask"`，则使用101作为平仓价格。  
如果设置为`"bid"`，则使用99作为平仓价格。

订单方向（_多头_/_空头_）不同，会导致不同的计算结果。建议使用`"same"`或`"other"`配置值。  
对应的定价矩阵如下所示：

| 方向   | 订单类型 | 设置       | 价格  | 是否穿越价差 |
|--------|----------|------------|-------|--------------|
| 多头   | 卖出     | ask        | 101   | 否           |
| 多头   | 卖出     | bid        | 99    | 是           |
| 多头   | 卖出     | same       | 101   | 否           |
| 多头   | 卖出     | other      | 99    | 是           |
| 空头   | 买入     | ask        | 101   | 是           |
| 空头   | 买入     | bid        | 99    | 否           |
| 空头   | 买入     | same       | 99    | 否           |
| 空头   | 买入     | other      | 101   | 是           |

#### 开仓（平仓）启用订单簿时

启用订单簿（`exit_pricing.use_order_book=True`）时，Freqtrade会提取订单簿中`exit_pricing.order_book_top`条目，  
并以`exit_pricing.order_book_top`指定的条目在配置的侧（`exit_pricing.price_side`）作为平仓价格。

1代表订单簿的最顶端，2代表第二个挂单，依此类推。

#### 不启用订单簿时

以下内容假设`side`为配置的`exit_pricing.price_side`（默认为`"ask"`）。

当未启用订单簿（`exit_pricing.use_order_book=False`）时，Freqtrade会选择行情中`side`方向的最佳价格，前提是该价格高于`last`成交价。  
如果`side`价格低于`last`，则会根据`exit_pricing.price_last_balance`在`side`价格和`last`价格之间插值，计算出平仓价格。

参数`exit_pricing.price_last_balance`的作用类似于建仓时的参数，值为`0.0`时使用`side`价格，值为`1.0`时使用`last`价格，两者之间的值进行插值。

### 市价单定价

使用市价单时，价格应配置为订单簿的“正确”一侧，以实现合理的定价检测。  
假设建仓和平仓都使用市价单，则应使用如下配置：

``` jsonc
  "order_types": {
    "entry": "market",
    "exit": "market"
    // ...
  },
  "entry_pricing": {
    "price_side": "other",
    // ...
  },
  "exit_pricing":{
    "price_side": "other",
    // ...
  },
```

当然，如果只在一侧使用限价单，则可以采用不同的定价组合。