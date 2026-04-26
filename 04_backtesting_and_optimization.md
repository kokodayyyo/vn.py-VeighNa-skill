# Summary：
本文件说明 CtaBacktester 和 BacktestingEngine 的回测参数、最小脚本、优化流程和结果解释方式。它用于帮助 agent 生成回测代码、配置参数，并定位无数据、无成交或统计异常等问题。

# 04 Backtesting & Optimization：回测脚本和参数

## 1. CtaBacktester UI 参数语义

| 字段 | 含义 | 示例 |
|---|---|---|
| 本地代码 | `vt_symbol` | `IF888.CFFEX`、`rb2405.SHFE` |
| K线周期/回测模式 | K线频率或 Tick 回测模式 | K线常用 `1m`、`1h`、`d`、`w`；Tick 回测需使用 Tick 模式 |
| 开始/结束日期 | 回测区间 | `2020/1/1` 到 `2023/12/31` |
| 手续费率 | 成交额比例 | `0.000023` |
| 滑点 | 下单价与成交价的价格点位差，按品种保守设定 | `1`、`0.2` |
| 合约乘数 | 每点价值 | 股指期货常见 `300`，按合约查 |
| 最小价格变动 | price tick | 如 `0.2`、`1` |
| 起始资金 | capital | `1_000_000` |

不要给用户编造手续费、乘数、pricetick。若用户没给，明确让其按交易所合约参数填写，或者仅用示例值。

## 2. 最小回测脚本

```python
from datetime import datetime

from vnpy.trader.constant import Interval
from vnpy_ctastrategy.backtesting import BacktestingEngine

from strategies.agent_double_ma_strategy import AgentDoubleMaStrategy


def run_backtest() -> None:
    engine = BacktestingEngine()

    engine.set_parameters(
        vt_symbol="rb888.SHFE",
        interval=Interval.MINUTE,
        start=datetime(2020, 1, 1),
        end=datetime(2021, 1, 1),
        rate=0.000023,
        slippage=1,
        size=10,
        pricetick=1,
        capital=1_000_000,
    )

    engine.add_strategy(
        AgentDoubleMaStrategy,
        setting={
            "fast_window": 10,
            "slow_window": 20,
            "fixed_size": 1,
            "price_add": 0.0,
        },
    )

    engine.load_data()
    engine.run_backtesting()
    df = engine.calculate_result()
    stats = engine.calculate_statistics()
    print(stats)

    # 可选：画图。无图形环境时可能失败。
    # engine.show_chart()


if __name__ == "__main__":
    run_backtest()
```

## 3. 脚本回测的数据前置条件

`BacktestingEngine.load_data()` 的前提是本地数据库已有对应 `vt_symbol`、`interval`、`start/end` 的历史数据。CtaBacktester UI 提供“下载数据”入口，下载后的数据会保存到本地数据库，后续回测可以直接复用；纯脚本回测不会自动替你从 datafeed 下载数据。

如果用户要求“给我一段能直接跑的回测脚本”，agent 应同时提示先用 CtaBacktester、DataManager 或 datafeed 脚本准备数据，或者在脚本中先调用 datafeed 查询并写入数据库。

## 4. 回测前检查清单

1. 策略文件可被 Python 导入。
2. 策略没有依赖 GUI。
3. `vt_symbol` 与数据库中的 `symbol/exchange/interval` 一致。
4. 数据库有覆盖 `start/end` 的历史数据。
5. `load_bar()` 天数足够初始化 `ArrayManager`。
6. `interval` 与策略内部逻辑一致。回测 1 分钟数据但策略内部合成 15 分钟是可行的。
7. `size/pricetick/rate/slippage` 按实际合约配置。

## 5. 参数优化脚本骨架

```python
from datetime import datetime

from vnpy.trader.constant import Interval
from vnpy.trader.optimize import OptimizationSetting
from vnpy_ctastrategy.backtesting import BacktestingEngine

from strategies.agent_double_ma_strategy import AgentDoubleMaStrategy


engine = BacktestingEngine()
engine.set_parameters(
    vt_symbol="rb888.SHFE",
    interval=Interval.MINUTE,
    start=datetime(2020, 1, 1),
    end=datetime(2021, 1, 1),
    rate=0.000023,
    slippage=1,
    size=10,
    pricetick=1,
    capital=1_000_000,
)
engine.add_strategy(AgentDoubleMaStrategy, {})

setting = OptimizationSetting()
setting.set_target("sharpe_ratio")
setting.add_parameter("fast_window", 5, 20, 5)
setting.add_parameter("slow_window", 20, 60, 10)
setting.add_parameter("fixed_size", 1)

# 穷举优化
results = engine.run_bf_optimization(setting)
print(results)

# 遗传算法优化，搜索空间大时使用
# results = engine.run_ga_optimization(setting)
```

目标字段名称可能因版本和语言设置变化。若报目标不存在，先打印统计结果的 keys 或在 UI 的目标下拉框确认。

## 6. 结果解释

重点看：

- 总收益率、年化收益、最大回撤、百分比最大回撤。
- 总手续费、总滑点、总成交额、成交笔数。
- 夏普比率、收益回撤比。
- 每日盈亏分布是否由少数极端交易贡献。
- K 线图交易点是否符合策略逻辑。

不要只用最终收益判断策略。高频策略必须看交易成本；趋势策略必须看回撤和连续亏损区间。

## 7. 常见回测失败

| 现象 | 原因 | 处理 |
|---|---|---|
| 历史数据加载完成，数据量 0 | 数据库无数据或 `vt_symbol/interval` 不匹配 | 用 DataManager/CtaBacktester 下载或导入 |
| 回测成交记录为空 | 没有信号、下单价格无法成交、`trading` 阶段逻辑没走到 | 打日志，放宽条件，检查 `am.inited` |
| 指标全是 0 | `ArrayManager` 未初始化 | 增大 `load_bar()` 天数或减小 size/window |
| 回测和实盘不同 | 回测用 bar，实盘 tick 合成 bar；盘口、滑点、涨跌停处理不同 | 增加滑点模型和实盘保护 |
| 参数优化过慢 | 搜索空间太大 | 缩小范围，先粗后细，或用遗传算法 |

## 8. 数据准备路线

1. CtaBacktester 一键下载：适合已配置 datafeed。
2. DataManager 导入 CSV：适合已有外部数据。
3. DataRecorder 实盘录制：适合交易接口推送数据。
4. 数据库脚本写入：适合批量数据管线。

详见 `05_data_gateway_database.md`。
