# Summary：
本文件定义 agent 编写 vn.py 代码时必须遵守的生成规则，包括默认模块选择、完整性要求、生命周期约束和禁止性写法。它用于降低代码能运行但行为错误的概率，确保输出符合 vn.py 的实际执行模型。

# 09 Code Generation Rules：agent 写 vn.py 代码必须遵守

## 1. 默认策略类型

除非用户明确说多合约、价差、期权、算法执行，否则默认生成 `CtaTemplate` 策略。

## 2. 代码完整性

必须输出完整文件级代码：imports、class、parameters、variables、生命周期回调、数据回调、委托/成交回调。

不要只给 `on_bar()` 片段，除非用户明确要求局部修改。

## 3. 导入规则

优先使用：

```python
from vnpy_ctastrategy import (
    CtaTemplate,
    StopOrder,
    TickData,
    BarData,
    TradeData,
    OrderData,
    BarGenerator,
    ArrayManager,
)
```

只有需要 `Interval`、`Direction`、`Offset` 等枚举时再从 `vnpy.trader.constant` 导入。

## 4. 参数类型规则

如果参数可能是小数，默认值必须写成 float：

```python
atr_multiplier = 3.0
price_add = 1.0
```

否则 UI 会按 int 类型限制输入。

## 5. 生命周期规则

必须有：

```python
def on_init(self):
    self.write_log("策略初始化")
    ...
    self.load_bar(10)
```

不要在 `on_init()` 中期待真实下单。初始化时 `trading=False`。

## 6. 指标规则

使用 `ArrayManager` 时必须：

```python
am.update_bar(bar)
if not am.inited:
    return
```

不要在 `am.inited` 之前计算指标或下单。

## 7. 下单规则

按 `self.pos` 控制仓位：

```python
if long_signal and self.pos <= 0:
    if self.pos < 0:
        self.cover(price, abs(self.pos))
    self.buy(price, self.fixed_size)
```

不要无视当前仓位持续重复开仓，除非策略明确是加仓模型且有最大仓位限制。

## 8. 撤单规则

K 线策略常在每根信号 K 开头调用：

```python
self.cancel_all()
```

如果是网格/做市策略，不应粗暴每个 tick 撤全部委托；需要维护活动委托。

## 9. UI 刷新规则

状态变量变化后调用：

```python
self.put_event()
```

通常放在 `on_bar()` 末尾和 `on_trade()` 中。

## 10. 时间规则

不要把中国期货所有品种都写死 09:00-15:00。夜盘、股指、商品、期权交易时段不同。若要日内清仓，把时间做成参数或明确示例。

## 11. 回测规则

不要编造合约参数。示例值必须标注“示例”。真实回测需要用户按合约填写 `size/pricetick/rate/slippage`。

## 12. 实盘安全规则

任何实盘部署建议都应包括：

- 先仿真或 PaperAccount。
- 开启 RiskManager。
- 小仓位试运行。
- 检查断线、拒单、涨跌停、夜盘重启。
- 确认策略逻辑持仓和账户真实持仓一致。

## 13. ScriptTrader 规则

ScriptTrader 的 `run(engine)` 里不要假设订阅后立即有 Tick/Contract。访问 `get_tick()`、`get_contract()`、`get_account()`、`get_position()` 返回值前先判空；循环条件应使用 `engine.strategy_active`，不要写不可停止的 `while True`。


## 14. PortfolioStrategy 规则

在 `on_bars()` 中先更新当前 `bars` 字典里所有合约的 `ArrayManager`，再统一判断 `am.inited`。不要在第一个未初始化合约处立即 `return`，否则同一切片中后续合约可能无法更新缓存。

PortfolioStrategy 不要生成 CTA 式 `on_order()`/`on_trade()` 用户回调；需要委托或成交状态时，在 `on_bars()` 中通过 `get_order()`、`get_pos()`、`get_all_active_orderids()` 查询。

## 15. SpreadTrading 规则

价差策略的信号应围绕价差对象运行，并优先用 `start_long_algo()` / `start_short_algo()` 调度算法。若使用布林带，官方示例里的 `am.boll()` 应写兼容分支：本地没有该方法时先用 `am.std()`，若 `am.std()` 也不存在，再用 `numpy.std(am.close[-window:])` 计算上下轨。若官方文档示例和本地源码的算法方向参数不一致，以本地 `vnpy_spreadtrading/template.py` 为准。

## 16. 不要做的事

- 不要承诺策略盈利。
- 不要把回测收益当实盘结果。
- 不要在不知道品种参数时填真实手续费/乘数。
- 不要忽略 `vt_symbol` 交易所后缀。
- 不要让多个 CTA 策略通过全局变量互相控制仓位。
- 不要把账户密码、授权码硬编码进脚本。
- 不要建议裸露 WebTrader/RPC 到公网。
