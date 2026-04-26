# Summary：
本文件把常见用户需求映射到 vn.py/VeighNa 的模块、模板和输出形态。它用于让 agent 在开始写代码前快速判断应使用 CTA、回测、数据、组合、价差、算法执行还是排错流程。

# 00 Agent Task Router：需求到 vn.py 代码路径

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

## 2. 生成答案时的默认流程

如果用户让你“写一个 vn.py 策略”：

1. 默认写 `CtaTemplate`，除非用户明确说多合约/价差/期权/组合。
2. 给完整代码，不只给片段。
3. 代码顶部包含 imports。
4. 类名驼峰式；参数和变量声明完整。
5. `on_init()` 创建或确认 `BarGenerator`、`ArrayManager`，然后 `load_bar()`。
6. `on_tick()` 调用 `self.bg.update_tick(tick)`。
7. 在实际用于交易的 K 线回调中执行：`cancel_all()`、`am.update_bar(bar)`、`if not am.inited: return`、计算指标、按 `self.pos` 下单、`put_event()`。若 `on_bar()` 只是把 1 分钟 K 线继续合成窗口 K 线，则 `on_bar()` 只调用 `self.bg.update_bar(bar)`，交易逻辑放到窗口回调中。
8. 补齐 `on_order`、`on_trade`、`on_stop_order`。
9. 给部署路径和加载步骤。

如果用户让你“回测策略”：

1. 确认策略类可导入。
2. 设置 `vt_symbol`、`interval`、`start/end`、`rate`、`slippage`、`size`、`pricetick`、`capital`。
3. `add_strategy()` 后 `load_data()`、`run_backtesting()`、`calculate_result()`、`calculate_statistics()`。
4. 如果没有数据，转到 `05` 的数据准备。

如果用户让你“连实盘跑”：

1. 说明先在 Trader 加载 gateway 和 app。
2. 连接接口，等合约查询成功。
3. 策略放入运行目录 `strategies`。
4. 创建策略实例、初始化、启动。
5. 风险控制和仿真建议独立说明。

## 3. 任务到模板快速映射

| 策略描述 | 推荐模板 |
|---|---|
| 均线金叉死叉 | `03` 的双均线模板 |
| 唐奇安/布林突破 | `03` 的通道突破模板 |
| 趋势 + ATR 移动止损 | `03` 的 ATR trailing stop 模板 |
| 目标仓位调仓 | `TargetPosTemplate` 或 PortfolioStrategy |
| 日内固定时间清仓 | CTA 模板加 `bar.datetime.time()` 判断 |
| 网格策略 | CTA 模板，基于 `pos`、网格价和活动委托管理 |
| 多合约价差 | SpreadTrading，不要硬写两个 CTA 互相对冲 |
| 跨品种组合轮动 | PortfolioStrategy |
| 只想临时订阅行情/下单 | ScriptTrader 或 MainEngine 脚本 |

## 4. agent 输出必须附带的运行说明

每次输出策略代码后，补一段极短部署说明：

```text
保存为 strategies/xxx_strategy.py；重启或重载策略；在 CTA 策略模块添加类 XxxStrategy；填写 vt_symbol 与参数；先初始化，确认 inited=True，再启动。
```

如果用户的代码用于回测，补：

```text
确认数据库已有对应 vt_symbol、interval、时间段的数据；若为空，先用 DataManager/CtaBacktester 下载或导入。
```
