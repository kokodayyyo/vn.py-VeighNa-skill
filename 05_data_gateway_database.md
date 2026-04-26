# Summary：
本文件整理交易接口、行情订阅、vt_symbol 规则、历史数据、数据库和数据服务配置。它用于让 agent 处理连接 gateway、准备回测数据、导入导出 K 线和配置数据库等任务。

# 05 Data, Gateway & Database：行情、交易接口、历史数据

## 1. gateway 与交易流程

典型流程：

1. `main_engine.add_gateway(CtpGateway)`。
2. 启动 Trader。
3. 菜单连接 CTP/IB/XTP 等接口。
4. 日志显示登录成功、合约查询成功。
5. 订阅行情或启动策略。
6. 策略下单经 CTA/组合/算法引擎转到 MainEngine，再到 gateway。

CTP/SimNow 常见字段：

```text
用户名：InvestorID，不是网页登录手机号
密码：交易密码
经纪商代码：SimNow 通常 9999
交易服务器：host:port
行情服务器：host:port
产品名称：按柜台/SimNow 要求
授权编码：按柜台/SimNow 要求
```

如果点击连接后没有任何日志，先检查服务器地址、端口、网络连通性。

IB 特别注意：IB 登录后通常不能自动获取全部合约信息，很多模块需要先手动订阅目标合约行情，才能查询到合约。

## 2. `vt_symbol` 规则

```text
symbol.EXCHANGE
```

示例：

```text
rb2405.SHFE
IF888.CFFEX
cu888.SHFE
AAPL.SMART
```

`symbol` 必须和交易接口/数据库中的合约代码一致。交易所后缀必须是 vn.py 的 `Exchange` 枚举值。

## 3. 数据库配置

默认 SQLite，适合本地入门：

```text
database.name: sqlite
database.database: database.db
```

MySQL 示例：

```text
database.name: mysql
database.database: vnpy
database.host: localhost
database.port: 3306
database.user: root
database.password: 你的密码
```

关系型数据库通常需要先手工创建数据库，例如：

```sql
create database vnpy;
```

常见支持项：SQLite、MySQL、PostgreSQL、MongoDB、InfluxDB、DolphinDB、Arctic、LevelDB。实际能否使用取决于安装的数据库驱动包和 Python 版本。

## 4. 数据库读写脚本

```python
from datetime import datetime

from vnpy.trader.constant import Exchange, Interval
from vnpy.trader.database import get_database


database = get_database()

bars = database.load_bar_data(
    symbol="rb888",
    exchange=Exchange.SHFE,
    interval=Interval.MINUTE,
    start=datetime(2020, 1, 1),
    end=datetime(2020, 2, 1),
)

print(f"loaded bars: {len(bars)}")
```

保存数据：

```python
# bar_data 必须是 list[BarData]
database.save_bar_data(bar_data)

# tick_data 必须是 list[TickData]
database.save_tick_data(tick_data)
```

删除数据，谨慎使用：

```python
database.delete_bar_data(
    symbol="rb888",
    exchange=Exchange.SHFE,
    interval=Interval.MINUTE,
)

database.delete_tick_data(
    symbol="rb888",
    exchange=Exchange.SHFE,
)
```

## 5. CSV 转 BarData 示例

```python
from datetime import datetime
import pandas as pd

from vnpy.trader.constant import Exchange, Interval
from vnpy.trader.object import BarData
from vnpy.trader.database import get_database


def import_csv(path: str) -> None:
    df = pd.read_csv(path)
    bars = []

    for _, row in df.iterrows():
        bar = BarData(
            symbol="rb888",
            exchange=Exchange.SHFE,
            datetime=datetime.strptime(row["datetime"], "%Y-%m-%d %H:%M:%S"),
            interval=Interval.MINUTE,
            volume=float(row["volume"]),
            turnover=float(row.get("turnover", 0)),
            open_interest=float(row.get("open_interest", 0)),
            open_price=float(row["open"]),
            high_price=float(row["high"]),
            low_price=float(row["low"]),
            close_price=float(row["close"]),
            gateway_name="DB",
        )
        bars.append(bar)

    database = get_database()
    database.save_bar_data(bars)
    print(f"saved {len(bars)} bars")
```

字段名因 CSV 来源不同会变化，生成代码时让用户确认列名。

## 6. datafeed 配置

datafeed 用于从外部数据服务下载历史数据。官方文档目录列出的服务包括迅投研（XT）、RQData、UData、TuShare、TQSDK、Wind、iFinD、Tinysoft 等。注意原文正文有“七种数据服务”的表述，但目录和正文小节实际列出上述 8 类；agent 应以具体插件是否已安装和可用为准。全局配置字段采用统一前缀：

```text
datafeed.name: rqdata        # 数据服务接口名称，全小写
datafeed.username: <用户名>  # 官方文档要求填写；token 服务也不要默认省略
datafeed.password: <密码或 token>
```

官方文档说明 `datafeed.name`、`datafeed.username`、`datafeed.password` 对所有数据服务都是必填字段；token 方式授权时，token 通常写入 `datafeed.password`。不同插件对账号字段含义和可用周期要求不同，agent 不应猜测用户的数据服务账号字段，也不应默认让 `datafeed.username` 留空；若某个插件实际允许留空，应以该插件说明和本地测试为准。

## 7. 数据流优先级

CTA 策略初始化调用 `load_bar(days, interval, callback, use_database=False)` 时，源码中的实际逻辑不是简单的“gateway → datafeed → database”线性顺序，而是：

1. `use_database=True`：直接读取本地数据库，跳过 gateway 和 datafeed。
2. `use_database=False` 且已能通过 `main_engine.get_contract(vt_symbol)` 找到合约，并且该合约的 `history_data=True`：优先向该合约所属 gateway 查询历史 K 线。
3. `use_database=False` 但找不到支持历史数据的合约：查询已配置的 datafeed。
4. 以上路径得到的 `bars` 为空时：回退读取本地数据库。

因此有一个容易误判的细节：如果 gateway 支持历史数据但返回空结果，CTA 引擎会回退数据库，而不会再改查 datafeed。若希望强制使用本地已清洗数据，策略初始化应写 `self.load_bar(days, use_database=True)`。

`load_tick(days)` 只从本地数据库读取 Tick 数据；它不走 gateway/datafeed。

## 8. 合约、行情、委托常用排查

| 问题 | 快速检查 |
|---|---|
| 找不到合约 | `vt_symbol`、交易所后缀、是否连接接口、是否完成合约查询 |
| IB 查不到合约 | 先手动订阅行情 |
| 行情不更新 | 是否订阅、交易时段、行情服务器、合约是否过期 |
| 委托不发出 | 策略是否 `trading=True`、是否风控拦截、价格/数量是否合法 |
| 委托拒单 | 资金、持仓、开平、涨跌停、最小变动价位、交易权限 |
