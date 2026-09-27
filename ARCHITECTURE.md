# Architecture (summary)

This is a condensed view. The authoritative documents are [docs/05-system-architecture.md](docs/05-system-architecture.md), [docs/09-real-time-data-architecture.md](docs/09-real-time-data-architecture.md) and [docs/06-data-architecture.md](docs/06-data-architecture.md).

## 1. Style

- **Modular monolith**: one long-running engine process in TypeScript/Node.js LTS. Strict module boundaries, enforced by dependency-cruiser.
- **Event-driven**: an in-process typed event bus with per-subject ordering and bounded queues. Normalized events are persisted to a partitioned PostgreSQL event log.
- **Ports & adapters**: pure domain logic, and I/O at the edges. The same code runs live, in replay, and in tests.
- **Dashboard**: Next.js on Vercel, talking only to the engine's REST/WS API.

## 2. Data flow

```text
Provider socket/HTTP ─(received_at)─▶ normalize ─▶ dedup ─▶ sequence ─┬─▶ event log (Postgres, async batched)
                                                                      ▼
                                                   event bus (per-subject order)
      ┌────────────────┬──────────────────┬───────────────────┬──────────────────┐
      ▼                ▼                  ▼                   ▼                  ▼
  reference      market-state        wallet-intel        observability       ai (async)
  (tokens/pools) (reserves, prices,  (swaps, scores,     (latency, health)
                  quality, bars)      smart buys)
      └────────────────┴───────┬──────────┘
                               ▼
                             wave ──signal──▶ strategy.DecisionEngine ──▶ risk ──▶ PaperExecutor ──▶ portfolio ledger
                                              strategy.ExitManager ─────────────────▲
```

## 3. Key guarantees

| Guarantee | Mechanism |
|---|---|
| No real execution | No signing code (CI scan), read-only RPC allowlist, host allowlist, LIVE blocked in config + bootstrap + DB, `RealExecutor` stub throws |
| No fake fills | Fills only against **FRESH** on-chain pool state at *simulated execution time* (decision time + sampled latency) |
| Realistic costs | Economics engine: CP pool math from reserves, DEX/network/priority fees, ATA rent, latency, failures, MEV |
| Correct money | Double-entry ledger in integer base units. Invariants checked after every transaction. Violation halts the run. |
| Traceability | Every decision links to the signal, trigger events, wallet scores, data quality, risk checks, order, fill and ledger |
| Reproducibility | Injected Clock/Rng, content-addressed config versions, flush barrier before decisions, replay through the same pipeline, recorded AI outputs |
| Time awareness | `slot`, `event_time`, `provider_time`, `received_at`, `normalized_at`, `processed_at` and stage timestamps. Latency measured per stage. |
| Data quality | FRESH/STALE/DEGRADED/UNKNOWN computed at read time per consumer max age |

## 4. Modes

| Mode | Execution | Status |
|---|---|---|
| SIMULATION | PaperExecutor | MVP |
| SHADOW | Paper + read-only real quotes | Phase 26 prep |
| LIVE | Isolated signer service (future) | Not implemented. Blocked. |

## 5. Deployment

Engine + Postgres on a small always-on host (Docker Compose + Caddy recommended). Dashboard on Vercel. See [docs/25-infrastructure-deployment.md](docs/25-infrastructure-deployment.md).

## 6. Decisions

All architecture decisions and open decision points: [docs/32-architecture-decision-records.md](docs/32-architecture-decision-records.md).
