# Getting started

## Select the source of truth

1. Inspect the consumer project's target framework and `Ctp.Net.Next` package version.
2. If the matching source checkout is available, prefer its `README`, `Demos/`, `Ctp.Net/Common.fs`, `Ctp.Net/Md.fs`, `Ctp.Net/Trader.fs`, and `Ctp.Net/CSharp/` over this snapshot.
3. Do not assume every F# `TraderClient` method has a C# wrapper. Inspect `Ctp.Net.CSharp.TraderClient` or the installed assembly before proposing an API.

The baseline inspected for this skill targets `net10.0` and publishes package `Ctp.Net.Next`. Add the package using the dependency-management convention already used by the consumer project. Do not silently upgrade the target framework or package version.

## Shared configuration

Create one `CtpOptions` per API instance. Required values are the front address, broker ID, user ID, and password. Optional values include `flowPath`, `productionMode`, `userProductInfo`, `appId`, and `authCode`.

- Use an MD front for `MdClient` and a Trader front for `TraderClient`.
- `productionMode` defaults to `true`; set it explicitly from trusted environment configuration.
- Authentication is optional in the API but required by many fronts. Supply `userProductInfo`, `appId`, and `authCode` only through protected configuration.
- If `flowPath` is set, create it before `Connect`/`ConnectAsync`. Give every API instance its own directory and ensure it is writable.

## C# market-data skeleton

The C# wrapper converts failures to exceptions. Subscribe to events before connecting and keep handlers lightweight.

```csharp
using Ctp.Net;
using Ctp.Net.CSharp;

var flowPath = Path.Combine(Path.GetTempPath(), "my-app", "ctp-md");
Directory.CreateDirectory(flowPath);

var options = CtpOptions.Create(
    frontAddress: configuration.MdFront,
    brokerId: configuration.BrokerId,
    userId: configuration.UserId,
    password: configuration.Password,
    flowPath: flowPath,
    productionMode: configuration.ProductionMode,
    userProductInfo: configuration.UserProductInfo,
    appId: configuration.AppId,
    authCode: configuration.AuthCode);

using var md = new Ctp.Net.CSharp.MdClient(options);
md.DepthMarketDataReceived += (_, data) => marketDataQueue.Enqueue(data);
md.RspError += (_, error) => logger.LogError(
    "CTP error {ErrorId}: {Message}", error.ErrorId, error.ErrorMessage);

using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(15));
await md.ConnectAsync(cancellationToken: timeout.Token);
await md.LoginAsync(timeout.Token);
await md.SubscribeMarketDataAsync(instrumentIds, timeout.Token);
```

The application must keep the client alive while consuming events. `Join()` blocks on the native API thread; use it only when that matches the application's shutdown model.

## C# trader lifecycle

Wire `OrderReceived`, `TradeReceived`, `RspError`, `AsyncErrorReceived`, and disconnect handling before sending requests. Then use this sequence:

1. `ConnectAsync` to the Trader front.
2. `AuthenticateAsync` when the front requires application authentication.
3. `LoginAsync`.
4. `SettlementInfoConfirmAsync` before trading when required by the front or application policy.
5. Run queries or submit commands only after the session prerequisites succeed.

The baseline C# wrapper includes core account, position, order, trade, investor, instrument, insert-order, and cancel-order methods. Verify the installed version before using other F# methods from C#.

Catch errors by meaning rather than with a blanket retry:

```csharp
try
{
    var accounts = await trader.QueryTradingAccountAsync(cancellationToken: cancellationToken);
}
catch (CtpResponseException ex) when (ex.ErrorId == -10001)
{
    logger.LogWarning(ex, "CTP query timed out");
}
catch (CtpResponseException ex)
{
    logger.LogError(ex, "CTP rejected the request with {ErrorId}", ex.ErrorId);
}
catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
{
    throw;
}
```

Do not automatically retry commands such as order insertion or transfer after an ambiguous timeout. Reconcile through query and event state first.

## F# market-data skeleton

The F# API returns explicit results. Do not discard `Error` cases.

```fsharp
open System
open Ctp.Net

let flowPath = IO.Path.Combine(IO.Path.GetTempPath(), "my-app", "ctp-md-fsharp")
IO.Directory.CreateDirectory(flowPath) |> ignore

let options =
    CtpOptions.Create(
        configuration.MdFront,
        configuration.BrokerId,
        configuration.UserId,
        configuration.Password,
        flowPath = flowPath,
        productionMode = configuration.ProductionMode
    )

async {
    use md = new MdClient(options)
    md.DepthMarketDataReceived.Add marketDataQueue.Enqueue

    match! md.Connect(timeout = TimeSpan.FromSeconds 15.0) with
    | Error connectError -> return failwithf "CTP connect failed: %A" connectError
    | Ok () ->
        match! md.LoginAsync() with
        | Error info -> return failwithf "CTP login failed: %d %s" info.ErrorId info.ErrorMessage
        | Ok _ ->
            match! md.SubscribeMarketDataAsync instrumentIds with
            | Error info -> return failwithf "CTP subscription failed: %d %s" info.ErrorId info.ErrorMessage
            | Ok subscribed -> return subscribed
}
```

Use the equivalent Trader sequence: `Connect`, optional `AuthenticateAsync`, `LoginAsync`, `SettlementInfoConfirmAsync`, then queries or commands. Match every `Result`; do not reduce distinct response, timeout, and cancellation outcomes to a generic success flag.
