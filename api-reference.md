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
| Valid token, navy reachable | `101 Switching Protocols` |
| Missing, malformed, unknown, or expired token | `401 Unauthorized` (generic; socket never opens) |
| Navy unreachable (uncached token) | `503 Service Unavailable` |

- The token prefix determines the environment: `si_sb_` → sandbox, `si_lv_` → live.
- The token is verified once at connect and the session is held for the connection's lifetime. No per-message auth.
- The token is **not** re-sent on individual frames.

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

These are unsolicited server→client frames. You receive them automatically from the moment the connection opens — no subscribe step.

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
| `fills` | repeated `OrderFill` | Individual fills ({price, quantity}) |
| `created_at` | `google.protobuf.Timestamp` | |
| `updated_at` | `google.protobuf.Timestamp` | |

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

Fetch the account snapshot: value, balance, buying power, and positions. Served from cache; timestamps let you judge freshness.

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

## Server Pushes

### `Welcome`

Pushed **once**, immediately after a successful WebSocket upgrade. Carries your environment, accounts, keepalive cadence, and server version — no separate `ListAccounts` call needed.

| Field | Type | Notes |
|---|---|---|
| `environment` | string | `"sandbox"` or `"live"` |
| `accounts` | repeated `AccountMeta` | Every account linked to your token |
| `heartbeat_interval_ms` | uint32 | Server ping cadence (for info; library handles pong) |
| `server_version` | string | Server version string |

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
| `broker_orig_order_id` | string | Original order ID on cancellations (FIX tag 41) |
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

### `OrderType`

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
| `SESSION_STATUS_NA` (8) | Not available / unknown |

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
| `retry_info` | `RetryInfo` | Present on `RATE_LIMITED` only |

### `ErrorCode`

| Code | Value | Meaning | Retryable |
|---|---|---|---|
| `ERROR_CODE_INVALID_REQUEST` | 1 | Malformed fields, validation failure, or upstream rejection | No (fix request) |
| `ERROR_CODE_UNAUTHENTICATED` | 2 | Mid-connection deauthorization (reserved) | No |
| `ERROR_CODE_PERMISSION_DENIED` | 3 | Account not owned, inactive for trading, or environment mismatch | No |
| `ERROR_CODE_RATE_LIMITED` | 6 | Order rate (5/sec) exceeded, or duplicate read in-flight | Yes (honor `retry_after_ms`) |
| `ERROR_CODE_UPSTREAM_UNAVAILABLE` | 7 | Trading system down or request timed out | Yes (backoff) |
| `ERROR_CODE_INTERNAL` | 9 | Unexpected server error | No |
| `ERROR_CODE_PIN_NOT_SETUP` | 10 | Account has no PIN configured; cannot place/cancel orders | No (set up PIN) |
| `ERROR_CODE_INVALID_PIN` | 11 | Supplied PIN missing or doesn't match account PIN | No (supply correct PIN) |

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

| Limit | Scope |
|---|---|
| **5 orders/sec** (`PlaceOrder` + `CancelOrder`) | Per connection |
| **Duplicate read rejection** | An identical read command still awaiting its result is rejected with `RATE_LIMITED` (`reason = "duplicate_in_flight"`) |

Server pushes are never throttled. There is no per-minute request ceiling.

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
7. **Reconnect with backoff** — on socket drop, reconnect with exponential backoff and resync via `ListOrders`.
8. **Do not auto-reconnect on `4002`** — `SESSION_SUPERSEDED` means another connection took over. Fix the duplicate.
9. **Honor `retry_after_ms`** — on `RATE_LIMITED`, wait before retrying.
