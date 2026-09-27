# 06 — Data Architecture

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [05](05-system-architecture.md), [03](03-system-requirements.md) |
| Downstream | [07](07-database-schema.md), [09](09-real-time-data-architecture.md), [19](19-backtesting-spec.md) |
| Used by phases | 02, 04, 08, 13, 17 |

## 1. Data categories

| Category | Examples | Nature | System of record | Mutability |
|---|---|---|---|---|
| **Event log** | Normalized provider events | High-frequency, append-only | `market.events` (partitioned) | Immutable |
| **Market projections** | Pool state snapshots, trades, snapshots, OHLCV bars | High/medium-frequency | `market.*` projection tables | Rebuildable from event log (plus reference backfills) |
| **Reference** | Tokens, pools, venues | Low-frequency | `ref.*` | Upserts with `updated_at`. Safety facts versioned by observation time. |
| **Wallet intelligence** | Wallets, swaps, round trips, scores | Medium | `intel.*` | Swaps immutable. Scores append-only (one `is_current`). |
| **Strategy/decision** | Strategy versions, config versions, runs, waves, signals, decisions, risk decisions, AI outputs | Low–medium | `strategy.*` | Immutable records + explicit status transitions |
| **Simulation/accounting** | Orders, fills, ledger, positions, portfolio snapshots | Low | `sim.*` | Ledger immutable. Orders/positions status-transitioned with history. |
| **Operational** | Latency samples, provider health, system events, gaps, credit usage, dead letters | Medium | `ops.*` | Append-only, short retention |
| **In-memory state** | Market state, wave state machines, dedup LRU, open positions cache | Hot | Engine memory, derived from records | Rebuilt on restart |

## 2. Sources of truth

```text
Blockchain (ultimate truth, external)
   └─▶ Provider payloads (external, may be wrong/late/duplicated)
        └─▶ market.events  ◀── internal source of truth for "what the system knew, and when"
             ├─▶ market projections (rebuildable)
             ├─▶ in-memory market state (rebuildable)
             └─▶ decisions/orders/fills reference event IDs they depended on

sim.ledger_entries  ◀── source of truth for money
   └─▶ positions, portfolio snapshots, analytics (derived, must reconcile)

strategy.config_versions + strategy.strategy_versions ◀── source of truth for "which rules applied"
```

Rules:
1. Derived tables MUST be reproducible from their sources. Tests verify rebuild equality on fixtures.
2. If a derived value disagrees with its source, the source wins and a `ops.system_events` `RECONCILIATION_MISMATCH` is raised.
3. Decisions store a **feature snapshot** as well as event references. The trace remains readable even after raw events age out of retention.

## 3. Data lifecycle and tiers

| Tier | Storage | Contents | Default retention |
|---|---|---|---|
| **Hot (memory)** | Engine process | Market state for watched subjects. Last `N` minutes of trades per watched pool (`market_state.trade_window_minutes`, default 60). Wave windows. | While watched + window |
| **Warm (Postgres)** | Partitioned tables | Event log, projections, ops data | Event log: `retention.events_days` (default 30). Trades/snapshots/pool states: 30. Bars 1m: 180. Bars 5m/1h: 730. Ops latency/health: 14. |
| **Permanent (Postgres)** | Regular tables | Reference, intel, strategy, sim | Indefinite (small volume) |
| **Cold archive (optional)** | Compressed NDJSON (gzip) per partition-day in object storage (Supabase Storage / S3-compatible) | Event log partitions before drop | Indefinite. Required for replay beyond warm retention. |

Partition drop procedure: `archive (if enabled) → verify checksum/row count → detach partition → drop`. Runs as a scheduled maintenance job ([07](07-database-schema.md) §6). Archive format matches the `market.events` row schema so `ReplayEventSource` can read archives directly ([19](19-backtesting-spec.md) §4).

## 4. High-frequency data strategy

The high-frequency path must not make Postgres the bottleneck:

1. **Filter at the edge.** Only subscribe to subjects in the watch set (tracked wallets + watched pools). No firehose ingestion.
2. **Coalesce.** `pool.state.updated` for the same pool within one slot collapse to the last one before persistence (state is absolute, so latest-wins is lossless for state reconstruction at slot granularity).
3. **Batch writes.** The persister accumulates events and writes with multi-row `INSERT` (or `COPY`) every `pipeline.persist.flush_ms` (default 250 ms) or `pipeline.persist.batch_size` (default 500), whichever comes first.
4. **Partition by day** on `received_at` for `market.events` and all high-frequency projections. Indexes are local to partitions and kept minimal.
5. **Throttle projections.** `market.pool_states` persists at most one row per pool per `market_state.pool_state_persist_interval_ms` (default 1,000 ms) *plus* every state used for a fill (referenced by the fill). The full-resolution history remains in `market.events`.
6. **Derived bars** are computed in memory and upserted on bar close (plus one "open bar" upsert every 5 s for charts).

### Volume estimates (for sizing, to be measured in Phase 23/25)

| Stream | Assumption | Rows/day (est.) | Size/day (est.) |
|---|---|---|---|
| Wallet swaps | 100 tracked wallets × 20 swaps/day | 2,000 | ~2 MB |
| Pool trades | 10 watched pools × avg 30 trades/min × watch time 25% of day | ~110,000 | ~100 MB |
| Pool states (event log, coalesced) | similar magnitude to trades | ~100,000 | ~80 MB |
| Snapshots | 10 tokens × 1 per 15 s × 25% of day | ~15,000 | ~10 MB |
| **Total event log** | | **~0.2–0.3 M** | **~0.2 GB/day** |

With 30-day retention that is roughly 6 GB, which exceeds the Supabase free-tier storage quota. Hosting consequences are in [ADR-0018](32-architecture-decision-records.md#adr-0018-database-hosting) and [25](25-infrastructure-deployment.md) §3.

## 5. Data ownership (module → schema/tables)

| Module | Tables owned (write) |
|---|---|
| pipeline | `market.events` |
| market-state | `market.pool_states`, `market.trades`, `market.snapshots`, `market.ohlcv_bars`, `market.reference_prices` |
| reference | `ref.venues`, `ref.tokens`, `ref.token_safety_observations`, `ref.pools` |
| wallet-intel | `intel.wallets`, `intel.wallet_swaps`, `intel.wallet_round_trips`, `intel.wallet_scores`, `intel.backfill_jobs` |
| strategy | `strategy.strategies`, `strategy.strategy_versions`, `strategy.config_versions`, `strategy.runs`, `strategy.waves`, `strategy.wave_transitions`, `strategy.signals`, `strategy.decisions` |
| risk | `strategy.risk_decisions` |
| ai | `strategy.ai_outputs` |
| execution | `sim.orders`, `sim.order_transitions`, `sim.fills` |
| portfolio | `sim.ledger_transactions`, `sim.ledger_entries`, `sim.positions`, `sim.position_transitions`, `sim.portfolio_snapshots` |
| observability | `ops.latency_samples`, `ops.provider_health`, `ops.system_events`, `ops.data_gaps`, `ops.credit_usage`, `ops.dead_letters`, `ops.clock_offsets` |

## 6. Identity and keys

- Internal IDs are **UUIDv7** (time-ordered, index-friendly, generated in the app so they are known before insert, which idempotency needs).
- On-chain identities keep their natural keys (base58 strings): mint, pool address, wallet address, signature.
- Idempotency keys:
  - Events: `source_event_key` (e.g. `helius.logs:<signature>:<logIndex>`, `dexscreener:<pair>:<pairUpdatedAt>`).
  - Orders: `idempotency_key = sha256(run_id | decision_id | intent | leg)`.
  - Ledger transactions: `ref_type + ref_id` unique.

## 7. Time columns (canonical names)

Used consistently across all tables ([09](09-real-time-data-architecture.md) §4):

| Column | Meaning |
|---|---|
| `slot` | Solana slot of the underlying on-chain event |
| `event_time` | When the event happened on chain / at provider (block time or provider trade time) |
| `provider_time` | Timestamp the provider attached, if distinct |
| `received_at` | When our process received the bytes |
| `normalized_at` | When the envelope was created |
| `processed_at` | When the consuming module applied it |
| `signal_at` / `decided_at` / `evaluated_at` / `submitted_at` / `executed_at` / `recorded_at` | Stage-specific timestamps |
| `created_at` / `updated_at` | Row bookkeeping only (never used for analytics) |

All are `timestamptz` (microsecond precision) in UTC.

## 8. Data quality lineage

Every derived value used in a decision carries `{quality, age_ms, source}` from its inputs. Quality propagates as the **worst** of its inputs (`UNKNOWN` > `STALE` > `DEGRADED` > `FRESH` in severity order). Signals and decisions persist these per-input quality tuples ([09](09-real-time-data-architecture.md) §7, [28](28-decision-logging-spec.md)).

## 9. Privacy

Wallet addresses are public on-chain data. No personal data is collected. Operator credentials are the only sensitive user data, and they live in platform secrets, not in the database.
