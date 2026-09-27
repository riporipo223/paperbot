# 05 — System Architecture

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [02](02-product-requirements.md), [03](03-system-requirements.md), [04](04-technical-spec.md) |
| Downstream | [06](06-data-architecture.md), [09](09-real-time-data-architecture.md), all domain specs, [25](25-infrastructure-deployment.md) |
| Used by phases | All |

## 1. Architectural style

- A **modular monolith**: a single long-running **engine** process holds all real-time logic ([ADR-0001](32-architecture-decision-records.md#adr-0001-modular-monolith)).
- An **in-process typed event bus** carries events between modules. **PostgreSQL** is the durable event log and system of record ([ADR-0004](32-architecture-decision-records.md#adr-0004-in-process-event-bus-with-postgres-event-log)).
- A separate **dashboard** (Next.js on Vercel) talks to the engine through its REST and WebSocket API. The dashboard has no database credentials.
- **Ports and adapters** inside each module. Pure domain logic is isolated from I/O so the same code runs in live mode, replay, and tests.

Why not microservices: one operator, tens of events per second, strong need for consistent in-memory state, and a single-machine latency budget. Distribution would add latency, failure modes and cost without a requirement that justifies them. The module boundaries ([04](04-technical-spec.md) §6) keep extraction possible later.

## 2. System context

```text
                  ┌───────────────────────────────────────────────┐
                  │                 External world                │
                  │                                               │
   Solana ───────▶│ Helius (WS + RPC [+ webhooks])                │
   mainnet        │ DexScreener REST   GeckoTerminal REST         │
                  │ CoinGecko REST (optional)   Anthropic API     │
                  └───────────────┬───────────────────────────────┘
                                  │ read-only data / LLM calls
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ ENGINE (Docker container on persistent host)                          │
│  ingestion → pipeline → market-state → wave → strategy → risk →       │
│  execution(Paper) → portfolio;  wallet-intel; ai; replay; api; obs     │
└───────────────┬──────────────────────────────────┬───────────────────┘
                │ SQL (pg)                          │ HTTPS REST + WSS
                ▼                                   ▼
        ┌───────────────┐                 ┌───────────────────────┐
        │ PostgreSQL    │                 │ DASHBOARD (Vercel)    │◀── Operator (browser)
        │ (Supabase or  │                 │ Next.js server routes │
        │  self-hosted) │                 │ + browser WS client   │
        └───────────────┘                 └───────────────────────┘
```

There are no outbound connections to any execution venue, signer, or wallet service.

## 3. Engine component view

```text
                        ┌──────────────── ingestion ────────────────┐
 Helius WS ────────────▶│ HeliusWsAdapter  (logs/account subs)      │
 Helius RPC ◀──────────▶│ HeliusRpcClient  (read-only allowlist)    │
 Helius webhook ───────▶│ HeliusWebhookAdapter (optional)           │
 DexScreener ◀─────────▶│ DexScreenerPoller                         │
 GeckoTerminal ◀───────▶│ GeckoTerminalPoller                       │
 CoinGecko ◀───────────▶│ SolUsdPoller                              │
                        └──────────────┬────────────────────────────┘
                                       │ raw provider msgs
                                       ▼
                        ┌──────────── pipeline ─────────────────────┐
                        │ Normalizers → Dedup → Sequencer           │
                        │ EventBus (typed, bounded queues)          │
                        │ EventPersister (batched, flush barrier)   │
                        │ EventSource: Live | Replay                │
                        │ Scheduler (Clock-bound timers)            │
                        └──────────────┬────────────────────────────┘
                     normalized events │
         ┌───────────────┬─────────────┼──────────────┬─────────────────┐
         ▼               ▼             ▼              ▼                 ▼
   reference       market-state   wallet-intel    observability     ai (async)
   (tokens,pools)  (pool state,   (swaps, round   (latency,         (advisory
                    quality, bars, trips, scores)  health)           outputs)
                    SOL/USD)
         └───────────────┴──────┬──────┴──────────────┘
                                ▼
                              wave  ──signal.emitted──▶ strategy (DecisionEngine, ExitManager)
                                                            │ proposed order
                                                            ▼
                                                          risk ──verdict──▶ execution (PaperExecutor)
                                                                                 │ fills (economics)
                                                                                 ▼
                                                                             portfolio (ledger)
                                api  ◀── read models / stream from all modules
```

## 4. Component responsibilities

| Module | Owns | Consumes | Produces |
|---|---|---|---|
| `pipeline` | Event bus, queues, dedup, sequencing, persistence, event sources, scheduler, clock | Raw adapter output | Normalized events on the bus. Rows in `market.events`. |
| `ingestion` | Provider connections, rate limiters, credit budget, reconnect, gap backfill | Watch-set commands (subscribe/unsubscribe) | Raw provider messages → pipeline |
| `reference` | Tokens, pools, DEX registry, primary pool selection, token safety facts | `token.discovered`, `pool.discovered`, RPC lookups | `ref.*` rows. `pool.selected`. |
| `market-state` | In-memory market state, decoders, data quality, OHLCV bars, SOL/USD | `pool.state.updated`, `pool.trade.observed`, `market.snapshot.observed`, `price.reference.updated` | `MarketStateView` (read API), `market.state.changed`, bars |
| `wallet-intel` | Wallet registry, swap history, round trips, scores, tracked set | `wallet.swap.detected`, backfill results | `intel.*` rows. `wallet.score.updated`. Tracked-set commands to ingestion. |
| `wave` | Wave state machines, features, scores | wallet swaps, market state changes, timer ticks | `wave.updated`, `signal.emitted` |
| `strategy` | Runs, decision engine, exit manager, watch-set policy | signals, market state, positions, AI advisories | `decision.made`, proposed orders, watch commands |
| `risk` | Risk checks and limits | proposed orders, portfolio, market state | `risk.evaluated` (verdict) |
| `execution` | `TradingExecutor` implementations, order lifecycle | approved orders, market state (at exec time), economics | `order.*` (incl. `order.filled` carrying the fill) |
| `economics` | Pure models (fees, impact, latency, failure, rent) | Inputs from execution | Cost breakdowns (pure functions) |
| `portfolio` | Ledger, positions, PnL, snapshots, analytics | fills, market state (marks) | `position.*`, `portfolio.updated` |
| `ai` | LLM client, agents, budgets, replay cache | subjects (wallets, tokens, signals, trades) | `ai.output.recorded` |
| `replay` | Replay orchestration | stored events | Drives `ReplayEventSource` + `SimulatedClock` |
| `observability` | Metrics, latency recorder, provider health, system events | All bus events | `ops.*` rows, `/metrics` |
| `api` | REST + WS for dashboard | Read APIs, streams | HTTP/WS responses |

## 5. Principal data flow (live SIMULATION)

1. **Detect.** Helius `logsSubscribe(mentions: walletX)` delivers a notification. `HeliusWsAdapter` stamps `received_at`.
2. **Normalize.** A decoder extracts the swap from logs (pump.fun/PumpSwap events), or `getTransaction` enrichment runs for other venues. The result is a `wallet.swap.detected` event with slot, event time and amounts.
3. **Sequence + dedup + persist.** The event gets `ingest_seq` and is published on the bus. The persister batches it into `market.events`.
4. **Wallet intel** updates the wallet's live activity. If the wallet is smart (score ≥ threshold) and the side is `BUY`, it emits `smart.buy.detected`.
5. **Reference** ensures the token and pools are resolved. The primary pool is selected, or the token is marked unsupported.
6. **Strategy watch-set policy** asks ingestion to subscribe to the pool's logs + reserve accounts, and the reference pollers to snapshot the token.
7. **Wave** seeds a wave (`SEEDED`) and recomputes features on each subsequent event and timer tick.
8. When gates pass and score ≥ threshold, **wave** emits `signal.emitted` (persisted with its feature snapshot).
9. **Strategy/DecisionEngine** evaluates the signal (+ AI advisory if `VETO` mode and available within timeout). It proposes an entry order (size from config).
10. **Risk** evaluates all checks. The verdict is persisted.
11. **Execution/PaperExecutor** persists the order (`SUBMITTED`, write-ahead), samples latency from the seeded RNG, and schedules the fill at `decided_at + latency`.
12. At execution time the executor reads the **current** pool state. If it is not `FRESH`, the order fails. Otherwise economics computes the fill (impact, fees, slippage check, failure draw).
13. **Portfolio** posts journal entries, opens or updates the position, and snapshots the portfolio.
14. **ExitManager** evaluates exit rules on every state change for held tokens and on timer ticks. Exits follow steps 9–13 with `side = SELL`.
15. **API** streams every step to the dashboard.

## 6. Mode and executor selection

```text
RunMode (config + CLI)  ──▶ ModeGuard (bootstrap)
   SIMULATION ──▶ PaperExecutor
   SHADOW     ──▶ ShadowExecutor (Phase 26; until then: bootstrap error "SHADOW not yet enabled")
   LIVE       ──▶ refused: config schema error + bootstrap error + DB CHECK violation
```

`RealExecutor` exists only as a class whose every method throws `LiveTradingDisabledError`. It is not registered in the executor factory. A test asserts that the factory cannot return it ([16](16-paper-trading-spec.md) §2, [21](21-security-spec.md) §3).

## 7. Event sources: live vs replay (critical)

```text
LiveEventSource   (adapters → normalizers)      ─┐
                                                 ├─▶ same pipeline: bus → market-state → wave → strategy → risk → execution → portfolio
ReplayEventSource (market.events ordered)       ─┘
Clock:  SystemClock (live)  |  SimulatedClock (replay: advances to each event's replay timestamp; fires due timers first)
AI:     LiveLlmClient       |  RecordedLlmClient (returns stored outputs by input hash; missing → treated as TIMEOUT)
```

Only the source, the clock and the AI client differ. Everything downstream is identical ([19](19-backtesting-spec.md)).

## 8. Concurrency model

- Node.js single event loop for all domain logic. This guarantees ordered, race-free state mutation within the engine.
- CPU-heavy or blocking work (large backfills, analytics recomputation) runs in a **worker thread** or as a separate CLI invocation, never on the hot loop.
- **Per-subject serialization**: events for the same subject (token/pool) are processed in order. Events for different subjects share the loop.
- DB writes are async. Hot-path consumers never await event persistence, except at **flush barriers** (before a decision is committed, all events up to its causation watermark must be durable; [09](09-real-time-data-architecture.md) §9).

## 9. Failure isolation

| Failure | Containment |
|---|---|
| One provider down | Its streams go `STALE`/`DEGRADED`. Dependent gates block entries. Other streams continue. |
| AI provider down | AI port returns `TIMEOUT`/`ERROR`. The deterministic fallback applies. |
| DB unavailable | Persister buffers up to `pipeline.persist.max_buffer`. New entries are paused. Exits proceed only if their write-ahead succeeds (retry loop). |
| Ledger invariant violation | Run halts (`HALTED_INVARIANT`). Alert raised. No further orders. |
| Unhandled exception in a module handler | Caught at the bus boundary, logged with the event, counted. The event goes to the dead-letter table `ops.dead_letters`. Repeated failures open a module circuit, and entries pause. |
| Engine crash | Supervisor restarts the container. On boot: rebuild portfolio from ledger, resolve in-flight orders ([16](16-paper-trading-spec.md) §9), resubscribe streams, mark gaps. |

## 10. Deployment topology (summary)

- **Engine**: one Docker container on a small always-on VM (Fly.io, Railway, Render, Hetzner or similar; decision in [25](25-infrastructure-deployment.md)), with a public HTTPS endpoint for the API (and for webhooks if used).
- **Database**: Postgres co-located (same region) for low latency.
- **Dashboard**: Vercel. Server routes call the engine with a service token and mint short-lived WS tokens for the browser ([ADR-0011](32-architecture-decision-records.md#adr-0011-dashboard-realtime-via-direct-engine-websocket)).
- Engine hosting is off Vercel because serverless functions cannot hold long-lived provider WebSockets ([ADR-0005](32-architecture-decision-records.md#adr-0005-engine-hosted-on-a-persistent-container-host-not-vercel)).

## 11. Evolution path

| Future need | Evolution |
|---|---|
| Higher event volume | Upgrade Helius tier (Enhanced WS/LaserStream). Move event log to a partitioned table on faster storage. Consider a Redis stream buffer only if measured backpressure requires it. |
| Multiple strategies concurrently | Multiple runs in one engine (runs are already first-class). Per-run portfolios. |
| Shadow mode | `ShadowExecutor` + read-only quote adapter (Phase 26). |
| Live mode | Separate, isolated signer service and a separate security review. Never in this engine process ([29](29-future-live-trading-architecture.md)). |
