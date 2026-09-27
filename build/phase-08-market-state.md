# Phase 08 — Market State Engine

| Field | Value |
|---|---|
| Milestone | M1 Truthful data |
| Depends on | 07 |
| Size | L |
| Requirements | FR-MKT-001..004, FR-ING-003, FR-ING-004 |

## 1. Objective
Build the single internal market state: per-pool reserves, prices, liquidity, trade ring buffers, snapshots and bars, all with per-consumer quality evaluation. Also the market snapshot pollers (DexScreener/GeckoTerminal), the SOL/USD reference, persisted projections, and OHLCV derivation. After this phase, milestone M1 is demonstrable.

## 2. Context (read first)
- [10](../docs/10-market-data-spec.md) §1, §5, §6, §8, §9
- [09](../docs/09-real-time-data-architecture.md) §7, §8
- [06](../docs/06-data-architecture.md) §4
- [07](../docs/07-database-schema.md) §3.2 (projections)
- [26](../docs/26-configuration-reference.md) `freshness.*`, `market_state.*`

## 3. Dependencies
Phase 07 DONE.

## 4. Inputs
Pool state/trade events, reference data, and the pipeline.

## 5. Outputs
- Migration `0007_market_projections.sql`.
- `modules/market-state`: `MarketStateStore` (in memory), `QualityEvaluator` (per-consumer max ages), `PriceCalculator`, `TradeRingBuffer`, `BarBuilder`, projection writers (throttled pool states, trades, snapshots, bars, reference prices), and `MarketStateView` (the public read API + change subscription).
- `modules/ingestion/dexscreener/snapshot-poller.ts`, `geckoterminal/snapshot-fallback.ts`, `sol-usd/*` pollers emitting `market.snapshot.observed` and `price.reference.updated`.
- `market.state.changed` bus event (coalesced).

## 6. Files To Create
```text
packages/db/migrations/0007_market_projections.sql
apps/engine/src/modules/market-state/
  index.ts ports.ts service.ts view.ts
  domain/{price.ts,quality-evaluator.ts,trade-ring-buffer.ts,bar-builder.ts,pool-market-state.ts,sol-usd.ts}
  adapters/{pool-state-writer.ts,trade-writer.ts,snapshot-writer.ts,bar-writer.ts,reference-price-writer.ts}
  __tests__/...
apps/engine/src/modules/ingestion/dexscreener/snapshot-poller.ts
apps/engine/src/modules/ingestion/geckoterminal/snapshot-fallback.ts
apps/engine/src/modules/ingestion/sol-usd/{dexscreener-sol.ts,geckoterminal-sol.ts,coingecko-sol.ts,sol-usd-aggregator.ts}
```

## 7. Files To Modify
- `packages/core/src/events/event-types.ts` (register `market.snapshot.observed`, `price.reference.updated`)
- The retention job registration (projection tables) in the observability/maintenance scheduler

## 8. Database Changes
`0007_market_projections.sql`: `market.pool_states`, `market.trades`, `market.snapshots` (partitioned daily), `market.ohlcv_bars`, `market.reference_prices`. Partition manager config updated for the new tables.

## 9. API Changes
None.

## 10. Environment Variables
`COINGECKO_API_KEY` (optional).

## 11. Implementation Tasks

**08.1 Price and liquidity math (pure)**
- Tests first (golden + property): mid price from raw reserves with decimals (e.g. 6-decimal token vs 9-decimal SOL). Virtual-curve price uses virtual reserves. Liquidity conventions per [10](../docs/10-market-data-spec.md) §5. Decimal precision is ≥ 1e-12 relative. There is no float usage (type-level/lint).

**08.2 QualityEvaluator**
- Tests first (SimulatedClock, table-driven): for each consumer (`WAVE`, `ENTRY`, `FILL`, `EXIT`, `MARK`) and input (`POOL_STATE`, `SNAPSHOT`, `SOL_USD`, `TRADES`, `WALLET_STREAM`), FRESH→STALE exactly at the max age boundary. DEGRADED for fallback sources or partial vault states. UNKNOWN when no value exists. A stream disconnect → STALE immediately.

**08.3 Pool market state + slot monotonicity**
- Tests first: a newer slot applies. An older slot is ignored + counted (FR-MKT-003). An equal slot with a later `ingestSeq` replaces. `curveComplete` is propagated. The closed pool flag is handled.

**08.4 TradeRingBuffer**
- Tests first: dedup by `(signature, logIndex)`. Window queries by slot/time (`W_short`, `W_base`, since-seed). Incremental aggregates (counts, volumes, unique traders) equal brute-force recomputation (property test). Eviction beyond `trade_window_minutes`.

**08.5 BarBuilder (1m/5m/1h)**
- Tests first: OHLC from trades. Buys/sells/volumes. Carry-forward rules per [10](../docs/10-market-data-spec.md) §8. Bars close at the bucket boundary on the simulated clock. The open-bar upsert happens every 5 s.

**08.6 Snapshot pollers**
- Tests first with recorded payloads: the DexScreener batch tokens call for watched tokens → `market.snapshot.observed` per token (fields mapped per verified names). Rate-limit adherence. Fallback to GeckoTerminal on failure with quality DEGRADED. Freshness uses `provider_time` when present.

**08.7 SOL/USD aggregator**
- Tests first: DexScreener primary at 30 s. GeckoTerminal secondary. CoinGecko cross-check at 5 min (when enabled). Divergence > 100 bps → DEGRADED + WARN. Staleness transitions. The emitted `price.reference.updated` carries the source.

**08.8 MarketStateStore + View**
- Tests first: consumes the four event types. `getPool`/`getToken`/`getSolUsd` return `Q<>` values with quality computed at read time per consumer. `subscribe` receives coalesced `market.state.changed` (max 4/s per pool, configurable). `poolStateAt(pool, t)` (used by the executor in replay) returns the latest state with a replay timestamp ≤ t, kept in a bounded per-pool history (last 120 s by default; backed by `market.events` for older lookups in recovery).

**08.9 Projection writers**
- Tests first (integration): pool states are throttled to 1/s per pool plus forced writes (API for the executor: `markUsedForFill(eventId)`). Trades are written with natural-key dedup. Snapshots, bars and reference prices are persisted. Writes are async and batched. The hot path never awaits them.

**08.10 M1 demo wiring**
- With `watch.debug_pools` or tokens from live wallet swaps (temporary auto-watch in dev config: `watch.debug_auto_watch_wallet_buys = true`), the engine maintains market state for real tokens. Log a periodic summary line per watched pool (price, liquidity, quality, trades/min).

## 12. Acceptance Criteria
1. Market state for real pools matches independent sources: mid price within 0.5% of DexScreener `priceNative` at comparable times (sample of 20; differences explained by timing), and reserves equal to `getAccountInfo` at the same slot.
2. Quality flags transition exactly at the configured thresholds (tests).
3. Projections persist at the specified cadence. Bars render plausible candles (manual check vs GeckoTerminal for the same pool).
4. SOL/USD is available with quality, and divergence detection works.
5. M1 demo recorded in the verification note (logs/screens).

## 13. Tests
Unit/property (math, buffers, bars, quality), contract (snapshot payloads), integration (writers), event-stream (out-of-order states).

## 14. Failure Cases
- All SOL/USD sources down → STALE → entries paused later (`SOL_USD_STALE`, Phase 15).
- A snapshot provider returns a stale `provider_time` → the value is applied only if newer. The age is computed from `provider_time`.
- A partial vault state → DEGRADED (never FRESH for fills).

## 15. Observability
Metrics: `paperbot_stream_age_seconds` (top-N), state updates/s, out-of-order count, snapshot poll latency, SOL/USD divergence gauge. Log `market.summary` every 60 s per watched pool (info) and quality transitions (debug).

## 16. Security
None specific. Snapshot strings from providers (e.g. token names) are untrusted and are not stored in snapshots (only numeric fields).

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- market-state
pnpm test:integration --filter @paperbot/engine -- market-state
HELIUS_API_KEY=... pnpm dev:engine   # with debug auto-watch; capture the summary lines
```

## 18. Commit Strategy
1. `feat(db): add market projection tables`
2. `feat(market-state): add price math and quality evaluator`
3. `feat(market-state): add pool state store with slot monotonicity`
4. `feat(market-state): add trade ring buffer and bar builder`
5. `feat(ingestion): add snapshot pollers and sol/usd aggregator`
6. `feat(market-state): add market state view and projection writers`
7. `docs: phase 08 verification (M1 demo)`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M1 marked in `build/PHASE-STATUS.md`.
