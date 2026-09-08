---
name: ctp-net-next
description: Integrate and operate the Ctp.Net.Next NuGet library from C# or F#, including market data, trader sessions, flow control, cancellation, reconnect handling, and runtime troubleshooting. Use for consumer applications; do not use for modifying the native bridge or managed wrapper implementation.
---

# Ctp.Net.Next

Help consumers use `Ctp.Net.Next` safely and according to the API version in their project.

## Establish the API version

When a `Ctp.Net.Next` checkout or installed package is available, inspect its version, README, demos, and public types before writing code. Treat the bundled references as a portable baseline derived from repository commit `487309c` and package version `0.5.0`, not as proof of the current latest API.

Choose the public surface from the consumer language:

- For C#, prefer `Ctp.Net.CSharp.MdClient` and `Ctp.Net.CSharp.TraderClient`. They expose `Task`, `CancellationToken`, `EventHandler<T>`, `IReadOnlyList<T>`, and typed exceptions.
- For F#, prefer `Ctp.Net.MdClient` and `Ctp.Net.TraderClient`. They expose `Async<Result<_, RspInfo>>`, F# options, lists, and events.
- Do not use types under `Ctp.Net.Bridge` unless a public request or response type requires that namespace.

Read [references/getting-started.md](references/getting-started.md) when creating or reviewing integration code. Read [references/runtime-behavior.md](references/runtime-behavior.md) for lifecycle, reconnect, events, and production configuration. Read [references/flow-control-and-troubleshooting.md](references/flow-control-and-troubleshooting.md) for concurrency, cancellation, errors, encoding, or native-library failures.

## Work safely

- Never print, copy into generated code, or commit credentials, front addresses, `AppId`, or `AuthCode` discovered in local configuration. Use placeholders, environment variables, secret stores, or an ignored local settings file.
- Treat connecting to any CTP front as an external side effect. Obtain the user's authorization before connecting, authenticating, subscribing, running live smoke tests, or sending requests against a real or simulated front.
- Treat order insertion, cancellation, bank-futures transfer, and any production-mode action as sensitive. Confirm the exact account, environment, instrument, direction, offset, price, and volume immediately before execution.
- Prefer compile checks and offline unit tests. Do not present a successful build as evidence that login, settlement confirmation, subscriptions, or orders work against a particular front.
- Keep native callbacks and .NET event handlers lightweight. Hand expensive work to a queue or background processor.

## Use the upstream CTP material only when needed

This skill covers the managed consumer API. If field meanings, exchange rules, callback ordering, order states, positions, options, combination contracts, or native error semantics are material, consult the separate `ctp` skill or the exact SDK reference bundled with the matching `Ctp.Net.Next` checkout. Do not infer native semantics from .NET type names alone.
