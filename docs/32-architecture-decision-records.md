# 32 — Architecture Decision Records (ADRs) and Open Decision Points

| Field | Value |
|---|---|
| Status | Living document |
| Upstream | All architecture documents |
| Downstream | All phases. Implementation deviations are recorded here. |
| Used by phases | All |

Format: **Context → Decision → Alternatives → Consequences**. Status: `Accepted`, `Proposed` (needs operator/phase confirmation), `Superseded`.

To record a deviation during implementation, add a new ADR (next number), reference it from the affected docs, and set the old ADR to `Superseded by ADR-XXXX` where applicable.

---

## ADR-0001 Modular monolith
**Status:** Accepted
**Context:** One operator, tens of events per second, strict latency and consistency needs, small team (an AI agent).
**Decision:** A single engine process with strictly bounded internal modules ([04](04-technical-spec.md) §3, §6).
**Alternatives:** Microservices (rejected: network hops, distributed state, ops burden). Serverless functions (rejected: can't hold WebSockets).
**Consequences:** Simple deploy and in-memory consistency. Module boundaries enforced by tooling allow later extraction.

## ADR-0002 TypeScript end-to-end
**Status:** Accepted
**Context:** Shared types between engine, API contract and dashboard. A single toolchain for an AI coding agent.
**Decision:** TypeScript (strict) on Node.js LTS for the engine. Next.js for the dashboard. Python only for optional offline research, never imported by the runtime.
**Alternatives:** Python engine (pandas convenience; rejected due to duplicated schemas). Rust (performance not required at MVP scale).
**Consequences:** One language and zod as the single schema source. CPU-heavy analytics go in worker threads if needed.

## ADR-0003 PostgreSQL with native partitioning, no TimescaleDB dependency
**Status:** Accepted
**Context:** High-frequency event data. The prompt suggests Supabase, where TimescaleDB extension availability is not guaranteed across Postgres versions.
**Decision:** Plain PostgreSQL with declarative daily range partitioning, app-managed partitions, and minimal indexes.
**Alternatives:** TimescaleDB (portability risk). ClickHouse/QuestDB (extra system).
**Consequences:** Portable across Supabase/self-hosted. We implement partition maintenance and retention ourselves ([07](07-database-schema.md) §6–7).

## ADR-0004 In-process event bus with Postgres event log
**Status:** Accepted
**Context:** An event-driven design is needed, but the volume is modest and there is a single process.
**Decision:** A typed in-process bus with bounded per-subject queues. Normalized events are persisted asynchronously to `market.events` with a flush barrier before decisions.
**Alternatives:** Kafka/Redpanda/NATS/Redis Streams (rejected as unnecessary for the MVP).
**Consequences:** Low latency and simple operations. Crash may lose unflushed non-decision events (bounded, recorded). The persistence backlog policy protects decisions.

## ADR-0005 Engine hosted on a persistent container host, not Vercel
**Status:** Accepted
**Context:** The engine holds long-lived provider WebSockets and timers. Vercel functions are request-scoped.
**Decision:** The engine runs as a Docker container on an always-on host. Vercel hosts only the dashboard.
**Consequences:** A small monthly host cost. Clear separation of the UI and the real-time core.

## ADR-0006 SOL-native cash accounting with USD reporting
**Status:** Accepted
**Context:** Memecoin trades on the supported venues are SOL-quoted. A USD bankroll is required for reporting ($20).
**Decision:** Fund the paper account in SOL at the recorded SOL/USD at run start. Report in SOL and USD, separating trading PnL from SOL revaluation.
**Alternatives:** USD/USDC cash (unrealistic conversions per trade; would hide SOL exposure).
**Consequences:** USD equity moves with SOL. Decomposition is shown to avoid confusion ([18](18-portfolio-accounting-spec.md) §5.1).

## ADR-0007 AI is advisory and veto-only
**Status:** Accepted
**Context:** LLMs are probabilistic and vulnerable to prompt injection via token metadata.
**Decision:** AI outputs are schema-validated, stored separately, and can at most veto an entry (VETO mode). Never size, execute or loosen risk.
**Consequences:** Deterministic, auditable core. AI value is measured by comparing ADVISORY runs with vetoes counterfactually (the post-trade analysis shows what vetoes would have done).

## ADR-0008 Replay ordering defaults to AS_RECEIVED
**Status:** Accepted
**Context:** Reproducing live decisions requires the arrival order and timing the system experienced.
**Decision:** Default replay ordering is `(session_id, ingest_seq)` with simulated time = `received_at`. `BY_EVENT_TIME` is offered as an idealized comparison.
**Consequences:** Live-vs-replay equivalence is testable. Latency cost can be quantified by comparing the two orderings.

## ADR-0009 Wallet activity source
**Status:** **Proposed**. Confirm in Phase 03 (verification spike) and Phase 05.
**Context:** Detecting tracked-wallet swaps with low latency on the Helius Free plan. Enhanced WebSockets (`transactionSubscribe` with account filters) appear to require a paid tier. The number of standard WS connections is limited (5 on Free, per search snippets). Subscriptions-per-connection is unknown.
**Decision (proposed):** Primary `helius_ws`: one `logsSubscribe({mentions:[wallet]})` per tracked wallet, packed across connections. Decode venue events directly from logs where possible (pump.fun/PumpSwap). Otherwise enrich via `getTransaction` and the balance-delta parser. Alternative `helius_webhook`: Helius enhanced webhooks for all tracked wallets (parsed swaps, 1 credit/event) delivered to `/api/v1/webhooks/helius`.
**Alternatives:** Enhanced WS / LaserStream (paid; best latency). Polling `getSignaturesForAddress` (too slow and expensive).
**Decision criteria for Phase 03/05:** measured detection latency (p50/p95 in slots and ms), credits per detected swap, capacity (wallets per plan), and reliability. Record the outcome here and set status to Accepted.
**Consequences:** Both sources emit identical `wallet.swap.detected` events (the same dedup key space), so switching is a config change (`ingestion.wallet_source`).

## ADR-0010 CP-only pool economics in MVP
**Status:** Accepted
**Context:** Accurate fill simulation requires exact pool math. CLMM/DLMM require tick/bin data and complex math.
**Decision:** Support pump.fun bonding curve, PumpSwap, Raydium AMM v4 and Raydium CPMM (after verification). Reject tokens without a supported pool (`UNSUPPORTED_POOL_TYPE`) rather than approximating.
**Consequences:** Some tokens are skipped. Every fill is modelled exactly. Adding CLMM later means a new decoder + economics module.

## ADR-0011 Dashboard realtime via direct engine WebSocket
**Status:** Accepted
**Context:** Vercel functions can't reliably proxy long-lived WebSockets. The browser must not hold the engine API token.
**Decision:** The dashboard server mints a short-lived (≤ 5 min) WS token via the engine. The browser connects directly to the engine WS.
**Alternatives:** Supabase Realtime (couples UI to DB tables and bypasses the engine read models). SSE through Vercel (timeouts).
**Consequences:** The engine must handle CORS/WS auth. Tokens are rotated via reconnect.

## ADR-0012 Single-operator auth for MVP
**Status:** Accepted
**Context:** One user. Auth complexity should be minimal but real.
**Decision:** Password (argon2id hash in env) + encrypted HTTP-only session cookie on the dashboard. The service token is used server-to-server.
**Alternatives:** Supabase Auth / OAuth (upgrade path).
**Consequences:** No users table in the MVP. Multi-user is future work.

## ADR-0013 Double-entry ledger
**Status:** Accepted
**Context:** Accounting correctness must be provable, with fees, rent and failures.
**Decision:** A per-asset balanced journal with a trading (EXCHANGE) clearing account. Positions and snapshots are derived.
**Consequences:** Invariants are checkable after every transaction. Slightly more code than naive balance updates.

## ADR-0014 Integer base units and decimal math
**Status:** Accepted
**Decision:** `bigint` for lamports and raw token amounts. `decimal.js` for prices/ratios. `numeric` in Postgres. Money is serialized as strings in the API. Floats are banned for money.
**Consequences:** Exact reconciliation. Careful conversion utilities in `@paperbot/core`.

## ADR-0015 Commitment level `confirmed`
**Status:** Accepted
**Decision:** Use `confirmed` for WS subscriptions and RPC reads.
**Alternatives:** `processed` (fastest, rollback risk). `finalized` (+10 s or more latency).
**Consequences:** Rare rollbacks are accepted. An optional nightly `finalized` reconciliation checks decision-trigger signatures.

## ADR-0016 Time model: epoch microseconds and an injected Clock
**Status:** Accepted
**Decision:** All timestamps are `EpochMicros` from an injected `Clock` (System or Simulated), and durations come from the monotonic clock. No ambient `Date.now()` in domain code.
**Consequences:** Deterministic replay and tests. Requires discipline (lint rule).

## ADR-0017 Custom Postgres schemas
**Status:** Accepted
**Decision:** Use the `ref`, `market`, `intel`, `strategy`, `sim` and `ops` schemas. Nothing goes in `public`. On Supabase these schemas are not API-exposed, and RLS is deny-all as a backstop.
**Consequences:** Clear ownership and no accidental PostgREST exposure.

## ADR-0018 Database hosting
**Status:** **Proposed**. Operator decides in Phase 24.
**Context:** The event log is estimated at ~0.2 GB/day, far beyond the Supabase Free storage quota. Engine↔DB latency matters for write-ahead.
**Options:** (A) Self-hosted Postgres on the engine VPS (recommended: lowest latency and cost, you own backups). (B) Supabase Pro in the same region as the engine. (C) Supabase Free for development only.
**Consequences:** The schema is identical in all cases. The decision only affects ops and backups.

## ADR-0019 Document numbering
**Status:** Accepted
**Context:** The master prompt lists `27-data-provider-reference.md` and `28-decision-logging-spec.md`, but its §27 says provider assumptions go in `28-data-provider-reference.md`.
**Decision:** The provider reference is **27**, and decision logging is **28**. Also added: `32-architecture-decision-records.md` (this file) and `33-documentation-audit.md`.

## ADR-0020 Kysely and SQL-first migrations
**Status:** Accepted
**Decision:** `node-pg-migrate` with plain SQL files. Kysely for typed queries, with types generated from the DB.
**Alternatives:** Prisma (weak partitioning/COPY support). Drizzle (acceptable, not chosen).
**Consequences:** Migrations are reviewable SQL. The codegen step runs after migrations.

## ADR-0021 Fastify for the engine API
**Status:** Accepted
**Decision:** Fastify + `@fastify/websocket`, `@fastify/rate-limit`, `@fastify/helmet`, `@fastify/cors`.
**Consequences:** Good performance and plugin ecosystem. Schemas from zod via a type provider.

## ADR-0022 No partial fills for atomic AMM swaps
**Status:** Accepted
**Context:** The brief asks for partial fills "where appropriate". A Solana AMM swap is atomic.
**Decision:** Single orders fill fully or fail. Partial position exits are separate orders (scale-out). The curve clamp edge case follows verified program behaviour.
**Consequences:** Realistic semantics. The order model keeps `sim.fills` 1..n for future venues (orderbooks) without schema change.

---

## Open decision points register

| ID | Decision | Owner | Needed by | Default if undecided |
|---|---|---|---|---|
| DP-01 | Wallet activity source (ADR-0009) | Phase 03/05 agent, operator confirms | Phase 05 | `helius_ws` |
| DP-02 | DB hosting (ADR-0018) | Operator | Phase 24 | Self-hosted on VPS |
| DP-03 | Engine host provider | Operator | Phase 24 | Small VPS + Docker Compose + Caddy |
| DP-04 | Venue support list after verification | Phase 03/06/07 agent | Phase 07 | Only venues whose layouts, fees and events are verified are `supported=true` |
| DP-05 | Seed wallet list source | Operator | Phase 09 (before validation) | Operator-provided CSV. The system does not ship a wallet list. |
| DP-06 | AI mode for validation | Operator | Phase 25 | `ADVISORY` (AI has no effect) |
| DP-07 | Helius plan upgrade | Operator | After Phase 25 metrics | Stay on Free unless conserve mode > 20% of the time |
| DP-08 | Cold archive storage target | Operator | Phase 24 | Disabled (30-day replay horizon) |
