# Summary：
本文件提供可直接复制和改写的 CTA 完整策略模板，并包含日内清仓、网格逻辑等可嵌入片段。它用于让 agent 根据用户描述快速生成完整策略文件，同时提醒哪些内容只是骨架或片段、需要嵌入标准 CTA 生命周期。

# 03 CTA Strategy Cookbook：可复制策略模板

## 1. 双均线策略

适合用户说“写一个均线金叉死叉策略”。

```python
import numpy as np

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


class AgentDoubleMaStrategy(CtaTemplate):
    author = "agent"

    fast_window = 10
    slow_window = 20
    fixed_size = 1
    price_add = 0.0

    fast_ma0 = 0.0
    fast_ma1 = 0.0
    slow_ma0 = 0.0
    slow_ma1 = 0.0

    parameters = ["fast_window", "slow_window", "fixed_size", "price_add"]
    variables = ["fast_ma0", "fast_ma1", "slow_ma0", "slow_ma1"]

    def on_init(self) -> None:
        self.write_log("策略初始化")
        self.bg = BarGenerator(self.on_bar)
        self.am = ArrayManager(size=max(100, self.slow_window + 5))
        self.load_bar(10)

    def on_start(self) -> None:
        self.write_log("策略启动")
        self.put_event()

    def on_stop(self) -> None:
        self.write_log("策略停止")
        self.put_event()

    def on_tick(self, tick: TickData) -> None:
        self.bg.update_tick(tick)

    def on_bar(self, bar: BarData) -> None:
        self.cancel_all()

        am = self.am
        am.update_bar(bar)
        if not am.inited:
            return

        fast_ma = am.sma(self.fast_window, array=True)
        slow_ma = am.sma(self.slow_window, array=True)

        self.fast_ma0 = fast_ma[-1]
        self.fast_ma1 = fast_ma[-2]
        self.slow_ma0 = slow_ma[-1]
        self.slow_ma1 = slow_ma[-2]

        cross_over = self.fast_ma0 > self.slow_ma0 and self.fast_ma1 <= self.slow_ma1
        cross_below = self.fast_ma0 < self.slow_ma0 and self.fast_ma1 >= self.slow_ma1

        buy_price = bar.close_price + self.price_add
        sell_price = bar.close_price - self.price_add

        if cross_over:
            if self.pos < 0:
                self.cover(buy_price, abs(self.pos))
            if self.pos <= 0:
                self.buy(buy_price, self.fixed_size)

        elif cross_below:
            if self.pos > 0:
                self.sell(sell_price, abs(self.pos))
            if self.pos >= 0:
                self.short(sell_price, self.fixed_size)

        self.put_event()

    def on_order(self, order: OrderData) -> None:
        pass

    def on_trade(self, trade: TradeData) -> None:
        self.put_event()

    def on_stop_order(self, stop_order: StopOrder) -> None:
        pass
```

## 2. 15 分钟布林突破 + ATR 移动止损

适合用户说“写一个趋势突破策略”。

```python
import numpy as np

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


class AgentBollAtrStrategy(CtaTemplate):
    author = "agent"

    boll_window = 20
    boll_dev = 2.0
    atr_window = 22
    atr_multiplier = 3.0
    fixed_size = 1

    boll_up = 0.0
    boll_down = 0.0
    boll_mid = 0.0
    atr_value = 0.0
    intra_trade_high = 0.0
    intra_trade_low = 0.0
    long_stop = 0.0
    short_stop = 0.0

    parameters = ["boll_window", "boll_dev", "atr_window", "atr_multiplier", "fixed_size"]
    variables = [
        "boll_up",
        "boll_down",
        "boll_mid",
        "atr_value",
        "intra_trade_high",
        "intra_trade_low",
        "long_stop",
        "short_stop",
    ]

    def on_init(self) -> None:
        self.write_log("策略初始化")
        self.bg = BarGenerator(self.on_bar, 15, self.on_15min_bar)
        self.am = ArrayManager(size=max(100, self.boll_window + self.atr_window + 5))
        self.load_bar(20)

    def on_start(self) -> None:
        self.write_log("策略启动")
        self.put_event()

    def on_stop(self) -> None:
        self.write_log("策略停止")
        self.put_event()

    def on_tick(self, tick: TickData) -> None:
        self.bg.update_tick(tick)

    def on_bar(self, bar: BarData) -> None:
        self.bg.update_bar(bar)

    def on_15min_bar(self, bar: BarData) -> None:
        self.cancel_all()

        am = self.am
        am.update_bar(bar)
        if not am.inited:
            return

        mid = am.sma(self.boll_window)
        if hasattr(am, "std"):
            std = am.std(self.boll_window)
        else:
            std = float(np.std(am.close[-self.boll_window:]))

        self.boll_mid = mid
        self.boll_up = mid + std * self.boll_dev
        self.boll_down = mid - std * self.boll_dev
        self.atr_value = am.atr(self.atr_window)

        if self.pos == 0:
            self.intra_trade_high = bar.high_price
            self.intra_trade_low = bar.low_price
            self.buy(self.boll_up, self.fixed_size, stop=True)
            self.short(self.boll_down, self.fixed_size, stop=True)

        elif self.pos > 0:
            self.intra_trade_high = max(self.intra_trade_high, bar.high_price)
            self.long_stop = self.intra_trade_high - self.atr_value * self.atr_multiplier
            self.sell(self.long_stop, abs(self.pos), stop=True)

        elif self.pos < 0:
            self.intra_trade_low = min(self.intra_trade_low, bar.low_price)
            self.short_stop = self.intra_trade_low + self.atr_value * self.atr_multiplier
            self.cover(self.short_stop, abs(self.pos), stop=True)

        self.put_event()

    def on_order(self, order: OrderData) -> None:
        pass

    def on_trade(self, trade: TradeData) -> None:
        self.put_event()

    def on_stop_order(self, stop_order: StopOrder) -> None:
        pass
```

注意：`am.std()` 在常见版本中可用；模板已加入 `hasattr(am, "std")` 分支，环境没有该方法时会退回 `float(np.std(am.close[-self.boll_window:]))`。

## 3. 日内时间过滤与收盘前清仓

在任意 CTA 的 `on_bar()` 或窗口回调中加入：

```python
from datetime import time

no_entry_after = time(14, 45)
force_exit_at = time(14, 55)
current_time = bar.datetime.time()

if current_time >= force_exit_at:
    self.cancel_all()
    if self.pos > 0:
        self.sell(bar.close_price, abs(self.pos))
    elif self.pos < 0:
        self.cover(bar.close_price, abs(self.pos))
    self.put_event()
    return

allow_entry = current_time < no_entry_after

if self.pos == 0 and allow_entry:
    # entry logic
    pass
```

生成日内策略时，避免夜盘品种误用固定 14:55。应让用户配置交易时段，或者把时间参数暴露为字符串。

## 4. TargetPosTemplate：目标仓位模板

适合“目标仓位到多少”的策略，减少手写开平逻辑。

```python
from vnpy_ctastrategy import TargetPosTemplate, TickData, BarData, BarGenerator, ArrayManager


class AgentTargetMaStrategy(TargetPosTemplate):
    author = "agent"

    fast_window = 10
    slow_window = 30
    max_pos = 1
    tick_add = 1.0

    fast_ma = 0.0
    slow_ma = 0.0

    parameters = ["fast_window", "slow_window", "max_pos", "tick_add"]
    variables = ["fast_ma", "slow_ma"]

    def on_init(self) -> None:
        self.write_log("策略初始化")
        self.bg = BarGenerator(self.on_bar)
        self.am = ArrayManager(size=max(100, self.slow_window + 5))
        self.load_bar(10)

    def on_start(self) -> None:
        self.write_log("策略启动")
        self.put_event()

    def on_stop(self) -> None:
        self.write_log("策略停止")
        self.put_event()

    def on_tick(self, tick: TickData) -> None:
        super().on_tick(tick)
        self.bg.update_tick(tick)

    def on_bar(self, bar: BarData) -> None:
        super().on_bar(bar)
        am = self.am
        am.update_bar(bar)
        if not am.inited:
            return

        self.fast_ma = am.sma(self.fast_window)
        self.slow_ma = am.sma(self.slow_window)

        if self.fast_ma > self.slow_ma:
            self.set_target_pos(self.max_pos)
        elif self.fast_ma < self.slow_ma:
            self.set_target_pos(-self.max_pos)
        else:
            self.set_target_pos(0)

        self.put_event()
```

`TargetPosTemplate` 会根据目标仓位自动发单。基类依赖最近一次 `on_tick` 或 `on_bar` 记录的价格来构造委托价，因此只用 K 线交易时应确保 `super().on_bar(bar)` 被调用；只用 Tick 交易时应确保 `super().on_tick(tick)` 被调用。`tick_add` 是价格加减值，不是“跳数”，实际品种中建议按 `get_pricetick()` 或用户参数设置。

基类会把 `target_pos` 加入变量列表；子类不要再次把 `target_pos` 写入 `variables`，避免 UI 变量重复。对复杂锁仓、净仓、交易所平今平昨细节仍需测试。

## 5. 网格策略骨架

网格比趋势策略更依赖活动委托管理。最低限度要维护网格中心、上下网格和当前持仓。

```python
class GridMixin:
    grid_step = 10.0
    grid_volume = 1
    max_pos = 5

    def trade_grid(self, bar):
        self.cancel_all()
        price = bar.close_price

        if self.pos == 0:
            self.buy(price - self.grid_step, self.grid_volume)
            self.short(price + self.grid_step, self.grid_volume)
        elif self.pos > 0:
            self.sell(price + self.grid_step, abs(self.pos))
            if self.pos < self.max_pos:
                self.buy(price - self.grid_step, self.grid_volume)
        elif self.pos < 0:
            self.cover(price - self.grid_step, abs(self.pos))
            if abs(self.pos) < self.max_pos:
                self.short(price + self.grid_step, self.grid_volume)
```

不要把这个片段当完整策略；要嵌入标准 CTA 模板，且必须限制最大仓位。

## 6. 写策略时的常见增强点

价格超价：

```python
pricetick = self.get_pricetick() or 1
self.buy(bar.close_price + 2 * pricetick, self.fixed_size)
```

避免重复反向开仓：

```python
if signal_long and self.pos <= 0:
    if self.pos < 0:
        self.cover(price, abs(self.pos))
    self.buy(price, self.fixed_size)
```

成交后记录入场价：

```python
def on_trade(self, trade: TradeData) -> None:
    if self.pos != 0:
        self.entry_price = trade.price
    self.put_event()
```

## 7. 策略部署说明模板

```text
保存为 `<运行目录>/strategies/agent_double_ma_strategy.py`。重启 Trader 或在回测模块里重载策略。打开 CTA 策略模块，添加 `AgentDoubleMaStrategy`，填写 `vt_symbol` 和参数。先初始化，确认 `inited=True`；再启动，确认 `trading=True`。
```
