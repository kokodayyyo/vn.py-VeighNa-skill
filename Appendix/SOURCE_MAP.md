# Summary：
本文件记录手册整理时参考的官方文档和源码来源。它用于追溯内容依据，并帮助维护者在 VeighNa/vn.py 文档或源码更新后重新校验相关章节。

# Source Map

本手册基于官方文档和公开源码整理。内容被重组为 agent 编码手册，并非官方全文复刻；当官方文档示例与当前源码存在差异时，代码生成规则优先按源码和用户本地安装版本校准。

## 官方文档

- VeighNa 用户文档目录：`https://www.vnpy.com/docs/cn/index.html`
- VeighNa Trader：`https://www.vnpy.com/docs/cn/community/info/veighna_trader.html`
- CtaStrategy：`https://www.vnpy.com/docs/cn/community/app/cta_strategy.html`
- CtaBacktester：`https://www.vnpy.com/docs/cn/community/app/cta_backtester.html`
- 数据库：`https://www.vnpy.com/docs/cn/community/info/database.html`
- 数据服务：`https://www.vnpy.com/docs/cn/community/info/datafeed.html`
- DataManager：`https://www.vnpy.com/docs/cn/community/app/data_manager.html`
- RiskManager：`https://www.vnpy.com/docs/cn/community/app/risk_manager.html`
- PortfolioStrategy：`https://www.vnpy.com/docs/cn/community/app/portfolio_strategy.html`
- SpreadTrading：`https://www.vnpy.com/docs/cn/community/app/spread_trading.html`
- AlgoTrading：`https://www.vnpy.com/docs/cn/community/app/algo_trading.html`
- ScriptTrader：`https://www.vnpy.com/docs/cn/community/app/script_trader.html`
- DataRecorder：`https://www.vnpy.com/docs/cn/community/app/data_recorder.html`
- RpcService：`https://www.vnpy.com/docs/cn/community/app/rpc_service.html`
- WebTrader：`https://www.vnpy.com/docs/cn/community/app/web_trader.html`
- ExcelRtd：`https://www.vnpy.com/docs/cn/community/app/excel_rtd.html`
- OptionMaster：`https://www.vnpy.com/docs/cn/community/app/option_master.html`

## 官方源码

- `vnpy_ctastrategy/__init__.py`：`https://raw.githubusercontent.com/vnpy/vnpy_ctastrategy/main/vnpy_ctastrategy/__init__.py`
- `vnpy_ctastrategy/template.py`：`https://raw.githubusercontent.com/vnpy/vnpy_ctastrategy/main/vnpy_ctastrategy/template.py`
- `vnpy_ctastrategy/engine.py`：`https://raw.githubusercontent.com/vnpy/vnpy_ctastrategy/main/vnpy_ctastrategy/engine.py`
- `vnpy_ctastrategy/base.py`：`https://raw.githubusercontent.com/vnpy/vnpy_ctastrategy/main/vnpy_ctastrategy/base.py`
- `vnpy_ctastrategy/backtesting.py`：`https://raw.githubusercontent.com/vnpy/vnpy_ctastrategy/main/vnpy_ctastrategy/backtesting.py`
- `vnpy/trader/utility.py`：`https://github.com/vnpy/vnpy/blob/master/vnpy/trader/utility.py`
- `vnpy/trader/object.py`：`https://github.com/vnpy/vnpy/blob/master/vnpy/trader/object.py`
- `vnpy_ctastrategy/strategies/double_ma_strategy.py`：`https://raw.githubusercontent.com/vnpy/vnpy_ctastrategy/main/vnpy_ctastrategy/strategies/double_ma_strategy.py`
- `vnpy_ctastrategy/strategies/atr_rsi_strategy.py`：`https://raw.githubusercontent.com/vnpy/vnpy_ctastrategy/main/vnpy_ctastrategy/strategies/atr_rsi_strategy.py`
- `vnpy_portfoliostrategy/template.py`：`https://github.com/vnpy/vnpy_portfoliostrategy/blob/main/vnpy_portfoliostrategy/template.py`
- `vnpy_spreadtrading/template.py`：`https://github.com/vnpy/vnpy_spreadtrading/blob/main/vnpy_spreadtrading/template.py`
- `vnpy_scripttrader/engine.py`：`https://github.com/vnpy/vnpy_scripttrader/blob/main/vnpy_scripttrader/engine.py`

## 文档与源码差异记录

- CTA `load_bar(use_database=False)`：官方文档按“交易接口 → 数据服务 → 数据库”描述初始化数据来源；源码实际逻辑更细，若 gateway 合约存在且支持历史数据则先查 gateway，否则查 datafeed，若返回空再回退数据库。若要强制使用本地数据库，应传 `use_database=True`。
- PortfolioStrategy：不要套用 CTA 的 `on_order()`/`on_trade()` 推送模式；组合策略应在 `on_bars()` 中查询持仓、委托和活动委托号。`get_all_active_orderids()` 按活动委托号字符串列表使用；若文档或类型注解不一致，以本地源码和运行返回值为准。
- SpreadTrading：官方示例常用 `am.boll(window, dev)`，但不同版本 `ArrayManager` 方法集可能不同。生成通用模板时应写三层兜底：`am.boll()`、`am.std()`、`numpy.std(am.close[-window:])`。
- SpreadTrading 算法方向：`start_long_algo()` 应对应 `Direction.LONG`，`start_short_algo()` 应对应 `Direction.SHORT`。若官方文档示例和本地源码不一致，以本地 `vnpy_spreadtrading/template.py` 为准。
- ScriptTrader：官方模板使用 `run(engine: ScriptEngine)` 和 `engine.strategy_active` 控制循环；实际生成代码时应补 `get_tick()`、`get_contract()`、`get_account()`、`get_position()` 的空值保护。

## 维护提示

- CTA 基础模板较稳定。
- PortfolioStrategy、SpreadTrading、Elite 相关模板更容易出现版本差异；生成完整代码时优先要求用户提供本地包版本或模板文件。
- 回测指标字段名、优化目标名可能随版本和语言环境变化。
- 数据库插件支持受 Python 版本、第三方驱动和安装包影响。
- 每次更新手册时，优先复核 `02`、`06`、`08`、`09`、`10` 中的 API 断言。

## 二次复核补充

- datafeed 配置：官方文档要求 `datafeed.name`、`datafeed.username`、`datafeed.password` 三项对所有数据服务均为必填；token 方式填入 `datafeed.password`。手册不应默认建议 `username` 留空。
- AlgoTrading 自定义算法：官方文档要求用户搭建的算法放到 `algo_trading.algos` 目录，内置示例位于 `vnpy_algotrading.algos`。这不同于 CTA 策略的运行目录 `strategies`。
- CTA 模板结构：官方示例通常在 `__init__()` 中创建 `BarGenerator` 和 `ArrayManager`，手册保留 `on_init()` 简化写法，同时补充官方结构供严格贴合文档时使用。
- PortfolioStrategy 目标仓位：官方进阶章节提供目标仓位调仓模式；手册避免写成官方明确“更推荐”，改为说明其更贴合组合策略定位。
