# CTP SDK Pitfalls & Behavioral Notes

Synthesized from: 开心秋水's "CTP开发爬坑指北" (Bilibili), CTP official API reference (v6.7.13), FAQ, version changelogs, and error code references.

---

## 1. TradingDay & Date Handling

- **TradingDay vs ActionDay in MD snapshots**: During day session they are equal. During night session, ActionDay is the actual calendar date, while TradingDay belongs to the next day's day session.
- **CZCE TradingDay is a natural day**, not a trading day. All other exchanges use the trading day.
- **Only SHFE/INE dates are accurate** in night-session snapshots. Other exchanges have varying rules — do not rely on their ActionDay/TradingDay from MD.
- **CZCE EnterTime** is CTP's internal queueing time, NOT the exchange time. CZCE contract status changes are obtained via market data query, not broadcast push — so there is no true exchange timestamp.
- **Workaround**: Use local system date as ActionDay, and `current_time + 4 hours` as TradingDay (with special handling for Friday→Monday).
- **Reliable sources for TradingDay**: (a) Login response from MD or Trader API; (b) `GetTradingDay()` static function — but **only valid after a successful login**; (c) `TradingDay` field in `OnRtnTrade` and `OnRtnOrder` callbacks — these are always accurate.
- **TradeDate in OnRtnTrade**: Like ActionDay, rules vary by exchange. Apply the same workaround.
- **InsertDate inconsistency in OnRtnOrder**: In unknown state, it's the calendar date (from exchange reports for ZCE). In CTP-source reports, it's the trading day. To determine exact time, use `TradingDay + InsertTime`.

## 2. Order Uniqueness & Cancellation

Three methods to identify/cancel an order:

| Method | Key Fields | Stability | Notes |
|--------|-----------|-----------|-------|
| **1 (recommended)** | `ExchangeID` + `OrderSysID` | Stable across callbacks | Exchange-assigned. Future CTP v7.0 will require ExchangeID on every order. |
| **2** | `FrontID` + `SessionID` + `OrderRef` | **UNSTABLE** — CTP may reassign after rejection | Also needs InstrumentID. Some exchanges reject this before the order reaches the exchange. Not in OnRtnTrade. |
| **3** | `ExchangeID` + `TraderID` + `OrderLocalID` | Stable across callbacks | CTP-assigned. Note: TraderID ≠ TradeID (has an extra 'r'). Not available if CTP rejects the order. |

- **Critical**: Method 2 values can CHANGE between callbacks — CTP reassigns some fields after rejecting an order. Never use FrontID+SessionID+OrderRef as a long-lived order key without considering this.
- **OrderSysID** is a right-aligned numeric string (e.g. `"   012865"`). Do NOT trim whitespace — store it verbatim. Different exchanges have different padding rules.
- **OrderRef** is a signed 32-bit int internally (some brokers use 64-bit, but assume 32-bit for safety). Range: [-2147483468, 2147483467]. Values outside this range may be treated as 0 by the server and cause rejections. Must increment monotonically per session, or you get "duplicate order" rejection.
- **OrderSysID is empty** in the first order response. Cannot cancel until you receive it.
- Many Trader API's functions require a parameter named `nRequestID`. The SPI's callback functions with `nRequestID` will have the same value so that developers can link the request and response. This rule doesn't apply to MD API's (callback functions always give `nRequestID = 0`).
- **RequestID**: Can correlate `OnRtnOrder` back to the order request (RequestID is echoed in the callback). CTP does NOT validate or clear RequestID — duplicates are allowed. RequestID alone can't distinguish orders across different sessions of the same account. Not present in OnRtnTrade. Note that `RequestID` has nothing to do with `nRequestID`.

### Linking OnRtnTrade to the original order

`OnRtnTrade` contains `OrderSysID`, `OrderRef`, `TraderID`, `OrderLocalID` but NOT `FrontID`/`SessionID`. Use method 1 or 3 above to link trades back to orders.

- `Volume` in `OnRtnTrade` is the volume of *this specific trade*, not the total filled volume. Total filled volume is `VolumeTraded` in `OnRtnOrder`.
- Combo contract trades generate at least 2 `OnRtnTrade` callbacks (one per leg).

### Quote SysID conventions by exchange

- **DCE**: `AskOrderSysID` and `BidOrderSysID` are EMPTY in quote responses.
- **SHFE**: `QuoteSysID` = sell OrderSysID, `QuoteSysID + 1` = buy OrderSysID.
- **CZCE**: `QuoteSysID` prefix `'a'` = sell OrderSysID, prefix `'b'` = buy OrderSysID.
- **OptionSelfCloseSysID**: CTP clears (zeros) this value — cannot use it as a composite key.

### Conditional order tracking (RelativeOrderSysID)

- Untriggered conditional orders get `OrderSysID = "TJBD_XX"`, status = "not triggered".
- When triggered, a new order is created with `RelativeOrderSysID = "TJBD_XX"`.
- **CTP does not validate that RelativeOrderSysID is non-empty**. If blank, the conditional order chain cannot be linked.

## 3. Order Flow: Insert → Reject → Notification → Trade

### Return value ≠ success

- `ReqOrderInsert` return value 0 only means `socket.write` succeeded. It does NOT guarantee the backend received the command or that any callback will fire.
- Return `-2`: Unprocessed requests exceeded the permitted count (order frequency limit).
- Return `-3`: Per-second send rate exceeded.

### Order rejection: Why didn't I receive OnErrRtnOrderInsert?

- `OnRspOrderInsert` (with nRequestID) goes only to the session that placed the order.
- `OnErrRtnOrderInsert` goes to ALL sessions of the same account — **but only if the `UserID` field was filled in the original order request**.
- CTP does NOT validate UserID when accepting the order, but uses it when broadcasting rejection errors.
- **Rule**: Always fill `UserID` in order requests.
- **Rejected orders may arrive via OnRtnErr (not OnRtnOrder) or OnRspOrderInsert.** Handle BOTH paths for complete order state management.

### When to update positions/funds during order lifecycle

1. **On order insert** → freeze margin (for open) or freeze position (for close). Do this immediately, not when `OnRtnOrder` arrives — otherwise the gap creates stale available-position data and could cause double-ordering.
2. **On `OnRspOrderInsert` (error)** → unfreeze what was frozen in step 1. Use order quantity.
3. **On `OnRtnOrder` with cancelled status** → unfreeze. Use remaining quantity (`VolumeTotal`), not original order quantity.
4. **On `OnRtnTrade`** → decrease frozen quantities by trade volume; update position details, margins, commissions.

### Multi-session account concern

If the same account is logged in from multiple terminals, `OnRtnOrder` and `OnRtnTrade` are broadcast to all sessions. You may receive callbacks for orders you didn't place. Two solutions:
1. Cross-terminal sync mechanism (e.g. distributed service).
2. On receiving `OnRtnOrder`, check `FrontID`/`SessionID`. If it's not from this session and it's the first time seeing this order, replay the freeze logic as if you had placed it.

### Order field notes

- **IsAutoSuspend**: Must be `0`.
- **OrderMemo**: CTP does NOT process this field — whatever the client writes is passed through transparently. Usable as a client-side tag.
- **SessionReqSeq**: Auto-maintained by the API per session. Client does not need to maintain it.
- **BusinessUnit**: CTP internal field — do NOT use.
- **IPAddress**: For IPv6, convert to compressed format and remove colons (e.g. `AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH`).
- **UserProductInfo** in login response comes from `ReqAuthenticate`, NOT from `ReqUserLogin`.
- **LoginRemark**: Format `IP|MAC|UUID|APPID|手机号|...`. Special characters (colon, pipe, `\r`, `\n`) are filtered out by CTP.
- **ClientIPAddress / MacAddress** in login are auto-detected by CTP — manual fill is ignored.
- **ZCETotalTradedVolume**: For ZCE batch fills, this field shows the total traded volume early, before the final individual `OnRtnTrade` arrives.

## 4. Position & Fund Maintenance

### Fund formulas (逐日盯市 basis)

```
StaticEquity = PreBalance + Deposit - Withdraw
DynamicEquity = StaticEquity + PositionProfit + CloseProfit + CashIn - Commission
Available = DynamicEquity - CurrMargin - FrozenMargin - FrozenCommission - FrozenCash - DeliveryMargin
```

### Position P&L on market data

```
Long PositionProfit = (LastPrice - AvgPositionPrice) × Multiplier × Position
Short PositionProfit = (AvgPositionPrice - LastPrice) × Multiplier × Position
```

### On order insert (open)
- Freeze margin = SettlementPrice × Multiplier × MarginRate × Qty
- Increase LongFrozen (for buy open) or ShortFrozen (for sell open)

### On order insert (close)
- Freeze position: increase ShortFrozen (for long position) or LongFrozen (for short position) by order qty
- Available to close = Position - ShortFrozen - CombShortFrozen (for long)

### On trade (open)
- Create position detail: OpenDate = TradingDay, TradeID = trade.TradeID, OpenPrice = trade.Price, HedgeFlag = trade.HedgeFlag
- Margin = Volume × Price × Multiplier × MarginRate
- Decrease LongFrozen/ShortFrozen by trade volume
- Decrease frozen margin by trade-volume proportion
- Recalculate avg position price from position cost

### On trade (close)
- Determine which position details to close (FIFO: 昨仓 first, then 今仓)
- For 昨仓: CloseProfit = (TradePrice - PrevSettlementPrice) × N × Multiplier
- For 今仓: CloseProfit = (TradePrice - OpenPrice) × N × Multiplier
- Commission differs by 平今 vs 平昨 rates, even for exchanges (DCE) that don't distinguish them for position management
- Decrease ShortFrozen/LongFrozen by trade volume
- Update position aggregates from position details

### Position closing order by exchange

| Exchange | Rule |
|----------|------|
| DCE, GFEX, CZCE | FIFO, 平昨 first. But if 平今 has commission waiver, 平今 first. |
| CFFEX (股指期货) | FIFO, 平昨 first. BUT commission calculated assuming 平今 first — two parallel "virtual" position systems. |
| SHFE, INE | Trader specifies 平今 vs 平昨 explicitly. Within each, FIFO. |

### CFFEX commission trick
Example: IF2203, 平昨=20/hand, 平今=300/hand. Position: 2 昨仓 + 1 今仓.
- Close 1: closes a 昨仓 but charged 平今 (300). Now "virtual 今仓" exhausted.
- Close 1 more: closes the other 昨仓, charged 平昨 (20).
- Total: 320. The actual positions closed are both 昨仓, but fees follow 平今-first rule.

### Portfolio margin systems
Each exchange is rolling out its own portfolio margin system (CFFEX RCAMS, SHFE PMM, CZCE SPBM, DCE 汝乐Rule). Accurate margin calculation across all these is extremely difficult. **Recommendation**: Use a simple conservative rule (e.g. charge the larger leg's full margin) instead of trying to match exchange margin exactly.

## 5. Commission Rate & Margin Rate Queries

- If the response returns a **product code** (e.g. "IF") instead of the specific instrument code ("IF2203"), the rate is the product-level default.
- If `InvestorID` returns as `"000000"`, the rate is exchange-default, not account-specific.
- Queried rates are the **account's effective rates**, not exchange-standard rates. Exchange-standard margin rates come from `OnRspQryInstrument`.
- Querying with empty InstrumentID returns rates only for positions held today. To get all market rates, query instrument-by-instrument.
- **LongMarginRatio** = Exchange `LongMarginRatioByMoney` (from `OnRspQryExchangeMarginRate`) + `LongMarginRatioByMoney` (from `OnRspQryExchangeMarginRateAdjust`).

## 6. Market Data Quirks

### Turnover (成交额) can decrease
- CZCE Level1 does not push turnover. CTP synthesizes it: `Turnover = AvgPrice × Volume × Multiplier`.
- Because AvgPrice is rounded to PriceTick, a new snapshot with lower AvgPrice can show *decreased* Turnover.
- CZCE Level2 has accurate exchange-pushed turnover. CTP uses Level1.

### Turnover formula varies by exchange and CTP backend version

| Exchange / Front | Formula |
|------------------|---------|
| CZCE (all fronts) | `AvgPrice × Volume` |
| CFFEX, DCE, SHFE, INE | `AvgPrice × Volume × Multiplier` |
| Regular front before v6.7.1 | `AvgPrice × Volume` |
| Regular front v6.7.1+ | `AvgPrice × Volume × Multiplier` |

Official exchange formula: `AvgPrice × Volume × Multiplier`.

### Tick frequency by exchange
- DCE: 4 ticks/sec
- CZCE: 2 ticks/sec (configurable to 4 via broker ini; most brokers use 2)
- All others (SHFE, CFFEX, INE, GFEX): 2 ticks/sec

### After-close market data (post 15:00)
- CZCE especially continues pushing snapshots after close.
- 期转现 (EFP) trades occur after close.
- TAS (Trading at Settlement) trades also occur after close.
- A final snapshot with **settlement price** arrives before ~16:00 — you can get settlement price early without waiting for settlement.

### CZCE UDP vs TCP mode
- UDP mode: `TradingDay` and `AveragePrice` are EMPTY.
- TCP mode: Both are populated by CTP.

### DBL_MAX sentinel (1.79769e+308)
- Combo contract Market Data: `LastPrice`, `Volume`, `Turnover`, etc. are `DBL_MAX` (invalid). Trading software synthesizes these from the two legs.
- When opening price, close price, settlement price show `DBL_MAX` or `0`, the field has no value (e.g. limit up/down with no trades).
- **Treat any value equal to `DBL_MAX` or negative as invalid.**

### OpenInterest vs PreOpenInterest
- Exchange data is time-shared; final settlement values are only finalized after market close. These fields may be inconsistent intraday.

### Subscription limits
- **Cannot subscribe to the entire market.** Must subscribe to individual contracts.
- Subscribing too many contracts at once (especially futures + options) can trigger flow control, causing the session to disconnect with reason `0x1001` (4097).
- **Workaround**: Batch subscribe ~1000 contracts at a time with 1-second delay between batches.
- **MaxSubInstCnt** (v6.7.2P6+): Per-session subscription limit. Exceeding returns error 6000 "CTP:sub too many insts". Default 0 (unlimited).

### Duplicate ticks during restart
- Before `front_md_se` v6.6.2P8, when CTP restarted (e.g. during night session watering), duplicate ticks could be replayed. After v6.6.2P8, the front-end caches data and no longer sends duplicates.
- **Recommendation**: Handle possible duplicate ticks in your code regardless.

### OnRtnDepthMarketData performance
- Time-consuming business logic in this callback will block the API's internal thread and can cause disconnections. **Must** dispatch heavy processing to another thread.

### bIsLast in market data subscription
- The number of `OnRspSubMarketData` callbacks may not match the actual number of contracts subscribed. `bIsLast=true` may not be reliable. This does NOT affect actual subscribed contracts. Use `isLast` only as a hint.

## 7. CombOffsetFlag & CombHedgeFlag

- `TThostFtdcCombOffsetFlagType` is a char array designed for multi-leg combo orders, where `[0]` = leg 1, `[1]` = leg 2, etc.
- **In practice**, all contract types only use `[0]`. Treat it as a plain `TThostFtdcOffsetFlagType`.
- Same rule applies to `CombHedgeFlag`: only fill `[0]`.
- **Exception (v6.7.13+)**: For broker-source medium orders, `CombOffsetFlag` and `CombHedgeFlag` can contain multi-digit character arrays. Example: `CombOffsetFlag="04"` = open first, then close today; `CombHedgeFlag="13"` = speculation + hedge.

### Exchange-specific hedge flag rules
- **ZCE**: Open orders must set `CombHedgeFlag[0] = THOST_FTDC_BHF_Speculation` (speculation).
- **SHFE**: Close orders can only fill `THOST_FTDC_BHF_Speculation`.

### TimeCondition restrictions
- Only **GFD** (`THOST_FTDC_TC_GFD`) and **IOC** (`THOST_FTDC_TC_IOC`) are supported.
- `THOST_FTDC_TC_GFS` is reserved for future GIS orders — do NOT use.
- **ZCE combo contracts** require `TimeCondition = THOST_FTDC_TC_IOC`. They support FAK/FOK.

### ZCE price rounding
- ZCE: Prices are rounded to the nearest PriceTick (e.g. `1500.9` with tick size 1 → `1500`).
- Other exchanges: Invalid-tick prices return an error directly.

### ForceCloseReason
- Regular traders must set `THOST_FTDC_FCC_NotForceClose`. Clients cannot submit `THOST_FTDC_OF_ForceClose` orders — CTP/exchange will reject them.

## 8. Thread Safety

- **CTP API is thread-safe** for most operations. Multiple threads can call API functions (order insert, query, etc.) concurrently.
- **EXCEPTION — NOT thread-safe**:
  - `Init()` and `Release()` are NOT thread-safe. Must be serialized if called from multiple threads.
  - Data collection APIs (`RegisterUserSystemInfo`, `SubmitUserSystemInfo`, `CTP_GetSystemInfo`) are NOT thread-safe. Must lock if calling from multiple threads.
- API and SPI (callback) run on different threads. This is platform-independent (Linux & Windows).
- **Callbacks in TraderSpi/MdSpi must return quickly.** They execute on the API's internal thread. Blocking them blocks all network communication and can cause disconnections.

### RegisterSpi(NULL) crash
- Do NOT call `RegisterSpi(NULL)` before `Release()`. This is a common mistake that leads to crashes.
- Correct approach: call `RegisterSpi(NULL)` only after ensuring no more callbacks will fire, or simply call `Release()` directly.

## 9. GDB Debugging

- CTP API internally uses `SIGUSR1`. GDB will pause on it by default.
- Fix: `(gdb) handle SIGUSR1 nostop noprint`

## 10. Settlement Confirmation

- Must confirm settlement (结算单) before trading. Workflow:
  1. `ReqQrySettlementInfoConfirm` — check if already confirmed today.
  2. If not confirmed: `ReqQrySettlementInfo` — fetch settlement info (Content may span multiple responses — concatenate them).
  3. `ReqSettlementInfoConfirm` — confirm.
- If already confirmed today, skip steps 2-3.
- **Unconfirmed settlement blocks trading**: If settlement is not confirmed, orders are rejected with error 42 "CTP:结算信息未确认".
- **Settlement info may not exist yet**: CTP settlement is generated by back-office staff after market close. If it hasn't been generated, the query returns nothing — this is not an error.
- **TradingDay format for settlement query**: `yyyymmdd` for a specific day, `yyyymm` for a month.
- Settlement info content format varies by broker but is stable over time. Can be auto-parsed for historical trade extraction.

## 11. InvestUnitID for Strategy Tracking

- `InvestUnitID` (char[17]) is intended for fund account name, but CTP doesn't strictly validate it.
- The value you set on order insert is echoed back in `OnRtnOrder` and `OnRtnTrade`.
- **Unofficial trick**: Use it as a strategy/signal ID to correlate orders/trades to strategies. It can be any string ≤ 17 chars, duplicates are allowed.
- Caveat: depends on CTP's continued lax enforcement. Not officially supported.

## 12. SimNow vs Exchange Simulation

| Aspect | SimNow | Exchange Simulation |
|--------|--------|-------------------|
| Operator | 上期技术 | Individual exchanges |
| Purpose | Public testing | Compliance testing (穿透式监管) |
| Counterparty | Other SimNow users | Other brokers' simulation accounts |
| Restrictions | Fewer | May restrict products, permissions (e.g. no cancel, no settlement query) |
| Self-trading | Allowed | CFFEX blocks self-trading + has price band limits |

- Exchange simulation is required for 穿透式监管 (penetrative supervision) testing before production access.

## 13. Contract & Product Query (v6.5.1+)

- **Before v6.5.1**: Only `ReqQryInstrument` for all contracts. Filter by `ProductClass` in response.
- **v6.5.1+**: `ReqQryClassifiedInstrument` with `ClassType`:
  - `INS_FUTURE` ('1') — futures
  - `INS_OPTION` ('2') — options
  - `INS_COMB` ('3') — combination contracts
- **Both `TradingType` and `ClassType` must have valid enum values** — do NOT set them to 0 for "all".
- **v6.5.1+: `ReqQryInstrument` no longer returns combo contracts.** Combo contracts are only returned when a specific investor logs in.
- Combo contracts returned by `ReqQryClassifiedInstrument` are grouped by product family to reduce data volume (e.g. `STD m_o&m_o` represents all standard spreads of that product).
- Contract names, product names, and PriceTick may vary between brokers or even be inconsistent within the same broker. Do NOT rely on them for trading logic. Maintain your own instrument reference table if accuracy matters.
- **EndDelivDate comes from the exchange** — updated daily. Other fields may be broker-counter values.
- When a query has no records, the response pointer returns **null** (not an empty list).
- `ExchangeID` matching in queries is **case-insensitive** (v6.7.0+).

## 14. Options vs Futures

### Key differences
- Options: `PositionProfit` and `CloseProfit` are **always 0**. P&L is reflected in `CashIn` (权利金收支).
- Buying options: pay premium → `CashIn` decreases. Selling options: receive premium → `CashIn` increases.
- `CashIn` change = `Volume × Price × OptionMultiplier` (regardless of direction, call/put, or strike).
- Only option **sellers** (short positions) pay margin. Buyers (long) only pay premium.
- `FrozenCash` freezes premium for pending buy orders.

### API versions
- **Futures options** (v6.x): 6 futures exchanges, all futures + commodity options + stock index options (IO, MO, HO).
- **Stock options** (v3.x): SSE/SZSE stock ETF options only. No individual stock options yet in China.
- Both versions have nearly identical interfaces.

### Dedicated option APIs
- `ReqQryOptionInstrTradeCost` / `OnRspQryOptionInstrTradeCost` — option trade cost (margin)
- `ReqQryOptionInstrCommRate` / `OnRspQryOptionInstrCommRate` — option commission
- Self-hedge: `ReqOptionSelfCloseInsert`, `ReqOptionSelfCloseAction`, `ReqQryOptionSelfClose`
- Exercise: `ReqExecOrderInsert`, `ReqExecOrderAction`, `ReqQryExecOrder`
- CTP does NOT provide option Greeks (Delta, Gamma, Theta, Vega, Rho).

### Option exercise changes (v6.7.9_P1)
- Old option exercise interface **no longer supports automatic hedge matching**. `CloseFlag` can only be `EOCF_NotToClose('1')`.
- To exercise options with hedging, must use the new `ReqExecOrderInsert` interface.
- During settlement processing and manual terminal exercise auto-hedge time, option self-close is **forbidden** (error 6022).

### Option margin calculation varies by exchange
- DCE/GFEX/CZCE: `max(Premium + FuturesMargin − 0.5×OutOfMoney, Premium + 0.5×FuturesMargin)`
- CFFEX: `StrikePrice × Multiplier + max(Index × Multiplier × 15% − OutOfMoney, 0.667 × Index × Multiplier × 15%)`
- SHFE: Delta-based margin model.
- SSE/SZSE: Premium + percentage of underlying.

### OptionValue field (v6.7.10+)
- Added to `TradingAccount` and `InvestorPosition` structures.
- Formula: `(Long premium − Short premium) × OptionMultiplier × UnderlyingMultiple`.
- Net equity: `OptionValue + Equity`.

## 15. Combination (套利) Contracts

### Basics
- Two-leg contracts: same product + different delivery months (跨期), or different products + same month (跨品种).
- Code includes spaces and `&` symbols (e.g. `SPC a2101&a2105`).
- `ProductClass == THOST_FTDC_PC_Combination` identifies combo contracts.
- Market data: `LastPrice`, `Volume` etc. are `1.79769e+308` (DBL_MAX = invalid). Trading software synthesizes these from the two legs' market data.
- **Combo contract market data is NOT provided by the MD front-end.** Client must calculate spreads from single-leg data.

### Trading
- No market orders (last price is invalid).
- Buy combo = buy leg1 + sell leg2. Sell combo = sell leg1 + buy leg2.
- Both legs trade simultaneously. Two `OnRtnTrade` callbacks per fill (one per leg). TradeType = `THOST_FTDC_TRDT_CombinationDerived`.
- `IsSwapOrder=1` reverses the direction of leg2.
- No commission discount for combo trades.

### Restrictions
- **Combo contracts do NOT support conditional orders** (error 6013), parked orders (error 6011), or parked order cancellation (error 6012).
- **Delete combo action requires 30-second wait** after the original order (error 152).
- **ZCE combo InstrumentID format**: `STD SR703C5300&SR703P5300` (standard spread) or `STG SR703C5300&SR703P5200` (custom spread).
- **ZCE has no standard/open combo contracts** — `ReqQryInstrument` cannot find them.

### Position maintenance for combos
- Opening a combo creates 3 position records: the combo position + two leg positions.
- The combo position record only has meaningful Position/YdPosition/LongFrozen/ShortFrozen/Margin.
- The leg position records have Margin=0 but accumulate all P&L, commission, etc.
- `CombPosition` field in leg positions = how many lots came from combo trades.
- Closing combos follows "single first, combo second; FIFO within each" — and can trigger position splitting from combo→single.
- `ReqQryInvestorPositionCombineDetail` queries combo-derived position details specifically.
- You can close via a combo order even if you have no combo positions — as long as both legs have available single positions.

## 16. Force Close (强平)

- `UserForceClose` flag can be set on orders, but brokers will ask you NOT to use it — it conflicts with their own force-close tracking.
- `TThostFtdcOffsetFlagType::THOST_FTDC_OF_ForceClose` orders cannot be submitted by clients — CTP/exchange will reject them.
- `ForceCloseReason` in `OnRtnOrder` indicates the reason if a force close occurred.

## 17. Combo Positions from Settlement

- DCE and CZCE may automatically pair eligible single positions into combo positions after settlement.
- Rules vary by exchange and product. Some require explicit application, others are automatic.

## 18. Margin Price Types (保证金价格类型)

Brokers use different basis prices for margin calculation: 昨结算价, 最新价, 成交均价, or 开仓价. Query via `ReqQryBrokerTradingParams` / `OnRspQryBrokerTradingParams`.

## 19. Connection & Session Lifecycle

### Connection attempt accounting
- **Each `Init()` call counts as one connection attempt** toward the `ConnectFreq` limit. Init is reentrant — each call is a new attempt.
- `ConnectFreq` default: 20 connections per minute per IP. Exceeding triggers `OnFrontDisconnected`.
- Can be bypassed with a `whiteiplist` file in the front-end's bin directory (one IP per line).

### Session limits
- **MaxIPSession** (v6.7.2P6+): Maximum active sessions per IP. Default 0 (unlimited).
- **Same UserID max concurrent sessions**: Configured by broker's counter system, default ~4. Exceeding triggers login error "CTP:用户超过会话数限制".
- **SendingListSize**: Per-session message queue limit, default 10000. Exceeding causes the front-end to FORCE-DISCONNECT the session (no error, just disconnect). Effective max = max(10000, configured).
- **LoginFreq**: Per-session login rate limit. Default 1 login per 5 seconds. Exceeding triggers "CTP: api login over limit freq".
- **API socket limits**: Linux ~180-200 sockets, Windows ~400. Plan thread count accordingly.

### Multiple API instances
- Multiple `CreateFtdcTraderApi` instances **must use separate flow directories**. Sharing the same directory causes lost order reports.

### Flow directory prerequisites
- Flow directory **must exist before `Init()`**. Otherwise: "RuntimeError: can not open CFlow file".
- **`ulimit -n` too small causes crashes**: If the process file descriptor limit is too low, the API cannot create enough threads. On Linux, this causes a core dump in `CThostUserFlow::OpenFile`.

### Auto-reconnect behavior
- API **auto-reconnects** after disconnection — no manual intervention needed.
- v6.7.9+: Reconnect delay is **fixed at 1 second** (previously immediate retry).
- Reconnect interval cannot be customized or queried via any API.
- **After reconnect, must re-authenticate** (`ReqAuthenticate`) and re-login. Callbacks resume normally after login.
- Excessive reconnect frequency triggers `ConnectFreq` limit → `OnFrontDisconnected`.

### Disconnection reason codes
`nReason` values must be **converted to hex** for interpretation:

| Hex | Decimal | Meaning |
|-----|---------|---------|
| 0x1001 | 4097 | Network read failure (`recv=-1`) |
| 0x1002 | 4098 | Network write failure (`send=-1`) |
| 0x2001 | 8193 | Heartbeat timeout (front sends heartbeat every 53s; API detects disconnect after 120s without data) |
| 0x2002 | 8194 | Heartbeat send failure (API sends heartbeat every 15s; front detects disconnect after 40s without data) |
| 0x2003 | 8195 | Received disconnect command |

### Logout behavior
- `ReqUserLogout` triggers `OnFrontDisconnected`, then auto `OnFrontConnected`.
- **Do NOT wait for `OnRspUserLogout` to finish logout.** The logout callback is not a reliable sync point for shutdown.

### SubscribePrivateTopic / SubscribePublicTopic
- **Must be called BEFORE `Init()`.** Otherwise, no private/public topic data is received.
- `THOST_TERT_NONE` cancels public topic subscription.

### Communication modes
Three modes: Dialog mode, Private topic, Broadcast topic. Each has different message delivery guarantees.
- `THOST_TERT_RESUME_FROM_SEQ_NO` (v6.7.13+): Resume private topic from a sequence number. `SubscribePrivateTopic` accepts `nSeqNo` parameter (default 1). New callback `OnRtnPrivateSeqNo(int nSeqNo)` reports the latest private flow sequence number.

## 20. Flow Control & Rate Limiting

### FTD message flow control (FTDMaxCommFlux)
- Front-end-level comprehensive throttling. All API calls (login, queries, orders, quotes, etc.) are subject to this limit.
- Default: ~6 commands per second.
- **Exceeding this limit does NOT return an error.** Requests are silently queued and sent in the next time window, causing apparent delays.

### Query flow control (QryFreq)
- Per-session query rate limit configured on the front-end. API obtains this during handshake.
- Default: 2 queries per second, with a queue depth of 1.
- A second query before the first completes → API returns `-2`.
- Exceeding front-end limit → `OnRspError` with "CTP:查询未完成请稍后重试" (error 90 `NEED_RETRY`).

### Order rate limiting
- Exchange-level order rate limit. Exceeding → `OnRtnOrder` with "CTP:交易所每秒发送委托次数超过了限制".
- Front-end order flow control: FLEX "Simplified Trade Frequency Control" setting. Exceeding → `ReqOrderInsert` returns `-2`.
- Per-second order rate limit. Exceeding → returns `-3`.

### Key error codes for flow control
| Code | Name | Meaning |
|------|------|---------|
| 90 | NEED_RETRY | Query did not return result; try again later |
| 116 | ORDER_FREQ_LIMIT | Order frequency too high |
| 154 | QK_BUSY | Query engine busy; try again later |
| 162 | NOT_TRADING_PERIOD | Product not in tradable session |
| 163 | PRICE_OVER_LIMIT | Price exceeds daily limit |
| 165 | PRICE_WRONG_TICK | Price does not meet tick size requirement |
| 6000 | OVER_SUB_INST_LIMIT | Subscribed too many instruments |

## 21. Authentication & Penetration Testing (穿透式监管)

### Data collection overview
- Penetration testing is a regulatory requirement. CTP provides collection APIs but does not enforce them — compliance is the broker's responsibility.
- **Direct mode**: TraderApi auto-collects during login. No need to call `CTP_GetSystemInfo`.
- **Relay/proxy mode**: Must call `CTP_GetSystemInfo` and submit via `RegisterUserSystemInfo` (before login) or `SubmitUserSystemInfo` (multiple times after operator login).

### Order of operations
1. Authenticate (`ReqAuthenticate`)
2. Report system info (`RegisterUserSystemInfo` or `SubmitUserSystemInfo`)
3. Login (`ReqUserLogin`)
- After sleep/wake: re-authenticate and re-report info before login.

### AppID requirements
- `AppID` is required for authentication. Wrong AppID → error 4043 "CTP: user client auth failed".
- Auth codes are configured per terminal product, not per account.
- Relay AppID and direct AppID can be different and need separate encryption keys.
- `userproductinfo` vs `AppID`: `userproductinfo` is for pre-penetration-test auth (old front-end); `AppID` is for penetration-enabled front-end.
- `RegisterUserSystemInfo` "operation not permitted" error usually means wrong AppID.

### Collection data format
- `CTP_GetSystemInfo` returns the **original binary string** (not base64). Submit as-is.
- `pSystemInfo` buffer must be at least **270 bytes**.
- Collection info is **UTF-8 encoded**.
- `RegisterUserSystemInfo` returns `-1` when client info fields (e.g. `ClientAppID`) are empty.

### Linux collection notes
- Hardware/CPU/BIOS serial numbers may fail to collect.
- Collection binary needs **`u+s` permission** on Linux.
- If the roaming/service is not deployed, collection fails.
- If hardware doesn't carry the serial number, it can't be collected.

### Data collection return values (bitmask)
`CTP_GetSystemInfo` returns a bitmask; 0 = all success.

| Bit | Meaning |
|-----|---------|
| 0 | Terminal info not collected |
| 1 | Exception during collection |
| 2 | IP retrieval failed |
| 3 | MAC retrieval failed |
| 4 | Device name retrieval failed |
| 5 | OS version retrieval failed |
| 6 | Hardware serial number retrieval failed |
| 7 | CPU serial number retrieval failed |
| 8 | BIOS retrieval failed |
| 9 | System disk partition retrieval failed (Windows only) |

### Thread safety
- Data collection APIs (`RegisterUserSystemInfo`, `SubmitUserSystemInfo`, `CTP_GetSystemInfo`) are **NOT thread-safe**. Lock if calling from multiple threads.

## 22. API Version Compatibility

### Version coupling
- **API and front-end versions must match.** Mismatched versions:
  - Old error: "Decrypt handshake data failed"
  - v6.3.20+: `OnRspError` with "CTP:API Front shake hand err : version err" or "decode err"
- V6.3.X APIs → only CTP backend v6.3.X~6.5.X (no combo contracts, no IPv6).
- V6.5.X APIs → only CTP backend v6.5.X+ (combo contracts, options combos, IPv6).
- V6.5.0 API features work on 6.5.0 backend only. New features in 6.5.1 need 6.5.1 backend.
- New API version connecting to old front-end: new features silent-fail.

### Breaking change: production mode default (v6.7.11)
- `CreateFtdcTraderApi(pszFlowPath, blsProductionMode)` — default changed from `false` to **`true`**.
- `CreateFtdcMdApi(pszFlowPath, bUsingUdp, bMulticast, blsProductionMode)` — default changed to `true`.
- **If using a test environment, must explicitly pass `false`**, or handshake fails with "version error".

### Field size changes (v6.5.1)
- InstrumentID: 31 → 81 bytes. Old fields renamed to `reserve1/2/3` (do NOT use).
- IPAddress: 16 → 33 bytes. Old field renamed to `reserve2` (do NOT use).
- `RegisterFront` / `RegisterNameServer` format: `tcp://ip:port` (IPv4) or `tcp6://[ip]:port` (IPv6).

### DLL upgrade
- Same OS family (e.g. Windows x64): just replace `.dll`, no recompile needed.
- Mobile: replace SDK and rebuild.

### API function rename (v6.5.1)
- `ReqQueryMaxOrderVolume` → `ReqQryMaxOrderVolume`. Old name still works but deprecated.

### Compression
- v6.7.0+: LZ4 compression for query responses (new API + new front only).

## 23. Key Error Codes Reference

### Trading errors
| Code | Name | Meaning |
|------|------|---------|
| 42 | SETTLEMENT_NOT_CONFIRMED | Settlement info not confirmed — cannot trade |
| 90 | NEED_RETRY | Query not complete; retry later |
| 91 | EXCHANGE_RTNERROR | Error returned by exchange |
| 116 | ORDER_FREQ_LIMIT | Order frequency limit exceeded |
| 131 | WEAK_PASSWORD | Password too weak; must change |
| 132 | SIMPLE_PASSWORD | Password too simple; must change |
| 140 | FIRST_LOGIN | First login; must change password |
| 152 | DEL_COMB_ACTION_TOO_FAST | Combo action: original order must wait 30s |
| 154 | QK_BUSY | Query engine busy; retry later |
| 155 | CFMMC_NO_CONNECTION | Not connected to settlement server |
| 162 | NOT_TRADING_PERIOD | Product not in tradable session |
| 163 | PRICE_OVER_LIMIT | Price exceeds daily limit |
| 165 | PRICE_WRONG_TICK | Price does not meet tick size |

### Connection / auth errors
| Code | Name | Meaning |
|------|------|---------|
| 4043 | USER_CLIENT_AUTH_FAILED | AppID not authorized for this investor |
| 4100 | SMAPI_CONNECT_FAILED | SSL connect failed |
| 4110 | SMAPI_CERT_EXPIRED | Client certificate expired |

### Subscription / combo errors
| Code | Name | Meaning |
|------|------|---------|
| 6000 | OVER_SUB_INST_LIMIT | Subscribed too many instruments |
| 6011 | COMB_PARKED_ORDER_INSERT | Combo contracts don't support parked orders |
| 6012 | COMB_PARKED_ORDER_ACTION | Combo contracts don't support parked order cancel |
| 6013 | COMB_CONDITION_ORDER_INSERT | Combo contracts don't support conditional orders |
| 6022 | OFFSET_TIME_FORBIDDEN | Option self-close forbidden during settlement/hedge time |

### SMS auth (v6.7.13+)
| Code | Name | Meaning |
|------|------|---------|
| 6014 | MOBILE_NOT_MATCH | Phone number doesn't match |
| 6017 | SMSCODE_MISMATCH | Verification code incorrect |
| 6018 | SMSCODE_USED | Verification code already used |
| 6019 | SMSCODE_OUTOFTIME | Verification code expired |
| 6020 | SMSCODE_NEEDED | SMS verification required |

### IP / login lock errors
- "CTP: user is not active" or "CTP: illegal login" → too many failed login attempts.
- IP lock: ~5000 cumulative failures per IP (never resets).
- IP + account lock: ~6-10 failures per account (cumulative in CTP backend).
- Actual thresholds set by broker.

## 24. Structural Notes for This Codebase

Given the three-layer architecture (C → Bridge.Net → .Net):

- **TradingDay**: The public API should expose TradingDay from login responses and `GetTradingDay()`, not from MD snapshot dates.
- **OrderSysID**: Store as-is (string, no trimming). The managed layer should preserve leading/trailing whitespace from the native buffer.
- **OrderRef**: The managed layer should validate int32 range before marshalling. Consider exposing as `int32` in the public API.
- **UserID**: Always populate in order requests. The bridge and managed layers should enforce this.
- **Turnover/Volume**: Handle the `DBL_MAX` sentinel. Treat as null/invalid in the managed layer.
- **Encoding**: Settlement info (结算单) content may contain mixed encodings. Existing GB18030 inbound decoding handles this, but concatenating multi-part responses must preserve byte boundaries.
- **InvestUnitID**: The public API could allow an optional `strategyTag : string` parameter that maps to this field.
- **Position/P&L**: If implementing position tracking in the managed layer, follow the FIFO-by-exchange rules and distinguish 平今/平昨 commission rates even for DCE/CZCE.
- **Options**: `CashIn` must flow through to account equity calculations. Position P&L for options is always zero.
- **ReqQryClassifiedInstrument**: v6.5.1+. Prefer this over `ReqQryInstrument` for filtered queries. Both TradingType and ClassType must be set to valid enum values — never 0.
- **Flow control**: The managed layer should handle `-2` and `-3` return values from API calls. Retry logic with backoff for error 90 (NEED_RETRY) and 154 (QK_BUSY).
- **Connection management**: Separate flow directories per API instance. Validate `ulimit -n` before Init. Handle reconnect with re-authentication.
- **Production mode flag**: Expose `blsProductionMode` in the public API constructor. Default to `true` for production, but allow test environments.

## 25. Original sources

### Bilibili (开心秋水)
- https://www.bilibili.com/opus/794777354719199264 (爬坑指北一: TradingDay, order keys, commission, 平仓顺序)
- https://www.bilibili.com/opus/795446433312407616 (爬坑指北二: OnErrRtnOrderInsert, GDB SIGUSR1, SimNow)
- https://www.bilibili.com/opus/795450762638393385 (爬坑指北三: app-only)
- https://www.bilibili.com/opus/795469492994965511 (爬坑指北五: CombOffsetFlag, thread safety, settlement)
- https://www.bilibili.com/opus/804127330764062741 (爬坑指北七: ClassifiedInstrument, InvestUnitID, 盘后行情)
- https://www.bilibili.com/opus/926136305032626176 (爬坑指北八: 强平, 股票期权 vs 期货期权)
- https://www.bilibili.com/opus/1084524373815066628 (爬坑指北九: Turnover, Level2 vs Level1)
- https://www.bilibili.com/opus/794766647360487447 (持仓和资金维护)
- https://www.bilibili.com/opus/794752684399788073 (期权与期货区别)
- https://www.bilibili.com/opus/794761858475098149 (组合合约浅析)

### CTP Official Reference (v6.7.13)
- `docs/reference/` — full CHM decompilation: API reference, FAQ, version changelogs, error codes
- Key documents: `CJFAQ.html` (data collection FAQ), `JKZYXZYXHSM.html` (order identifier notes), `LK.html` (connection flow control), `HQSM.html` (market data notes)
