# Flow control and troubleshooting

## Managed flow control

`CtpFlowControlOptions` governs dispatch rate, query rate, retry budgets and delays, query-completion timeout, and market-data subscription batching. Start from `CtpFlowControlOptions.Default`; override only with evidence from the target front and application workload.

The baseline behavior includes these invariants:

- Trader queries are serialized because CTP permits only one in-flight query for a client.
- General dispatches and queries have separate rate gates.
- Only documented retryable native return codes and query response errors are retried, and only within the configured budget.
- Market-data subscriptions are split into batches with a delay between them.
- A query-completion timeout clears its managed pending entry, but the application must still treat the remote outcome cautiously.

Do not add a second generic retry or rate-limiting layer without checking how it interacts with the library. Never automatically retry non-idempotent commands after an ambiguous result.

## Cancellation is local

A `CancellationToken` can cancel waiting for a dispatch gate or managed result. Once a query reaches the native SDK/server, client-side cancellation cannot cancel that server-side operation. The client retains the single in-flight query slot until the native operation completes or times out, so a subsequent query can remain blocked.

Consequences:

- Cancellation improves caller responsiveness but does not guarantee reduced CTP load.
- Do not dispose and recreate clients merely to free a cancelled query slot.
- Size `QueryCompletionTimeout` for the real front and monitor timeout frequency.
- Reconcile state before retrying any operation whose outcome is uncertain.

## Error model

F# consumers receive `Result<_, RspInfo>` for ordinary operations and `ConnectError` for connection establishment. C# wrappers map these to typed exceptions:

- `CtpConnectionException`: native connection initiation failed.
- `CtpTimeoutException`: waiting for `ConnectAsync` timed out in the baseline wrapper.
- `CtpResponseException`: CTP returned a response error; it also represents the baseline synthetic query-completion timeout with error ID `-10001`.
- `OperationCanceledException`: caller cancellation.

Also monitor error events. A method result alone does not cover asynchronous order, cancel, or notification failures.

## Encoding

Managed strings remain Unicode. The baseline defaults are asymmetric by design:

- outbound requests: GBK;
- inbound responses and events: GB18030.

If Chinese text is corrupted, first capture the raw direction and field involved, confirm the front's actual encoding, and inspect any custom `CtpEncodingOptions`. Do not switch both directions to UTF-8 as a generic fix.

## Native library loading

The managed layer does not resolve the bridge from the process working directory. The baseline lookup order is:

1. `CTP_BRIDGE_DIR`;
2. `AppContext.BaseDirectory`;
3. `AppContext.BaseDirectory/native`.

For `DllNotFoundException`, `BadImageFormatException`, or loader errors:

1. Confirm the application's runtime identifier, OS, architecture, target framework, and `Ctp.Net.Next` package version.
2. Inspect the publish/build output for the bridge and vendor runtime libraries.
3. Inspect transitive dependencies with the platform loader tools (`ldd` on Linux or an appropriate dependency viewer on Windows).
4. Use `CTP_BRIDGE_DIR` only as an explicit deployment override, not as a substitute for correct packaging.
5. Confirm that the bridge and vendor SDK versions match; do not mix binaries from unrelated package releases.

If building from a source checkout, follow its `NativeBridge/README` and use its supported build scripts. A managed build can invoke the native bridge build automatically, and checkout-specific build flags take precedence over this snapshot.

## Verification ladder

Use the least risky level that answers the question:

1. Compile the consumer and inspect its output assets.
2. Run repository offline unit tests.
3. Run a mock or isolated application test with synthetic events where available.
4. With explicit authorization, connect to a simulation front and verify login/query/subscription behavior.
5. Run order-related or production checks only with separately confirmed parameters and rollback/containment procedures.

Report exactly which level ran. A skipped live test remains unverified, even when compilation and unit tests pass.
