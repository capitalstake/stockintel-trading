# StockIntel Trading API

The StockIntel Trading API is a real-time, low-latency **WebSocket** interface for placing and managing orders, querying account positions, and receiving live execution and market-session updates. The protocol speaks **Protocol Buffers** (proto3) over binary WebSocket frames — compact, typed, and efficient.

Designed for trading bots, algorithmic strategies, terminal applications, and broker integrations.

---

## Quick Facts

| | |
|---|---|
| **Transport** | WebSocket over TLS (`wss://`) |
| **Public endpoint** | `wss://trading.stockintel.com/ws/v1` |
| **Wire format** | Protocol Buffers (proto3), binary frames |
| **Authentication** | Bearer token in the WebSocket upgrade `Authorization` header |
| **API style** | Command/response for reads; fire-and-acknowledge for orders; automatic server push for real-time data |
| **Rate limit** | 5 orders per second per connection |
| **Environments** | Sandbox (test) and Live (production) — separate tokens |

---

## Documentation Index

| Document | What's covered |
|---|---|
| [Getting Started](./getting-started.md) | Create tokens, open a connection, first command |
| [API Reference](./api-reference.md) | Full operation reference: commands, pushes, errors, close codes, correlation model |
| [Protobuf Guide](./protobuf.md) | Compile the `.proto` for Python, JavaScript, and other languages |
| [capri.proto](./capri.proto) | Authoritative protobuf schema (download) |

---

## How It Works

```
┌──────────────────┐                         ┌─────────────────────────┐
│  Your App / Bot  │──── WSS + Protobuf ────▶│  StockIntel Trading API │
└──────────────────┘                         └─────────────────────────┘
```

1. **Connect** — Open a WSS connection with your API token in the `Authorization` header.
2. **Welcome** — The server immediately sends your environment and linked trading accounts. For live tokens, it may require a one-time OTP email verification before accounts are released — check `Welcome.otp_required` and complete `SubmitOtp` if needed. Sandbox tokens are never gated.
3. **Real-time data flows automatically** — Execution reports and trading-session status are pushed from the moment you connect. No subscriptions needed.
4. **Send commands** — Place orders, cancel orders, list order history, fetch account balances and positions. Every command gets exactly one response, correlated by a UUID you supply.

---

## Key Design Points

- **Streaming-first** — Order lifecycle (submitted → queued → partial → filled) and market-session state are pushed to you automatically. You never poll.
- **Fire-and-acknowledge orders** — `PlaceOrder` returns an immediate empty acknowledgement. All outcomes (fills, rejections, cancels) arrive on the real-time execution stream, keyed by your own correlation ID.
- **One connection per token** — The WebSocket *is* your session. Sandbox and live are separate tokens, so you can run one of each concurrently. A token can only have one active connection at a time (newest-wins takeover).
- **Stateless server** — No server-side persistence. Reconnect and resync via the execution stream and `ListOrders`.

---

## Quick Example

```
# Open a connection (protobuf binary frames, not text)
wss://trading.stockintel.com/ws/v1
Authorization: Bearer si_sb_aBcDeFgHiJkLmNoPqRsTuVwXyZ...
Sec-WebSocket-Protocol: capri.v1

# Server pushes Welcome immediately, then you send commands as binary frames

# Place a market order (pseudo — actual frame is a serialized ClientFrame protobuf)
→ ClientFrame {
    request_id: "550e8400-e29b-41d4-a716-446655440000"
    place_order: {
      broker_code: "sandbox"
      client_code: "CS01"
      market: MARKET_REG
      symbol: "AAPL"
      type: ORDER_TYPE_MARKET
      side: ORDER_SIDE_BUY
      quantity: 100
      time_in_force: TIME_IN_FORCE_DAY
      pin: "1234"
    }
  }

# Immediate empty ack
← ServerFrame { request_id: "550e8400-..."  place_order: {} }

# Then execution events stream in automatically
← ServerFrame { request_id: "550e8400-..."  execution_event: { status: SUBMITTED ... } }
← ServerFrame { request_id: "550e8400-..."  execution_event: { status: QUEUED ... } }
← ServerFrame { request_id: "550e8400-..."  execution_event: { status: FILLED ... } }
```

---

## Reference Client

A Python terminal client is available for developers to explore and test the API interactively:

**[stockintel-trading-client](https://github.com/capitalstake/stockintel-trading-client)** — a Textual TUI that connects over WebSocket + Protobuf and exposes all API commands (place/cancel orders, list accounts, get positions, session status, OTP verification) via keyboard shortcuts. Intended as a developer testing view, not a production application.

Use it to observe real protocol frames, test sandbox order outcomes, and verify your token and account setup before building your own client.

---

## Language Support

Protocol Buffers gives you generated, type-safe clients in Python, JavaScript/TypeScript, Go, Java, C++, C#, Rust, and more. See the [Protobuf Guide](./protobuf.md) for compilation instructions and minimal client examples.

---

## Environments

| Environment | Token prefix | Purpose |
|---|---|---|
| **Sandbox** | `si_sb_` | Test strategies with deterministic order outcomes. The sandbox broker accepts any symbol; order outcomes are controlled by quantity. |
| **Live** | `si_lv_` | Production trading with real brokers and markets. |

Always develop and test against the sandbox first.

---

## Next Steps

→ [Getting Started](./getting-started.md) — generate your tokens and open your first connection.  
→ [API Reference](./api-reference.md) — the full operation and error reference.  
→ [Protobuf Guide](./protobuf.md) — compile the schema for your language.
