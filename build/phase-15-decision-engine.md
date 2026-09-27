# Phase 15 — Decision Engine & Strategy Runtime

| Field | Value |
|---|---|
| Milestone | M3 Headless paper trading |
| Depends on | 10, 14 |
| Size | L |
| Requirements | FR-DEC-001..003, FR-RSK-004, FR-LOG-001 (records), FR-LOG-002, FR-CFG-003 |

## 1. Objective
Wire the full SIMULATION loop: RunManager (replacing the temporary RunContext), WatchSetPolicy, DecisionEngine (`decision@1.0.0`), OrderPlanner, ExitManager (`exit@1.0.0`), the entry-pause aggregator, deterministic explanation templates, and an AI port with a no-op implementation. The golden end-to-end scenarios S01–S18 (those not requiring the API/dashboard) pass headless. This reaches milestone M3.

## 2. Context (read first)
- [14](../docs/14-strategy-engine-spec.md) (entire)
- [28](../docs/28-decision-logging-spec.md) (entire)
- [23](../docs/23-testing-strategy.md) §4 (scenario harness and the golden list)
- [10](../docs/10-market-data-spec.md) §3 (watch-set policy)
- [13](../docs/13-ai-agent-architecture.md) §3, §6 (AI port contract only)

## 3. Dependencies
Phases 10 and 14 DONE.

## 4. Inputs
Wave signals, risk, the executor, portfolio, market state, reference, and the scheduler.

## 5. Outputs
- `modules/strategy`: `RunManager`, `WatchSetPolicy`, `DecisionEngine` (pure), `OrderPlanner`, `ExitManager` (pure rules + service), `EntryPause`, `explanations.ts` (templates), and an `AiAdvisor` port + `NoopAiAdvisor`.
- Removal of the temporary `RunContext` and the direct wave→ingestion watch calls (now via WatchSetPolicy).
- A scenario harness `createTestEngine()` and golden scenario tests.
- CLI: `pnpm run:start -- --config <id>` / `run:stop` (until the API exists).

## 6. Files To Create
```text
apps/engine/src/modules/strategy/
  index.ts ports.ts service.ts
  run-manager.ts watch-set-policy.ts order-planner.ts entry-pause.ts
  domain/{decision-engine.ts,exit-rules.ts,exit-plan.ts,explanations.ts,reason-codes.ts,types.ts}
  ai-port.ts noop-ai-advisor.ts
  __tests__/{decision-engine.table.test.ts,exit-rules.test.ts,explanations.test.ts,entry-pause.test.ts,run-manager.int.test.ts}
apps/engine/test/harness/{create-test-engine.ts,assertions.ts,network-guard.ts}
apps/engine/test/scenarios/{s01-take-profit.test.ts, s02-liquidity-vanishes.test.ts, ..., s18-unsupported-pool.test.ts}
fixtures/scenarios/s01..s18/{events.ndjson,expected.json,README.md}
apps/engine/src/cli/{run-start.ts,run-stop.ts,trace-audit.ts}
apps/engine/src/modules/strategy/trace/{trace-builder.ts,narrative.ts,__tests__/trace-builder.int.test.ts,__tests__/narrative.test.ts}
```

## 7. Files To Modify
- Delete `apps/engine/src/modules/strategy/run-context.ts` (temporary). Update wave to use RunManager.
- `modules/wave/service.ts` → emits watch intents to WatchSetPolicy instead of calling ingestion directly.
- `bootstrap/composition-root.ts` → the full module graph.

## 8. Database Changes
None (uses existing tables). Adds `strategy.runs` lifecycle updates.

## 9. API Changes
None (CLI only).

## 10. Environment Variables
None new.

## 11. Implementation Tasks

**15.1 Reason codes + explanation templates**
- Tests first: every decision/exit code has a template, every template parameter is supplied by its builder, and the output is stable (snapshot).

**15.2 DecisionEngine (pure)**
- Tests first (table-driven): each rule in [14](../docs/14-strategy-engine-spec.md) §3.2 alone and in priority combinations. The AI modes OFF/ADVISORY/VETO with the timeout behaviours (using the port's result shape). The output includes the reason codes, explanation and `ai_effect`.

**15.3 OrderPlanner**
- Tests first: size in lamports from USD/SOL price (floor). The priority fee mode FIXED/ESTIMATE (the estimate uses a port with a cached value; fallback to FIXED). Quote via economics on the decision-time state. `min_out` computed. Exit orders use the position quantity/fraction with the remainder rule.

**15.4 EntryPause aggregator**
- Tests first: manual + automatic reasons. The clear conditions per reason ([14](../docs/14-strategy-engine-spec.md) §7). Events `entries.paused`/`entries.resumed`. Wiring from: risk pause signals, pipeline backlog, module circuit, clock drift, SOL/USD stale, wallet stream down, watch capacity critical, credit budget.

**15.5 ExitManager**
- Tests first (SimulatedClock): each rule fires at its threshold. Priority resolution. Scale-out then trailing. Price rules are suppressed on stale price. One exit in flight per position. Failed exit handling (emergency slippage escalation, stale retry, tx failure retry with a new idempotency leg). The exit plan is frozen at entry.

**15.6 WatchSetPolicy**
- Tests first: P0 for open positions, P1 for active waves, release + linger, capacity handling (release the lowest-score waves first), and `WATCH_CAPACITY_CRITICAL` → entry pause.

**15.7 RunManager**
- Tests first (integration): start run (validation, funding, seed, versions, `data_sources` snapshot incl. the engine config hash), a single LIVE_FEED run constraint, stop run (graceful, optional close positions), restart recovery (rebuild portfolio, reload exit plans, executor recovery, resubscribe the watch set), and status transitions including HALTED_*.

**15.8 Strategy service wiring (entry flow)**
- Tests first (scenario): signal → flush barrier → (AI port) → decision → plan → risk → persist decision + risk (write-ahead) → executor submit. Wave transitions ENTERED/REJECTED. `system_latency_us` is recorded. The trace chain is complete (`expectTraceComplete`).

**15.9 Scenario harness**
- Implement `createTestEngine` per [23](../docs/23-testing-strategy.md) §4 (in-memory and Postgres modes), plus the network guard.

**15.10 Golden scenarios S01–S18**
- Author the fixtures with the DSL (and hand-computed `expected.json` for S01 PnL). Implement the tests. S11 uses the `FakeAiAdvisor` implementing the port (the real AI comes in Phase 16).

**15.11 Trace builder and audit CLI**
- Tests first: `buildDecisionTrace(decisionId)` assembles the structure in [28](../docs/28-decision-logging-spec.md) §4 from the DB (integration). The deterministic `narrative` lines match a snapshot for S01. The completeness check lists missing links. `pnpm trace:audit -- --run <id>` exits non-zero if any decision is incomplete.
- The API endpoint in Phase 18 reuses this builder.

**15.12 Headless live run**
- Run the engine against live data in SIMULATION for ≥ 24 h. Record the trades (if any), the rejections by reason, the latency percentiles, and any issues in the verification note.

## 12. Acceptance Criteria
1. Golden scenarios S01–S18 pass (in-memory mode; the S01, S02, S04, S14 and S16 subset also in Postgres mode).
2. Every decision has a complete trace (`pnpm trace:audit` on scenario runs → 1.0).
3. The live headless run completes 24 h without crash. All entries/exits are paper-only, with the fill/latency/cost fields populated.
4. The temporary RunContext is removed. Wave no longer calls ingestion directly.

## 13. Tests
Unit/table (decision, exits, explanations, pause), integration (run manager, recovery), scenario (golden set).

## 14. Failure Cases
All from [23](../docs/23-testing-strategy.md) §5 that concern the trading loop are covered by the scenarios.

## 15. Observability
Metrics: `paperbot_decisions_total{action}`, `paperbot_entries_paused{reason}`, `e2e_system`, `L4_decision`. Logs: `decision.made`, `entries.paused`, `run.*`.

## 16. Security
The AI port is veto-only by type: `AiAdvisory.verdict` is an enum, and DecisionEngine code can only map VETO → REJECT (a test asserts that no other path reads AI fields).

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- strategy
pnpm test --filter @paperbot/engine -- scenarios
pnpm test:integration --filter @paperbot/engine -- strategy scenarios
pnpm trace:audit -- --run <scenario-run-id>
```

## 18. Commit Strategy
1. `feat(strategy): add reason codes and explanation templates`
2. `feat(strategy): add decision engine v1`
3. `feat(strategy): add order planner and entry pause`
4. `feat(strategy): add exit manager v1`
5. `feat(strategy): add watch set policy`
6. `feat(strategy): add run manager with restart recovery`
7. `test: add scenario harness and golden scenarios s01-s18`
8. `docs: phase 15 verification (M3 headless run)`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M3 marked.
