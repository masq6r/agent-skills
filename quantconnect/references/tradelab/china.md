# China：TradeLab 国内期货扩展

适用于 `QuantConnect.China`、国内期货模型和 DailyPlus 数据。源码根目录下文记为 `$TradeLab`；本机为 `~/repos/eXp.nXt/TradeLab`。基线为 `94c61738d86d102ac3ca7f1f6398ba9272a2f56c`，核对日期为 2026-10-25。以下来自当前源码和测试阅读；运行用户任务前仍须核对其版本与有效数据配置。

## 用途和接入位置

`Lean/China/China.fsproj` 是 `net10.0` F# 类库，引用 LEAN Common 和 Algorithm，命名空间为 `QuantConnect.China`。它提供算法基类、自定义日行情和四个交易模型。`Backtests` 是消费该类库的策略工程；`Fathom` 是启动器，见 [fathom.md](fathom.md)。China 不提供行情下载器、CTP 网关或国内市场日历生成器。

| 类型 | 当前用途与契约 |
| --- | --- |
| `FutureAlgorithm` | 继承 `QCAlgorithm`，设置上海算法时区、CNH 账户币种、按自然日复利的固定收益基准，并注册自定义证券初始化器 |
| `FutureDailyPlusBar` | 继承 `TradeBar`，读取本地日行情的结算价、昨结、昨收、涨跌停价、成交额和持仓量 |
| `FixedRatioFeeModel` | 按订单名义价值收取固定比例手续费，返回报价币种的 `CashAmount` |
| `FixedRatioFutureMarginModel` | 固定比例初始／维持保证金；初始使用当前价，维持优先使用结算价 |
| `FutureSettlementModel` | 期货逐日盯市结算，将已结算盈亏转入报价币种现金，平仓时避免重复支付 |
| `PessimisticSlippageModel` | 以当前 bar 的不利极值加 tick 偏移估算滑点，适合 TradeBar 输入 |

初始化器先调用 LEAN 基础初始化，再为所有 `Future` 替换上述四个模型，**没有按中国 market 过滤**；混合市场策略也会受影响。股票仍使用其原有模型。`SecuritySeeder.Null` 不主动补种价格；首次下单前检查价格、数据和指标是否就绪。自行再次 `SetSecurityInitializer` 会替换已有初始化逻辑，应保留或明确重建需要的部分。

## 算法参数

这些名称在 `FutureAlgorithm` 构造时通过 `GetParameter` 读取。**当前初始化顺序存在限制：普通回测先构造算法实例，随后 `BacktestingSetupHandler.Setup` 才调用 `algorithm.SetParameters(job.Parameters)`。** `QCAlgorithm` 的参数字典初始为空，因此在该路径上，基类已捕获下表默认值；配置里即使提供同名参数，也不能据此认定模型或基准已经改变。子类在 `Initialize` 中能读到 job 参数，但不会重新执行基类构造。按名称做费率／保证金／滑点敏感性分析前，必须验证实际模型；如需修复构造时机，作为单独的实现任务处理。

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| `slippage` | `0` | 不利 bar 极值之外的 tick 数，偏移为参数 × `MinimumPriceVariation` |
| `fee-ratio` | `0.00005` | 固定比例手续费；`0.00005` 为万分之 0.5 |
| `margin-ratio` | `0.15` | 固定保证金比例 |
| `annual-return-rate` | `0.05` | 基准的年收益率，按 `(1 + rate)^(经过自然天数 / 365)` 计算 |

单独构造 `FixedRatioFeeModel()` 的默认费率为零，与 `FutureAlgorithm` 的默认值不同。固定费率不表达按手收费、平今差异、交易所阶梯或动态保证金；需要这些行为时检查并替换相应模型，不能把默认值当作真实交易所规则。

**`slippage=0` 仍有悲观滑点。** 买单返回 `High - Close + ticks × tickSize`，卖单返回 `Close - Low + ticks × tickSize`。在按 Close 加减滑点的成交路径上，结果分别为 High 加偏移、Low 减偏移；具体订单的最终成交价仍取决于 LEAN fill model，不能声称所有订单都精确按极值成交。

## 最小 F# 接入示例

在已有、引用 `China.fsproj` 的策略工程中添加类型，并将文件列入 `.fsproj` 的 `Compile` 清单。下例只演示订阅和读取；合约与日期来自现有 demo，必须换成实际有数据的区间。

```fsharp
namespace Example

open System
open QuantConnect
open QuantConnect.China

type ChinaDataProbe() =
    inherit FutureAlgorithm()

    let mutable dailyPlusSymbol = Unchecked.defaultof<Symbol>

    override me.Initialize() =
        me.SetStartDate(2020, 12, 16)
        me.SetEndDate(2020, 12, 18)
        me.SetCash(450000m)
        let contract = Symbol.CreateFuture("chnag", Market.SHFE, DateTime(2021, 12, 1))
        me.AddFutureContract(contract, Resolution.Minute, false) |> ignore
        dailyPlusSymbol <- me.AddDailyPlus(contract, fillForward = false).Symbol

    override me.OnData(slice) =
        match slice.Bars.TryGetValue(dailyPlusSymbol) with
        | true, (:? FutureDailyPlusBar as bar) ->
            me.Log $"{bar.TradingDay}: close={bar.Close}, settlement={bar.SettlementPrice}"
        | _ -> ()
```

`AddDailyPlus(underlying, ?fillForward)` 使用 Daily 分辨率，默认 `fillForward=false`，返回自定义数据证券；保存其 `.Symbol`。这个 symbol 的 `.Underlying` 是可交易期货合约，不能把自定义数据 symbol 当成下单目标。可用 `AddData<FutureDailyPlusBar>` 手工订阅；只继承 `QCAlgorithm` 并订阅 DailyPlus **不会自动安装 China 的交易模型**。

该分支将继承 `TradeBar` 的 DailyPlus 数据放入 `slice.Bars`，按自定义数据 symbol 取值后向下转换。不要只检查 `slice.ContainsKey(contractSymbol)` 就假定 DailyPlus 到达；也不要假定每根分钟 bar 都有 DailyPlus。示例为 F# 用法；C# 调用 F# optional 参数时需按实际签名处理 `FSharpOption`，不要直接照搬 C# optional 参数语法。

## 数据、时间与合约身份

- 标准行情仍用 `AddFutureContract` 或 `AddFuture`。本地合约根名称带 `chn`，例如 `chnag`；真实 market 常量在本地 `Lean/Common/Market.cs` 中，包括 SHFE、INE、CFFEX、DCE、CZCE、GFEX。`Symbol.CreateFuture` 的日期是合约身份的一部分，按实际文件／数据库约定构造。
- `FutureDailyPlusBar.GetSource` 使用 `Globals.DataFolder/future/<market小写>/daily+/<交割年>/<根名小写><yyMM>.csv`，例如 `future/shfe/daily+/2021/chnag2112.csv`。交割年月来自订阅 symbol，不是回测当天；为普通本地 CSV，不是标准 daily ZIP。
- CSV 为 15 列：`contract,market,tradingDay,open,high,low,close,settlementPrice,volume,turnover,openInterest,preSettlementPrice,preClosePrice,lowerBound,upperBound`。日期为 `yyyyMMdd`。Reader 实际读取第 3–15 列，身份来自订阅配置，不能靠前两列自动纠正错误订阅。空行和以 `#` 开头的注释跳过；坏日期、坏数值或缺列会抛错，未做完整输入校验。数值使用 `Decimal.Parse`，排错时留意进程 culture。
- `Time` 和 `EndTime` 都设为交易日上海 15:00 转换到订阅交易所时区后的时刻；不能按普通日线减一天，也不要对 DailyPlus 再做日内 `+4`。
- `FutureAlgorithm.DataTimeZone` 暴露 `Pacific/Wake`，并不自动生成或重写数据／market-hours 配置。标准国内日内存量数据的时间偏移、夜盘归属、周末行为须一起检查实际数据根目录的 market-hours、symbol-properties、map/factor 文件。对采用“交易日归档 +4 / Wake”方案的数据，周五夜盘可能映射为算法上海周日晚间；不要当成真实 UTC 时间或仅减四小时恢复自然日期。
- 连续合约可参考 `ContinuousChnFutureTest` 的 `AddFuture("chnag", ..., dataNormalizationMode=Raw)`；交易、结算和 DailyPlus 应绑定 `Future.Mapped` 的具体合约。映射变化不自动完成仓位迁移，也不自动迁移自定义订阅／指标。首次映射不保证触发 `OnSymbolChangedEvents`，同时处理 `OnSecuritiesChanged` 的初始合约。`Turtle/V13/Turtle.fs` 展示了 Raw 模式下按具体合约建指标、换月后清理旧订阅并重新预热的做法；是否平旧仓按策略决定。

## 费用、保证金与结算语义

手续费按 `abs(order.GetValue(security) / conversionRate) × feeRatio` 返回报价币种；非零费率时要求汇率大于零。不要再次把已经转换到报价币种的费用贴成账户币种。

初始保证金为 `quantity × 当前价 × 合约乘数 × marginRatio × 报价到帐户汇率`，API 中带 quantity 方向符号。订单保证金再加同方向符号的账户币种手续费。维持保证金使用绝对持仓量，价格优先从同 underlying 的 DailyPlus `SettlementPrice > -1` 取，否则使用 `FutureCache.SettlementPrice`；不能把日志中的 “fallback to close price” 当作独立的 Close 读取逻辑。

`FutureSettlementModel.Scan` 在证券本地日期首次推进时处理，首次调用只建立日期前沿；同日本地日期重复 Scan 不重复结算。之后有持仓时优先从 `ActiveSecurities` 中匹配 DailyPlus，否则用 FutureCache。它不是在硬编码的上海 15:00 直接执行；触发还依赖引擎 Scan 和证券时区。

结算以报价币种计算：总浮盈减去 `GetSettledProfitAmount().Amount` 得到当天新增结算额，通过 `AddSettledProfit` 记录，并加到相应币种 `CashBook`。平仓时按已结算持仓被关闭的比例扣除已付结算盈亏，避免再付一次。`FutureHolding.SettledProfit` 是账户币种视图；不得直接赋值或将账户币种金额混入报价币种结算。维护模型时覆盖部分平仓、加仓再减仓、反向持仓和汇率变化。

## 定位与验证

| 源码相对 `$TradeLab` 的位置 | 何时查看 |
| --- | --- |
| `Lean/China/China.fsproj`、六个同目录 `.fs` | 当前 API、构造默认值、实际模型实现 |
| `Lean/Algorithm/QCAlgorithm.cs`、`Lean/Engine/Setup/BacktestingSetupHandler.cs` | 参数字典、实例构造后注入 job 参数的时机 |
| `Lean/Tests/China/` | DailyPlus 字段／时间／路径、算法初始化、费用、保证金、滑点、结算与 market 编码的离线断言 |
| `Backtests/Strategies/Demo/FutureDailyPlusPlayer.fs`、`MixMinuteDailyPlus.fs` | 日线／分钟线与 DailyPlus 并存的读取 |
| `Backtests/Strategies/Demo/FutureDailySettlementTest.fs` | 具体合约与 DailyPlus 的盯市结算示例 |
| `Backtests/Strategies/Demo/ContinuousChnFutureTest.fs`、`Backtests/Strategies/Turtle/V13/Turtle.fs` | 连续映射、具体合约、换月订阅与指标生命周期 |

修改实现时可按项目现有环境运行 `dotnet test Lean/Tests/QuantConnect.Tests.csproj --filter 'FullyQualifiedName~QuantConnect.Tests.China'`。测试工程的 Python.NET 初始化可能另需匹配的 Python 运行库；先检查实际依赖，勿更改全局 Python 只为运行测试。这里列的是验证入口，不代表本次已运行引擎测试或真实历史回测。
