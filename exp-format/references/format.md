# eXp 格式

`eXp` 是本格式的名称，用户提及它时不区分大小写，例如 `exp`、`EXP`、`Exp`。文件路径、CSV 字段名、交易所及合约代码仍保留原有大小写。

## 格式基线与来源

2026-10-23 核对本地 Trader 仓库提交 `247dae27aed2c3ff8409dca6f218b6c29ded1a66` 的以下文件：

- `TurtleTradingSystem/History/Readme.md`：eXp 布局及字段说明。
- `History/Writer.fs`（相对于 `TurtleTradingSystem/`）：路径、四种基础 CSV 表头和序列化。
- `History/HistoryRanking.fs`、`History/HistoryAdjustment.fs`：排名、换月表头和计算。
- `History/HistoryRankStore.fs`、`History/HistoryCoordinator.fs`、`History/HistoryFileStore.fs`：发布状态、生成流程和 UTF-8 编码。

原始检索位置为 `~/repos/eXp.nXt/Trader/TurtleTradingSystem/History`，供有此 checkout 的环境核对；技能不依赖该绝对位置。Readme 仍含 LEAN 说明，而此提交已无 LEAN 生成器；本参考只覆盖 eXp。以上为源码证据，不证明任一部署版本、数据覆盖或在线服务状态。

## 生成与通用约定

Collector 在收盘后补齐分钟 bar、生成日线并发布 `TradingDayFinished day`。History 固定输入快照，准备、校验并发布输出。数据根目录 `<output>` 来自 `ConnectionStrings:HistoryOutputFolder`，相对路径以 History content root 解析；消费已有文件时直接使用用户提供的根目录。

CSV 使用 UTF-8、无 BOM、逗号分隔，第一行以 `#` 开头列出字段。日期 `yyyyMMdd` 应作为日期或字符串读取。数字不带额外的整数价格倍率，原始价格未经主力复权。本文示例为合成数据，不代表真实行情或合约生命周期。

`exchange` 如 `SHFE`、`DCE`、`CZCE`、`CFFEX`、`INE`、`GFEX`。`asset` 为品种代码；`instrument` 为品种加四位年月，例如 `ag2701`，郑商所也使用四位年月。保留大小写；品种可含下划线，例如 `v_f`。完整交割年/月应取合约资料，不以交易日年份或仅凭两位年份推断。

## 路径与归档

```text
<output>/
├── catalogue.csv
├── listed/<tradingYear>/<day>.csv
├── daily/<exchange>/<deliveryYear>/<instrument>.csv
├── minute/<exchange>/<deliveryYear>/<instrument>.zip
├── depth/<exchange>/<deliveryYear>/<instrument>.zip
├── ranks/<exchange>/<asset>/<tradingYear>.csv
└── rollovers/<exchange>/<asset>/<rule>/<switchYear>.csv
```

| 数据 | 组织方式 |
| --- | --- |
| catalogue | 累积合约资料，更新本次在市合约并保留历史合约；纠正旧交易日不回退最新目录。不是任意历史日的在市集合。 |
| listed | 指定交易日的在市快照，字段与 catalogue 相同。 |
| daily | 每合约一个累积 CSV，每交易日一行，按交易日排列。 |
| minute / depth | 每合约一个累积 ZIP，包内根目录每交易日一个 `yyyyMMdd.csv`；不是按自然日期拆包或按品种拆包。无日内记录时当日 entry 仍保留表头。 |
| ranks | 每交易日、每个有日线的合约一行，按交易日年份归档。 |
| rollovers | `rule` 为 `ByOi` 或 `ByVol`，按 `SwitchDay` 年份归档。 |

例如 2026 年交易的 2027 年交割合约，文件为 `minute/SHFE/2027/ag2701.zip`，其中可含 `20260916.csv`。同一天纠正会替换目标交易日的行或 ZIP entry，保留其他日期；不能靠无条件追加实现兼容写入。

## 合约目录 / 在市快照：17 列

```csv
#InstrumentId,ExchangeId,CreateDate,OpenDate,ExpiryDate,DeliveryStartDate,DeliveryEndDate,DeliveryYear,DeliveryMonth,UnderlyingId,MinimalPriceVariation,Multiplier,UnderlyingMultiplier,Phase,MaxMarginSideAlgorithm,PositionType,PositionDateType
```

五个日期列使用 `yyyyMMdd`，`ExpiryDate` 是实际到期日。`MinimalPriceVariation` 是最小价格变动，`Multiplier` 是合约乘数，`UnderlyingId`、`UnderlyingMultiplier` 是标的资料。最后四列为 CTP 枚举 byte 数值，而非名称，例如 `Phase=49` 为 `Started`，未知值为 `0`。

这些文件不包含完整品种资料、交易时段或限价/市价委托量上下限，不能仅凭此 CSV 恢复这些资料。

## 日线：15 列

```csv
#InstrumentId,ExchangeId,TradingDay,Open,High,Low,Close,Settlement,Volume,Turnover,OpenInterest,PreSettlement,PreClose,LowerBound,UpperBound
ag2701,SHFE,20260916,100,105,99,102,101,1234,124658,200,100,99,80,120
```

不保存日线 `Time`。OHLC 为全日开高低收；`Settlement` 为结算价，`Volume`、`Turnover` 为全日成交量、额，`OpenInterest` 为期末持仓量；`PreSettlement`、`PreClose` 为昨结、昨收，`LowerBound`、`UpperBound` 为跌停、涨停价格。来自 Collector 末次行情生成的 `FuturesDailyBar`，History 不重新聚合。

## 分钟 bar：12 列

```csv
#InstrumentId,ExchangeId,TradingDay,Time,Open,High,Low,Close,Settlement,Volume,Turnover,OpenInterest
ag2701,SHFE,20260916,2026-09-15 21:00,100,105,99,102,101,1,0.00,100
```

`Time` 为 `yyyy-MM-dd HH:mm`，保留自然日期及原始分钟时间，不加 4 小时。夜盘可在交易日前一个自然日，也可能跨周末或假期；不能用 `TradingDay - 1` 推算所有夜盘日期。文件不声明时间戳是 UTC，也未声明 bar 起止标签约定；需要转换或对齐其他源时核对生成代码和实际样本。

OHLC 为 bar 聚合值，`Volume`、`Turnover` 为该 bar 增量，`OpenInterest`、`Settlement` 取 bar 末次行情。

## 行情快照 depth：20 列

```csv
#InstrumentId,ExchangeId,TradingDay,CalendarDay,Time,Volume,Turnover,OpenInterest,LastPrice,OpenPrice,ClosePrice,HighestPrice,LowestPrice,SettlementPrice,AveragePrice,UpperLimitPrice,LowerLimitPrice,PreSettlementPrice,PreClosePrice,PreOpenInterest
ag2701,SHFE,20260916,20260915,21:00:00.000,1,0.00,100,101.25,100,102,105,99,101,101.5,120,80,100,99,80
```

`TradingDay`、`CalendarDay` 分别是交易日、自然日期，格式 `yyyyMMdd`。`Time` 为 `HH:mm:ss.fff`，用 `CalendarDay + Time` 构造自然时间，仍用 `TradingDay` 归属交易日。

`Volume`、`Turnover` 为交易日累计量、额，不能对所有快照直接求和作为日成交量、额。若任务需要增量，需处理交易日边界、缺失首条和累计值异常，不能默认首条之前累计为零。`OpenInterest` 为当前持仓，`PreOpenInterest` 为昨持仓。

价格列保留源快照值；系统可能以 `-1` 表示无效价格，History 不改为空。不要把该哨兵参与收益或均值计算，也不要把所有非正价格无条件判为此哨兵。该格式没有 bid/ask 及盘口数量，不支持从这些文件恢复盘口。

## 排名：9 列

```csv
#TradingDay,ExchangeId,AssetId,InstrumentId,ExpiryDate,Volume,OpenInterest,RankByVol,RankByOi
```

排名从 `1` 开始，在相同交易日、交易所、品种内分别按成交量或持仓量降序。平局先按实际 `ExpiryDate` 升序，再按 `InstrumentId` 的 ordinal 顺序，不是随意取文件第一行。`RankByOi=1` 与 `RankByVol=1` 可是不同合约。

缺日线的合约不生成排名行，`.history-state/rank-coverage/<day>.csv` 可记录 `MissingDailyBar`。缺排名不能直接解释为零成交或停牌。排名来自该日收盘，选择只能在收盘后已知；日内或开盘决策需使用当时已经发布的更早排名，按交易日历衔接。

## 换月：16 列

```csv
#SwitchDay,KnownAfterTradingDay,ReferenceDay,ExchangeId,AssetId,Rule,OldInstrumentId,NewInstrumentId,ReferencePolicy,OldPrice,NewPrice,ForwardDelta,BackwardDelta,BackwardRatio,AdditiveStatus,RatioStatus
20260916,20260916,20260915,SHFE,ag,ByOi,ag2612,ag2701,PreviousTradingDayClose,100,102,-2,2,1.02,Valid,Valid
```

相邻排名日第一名改变时生成记录。`SwitchDay` 是新合约成为第一名的排名日，`KnownAfterTradingDay` 当前与它相同，表示到该日收盘后才知道此次选择；不是允许在该日开盘切换的指令。

`ReferencePolicy` 固定为 `PreviousTradingDayClose`。`ReferenceDay` 为交易日历中换月日前一交易日，参考两合约该日的收盘价：

- `ForwardDelta = OldPrice - NewPrice`。
- `BackwardDelta = NewPrice - OldPrice`。
- `BackwardRatio = NewPrice / OldPrice`。

这些是单次换月值，未自动累计，也未修改原始行情。状态包括 `Valid`、`ReferenceCalendarMissing`、`CoverageGap`、`MissingReference`、`NonPositiveReference`。缺日历、排名覆盖或参考价时相应字段可为空。非正参考价使比率不可用，但加法因子仍可能为 `Valid`，因此分别检查两个状态；按列解析空值为缺失值。

需要累计复权时，明确方法、锚定日、事件链及应用区间；跨年读取所需换月文件，不混合 `ByOi`/`ByVol`，不把因子直接作用于成交量或持仓量。

## 发布与覆盖

单个 CSV/ZIP 存在不代表整日发布完成。`.history-runs/<day>/<runId>/manifest.json` 记录运行状态和输出操作；准备快照、候选文件和恢复材料可能在 `<History:WorkFolder>/.history-runs/`，不能当作发布数据。未另设 WorkFolder 时工作目录与输出根目录可重合。

排名及换月的 `.history-state/rank-series/<exchange>/<asset>.json` 含 `Version`、`CompletionPath`、`CompleteHistory`、`Files`。要求与 History 已完成发布一致时，沿该系列实际的 `CompletionPath` 读取完成标记，检查 `State=Completed`、版本及完整性标志，核对文件相对路径、大小和哈希；并在读取前后确认发布索引和所读文件未改变。不要用其他运行的完成标记替代该路径，或仅凭全局某个 manifest 已完成判断该系列可用。

仅有离线 CSV/ZIP 子集、没有状态文件时，仍可做文件级解析，但应说明无法验证整次发布完整性和当时实际可用时点，不伪造状态文件。

## 最小只读示例

在数据根目录检查 entry，无需解压整包：

```bash
unzip -l minute/SHFE/2027/ag2701.zip
unzip -p minute/SHFE/2027/ag2701.zip 20260916.csv
```

Python 标准库可流式读取；替换成实际归档位置：

```python
import csv
import io
from decimal import Decimal
from zipfile import ZipFile

expected = (
    "InstrumentId,ExchangeId,TradingDay,Time,Open,High,Low,Close,"
    "Settlement,Volume,Turnover,OpenInterest"
).split(",")
with ZipFile("minute/SHFE/2027/ag2701.zip") as archive:
    with archive.open("20260916.csv") as raw:
        with io.TextIOWrapper(raw, encoding="utf-8", newline="") as stream:
            reader = csv.reader(stream)
            header = next(reader, None)
            if header != ["#" + expected[0], *expected[1:]]:
                raise ValueError(f"Unexpected minute header: {header!r}")
            for line_number, values in enumerate(reader, start=2):
                if len(values) != len(expected):
                    raise ValueError(f"Invalid minute row at line {line_number}: {values!r}")
                row = dict(zip(expected, values))
                if row["TradingDay"] != "20260916":
                    raise ValueError(f"Trading day mismatch at line {line_number}")
                close = Decimal(row["Close"])
                # Consume the row here; do not infer a timezone from the timestamp.
```

这个示例演示表头处理及流式读取，不替代完整的合约校验、业务计算或发布一致性验证。
