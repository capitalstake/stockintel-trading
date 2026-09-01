# Getting Started

Connect to the StockIntel Trading API in three steps: get your tokens, compile the protobuf schema, and open a WebSocket.

---

## 1. Prerequisites

- A brokerage account opened through StockIntel with one of the available brokers at [stockintel.com](https://stockintel.com). API tokens are only issued to users with a linked brokerage account.
- Familiarity with Protocol Buffers (proto3) in at least one of Python, JavaScript, Go, or similar.
- A WebSocket client that can set custom HTTP headers on the upgrade request (the API requires an `Authorization: Bearer` header — browser `WebSocket` constructors cannot set this, so the API is for server-side/programmatic clients only).

---

## 2. Generate API Tokens

> **Eligibility:** API tokens (both sandbox and live) are only available to users who have opened a brokerage account with one of the available brokers through StockIntel. If you have not yet opened a brokerage account, do so first — the **API Tokens** section will not issue keys until an account is linked. An account connected from an outside broker does not qualify: connecting with such a token is refused at the handshake with `403 not_eligible`.
>
> **An active plan is also required.** Your plan sets your API limits, and a lapsed subscription is refused at the handshake with `403 invalid_plan`. Sandbox access itself is a plan entitlement: on a plan without it, generating a sandbox token fails with `sandbox_not_allowed`. See [Plan Quotas](./api-reference.md#plan-quotas).

Tokens are issued from your StockIntel account dashboard:

1. Log in to [stockintel.com](https://stockintel.com).
2. Navigate to **Settings** → **API Tokens**.
3. Click **Generate Sandbox Token** to create a test token (prefix `si_sb_`).
4. Click **Generate Live Token** to create a production token (prefix `si_lv_`).

> **Important:**
> - You may hold at most **one sandbox token** and **one live token** at a time.
> - Tokens are shown in plaintext **only once** at generation time. Copy and store them securely — they cannot be retrieved later.
> - Tokens expire after **one year**. Generate a new token before expiry to avoid downtime.
> - Each token can power at most **one active WebSocket connection** at a time (newest connection supersedes previous).
> - If you need multiple concurrent connections (e.g., separate bots), generate separate tokens.

Token format:

```
si_sb_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890  (sandbox)
si_lv_zYxWvUtSrQpOnMlKjIhGfEdCbA0987654321  (live)
```

---

## 3. Compile the Protobuf Schema

Download [`capri.proto`](./capri.proto) and compile it for your language. See the [Protobuf Guide](./protobuf.md) for detailed instructions with Python and JavaScript examples.

**Python quickstart:**

```bash
pip install protobuf

# Download capri.proto, then:
protoc --proto_path=. --python_out=. capri.proto
```

```python
from capri.v1 import capri_pb2

frame = capri_pb2.ClientFrame()
frame.request_id = "550e8400-e29b-41d4-a716-446655440000"
frame.place_order.broker_code = "sandbox"
frame.place_order.client_code = "CS01"
frame.place_order.market = capri_pb2.MARKET_REG
frame.place_order.symbol = "AAPL"
frame.place_order.type = capri_pb2.ORDER_TYPE_MARKET
frame.place_order.side = capri_pb2.ORDER_SIDE_BUY
frame.place_order.quantity = 100
frame.place_order.time_in_force = capri_pb2.TIME_IN_FORCE_DAY
frame.place_order.pin = "1234"

binary = frame.SerializeToString()
```

**JavaScript quickstart:**

```bash
npm install google-protobuf

# Download capri.proto, then:
protoc --proto_path=. --js_out=import_style=commonjs,binary:. capri.proto
```

```javascript
const { capri } = require('./proto/capri/v1/capri_pb');

const frame = new capri.v1.ClientFrame();
frame.setRequestId('550e8400-e29b-41d4-a716-446655440000');

const order = new capri.v1.PlaceOrderRequest();
order.setBrokerCode('sandbox');
order.setClientCode('CS01');
order.setMarket(capri.v1.Market.MARKET_REG);
order.setSymbol('AAPL');
order.setType(capri.v1.OrderType.ORDER_TYPE_MARKET);
order.setSide(capri.v1.OrderSide.ORDER_SIDE_BUY);
order.setQuantity(100);
order.setTimeInForce(capri.v1.TimeInForce.TIME_IN_FORCE_DAY);
order.setPin('1234');

frame.setPlaceOrder(order);

const binary = frame.serializeBinary();
```

---

## 4. Open a WebSocket Connection

Connect to the public endpoint and authenticate with your token:

```
Endpoint:  wss://trading.stockintel.com/ws/v1
```

**Required headers on the upgrade request:**

| Header | Value |
|---|---|
| `Authorization` | `Bearer <your-token>` |
| `Sec-WebSocket-Protocol` | `capri.v1` |

**Example with Python (`websockets` library):**

```python
import asyncio
import websockets
from capri.v1 import capri_pb2

TOKEN = "si_sb_aBcDeFgHiJkLmNoPqRsTuVwXyZ..."

async def connect():
    async with websockets.connect(
        "wss://trading.stockintel.com/ws/v1",
        extra_headers={
            "Authorization": f"Bearer {TOKEN}",
            "Sec-WebSocket-Protocol": "capri.v1",
        },
        ping_interval=None,  # server sends native pings; you handle pong
    ) as ws:
        # 1. Read the Welcome frame (first message after connect)
        welcome_bin = await ws.recv()
        welcome = capri_pb2.ServerFrame()
        welcome.ParseFromString(welcome_bin)
        print(f"Connected to {welcome.welcome.environment} environment")
        for acct in welcome.welcome.accounts:
            print(f"  Account: {acct.broker_code}/{acct.client_code} "
                  f"({'active' if acct.status == capri_pb2.ACCOUNT_STATUS_ACTIVE else 'inactive'})")

        # 2. Send a command — e.g., list orders
        cmd = capri_pb2.ClientFrame()
        cmd.request_id = "b7e3c1d2-4f5a-6b7c-8d9e-0f1a2b3c4d5e"
        cmd.list_orders.broker_code = "sandbox"
        cmd.list_orders.client_code = "CS01"
        cmd.list_orders.count = 25
        await ws.send(cmd.SerializeToString())

        # 3. Read the response
        resp_bin = await ws.recv()
        resp = capri_pb2.ServerFrame()
        resp.ParseFromString(resp_bin)

        if resp.HasField("list_orders"):
            print(f"Orders: {resp.list_orders.total} total, "
                  f"showing {resp.list_orders.count}")
        elif resp.HasField("error"):
            print(f"Error: {resp.error.code} — {resp.error.message}")

asyncio.run(connect())
```

**Example with Node.js (`ws` library):**

```javascript
const WebSocket = require('ws');
const { capri } = require('./proto/capri/v1/capri_pb');

const TOKEN = 'si_sb_aBcDeFgHiJkLmNoPqRsTuVwXyZ...';

const ws = new WebSocket('wss://trading.stockintel.com/ws/v1', 'capri.v1', {
  headers: {
    Authorization: `Bearer ${TOKEN}`,
  },
});

ws.on('open', () => {
  // Welcome frame arrives as the first binary message
});

ws.on('message', (data) => {
  const frame = capri.v1.ServerFrame.deserializeBinary(new Uint8Array(data));

  if (frame.hasWelcome()) {
    const w = frame.getWelcome();
    console.log(`Connected to ${w.getEnvironment()} environment`);
    w.getAccountsList().forEach((acct) => {
      console.log(`  Account: ${acct.getBrokerCode()}/${acct.getClientCode()} ` +
        `(${acct.getStatus() === capri.v1.AccountStatus.ACCOUNT_STATUS_ACTIVE ? 'active' : 'inactive'})`);
    });
  } else if (frame.hasListOrders()) {
    console.log(`Orders: ${frame.getListOrders().getTotal()} total`);
  } else if (frame.hasError()) {
    console.log(`Error: ${frame.getError().getCode()} — ${frame.getError().getMessage()}`);
  }
});
```

---

## 5. What Happens After Connect

Immediately after the WebSocket upgrade succeeds, the server pushes a **`Welcome` frame** telling you your environment (`"sandbox"` or `"live"`), the server's heartbeat interval, and your linked trading accounts.

**If OTP is required:**

- `Welcome.otp_required` is `true`, `Welcome.accounts` is empty and `Welcome.quotas` is absent.
- **Live token:** if `Welcome.has_email` is `true`, the server emailed you a 5-digit code — send it back with `SubmitOtp` before any trading command. If it is `false`, no email is configured for your account: add one in your **StockIntel settings**, then reconnect.
- **Sandbox token:** the code is always **`54321`**. Nothing is emailed. Sandbox runs the same gate as live so you only ever write one connect flow; the code is published, not secret.
- All trading/account commands return `ERROR_CODE_OTP_REQUIRED` until OTP succeeds.
- After successful `SubmitOtp`, the server replies with `SubmitOtpResponse` carrying your unlocked accounts and your quotas.

`Welcome.otp_message` always says which of these applies in plain language — display it to your user rather than writing your own copy.

Once the session is unlocked (or OTP was not required):

1. **Accounts are available** — listed in `Welcome.accounts` or `SubmitOtpResponse.accounts`.
2. **Your plan's limits are available** — in `Welcome.quotas` or `SubmitOtpResponse.quotas`: how many orders a day you may place, how many other commands a minute, which order types your plan includes, how many symbols you may subscribe to at once, and how far back your history reaches. See [API Reference — Plan Quotas](./api-reference.md#plan-quotas).
3. **Real-time data begins** — `ExecutionEvent` and `TradingSessionStatus` frames start arriving automatically. No subscribe step required.

You can now send commands: place orders, cancel orders, fetch account balances, query order history, or check market session status. See the [API Reference](./api-reference.md) for the full operation catalog.

### Market data is opt-in

Quotes and order books are the one thing that does **not** start automatically. They are per-symbol and high-volume, so you name the symbols you want:

```python
cmd = capri_pb2.ClientFrame()
cmd.request_id = str(uuid.uuid4())
ref = cmd.subscribe_quotes.symbols.add()
ref.market = capri_pb2.MARKET_REG
ref.symbol = "PSO"
await ws.send(cmd.SerializeToString())
```

The server answers with a `SubscriptionResponse` and pushes the latest snapshot straight away, so you see data without waiting for the market to move. Check its `rejected` list — symbols past your plan's cap are refused individually while the rest take effect.

Historical candles are a plain request/response: send `GetHistoricalRequest` with a market, symbol, interval, and both ends of the range. See [API Reference — Market Data](./api-reference.md#market-data) and [Historical Data](./api-reference.md#historical-data).

---

## 6. Sandbox Testing

> **Sandbox account:** When you connect with a sandbox token, your session includes a pre-configured synthetic trading account:
> - `broker_code`: `"sandbox"`
> - `client_code`: `"CS01"`
> - PIN: `"1234"`
>
> No account setup is needed — use these values directly in `PlaceOrder`, `CancelOrder`, and any other account-scoped commands.

The sandbox broker provides **deterministic order outcomes** based on the `quantity` you submit:

| Quantity | Outcome |
|---|---|
| `5000` | Order is **rejected** (`ORDER_STATUS_ERROR`) |
| `1000` | Order goes `PARTIAL` (50%) then `FILLED` |
| `10` | Order is immediately **cancelled** by the broker |
| `1` | Order is **rejected** (`ORDER_STATUS_REJECTED`) |
| `50` | Order stays `QUEUED` until you cancel it manually |
| Any other value | Order fills immediately (`SUBMITTED` → `RECEIVED` → `QUEUED` → `FILLED`) |

Use these scenarios to test your order lifecycle handling, error recovery, and cancellation logic before going live.

---

## 7. Connection Lifecycle

**Token within its OTP window (verified in the last 7 days):**

```
  Client                         StockIntel API
    │                                    │
    │── WSS Upgrade + Auth ─────────────▶│
    │◀──────── 101 + Welcome ────────────│  (otp_required:false, accounts, quotas;
    │                                    │   real-time pushes start)
    │                                    │
    │── ClientFrame { PlaceOrder } ─────▶│
    │◀── ServerFrame { PlaceOrder {} } ──│  (immediate empty ack)
    │◀── ServerFrame { ExecutionEvent } ─│  (status: SUBMITTED → QUEUED → FILLED)
    │                                    │
    │── ClientFrame { GetAccount } ─────▶│
    │◀── ServerFrame { GetAccount } ─────│
    │                                    │
    │── close ──────────────────────────▶│ (or keepalive ping/pong)
```

**Token outside its OTP window (first connect, or 7 days elapsed) — either environment:**

```
  Client                         StockIntel API
    │                                    │
    │── WSS Upgrade + Auth ─────────────▶│
    │◀──────── 101 + Welcome ────────────│  (otp_required:true, accounts:[], otp_message)
    │                                    │
    │   [live: user checks email]        │
    │   [sandbox: the code is 54321]     │
    │── ClientFrame { SubmitOtp } ──────▶│
    │◀── ServerFrame { SubmitOtpResponse }│  (accounts and quotas now populated)
    │                                    │   (real-time pushes start after unlock)
    │── ClientFrame { PlaceOrder } ─────▶│
    │◀── ServerFrame { PlaceOrder {} } ──│  (immediate empty ack)
    │◀── ServerFrame { ExecutionEvent } ─│  (status: SUBMITTED → QUEUED → FILLED)
    │                                    │
    │── close ──────────────────────────▶│
```

- **Keepalive:** The server sends native WebSocket pings every 30 seconds. Your WebSocket library should respond with pongs automatically.
- **Disconnect:** If the socket drops, reconnect with exponential backoff and resync state via `ListOrders`.
- **Supersede:** If a new connection opens with the same token, the old one is closed with close code `4002` (`SESSION_SUPERSEDED`). Do not auto-reconnect on `4002` — fix the duplicate connection instead.
- **Close codes:** The server uses application close codes (4000–4999) to tell you *why* the socket was closed. See [API Reference — Close Codes](./api-reference.md#websocket-close-codes).

---

## Next Steps

→ [API Reference](./api-reference.md) — every command, server push, error code, and validation rule.  
→ [Protobuf Guide](./protobuf.md) — deep dive on protobuf compilation and client patterns.  
→ [capri.proto](./capri.proto) — the authoritative schema file.
