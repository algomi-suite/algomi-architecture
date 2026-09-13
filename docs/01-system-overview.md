# System Overview

## Services

Algomi is split into repos/services along product and runtime boundaries,
each deployed as its own Cloud Run service:

| Service | Type | Responsibility |
|---|---|---|
| `api` | Python backend | Core REST API — auth, accounts, orders, positions, recommendations, admin config. Single source of truth for business rules. |
| `fi` | Next.js frontend | Trading + Invest web app. Host-aware: `trade.algomi.in` and `invest.algomi.in` are the same deployment, branching on request host. |
| `career` | Next.js frontend + Python backend | Career copilot: resume tooling, job aggregation, application tracking. AI-assisted (resume rewriting, job matching). |
| `charts` | Frontend + Python backend | Charting UI, a Pine-subset script interpreter, and strategy backtesting against historical OHLC/option-chain data. |
| `worker` | Python background service | Always-on and scheduled jobs: Telegram signal listener, order execution, real-time risk monitoring, cron-style scheduled tasks. No public HTTP surface. |
| `ui` | Shared package | Common header/nav/branding components, built once and vendored into each frontend so product surfaces look and behave consistently. |

## Shared domain package

`api` and `worker` both depend on one internal Python package containing:

- Database models (SQLAlchemy) and migration history
- Broker adapters (order placement, auth, account data, WebSocket order updates)
- Signal parsing (Telegram message → structured trade intent)
- Order lifecycle state machine

It's built and published as a private wheel to a package registry, then
pinned as a dependency in both services. This keeps business logic in one
place instead of duplicating it between the request/response API and the
background worker.

## Why split `api` and `worker`

The API needs to answer requests in milliseconds and scale with user
traffic. The worker holds long-lived connections (a Telegram client,
broker WebSocket sessions) and runs scheduled jobs — a different scaling and
availability profile. Separating them means:

- The worker can hold persistent connections without tying up API request
  capacity.
- The API can scale to zero between requests; the worker runs on a
  schedule/uptime window tied to market hours instead of always-on.
- A worker crash (e.g. a broker SDK exception) doesn't take the API down,
  and vice versa.

## Multi-tenancy

All products share one account/auth system. A user can hold a trading
account, an investment profile, and a career profile under the same login.
Product-specific data (broker connections, resumes, watchlists) is scoped
per user at the data layer, not by separate databases per product.
