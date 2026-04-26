# Summary：
本文件把常见用户需求映射到 vn.py/VeighNa 的模块、模板和输出形态。它用于让 agent 在开始写代码前快速判断应使用 CTA、回测、数据、组合、价差、算法执行还是排错流程。
需求到 vn.py 代码路径
## 1. 先判断用户要什么

| 用户需求 | 优先模块 | 输出形态 | 必读文件 |
|---|---|---|---|
| 写单品种 CTA 策略 | `vnpy_ctastrategy.CtaTemplate` | `.py` 策略类 | `02`、`03`、`09` |
| 写双均线、突破、ATR 止损、网格 | CTA cookbook | 双均线和突破模板可直接改写；网格为骨架 | `03` |
| 写回测脚本或解释回测参数 | `BacktestingEngine` 或 CtaBacktester UI | Python 脚本/配置说明 | `04` |
| 导入/读取历史数据 | `get_database()`、DataManager | 脚本/步骤 | `05` |
| 连 CTP、订阅行情、启动 Trader | `MainEngine` + gateway + app | 启动脚本/配置说明 | `01`、`05` |
| 多合约组合策略 | `vnpy_portfoliostrategy.StrategyTemplate` | 组合策略骨架；优先考虑目标仓位调仓模式 | `06` |
| 价差套利 | SpreadTrading | 价差配置/策略骨架 | `06` |
| TWAP/Iceberg/BestLimit 等执行算法 | AlgoTrading | 使用说明/脚本 | `06` |
| 轻量脚本交易，不做完整 app | ScriptTrader | 脚本函数 | `07` |
| 策略不显示、不成交、不刷新 | 排错 | 排查清单 | `08` |
| 让 agent 生成稳健代码 | 代码规则 | 生成约束 | `09`、`10` |

下面是精简版，可直接保存为 `SKILL.md`。内容覆盖你上传的 01-10 文档：启动运行、CTA、回测、数据、组合/价差/算法、脚本交易、排错、生成规则和 API 速查。         


# VeighNa/vn.py Agent Skill

本技能用于指导 Agent 生成、检查和修复 VeighNa/vn.py 代码。适用范围包括 Trader 启动、CTA 策略、回测、历史数据、gateway、数据库、组合策略、价差策略、算法执行、ScriptTrader 脚本和常见报错排查。

使用 01-10 文档时，先按任务类型选择文件：启动和运行目录看 `01`；CTA 策略看 `02` 和 `03`；回测看 `04`；数据、gateway、数据库看 `05`；多合约、价差、算法执行看 `06`；脚本交易看 `07`；排错看 `08`；生成代码前检查 `09`；不确定 API、字段、函数签名时查 `10`。

推荐流程：先判断用户需求属于 CTA、回测、数据、组合、价差、算法、脚本交易还是排错；再选择对应模板；最后用 `09` 和 `10` 检查生命周期、导入、参数、持仓、下单、撤单、数据源和 API 是否正确。

## 1. 总原则

- 默认生成 `CtaTemplate` 策略，除非用户明确要求多合约、价差、算法执行或脚本交易。
- 用户要求写策略时，默认输出完整 `.py` 文件级代码，不只输出 `on_bar()` 片段。
- 不要编造手续费、滑点、合约乘数、最小价格变动、交易时间等真实参数。
- 不要承诺收益，不要把回测结果当作实盘保证。
- 若文档、指南和用户本地源码冲突，以用户本地源码为准。

## 2. 任务路由

- 单品种策略：使用 `vnpy_ctastrategy.CtaTemplate`。
- 双均线、突破、ATR 止损、普通网格：优先 CTA。
- 回测：使用 `BacktestingEngine` 或 CtaBacktester 参数说明。
- 历史数据、CSV、数据库、datafeed：使用 `get_database()`、DataManager 或 datafeed。
- 多合约共同计算信号：使用 PortfolioStrategy。
- 跨期、跨品种、期现价差：使用 SpreadTrading。
- TWAP、冰山、追价、大单拆分：使用 AlgoTrading。
- 临时订阅、临时下单、巡检脚本：使用 ScriptTrader。
- 策略不显示、不成交、不刷新、回测无成交：进入 Debugging Playbook。

## 3. 启动和运行目录

自写策略通常放在运行目录下的 `strategies` 文件夹。文件名使用下划线风格，例如 `double_ma_strategy.py`；类名使用驼峰式，例如 `DoubleMaStrategy`。UI 下拉框显示的是策略类名，不是文件名。

不要直接覆盖 `.vntrader` 配置文件，除非用户明确要求并理解风险。

实盘启动顺序：启动 Trader，连接接口，等待登录和合约查询完成，确认行情订阅正常，添加策略，初始化策略，启动策略，小仓位观察。

## 4. CTA 代码规则

标准导入优先使用：

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
````

只有需要枚举时再导入：

```python
from vnpy.trader.constant import Interval, Direction, Offset
```

CTA 策略必须包含：

* `on_init`
* `on_start`
* `on_stop`
* `on_tick`
* `on_bar`
* `on_order`
* `on_trade`
* `on_stop_order`

`parameters` 只放用户可配置参数，`variables` 只放运行状态变量。复杂对象如 `BarGenerator`、`ArrayManager`、字典、列表不要放入 `parameters` 或 `variables`。

可能需要小数的参数，默认值必须写成 float，例如：

```python
price_add = 1.0
atr_multiplier = 3.0
```

## 5. CTA 生命周期

`on_init()` 中写初始化日志并加载历史数据：

```python
def on_init(self) -> None:
    self.write_log("策略初始化")
    self.load_bar(10)
```

不要在 `on_init()` 中期待真实下单。初始化阶段 `trading=False`。

状态变量变化后调用：

```python
self.put_event()
```

通常放在 `on_bar()` 末尾、`on_trade()`、`on_start()`、`on_stop()` 中。

## 6. BarGenerator 和 ArrayManager

Tick 合成 K 线：

```python
self.bg = BarGenerator(self.on_bar)

def on_tick(self, tick: TickData) -> None:
    self.bg.update_tick(tick)
```

1 分钟 K 线合成 15 分钟 K 线：

```python
self.bg = BarGenerator(self.on_bar, 15, self.on_15min_bar)

def on_bar(self, bar: BarData) -> None:
    self.bg.update_bar(bar)

def on_15min_bar(self, bar: BarData) -> None:
    pass
```

使用 `ArrayManager` 必须先更新，再判断初始化：

```python
am = self.am
am.update_bar(bar)

if not am.inited:
    return
```

不要在 `am.inited` 之前计算指标或下单。

## 7. CTA 下单规则

必须按 `self.pos` 控制仓位，避免重复开仓。

多头信号：

```python
if long_signal and self.pos <= 0:
    if self.pos < 0:
        self.cover(price, abs(self.pos))
    self.buy(price, self.fixed_size)
```

空头信号：

```python
if short_signal and self.pos >= 0:
    if self.pos > 0:
        self.sell(price, abs(self.pos))
    self.short(price, self.fixed_size)
```

普通 K 线策略可在信号 K 线开始时调用：

```python
self.cancel_all()
```

网格、做市、盘口类策略不要每个 Tick 粗暴全撤，应维护活动委托。

## 8. 回测规则

回测脚本必须显式设置：

* `vt_symbol`
* `interval`
* `start`
* `end`
* `rate`
* `slippage`
* `size`
* `pricetick`
* `capital`

手续费、滑点、合约乘数、最小价格变动必须由用户按真实合约确认。示例值必须标注“示例”。

回测前检查：

* 策略类可导入。
* 本地数据库有历史数据。
* `vt_symbol` 与数据库一致。
* `interval` 与策略逻辑一致。
* `load_bar()` 天数足够初始化指标。
* `fixed_size` 不为 0。

## 9. 数据和 gateway

`vt_symbol` 格式为：

```text
symbol.EXCHANGE
```

示例：

```text
rb2405.SHFE
IF888.CFFEX
cu888.SHFE
```

数据库读取使用：

```python
from vnpy.trader.database import get_database
database = get_database()
```

若用户明确要使用本地清洗后的数据初始化 CTA，使用：

```python
self.load_bar(30, use_database=True)
```

`load_bar(use_database=False)` 不是无条件按 gateway、datafeed、database 完整轮询。应按当前 vn.py 源码逻辑理解，并以本地版本为准。

## 10. PortfolioStrategy 规则

多合约共同计算信号时，优先使用 PortfolioStrategy，不要让多个 CTA 互相读写文件。

组合策略中不要照搬 CTA 的 `on_order()`、`on_trade()` 用户回调。应在 `on_bars()` 中查询：

```python
self.get_pos(vt_symbol)
self.get_order(vt_orderid)
self.get_all_active_orderids()
```

在 `on_bars()` 中应先更新当前 `bars` 字典里所有合约的 `ArrayManager`，再统一判断是否全部初始化。不要遇到第一个未初始化合约就立即 `return`。

目标仓位模式优先使用：

```python
self.set_target(vt_symbol, target_pos)
self.rebalance_portfolio(bars)
```

## 11. SpreadTrading 规则

价差套利优先使用 SpreadTrading，不要默认用两个 CTA 各自下单。

SpreadTrading 适合：

* 跨期套利
* 跨品种套利
* 期现价差
* 多腿价差

价差策略围绕价差对象交易，而不是直接围绕单腿合约交易。SpreadTrading 版本差异较多，生成代码时应提醒用户以本地 `vnpy_spreadtrading/template.py` 为准。

价差策略通常需要实现：

```python
on_init()
on_start()
on_stop()
on_spread_data()
on_spread_tick()
on_spread_bar()
on_spread_pos()
on_spread_algo()
```

不用的回调也应写成 `pass`。

## 12. AlgoTrading 规则

如果用户需求是 TWAP、冰山、追价、最优价、大单拆分，应优先使用 AlgoTrading，而不是在 CTA 中写循环下单。

不要生成不可控的循环下单逻辑。

## 13. ScriptTrader 规则

ScriptTrader 文件不是策略类，而是：

```python
def run(engine: ScriptEngine) -> None:
    pass
```

循环必须使用：

```python
while engine.strategy_active:
    pass
```

不要写不可停止的：

```python
while True:
    pass
```

订阅后不要假设马上有 Tick、合约、账户、持仓。访问前必须判空：

```python
tick = engine.get_tick(vt_symbol)
if tick is None:
    continue
```

## 14. 排错规则

策略类不显示，检查：

* 是否放在运行目录 `strategies` 文件夹。
* 文件是否为 `.py`。
* 类是否继承正确模板。
* 顶部 import 是否报错。
* 类名是否重复。
* 是否重启或重载策略。

初始化完成但不发信号，优先检查：

* `ArrayManager` 是否未初始化。
* `load_bar()` 天数是否太少。
* 历史数据是否足够。
* 回测周期是否与策略周期一致。

下单函数调用但没有委托，检查：

* 策略是否已启动。
* 是否在 `on_init()` 中下单。
* 价格是否超出涨跌停。
* 数量是否有效。
* 是否在交易时段。
* RiskManager 是否拦截。
* 资金或持仓是否足够。

## 15. 实盘安全

任何实盘建议必须包含：

* 先仿真或 PaperAccount。
* 开启 RiskManager。
* 小仓位试运行。
* 检查断线、拒单、涨跌停、夜盘重启。
* 确认策略逻辑持仓与账户真实持仓一致。

不得承诺收益。

## 16. 输出要求

生成代码时：

* 给完整代码。
* 说明保存路径。
* 说明运行方式。
* 标注示例参数。
* 提醒版本差异。
* 涉及实盘时给风险提示。

排错时：

* 先给最可能原因。
* 再给最短修复代码。
* 不要泛泛解释。

```
```
