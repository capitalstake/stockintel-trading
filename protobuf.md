# Protobuf Guide

Every message on the StockIntel Trading API is a **Protocol Buffers** (proto3) binary frame. This guide covers compiling the schema and building minimal clients in Python and JavaScript. The authoritative `.proto` file is available at [capri.proto](./capri.proto).

---

## Schema Overview

The API defines two top-level envelope messages:

- **`ClientFrame`** — every message your client sends (commands: place/cancel orders, list orders, get account, get session status).
- **`ServerFrame`** — every message the server sends back (command responses, execution events, session status updates, errors, and the initial Welcome).

All frames are **binary** WebSocket messages (opcode 0x2). Text frames are rejected. Each frame contains exactly one serialized `ClientFrame` or `ServerFrame`.

The schema also defines all request/response messages, enums (market, order type, order side, time-in-force, order status, session status, error codes), and the `google.protobuf.Timestamp` import for timestamps.

---

## Download & Prerequisites

- **Schema file:** [capri.proto](./capri.proto)
- **Requires:** `google/protobuf/timestamp.proto` (bundled with the protobuf compiler; no extra download needed)
- **Compiler:** `protoc` — [install via package manager or from GitHub](https://github.com/protocolbuffers/protobuf/releases)

```bash
# macOS
brew install protobuf

# Ubuntu/Debian
sudo apt install protobuf-compiler

# Verify
protoc --version
```

---

## Python

### Install

```bash
pip install protobuf
```

### Compile

```bash
# From the directory containing capri.proto:
protoc --proto_path=. --python_out=. capri.proto
```

This generates `capri/v1/capri_pb2.py`. Import it:

```python
from capri.v1 import capri_pb2
```

### Minimal Client

```python
import asyncio
import uuid
import websockets
from capri.v1 import capri_pb2

TOKEN = "si_sb_YOUR_SANDBOX_TOKEN"
URL = "wss://trading.stockintel.com/ws/v1"


class StockIntelClient:
    def __init__(self, token):
        self.token = token
        self.ws = None

    async def connect(self):
        self.ws = await websockets.connect(
            URL,
            extra_headers={
                "Authorization": f"Bearer {self.token}",
                "Sec-WebSocket-Protocol": "capri.v1",
            },
            ping_interval=None,
        )
        # Read Welcome
        welcome = await self._recv_frame()
        if welcome.HasField("welcome"):
            self.env = welcome.welcome.environment
            self.accounts = list(welcome.welcome.accounts)
            print(f"Connected to {self.env}")
        return self

    async def _recv_frame(self):
        data = await self.ws.recv()
        frame = capri_pb2.ServerFrame()
        frame.ParseFromString(data)
        return frame

    async def _send_command(self, **kwargs):
        """Build a ClientFrame with the given oneof field set."""
        frame = capri_pb2.ClientFrame()
        frame.request_id = str(uuid.uuid4())
        for field, value in kwargs.items():
            getattr(frame, field).CopyFrom(value)
        await self.ws.send(frame.SerializeToString())
        return frame.request_id

    async def place_order(self, broker, client, symbol, side, qty,
                          order_type=capri_pb2.ORDER_TYPE_MARKET,
                          price=0.0, market=capri_pb2.MARKET_REG,
                          tif=capri_pb2.TIME_IN_FORCE_DAY, pin=""):
        req = capri_pb2.PlaceOrderRequest()
        req.broker_code = broker
        req.client_code = client
        req.market = market
        req.symbol = symbol
        req.type = order_type
        req.side = side
        req.quantity = qty
        req.price = price
        req.time_in_force = tif
        req.pin = pin
        return await self._send_command(place_order=req)

    async def get_account(self, broker, client):
        req = capri_pb2.GetAccountRequest()
        req.broker_code = broker
        req.client_code = client
        return await self._send_command(get_account=req)

    async def list_orders(self, broker, client, count=25, offset=0):
        req = capri_pb2.ListOrdersRequest()
        req.broker_code = broker
        req.client_code = client
        req.count = count
        req.offset = offset
        return await self._send_command(list_orders=req)

    async def cancel_order(self, broker, client,
                           broker_order_id="", exchange_order_id="", pin=""):
        req = capri_pb2.CancelOrderRequest()
        req.broker_code = broker
        req.client_code = client
        req.broker_order_id = broker_order_id
        req.exchange_order_id = exchange_order_id
        req.pin = pin
        return await self._send_command(cancel_order=req)

    async def listen(self):
        """Process incoming frames indefinitely."""
        while True:
            frame = await self._recv_frame()
            if frame.HasField("execution_event"):
                ex = frame.execution_event.execution
                status_name = capri_pb2.OrderStatus.Name(ex.status)
                print(f"[{frame.request_id}] {ex.symbol} {status_name} "
                      f"qty={ex.quantity} fill={ex.last_price}@{ex.last_quantity}")
            elif frame.HasField("session_status"):
                ss = frame.session_status
                print(f"Session: {ss.broker_code}/{capri_pb2.Market.Name(ss.market)} "
                      f"→ {capri_pb2.SessionStatus.Name(ss.status)}")
            elif frame.HasField("error"):
                e = frame.error
                print(f"Error [{frame.request_id}]: "
                      f"{capri_pb2.ErrorCode.Name(e.code)} — {e.message}")
            elif frame.HasField("get_account"):
                acc = frame.get_account
                print(f"Account {acc.broker_code}/{acc.client_code}: "
                      f"balance={acc.balance} bp={acc.buying_power} "
                      f"positions={len(acc.positions)}")
            elif frame.HasField("list_orders"):
                lo = frame.list_orders
                print(f"Orders: {lo.total} total, showing {lo.count}")
            # handle other response types as needed


async def main():
    client = await StockIntelClient(TOKEN).connect()

    # Place an order
    req_id = await client.place_order(
        broker="sandbox", client="CS01", symbol="AAPL",
        side=capri_pb2.ORDER_SIDE_BUY, qty=100,
        pin="1234"
    )
    print(f"Placed order with request_id={req_id}")

    # Listen for execution events + other pushes
    await client.listen()


asyncio.run(main())
```

### Python Tips

- Use `frame.HasField("field_name")` to check which `oneof` variant is set — checking `frame.field_name` directly on a missing field returns a default (empty) message, which is indistinguishable from a real empty response like `PlaceOrderResponse {}`.
- `request_id` must be a non-empty string, unique per command, and never reused on the same connection. Use `uuid.uuid4()`.
- Always read the `Welcome` frame first after connecting — it arrives immediately and is not optional.

---

## JavaScript / TypeScript

### Install

```bash
npm install google-protobuf
npm install ws              # WebSocket client
npm install uuid            # UUID generation
```

### Compile

```bash
# CommonJS output:
protoc --proto_path=. --js_out=import_style=commonjs,binary:. capri.proto

# Or for TypeScript with grpc-web/protobuf-ts (alternative approach):
npm install @protobuf-ts/plugin
protoc --proto_path=. --ts_out=. capri.proto
```

This generates `capri/v1/capri_pb.js`. Import it:

```javascript
const { capri } = require('./capri/v1/capri_pb');
```

### Minimal Client

```javascript
const WebSocket = require('ws');
const { v4: uuidv4 } = require('uuid');
const { capri } = require('./capri/v1/capri_pb');

const TOKEN = 'si_sb_YOUR_SANDBOX_TOKEN';
const URL = 'wss://trading.stockintel.com/ws/v1';

class StockIntelClient {
  constructor(token) {
    this.token = token;
    this.pending = new Map(); // request_id → { resolve, reject }
  }

  connect() {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(URL, 'capri.v1', {
        headers: { Authorization: `Bearer ${this.token}` },
      });
      this.ws.binaryType = 'arraybuffer';

      this.ws.on('open', () => {
        // Welcome arrives as first message — handled in onmessage
      });

      this.ws.on('message', (data) => {
        const frame = capri.v1.ServerFrame.deserializeBinary(new Uint8Array(data));
        this._handleFrame(frame);
      });

      this.ws.on('error', (err) => reject(err));

      // Resolve once we get the Welcome
      this._onWelcome = resolve;
    });
  }

  _handleFrame(frame) {
    const rid = frame.getRequestId();

    if (frame.hasWelcome()) {
      this.env = frame.getWelcome().getEnvironment();
      this.accounts = frame.getWelcome().getAccountsList();
      console.log(`Connected to ${this.env}`);
      if (this._onWelcome) { this._onWelcome(this); this._onWelcome = null; }
      return;
    }

    if (frame.hasError()) {
      const err = frame.getError();
      console.log(`Error [${rid}]: ${err.getCode()} — ${err.getMessage()}`);
      return;
    }

    if (frame.hasExecutionEvent()) {
      const ex = frame.getExecutionEvent().getExecution();
      const status = Object.entries(capri.v1.OrderStatus)
        .find(([, v]) => v === ex.getStatus())?.[0] || 'UNKNOWN';
      console.log(`[${rid}] ${ex.getSymbol()} ${status} ` +
        `qty=${ex.getQuantity()} fill=${ex.getLastPrice()}@${ex.getLastQuantity()}`);
      return;
    }

    if (frame.hasGetAccount()) {
      const acc = frame.getGetAccount();
      console.log(`Account: balance=${acc.getBalance()} ` +
        `bp=${acc.getBuyingPower()} positions=${acc.getPositionsList().length}`);
      return;
    }

    if (frame.hasListOrders()) {
      const lo = frame.getListOrders();
      console.log(`Orders: ${lo.getTotal()} total, showing ${lo.getCount()}`);
      return;
    }

    // Other response types handled here
  }

  _send(frame) {
    this.ws.send(frame.serializeBinary());
  }

  placeOrder({ broker, client, symbol, side, qty, type, price, market, tif, pin }) {
    const frame = new capri.v1.ClientFrame();
    frame.setRequestId(uuidv4());

    const req = new capri.v1.PlaceOrderRequest();
    req.setBrokerCode(broker);
    req.setClientCode(client);
    req.setMarket(market || capri.v1.Market.MARKET_REG);
    req.setSymbol(symbol);
    req.setType(type || capri.v1.OrderType.ORDER_TYPE_MARKET);
    req.setSide(side);
    req.setQuantity(qty);
    if (price != null) req.setPrice(price);
    req.setTimeInForce(tif || capri.v1.TimeInForce.TIME_IN_FORCE_DAY);
    if (pin) req.setPin(pin);

    frame.setPlaceOrder(req);
    this._send(frame);
    return frame.getRequestId();
  }

  getAccount(broker, client) {
    const frame = new capri.v1.ClientFrame();
    frame.setRequestId(uuidv4());

    const req = new capri.v1.GetAccountRequest();
    req.setBrokerCode(broker);
    req.setClientCode(client);

    frame.setGetAccount(req);
    this._send(frame);
  }

  listOrders(broker, client, count = 25, offset = 0) {
    const frame = new capri.v1.ClientFrame();
    frame.setRequestId(uuidv4());

    const req = new capri.v1.ListOrdersRequest();
    req.setBrokerCode(broker);
    req.setClientCode(client);
    req.setCount(count);
    req.setOffset(offset);

    frame.setListOrders(req);
    this._send(frame);
  }

  cancelOrder(broker, client, brokerOrderId, exchangeOrderId, pin) {
    const frame = new capri.v1.ClientFrame();
    frame.setRequestId(uuidv4());

    const req = new capri.v1.CancelOrderRequest();
    req.setBrokerCode(broker);
    req.setClientCode(client);
    if (brokerOrderId) req.setBrokerOrderId(brokerOrderId);
    if (exchangeOrderId) req.setExchangeOrderId(exchangeOrderId);
    if (pin) req.setPin(pin);

    frame.setCancelOrder(req);
    this._send(frame);
  }
}

// Usage
async function main() {
  const client = await new StockIntelClient(TOKEN).connect();

  client.placeOrder({
    broker: 'sandbox',
    client: 'CS01',
    symbol: 'AAPL',
    side: capri.v1.OrderSide.ORDER_SIDE_BUY,
    qty: 100,
    pin: '1234',
  });
}

main().catch(console.error);
```

### JavaScript Tips

- Set `ws.binaryType = 'arraybuffer'` before connecting — the default is `'nodebuffer'` (Buffer), but `deserializeBinary` expects `Uint8Array`.
- Use `frame.hasFieldName()` methods to check which `oneof` variant is set. On a missing field, the getter returns a default (empty) message — indistinguishable from a real empty `PlaceOrderResponse {}`.
- The `google-protobuf` package uses getter/setter methods, not direct property access: `frame.getRequestId()`, `req.setSymbol(...)`.
- For TypeScript, consider [`@protobuf-ts`](https://github.com/timostamm/protobuf-ts) — it generates idiomatic TypeScript interfaces with better ergonomics than `google-protobuf`.

---

## Other Languages

The `.proto` file is standard proto3 and compiles with `protoc` for any supported language:

| Language | Command |
|---|---|
| **Go** | `protoc --proto_path=. --go_out=. capri.proto` |
| **Java** | `protoc --proto_path=. --java_out=. capri.proto` |
| **C++** | `protoc --proto_path=. --cpp_out=. capri.proto` |
| **C#** | `protoc --proto_path=. --csharp_out=. capri.proto` |
| **Rust** | Use [`prost`](https://github.com/tokio-rs/prost) with `prost-build` or `protoc --proto_path=. --rust_out=.` via `protoc-gen-rust` |
| **Ruby** | `protoc --proto_path=. --ruby_out=. capri.proto` |
| **Kotlin** | `protoc --proto_path=. --kotlin_out=. capri.proto` |

---

## Common Pitfalls

1. **Text frames** — The API rejects text WebSocket frames (opcode 0x1). Always send **binary** frames (opcode 0x2). All protobuf serialization libraries produce binary output by default, so this is normally automatic — but ensure your WebSocket library's `send()` uses binary mode.

2. **`oneof` field detection** — Never check a `oneof` field by comparing to a default/empty message. Both Python (`HasField`) and JavaScript (`hasPlaceOrder()`) provide explicit presence checks. Use them.

3. **`request_id` reuse** — Every command needs a fresh UUID. Reusing a `request_id` on the same connection is a protocol error and the server closes the socket with code `4000`. Never recycle IDs.

4. **Duplicate reads** — Sending a read command identical to one already awaiting its result (same `oneof` type with same field values, excluding `request_id`) is rejected with `RATE_LIMITED` (`reason = "duplicate_in_flight"`). Wait for the first result before retrying.

5. **Welcome frame** — The first message after a successful connect is always a `ServerFrame` with the `welcome` payload. Your client must read it before sending commands. It arrives immediately; you won't miss it.

6. **Execution ordering** — An `ExecutionEvent` for your order may arrive **before** the `PlaceOrderResponse` acknowledgement. Always be ready to receive executions tagged with a `request_id` you just sent, even before the ack.

7. **Keepalive** — The server sends native WebSocket pings. Your library should handle pong responses automatically. Do not implement application-level ping/pong — the `Ping`/`Pong` messages in the proto are reserved and not used.
