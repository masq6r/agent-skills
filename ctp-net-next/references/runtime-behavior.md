# Runtime behavior

## Client lifetime and connection

- Keep each `MdClient` or `TraderClient` alive for the full event-consumption period and dispose it during orderly shutdown.
- Concurrent callers share the initial connection start. A caller timing out does not restart or tear down the underlying native connection attempt.
- After a disconnect, call `Connect`/`ConnectAsync` again to await the next successful front connection. Do not create replacement clients in an unbounded reconnect loop.
- Subscribe to disconnect, response-error, order, trade, and market-data events before initiating the session so early callbacks are not lost.
- Do not block a native callback or event handler with database, network, strategy, or UI work. Copy the needed data and enqueue it.

## Market-data recovery

The baseline `MdClient` remembers successfully subscribed instruments. With `autoResubscribe` enabled (the default), a successful later `Connect` after reconnection starts login and resubscription in the background.

- Retain and log `FrontDisconnected`, `RspError`, and subscription failures; automatic recovery is not proof of recovery success.
- Avoid racing manual login/subscription with automatic resubscription. If the application owns the full recovery workflow, construct the client with auto-resubscribe disabled where the installed API supports it.
- An unsubscribe removes only instruments that the server confirms from the remembered subscription set.

Trader recovery is application-specific. Re-establish the session and reconcile orders, trades, positions, settlement state, and private-topic sequence state before resuming commands. Never assume a reconnect means an interrupted order failed.

## Trader events and command completion

- Request/response completion and asynchronous notifications are separate channels. Observe `OrderReceived`, `TradeReceived`, `RspError`, and `AsyncErrorReceived` even when an insert or cancel call returned normally.
- A returned request ID or accepted native call is not exchange acceptance or fill confirmation. Drive order state from correlated responses plus order/trade notifications.
- The baseline logout operation intentionally completes after the native logout request is accepted because the SDK does not reliably produce the expected logout callback.
- Topic resume settings and private sequence numbers affect recovery semantics. Preserve them when the application relies on replay; verify their exact meaning against the matching CTP SDK.

## Configuration and secrets

- Keep MD and Trader fronts distinct and validate their schemes and environments before use.
- Resolve `productionMode` from explicit environment configuration. Its baseline default is `true`, so omission is not a safe simulation default.
- Never use the same `flowPath` for two live API instances. Prefer stable per-account, per-client paths in production so native flow files survive process restarts as intended.
- Do not log passwords, authentication codes, complete account identifiers, or the contents of ignored local settings files.
- A source checkout may contain ignored smoke-test configuration. Its presence does not authorize reading or using it.

## Version-aware code generation

Before emitting a method call, inspect the installed public surface when practical. The F# Trader API is broader than the baseline C# wrapper, overloads and optional arguments can change, and SDK-generated request/response fields depend on the packaged CTP version.

When no matching source is available, state which bundled baseline informed the answer and identify any call whose current signature remains unverified.
