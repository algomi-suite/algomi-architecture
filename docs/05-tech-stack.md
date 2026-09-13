# Tech Stack

| Layer | Choice |
|---|---|
| Frontend | Next.js / React, TypeScript, shared component package for cross-product branding |
| Backend API | Python (FastAPI-style REST service) |
| Background processing | Python worker service — Telethon (Telegram client), broker SDKs, scheduled jobs |
| Database | PostgreSQL (Cloud SQL), SQLAlchemy models, hand-applied SQL migrations |
| Charting / scripting | Custom Pine-subset language interpreter (Python), lightweight-charts on the frontend |
| AI / LLM | Multi-provider LLM gateway (NIM, hosted + external providers) with a DB-backed routing registry, used for resume tailoring, job matching, and trade-related copy |
| Messaging / signals | Telegram (Telethon client) for trade signal ingestion and user-facing bot interactions; TradingView webhooks as an alternate signal source |
| Broker integrations | Adapter-per-broker layer across multiple Indian brokers (order placement, auth, account data, WebSocket order/trade updates) |
| Infra | Google Cloud Run, Cloud Build (per-repo CI/CD), Cloud SQL, Secret Manager, Artifact Registry, Direct VPC egress |
| Scheduling | Cloud Scheduler-driven jobs plus an in-app job scheduler for user-configurable recurring tasks |

## Why these choices

- **Python across API and worker** — one language for the shared domain
  package means broker adapters, order state, and signal parsing are
  written once and imported by both services instead of ported between two
  runtimes.
- **A real (subset) Pine interpreter instead of a fixed indicator list** —
  lets users write and backtest their own strategy logic instead of
  choosing from a small preset menu, at the cost of building and
  maintaining a small language interpreter.
- **Provider-agnostic LLM gateway** — AI features (resume rewriting, trade
  copy, job matching) aren't hardcoded to one vendor's SDK; provider,
  model, and routing rules are admin-configurable data, which also made it
  straightforward to add a structural rule that personal data only ever
  reaches self-hosted models.
