# Trading Signal Flow

How a trade call becomes a live broker order, end to end.

```mermaid
sequenceDiagram
    participant Src as Signal source<br/>(Telegram / TradingView webhook)
    participant Worker as worker: listener
    participant Parser as worker: signal parser
    participant State as Order state machine
    participant Broker as Broker adapter
    participant BrokerAPI as Broker API
    participant Risk as Risk monitor (WebSocket)
    participant DB as Postgres

    Src->>Worker: raw message / webhook payload
    Worker->>Parser: parse(raw)
    Parser->>Parser: extract symbol, side, entry,<br/>targets, stop-loss
    Parser->>State: create pending order intent
    State->>DB: persist order (status=pending)
    State->>Broker: place_entry(order)
    Broker->>BrokerAPI: submit order (product-specific:<br/>intraday entry / resting order)
    BrokerAPI-->>Broker: order id / ack
    Broker->>DB: update order (status=open, broker_order_id)

    par Real-time updates
        BrokerAPI-->>Risk: order/trade update (WebSocket)
        Risk->>State: on fill: place exit legs (target + stop)
        State->>Broker: place_exit(order, OCO pair)
    and Interval fallback
        Worker->>BrokerAPI: poll positions/orders (scheduled job)
        Worker->>State: reconcile any missed updates
    end

    State->>DB: close order in place on final fill/exit
```

## Design points

- **One row per trade, closed in place.** An order's lifecycle (pending →
  open → exited) updates a single database row rather than inserting a new
  row per state change, so P&L and history queries don't need to reconstruct
  state from an event log.
- **Real-time path + scheduled fallback.** A WebSocket connection reacts to
  fills immediately (to place exit legs with minimal slippage), while a
  scheduled reconciliation job catches anything the socket missed —
  dropped connections, missed messages, restarts.
- **Broker adapter boundary.** `place_entry` / `place_exit` / `get_orders`
  are the same interface regardless of broker; product-type differences
  (e.g. intraday vs. resting entry orders, OCO vs. two separate exit orders)
  are handled inside each adapter, not in the signal parser or state
  machine.
- **Segment-aware routing.** A user can route options signals to one broker
  and equity/cash signals to another; the state machine picks the adapter
  per order based on instrument segment and the user's configured routing.
