# Summary：
# VeighNa/vn.py Agent Coding Manual

本项目是一套面向 AI Agent 的 VeighNa/vn.py 编码指南，用于帮助智能体在生成量化交易代码时，尽量遵循 VeighNa 的实际运行模型、策略生命周期、回测流程和常见 API 用法。

它不是 VeighNa 官方文档的替代品，而是一份面向代码生成场景的工程化速查手册。重点目标是减少 AI 生成代码时消耗的token以及幻觉
## 适用范围

本手册主要覆盖以下场景：

- 单品种 CTA 策略生成
- CTA 策略模板、生命周期和下单逻辑
- 回测脚本与参数配置
- 历史数据读取、数据库导入和 gateway 连接
- 多合约组合策略
- 价差交易策略
- 算法执行和 ScriptTrader 脚本
- 常见报错排查
- Agent 生成 vn.py 代码时的约束规则

## 官方参考链接

使用本项目时，请优先参考 VeighNa 官方资料，并以本地安装版本的源码为最终准绳。

- VeighNa 官方文档：<https://www.vnpy.com/docs/cn/index.html>
- VeighNa GitHub 组织：<https://github.com/vnpy>
- vn.py 主仓库：<https://github.com/vnpy/vnpy>
- VeighNa 量化社区：<https://www.vnpy.com/forum/>

## 文件说明

```text
README.md
00_agent_task_router.md
01_project_setup_and_runtime.md
02_cta_template_reference.md
03_cta_strategy_cookbook.md
04_backtesting_and_optimization.md
05_data_gateway_database.md
06_portfolio_spread_algo_recipes.md
07_script_trader_and_service_recipes.md
08_debugging_playbook.md
09_code_generation_rules.md
10_api_cheatsheet.md

# VeighNa/vn.py Agent Coding Manual v2

下面的 README 介绍可直接放到 GitHub。官方入口我按 VeighNa 文档站和 GitHub 主页核对过；内容结构参考了你上传的手册总览、任务路由、代码规则和 API 速查表。([GitHub][1])    

````markdown
# VeighNa/vn.py Agent Coding Manual

本项目是一套面向 AI Agent 的 VeighNa/vn.py 编码指南，用于帮助智能体在生成量化交易代码时，尽量遵循 VeighNa 的实际运行模型、策略生命周期、回测流程和常见 API 用法。

它不是 VeighNa 官方文档的替代品，而是一份面向代码生成场景的工程化速查手册。重点目标是减少 AI 生成代码时常见的错误，例如生命周期误用、策略模板选错、`ArrayManager` 未初始化、`vt_symbol` 格式错误、回测参数误填、组合策略照搬 CTA 回调等问题。

## 适用范围

本手册主要覆盖以下场景：

- 单品种 CTA 策略生成
- CTA 策略模板、生命周期和下单逻辑
- 回测脚本与参数配置
- 历史数据读取、数据库导入和 gateway 连接
- 多合约组合策略
- 价差交易策略
- 算法执行和 ScriptTrader 脚本
- 常见报错排查
- Agent 生成 vn.py 代码时的约束规则

## 官方参考链接

使用本项目时，请优先参考 VeighNa 官方资料，并以本地安装版本的源码为最终准绳。

- VeighNa 官方文档：<https://www.vnpy.com/docs/cn/index.html>
- VeighNa GitHub 组织：<https://github.com/vnpy>
- vn.py 主仓库：<https://github.com/vnpy/vnpy>
- VeighNa 量化社区：<https://www.vnpy.com/forum/>

## 文件说明

```text
README.md
00_agent_task_router.md
01_project_setup_and_runtime.md
02_cta_template_reference.md
03_cta_strategy_cookbook.md
04_backtesting_and_optimization.md
05_data_gateway_database.md
06_portfolio_spread_algo_recipes.md
07_script_trader_and_service_recipes.md
08_debugging_playbook.md
09_code_generation_rules.md
10_api_cheatsheet.md
````

## 推荐阅读顺序：

1. 先阅读 `00_agent_task_router.md`，判断用户需求应使用 CTA、PortfolioStrategy、SpreadTrading、AlgoTrading、ScriptTrader 还是回测/数据模块。
2. 写代码前阅读 `09_code_generation_rules.md`，避免生命周期、导入、下单、撤单、持仓和 UI 刷新相关错误。
3. 编写 CTA 策略时阅读 `02_cta_template_reference.md` 和 `03_cta_strategy_cookbook.md`。
4. 生成回测脚本时阅读 `04_backtesting_and_optimization.md`。
5. 涉及 gateway、行情、数据库和历史数据时阅读 `05_data_gateway_database.md`。
6. 涉及多合约、价差和算法执行时阅读 `06_portfolio_spread_algo_recipes.md`。
7. 涉及轻量脚本交易或服务化任务时阅读 `07_script_trader_and_service_recipes.md`。
8. 遇到策略不显示、不成交、不刷新、回测无结果等问题时阅读 `08_debugging_playbook.md`。
9. 需要快速确认 API 名称、参数和字段时阅读 `10_api_cheatsheet.md`。

## 设计原则

本手册倾向于服务“可运行代码生成”，而不是复刻官方文档全文。

主要约束包括：

* 默认生成 `CtaTemplate` 策略，除非用户明确要求多合约、价差、算法执行或脚本交易。
* 输出完整文件级代码，而不是只输出 `on_bar()` 片段。
* 策略参数写入 `parameters`，运行状态写入 `variables`。
* 使用 `ArrayManager` 前必须先 `update_bar()`，并在 `am.inited` 后再计算指标。
* 按 `self.pos` 或目标仓位控制下单，避免无约束重复开仓。
* K 线策略通常在信号 K 线开始时撤销旧委托，但网格、做市等策略不应粗暴每 tick 全撤。
* 回测参数中的手续费、滑点、合约乘数、最小价格变动必须由用户按真实合约确认。
* 实盘部署前应先经过仿真、风控、小仓位和异常场景检查。

## 使用方式

本项目适合被 AI Agent、代码助手或开发者作为上下文资料使用。

典型使用方式：

```text
用户需求 → 任务路由 → 选择模板 → 生成代码 → 按规则检查 → 按排错手册修复
```

例如：

* 用户说“写一个双均线策略”：优先使用 CTA 模板。
* 用户说“写一个两个合约的配对策略”：优先考虑 PortfolioStrategy 或 SpreadTrading。
* 用户说“帮我回测这个策略”：使用 BacktestingEngine 或 CtaBacktester 参数流程。
* 用户说“策略不成交”：进入 debugging playbook，检查初始化、行情、合约、交易时段、风控、价格和账户状态。

## 免责声明

本项目仅用于学习、研究和辅助代码生成，不构成任何投资建议、交易建议、收益承诺或风险控制保证。

量化交易涉及市场风险、流动性风险、滑点风险、模型失效风险、数据错误风险、接口异常风险和程序运行风险。任何由本项目生成、改写或参考的策略代码，在实盘使用前都应经过充分的人工审查、单元测试、历史回测、仿真测试和小资金验证。

本项目并非 VeighNa 官方项目，未获得 VeighNa 官方背书或保证。VeighNa/vn.py 的 API、模块结构和插件行为可能随版本变化而调整。若本手册内容与官方文档或你本地安装版本的源码不一致，应以官方文档和本地源码为准。

使用者应自行承担因使用本项目内容、生成代码、策略逻辑或实盘部署造成的任何损失和责任。

