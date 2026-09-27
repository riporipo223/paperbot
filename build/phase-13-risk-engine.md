# Phase 13 — Risk Engine (+ decision record tables)

| Field | Value |
|---|---|
| Milestone | M3 Headless paper trading |
| Depends on | 08, 12 |
| Size | M |
| Requirements | FR-RSK-001..004 |

## 1. Objective
Implement the deterministic risk engine: every check in the catalog, entry vs exit semantics, size reduction, and pause-reason signalling, all persisted as auditable risk decisions. Also create the `strategy.decisions` table now, because orders (Phase 14) reference both decisions and risk decisions.

## 2. Context (read first)
- [15](../docs/15-risk-engine-spec.md) (entire)
- [17](../docs/17-economics-engine-spec.md) §4.3, §8
- [18](../docs/18-portfolio-accounting-spec.md) §6–7 (summary inputs)
- [28](../docs/28-decision-logging-spec.md) §3, §6

## 3. Dependencies
Phases 08 and 12 DONE.

## 4. Inputs
`PortfolioSummary`, the `MarketStateView` quality snapshot, reference safety snapshots, economics quotes, and config `risk.*`/`sizing.*`.

## 5. Outputs
- Migration `0011_risk_decisions.sql` (`strategy.risk_decisions`, `strategy.decisions` without the AI FK).
- `modules/risk`: `evaluate()` (pure), individual check functions, the reduction algorithm, the `RiskService` (persist + emit `risk.evaluated`), and a pause-reason publisher (`DAILY_LOSS_LIMIT`, `DRAWDOWN_KILL`, `CONSECUTIVE_LOSS_COOLDOWN`).
- `DecisionRepository` in `modules/strategy/adapters` (the table exists now; the DecisionEngine logic comes in Phase 15).

## 6. Files To Create
```text
packages/db/migrations/0011_risk_decisions.sql
apps/engine/src/modules/risk/
  index.ts ports.ts service.ts
  domain/{evaluate.ts,checks/entry/*.ts,checks/exit/*.ts,checks/token/*.ts,reduction.ts,types.ts}
  adapters/risk-decision-repository.ts
  __tests__/{checks.table.test.ts,reduction.test.ts,evaluate.test.ts,properties.test.ts,service.int.test.ts}
apps/engine/src/modules/strategy/adapters/decision-repository.ts
```

## 7. Files To Modify
None outside the new modules.

## 8. Database Changes
`0011_risk_decisions.sql` per [07](../docs/07-database-schema.md) §3.4 (the `strategy.decisions.ai_output_id` column exists without its FK, which is added in migration 0013).

## 9. API Changes
None.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**13.1 Check framework**
- Tests first: each check returns `{code, passed, value, limit, severity, maxAllowedSize?}`. `evaluate` runs **all** checks even after failures.

**13.2 Entry checks (table-driven)**
- Tests first: every `RSK_*` entry check in [15](../docs/15-risk-engine-spec.md) §3 at its boundary (pass at the limit, fail just beyond), with values recorded.

**13.3 Token safety checks**
- Tests first: each check in [15](../docs/15-risk-engine-spec.md) §4, including the unknown holder data with `require_holder_data=false` → pass with a flag, and a stale safety observation → fail.

**13.4 Exit checks**
- Tests first: exits skip the exposure/daily-loss/frequency/cost-ratio checks. Stale data → REJECTED with `retryable=true`. The slippage cap reduces the tolerance.

**13.5 Reduction algorithm**
- Tests first: the minimum of the max-allowed sizes. The impact-based max via `economics.maxInputForImpact`. Below the minimum order → REJECTED. `APPROVED_REDUCED` records `reducedTo`.

**13.6 Pause signalling**
- Tests first: the daily loss breach emits the `DAILY_LOSS_LIMIT` pause reason (it clears at the UTC day boundary on the simulated clock). Drawdown kill → `DRAWDOWN_KILL` (manual clear only). The consecutive-loss cooldown clears after the configured minutes.

**13.7 RiskService**
- Tests first (integration): persists `strategy.risk_decisions` with checks, failed codes, portfolio state and model version `risk@1.0.0`. Emits `risk.evaluated`.

**13.8 Property tests**
- Random portfolios/orders: the approved size never exceeds any limit. Exit approval is never blocked by entry-only checks.

## 12. Acceptance Criteria
1. All checks are implemented and boundary-tested.
2. Risk decisions persist complete check lists.
3. The pause reasons behave as specified.
4. The decisions table exists and is ready for Phase 14/15.

## 13. Tests
Unit/table/property, integration (service persistence).

## 14. Failure Cases
- Missing input (e.g. no safety observation) → the corresponding check fails with `value=null` and reason `MISSING_INPUT` (never passes by default), except those explicitly configured to pass with a flag.

## 15. Observability
Metrics: `paperbot_risk_rejections_total{code}`, `L5_risk` latency. Log `risk.evaluated` (info) with the outcome and failed codes.

## 16. Security
Risk has no dependency on AI (depcruise rule: `risk` must not import `ai`).

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- risk
pnpm test:integration --filter @paperbot/engine -- risk
```

## 18. Commit Strategy
1. `feat(db): add risk decisions and decisions tables`
2. `feat(risk): add check framework and entry checks`
3. `feat(risk): add token safety and exit checks`
4. `feat(risk): add size reduction and pause signalling`
5. `feat(risk): add risk service persistence`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
