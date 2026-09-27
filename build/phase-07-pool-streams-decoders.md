# Phase 07 — Pool Streams & Venue Decoders

| Field | Value |
|---|---|
| Milestone | M1 Truthful data |
| Depends on | 05, 06 |
| Size | L |
| Requirements | FR-ING-002, FR-ING-005/006/007 (for pools), FR-MKT-003 (slot monotonic input) |

## 1. Objective
Stream on-chain pool state (reserves) and trades for watched pools, using pure, fixture-verified decoders per supported venue. Emit `pool.state.updated` and `pool.trade.observed` events, handle reconnection with immediate state refresh, and integrate pool watch requests into the subscription manager.

## 2. Context (read first)
- [10](../docs/10-market-data-spec.md) §2–5
- [09](../docs/09-real-time-data-architecture.md) §6, §8, §11
- [17](../docs/17-economics-engine-spec.md) §4 (reserve semantics per venue)
- `fixtures/venues/**` READMEs (Phase 03)
- ADR-0010

## 3. Dependencies
Phases 05 and 06 DONE.

## 4. Inputs
SubscriptionManager (wallets), the reference module (pools with state accounts/vaults), and venue fixtures.

## 5. Outputs
- `modules/market-state/decoders/<venue>.ts` implementing `VenueDecoder` for each **verified** venue (DP-04): `decodePoolState`, `decodeTradeLogs`, and optionally `decodeTransaction`.
- `modules/ingestion/helius/pool-stream.ts`: pool watch → `accountSubscribe` (state account and/or vaults) + `logsSubscribe({mentions:[pool]})`, reconnect refresh via `getMultipleAccounts`.
- Normalizers for `pool.state.updated` v1 and `pool.trade.observed` v1 (schemas registered).
- Watch command API on ingestion: `watchPool(pool, priority)`, `releasePool(pool)`.
- GeckoTerminal trades poller (fallback + gap backfill) emitting `pool.trade.observed` with the same dedup key space.

## 6. Files To Create
```text
apps/engine/src/modules/market-state/decoders/
  venue-decoder.ts  registry.ts
  pumpfun-curve.ts  pumpswap.ts  raydium-amm-v4.ts  raydium-cpmm.ts   (only verified venues)
  anchor-event.ts   # shared Anchor event discriminator + borsh helpers
  __tests__/<venue>.test.ts
apps/engine/src/modules/ingestion/helius/{pool-stream.ts,pool-state-normalizer.ts,pool-trade-normalizer.ts}
apps/engine/src/modules/ingestion/geckoterminal/trades-poller.ts
apps/engine/src/modules/ingestion/__tests__/pool-stream.test.ts
```

## 7. Files To Modify
- `modules/ingestion/helius/subscription-manager.ts` (pool subscriptions with priorities P0/P1)
- Phase 05 wallet log decoding → reuse the venue decoders (remove duplication)
- `packages/core/src/events/event-types.ts` (register the payload schemas)

## 8. Database Changes
None (events go to `market.events`. Projections come in Phase 08).

## 9. API Changes
None.

## 10. Environment Variables
None new.

## 11. Implementation Tasks

**07.1 Anchor/borsh helpers**
- Tests first: discriminator computation (`sha256("event:<Name>")[0..8]` for events, `sha256("account:<Name>")[0..8]` for accounts). u64/u128/i64/pubkey/bool readers against known byte arrays. Base64 `Program data:` extraction from logs.

**07.2 Decoder per verified venue (repeat per venue)**
- Tests first using `fixtures/venues/<venue>/`: `decodePoolState` reproduces the reserves recorded in the README (exact integers). `decodeTradeLogs` yields the expected trades (side, amounts, trader when available, reserves after when available) for every recorded notification. Malformed data → `DecodeError` (never throws past the boundary).
- Implement per the verified layouts. Each decoder file header cites the verification reference (IDL commit/URL).
- Include **Raydium AMM v4** reserve semantics (vault balances adjusted per the official source) and **CPMM** fee-accrual subtraction, as verified.

**07.3 PoolStream**
- Tests first with a fake WS server: `watchPool` subscribes to the account(s) + logs. The first state is fetched immediately via `getMultipleAccounts` (mock). Account notifications → `pool.state.updated` (with `context.slot`). Logs notifications → decoded trades → `pool.trade.observed` (`sourceEventKey = sol:<sig>:<logIndex>`). Failed transactions (`err != null`) are ignored for trades. On reconnect, resubscribe and refresh state. `releasePool` unsubscribes after the linger period.
- For vault-based venues: combine two vault account updates into one state by slot (emit when both vaults are known for the slot, or after a 1-slot grace with the last known other vault, flagged `partial=true` → quality DEGRADED).

**07.4 Subscription priorities**
- Tests first: P0 (held) pool requests preempt P1/P2 when at capacity. Release frees capacity. `WATCH_CAPACITY_CRITICAL` when P0 can't be served.

**07.5 GeckoTerminal trades poller**
- Tests first with recorded trades: maps to `pool.trade.observed` (dedup key `sol:<tx_hash>:<index>` aligned with on-chain keys; verify how GT exposes the log index, and if it doesn't, use the key `sol:<tx_hash>:gt` and document that cross-source dedup works only per-signature for GT). It respects the 30/min budget allocation. It is used only (a) when on-chain trade decoding is unavailable for the venue, or (b) for gap backfill.

**07.6 Engine wiring (temporary)**
- Until the WatchSetPolicy (Phase 15) exists: add a dev-only config `watch.debug_pools: []` to watch specific pools, for manual verification. This is removed or ignored once the policy exists (keep it for diagnostics, documented in [26](../docs/26-configuration-reference.md)).

## 12. Acceptance Criteria
1. Each verified venue decoder passes all fixture tests exactly.
2. Watching a real active pool yields a continuous stream of state and trade events. Manual spot-check: 10 trades vs a block explorer, and state vs `getAccountInfo` at the same slot.
3. Reconnect refreshes state immediately (test).
4. Unverified venues have no decoder, and tokens on them are rejected by reference (Phase 06 behaviour intact).

## 13. Tests
Unit (decoders, helpers), contract (fixtures), event-stream (fake WS: subscribe/reconnect/refresh), integration (events persisted).

## 14. Failure Cases
- Decode error rate > 1% for a venue over 5 min → WARN, and the venue is marked `DEGRADED` in the market state (Phase 08 consumes this). Never guess values.
- Account closed (pool removed) → `pool.state.updated{closed:true}` → reference marks the pool `CLOSED`.

## 15. Observability
Metrics per venue: decode success/failure counts, trades/s, state updates/s, bytes received (for the credit estimate). Credits charged per MB via the CreditBudget.

## 16. Security
Decoders operate on untrusted bytes. Bounds-check every read and cap array lengths. Fuzz test each decoder with random bytes (fast-check): it must never throw uncaught or hang.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- decoders pool-stream
# manual: set watch.debug_pools to a live pool, run the engine, compare with explorer
```

## 18. Commit Strategy
1. `feat(market-state): add anchor event and borsh helpers`
2. `feat(market-state): add <venue> decoder` (one commit per venue)
3. `feat(ingestion): add pool stream with state refresh on reconnect`
4. `feat(ingestion): add pool watch priorities`
5. `feat(ingestion): add geckoterminal trades fallback poller`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Decoder fuzz tests pass.
