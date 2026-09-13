# Design Decisions & Trade-offs

Notable decisions made while building Algomi, and the reasoning behind them.

## Host-aware routing instead of separate apps

**Decision:** `trade.algomi.in` and `invest.algomi.in` are the same Next.js
deployment; middleware reads the request host and renders the matching
product experience.

**Trade-off:** Saves maintaining two nearly-identical frontends and two
deploy pipelines. Costs some indirection — a change intended for one
surface has to be checked against the other, since they share a
build and a runtime.

## Single shared database across products

**Decision:** One Postgres instance serves Trading, Invest, Career, Charts,
and Admin.

**Trade-off:** Simpler operationally (one connection story, one migration
history, cross-product queries are plain joins) but no hard blast-radius
isolation between products — a bad migration or a runaway query in one
product's code path can affect the others. Mitigated with hand-gated,
idempotent migrations (see [Data & schema](03-data-and-schema.md)) rather
than automated schema changes.

## Broker integration as a plugin interface

**Decision:** Every broker implements the same adapter interface
(`place_entry`, `place_exit`, `get_orders`, auth, WebSocket subscription)
rather than the order/signal logic branching per broker.

**Trade-off:** Adding a sixth broker means writing one adapter, not
touching the state machine or parser. The cost shows up when a broker's
API doesn't map cleanly onto the interface (e.g. a broker with no native
OCO order support needs the adapter to simulate one) — that complexity is
absorbed in the adapter rather than leaking into shared logic.

## Real-time WebSocket risk closer alongside an interval job, not instead of it

**Decision:** Keep the scheduled interval-based risk job running even after
adding a WebSocket-driven real-time closer, with independent enable/disable
toggles for each.

**Trade-off:** Two mechanisms that can, in principle, race each other on
the same position — solved by making each user-configurable and
independent rather than trying to fully unify them into one code path.
The interval job is the fallback for exactly the failure mode a
WebSocket-based system has: dropped connections and missed messages.

## Config-driven LLM provider selection

**Decision:** Which LLM provider/model handles a given task is a database
row (task-routing precedence), read at request time, not a hardcoded SDK
call baked into the code that needs AI.

**Trade-off:** More moving parts than calling one vendor's SDK directly,
but it means adding a provider, changing a default model, or restricting
a task to self-hosted-only (for personal-data requests) is an admin-panel
change, not a code deploy.

## Worker uptime tied to market hours, not always-on

**Decision:** The background worker scales down outside an admin-configured
active window instead of running continuously.

**Trade-off:** Meaningfully lower compute cost for a workload that's only
useful while markets are open — but it means the on/off schedule itself
becomes a piece of business logic that has to be gotten right (a
misconfigured window, or a scheduler that only fires forward in time, can
mean the worker simply isn't running when a trading day starts).
