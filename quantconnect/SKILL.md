---
name: quantconnect
description: "使用 QuantConnect／LEAN 编写、调试和优化交易算法，处理数据、指标、选股、期货、期权及基本面字段，查询 Cloud 导出与定价规则；支持 TradeLab 本地 F# China 国内期货扩展和 Fathom 回测／优化启动器。适用于明确使用 QuantConnect、LEAN、QuantConnect.China 或 Fathom 的任务，不用于一般投资建议。"
license: Apache-2.0
---

# QuantConnect / LEAN

统一使用 QuantConnect 官方技能及 TradeLab 本地扩展中的算法开发、数据与平台指引。先确认用户任务、语言、项目运行环境和目标版本，再按需读取相关主题；无需一次加载全部字段表或另一种语言的示例。

## TradeLab 本地扩展

本机源码根目录为 `~/repos/eXp.nXt/TradeLab`，LEAN 引擎在其 `Lean/` 下；China 位于 `Lean/China`，Fathom 位于同级的 `Fathom/`。这些路径是已核对的本机位置，其他环境应先定位对应 checkout。

- [China：国内期货算法与交易模型](references/tradelab/china.md)：涉及 F# `FutureAlgorithm`、`FutureDailyPlusBar`／`AddDailyPlus`、CNH、结算价、保证金、手续费、悲观滑点及国内合约身份时先读。
- [Fathom：本地回测与参数优化](references/tradelab/fathom.md)：涉及 `Backtests`／`Fathom` 启动、`--optimisation`、`.o.json`、TUI、子进程和共享内存数据提供器时先读。

本地扩展的专用契约优先于下方上游通用示例：先确认是否继承 `QuantConnect.China.FutureAlgorithm`，以及实际使用的模型、合约和数据配置。上游 Python/C# 示例不代表这些 F# 扩展已经在 Cloud 或其他 LEAN 发行版中安装。

## 使用方式

1. **选择语言与环境。** Python 查 `references/python/`，C# 查 `references/csharp/`；平台共用内容查 `references/shared/`。保留用户指定的本地 LEAN、LEAN CLI、Research 或 Cloud 环境。以项目实际版本、签名和源码为准，不因参考示例自动升级依赖。
2. **读取相关主题。** 下表链接到本包的普通参考文件，整个目录只有这一个技能入口。参考正文保留了上游措辞，其中 “X skill”、`/debug-algorithm`、`/debug-performance`、`/custom-indicator` 均指下表对应主题，不要求另装技能或存在同名斜杠命令。每份指南开头的 `Upstream scope` 保留原适用条件。
3. **组合必要指引。** 动态 universe 先读 `universes`，再读具体选股主题；基本面字段先读 `equity-fundamental-data`，再查对应字段族。滚动统计先看 `indicators`，一次性回看再看 `history`。期权区分单标的订阅、动态链、历史数据和多腿交易。下单与调度问题同时核对 `orders`、`scheduled-events` 及资产类别指引。
4. **适配可用工具。** 上游的 `quantconnect:*` 编译、文件和回测步骤是工具示例。先检查当前连接器实际能力；没有 MCP 时使用项目已有的本地编辑、构建或 LEAN CLI 流程。尚无运行环境时完成可离线验证的部分，明确哪些编译、取数或回测未执行。云端写入、付费运行、数据购买、实盘部署与交易遵循用户授权范围；本技能不自动授权这些操作。
5. **验证实际结果。** 检查订阅与数据类型、bar 的起止时间及可用时点、时区、指标就绪和 warm-up、订单状态。诊断零交易时先判断策略是否满足入场条件；保留原策略语义，不为了产生订单而改变信号。报告实际版本、验证方式与未验证部分，不能把语法通过称为回测成功。

## 算法与数据主题

表中主题名也用于解析参考正文里的跨主题指引；选择用户语言对应的链接。

| 主题 | 何时查阅 | Python | C# |
| --- | --- | --- | --- |
| `calendar` | 交易日、节假日、开闭市时间 | [指南](references/python/calendar/guide.md) | [指南](references/csharp/calendar/guide.md) |
| `scheduled-events` | 定时任务、再平衡、时区与数据分辨率 | [指南](references/python/scheduled-events/guide.md) | [指南](references/csharp/scheduled-events/guide.md) |
| `history` | 一次性历史回看、预热、上一交易日与当前 session | [指南](references/python/history/guide.md) | [指南](references/csharp/history/guide.md) |
| `indicators` | 流式指标、滚动统计、consolidator 与指标预热 | [指南](references/python/indicators/guide.md) | [指南](references/csharp/indicators/guide.md) |
| `custom-indicator` | 自定义指标类及算法集成 | [指南](references/python/indicators/custom-indicator/guide.md) | [指南](references/csharp/indicators/custom-indicator/guide.md) |
| `universes` | 所有动态 universe 共用的订阅、分辨率、调度和预热规则 | [指南](references/python/universes/guide.md) | [指南](references/csharp/universes/guide.md) |
| `fundamental-universes` | Morningstar 基本面选股、期间值与同比 | [指南](references/python/universes/equity/fundamental-universes/guide.md) | [指南](references/csharp/universes/equity/fundamental-universes/guide.md) |
| `indicator-universes` | 按每个证券的指标筛选 universe | [指南](references/python/indicators/indicator-universes/guide.md) | [指南](references/csharp/indicators/indicator-universes/guide.md) |
| `cross-sectional` | 横截面排序、缩小候选集、目标权重 | [指南](references/python/universes/cross-sectional/guide.md) | [指南](references/csharp/universes/cross-sectional/guide.md) |
| `alternative-data-universes` | 另类数据 universe 及其选择对象 | [指南](references/python/datasets/alternative-data-universes/guide.md) | [指南](references/csharp/datasets/alternative-data-universes/guide.md) |
| `crypto` | 交易所原生加密货币 universe、成交额与做空 | [指南](references/python/crypto/guide.md) | [指南](references/csharp/crypto/guide.md) |
| `futures` | 期货合约数量、名义敞口、连续合约历史与预热 | [指南](references/python/futures/guide.md) | [指南](references/csharp/futures/guide.md) |
| `equity-options` | 单一股票标的期权订阅、链、Greeks/IV 与时效 | [指南](references/python/equity-options/guide.md) | [指南](references/csharp/equity-options/guide.md) |
| `chained-universes-options` | 动态股票 universe 链接期权 universe | [指南](references/python/universes/equity/chained-universes-options/guide.md) | [指南](references/csharp/universes/equity/chained-universes-options/guide.md) |
| `historical-data-equity-options` | 历史期权价格、IV、Greeks 与日内计算 | [指南](references/python/historical-data-equity-options/guide.md) | [指南](references/csharp/historical-data-equity-options/guide.md) |
| `option-strategies` | 多腿期权、组合订单、保证金、指派与到期 | [指南](references/python/option-strategies/guide.md) | [指南](references/csharp/option-strategies/guide.md) |
| `orders` | 订单、目标权重、退出、成交与费用模型 | [指南](references/python/orders/guide.md) | [指南](references/csharp/orders/guide.md) |
| `alternative-data` | 另类数据订阅及单点／列表返回形状 | [指南](references/python/datasets/alternative-data/guide.md) | [指南](references/csharp/datasets/alternative-data/guide.md) |
| `economic-surveys-data` | BLS CPI、CES、PPI、JOLTS 与发布时间语义 | [指南](references/python/datasets/economic-surveys-data/guide.md) | [指南](references/csharp/datasets/economic-surveys-data/guide.md) |
| `custom-data` | 自定义 CSV／JSON／ZIP／外部数据读取器 | [指南](references/python/importing-data/custom-data/guide.md) | [指南](references/csharp/importing-data/custom-data/guide.md) |
| `equity-fundamental-data` | 基本面字段路径规则、顶层字段与财报元数据；查字段先读此项 | [指南](references/python/equity-fundamental-data/guide.md) | [指南](references/csharp/equity-fundamental-data/guide.md) |
| `fundamental-income-statement` | 利润表：收入、成本、净利润、EBIT/EBITDA | [指南](references/python/equity-fundamental-data/fundamental-income-statement/guide.md) | [指南](references/csharp/equity-fundamental-data/fundamental-income-statement/guide.md) |
| `fundamental-balance-sheet` | 资产负债表：资产、负债、权益、债务与股数 | [指南](references/python/equity-fundamental-data/fundamental-balance-sheet/guide.md) | [指南](references/csharp/equity-fundamental-data/fundamental-balance-sheet/guide.md) |
| `fundamental-cash-flow-statement` | 现金流量表：经营／投资／融资、capex 与分红 | [指南](references/python/equity-fundamental-data/fundamental-cash-flow-statement/guide.md) | [指南](references/csharp/equity-fundamental-data/fundamental-cash-flow-statement/guide.md) |
| `fundamental-ratios` | 经营、估值与盈利比率 | [指南](references/python/equity-fundamental-data/fundamental-ratios/guide.md) | [指南](references/csharp/equity-fundamental-data/fundamental-ratios/guide.md) |
| `fundamental-company-data` | 盈利报告、公司／证券参考、公司资料与 IPO 日期 | [指南](references/python/equity-fundamental-data/fundamental-company-data/guide.md) | [指南](references/csharp/equity-fundamental-data/fundamental-company-data/guide.md) |
| `fundamental-classification` | 行业、板块、分类字段与命名代码常量 | [指南](references/python/equity-fundamental-data/fundamental-classification/guide.md) | [指南](references/csharp/equity-fundamental-data/fundamental-classification/guide.md) |
| `debug-algorithm` | 运行异常、缺失属性、零订单与信号链排查 | [指南](references/python/debug-algorithm/guide.md) | [指南](references/csharp/debug-algorithm/guide.md) |
| `debug-performance` | 回测 CPU／内存／耗时、Performance Chart 与 profiler | [指南](references/python/debug-performance/guide.md) | [指南](references/csharp/debug-performance/guide.md) |
| `data-issues` | 在 Research 验证可疑数据、按 Security ID 排除异常资产 | [指南](references/python/data-issues/guide.md) | [指南](references/csharp/data-issues/guide.md) |
| `logging` | 算法日志、配额、重复抑制与诊断数据 | [指南](references/python/logging/guide.md) | [指南](references/csharp/logging/guide.md) |
| `notifications` | 实盘通知、信号导出、外部命令与多算法通信 | [指南](references/python/live-trading/notifications/guide.md) | [指南](references/csharp/live-trading/notifications/guide.md) |
| `code-quality` | 算法代码风格、订阅变量与错误处理 | [指南](references/python/code-quality/guide.md) | [指南](references/csharp/code-quality/guide.md) |

## 平台共用主题

- [data-export：数据导出](references/shared/data-export/guide.md)：Object Store、文件、图表、日志、订单标签、通知和导出限制。编写相关代码时同时核对；将 QuantConnect Cloud 数据使用规则与本地用户自有数据区分。
- [pricing：平台与数据定价](references/shared/pricing/guide.md)：套餐、席位、节点、QCC、Cloud Access、Download 与成本估算。指南链接到本包保留的五份价格和计费参考文件。

价格、套餐、配额、许可与平台功能可能变化。报价或据此提出购买／导出方案前，查阅对应指南链接的当前官方页面并标明查证日期；离线时将本包中的数字明确标为历史快照。上游的 “verified” 和案例日期是原作者的历史验证记录，不代表已在用户当前版本复现。

## 来源与维护基线

- 来源：[QuantConnect/Documentation 的 skills 目录](https://github.com/QuantConnect/Documentation/tree/4f39bafb4bee339ec252276b2cfae98619e92ab0/skills)，获取日期：2026-10-25，固定提交：`4f39bafb4bee339ec252276b2cfae98619e92ab0`。
- 覆盖 Python、C# 各 35 个上游技能及其全部 10 份定价参考；两种语言内容完全相同的 `data-export`、`pricing` 及定价参考只保存一份。语言相关示例与完整字段表保持分开。
- 原技能改为 `guide.md`，原 frontmatter 中的适用条件移为 `Upstream scope`，加入来源与修改说明，正文保留。跨主题名称通过此入口解析；定价指南的相对链接保持可用。
- [sources.json](sources.json) 逐项记录上游路径、Git blob、目标文件与转换方式；[LICENSE](LICENSE) 保留上游 Apache-2.0 许可。更新时先核对固定提交与各文件来源，再检查主题覆盖、相对链接和单一入口。
- 本地扩展另以 TradeLab 提交 `94c61738d86d102ac3ca7f1f6398ba9272a2f56c` 为源码基线，核对日期为 2026-10-25；证据路径与 blob 记录在 `sources.json` 的 `local_extensions`，与官方技能来源分开维护。
