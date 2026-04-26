# Summary：
本文件说明 ScriptTrader 的适用边界，并提供 MainEngine 脚本化调用、事件监听和服务化任务的常见写法。它适合 agent 生成轻量订阅、临时下单、自动化脚本和非完整策略生命周期的代码。

# 07 ScriptTrader & Service Recipes：脚本交易、服务化、临时任务

## 1. 什么时候用 ScriptTrader

适合：

- 快速订阅行情。
- 临时下单/撤单。
- 小型自动化脚本。
- 不需要完整策略生命周期和 UI 实例管理的任务。

不适合：

- 长期稳定运行的复杂策略。
- 需要回测同一套代码。
- 多实例参数管理。

## 2. ScriptEngine 官方模板

ScriptTrader 的脚本策略文件不是 `CtaTemplate` 类，而是定义一个 `run(engine: ScriptEngine)` 函数。官方模板的核心模式是订阅一组 `vt_symbol`，然后用 `engine.strategy_active` 控制循环退出。

```python
from time import sleep
from vnpy_scripttrader import ScriptEngine


def run(engine: ScriptEngine) -> None:
    vt_symbols = ["sc2209.INE", "sc2203.INE"]

    engine.subscribe(vt_symbols)

    for vt_symbol in vt_symbols:
        contract = engine.get_contract(vt_symbol)
        engine.write_log(f"合约信息：{contract}")

    while engine.strategy_active:
        for vt_symbol in vt_symbols:
            tick = engine.get_tick(vt_symbol)
            if tick is None:
                continue
            engine.write_log(f"最新行情：{tick}")

        sleep(3)
```

订阅后第一段时间 `get_tick()`、`get_contract()` 可能返回 `None`，实际脚本应做空值判断，避免刚启动就异常退出。

生成 ScriptTrader 文件时，优先使用这种 `run(engine)` 结构。它适合轻量脚本、巡检、篮子订阅、临时交易任务；如果用户要回测或 UI 参数化管理，仍应转为 CTA/Portfolio。

## 3. Jupyter/CLI 模式

官方文档还提供了 `init_cli_trading()`，它封装 MainEngine 和 ScriptEngine，适合 Notebook 或命令行交互式交易。

```python
from vnpy_ctp import CtpGateway
from vnpy_scripttrader import init_cli_trading

engine = init_cli_trading([CtpGateway])

setting = {
    "用户名": "xxxx",
    "密码": "xxxx",
    "经纪商代码": "9999",
    "交易服务器": "180.168.146.187:10202",
    "行情服务器": "180.168.146.187:10212",
    "产品名称": "simnow_client_test",
    "授权编码": "0000000000000000",
}
engine.connect_gateway(setting, "CTP")
engine.subscribe(["rb2209.SHFE", "rb2210.SHFE"])
```

不要把账号、密码、授权码硬编码进长期仓库；示例中只展示字段形态。

## 4. ScriptEngine 常用函数

```python
engine.connect_gateway(setting, "CTP")
engine.subscribe(["rb2210.SHFE"])

tick = engine.get_tick("rb2210.SHFE")
contract = engine.get_contract("rb2210.SHFE")
account = engine.get_account("CTP.189672")
position = engine.get_position("CTP.rb2210.SHFE.多")

vt_orderid = engine.buy("rb2210.SHFE", price=4200, volume=1)
engine.cancel_order(vt_orderid)

engine.write_log("message")
engine.send_email("message")
```

`buy/sell/short/cover` 的参数形态是 `vt_symbol, price, volume, order_type=OrderType.LIMIT`，返回本地委托号 `vt_orderid`。

## 5. MainEngine 低层调用

直接使用 MainEngine 适合自定义服务或特殊集成，不是 ScriptTrader 文档的首选模板。使用它时要自行处理连接、事件、合约查询、异常、撤单和退出。

```python
from vnpy.event import EventEngine
from vnpy.trader.engine import MainEngine
from vnpy.trader.object import SubscribeRequest, OrderRequest
from vnpy.trader.constant import Exchange, Direction, Offset, OrderType

from vnpy_ctp import CtpGateway


event_engine = EventEngine()
main_engine = MainEngine(event_engine)
main_engine.add_gateway(CtpGateway)

# main_engine.connect(setting, "CTP")

req = SubscribeRequest(symbol="rb2405", exchange=Exchange.SHFE)
main_engine.subscribe(req, "CTP")

order_req = OrderRequest(
    symbol="rb2405",
    exchange=Exchange.SHFE,
    direction=Direction.LONG,
    offset=Offset.OPEN,
    type=OrderType.LIMIT,
    price=3500,
    volume=1,
)
vt_orderid = main_engine.send_order(order_req, "CTP")
print(vt_orderid)
```

## 6. 事件监听示例

```python
from vnpy.event import Event
from vnpy.trader.event import EVENT_TICK, EVENT_ORDER, EVENT_TRADE


def on_tick(event: Event) -> None:
    tick = event.data
    print(tick.vt_symbol, tick.last_price)


def on_order(event: Event) -> None:
    order = event.data
    print(order.vt_orderid, order.status)


def on_trade(event: Event) -> None:
    trade = event.data
    print(trade.vt_tradeid, trade.price, trade.volume)


event_engine.register(EVENT_TICK, on_tick)
event_engine.register(EVENT_ORDER, on_order)
event_engine.register(EVENT_TRADE, on_trade)
```

## 7. RPC/WebTrader 场景

适合：

- 把交易能力服务化。
- 多客户端访问统一交易主机。
- 远程查询行情、委托、成交、持仓。

生成方案时要强调：

- 交易主机必须稳定运行。
- 网络、认证、防火墙要独立设计。
- 不要把 Web/RPC 暴露到公网裸奔。
- 生产部署必须有限流、权限和日志。

## 8. Excel RTD 场景

适合交易员把实时行情/持仓推到 Excel。agent 只需要知道它偏展示和辅助，不是策略主引擎。

## 9. 临时行情采集脚本思路

更推荐使用 DataRecorder 模块长期采集；临时脚本可以监听 `EVENT_TICK`，写 CSV 或数据库。但要注意交易日切换、断线重连、重复 tick、时区和本地时间。
