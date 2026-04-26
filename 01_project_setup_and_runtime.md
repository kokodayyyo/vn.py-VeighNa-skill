# Summary：
本文件说明 vn.py 项目的最小启动脚本、运行目录、策略放置位置和常见 app/gateway 加载方式。它用于帮助 agent 生成可运行的 Trader 启动代码，并解释从本地工程到 UI 加载策略的完整路径。

# 01 Project Setup & Runtime：启动、目录、加载

## 1. 最小 Trader 启动脚本

适合本地调试、明确加载哪些 gateway/app。

```python
from vnpy.event import EventEngine
from vnpy.trader.engine import MainEngine
from vnpy.trader.ui import MainWindow, create_qapp

from vnpy_ctp import CtpGateway
from vnpy_ctastrategy import CtaStrategyApp
from vnpy_ctabacktester import CtaBacktesterApp
from vnpy_datamanager import DataManagerApp
from vnpy_riskmanager import RiskManagerApp


def main() -> None:
    qapp = create_qapp()
    event_engine = EventEngine()
    main_engine = MainEngine(event_engine)

    main_engine.add_gateway(CtpGateway)
    main_engine.add_app(CtaStrategyApp)
    main_engine.add_app(CtaBacktesterApp)
    main_engine.add_app(DataManagerApp)
    main_engine.add_app(RiskManagerApp)

    main_window = MainWindow(main_engine, event_engine)
    main_window.showMaximized()
    qapp.exec()


if __name__ == "__main__":
    main()
```

按需替换 gateway/app。不要无脑加载全部模块；启动慢、依赖缺失、UI 复杂度都会上升。

## 2. 常见 app/gateway 导入

```python
from vnpy_ctp import CtpGateway
from vnpy_ib import IbGateway
from vnpy_xtp import XtpGateway

from vnpy_ctastrategy import CtaStrategyApp
from vnpy_ctabacktester import CtaBacktesterApp
from vnpy_datamanager import DataManagerApp
from vnpy_datarecorder import DataRecorderApp
from vnpy_portfoliostrategy import PortfolioStrategyApp
from vnpy_spreadtrading import SpreadTradingApp
from vnpy_algotrading import AlgoTradingApp
from vnpy_riskmanager import RiskManagerApp
from vnpy_scripttrader import ScriptTraderApp
from vnpy_webtrader import WebTraderApp
```

依赖包可能需要单独安装。生成代码时不要假设所有 app 都已安装。

## 3. 运行目录和策略目录

自写策略默认放在运行目录下：

```text
<运行目录>/strategies/
```

Windows 默认安装用户常见路径：

```text
C:\Users\<用户名>\strategies
```

运行目录可在 Trader 主窗口标题栏查看。策略文件命名采用下划线：

```text
double_ma_strategy.py
boll_atr_strategy.py
intraday_breakout_strategy.py
```

策略类名采用驼峰式：

```python
class DoubleMaStrategy(CtaTemplate):
    ...
```

UI 下拉框显示的是策略类名，不是文件名。

## 4. `.vntrader` 与配置文件

常见缓存/配置：

```text
.vntrader/cta_strategy_setting.json    # CTA 策略实例配置
.vntrader/cta_strategy_data.json       # CTA 策略变量与持仓缓存
.vntrader/portfolio_strategy_setting.json
.vntrader/portfolio_strategy_data.json
.vntrader/log/
```

不要让 agent 直接覆盖这些文件，除非用户明确要求修复配置。涉及实盘持仓缓存时必须提示风险。

## 5. 实盘启动顺序

1. 启动 Trader。
2. 连接交易接口。
3. 等日志显示登录成功、合约查询完成。
4. 如使用 IB，通常先手动订阅目标合约行情。
5. 打开 CTA/组合/价差等模块。
6. 添加策略实例。
7. 初始化，确认 `inited=True`。
8. 启动，确认 `trading=True`。

## 6. Station 与脚本模式

Station 适合普通用户用 GUI 勾选 gateway/app。脚本模式适合开发者固定加载清单、复现 bug、做定制启动。

agent 给开发者答案时优先给脚本模式；给非开发者答案时补充 Station 操作路径。
