# Data & Schema Evolution

## One database, many products

All products (Trading, Invest, Career, Charts, Admin) read and write a
single Postgres instance (Cloud SQL). There's no per-product database split.
This trades some blast-radius isolation for a big simplification: one
connection/auth story, one migration history, and cross-product queries
(e.g. "does this user have both a trading account and a career profile")
without cross-database joins.

Access from services runs over private (VPC-native) connectivity rather than
a public IP — application code never handles a public database endpoint.

## Migration discipline

Schema changes are sequential, numbered SQL migrations, applied by hand
against production rather than run automatically on deploy. Every schema
change ships as:

1. A numbered migration file, written to be idempotent (safe to re-run)
2. A short diagnostic query the operator runs first to confirm the
   migration is actually needed on that environment
3. Application code that tolerates the pre-migration schema until the
   migration is confirmed applied

This is a deliberate trade-off for a small team running its own production
database: automatic migrations-on-deploy are faster but riskier without a
dedicated DBA reviewing every change; a human-gated apply step catches
mistakes before they touch live trading data.

## Example: evolving a single feature safely

The risk-monitoring subsystem is a good example of the pattern. It started
as one interval-based job (`risk`) that polled positions on a timer. Adding
a near-real-time WebSocket-based risk closer alongside it meant the two
mechanisms could otherwise race — the interval job and the socket both
trying to close the same position. The fix was a schema change adding a
second, independent toggle (`risk_monitor_enabled` separate from
`webhook_enabled`) so each mechanism's on/off state is explicit and the two
can be enabled independently per user instead of implicitly coupled.
