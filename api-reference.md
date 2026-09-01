# API Reference

Complete reference for the StockIntel Trading API WebSocket protocol.

---

## Connection

```
wss://trading.stockintel.com/ws/v1
```

- **TLS required** — plaintext connections are refused.
- **Subprotocol** — send `Sec-WebSocket-Protocol: capri.v1`. The server echoes it on success.
- **All frames are binary** (opcode 0x2). Text frames are rejected with close code `4000`.
- **One frame = one message** — each binary frame is exactly one serialized `ClientFrame` (client→server) or `ServerFrame` (server→client).
- **Compression** — `permessage-deflate` is disabled. Protobuf frames are already compact.

---

## Authentication

Authenticate at the WebSocket upgrade by including your API token in the `Authorization` header:

```
GET /ws/v1 HTTP/1.1
Host: trading.stockintel.com
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer si_lv_YourLiveTokenHere...
Sec-WebSocket-Protocol: capri.v1
```

| Scenario | HTTP Response |
|---|---|
| Valid token, token service reachable | `101 Switching Protocols` |
| Missing, malformed, unknown, or expired token | `401 Unauthorized` (generic; socket never opens) |
| Not entitled to the API | `403 Forbidden`, body is the reason: `not_eligible`, `invalid_plan`, or `ip_not_allowed` |
| Token service unreachable (uncached token) | `503 Service Unavailable` |

A `403` will not clear by retrying:

| Body | Meaning | What to do |
|---|---|---|
| `not_eligible` | No brokerage account opened through StockIntel | Open one, then reconnect |
| `invalid_plan` | Your subscription is not active | Renew it, then reconnect |
| `ip_not_allowed` | Your IP is outside the token's allow-list | Connect from an allowed address, or change the token's restrictions |

- The token prefix determines the environment: `si_sb_` → sandbox, `si_lv_` → live.
- The token is verified once at connect, and the server may require a one-time code before trading — in **both** environments. See the [OTP Gate](#otp-gate) below.
- Your plan's limits arrive with the connection in `Welcome.quotas`. See [Plan Quotas](#plan-quotas).
- The token is **not** re-sent on individual frames.
- Tokens are only issued to users who have opened a brokerage account with one of the available brokers through StockIntel. See [Getting Started](./getting-started.md#2-generate-api-tokens) for how to obtain one.

---

## Envelope Protocol

Every frame is one of two top-level envelopes defined in [`capri.proto`](./capri.proto):

- **`ClientFrame`** — sent by you. Contains a `request_id` and exactly one `command` (via `oneof`).
- **`ServerFrame`** — sent by the server. Contains a `request_id` (mirrors the command's, or empty for pushes) and exactly one `payload` (via `oneof`).

### `request_id` Rules

- Every command must carry a non-empty `request_id` — a **UUID string** (UUIDv4 recommended).
- The server echoes `request_id` on the matching result or error.
- For `PlaceOrder`, the `request_id` becomes the order's lifecycle handle — it is echoed on **every** `ExecutionEvent` for that order.
- `request_id` must **never be reused** on a connection. Reuse closes the socket with `4000` (`PROTOCOL_ERROR`).
- Server pushes (`Welcome`, `ExecutionEvent` for external orders, `TradingSessionStatus`) carry an empty `request_id` (`""`).

---

## Operations

### Command/Response Operations

These follow a strict request→single-response pattern. You send a `ClientFrame`, the server replies with one `ServerFrame` carrying the same `request_id`.

### Server Push Operations

These are unsolicited server→client frames. `Welcome`, `ExecutionEvent`, and `TradingSessionStatus` arrive automatically from the moment the connection opens — no subscribe step.

`QuoteUpdate` and `OrderBookUpdate` are the exception: they are per-symbol and high-volume, so you name the symbols you want first. See [Market Data](#market-data).

---

## OTP Gate

The server requires a one-time verification code before trading commands are allowed, whenever your token has not been OTP-verified within the past 7 days. **Both environments are gated**, so a client drives one flow for sandbox and live alike — only the source of the code differs.

**When OTP is required**, the `Welcome` frame carries:

| Field | Value |
|---|---|
| `otp_required` | `true` |
| `has_email` | `true` if a code was emailed; `false` if no email is on file. Meaningless for sandbox — no email is ever sent |
| `otp_message` | Human-readable instruction to display to the user |
| `quotas` | Absent — your entitlements are published with your accounts |

- **Live (`si_lv_...`)**: if `has_email` is `true`, check your email for a 5-digit code. If it is `false`, add an email address in your **StockIntel settings** and reconnect.
- **Sandbox (`si_sb_...`)**: the code is always **`54321`**. Nothing is emailed. It is a published constant, not a secret — the gate exists so sandbox behaves like live.

Until OTP succeeds, **all trading and account commands return `ERROR_CODE_OTP_REQUIRED`**.

### `SubmitOtp`

Submit the 5-digit code for a session whose `Welcome` carried `otp_required: true`.

**Request: `SubmitOtpRequest`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `code` | string | Yes | 5-digit code — from your email for live, always `54321` for sandbox |

**Response: `SubmitOtpResponse`**

| Field | Type | Notes |
|---|---|---|
| `environment` | string | `"sandbox"` or `"live"` |
| `accounts` | repeated `AccountMeta` | Your now-unlocked trading accounts |
| `quotas` | `Quotas` | The limits now in force — see [Plan Quotas](#plan-quotas) |

After a successful `SubmitOtp`, trading and account commands are allowed and the session is cached for future reconnects within the 7-day window.

**Error cases:**

| Error | Meaning |
|---|---|
| `ERROR_CODE_INVALID_OTP` | Wrong, expired, or rate-limited code (5 attempts per 10 min). Session stays gated. |
| `ERROR_CODE_INVALID_REQUEST` (`reason: "otp_not_required"`) | Sent `SubmitOtp` when no OTP was required. |

---

## Plan Quotas

Your StockIntel plan sets what the API lets you do. The limits arrive with your session in `Welcome.quotas` (or in `SubmitOtpResponse.quotas` if your session was OTP-gated), so you can read them at connect rather than discover them by being refused.

**`-1` means unlimited** on every numeric field.

| Field | Type | Meaning |
|---|---|---|
| `orders_per_day` | int32 | Orders you may place per trading day |
| `api_requests_per_minute` | int32 | Commands per minute. **`PlaceOrder` and `CancelOrder` do not count** against it |
| `order_types` | repeated `OrderType` | The order types you may place |
| `good_till_cancelled` | bool | Whether you may set `time_in_force = TIME_IN_FORCE_GTC` |
| `quote_subscriptions` | int32 | Concurrent quote subscriptions per connection — see [Market Data](#market-data) |
| `orderbook_subscriptions` | int32 | Concurrent order-book subscriptions per connection |
| `historical_data_days` | int32 | How far back [`GetHistorical`](#gethistorical) reaches, in **trading days**. `0` means no access |
| `historical_data_intervals` | repeated string | The candle intervals you may ask for. Currently `1d` — daily candles are the only interval offered |

### What happens when you hit one

| Quota | Error |
|---|---|
| `orders_per_day` | `ERROR_CODE_QUOTA_EXCEEDED`, `reason: "daily_order_quota_exceeded"`, with `retry_after_ms` counting down to the next trading day |
| `api_requests_per_minute` | `ERROR_CODE_RATE_LIMITED`, `reason: "api_rate_exceeded"`, with `retry_after_ms` |
| `order_types` | `ERROR_CODE_PERMISSION_DENIED`, `reason: "order_type_not_entitled"` |
| `good_till_cancelled` | `ERROR_CODE_PERMISSION_DENIED`, `reason: "time_in_force_not_entitled"` |
| `quote_subscriptions` / `orderbook_subscriptions` | Not an error. The symbols past the cap come back in `SubscriptionResponse.rejected` with `SUBSCRIPTION_REJECT_REASON_QUOTA_EXCEEDED` while the rest take effect |
| `historical_data_intervals` | `ERROR_CODE_PERMISSION_DENIED`, `reason: "historical_interval_not_entitled"` — an interval other than the `1d` currently offered |
| `historical_data_days` = `0` | `ERROR_CODE_PERMISSION_DENIED`, `reason: "historical_data_not_entitled"` |
| `historical_data_days` exceeded | Not an error. The range is **trimmed** to what the plan covers and served — see [`GetHistorical`](#gethistorical) |

### Rules worth knowing

- **A day is a trading day in market time (`Asia/Karachi`)**, not a UTC day — your allowance turns over between sessions, not mid-session.
- **Only accepted orders count.** An order rejected by validation or by a limit costs nothing. An order the broker later rejects is *not* refunded — the cap is on order flow you push at the market.
- **Reconnecting does not reset anything.** Counters follow your user and environment, not your connection or your token, so a fresh socket or a new token starts where the old one left off. Sandbox and live count separately.
- **Sandbox always runs on the default plan limits**, whatever your live plan grants.
- **Subscription caps are concurrency, not consumption.** They limit how many symbols you may hold at once; unsubscribing frees the allowance immediately, and there is no daily total.
- **Market data is entitled by plan alone.** Any symbol the exchange publishes is readable by any entitled session — unlike trading, which is scoped to the accounts on your token.
- The `5 orders/sec` burst limit applies on top of these quotas, and is not part of your plan.

---

## Commands

### `PlaceOrder`

Place a new order. Returns an **immediate empty acknowledgement** — the order's lifecycle (submitted → queued → partial → filled → cancelled) arrives on the automatic `ExecutionEvent` stream, correlated by the same `request_id`.

**Request: `PlaceOrderRequest`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `broker_code` | string | Yes | Broker identifier (from Welcome.accounts) |
| `client_code` | string | Yes | Account client code (from Welcome.accounts) |
| `market` | `Market` enum | Yes | Must be `MARKET_REG` for v1 |
| `symbol` | string | Yes | Trading symbol, e.g. `"AAPL"` |
| `type` | `OrderType` enum | Yes | `MARKET`, `LIMIT`, `MARKET_TO_LIMIT`, `STOP_LOSS` |
| `side` | `OrderSide` enum | Yes | `BUY`, `SELL`, `BORROW`, `SHORT_SELL` |
| `quantity` | double | Yes | Must be > 0 |
| `price` | double | LIMIT only | Required for `LIMIT`, ignored for `MARKET` |
| `stop_price` | double | STOP_LOSS only | Stop trigger price |
| `time_in_force` | `TimeInForce` enum | No | Defaults to `DAY` |
| `pin` | string | Yes | Account PIN (set in StockIntel Settings); verified server-side, never forwarded to broker |

**Response: `PlaceOrderResponse`** — empty message (acknowledgement only).

**Key behavior:**
- The response is an **acceptance ack** — not an execution result.
- All outcomes (fills, rejections, cancellations) arrive on the `ExecutionEvent` stream.
- An `ExecutionEvent` for the order **may arrive before** the `PlaceOrderResponse` ack. Always correlate by `request_id`.
- Track the order's lifecycle on the stream; once `broker_order_id` / `exchange_order_id` are assigned, they become the durable handles.
- The client-supplied `pin` is verified against the account's reference PIN. If the account has no PIN configured → `PIN_NOT_SETUP`. If missing or mismatched → `INVALID_PIN`.

---

### `CancelOrder`

Cancel an open order. Returns an **immediate empty acknowledgement** — the definitive outcome (cancelled or rejection) arrives on the `ExecutionEvent` stream.

**Request: `CancelOrderRequest`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `broker_code` | string | Yes | Account broker |
| `client_code` | string | Yes | Account client code |
| `broker_order_id` | string | At least one | Broker-assigned order ID (FIX tag 11) |
| `exchange_order_id` | string | At least one | Exchange-assigned order ID (FIX tag 37) |
| `pin` | string | Yes | Account PIN |

**Response: `CancelOrderResponse`** — empty message (acknowledgement only).

**Key behavior:**
- At least one of `broker_order_id` / `exchange_order_id` is required.
- Like `PlaceOrder`, the response is an ack only. The definitive outcome arrives as an `ExecutionEvent` on the stream.
- A successful cancel produces `ORDER_STATUS_CANCELLED` with `broker_orig_order_id` referencing the original order.
- A broker rejection of the cancel surfaces as an `ExecutionEvent` (e.g., `ORDER_STATUS_REJECTED`).

---

### `ListOrders`

Retrieve paginated order history and status.

**Request: `ListOrdersRequest`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `broker_code` | string | Yes | |
| `client_code` | string | Yes | |
| `status` | `OrderStatus` enum | No | Filter by status; `UNSPECIFIED` = all |
| `symbol` | string | No | Filter by symbol |
| `market` | `Market` enum | No | Filter by market |
| `side` | `OrderSide` enum | No | Filter by side |
| `type` | `OrderType` enum | No | Filter by type |
| `date_from` | string | No | `YYYY-MM-DD` |
| `date_to` | string | No | `YYYY-MM-DD` |
| `offset` | int32 | No | Pagination offset, ≥ 0 |
| `count` | int32 | No | Page size, clamped to [1, 25] |

**Response: `ListOrdersResponse`**

| Field | Type | Notes |
|---|---|---|
| `offset` | int32 | Current offset |
| `count` | int32 | Number of orders in this page |
| `total` | int32 | Total matching orders |
| `orders` | repeated `Order` | Order list |

**`Order`**

| Field | Type | Notes |
|---|---|---|
| `id` | string | Order record ID (assigned once persisted) |
| `broker_code` | string | |
| `client_code` | string | |
| `market` | `Market` enum | |
| `symbol` | string | |
| `type` | `OrderType` enum | |
| `side` | `OrderSide` enum | |
| `status` | `OrderStatus` enum | Current status |
| `price` | double | |
| `stop_price` | double | |
| `quantity` | double | Original order quantity |
| `time_in_force` | `TimeInForce` enum | |
| `broker_order_id` | string | Broker order ID (FIX tag 11) |
| `exchange_order_id` | string | Exchange order ID (FIX tag 37) |
| `fills` | repeated `OrderFill` | Individual fills |
| `created_at` | `google.protobuf.Timestamp` | |
| `updated_at` | `google.protobuf.Timestamp` | |

**`OrderFill`**

| Field | Type | Notes |
|---|---|---|
| `price` | double | Fill price |
| `quantity` | double | Fill quantity |

---

### `ListAccounts`

Returns trading accounts linked to your token — directly from the verified session, no upstream round-trip. (The same data is delivered unsolicited in the `Welcome` frame at connect time.)

**Request: `ListAccountsRequest`** — empty.

**Response: `ListAccountsResponse`**

| Field | Type | Notes |
|---|---|---|
| `environment` | string | `"sandbox"` or `"live"` |
| `accounts` | repeated `AccountMeta` | |

**`AccountMeta`**

| Field | Type | Notes |
|---|---|---|
| `broker_code` | string | Broker identifier |
| `client_code` | string | Account client code |
| `status` | `AccountStatus` enum | `ACTIVE` (1) or `INACTIVE` (0) |

---

### `GetAccount`

Fetch the account snapshot: value, balance, buying power, and positions. Served from a cache that refreshes automatically in the background when data is older than 5 minutes. Use `created_at` and `last_updated` to determine data age.

**Request: `GetAccountRequest`**

| Field | Type | Required |
|---|---|---|
| `broker_code` | string | Yes |
| `client_code` | string | Yes |

**Response: `GetAccountResponse`**

| Field | Type | Notes |
|---|---|---|
| `broker_code` | string | |
| `client_code` | string | |
| `value` | double | Total account value |
| `market_value` | double | Market value of holdings |
| `balance` | double | Cash balance |
| `buying_power` | double | Available buying power |
| `positions` | repeated `Position` | Current holdings |
| `created_at` | `google.protobuf.Timestamp` | Snapshot creation time |
| `last_updated` | `google.protobuf.Timestamp` | Snapshot last-update time |

**`Position`**

| Field | Type |
|---|---|
| `symbol` | string |
| `market` | `Market` enum |
| `quantity` | int64 |
| `market_price` | double |
| `market_value` | double |
| `avg_price` | double |
| `avg_value` | double |
| `haircut` | double |
| `haircut_value` | double |

---

### `GetSessionStatus`

Current trading session status for a broker + market — a point-in-time query. Ongoing changes arrive automatically via `TradingSessionStatus` pushes; use this for the current value on demand.

**Request: `GetSessionStatusRequest`**

| Field | Type | Required |
|---|---|---|
| `broker_code` | string | Yes |
| `market` | `Market` enum | Yes |

**Response: `GetSessionStatusResponse`**

| Field | Type |
|---|---|
| `status` | `TradingSessionStatus` |

**`TradingSessionStatus`**

| Field | Type |
|---|---|
| `broker_code` | string |
| `market` | `Market` enum |
| `status` | `SessionStatus` enum |
| `timestamp` | `google.protobuf.Timestamp` |

---

## Market Data

Quotes and order books are **not** pushed from connect. They are per-symbol and high-volume, unlike the entitlement-scoped execution and session-status streams, so you name the symbols you want.

Four commands — subscribe and unsubscribe, quotes and order book — all answer with the same `SubscriptionResponse`. Updates then arrive as `QuoteUpdate` and `OrderBookUpdate` pushes.

Both streams are **snapshot-based**: every message carries a symbol's full state rather than a delta against the last one. Three things follow from that, and they are the whole design:

- **A slow client loses frames, not its connection.** Each connection has a second, separate queue for market data, so a burst of quotes can never cost you an execution report. When it overflows, market-data frames are dropped; the socket stays open. (The trading queue still closes the socket with `4004` / `SLOW_CONSUMER` — that data is not replaceable.)
- **Subscribing replays the last known snapshot immediately**, so you see data without waiting for the market to move — or, outside session hours, at all.
- **There is no sequence number and no gap recovery**, because a missed frame costs nothing the next frame will not carry. Do not build resync logic for this stream.

If the server has no market-data feed configured, every subscribe is refused with `ERROR_CODE_UPSTREAM_UNAVAILABLE` (`reason: "market_data_disabled"`). Trading is unaffected — a feed outage does not gate the API.

### `SubscribeQuotes` / `SubscribeOrderBook`

**Request: `SubscribeQuotesRequest` / `SubscribeOrderBookRequest`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `symbols` | repeated `SymbolRef` | Yes | At most **500** per request |

`SymbolRef` is `{ market: Market, symbol: string }` — the same shape in both directions.

Symbols are **resolved individually, never all-or-nothing.** Malformed symbols and those past your plan's cap come back in `rejected` while the rest take effect. Only a request that is wrong as a whole — market data not configured, or more than 500 symbols — is answered with an `Error` instead of a `SubscriptionResponse`.

Subscribing to a symbol you already hold is accepted and consumes no further allowance. Duplicates within one request collapse to one.

Symbols are **not** checked against a universe of known instruments: the feed is the only authority on what exists. A well-formed subscription to a symbol that never trades is accepted and simply stays silent.

### `UnsubscribeQuotes` / `UnsubscribeOrderBook`

**Request: `UnsubscribeQuotesRequest` / `UnsubscribeOrderBookRequest`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `symbols` | repeated `SymbolRef` | No | **Empty drops every subscription** on that stream |

Unsubscribing a symbol you never held is not an error — it is simply absent from `accepted`.

### `SubscriptionResponse`

Answers all four commands. Which oneof slot it arrives in tells you which command it answers.

| Field | Type | Notes |
|---|---|---|
| `accepted` | repeated `SymbolRef` | Now subscribed. On unsubscribe, the ones removed |
| `rejected` | repeated `SubscriptionRejection` | Per-symbol refusals, with a reason each |
| `active` | int32 | Your subscription count for this stream **after** the change |
| `limit` | int32 | Your plan's cap on it (`-1` unlimited) |

`active` is authoritative — track it rather than counting your own subscribes.

### `SubscriptionRejectReason`

| Value | Meaning |
|---|---|
| `SUBSCRIPTION_REJECT_REASON_QUOTA_EXCEEDED` (1) | Your plan's concurrent-subscription cap is already spent |
| `SUBSCRIPTION_REJECT_REASON_INVALID_SYMBOL` (2) | `market` is `UNSPECIFIED`, or `symbol` is empty or over 32 characters |
| `SUBSCRIPTION_REJECT_REASON_UNSUPPORTED_MARKET` (3) | That market publishes nothing on this stream — indices have no order book |

### `QuoteUpdate`

A full quote snapshot for one symbol.

| Field | Type | Notes |
|---|---|---|
| `market` / `symbol` | `Market` / string | Which instrument |
| `state` | `SessionStatus` | The market's session state at this snapshot |
| `timestamp` | Timestamp | When the feed published it |
| `open` / `high` / `low` / `close` | double | The day's prices. `close` is the last traded price, not a settled close |
| `volume` | int64 | Shares traded today |
| `last_day_close` | double | Yesterday's close (LDCP), the reference `change` is measured from |
| `change` / `change_percent` | double | Against `last_day_close` |
| `bid_price` / `bid_quantity` | double / int64 | Best bid |
| `ask_price` / `ask_quantity` | double / int64 | Best ask |
| `value` | double | Traded value for the day, in currency rather than shares |
| `trades` | int64 | Trade count for the day |
| `last_trade` | `LastTrade` | Absent when the symbol has not traded yet |

`LastTrade` is `{ timestamp, price, quantity }`.

### `OrderBookUpdate`

A full order-book snapshot for one symbol.

| Field | Type | Notes |
|---|---|---|
| `market` / `symbol` | `Market` / string | Which instrument |
| `timestamp` | Timestamp | When the feed published it |
| `buy` / `sell` | `OrderBookSide` | The two sides |

`OrderBookSide` carries `weighted_avg_price` and `total_quantity` for the **whole** side — including levels beyond those listed — plus `levels`, best price first. The exchange publishes up to 10.

Each `OrderBookLevel` is `{ level, price, quantity, order_count }`, where `order_count` is how many separate orders rest at that price.

Indices (`MARKET_IDX`) are rejected on this stream: they are computed from the market rather than traded, so they have no resting orders.

---

## Historical Data

### `GetHistorical`

One symbol's history over a time range. Unlike quotes and the order book this is a plain request/response — history does not change, so there is nothing to stream and nothing to hold open.

**Request: `GetHistoricalRequest`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `market` | `Market` | Yes | Required though history is keyed by symbol alone today, so an instrument is named the same way here as in a subscription |
| `symbol` | string | Yes | Up to 32 characters |
| `interval` | string | Yes | `1d`. Read the codes your plan grants from `historical_data_intervals` rather than hard-coding this |
| `time_from` | Timestamp | Yes | Start of the range |
| `time_to` | Timestamp | Yes | End of the range. Must be after `time_from` |

**Both bounds are required.** An open-ended range would have to be closed with a server-side default, and a client that meant something else would never find out.

**One request may cover at most 90 days** at `1d`. A wider range is refused with `ERROR_CODE_INVALID_REQUEST` (`reason: "range_too_large"`) — page through longer histories by asking again with an earlier range.

The cap is per interval, so it will change if finer intervals are offered later: a range costs what it resolves into, and a year of daily candles is a very different answer from a year of minute ones. Read `historical_data_intervals` at connect rather than assuming the set.

**Response: `GetHistoricalResponse`**

| Field | Type | Notes |
|---|---|---|
| `market` / `symbol` / `interval` | | Echo the request, so a client holding several charts can route a result without tracking `request_id`s itself |
| `data` | repeated `HistoricalData` | Ordered **oldest first** |
| `time_from` / `time_to` | Timestamp | The range **actually served** — see below |

`HistoricalData` is one interval of trading, as a candle: `{ timestamp, open, high, low, close, volume }`.

`timestamp` is where the interval **closes** — the last trade counted into it — so a daily point is stamped at the session's close rather than its open.

### Reaching further back than your plan covers

This is **not** an error. The range is trimmed to your plan's last `historical_data_days` sessions and served, and the response's `time_from` reports what it was trimmed to. Compare it against what you asked for to know it happened; a client charting further back pages by asking again with an earlier range.

Two cases are still refused outright:

| Case | Error |
|---|---|
| `historical_data_days` is `0` | `ERROR_CODE_PERMISSION_DENIED`, `reason: "historical_data_not_entitled"` |
| The range lies **entirely** outside the plan's window | `ERROR_CODE_PERMISSION_DENIED`, `reason: "historical_window_not_entitled"` |

The second is refused rather than trimmed because there is nothing to trim it to, and an empty result would be indistinguishable from a symbol that did not trade.

**Days are counted in trading days**, so a seven-day plan reaches seven *sessions* back however many weekends and market holidays lie between them.

A range the symbol never traded in comes back **empty rather than as an error**.

If the server has no market-data feed configured, the request is refused with `ERROR_CODE_UPSTREAM_UNAVAILABLE` (`reason: "historical_data_disabled"`).

---

## Server Pushes

### `Welcome`

Pushed **once**, immediately after a successful WebSocket upgrade. Carries your environment, accounts, keepalive cadence, and server version — no separate `ListAccounts` call needed.

| Field | Type | Notes |
|---|---|---|
| `environment` | string | `"sandbox"` or `"live"` |
| `accounts` | repeated `AccountMeta` | Every account linked to your token. Empty when `otp_required` is `true`. |
| `heartbeat_interval_ms` | uint32 | Server ping cadence (for info; library handles pong) |
| `server_version` | string | Server version string |
| `otp_required` | bool | `true` when the token needs OTP verification. Applies to both environments. |
| `has_email` | bool | `true` if an OTP code was emailed. Only meaningful for a gated **live** token. |
| `otp_message` | string | Human-readable instruction (non-empty only when `otp_required` is `true`). Display this to the user. |
| `quotas` | `Quotas` | What your plan entitles you to. Absent while `otp_required` is `true`. See [Plan Quotas](#plan-quotas). |

---

### `ExecutionEvent`

Pushed **automatically** for every order execution update your accounts are entitled to. No subscription needed.

| Field | Type | Notes |
|---|---|---|
| `execution` | `OrderExecution` | |

**`OrderExecution`**

| Field | Type | Notes |
|---|---|---|
| `broker_code` | string | |
| `client_code` | string | |
| `status` | `OrderStatus` enum | Current order status |
| `symbol` | string | |
| `market` | `Market` enum | |
| `type` | `OrderType` enum | |
| `side` | `OrderSide` enum | |
| `time_in_force` | `TimeInForce` enum | |
| `broker_order_id` | string | Broker order ID (FIX tag 11) |
| `exchange_order_id` | string | Exchange order ID (FIX tag 37) |
| `broker_orig_order_id` | string | Set only on cancellation executions; references the cancelled order's `broker_order_id` (FIX tag 41) |
| `price` | double | Order limit price |
| `quantity` | double | Original order quantity |
| `last_price` | double | Last fill price |
| `last_quantity` | double | Last fill quantity |
| `quantity_remaining` | double | Unfilled quantity |
| `quantity_executed` | double | Cumulative filled quantity |
| `message` | string | Human-readable status message |
| `timestamp` | `google.protobuf.Timestamp` | Event timestamp |

**Correlation rules:**
- Orders **you placed** on this connection: `request_id` in the `ServerFrame` echoes your `PlaceOrder`'s `request_id`.
- Orders placed **outside** this connection (broker app, terminal, another channel): `request_id` is empty (`""`). Correlate by `broker_order_id` / `exchange_order_id`.

---

### `TradingSessionStatus`

Pushed **automatically** when a market's trading session status changes. You only receive status for brokers/markets your accounts belong to.

| Field | Type | Notes |
|---|---|---|
| `broker_code` | string | |
| `market` | `Market` enum | |
| `status` | `SessionStatus` enum | |
| `timestamp` | `google.protobuf.Timestamp` | |

---

## Enums

### `Market`

| Value | Description |
|---|---|
| `MARKET_REG` (1) | Regular — tradeable in v1 |
| `MARKET_BNB` (2) | Bills and Bonds |
| `MARKET_FUT` (3) | Futures |
| `MARKET_ODL` (4) | Odd Lot |
| `MARKET_SQR` (5) | Square-up |
| `MARKET_FSR` (6) | Futures Square-up |
| `MARKET_NDM` (7) | Negotiated Deal Market |
| `MARKET_SIF` (8) | Stock Index Futures |
| `MARKET_IOM` (9) | Index Options Market |
| `MARKET_SOM` (10) | Stock Options Market |
| `MARKET_IDX` (11) | Indices. Quote-only — no order book, never tradeable |
| `MARKET_CSF` (12) | Cash-Settled Futures. Market data only |
| `MARKET_TRM` (13) | Term Finance. Market data only |

Markets 1–10 carry order flow. 11–13 arrive only on the market-data feed, which publishes segments there is no trading on.

The feed spells three markets differently from the trading system, and the API reports one name for each either way: feed `FSQ` is `MARKET_FSR`, `IOPT` is `MARKET_IOM`, and `SOPT` is `MARKET_SOM`. You never see the feed's spelling.

### `OrderType`

Which of these you may place depends on your plan — see [Plan Quotas](#plan-quotas).

| Value | Description |
|---|---|
| `ORDER_TYPE_MARKET` (1) | Market order |
| `ORDER_TYPE_LIMIT` (2) | Limit order |
| `ORDER_TYPE_MARKET_TO_LIMIT` (3) | Market-to-limit |
| `ORDER_TYPE_STOP_LOSS` (4) | Stop-loss |

### `OrderSide`

| Value | Description |
|---|---|
| `ORDER_SIDE_BUY` (1) | Buy |
| `ORDER_SIDE_SELL` (2) | Sell |
| `ORDER_SIDE_BORROW` (3) | Borrow |
| `ORDER_SIDE_SHORT_SELL` (4) | Short sell |

### `TimeInForce`

| Value | Description |
|---|---|
| `TIME_IN_FORCE_DAY` (1) | Day order |
| `TIME_IN_FORCE_IOC` (2) | Immediate or Cancel |
| `TIME_IN_FORCE_FOK` (3) | Fill or Kill |
| `TIME_IN_FORCE_GTC` (4) | Good Till Cancelled |

### `OrderStatus`

| Value | Description |
|---|---|
| `ORDER_STATUS_SUBMITTED` (1) | Accepted by API, forwarded |
| `ORDER_STATUS_RECEIVED` (2) | Received by trading system |
| `ORDER_STATUS_QUEUED` (3) | Queued at broker/exchange |
| `ORDER_STATUS_PARTIAL` (4) | Partially filled |
| `ORDER_STATUS_FILLED` (5) | Completely filled |
| `ORDER_STATUS_ERROR` (6) | Error |
| `ORDER_STATUS_CANCELLED` (7) | Cancelled |
| `ORDER_STATUS_REJECTED` (8) | Rejected by broker/exchange |

**Normal lifecycle:**

```
SUBMITTED → RECEIVED → QUEUED → FILLED
                              ↘ PARTIAL → FILLED
                              ↘ CANCELLED
                              ↘ REJECTED
                              ↘ ERROR
```

Any non-terminal status may transition to `CANCELLED` when a cancel request is accepted by the broker.

**Terminal states:** `FILLED` (5), `ERROR` (6), `CANCELLED` (7), `REJECTED` (8). Once an order reaches one of these, no further updates are sent.

### `SessionStatus`

| Value | Description |
|---|---|
| `SESSION_STATUS_OPEN` (1) | Open |
| `SESSION_STATUS_PRE_MARKET` (2) | Pre-market |
| `SESSION_STATUS_SUSPENDED` (3) | Suspended |
| `SESSION_STATUS_PRE_CLOSE` (4) | Pre-close |
| `SESSION_STATUS_ON_HOLD` (5) | On hold |
| `SESSION_STATUS_BREAK` (6) | Break |
| `SESSION_STATUS_READY` (7) | Ready |
| `SESSION_STATUS_NA` (8) | Not available — the broker has not yet reported a session state |

### `AccountStatus`

| Value | Description |
|---|---|
| `ACCOUNT_STATUS_INACTIVE` (1) | Inactive — cannot trade |
| `ACCOUNT_STATUS_ACTIVE` (2) | Active — can trade |

---

## `Error` Model

Errors arrive as a `ServerFrame` with the `error` payload, carrying the command's `request_id`.

| Field | Type | Notes |
|---|---|---|
| `code` | `ErrorCode` enum | |
| `message` | string | Safe, human-readable summary |
| `reason` | string | Machine-readable upstream/internal reason |
| `retry_info` | `RetryInfo` | Present on `RATE_LIMITED` and `QUOTA_EXCEEDED` |

### `ErrorCode`

| Code | Value | Meaning | Retryable |
|---|---|---|---|
| `ERROR_CODE_INVALID_REQUEST` | 1 | Malformed fields, validation failure, or upstream rejection | No (fix request) |
| `ERROR_CODE_UNAUTHENTICATED` | 2 | Mid-connection deauthorization (reserved) | No |
| `ERROR_CODE_PERMISSION_DENIED` | 3 | Account not owned, inactive for trading, environment mismatch, or an order type your plan does not include | No |
| `ERROR_CODE_RATE_LIMITED` | 6 | Order rate (5/sec) exceeded, duplicate read in-flight, or your plan's per-minute request quota exhausted | Yes (honor `retry_after_ms`) |
| `ERROR_CODE_UPSTREAM_UNAVAILABLE` | 7 | Trading system down or read request timed out | Yes (backoff) |
| `ERROR_CODE_INTERNAL` | 9 | Unexpected server error | No |
| `ERROR_CODE_PIN_NOT_SETUP` | 10 | Account has no PIN configured; cannot place/cancel orders | No (set up PIN) |
| `ERROR_CODE_INVALID_PIN` | 11 | Supplied PIN missing or doesn't match account PIN | No (supply correct PIN) |
| `ERROR_CODE_OTP_REQUIRED` | 12 | Trading/account command sent before OTP is satisfied | No (send `SubmitOtp` first) |
| `ERROR_CODE_INVALID_OTP` | 13 | Wrong, expired, or rate-limited OTP code | No (wait for retry window or reconnect to get a new code) |
| `ERROR_CODE_QUOTA_EXCEEDED` | 14 | Your plan's daily order quota is spent | Yes, but not today — `retry_after_ms` is the wait until the next trading day |

> **Note:** `UPSTREAM_UNAVAILABLE` is only returned for read commands (`ListOrders`, `GetAccount`, `GetSessionStatus`, `GetHistorical`) and for market-data subscribes when no feed is configured. `PlaceOrder` and `CancelOrder` do not time out server-side — the server holds them open until the broker responds. Design your client-side timeout logic accordingly: it is not safe to blindly retry an order command without first checking whether the original was accepted.

### `RetryInfo`

| Field | Type | Notes |
|---|---|---|
| `retry_after_ms` | uint32 | Minimum milliseconds to wait before retrying |

---

## WebSocket Close Codes

Application close codes (4000–4999) and standard codes tell you why the socket closed.

| Code | Reason | Meaning |
|---|---|---|
| `1000` | `NORMAL` | Clean close |
| `1001` | `GOING_AWAY` | Server draining for maintenance — reconnect |
| `1011` | `INTERNAL` | Unexpected server error — reconnect with backoff |
| `4000` | `PROTOCOL_ERROR` | Malformed envelope, text frame, or reused `request_id` |
| `4001` | `UNAUTHENTICATED` | Mid-connection deauthorization (reserved) |
| `4002` | `SESSION_SUPERSEDED` | Another connection took over with the same token — **do not auto-reconnect**; investigate the duplicate |
| `4003` | `RATE_LIMITED` | Connection-level abuse (flooding) |
| `4004` | `SLOW_CONSUMER` | Client couldn't keep up — outbound buffer overflowed; reconnect and resync |

---

## Rate Limits

These apply to every connection regardless of plan:

| Limit | Scope |
|---|---|
| **5 orders/sec** (`PlaceOrder` + `CancelOrder`) | Per connection |
| **Duplicate read rejection** | An identical read command still awaiting its result is rejected with `RATE_LIMITED` (`reason = "duplicate_in_flight"`). Applies to `GetHistorical` too — repeating a query while the first is in flight asks the upstream twice for the same answer |
| **500 symbols per subscribe** | One subscribe or unsubscribe may name at most 500 symbols (`reason = "too_many_symbols"`) |

Your plan adds a per-minute request cap and a daily order cap on top — see [Plan Quotas](#plan-quotas). Server pushes are never throttled.

---

## Validation Rules

The server validates all commands before forwarding upstream. Failures return `ERROR_CODE_INVALID_REQUEST` unless noted:

1. `broker_code` and `client_code` must belong to the session.
2. Trading operations (`PlaceOrder`, `CancelOrder`) require `ACCOUNT_STATUS_ACTIVE` and a valid PIN.
3. `symbol` must be non-empty (normalized to uppercase).
4. `quantity > 0`; `price > 0` for `LIMIT` orders; `stop_price > 0` for `STOP_LOSS`.
5. Enums must not be `*_UNSPECIFIED` where required (`market`, `type`, `side`).
6. `CancelOrder` requires at least one of `broker_order_id` / `exchange_order_id`.
7. `date_from` / `date_to` must be `YYYY-MM-DD` and `date_from ≤ date_to`.
8. `count` clamped to `[1, 25]`; `offset ≥ 0`.

**Envelope-level violations** (close the socket with `4000`):
- Text frame (not binary).
- Missing or empty `request_id` on a command.
- `request_id` reuse on the same connection.
- Frame fails to decode as a valid `ClientFrame`.

---

## Correlation Model Summary

| Identifier | Source | Purpose |
|---|---|---|
| `request_id` | You (UUID) | Correlates command→response; also the lifecycle handle for orders you place |
| `id` | Server (order record ID) | Persistent order identifier in `ListOrders` |
| `broker_order_id` | Broker (FIX 11) | Broker-assigned order ID |
| `exchange_order_id` | Exchange (FIX 37) | Exchange-assigned order ID |
| `broker_orig_order_id` | Broker (FIX 41) | Original order ID on cancellations |

---

## Client Guidelines

1. **Use sandbox first** — test every flow against the sandbox before going live.
2. **One connection per token** — hold at most one. Use separate tokens for separate bot instances.
3. **Fresh UUID per command** — and never reuse one on a connection.
4. **Tolerate early executions** — an `ExecutionEvent` tagged with your `request_id` may arrive before the `PlaceOrderResponse` ack.
5. **Track order lifecycle on the stream** — don't rely on command responses for order outcomes.
6. **Correlate external executions** — executions for orders placed outside your connection carry empty `request_id`; use `broker_order_id` / `exchange_order_id`.
7. **Reconnect with backoff** — on socket drop, reconnect with exponential backoff and resync via `ListOrders`. Execution events restart after reconnect but events missed during the gap are not replayed; reconcile open orders by diffing `ListOrders` against your locally tracked state.
8. **Do not auto-reconnect on `4002`** — `SESSION_SUPERSEDED` means another connection took over. Fix the duplicate.
9. **Honor `retry_after_ms`** — on `RATE_LIMITED` and `QUOTA_EXCEEDED`, wait before retrying. On a daily quota that wait runs to the next trading day; stop placing orders rather than spinning.
10. **Read your quotas at connect** — `Welcome.quotas` tells you your order types and caps. Check `order_types` before offering an order type to a user, rather than discovering it with a rejection.
11. **Read `SubscriptionResponse.rejected`** — a subscribe that succeeds partially still succeeds. Check which symbols were refused rather than assuming you hold everything you asked for, and track `active` rather than counting your own subscribes.
12. **Do not build gap recovery for market data** — both streams are snapshots, so a dropped frame is superseded by the next one. Re-subscribing after a reconnect is all the resync there is.
13. **Compare `GetHistoricalResponse.time_from` against what you asked for** — a range reaching past your plan's window is trimmed and served, not refused. Without the comparison, a short answer looks like missing data.

---

## Versioning

This API is `v1`. Backward compatibility is maintained within v1 via additive-only changes:

- New **fields** may be added to any message. Ignore unknown fields rather than treating them as errors.
- New **enum values** may be added. Treat unknown enum values as `0` / `*_UNSPECIFIED` rather than failing.
- Existing field numbers and enum values are never removed or renumbered within v1.
- **Breaking changes** (removed fields, type changes, semantic changes) require a new major version (`v2`) introduced side-by-side.

**Reserved field range:** Field numbers 40–49 in `ClientFrame` and `ServerFrame` are reserved for a future market data extension (quotes, order book). Do not use these field numbers in any custom proto extensions.
