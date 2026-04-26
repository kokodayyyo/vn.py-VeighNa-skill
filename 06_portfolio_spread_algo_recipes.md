# Summary：
本文件说明何时使用 PortfolioStrategy、SpreadTrading、AlgoTrading 和期权相关模块，而不是普通 CTA。它提供多合约、价差和执行算法的编码形态与边界；其中期权模块只做选型提示，不提供完整期权策略模板。

# 06 Portfolio, Spread & Algo Recipes：多合约、价差、算法执行

## 1. 什么时候不用 CTA

| 需求 | 不建议 | 推荐 |
|---|---|---|
| 两个以上合约一起算信号 | 多个 CTA 互相读文件 | PortfolioStrategy |
| 价差套利、腿比例、主动腿/被动腿 | 手写两个 CTA 同步下单 | SpreadTrading |
| 大单拆分、TWAP、冰山、追价 | 策略里循环下单 | AlgoTrading |
| 期权波动率、Greeks、Delta 对冲 | 普通 CTA | OptionMaster/Elite 期权模块 |

## 2. PortfolioStrategy 概念

组合策略创建实例时，合约品种用多个 `vt_symbol` 逗号分隔，中间不要空格：

```text
rb2405.SHFE,hot2405.SHFE,i2405.DCE
```

组合策略模板适合：

- 截面轮动。
- 多合约信号同步。
- 配对但不需要 SpreadTrading 完整价差引擎的逻辑。
- 目标仓位调仓。

典型导入：

```python
from typing import Dict

from vnpy.trader.object import TickData, BarData
from vnpy.trader.utility import ArrayManager
from vnpy_portfoliostrategy import StrategyTemplate, StrategyEngine
from vnpy_portfoliostrategy.utility import PortfolioBarGenerator
```

## 3. PortfolioStrategy 骨架

不同版本 API 细节可能略有差异，生成时优先以用户环境为准。以下是编码形态参考：

```python
from typing import Dict

from vnpy.trader.object import TickData, BarData
from vnpy.trader.utility import ArrayManager
from vnpy_portfoliostrategy import StrategyTemplate, StrategyEngine
from vnpy_portfoliostrategy.utility import PortfolioBarGenerator


class AgentPortfolioMaStrategy(StrategyTemplate):
    author = "agent"

    fast_window = 10
    slow_window = 30
    fixed_size = 1

    parameters = ["fast_window", "slow_window", "fixed_size"]
    variables = []

    def __init__(
        self,
        strategy_engine: StrategyEngine,
        strategy_name: str,
        vt_symbols: list[str],
        setting: dict,
    ) -> None:
        super().__init__(strategy_engine, strategy_name, vt_symbols, setting)
        self.pbg = PortfolioBarGenerator(self.on_bars)
        self.ams: Dict[str, ArrayManager] = {
            vt_symbol: ArrayManager() for vt_symbol in vt_symbols
        }

    def on_init(self) -> None:
        self.write_log("组合策略初始化")
        self.load_bars(10)

    def on_start(self) -> None:
        self.write_log("组合策略启动")
        self.put_event()

    def on_stop(self) -> None:
        self.write_log("组合策略停止")
        self.put_event()

    def on_tick(self, tick: TickData) -> None:
        self.pbg.update_tick(tick)

    def on_bars(self, bars: Dict[str, BarData]) -> None:
        self.cancel_all()

        all_inited = True
        for vt_symbol, bar in bars.items():
            am = self.ams[vt_symbol]
            am.update_bar(bar)
            if not am.inited:
                all_inited = False

        if not all_inited:
            return

        signals: Dict[str, int] = {}
        for vt_symbol, bar in bars.items():
            am = self.ams[vt_symbol]
            fast_ma = am.sma(self.fast_window)
            slow_ma = am.sma(self.slow_window)
            signals[vt_symbol] = 1 if fast_ma > slow_ma else -1

        for vt_symbol, signal in signals.items():
            pos = self.get_pos(vt_symbol)
            bar = bars[vt_symbol]
            price = bar.close_price

            if signal > 0 and pos <= 0:
                if pos < 0:
                    self.cover(vt_symbol, price, abs(pos))
                self.buy(vt_symbol, price, self.fixed_size)
            elif signal < 0 and pos >= 0:
                if pos > 0:
                    self.sell(vt_symbol, price, abs(pos))
                self.short(vt_symbol, price, self.fixed_size)

        self.put_event()

```

注意：官方 PortfolioStrategy 文档明确说明，多合约组合策略在回测时无法判断同一段 K 线内部不同合约委托成交的先后顺序，因此不提供 CTA 那种 `on_order()`/`on_trade()` 推送。组合策略应在 `on_bars()` 中通过 `get_pos(vt_symbol)`、`get_order(vt_orderid)`、`get_all_active_orderids()` 查询状态，不要把 CTA 的成交回调模式照搬过来。

`get_all_active_orderids()` 按活动委托号列表使用；若某些版本的文档或类型标注写成 `OrderData` 列表，以本地模板源码和运行返回值为准。

注意：不要在遍历 `bars.items()` 更新 `ArrayManager` 时遇到第一个未初始化合约就立即 `return`。这会导致同一时间切片中排在后面的合约没有机会更新指标缓存。应先更新当前 `bars` 中所有合约，再统一判断是否全部初始化。

如果用户环境报 `load_bars/get_pos/buy` 签名不一致，打开其本地 `vnpy_portfoliostrategy/template.py` 对齐。组合策略版本差异比 CTA 更常见。

`load_bars(days)` 在回测里按交易日理解，在实盘里按自然日理解；为了避免 `ArrayManager` 初始化不足，组合策略初始化天数宁可偏多。组合策略只支持 K 线回测，没有 `load_ticks()`。

### 3.1 PortfolioStrategy 目标仓位调仓模式

官方进阶章节提供了更贴合组合策略定位的目标仓位调仓模式：策略只计算目标仓位，然后交给 `rebalance_portfolio(bars)` 统一执行调仓。这个模式通常比手写 `buy/sell/short/cover` 更适合组合策略。`set_target()` 设置的是持续性目标状态，后续会一直保持，直到再次修改；`rebalance_portfolio(bars)` 只会处理当前 `bars` 字典里有 K 线切片的合约，避免非交易时段合约被误下单。

```python
from typing import Dict

from vnpy.trader.constant import Direction
from vnpy.trader.object import TickData, BarData
from vnpy.trader.utility import ArrayManager
from vnpy_portfoliostrategy import StrategyTemplate, StrategyEngine
from vnpy_portfoliostrategy.utility import PortfolioBarGenerator


class AgentPortfolioTargetStrategy(StrategyTemplate):
    author = "agent"

    fast_window = 10
    slow_window = 30
    fixed_size = 1
    price_add = 1.0

    parameters = ["fast_window", "slow_window", "fixed_size", "price_add"]
    variables = []

    def __init__(
        self,
        strategy_engine: StrategyEngine,
        strategy_name: str,
        vt_symbols: list[str],
        setting: dict,
    ) -> None:
        super().__init__(strategy_engine, strategy_name, vt_symbols, setting)
        self.pbg = PortfolioBarGenerator(self.on_bars)
        self.ams: Dict[str, ArrayManager] = {
            vt_symbol: ArrayManager(size=max(100, self.slow_window + 5))
            for vt_symbol in vt_symbols
        }

    def on_init(self) -> None:
        self.write_log("组合目标仓位策略初始化")
        self.load_bars(30)

    def on_start(self) -> None:
        self.write_log("组合目标仓位策略启动")
        self.put_event()

    def on_stop(self) -> None:
        self.write_log("组合目标仓位策略停止")
        self.put_event()

    def on_tick(self, tick: TickData) -> None:
        self.pbg.update_tick(tick)

    def on_bars(self, bars: Dict[str, BarData]) -> None:
        all_inited = True
        for vt_symbol, bar in bars.items():
            am = self.ams[vt_symbol]
            am.update_bar(bar)
            if not am.inited:
                all_inited = False

        if not all_inited:
            return

        for vt_symbol, bar in bars.items():
            am = self.ams[vt_symbol]
            fast_ma = am.sma(self.fast_window)
            slow_ma = am.sma(self.slow_window)

            if fast_ma > slow_ma:
                self.set_target(vt_symbol, self.fixed_size)
            elif fast_ma < slow_ma:
                self.set_target(vt_symbol, -self.fixed_size)
            else:
                self.set_target(vt_symbol, 0)

        # rebalance_portfolio 内部会按目标仓位执行调仓，并处理未成交委托；目标仓位模式下不要再在 on_bars 开头粗暴 cancel_all。
        self.rebalance_portfolio(bars)
        self.put_event()

    def calculate_price(self, vt_symbol: str, direction: Direction, reference: float) -> float:
        if direction == Direction.LONG:
            return reference + self.price_add
        return reference - self.price_add
```

## 4. SpreadTrading 使用思路

价差模块核心对象不是单个合约，而是“价差”：

- 多条腿合约。
- 主动腿/被动腿。
- 每条腿的交易乘数、价格乘数、合约比例。
- 价差行情由腿行情合成。
- 价差策略围绕价差价格下单。

适合：

- 跨期套利。
- 跨品种套利。
- 期现/相关品种价差。

agent 不应默认用两个 CTA 各自下单实现价差；那会造成成交不同步、撤单困难、风险暴露不可控。

## 5. Spread 策略骨架概念

官方文档示例以 `SpreadStrategyTemplate` 为基类，构造函数入参应与模板一致：`strategy_engine, strategy_name, spread, setting`。更接近可运行代码的骨架如下：

```python
import numpy as np

from vnpy.trader.utility import BarGenerator, ArrayManager
from vnpy_spreadtrading import (
    SpreadStrategyTemplate,
    SpreadAlgoTemplate,
    SpreadData,
    OrderData,
    TradeData,
    TickData,
    BarData,
)


class AgentSpreadMeanReversionStrategy(SpreadStrategyTemplate):
    author = "agent"

    window = 100
    dev = 2.0
    max_pos = 1
    payup = 10
    interval = 5

    spread_pos = 0.0
    spread_mean = 0.0
    spread_std = 0.0
    upper = 0.0
    lower = 0.0

    parameters = ["window", "dev", "max_pos", "payup", "interval"]
    variables = ["spread_pos", "spread_mean", "spread_std", "upper", "lower"]

    def __init__(self, strategy_engine, strategy_name: str, spread: SpreadData, setting: dict) -> None:
        super().__init__(strategy_engine, strategy_name, spread, setting)
        self.bg = BarGenerator(self.on_spread_bar)
        self.am = ArrayManager(size=max(100, self.window + 5))

    def on_init(self) -> None:
        self.write_log("价差策略初始化")
        self.load_bar(10)

    def on_start(self) -> None:
        self.write_log("价差策略启动")

    def on_stop(self) -> None:
        self.write_log("价差策略停止")
        self.put_event()

    def on_spread_data(self) -> None:
        tick = self.get_spread_tick()
        self.on_spread_tick(tick)

    def on_spread_tick(self, tick: TickData) -> None:
        self.bg.update_tick(tick)

    def on_spread_bar(self, bar: BarData) -> None:
        self.stop_all_algos()
        self.spread_pos = self.get_spread_pos()

        am = self.am
        am.update_bar(bar)
        if not am.inited:
            return

        self.spread_mean = am.sma(self.window)
        if hasattr(am, "boll"):
            self.upper, self.lower = am.boll(self.window, self.dev)
            self.spread_std = (self.upper - self.spread_mean) / self.dev if self.dev else 0.0
        elif hasattr(am, "std"):
            self.spread_std = am.std(self.window)
            self.upper = self.spread_mean + self.spread_std * self.dev
            self.lower = self.spread_mean - self.spread_std * self.dev
        else:
            self.spread_std = float(np.std(am.close[-self.window:]))
            self.upper = self.spread_mean + self.spread_std * self.dev
            self.lower = self.spread_mean - self.spread_std * self.dev

        if not self.spread_pos:
            if bar.close_price >= self.upper:
                self.start_short_algo(bar.close_price - self.payup, self.max_pos, payup=self.payup, interval=self.interval)
            elif bar.close_price <= self.lower:
                self.start_long_algo(bar.close_price + self.payup, self.max_pos, payup=self.payup, interval=self.interval)
        elif self.spread_pos > 0 and bar.close_price >= self.spread_mean:
            self.start_short_algo(bar.close_price - self.payup, abs(self.spread_pos), payup=self.payup, interval=self.interval)
        elif self.spread_pos < 0 and bar.close_price <= self.spread_mean:
            self.start_long_algo(bar.close_price + self.payup, abs(self.spread_pos), payup=self.payup, interval=self.interval)

        self.put_event()

    def on_spread_pos(self) -> None:
        self.spread_pos = self.get_spread_pos()
        self.put_event()

    def on_spread_algo(self, algo: SpreadAlgoTemplate) -> None:
        pass

    def on_order(self, order: OrderData) -> None:
        pass

    def on_trade(self, trade: TradeData) -> None:
        pass
```

该骨架仍应标记为“需要按本地 `vnpy_spreadtrading/template.py` 校准”。官方示例使用 `am.boll(window, dev)` 生成上下轨，但不同安装版本的 `ArrayManager` 指标方法可能不同；模板里已经写成三层兼容分支：优先 `am.boll()`，其次 `am.std()`，最后退回 `numpy.std(am.close[-window:])`。价差策略 API 比 CTA 更容易因版本差异变化，尤其是算法启动函数的参数位置、`payup/interval` 名称和回调细节。

注意：若发现官方文档示例和本地源码在 `start_long_algo()`/`start_short_algo()` 方向参数上不一致，应以本地 `vnpy_spreadtrading/template.py` 为准。当前源码中 `start_long_algo()` 对应 `Direction.LONG`，`start_short_algo()` 对应 `Direction.SHORT`。

## 6. AlgoTrading 使用思路

算法交易模块用于执行，而不是产生策略信号。典型算法：

- TWAP：按时间拆单。
- Iceberg：冰山委托。
- Sniper：狙击盘口。
- BestLimit：最优限价。
- Stop：停止委托。

使用建议：

- 策略负责产生目标方向、价格、总量、约束。
- AlgoTrading 负责拆单执行和盘口控制。
- 不要在 CTA 中用 `for` 循环瞬间发大量小单模拟 TWAP。

自定义算法的加载位置不要和 CTA 策略目录混淆。官方文档说明，用户搭建的算法需要放到 `algo_trading.algos` 目录中才能被识别加载；内置示例算法位于 `vnpy_algotrading.algos`。因此 agent 生成自定义 AlgoTrading 算法时，不应默认放到运行目录的 `strategies` 文件夹。

## 7. 风控模块

RiskManager 是事前风控，适合限制：

- 委托流控。
- 单笔委托数量。
- 总成交数量。
- 活动委托数。
- 撤单次数。
- 价格偏离。

写实盘部署建议时，应建议先开 RiskManager 和 PaperAccount/仿真环境测试。
