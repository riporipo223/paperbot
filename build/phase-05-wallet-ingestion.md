# Phase 05 — Wallet Activity Ingestion (Helius)

| Field | Value |
|---|---|
| Milestone | M1 Truthful data |
| Depends on | 04 |
| Size | L |
| Requirements | FR-ING-001, FR-ING-005, FR-ING-007, FR-WAL-001 (registry part) |

## 1. Objective
Detect tracked-wallet swaps on Solana in real time and emit normalized `wallet.swap.detected` events. This covers the wallet registry, the subscription manager across limited WS connections, log-based and transaction-based swap decoding (generic balance-delta parser), gap detection and backfill after reconnect, the slot clock, and detection-latency measurement. Execute and record the ADR-0009 measurement.

## 2. Context (read first)
- [10](../docs/10-market-data-spec.md) §3, §4.1–4.2
- [09](../docs/09-real-time-data-architecture.md) §2.2, §5, §6, §11, §12
- [11](../docs/11-wallet-intelligence-spec.md) §2, §7
- [27](../docs/27-data-provider-reference.md) §4 (as verified in Phase 03)
- ADR-0009, ADR-0015

## 3. Dependencies
Phase 04 DONE.

## 4. Inputs
Pipeline, verified Helius facts, fixtures (logs notifications, transactions), and the WS/RPC clients.

## 5. Outputs
- Migration `0005_intel_wallets.sql` (`intel.wallets`).
- `modules/wallet-intel` (registry part only): wallet repository, import/add/status API (internal service methods; HTTP comes in Phase 18), tracked-set change events.
- `modules/ingestion/helius`: `SubscriptionManager`, `HeliusWsAdapter` (logs subscriptions for wallets), `WalletSwapNormalizer`, `TxEnricher` (`getTransaction` + balance-delta parser), `GapBackfiller`, `SlotClock`.
- Optional `HeliusWebhookAdapter` (the handler function only; the HTTP route arrives in Phase 18), behind `ingestion.wallet_source = helius_webhook`.
- Payload schema for `wallet.swap.detected` v1 registered.
- ADR-0009 updated with measured results and status.

## 6. Files To Create
```text
packages/db/migrations/0005_intel_wallets.sql
apps/engine/src/modules/wallet-intel/{index.ts,ports.ts,service.ts}
apps/engine/src/modules/wallet-intel/adapters/wallet-repository.ts
apps/engine/src/modules/wallet-intel/domain/{wallet.ts,import-parser.ts}
apps/engine/src/modules/ingestion/helius/
  subscription-manager.ts ws-adapter.ts wallet-swap-normalizer.ts tx-enricher.ts
  balance-delta-parser.ts gap-backfiller.ts slot-clock.ts webhook-adapter.ts schemas.ts(extend)
  __tests__/...
apps/engine/src/modules/ingestion/index.ts (public API: watch commands)
scripts/measure-wallet-latency.ts        # ADR-0009 measurement tool (real network; manual)
```

## 7. Files To Modify
- `packages/core/src/events/event-types.ts` (register `wallet.swap.detected` v1)
- `docs/32-architecture-decision-records.md` (ADR-0009 outcome)
- `docs/27-data-provider-reference.md` (any newly learned limits)

## 8. Database Changes
`0005_intel_wallets.sql`: `intel.wallets` per [07](../docs/07-database-schema.md) §3.3.

## 9. API Changes
None (internal services only).

## 10. Environment Variables
`HELIUS_API_KEY`. `HELIUS_WEBHOOK_AUTH` (only if the webhook source is enabled).

## 11. Implementation Tasks

**05.1 Wallet registry**
- Tests first: add wallet (base58 validated; idempotent). Import CSV/JSON with per-row results (invalid rows reported, valid ones inserted as `CANDIDATE`). Status transitions per [11](../docs/11-wallet-intelligence-spec.md) §2 (BLOCKED never TRACKED without an explicit operator unblock). A `tracked.set.changed` bus event on changes.

**05.2 SubscriptionManager**
- Tests first: packs N wallet subscriptions across ≤ `max_ws_connections` × `max_subscriptions_per_connection`. Priority ordering (held > watched pools > wallets; only wallets exist in this phase, and pool watch requests come in Phase 07). Over capacity → `WATCH_CAPACITY_EXCEEDED` with the lowest-priority subjects unserved. Add/remove at runtime. Rebalance after reconnect.

**05.3 SlotClock**
- Tests first: records `slot → first_observed_at`. Estimates slot time by EWMA slot rate. Interpolation/extrapolation. Periodic `getSlot` polling respects the credit budget.

**05.4 Log-based swap decoding for wallets**
- Tests first, using recorded pump.fun/PumpSwap logs notifications that mention a wallet: produce a `DecodedSwap` with side, amounts, mint, pool (where derivable), and `parseMethod='VENUE_EVENT'`. Logs without a recognizable venue event → `NEEDS_ENRICHMENT`.
- Note: the full venue decoders are built in Phase 07. This task implements only the minimal event parsing needed for wallet swaps. Structure it as the `decodeTradeLogs` function of the pump venues so Phase 07 extends it rather than duplicating it.

**05.5 Balance-delta parser**
- Tests first with recorded transactions: BUY, SELL, WSOL-wrapped, fee-payer vs non-fee-payer, a multi-hop/token-token swap → `UNSUPPORTED_SWAP_SHAPE`, and a plain transfer → no swap. Venue attribution by program ID.
- Implement per [10](../docs/10-market-data-spec.md) §4.2.

**05.6 TxEnricher**
- Tests first: a `NEEDS_ENRICHMENT` notification → `getTransaction` (mock with a recorded response) → parsed swap. Priority queue (per [09](../docs/09-real-time-data-architecture.md) §10). A credit charge per call. On failure → retry policy → the event is emitted with `quality=DEGRADED` and missing amounts are marked (not fabricated) if the retries are exhausted. It does **not** emit a swap with invented values: it emits `wallet.swap.unparsed` (dead-letter-like system record) instead.

**05.7 WalletSwapNormalizer**
- Tests first: builds the envelope with `sourceEventKey = sol:<signature>:<wallet>`, `slot`, `eventTime` (blockTime if known, else null), `receivedAt` from the adapter stamp, `normalizedAt`, and `quality`. Schema-valid payload. Latency samples `L1_ingest`, `L1b_enrich`, `detect_slots`, `detect_ms_est` recorded.

**05.8 HeliusWsAdapter + reconnect + gap backfill**
- Tests first with a fake WS server: subscribe the tracked wallets. Drop the connection → reconnect → resubscribe. A gap is recorded in `ops.data_gaps`. `GapBackfiller` calls `getSignaturesForAddress(wallet, {until: lastSeenSig})` (mock) and emits the recovered swaps with `late` set correctly. Backfill is prioritized by wallet score (score is unknown in this phase → by `added_at`) and budgeted.

**05.9 Webhook adapter (optional source)**
- Tests first: validates the auth header (constant time), parses recorded enhanced-webhook payloads (if Phase 03 recorded them) into the same `wallet.swap.detected` events with the same dedup keys, and stamps `receivedAt` at handler start.
- If the webhook was not verified or recorded in Phase 03: implement only the interface and mark it `NOT_VERIFIED` (source selection refuses it at config validation).

**05.10 ADR-0009 measurement (manual, real network)**
- Run `scripts/measure-wallet-latency.ts` for ≥ 2 hours against ≥ 10 active wallets with each available source. Record detection latency p50/p95 (slots and ms est.), credits per detected swap, and missed swaps (cross-checked with `getSignaturesForAddress` afterwards).
- Update ADR-0009 to `Accepted` with the chosen default and evidence.

**05.11 Engine wiring**
- The engine at boot loads TRACKED wallets and subscribes. Tracked-set changes apply live. Swaps flow into the bus and are persisted.

## 12. Acceptance Criteria
1. With a real key, the engine detects swaps of tracked wallets and persists `wallet.swap.detected` events with correct amounts (spot-check ≥ 10 against a block explorer; record the links).
2. Reconnect + gap backfill works (fake server test, plus a manual disconnect test on the real network).
3. Detection latency samples are persisted and summarized (p50/p95) in the verification note.
4. ADR-0009 is decided with evidence.
5. No swap event is ever emitted with fabricated amounts (tests for the unparsed path).

## 13. Tests
Unit (parsers, subscription packing, slot clock), contract (recorded fixtures), event-stream (fake WS server reconnect/gap), integration (persistence of swaps + gaps).

## 14. Failure Cases
- WS capacity exhausted → `WATCH_CAPACITY_EXCEEDED`, lower-priority wallets unsubscribed, logged.
- `getTransaction` returns null (not yet available at `confirmed`) → retry after 500 ms up to 3 times, then unparsed.
- Credit conserve mode → backfills suspended. Live decoding continues. Enrichment is limited to TRACKED wallets with a score ≥ threshold (scores arrive in Phase 09; until then, all TRACKED wallets).

## 15. Observability
Metrics: `paperbot_ws_subscriptions`, `paperbot_provider_up`, `paperbot_provider_reconnects_total`, `paperbot_detect_latency_slots`, `paperbot_helius_credits_used`. System events: `RECONNECT`, `GAP_DETECTED`, `GAP_RESOLVED`, `WATCH_CAPACITY_EXCEEDED`.

## 16. Security
- The API key is only in the WS/RPC URL, and it is redacted in logs.
- Webhook auth uses constant-time comparison. The payload size limit is 1 MB.
- The RPC client allowlist is unchanged.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- helius wallet-intel
pnpm test:integration --filter @paperbot/engine -- helius
HELIUS_API_KEY=... pnpm dev:engine     # observe swaps for seeded tracked wallets
pnpm tsx scripts/measure-wallet-latency.ts --minutes 120
```

## 18. Commit Strategy
1. `feat(db): add intel wallets table`
2. `feat(wallet-intel): add wallet registry and import`
3. `feat(ingestion): add helius subscription manager and slot clock`
4. `feat(ingestion): add wallet swap log decoding and balance-delta parser`
5. `feat(ingestion): add tx enricher and wallet swap normalizer`
6. `feat(ingestion): add reconnect gap backfill for wallets`
7. `feat(ingestion): add helius webhook adapter`
8. `docs: record ADR-0009 measurement and decision`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] ADR-0009 status is `Accepted`.
