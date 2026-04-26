# Summary：
本文件按错误现象组织 vn.py 常见排查路径，包括策略加载、初始化、历史数据、合约订阅、发单、回测无成交等问题。它用于让 agent 在生成代码后快速定位运行失败原因，并给出可执行修复步骤。

# 08 Debugging Playbook：现象到排查

## 1. 策略类不出现在 UI 下拉框

检查：

1. 文件是否放在运行目录的 `strategies` 文件夹。
2. 文件名是否 `.py`，不是 `.txt` 或错误后缀。
3. 文件名是否下划线风格，类名是否驼峰式。
4. 类是否继承 `CtaTemplate`。
5. 顶部 import 是否报错。任何 import 异常都会导致策略加载失败。
6. 类名是否与示例策略或其他策略重名。
7. 重启 Trader 或使用策略重载功能。

快速验证：

```bash
python -m py_compile strategies/your_strategy.py
```

## 2. 添加策略失败

| 日志/现象 | 原因 | 处理 |
|---|---|---|
| 存在重名 | 实例名重复 | 换实例名 |
| 本地代码缺失交易所后缀 | `vt_symbol` 写成 `rb2405` | 改成 `rb2405.SHFE` |
| 交易所后缀不正确 | `rb2405.SH` 等错误 | 用 `Exchange` 枚举/合约查询结果 |
| 参数类型不对 | 默认参数是 int，却填小数 | 把默认值写成 `1.0` |

## 3. 初始化失败或找不到合约

检查：

1. 是否连接交易接口。
2. 日志是否显示合约查询成功。
3. `vt_symbol` 是否大小写正确。
4. 合约是否过期或不存在。
5. IB 是否先手动订阅行情。
6. 组合策略里多个合约是否用英文逗号且无空格。

## 4. 初始化完成但策略不发信号

高概率原因：`ArrayManager` 未初始化。

检查代码：

```python
am.update_bar(bar)
if not am.inited:
    return
```

处理：

- 增大 `self.load_bar(10)` 的天数，如 `self.load_bar(30)`。
- 减小 `ArrayManager(size=100)` 的 size，但不能小于最大指标窗口。
- 确认历史数据覆盖交易时段。
- 回测时确认 interval 与策略合成周期一致。

## 5. 策略变量不刷新

检查是否调用：

```python
self.put_event()
```

通常在 `on_bar()` 末尾、`on_start()`、`on_stop()`、`on_trade()` 调用。

`put_event()` 只有 `inited=True` 后才刷新。

## 6. 下单函数调用了但没有委托

检查：

1. 策略是否已启动，`trading=True`。
2. 是否在 `on_init()` 里调用了下单。初始化阶段不会真实下单。
3. 下单价格是否在涨跌停范围内。
4. 数量是否满足最小成交单位。
5. 合约是否可交易，是否在交易时段。
6. RiskManager 是否拦截。
7. 账户资金/持仓是否足够。
8. 对上期所/能源中心品种，平今平昨转换是否符合预期。

## 7. 回测无成交

检查：

1. `engine.load_data()` 是否加载到数据。
2. 策略是否真的发出订单，可在策略中 `write_log()`。
3. 下单价格是否太远，无法撮合。
4. `stop=True` 停止单触发条件是否满足。
5. `am.inited` 是否长期 False。
6. `fixed_size` 是否为 0。

## 8. 实盘和回测差异大

常见原因：

- 回测用 K 线，实盘 Tick 合成 K 线。
- 回测撮合模型简化，实盘有盘口、排队、延迟、拒单。
- 手续费、滑点、合约乘数、最小跳动价位错误。
- 数据复权/主连处理和实盘合约不同。
- 策略使用当前 K 线 close 产生信号并同价成交，存在未来函数风险。

处理：

- 提高滑点和手续费压力测试。
- 用下一根 K 线或保守价格撮合思路写策略。
- 分离信号价、下单价、成交价假设。

## 9. 本地停止单风险

本地停止单特点：

- 保存在本机，关机失效。
- 只有本地可见。
- 触发有延迟，可能滑点。
- 触发后通常用涨跌停价或盘口价发限价单。

只建议用于流动性较好的合约。关键风控不要完全依赖本地停止单。

## 10. 生成修复建议时的格式

先给结论，再给最短修复代码。例如：

```text
问题是 ArrayManager 没初始化，`on_bar` 每次都 return。把 load_bar 从 10 天增加到 30 天，并把 size 调整为 max(100, slow_window + 5)。
```

```python
self.am = ArrayManager(size=max(100, self.slow_window + 5))
self.load_bar(30)
```

## 11. 价差策略类无法实例化

常见原因是继承 `SpreadStrategyTemplate` 后漏实现抽象回调。当前官方模板中至少要实现：`on_init()`、`on_start()`、`on_stop()`、`on_spread_data()`、`on_spread_tick()`、`on_spread_bar()`、`on_spread_pos()`、`on_spread_algo()`。即使某些回调暂时不用，也应写成 `pass`。


## 12. `load_bar()` 没按预期走 datafeed

CTA `load_bar(use_database=False)` 的源码逻辑是：有支持历史数据的合约 gateway 时先查 gateway；没有这种 gateway 时才查 datafeed；若结果为空则回退数据库。它不是无条件按 gateway、datafeed、database 逐个尝试。

如果用户明确要用本地清洗后的数据，使用：

```python
self.load_bar(30, use_database=True)
```

如果用户明确要用外部 datafeed 初始化，需确认当前 gateway 不会优先接管历史数据，或改为先把 datafeed 数据下载/写入数据库后用 `use_database=True`。

## 13. 组合策略里 `on_order()`/`on_trade()` 不触发

PortfolioStrategy 与 CTA 不同。官方文档说明多合约组合策略无法在回测中提供每个合约委托成交的先后推送，因此不要依赖 CTA 式 `on_order()`/`on_trade()`。应在 `on_bars()` 中使用 `get_pos(vt_symbol)`、`get_order(vt_orderid)` 和 `get_all_active_orderids()` 查询状态。


## 14. ScriptTrader 刚启动就报 NoneType

订阅行情和连接接口后，合约、行情、账户、持仓对象不一定立刻可用。`engine.get_tick()`、`engine.get_contract()` 等可能暂时返回 `None`。脚本循环中必须先判断空值，再访问字段；必要时在启动后先 `sleep()` 等待数据到达。

```python
tick = engine.get_tick(vt_symbol)
if tick is None:
    continue
```


## 15. 组合策略里部分合约指标长期不初始化

常见原因是在 `on_bars()` 的第一个循环里写了：

```python
for vt_symbol, bar in bars.items():
    am.update_bar(bar)
    if not am.inited:
        return
```

这种写法会导致同一时间切片里排在后面的合约得不到更新。正确方式是先更新当前 `bars` 中所有合约的 `ArrayManager`，再统一判断是否全部初始化：

```python
all_inited = True
for vt_symbol, bar in bars.items():
    am = self.ams[vt_symbol]
    am.update_bar(bar)
    if not am.inited:
        all_inited = False

if not all_inited:
    return
```

## 16. 价差策略报 `ArrayManager` 没有 `boll`

官方 SpreadTrading 示例使用 `am.boll(window, dev)` 计算布林带，但不同 vn.py 安装版本的 `ArrayManager` 指标函数可能不同。生成价差模板时应写兼容分支：优先使用 `am.boll()`，不存在时用 `am.sma()` + `am.std()`；如果连 `am.std()` 也没有，再退回 `numpy.std(am.close[-window:])`。

```python
import numpy as np

mean = am.sma(window)
if hasattr(am, "boll"):
    upper, lower = am.boll(window, dev)
elif hasattr(am, "std"):
    std = am.std(window)
    upper = mean + std * dev
    lower = mean - std * dev
else:
    std = float(np.std(am.close[-window:]))
    upper = mean + std * dev
    lower = mean - std * dev
```
