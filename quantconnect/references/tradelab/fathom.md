# Fathom：TradeLab 本地回测与优化

适用于本地 `Backtests`／`Fathom`、`--optimisation`、TUI 和子进程排错。源码根目录下文记为 `$TradeLab`；本机为 `~/repos/eXp.nXt/TradeLab`。基线为 `94c61738d86d102ac3ca7f1f6398ba9272a2f56c`，核对日期为 2026-10-25。涉及国内期货模型时同时读 [china.md](china.md)。

## 用途与真实入口

`Fathom/Fathom.fsproj` 是 `net10.0` F# 可执行项目，同时暴露启动与优化模块供 `Backtests` 引用。它封装本地 LEAN Engine、handlers、日志、参数网格搜索和优化终端界面；算法放在 `Backtests`，China 模型放在 `Lean/China`。这套流程不需要 QuantConnect Cloud 项目或 MCP。

| 入口／组件 | 当前行为 |
| --- | --- |
| `Fathom/Program.fs` | 始终调用 `Backtester.launch`；独立 Fathom 主程序没有优化分流 |
| `Backtests/Program.fs` | 参数包含英式拼写 `--optimisation` 时调用 `Optimiser.launch`，否则调用 `Backtester.launch`；过滤 `--optimisation` 和 `--backtest` 后交给启动模块 |
| `Backtester.launch` | 解析 LEAN 配置，初始化 handlers，取本地 job，以非实盘模式运行 Engine |
| `Optimiser.launch` | 从配置创建 `OptimizationNodePacket`，运行 `FathomOptimiser`；支持 `--estimate` |
| `FathomOptimiser` | 继承 `LeanOptimizer`，对每组参数启动一个 Fathom 子进程，按 backtest ID 收集 JSON 结果 |
| `Tui.buildTuiApp` | 展示优化进度、完成／失败／运行数量、最佳结果、指标、参数和约束 |

因此，**从 `Backtests --optimisation` 启动优化**；不要假设 `Fathom --optimisation` 也能进入优化。`.o.json` 只是工程中采用的优化配置命名习惯，文件后缀本身不选择运行模式。两处主程序都忽略模块返回值，不能只看 shell 退出码判定算法成功。

## 构建与一次回测

当前项目目标为 .NET 10；先检查实际 checkout 的 `.fsproj`、项目 SDK 和已有产物。F# 策略也设 `algorithm-language: CSharp`，因为 LEAN 加载的是编译后的 .NET 类型。

以下命令从 `Backtests/` 工作目录运行。先把 `data_root` 换成真实 **LEAN 格式**数据根目录；其中应具备策略所需行情及 market-hours、symbol-properties 等元数据。示例不会将 eXp 原生文件自动转换成 LEAN 格式。

```bash
cd ~/repos/eXp.nXt/TradeLab/Backtests
dotnet build Backtests.fsproj -c Release
data_root=/absolute/path/to/lean-data
./bin/Release/net10.0/Backtests \
  --config Strategies/Demo/FutureDailyPlusPlayer.json \
  --algorithm-location "$PWD/bin/Release/net10.0/Backtests.dll" \
  --data-folder "$data_root" \
  --results-destination-folder "$PWD/run-results/dailyplus-probe" \
  --close-automatically true \
  --ui none
```

`FutureDailyPlusPlayer` 是读数 demo，零订单正常。更换策略时同步检查 `algorithm-type-name`；完整类型名通常更明确。已有 demo 配置中的 `bin/Debug/net10.0/Backtests.dll` 需要与实际构建配置匹配，上面的 CLI 显式覆盖为 Release 产物。

Fathom 会作为项目依赖构建并复制到 Backtests 输出目录；优化前检查同目录有可执行的 `Fathom` apphost、`Fathom.dll`、依赖和策略程序集。仅有 `dotnet Fathom.dll` 可启动不足以保证优化成功，因为优化器直接启动 `Fathom`／Windows 的 `Fathom.exe`。

路径以进程当前工作目录解析，不以 `--config` 所在目录解析。使用绝对 `algorithm-location`、`data-folder` 和独立结果目录可减少歧义。优化器拼接子进程命令行时没有为所有路径参数加引号；路径含空格时先核对生成参数，不能认为外层 shell 加引号就解决了内层拼接。

## 配置、日志与调试

- `--config` 指定 LEAN JSON，支持项可由 `LeanArgumentParser` 的 CLI 参数覆盖。不是所有 JSON 键都有同名 CLI 选项；例如并发数放入 JSON。`environment` 选择 `environments` 中的配置层，排错应查看最终有效值。配置文件不存在时 `Config.SetConfigurationFile` 会记录错误并保留此前文件名，运行前主动检查文件存在。
- `parameters` 在普通回测配置中是名称到值的对象；优化配置中是参数定义数组。不能把 `.o.json` 参数数组当成普通回测参数对象。
- 单次回测默认设置 `close-automatically=true`，CLI 可明确覆盖。`--verbose` 映射引擎 `debug-mode`，增加 LEAN DEBUG 日志，不等于附加调试器。`debugging` 控制等待／启用调试器；`--environment debug` 只有在配置提供该层时才具有预期效果。
- 日志由 `log-handler` 控制，初始化器在结果目录设置 `log.txt`。`--ui none` 强制使用普通 Messaging；单次回测的 `--ui tui`／`gui` 仅切换到 StreamingMessageHandler，默认设置 `desktop-http-port=55555`，**没有在此启动 TUI／GUI**。
- 优化时只有 `--ui tui` 实际创建 Terminal.Gui 应用，并通过 mailbox 接收优化进度；`--ui none` 在控制台等待结束。当前没有已实现的图形界面启动路径，不要把选项帮助中的 `gui` 当成现成功能。

## 参数优化

先准备独立的优化 JSON，固定参数使用 `value`，网格参数使用 `min/max/step`。下面使用现有 Turtle V13 在 `Initialize` 中读取的 `EnterPeriod` 展示网格配置；合约和区间仍须匹配数据。China 基类构造时读取的费率／保证金／滑点参数有注入时机限制，见 [china.md](china.md)，不能仅改变其配置值就声称完成有效敏感性分析。

```json
{
  "algorithm-language": "CSharp",
  "algorithm-type-name": "eXp.Lab.Turtle.V13.Turtle",
  "results-destination-folder": "/tmp/turtle-v13-optimisation",
  "maximum-concurrent-backtests": 2,
  "parameters": [
    { "name": "EnterPeriod", "min": 20, "max": 40, "step": 10 },
    { "name": "StartDate", "value": "2020-12-01" },
    { "name": "EndDate", "value": "2021-12-15" },
    { "name": "PortfolioId", "value": 1 },
    { "name": "Portfolio-1", "value": "CHNAG.SHFE" },
    { "name": "GenesisCash", "value": 450000 }
  ],
  "optimization-criterion": {
    "target": "totalPerformance.portfolioStatistics.compoundingAnnualReturn",
    "extremum": "max"
  },
  "constraints": []
}
```

将配置保存为自己的 `strategy.o.json` 后，仍从 `Backtests/` 目录调用；用 `--estimate` 先查看组合规模：

```bash
./bin/Release/net10.0/Backtests \
  --optimisation --estimate --ui none \
  --config /absolute/path/to/strategy.o.json \
  --algorithm-location "$PWD/bin/Release/net10.0/Backtests.dll" \
  --data-folder "$data_root"
```

确认任务范围和输出位置后，去掉 `--estimate`，可用 `--ui tui` 监视实际优化。估算分支不启动回测，但仍可能创建结果目录与 `log.txt`。默认搜索策略为 `GridSearchOptimizationStrategy`，默认目标为最大 CAGR，默认并发为 `max(1, CPU逻辑核数/2)`；默认值不表示适合任意机器或研究目标。其他支持项包括 `optimization-strategy-settings`、`constraints`、`out-of-sample-days`、`out-of-sample-max-end-date` 和 `optimization-id`；其语义继续按当前 Lean Optimizer 源码核对。

每个子进程使用与父进程相同的工作目录，显式传递语言、程序集、算法类型、数据根目录、optimization ID、具体参数、backtest 名称和 ID。输出到 `<results-root>/<backtest-id>/`，完成后读取其中的 `<backtest-id>.json`。子进程强制 `--ui none`、自动关闭和文件日志。

**父配置没有完整传递。** `RunLean` 不给子进程传 `--config`，也不会转发任意 handler、环境层或自定义 JSON 键；子进程会从工作目录的默认 `config.json` 和显式 CLI 构成配置。自定义数据提供器、object-store-root 等不能仅因写在父 `.o.json` 中就认定已在子进程生效。策略需要的固定算法参数应放入优化数组的 `value` 项。遇到父进程正常而子回测异常时，首先检查此配置边界。

## SharedMemoryDataProvider 的实现与限制

`SharedMemoryDataProvider(optimizationId)` 实现 `IDataProvider`，用带 optimization ID 和文件路径的命名 MemoryMappedFile，在不同参数回测进程之间共享文件内容；以命名 Mutex 协调首次创建，进程内字典缓存句柄。`Fetch` 返回只读 view stream，缺文件／目录返回 null，其他异常继续抛出，并触发 `NewDataRequest`。

当前默认配置使用 `DefaultDataProvider`；启动器仅有结束时识别、统计和释放 SharedMemoryDataProvider 的分支，未见主动构造／注册它的接线。该类型要求一个构造参数，而常规 handler 从配置按类型名创建，所以**仅设置 `data-provider: Fathom.SharedMemoryDataProvider` 不能视为已经启用**。如果任务要求启用，应先设计并验证传入 optimization ID 的实例注册及子进程接线，不要在普通策略任务中擅自切换。

此实现使用命名映射，不应推断已跨平台可用；在 Linux 等目标环境需单独验证 .NET 支持情况。源码意图是减少跨进程重复文件读取，不等于共享已解析 bar、跨进程 ZipDataCacheProvider 或已测得的性能收益。相同 optimization ID 下修改输入文件也有陈旧映射风险，运行期间保持输入快照稳定。

## 排查与源码定位

| 现象 | 优先检查 |
| --- | --- |
| 参数不识别／没有开始优化 | 是否用 `Backtests` 入口、是否精确为 `--optimisation`、JSON 参数对象／数组是否匹配模式 |
| 找不到算法或 DailyPlus | 当前目录、程序集路径／类型、实际 LEAN 数据根目录、合约及日期、有效 market-hours 与 symbol-properties |
| 单次运行无界面 | 只有优化分支调用 `Tui.buildTuiApp`；单次分支只切换 messaging handler |
| 子进程启动失败／结果缺失 | Fathom apphost、依赖、路径引号、子配置、各 ID 目录日志与 JSON，而非父进程退出码 |
| 优化显示失败或界面无更新 | `Fathom/Optimiser.fs` 的进程退出结果读取、`Launcher.fs` 的异常日志、`Tui.fs` mailbox；不要用界面存在证明回测成功 |

主要证据：`Fathom/{Program,Launcher,Optimiser,Tui,SharedMemoryDataProvider,Extensions}.fs`、`Backtests/{Program.fs,Backtests.fsproj}`、`Lean/Configuration/{Config,LeanArgumentParser}.cs`、`Lean/Engine/{Initializer,LeanEngineAlgorithmHandlers}.cs`、`Lean/Common/Optimizer/Parameters/OptimizationParameterJsonConverter.cs`。`Fathom/Extensions.fs` 的 `ToContractIdString` 仅格式化 CHN 合约为 `ag2512.SHFE`／`ag.SHFE` 等可读字符串（CZCE、CFFEX 根名大写），不创建订阅，也不替代原始 Symbol ID。

本指南的命令是使用入口；执行前检查数据与输出位置。本次技能更新不代表已经运行完整回测、优化或验证共享内存提供器。
