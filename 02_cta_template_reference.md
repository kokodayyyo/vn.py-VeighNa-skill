# Summary：
本文件整理 CtaTemplate 的标准导入、类结构、生命周期回调和下单方法。它是 agent 编写单品种 CTA 策略时的基准参考，用于保证代码结构完整、参数变量可被 UI 识别。

# 02 CTA Template Reference：CtaTemplate 编码基准

## 1. 标准导入

推荐写法：

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
from vnpy.trader.constant import Interval
```

如果不需要 `np` 或 `Interval`，可以删掉。`vnpy_ctastrategy.__init__` 已导出常用类型，写策略时更简洁。

## 2. 最小 CTA 策略骨架

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


class MyStrategy(CtaTemplate):
    author = "agent"

    fast_window = 10
    slow_window = 20
    fixed_size = 1

    fast_ma = 0.0
    slow_ma = 0.0

    parameters = ["fast_window", "slow_window", "fixed_size"]
    variables = ["fast_ma", "slow_ma"]

    def on_init(self) -> None:
        self.write_log("策略初始化")
        self.bg = BarGenerator(self.on_bar)
        self.am = ArrayManager()
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

        self.fast_ma = am.sma(self.fast_window)
        self.slow_ma = am.sma(self.slow_window)

        # TODO: trading logic
        self.put_event()

    def on_order(self, order: OrderData) -> None:
        pass

    def on_trade(self, trade: TradeData) -> None:
        self.put_event()

    def on_stop_order(self, stop_order: StopOrder) -> None:
        pass
```

说明：上面的骨架把 `BarGenerator` 和 `ArrayManager` 放在 `on_init()` 中创建，适合生成简洁模板。官方文档示例更常见的写法是在策略类 `__init__()` 中创建工具对象，在 `on_init()` 中只写初始化日志并调用 `load_bar()`。如果用户要求严格贴近官方示例，优先使用下面这种结构：

```python
from vnpy_ctastrategy import CtaTemplate, BarGenerator, ArrayManager


class MyStrategy(CtaTemplate):
    author = "agent"

    def __init__(self, cta_engine, strategy_name: str, vt_symbol: str, setting: dict) -> None:
        super().__init__(cta_engine, strategy_name, vt_symbol, setting)
        self.bg = BarGenerator(self.on_bar)
        self.am = ArrayManager()

    def on_init(self) -> None:
        self.write_log("策略初始化")
        self.load_bar(10)
```

两种写法都不应把 `bg/am` 写入 `parameters` 或 `variables`；它们是运行期对象，不是 UI 参数。

## 3. 生命周期

| 方法 | 调用时机 | 典型用途 |
|---|---|---|
| `on_init()` | 初始化策略 | 写日志、创建工具、加载历史数据 |
| `on_start()` | 启动策略 | 写日志、刷新 UI |
| `on_stop()` | 停止策略 | 写日志、刷新 UI |
| `on_tick(tick)` | 收到 Tick | 推给 `BarGenerator` 或直接 Tick 交易 |
| `on_bar(bar)` | 收到 K 线 | 更新指标、发出交易信号 |
| `on_order(order)` | 委托状态更新 | 跟踪活动委托、拒单 |
| `on_trade(trade)` | 成交更新 | 记录成交、同步变量、刷新 UI |
| `on_stop_order(stop_order)` | 停止单更新 | 跟踪本地/交易所停止单 |

## 4. 参数与变量

```python
fast_window = 10          # int
threshold = 1.5           # float，若后续可能填小数，默认值必须是 1.0 这种 float
enabled = True            # bool
symbol_tag = "main"       # str

parameters = ["fast_window", "threshold", "enabled", "symbol_tag"]
variables = ["fast_ma", "slow_ma", "entry_price"]
```

`parameters` 和 `variables` 只适合 `str/int/float/bool`。复杂对象放在 `__init__` 或 `on_init()` 中，不要写入列表。

## 5. BarGenerator 用法

Tick 合成 1 分钟 K 线：

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
    ...
```

小时线：

```python
from vnpy.trader.constant import Interval
self.bg = BarGenerator(self.on_bar, 2, self.on_2hour_bar, Interval.HOUR)
```

分钟窗口必须能整除 60，如 2、3、5、6、10、15、20、30。小时窗口没有这个限制。

## 6. ArrayManager 用法

```python
self.am = ArrayManager(size=100)

am = self.am
am.update_bar(bar)
if not am.inited:
    return

ma = am.sma(20)
ma_array = am.sma(20, array=True)
rsi = am.rsi(14)
atr = am.atr(22)
cci = am.cci(20)
macd, signal, hist = am.macd(12, 26, 9)
```

常用数据序列：

```python
am.open
am.high
am.low
am.close
am.volume
am.open_interest
```

## 7. 下单函数

| 函数 | 方向/开平 | 含义 |
|---|---|---|
| `buy(price, volume, stop=False, lock=False, net=False)` | LONG/OPEN | 买入开仓 |
| `sell(price, volume, stop=False, lock=False, net=False)` | SHORT/CLOSE | 卖出平多 |
| `short(price, volume, stop=False, lock=False, net=False)` | SHORT/OPEN | 卖出开空 |
| `cover(price, volume, stop=False, lock=False, net=False)` | LONG/CLOSE | 买入平空 |

规则：

- `trading=False` 时下单函数返回空列表，不会真实下单。
- `stop=True` 会转为停止单。接口不支持时由本地停止单实现。
- `lock=True` 用于锁仓转换。
- `net=True` 用于净仓转换，不能和 `lock=True` 同时使用。
- 国内期货有开平仓；股票/外盘期货常用净持仓逻辑时只需 buy/sell。

## 8. 功能函数

```python
self.cancel_order(vt_orderid)
self.cancel_all()
self.write_log("message")
self.put_event()
self.send_email("message")
self.load_bar(days=10)
self.load_bar(30, interval=Interval.HOUR, callback=self.on_hour_bar)
self.load_tick(3)
self.get_pricetick()
self.get_size()
self.get_engine_type()
```

关键限制：

- `put_event()` 只有 `inited=True` 后刷新 UI。
- `send_email()` 需要邮箱配置，且 `inited=True`。
- `sync_data()` 通常由引擎调用；不要在普通策略逻辑里频繁调用。

## 9. 数据对象常用字段

`TickData` 常用：

```python
tick.symbol
tick.exchange
tick.vt_symbol
tick.datetime
tick.last_price
tick.volume
tick.open_interest
tick.bid_price_1
tick.ask_price_1
tick.bid_volume_1
tick.ask_volume_1
tick.limit_up
tick.limit_down
```

`BarData` 常用：

```python
bar.symbol
bar.exchange
bar.vt_symbol
bar.datetime
bar.interval
bar.open_price
bar.high_price
bar.low_price
bar.close_price
bar.volume
bar.open_interest
```

`OrderData` 常用：

```python
order.vt_orderid
order.direction
order.offset
order.price
order.volume
order.traded
order.status
order.is_active()
```

`TradeData` 常用：

```python
trade.vt_orderid
trade.vt_tradeid
trade.direction
trade.offset
trade.price
trade.volume
trade.datetime
```
