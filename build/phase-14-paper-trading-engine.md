# Phase 14 — Paper Trading Engine

| Field | Value |
|---|---|
| Milestone | M3 Headless paper trading |
| Depends on | 11, 12, 13 |
| Size | L |
| Requirements | FR-EXE-001..006, FR-ECO-002, FR-SAF-003 (executor boundary) |

## 1. Objective
Implement `TradingExecutor` with the operational `PaperExecutor` (order lifecycle, write-ahead persistence, latency scheduling, execution against pool state at simulated execution time, fills with full economics, ledger posting, idempotency, restart recovery). Also the `ShadowExecutor` and `RealExecutor` stubs and the `ExecutorFactory` mode enforcement.

## 2. Context (read first)
- [16](../docs/16-paper-trading-spec.md) (entire)
- [17](../docs/17-economics-engine-spec.md) §5–7, §10
- [18](../docs/18-portfolio-accounting-spec.md) §4, §6
- [09](../docs/09-real-time-data-architecture.md) §7 (FILL freshness)
- [21](../docs/21-security-spec.md) §2–3

## 3. Dependencies
Phases 11, 12 and 13 DONE.

## 4. Inputs
Economics, portfolio (ledger, reservations), market state (`poolStateAt`, quality for FILL), the scheduler, forked RNG, and the decision/risk tables.

## 5. Outputs
- Migration `0012_sim_orders_fills.sql` (orders, order_transitions, fills, transition trigger).
- `modules/execution`: `TradingExecutor` interface, `PaperExecutor`, `ShadowExecutor` (throws unless `features.shadow_enabled`; the body is implemented in Phase 26), `RealExecutor` (always throws), `ExecutorFactory`, `OrderStateMachine`, and `RecoveryService`.
- Events: `order.created`, `order.submitted`, `order.filled`, `order.failed`.

## 6. Files To Create
```text
packages/db/migrations/0012_sim_orders_fills.sql
apps/engine/src/modules/execution/
  index.ts ports.ts
  executor.ts                 # TradingExecutor interface
  paper-executor.ts shadow-executor.ts real-executor.ts executor-factory.ts
  domain/{order-state-machine.ts,idempotency.ts,types.ts}
  recovery.ts
  adapters/{order-repository.ts,fill-repository.ts}
  __tests__/{order-state-machine.test.ts,paper-executor.test.ts,executor-factory.test.ts,recovery.int.test.ts,paper-executor.int.test.ts}
```

## 7. Files To Modify
- `.dependency-cruiser.cjs`: `execution` cannot import network clients, `@solana/*` send/sign modules, or the ingestion module.
- `scripts/check-safety.ts`: add a rule that no file under `modules/execution` references `fetch`, `undici`, or `ws`.

## 8. Database Changes
`0012_sim_orders_fills.sql` per [07](../docs/07-database-schema.md) §3.5, plus a trigger that rejects illegal `sim.orders.status` transitions.

## 9. API Changes
None.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**14.1 Order state machine**
- Tests first: the legal transitions per [16](../docs/16-paper-trading-spec.md) §4 are accepted and all others rejected (app + DB trigger integration test).

**14.2 ExecutorFactory and stubs**
- Tests first: SIMULATION → PaperExecutor. SHADOW → throws unless the flag is on. LIVE → `SAF_MODE_NOT_ALLOWED`. Every `RealExecutor` method throws `SAF_LIVE_DISABLED`. A static test asserts the factory source has no reference to `RealExecutor`.

**14.3 submit(): idempotency, reservation, write-ahead, scheduling**
- Tests first (SimulatedClock + fakes): a duplicate idempotency key returns the same handle. Insufficient funds → FAILED(INSUFFICIENT_FUNDS) with no scheduling. On success, CREATED→SUBMITTED is persisted before scheduling. The latency is drawn from `rng.fork('latency:'+orderId)`. `scheduled_exec_at = decided_at + overhead + L`.

**14.4 execute(): fill against state at execution time**
- Tests first:
  - The state used is `poolStateAt(pool, scheduled_exec_at)`. The fill reflects the price **at execution**, not at decision (fixture with a price change during latency).
  - Non-FRESH state → FAILED(STALE_MARKET_DATA), no fees, reservation released (FR-EXE-004).
  - Slippage breach → FAILED(SLIPPAGE_EXCEEDED) with a fee-only ledger transaction.
  - Liquidity collapse → INSUFFICIENT_LIQUIDITY/SLIPPAGE_EXCEEDED, never a fill at the old price.
  - A double timer fire is idempotent.
  - The fill row contains every field from [16](../docs/16-paper-trading-spec.md) §6. `market.pool_states` marks `used_for_fill`.
  - A single DB transaction covers order status + fill + ledger + position.

**14.5 ATA handling**
- Tests first: the first buy of a mint locks rent. A failed first buy locks nothing. A full exit refunds rent when `close_ata_on_full_exit`. A partial exit keeps it.

**14.6 Recovery**
- Tests first (integration): each row of [16](../docs/16-paper-trading-spec.md) §9, including execution against a historical FRESH state within the grace period. The recovery report is logged as a system event.

**14.7 Safety scan and boundary rules**
- Tests first: the scanner flags a fixture file under `modules/execution` that imports `undici`. Depcruise flags `execution` → `ingestion`.

## 12. Acceptance Criteria
1. Fills are computed at the simulated execution time from FRESH state only (tests S02, S04-like at the unit level).
2. Every fill reconstructs signal→decision→execution prices and costs.
3. Idempotency and recovery are proven.
4. LIVE/SHADOW are impossible through the factory.
5. The executor has no network access (static rules + tests).

## 13. Tests
Unit (state machine, factory), component (executor with fakes), integration (DB transactionality, trigger, recovery).

## 14. Failure Cases
Covered above. A DB failure during execute → the transaction rolls back and the order stays SUBMITTED. Retry at the next scheduler tick (with backoff). If it is still unresolved after `recovery_grace_ms`, treat it per the recovery rules.

## 15. Observability
Metrics: `paperbot_orders_total{side,status,failure_reason}`, `L6_order`, `L7_fill_compute`, `L8_portfolio`. Logs: `order.*` with the latency draw and cost summary.

## 16. Security
This phase is where fake fills or real execution could leak in. The static bans, factory tests and FRESH-only rule are all mandatory.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- execution
pnpm test:integration --filter @paperbot/engine -- execution
pnpm check:safety
```

## 18. Commit Strategy
1. `feat(db): add orders, fills and transition trigger`
2. `feat(execution): add order state machine and executor factory with stubs`
3. `feat(execution): add paper executor submit with write-ahead and scheduling`
4. `feat(execution): add execution against state at simulated time`
5. `feat(execution): add ata rent handling`
6. `feat(execution): add restart recovery`
7. `chore: tighten execution boundary rules`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
