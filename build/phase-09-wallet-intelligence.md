# Phase 09 — Wallet Intelligence

| Field | Value |
|---|---|
| Milestone | M2 Signals |
| Depends on | 05, 08 |
| Size | L |
| Requirements | FR-WAL-001..004 |

## 1. Objective
Turn wallet activity into deterministic, versioned wallet scores. This covers history backfill (credit-budgeted), persisted swaps, FIFO round-trip reconstruction, the `wallet-score@1.0.0` model with penalties and confidence shrinkage, as-of-time score resolution, and the live `smart.buy.detected` / `smart.sell.detected` path.

## 2. Context (read first)
- [11](../docs/11-wallet-intelligence-spec.md) (entire)
- [07](../docs/07-database-schema.md) §3.3
- [26](../docs/26-configuration-reference.md) `wallet.*`
- [19](../docs/19-backtesting-spec.md) §2 (as-of-time)

## 3. Dependencies
Phases 05 and 08 DONE.

## 4. Inputs
Wallet registry, live swaps, the RPC client + credit budget, market state (SOL/USD), and GeckoTerminal OHLCV access (for historical SOL/USD and the C5 early-entry metric).

## 5. Outputs
- Migration `0008_intel.sql` (`wallet_swaps` monthly partitions, `wallet_round_trips`, `wallet_scores`, `backfill_jobs`).
- `modules/wallet-intel`: `SwapRecorder` (live → DB), `BackfillWorker` (worker thread or low-priority loop), `RoundTripBuilder` (pure), `WalletScorer` (pure), `ScoreRepository` with as-of lookup, `SmartActivityDetector` (live path), and scheduler jobs (daily recompute).
- Events: `wallet.score.updated`, `smart.buy.detected`, `smart.sell.detected`.
- `wallet.mining` capability behind a flag (off by default).

## 6. Files To Create
```text
packages/db/migrations/0008_intel.sql
apps/engine/src/modules/wallet-intel/domain/{round-trips.ts,scoring.ts,penalties.ts,normalizers.ts,smart-activity.ts,mining.ts}
apps/engine/src/modules/wallet-intel/adapters/{swap-repository.ts,round-trip-repository.ts,score-repository.ts,backfill-repository.ts,history-source-rpc.ts,sol-usd-history.ts}
apps/engine/src/modules/wallet-intel/{backfill-worker.ts,score-jobs.ts}
apps/engine/src/modules/wallet-intel/__tests__/...
fixtures/wallets/<case>/{swaps.json,expected-round-trips.json,expected-score.json}
```

## 7. Files To Modify
- `modules/wallet-intel/service.ts` (subscribe to swaps, emit smart events)
- `modules/ingestion/helius/gap-backfiller.ts` (prioritize by score now that scores exist)
- `packages/core/src/events/event-types.ts` (internal event types)

## 8. Database Changes
`0008_intel.sql` per [07](../docs/07-database-schema.md) §3.3 (excluding `wallets`, which exists). Monthly partitions for `intel.wallet_swaps`.

## 9. API Changes
None (HTTP in Phase 18).

## 10. Environment Variables
None new.

## 11. Implementation Tasks

**09.1 Swap persistence**
- Tests first (integration): live `wallet.swap.detected` → `intel.wallet_swaps` (`origin=LIVE`), idempotent on `(event_time, signature, log_index, wallet)`. Async, so the hot path does not await it.

**09.2 RoundTripBuilder (pure)**
- Tests first (fixtures + property): FIFO matching. Partial sells. Dust threshold closing. Unknown-cost transfers flagged. Invariant: Σ realized = Σ proceeds − Σ consumed cost. Deterministic ordering by `(slot, log_index)`.

**09.3 Normalizers and scoring (pure)**
- Tests first: each component's normalization function at key points (logistic mid → 0.5, clamps). Weight renormalization when C5 is unavailable. Shrinkage formula. Minimum sample flag. Golden expected scores for ≥ 5 fixture wallets (hand-verified in `expected-score.json`, with a calculation sheet in the fixture README).

**09.4 Penalty flags (pure)**
- Tests first: each flag triggers on its crafted fixture and not on the near-miss fixture. Multipliers compound. Clamped to [0,1].

**09.5 Score repository with as-of-time**
- Tests first (integration): insert a new score → the previous `is_current` flips in the same transaction. `getScoreAsOf(wallet, t)` returns the latest `computed_at ≤ t`, never a future one.

**09.6 Backfill worker**
- Tests first (mock RPC with recorded pages): pages `getSignaturesForAddress` to the lookback/limits. Decodes via venue decoders / balance-delta. Persists `origin=BACKFILL`. Resumable from `cursor`. Stops with `BUDGET_EXHAUSTED` at the per-job cap or on global refusal. Runs at the lowest priority. SOL/USD history from GeckoTerminal SOL/USDC bars (cached) for `value_usd`, else null.

**09.7 Early-entry metric C5**
- Tests first: for buys with available pool price history (GT OHLCV 1m bars, fetched within budget and cached in `market.ohlcv_bars` with source GECKOTERMINAL), compute max price in 60 min / entry. Fewer than 5 observations → the component is unavailable.

**09.8 Score jobs**
- Tests first (SimulatedClock): recompute on backfill completion. Recompute on a live closed round trip. The daily full recompute runs for TRACKED/CANDIDATE wallets. `wallet.score.updated` is emitted.

**09.9 SmartActivityDetector (live path)**
- Tests first: TRACKED + score ≥ min + not flagged `INSUFFICIENT_SAMPLE` + not BLOCKED + BUY ≥ `min_buy_value_sol` → `smart.buy.detected` with a `walletScoreSnapshot` (score ID, value, flags). Each failing condition → no event. SELL by a smart wallet → `smart.sell.detected`. The latency stage for this step is recorded.

**09.10 Auto-promotion (flagged) and mining (flagged)**
- Tests first: when `wallet.auto_promote=true`, the promote/demote rules apply with system events. When false, nothing changes. Mining with the flag on adds CANDIDATE wallets from recorded early buyers (uses only stored trades).

## 12. Acceptance Criteria
1. Imported real wallets are backfilled within the credit caps, and scores are computed with component breakdowns (record ≥ 5 examples in the verification note).
2. Golden fixture scores match exactly.
3. As-of-time lookup never returns future scores (test).
4. Live smart buys emit `smart.buy.detected` with a score snapshot.
5. Credit usage for backfills is recorded per job.

## 13. Tests
Unit/property (round trips, scoring, penalties), integration (repositories, backfill worker with a mock RPC), scenario (live smart-buy detection).

## 14. Failure Cases
- Backfill RPC errors → job `FAILED` with the error. Resumable. It does not block live processing.
- A transaction that can't be parsed → skipped + counted (not guessed).
- Missing SOL/USD history → `value_usd` null. Metrics use SOL-denominated values.

## 15. Observability
Logs: `backfill.progress`, `backfill.done`, `wallet.score.updated`. Metrics: backfill credits, jobs by status, smart buys/sells per minute.

## 16. Security
None specific (public on-chain data).

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- wallet-intel
pnpm test:integration --filter @paperbot/engine -- wallet-intel
# manual: import 10 real wallets, run backfill, inspect scores
```

## 18. Commit Strategy
1. `feat(db): add intel swaps, round trips, scores, backfill jobs`
2. `feat(wallet-intel): persist live swaps`
3. `feat(wallet-intel): add fifo round-trip builder`
4. `feat(wallet-intel): add wallet score model v1 with penalties`
5. `feat(wallet-intel): add as-of-time score repository`
6. `feat(wallet-intel): add credit-budgeted backfill worker`
7. `feat(wallet-intel): add score jobs and smart activity detector`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
