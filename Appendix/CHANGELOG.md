# 修复整理说明

本目录为整理后的 VeighNa/vn.py Agent Coding Manual。原上传文件未被覆盖；本目录中的文件名已去掉括号版本号。

## 本次修复

1. 统一文件名：`00_agent_task_router(13).md` → `00_agent_task_router.md` 等。
2. 修正 CTA 默认流程：区分原始 `on_bar()` 和窗口 K 线回调，避免 agent 把交易逻辑错误写进只负责合成窗口 K 线的 `on_bar()`。
3. 补强 CTA 布林突破模板：`am.std()` 不存在时退回 `numpy.std()`。
4. 补强 SpreadTrading 布林带模板：按 `am.boll()` → `am.std()` → `numpy.std()` 三层兜底。
5. 补充 SpreadTrading 算法方向说明：`start_long_algo()` 对应 `Direction.LONG`，`start_short_algo()` 对应 `Direction.SHORT`；文档和源码不一致时以本地源码为准。
6. 补充 PortfolioStrategy 活动委托号说明：`get_all_active_orderids()` 按委托号字符串列表使用；若类型标注不一致，以本地源码和运行返回值为准。
7. 重写 `SOURCE_MAP.md`，删除重复的审计备注，改为“文档与源码差异记录”和“维护提示”。
8. 清理 `README.md` 的英文审计标题，改为中文修订说明。

## 仍建议使用者注意

- PortfolioStrategy、SpreadTrading 版本差异比 CTA 更常见；生成完整策略时最好让用户提供本地包版本或 `template.py`。
- 回测参数中的手续费、合约乘数、最小变动价位不能由 agent 编造，只能示例或由用户提供。
- 实盘部署建议必须包含仿真、小仓位、RiskManager、断线/拒单/涨跌停检查。

## 二次复核修复（按官方用户文档目录）

1. 修正 `05_data_gateway_database.md`：明确 `datafeed.name`、`datafeed.username`、`datafeed.password` 按官方文档均为必填；token 方式写入 `datafeed.password`，不再默认建议 `username` 留空。
2. 修正 `06_portfolio_spread_algo_recipes.md`：补充 AlgoTrading 自定义算法应放入 `algo_trading.algos`，不要误放入 CTA 的 `strategies` 目录。
3. 修正 `02_cta_template_reference.md`：补充官方风格的 `__init__()` 写法，说明 `BarGenerator`、`ArrayManager` 可在构造函数中创建。
4. 调整 `06_portfolio_spread_algo_recipes.md`：将“官方更推荐目标仓位调仓模式”改为“官方进阶章节提供更贴合组合策略定位的目标仓位调仓模式”，避免把解释写成官方原话。
5. 更新 `README.md` 与 `SOURCE_MAP.md`，记录二次复核依据和维护差异。
