# Phase 12 — Portfolio Accounting

| Field | Value |
|---|---|
| Milestone | M3 Headless paper trading |
| Depends on | 02, 11 |
| Size | L |
| Requirements | FR-PFL-001..004 |

## 1. Objective
Implement the double-entry ledger, positions with average-cost PnL, reservations, marks, portfolio snapshots, invariant checks with fail-safe halting, restart rebuild, and the performance analytics calculator.

## 2. Context (read first)
- [18](../docs/18-portfolio-accounting-spec.md) (entire)
- [07](../docs/07-database-schema.md) §3.5 (ledger, positions, snapshots)
- [17](../docs/17-economics-engine-spec.md) §10 (ExecutionResult shape)
- ADR-0006, ADR-0013, ADR-0014

## 3. Dependencies
Phases 02 and 11 DONE.

## 4. Inputs
`ExecutionResult` type, the DB, and `MarketStateView` (marks).

## 5. Outputs
- Migration `0010_sim_ledger_positions.sql` (ledger transactions/entries + balance constraint trigger, positions, position_transitions, portfolio_snapshots).
- `modules/portfolio`: `Journal` templates (pure), `LedgerService` (transactional posting + invariants), `PositionBook` (pure average-cost logic), `ReservationBook`, `MarkService`, `SnapshotService`, `PortfolioSummary` read model, `computePerformance` (pure analytics), and `rebuildFromLedger`.
- Events: `position.opened`, `position.updated`, `position.closed`, `portfolio.updated`, `run.halted` (on invariant violation).

## 6. Files To Create
```text
packages/db/migrations/0010_sim_ledger_positions.sql
apps/engine/src/modules/portfolio/
  index.ts ports.ts service.ts
  domain/{journal.ts,accounts.ts,position-book.ts,reservations.ts,invariants.ts,marks.ts,performance.ts,types.ts}
  adapters/{ledger-repository.ts,position-repository.ts,snapshot-repository.ts}
  __tests__/{journal.test.ts,position-book.test.ts,invariants.test.ts,performance.test.ts,service.int.test.ts,rebuild.int.test.ts}
```

## 7. Files To Modify
- Temporary `RunContext` (Phase 10) gains `fundRun()`, which posts `RUN_FUNDING` at run creation.

## 8. Database Changes
`0010_sim_ledger_positions.sql` per [07](../docs/07-database-schema.md) §3.5 (excluding orders/fills, which come in Phase 14). Includes:
- a deferred constraint trigger verifying Σ amount per (transaction, asset) = 0 at commit
- the unique partial index for one open position per token per run

## 9. API Changes
None.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**12.1 Chart of accounts + journal templates (pure)**
- Tests first (property): `journalFor` for RUN_FUNDING, FILL BUY, FILL SELL (partial/full, with/without ATA close), and FAILED_TX_FEE produces entries balanced per asset for random valid amounts. Signs follow the convention.

**12.2 PositionBook (pure)**
- Tests first (golden + property): open, add, partial sell (floor released basis), full close (residual basis to realized). Realized PnL exactness over sequences. MFE/MAE updates.

**12.3 Reservations**
- Tests first: reserve/release per order. `free_cash = cash − reserved`. Reservation beyond free cash → error. Rebuild from SUBMITTED orders (the order table arrives in Phase 14; the interface is tested with a fake).

**12.4 LedgerService (transactional)**
- Tests first (integration): posting writes the transaction + entries atomically. The DB trigger rejects an unbalanced transaction (a direct SQL test). The idempotency unique `(ref_type, ref_id, kind)` holds. Invariants from [18](../docs/18-portfolio-accounting-spec.md) §9 are checked after each post. A simulated violation → run `HALTED_INVARIANT` + a FATAL system event + a `run.halted` event, and further postings are refused.

**12.5 applyExecution**
- Tests first: given an `ExecutionResult` (FILLED/FAILED), in one DB transaction: post the ledger, update the position (open/close transitions), release the reservation, and emit events after commit.

**12.6 Marks and snapshots**
- Tests first (SimulatedClock): LIQUIDATION mark via an economics quote on FRESH state. MID mark. Stale state → last mark with `marks_quality=STALE`. Snapshots on interval, fill, run start and run end with all fields. Peak equity and drawdown tracking.

**12.7 PortfolioSummary read model**
- Tests first: cash, reserved, free, rent locked, equity SOL/USD, exposure, realized/unrealized, ROI (SOL and USD), open positions, daily realized PnL (UTC day), consecutive losses, entries in the last hour. This is used by risk in Phase 13.

**12.8 computePerformance (pure)**
- Tests first: a synthetic trade set with hand-computed metrics (all in [18](../docs/18-portfolio-accounting-spec.md) §7), the edge cases (no losses → PF n/a; no trades), and the sample-size warning.

**12.9 Rebuild from ledger**
- Tests first (integration): post a sequence, discard memory, rebuild → equal cash, positions and realized PnL.

## 12. Acceptance Criteria
1. Ledger balance is enforced by the app and the DB (tests).
2. The invariant violation halts the run (test).
3. Golden round trip: fund $20 → buy → partial sell → full sell gives exact expected balances and PnL.
4. Rebuild equality.
5. Analytics match hand calculations.

## 13. Tests
Unit/property (journal, position book, performance), integration (ledger service, trigger, rebuild).

## 14. Failure Cases
- DB error mid-post → the transaction is rolled back, the in-memory state is unchanged, and the caller gets the error (the executor retries per its policy).
- A negative cash attempt → invariant violation → halt (should be impossible given reservations; the test proves the halt path).

## 15. Observability
Metrics: `paperbot_equity_usd`, `paperbot_equity_sol`, `paperbot_drawdown_ratio`. Logs: `position.*`, `ledger.invariant.violation` (fatal).

## 16. Security
None specific.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- portfolio
pnpm test:integration --filter @paperbot/engine -- portfolio
```

## 18. Commit Strategy
1. `feat(db): add ledger, positions and snapshots tables`
2. `feat(portfolio): add chart of accounts and journal templates`
3. `feat(portfolio): add position book with average cost pnl`
4. `feat(portfolio): add ledger service with invariants and halt`
5. `feat(portfolio): add marks, snapshots and summary`
6. `feat(portfolio): add performance analytics`
7. `feat(portfolio): add rebuild from ledger`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
