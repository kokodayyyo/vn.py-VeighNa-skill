# Summary：
本文件提供 vn.py 常用类、方法、数据结构和调用顺序的速查表。它用于让 agent 在写策略、回测、数据脚本和交易脚本时快速查找核心 API，而不必反复阅读长文档。

# 10 API Cheatsheet：写代码速查

## 1. CTA 核心类

```text
CtaTemplate(
    cta_engine,
    strategy_name: str,
    vt_symbol: str,
    setting: dict,
)
```

类属性：

```text
author: str
parameters: list[str]
variables: list[str]
```

实例属性：

```text
self.cta_engine
self.strategy_name
self.vt_symbol
self.inited
self.trading
self.pos
```

## 2. CTA 回调

```text
on_init() -> None
on_start() -> None
on_stop() -> None
on_tick(tick: TickData) -> None
on_bar(bar: BarData) -> None
on_order(order: OrderData) -> None
on_trade(trade: TradeData) -> None
on_stop_order(stop_order: StopOrder) -> None
```

## 3. CTA 主动函数

```text
buy(price: float, volume: float, stop: bool = False, lock: bool = False, net: bool = False) -> list
sell(price: float, volume: float, stop: bool = False, lock: bool = False, net: bool = False) -> list
short(price: float, volume: float, stop: bool = False, lock: bool = False, net: bool = False) -> list
cover(price: float, volume: float, stop: bool = False, lock: bool = False, net: bool = False) -> list
send_order(direction, offset, price, volume, stop=False, lock=False, net=False) -> list
cancel_order(vt_orderid: str) -> None
cancel_all() -> None
```

## 4. CTA 工具函数

```text
write_log(msg: str) -> None
get_engine_type() -> EngineType
get_pricetick() -> float
get_size() -> int
load_bar(days: int, interval=Interval.MINUTE, callback=None, use_database=False) -> None
load_tick(days: int) -> None
put_event() -> None
send_email(msg: str) -> None
sync_data() -> None
```

## 5. BarGenerator

```text
BarGenerator(
    on_bar: Callable,
    window: int = 0,
    on_window_bar: Callable | None = None,
    interval: Interval = Interval.MINUTE,
    daily_end: time | None = None,
)

bg.update_tick(tick)
bg.update_bar(bar)
bg.generate()
```

用途：

- Tick -> 1 分钟 K 线。
- 1 分钟 K 线 -> X 分钟/X 小时/X 日 K 线。

## 6. ArrayManager

```text
ArrayManager(size: int = 100)

am.update_bar(bar)
am.inited
am.open
am.high
am.low
am.close
am.volume
am.open_interest
```

常用指标：

```text
am.sma(n, array=False)
am.ema(n, array=False)
am.wma(n, array=False)
am.kama(n, array=False)
am.atr(n, array=False)
am.rsi(n, array=False)
am.cci(n, array=False)
am.macd(fast_period, slow_period, signal_period, array=False)
```

不同版本的指标函数可能增减。若报属性不存在，打开本地 `vnpy/trader/utility.py` 的 `ArrayManager` 查看。

价差策略的官方示例会用到 `am.boll(window, dev)`，但部分安装版本可能没有该方法。生成通用模板时应写兼容分支：优先 `am.boll()`，其次 `am.std()`，最后用 `numpy.std(am.close[-window:])`。

## 7. TickData/BarData 字段

```text
TickData:
    symbol, exchange, vt_symbol, datetime, name
    volume, turnover, open_interest
    last_price, last_volume
    limit_up, limit_down
    open_price, high_price, low_price, pre_close
    bid_price_1..5, ask_price_1..5
    bid_volume_1..5, ask_volume_1..5

BarData:
    symbol, exchange, vt_symbol, datetime, interval
    volume, turnover, open_interest
    open_price, high_price, low_price, close_price
```

## 8. 数据库

```text
from vnpy.trader.database import get_database

database = get_database()

database.load_bar_data(symbol, exchange, interval, start, end)
database.load_tick_data(symbol, exchange, start, end)
database.save_bar_data(bars)
database.save_tick_data(ticks)
database.delete_bar_data(symbol, exchange, interval)
database.delete_tick_data(symbol, exchange)
```

## 9. BacktestingEngine

```text
engine = BacktestingEngine()
engine.set_parameters(
    vt_symbol: str,
    interval: Interval,
    start: datetime,
    rate: float,
    slippage: float,
    size: float,
    pricetick: float,
    capital: int = 0,
    end: datetime | None = None,
    mode: BacktestingMode = BacktestingMode.BAR,
    risk_free: float = 0,
    annual_days: int = 240,
    half_life: int = 120,
)
engine.add_strategy(strategy_class, setting: dict)
engine.load_data()
engine.run_backtesting()
engine.calculate_result()
engine.calculate_statistics()
engine.show_chart()
```

实际脚本中应显式传入 `capital`，不要依赖默认值；源码中 `set_parameters()` 的默认 `capital=0`，不适合直接计算绩效。

## 10. 常用枚举

```text
from vnpy.trader.constant import Direction, Offset, Exchange, Interval, OrderType, Status

Direction.LONG
Direction.SHORT
Offset.OPEN
Offset.CLOSE
Interval.MINUTE
Interval.HOUR
Interval.DAILY
OrderType.LIMIT
OrderType.MARKET
```

交易所枚举常见：

```text
Exchange.SHFE
Exchange.CFFEX
Exchange.DCE
Exchange.CZCE
Exchange.INE
Exchange.SSE
Exchange.SZSE
```

## 11. ScriptEngine

```text
run(engine: ScriptEngine) -> None
engine.strategy_active
engine.connect_gateway(setting: dict, gateway_name: str) -> None
engine.subscribe(vt_symbols: Sequence[str]) -> None
engine.get_tick(vt_symbol: str, use_df: bool = False)
engine.get_contract(vt_symbol: str, use_df: bool = False)
engine.get_account(vt_accountid: str, use_df: bool = False)
engine.get_position(vt_positionid: str, use_df: bool = False)
engine.buy(vt_symbol, price, volume, order_type=OrderType.LIMIT) -> str
engine.sell(vt_symbol, price, volume, order_type=OrderType.LIMIT) -> str
engine.short(vt_symbol, price, volume, order_type=OrderType.LIMIT) -> str
engine.cover(vt_symbol, price, volume, order_type=OrderType.LIMIT) -> str
engine.cancel_order(vt_orderid: str) -> None
engine.write_log(msg: str) -> None
engine.send_email(msg: str) -> None
```


## 12. PortfolioStrategy

```text
StrategyTemplate(
    strategy_engine,
    strategy_name: str,
    vt_symbols: list[str],
    setting: dict,
)

on_init() -> None
on_start() -> None
on_stop() -> None
on_tick(tick: TickData) -> None
on_bars(bars: dict[str, BarData]) -> None

load_bars(days: int, interval=Interval.MINUTE) -> None
get_pos(vt_symbol: str) -> int
get_order(vt_orderid: str)
get_all_active_orderids() -> list[str]  # 活动委托号列表；若文档或类型标注不一致，以本地运行返回值为准
buy(vt_symbol: str, price: float, volume: float, lock=False, net=False) -> list[str]
sell(vt_symbol: str, price: float, volume: float, lock=False, net=False) -> list[str]
short(vt_symbol: str, price: float, volume: float, lock=False, net=False) -> list[str]
cover(vt_symbol: str, price: float, volume: float, lock=False, net=False) -> list[str]
set_target(vt_symbol: str, target: int) -> None
get_target(vt_symbol: str) -> int
rebalance_portfolio(bars: dict[str, BarData]) -> None
calculate_price(vt_symbol: str, direction: Direction, reference: float) -> float
```

PortfolioStrategy 不提供 CTA 式 `on_order()`/`on_trade()` 推送；组合策略应在 `on_bars()` 中查询持仓和委托状态。`load_bars()` 只加载 K 线；组合策略不支持 Tick 回测。目标仓位模式下，`set_target()` 是持续状态，`rebalance_portfolio(bars)` 只处理当前 `bars` 字典中有 K 线的合约。`get_all_active_orderids()` 按委托号字符串列表使用。

## 13. SpreadTrading 策略

```text
SpreadStrategyTemplate(
    strategy_engine,
    strategy_name: str,
    spread: SpreadData,
    setting: dict,
)

on_init() -> None
on_start() -> None
on_stop() -> None
on_spread_data() -> None
on_spread_tick(tick: TickData) -> None
on_spread_bar(bar: BarData) -> None
on_spread_pos() -> None
on_spread_algo(algo: SpreadAlgoTemplate) -> None
on_order(order: OrderData) -> None
on_trade(trade: TradeData) -> None

get_spread_tick() -> TickData
get_spread_pos() -> float
load_bar(days: int, interval=Interval.MINUTE) -> None
load_tick(days: int) -> None
start_long_algo(price: float, volume: float, payup: int, interval: int, lock=False, extra=None) -> str
start_short_algo(price: float, volume: float, payup: int, interval: int, lock=False, extra=None) -> str
stop_algo(algoid: str) -> None
stop_all_algos() -> None
```

SpreadTrading 的策略信号围绕“价差”运行；通常用 `start_long_algo()`/`start_short_algo()` 调度价差算法，而不是直接对每条腿手写同步下单。

`load_tick()` 只适合已经通过 DataRecorder 录制并保存在数据库中的 `xx-spread.LOCAL` 价差 Tick 数据；普通价差策略优先使用 `load_bar()`。


> Spread 算法方向说明：`start_long_algo()` 应对应买入价差/`Direction.LONG`，`start_short_algo()` 应对应卖出价差/`Direction.SHORT`；若官方文档示例和本地源码不一致，以本地源码为准。
