# LEAN 期货文件格式与读取语义

## 基线与范围

2026-10-24 从已核对的 `Trader/TurtleTradingSystem/History/Readme.md`、History 历史生成代码及本机 `TradeLab/Lean` 源码整理。LEAN 读取侧基线为 TradeLab 提交 `9d25a5f2675fc3ccfdd19fc881560db19c0ec354`；History 命名基线为提交 `b1fc13d46aa6`，其后提交 `1da004d0` 已移除 LEAN 生成器与迁移工具。不要假定当前 History 仍会生成这些文件。

这里覆盖本地 `CHN` 国内期货数据集，不是所有证券类型的完整 LEAN 格式规范。`CHN` 前缀、`daily+`、4 小时平移、CNH 报价及历史映射策略是该数据集／本地扩展约定，不是任意 LEAN 数据的通用要求。源码版本或真实文件不一致时，以任务所用版本的源码与数据为准，报告差异。

`<output>/lean` 是历史目录；独立使用数据时将其视为 `<data-root>`，即 `Globals.DataFolder`。技能不依赖原始 checkout、服务、数据库或生产目录。示例均为合成数据，不代表真实行情或合约生命周期。完整交割年月应取合约资料或已核实的 SID，不能用交易日年份猜测。

CSV 使用 UTF-8、无 BOM、逗号分隔。标准行情 CSV 无表头；`daily+` 首行是 `#` 表头。价格按原值保存，必要的精确计算用十进制数，不能套用股票的整数价格倍率。品种可能含下划线（如 `v_f`、`l_f`、`pp_f`），解析名称时不要假设品种仅含字母。

## 目录与命名

```text
<output>/lean/
├── future/<exchange>/
│   ├── daily/
│   │   ├── chn<asset>_trade.zip
│   │   └── chn<asset>_openinterest.zip
│   ├── daily+/<deliveryYear>/chn<instrument>.csv
│   ├── minute/chn<asset>/
│   │   ├── <day>_trade.zip
│   │   └── <day>_openinterest.zip
│   ├── tick/chn<asset>/
│   │   ├── <day>_trade.zip
│   │   └── <day>_openinterest.zip
│   └── map_files/chn<asset>.csv
├── symbol-properties/symbol-properties-database.csv
└── market-hours/market-hours-database.json
```

LEAN 存储路径中的交易所、品种/合约目录、文件名及 ZIP entry 名称均为小写，例如 `future/shfe/daily/chnag_trade.zip`、`daily+/2026/chnag2601.csv`。目录树中的 `exchange`、`asset`、`instrument` 均取小写；CSV 内容不随路径转小写：`daily+` 行中的合约标识仍为 `CHNag2601`，参考数据保留 `SHFE`、`CHNag`，映射数据行中的 symbol/SID 仍为小写。

| 数据 | ZIP 路径 | ZIP entry 名称 |
| --- | --- | --- |
| 日线成交 | `daily/chn<asset>_trade.zip` | `chn<asset>_trade_<yyyyMM>.csv` |
| 日线持仓量 | `daily/chn<asset>_openinterest.zip` | `chn<asset>_openinterest_<yyyyMM>.csv` |
| 分钟成交/持仓量 | `minute/chn<asset>/<day>_<type>.zip` | `<day>_chn<asset>_minute_<type>_<yyyyMM>.csv` |
| tick 成交/持仓量 | `tick/chn<asset>/<day>_<type>.zip` | `<day>_chn<asset>_tick_<type>_<yyyyMM>.csv` |

表中的 `type` 为 `trade` 或 `openinterest`，`yyyyMM` 为四位交割年加两位交割月。LEAN 通过 `LeanData.GenerateZipEntryName` 调用 `FuturesExpiryUtilityFunctions.GetFutureContractMonth`，从合约 SID 日期推算该年月；此源码基线对 `CHN` 前缀期货使用交割月第一日作为到期日期。ZIP entry 只包含这一段交割年月，不再追加 `_<yyyyMM>01`，扩展名统一为单个 `.csv`。SID 日期仍使用交割月第一日，不能当成合约资料中的实际 `ExpiryDate`。旧格式中的大写路径、额外日期段和 `.csv.csv` 不属于此处的规范命名。

日线 ZIP 按品种累积，包内每个交割月一个 entry，该 entry 内保存此合约多个交易日的行。日内 ZIP 按品种、交易日和类型组织，包内每个交割合约一个 entry；即使合约没有记录，也会创建空 entry。eXp 的 `depth` 对应这里的 `tick`。

例如 `tick/chnag/20260916_trade.zip` 中可以包含 `20260916_chnag_tick_trade_202601.csv` 和同品种其他交割月的 entry；它不使用 `chnag2601.zip` 作为外层 ZIP 名称。

## 标准行情 CSV 内容

这些 ZIP entry 均无表头。LEAN 的 `TradeBar`、`Tick` 和 `OpenInterest` 读取器按以下顺序解释每条记录：

| 分辨率 | `trade` 字段顺序 | `openinterest` 字段顺序 |
| --- | --- | --- |
| daily | `Time,Open,High,Low,Close,Volume` | `Time,OpenInterest` |
| minute | `Milliseconds,Open,High,Low,Close,Volume` | `Milliseconds,OpenInterest` |
| tick | `Milliseconds,LastPrice,Quantity,Exchange,SaleCondition,Suspicious` | `Milliseconds,OpenInterest` |

日线 `Time` 格式为 `yyyyMMdd HH:mm`。History 历史写入值直接取日线对象的 `Time`，没有额外平移；Collector 通常将它设置为交易日 `00:00`。LEAN 的标准日线读取器将此值从 `config.DataTimeZone` 转为 `config.ExchangeTimeZone`；`TradeBar.Period` 取订阅周期，`EndTime = Time + Period`，因此标准日线的行时间与下文 `daily+` 的收盘时间处理不同。日线示例：

```csv
20260916 00:00,100,105,99,102,1234
```

分钟成交和持仓量示例（分别位于两个 ZIP 的对应 entry 中）：

```csv
3600000,100,105,99,102,1
```

```csv
3600000,100
```

tick 成交和持仓量示例：

```csv
3600000,101.25,1,SHFE,,0
```

```csv
3600000,100
```

分钟 `Volume` 为 bar 的成交量。tick 第三列在 LEAN 中读入 `Tick.Quantity`，表示该笔成交数量；`TickConsolidator` 会将它逐条累加为 bar 的 `Volume`。**History 存量 tick 实际写入源行情的当日累计成交量，没有计算逐笔增量，与 LEAN 的数量语义不一致；直接聚合这些 tick 会重复累计成交量。** 这是存量数据限制，不能将这一列视作已正确转换的逐笔数量。

tick 的 `SaleCondition` 历史写入值为空，`Suspicious=0`，交易所写入第四列；LEAN 将第六列 `1` 解释为可疑成交，其余值为 false。期货 `TradeBar` 价格按原值读取，`Tick` 对期货使用倍率 `1`。标准 LEAN trade 行不保存结算价、成交额、涨跌停、昨收/昨结；持仓量通过独立的 openinterest 文件提供。

国内期货日内时间编码、周末影响和必要的市场配置见 [china-time.md](china-time.md)；处理时间或实现转换时须先阅读。

## 扩展日线 daily+

`daily+/<deliveryYear>/chn<instrument>.csv` 是历史实现额外生成的扩展 CSV，每个合约累积多个交易日，具有与 eXp 日线完全相同的 15 列表头和字段顺序，仅 `InstrumentId` 增加 `CHN` 前缀：

```csv
#InstrumentId,ExchangeId,TradingDay,Open,High,Low,Close,Settlement,Volume,Turnover,OpenInterest,PreSettlement,PreClose,LowerBound,UpperBound
CHNag2601,SHFE,20260916,100,105,99,102,101,1234,124658,200,100,99,80,120
```

它保存标准 LEAN 日线未携带的结算价等字段，History 历史实现也使用它重建合约映射。此 LEAN 分支已有 `China/FutureDailyPlusBar.fs` 专用读取器，通过 `FutureAlgorithm.AddDailyPlus(underlying)` 订阅：

- `GetSource` 使用 `Globals.DataFolder/future/<market小写>/daily+/<SID日期年份>/<品种小写><yyMM>.csv`，例如 `future/shfe/daily+/2026/chnag2601.csv`；这里的 `Globals.DataFolder` 应指向 `<output>/lean`。
- `Reader` 跳过空行和 `#` 注释，按位置读取第 3 至第 15 列。前两列不用于恢复合约身份，返回对象的 `Symbol` 使用订阅的 `config.Symbol`。
- `TradingDay` 取第三列；`Time` 和 `EndTime` 均设为该交易日的上海时间 `15:00` 转换到 `config.ExchangeTimeZone` 后的时刻，而非标准日线的午夜或日内行情的 `+4` 写入规则。价格与数量字段直接按数值解析，`Value = Close`。

## 合约映射 map_files

`future/<exchange>/map_files/chn<asset>.csv` 没有表头。首行为起始记录，随后是日期、symbol/SID、交易所和 mode：

```text
19900101,CHNag,SHFE
<yyyyMMdd>,chnag <sid>,SHFE,<mode>
```

`sid` 由当时的 `Base/QcSecurityIdentifier.fs` 编码，包含交易所、期货类型和交割月第一日。History 该版本的渲染规则是：

- 每个已知合约在 SID 日期各输出一条 `mode=0` 和 `mode=1` 记录。
- `mode=2` 使用每日持仓量最高的合约，连续相同合约合并为一个区段，输出日期为该区段最后一个排名日；持仓量相同时按 SID 的 ordinal 顺序取较小者。
- 三种 mode 的记录合并后按日期排序。映射中的主力选择与 eXp `RankByOi` 的平局规则不同，读取时应分别遵循各自数据。

LEAN 通过 `MapFileRow.Parse` 读取日期、映射 symbol、可选交易所和可选 mode；映射 symbol 在内存中转为大写。`MapFile.GetMappedSymbol` 按日期升序寻找**第一条日期不早于查询日且 mode 为空或与请求 mode 匹配的记录**，所以行日期是该映射区间的结束日（含当日），不是新合约开始生效的日期；查询晚于末条适用记录时返回调用方提供的默认值。

LEAN 的 mode 枚举为 `0=LastTradingDay`、`1=FirstDayMonth`、`2=OpenInterest`，另支持 `3=OpenInterestAnnual`。读取器直接使用文件中的映射，不会重新比较持仓量或计算换月。上述 History 写法将 mode 0 和 1 都放在交割月第一日，mode 2 则按全品种每日最大持仓量分段；这描述的是存量文件的生成策略，不能据此断言它实现了 LEAN 枚举中所有换月策略的标准含义。

## 品种属性与交易时段

`symbol-properties/symbol-properties-database.csv` 的历史输出表头为：

```csv
market,symbol,type,description,quote_currency,contract_multiplier,minimum_price_variation,lot_size,market_ticker,minimum_order_size,price_magnifier
```

History 历史实现每个在市品种一行，只写入前 8 个字段，例如：

```csv
SHFE,CHNag,future,<品种名称>,CNH,15,0.5,1
```

其中交易所作为 `market`，报价货币固定为 `CNH`，乘数和最小价格变动来自合约资料，`lot_size=1`；表头末尾三个可选字段没有输出值。LEAN 的 `SymbolPropertiesDatabase.FromCsvLine` 接受该 8 列布局，缺省 `market_ticker` 为空、`minimum_order_size` 为 null、`price_magnifier` 为 `1`，还支持第 12 列 `strike_multiplier`（缺省为 `1`）。上述表头是 History 历史输出的表头，并非 LEAN 可接受字段的完整清单。历史更新会保留其他已有品种。

### Market hours database 的作用

`market-hours/market-hours-database.json` 为 LEAN 提供各市场、证券类型和品种的**数据时区与市场开闭市规则**。在标准加载流程中，`MarketHoursDatabase.FromDataFolder()` 从 `Globals.DataFolder` 下读取该文件；使用本文的目录布局时，`Globals.DataFolder` 应指向 `<output>/lean`。引擎按 market、security type 和品种查找配置，构造供订阅和交易所对象使用的时区及 `SecurityExchangeHours`。若消费者显式覆盖时区或交易时段，应以订阅实际生效的配置为准。

它直接参与以下处理，而不仅是文件格式的说明信息：

| 环节 | 具体作用 | 配置不匹配时的结果 |
| --- | --- | --- |
| 时间解释 | `dataTimeZone` 说明 CSV 时间所属时区，`exchangeTimeZone` 说明交易所时间及交易时段所属时区；读取器据订阅配置做时区转换。 | 同一行可能被解释为不同的 UTC 时刻，算法看到的日期和时间也会改变。 |
| 文件选日 | `Time.EachTradeableDayInTimeZone` 根据开市区间生成数据时区中的可交易文件日期，`SubscriptionDataReader` 再按这些日期获取数据源。 | ZIP 即使存在，若其日期对应的区间没有开市时段，也可能不进入正常读取流程。 |
| 行情过滤 | 标准数据馈送中的 `SubscriptionFilterEnumerator` 使用 `SecurityExchangeHours.IsOpen` 检查行情的 `Time`／`EndTime` 是否与允许交易的时段相交。 | CSV 行即使已成功解析，仍可能因处于休市区间而被跳过，不能据“文件里有记录”判断算法一定能收到。 |
| 向前填充 | 启用 fill-forward 时，填充器使用交易所开市规则判断可填充的区间及下一次开市时间。 | 错误的时段、节假日或延迟开盘配置会改变填充范围；休市日不能仅靠填充选项变成开市日。 |

对于 [国内期货 `+4` 方案](china-time.md)，行情文件、这个数据库和订阅配置必须配套：例如 `ag` 原始上海时间 `21:00–02:30` 的夜盘，在交易日文件中写为 `01:00–06:30`，数据库也须在同一交易日配置平移后的时段，并使用配套的 `Pacific/Wake` 时区。若 CSV 已平移而数据库仍保留原始 `21:00–02:30` 时段，记录就可能落在配置的休市区间而被过滤。数据库不会修改 CSV 中的时间，也不会自动恢复跨周末夜盘的真实自然日期或计算 CTP 交易日归属。

历史输出字段如下；这些是该数据集的配置值，并非 LEAN 对所有市场的固定要求。

`market-hours/market-hours-database.json` 的顶层为 `entries` 对象，历史输出的键形如 `Future-SHFE-CHNag`，每个值包含：

- `dataTimeZone`、`exchangeTimeZone`：均为 `Pacific/Wake`，与日内行情加 4 小时后的时间表达配套。
- `monday` 至 `friday`：由品种交易时段生成的 `start`、`end`、`state` 列表；时段也加 4 小时，状态为 `market`。
- `holidays`：交易日历覆盖区间内、非周末且未列为交易日的日期，格式为 `MM/dd/yyyy`。
- `lateOpens`：根据交易日历判断没有前置夜盘的交易日，设置为平移后的 `13:00:00`；`earlyCloses` 当前为空对象。

LEAN 的参考数据查询键 `SecurityDatabaseKey` 对 market 和 symbol 使用忽略大小写的比较，因此参考文件中的 `SHFE`、`CHNag` 可以保留原样；这与行情文件路径必须按小写生成是两套规则。交易时段 JSON 由 `MarketHoursDatabaseJsonConverter` 读取，时区取自文件中的配置，读取器不会自行替换成上海时区。

## 消费与校验

检查 ZIP 的真实 entry 名称，按精确名称读取。日线 ZIP 按品种累积多个合约 entry，日内 ZIP 按交易日／类型组织多个合约 entry；不能把整个 ZIP 当作一个合约的 CSV。区分文件不存在、entry 不存在和空 entry。旧格式大小写、重复交割日期段或 `.csv.csv` 须显式识别为旧版本，不静默混用。

只读检查示例（在 `<data-root>` 下）：

```bash
unzip -l future/shfe/minute/chnag/20261026_trade.zip
unzip -p future/shfe/minute/chnag/20261026_trade.zip \
  20261026_chnag_minute_trade_202701.csv
```

对实际任务同时验证文件、解析后的时间和引擎是否接收记录。连续合约映射不等于价格复权：`map_files` 指定合约选择，不能仅凭映射文件宣称已支持所有复权模式；需要复权时检查目标 LEAN 版本要求的 factor 数据及配置。

## 源码核对入口

下列路径相对 `TradeLab/Lean/`，是可选的原始证据定位，不是技能运行依赖：

- `Common/Util/LeanData.cs`：`GenerateRelativeZipFileDirectory`、`GenerateZipFileName`、`GenerateZipEntryName`。
- `Common/Data/Market/{TradeBar,Tick,OpenInterest}.cs`：CSV 解析、价格倍率和时间转换。
- `China/FutureDailyPlusBar.fs`、`China/FutureAlgorithm.fs`：扩展日线路径、字段、收盘时间与订阅入口。
- `Common/Data/Auxiliary/{MapFile,MapFileRow}.cs`、`Common/Global.cs`：映射区间与 mode。
- `Common/Securities/Future/{FuturesExpiryUtilityFunctions,FuturesExpiryFunctions}.cs`：交割月与 CHN 到期日期。
- `Common/Securities/{MarketHoursDatabase,SymbolPropertiesDatabase,SecurityDatabaseKey}.cs`、`Common/Util/MarketHoursDatabaseJsonConverter.cs`：参考数据读取。
- `Common/Data/Consolidators/TickConsolidator.cs`：逐笔数量累加为 bar 成交量。

History 历史写入逻辑见 Trader 提交 `b1fc13d46aa6` 中的 `TurtleTradingSystem/History/{LeanNaming,QcReshaper,HistoryMapping}.fs`。不能将历史模块不存在误判为需要恢复生成器或执行迁移。
