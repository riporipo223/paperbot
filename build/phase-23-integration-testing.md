# Phase 23 — Integration, Failure & Performance Testing

| Field | Value |
|---|---|
| Milestone | M6 Production paper |
| Depends on | 21, 22 |
| Size | L |
| Requirements | NFR-REL-001..005, [23](../docs/23-testing-strategy.md) §5 and §7, success metrics in [01](../docs/01-product-spec.md) §7 (pre-validation) |

## 1. Objective
Consolidate and complete the test suites: the full failure-mode matrix (chaos), performance/load tests, the replay throughput benchmark, full E2E coverage, mutation testing turned into a gate for core domains, and a readiness review before deployment.

## 2. Context (read first)
- [23](../docs/23-testing-strategy.md) (entire)
- [03](../docs/03-system-requirements.md) §2, §4, §5
- [09](../docs/09-real-time-data-architecture.md) §9–14

## 3. Dependencies
Phases 21 and 22 DONE.

## 4. Inputs
All modules, the scenario harness, the fixture engine, and the E2E harness.

## 5. Outputs
- `apps/engine/test/chaos/*`: fault injection for every row of [23](../docs/23-testing-strategy.md) §5.
- `apps/engine/bench/*`: pipeline throughput, live-path latency under load, persister burst, replay throughput.
- `reports/perf/<date>.md` with results and host specs.
- Nightly CI workflow (mutation, full chaos, perf smoke). Mutation score gating for `economics`, `portfolio/domain`, `risk/domain` (≥ 70%).
- `build/verification/phase-23.md` including a readiness checklist.

## 6. Files To Create
```text
apps/engine/test/chaos/{api-outage.test.ts,ws-disconnect.test.ts,rate-limit.test.ts,duplicate-events.test.ts,missing-events.test.ts,out-of-order.test.ts,invalid-token.test.ts,invalid-pool.test.ts,stale-price.test.ts,liquidity-collapse.test.ts,db-outage.test.ts,ai-outage.test.ts,ai-timeout.test.ts,ai-malformed.test.ts,clock-drift.test.ts,frontend-disconnect.test.ts,server-restart.test.ts,partial-failure.test.ts}
apps/engine/test/harness/fault-injector.ts
apps/engine/bench/{pipeline-throughput.bench.ts,live-latency.bench.ts,persister-burst.bench.ts,replay-throughput.bench.ts,fake-ws-load-server.ts}
.github/workflows/nightly.yml
stryker.conf.json
```

## 7. Files To Modify
- CI: add the chaos subset to PR runs. The full set runs nightly.
- [22](../docs/22-observability-spec.md) §8 is not filled here (Phase 25 does that). Perf results go to `reports/perf/`.

## 8. Database Changes
None.

## 9. API Changes
None.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**23.1 Fault injector**
- Tests first: it can drop/delay/duplicate/reorder events, return HTTP errors/429s, disconnect WS, pause the DB (Testcontainers pause), shift the clock offset, and make AI time out or return malformed output.

**23.2 Chaos tests (one per failure row)**
- Each test asserts: (a) no fill on stale/unknown data, (b) no ledger imbalance, (c) the correct entry-pause/alert/system events, (d) recovery to a normal state when the fault clears, (e) trace completeness for any decisions made.

**23.3 Performance benchmarks**
- Implement and run the benchmarks in [23](../docs/23-testing-strategy.md) §7. Record the results. If the targets are missed, profile, fix, and re-run (each fix is a separate commit with before/after numbers).

**23.4 Long-run soak (recorded data)**
- Replay a recorded ≥ 24 h dataset at MAX speed with the full engine (Postgres mode). Assert no memory growth beyond a bound (heap snapshots at the start/middle/end), no dead letters beyond the baseline, and invariants holding.

**23.5 E2E completeness**
- Ensure the E2E suite covers: login, all screens render, a live position lifecycle, paper close, the decision trace, starting a replay, a config version creation, and the stale banner.

**23.6 Mutation testing gate**
- Configure Stryker for the three core domains. Raise the tests until ≥ 70%. Gate nightly (and PRs touching those folders).

**23.7 Readiness review**
- A checklist in the verification note: all golden scenarios, the chaos matrix, perf targets, the safety scans, secrets policy checks, trace audit = 1.0 on the soak run, and open issues triaged.

## 12. Acceptance Criteria
1. Every failure mode in [23](../docs/23-testing-strategy.md) §5 has a passing chaos test.
2. The performance targets are met or deviations documented with an ADR and mitigation.
3. The soak run shows no leaks, and invariants hold.
4. The mutation score is ≥ 70% on core domains.
5. The readiness checklist is complete.

## 13. Tests
Chaos, performance, soak, E2E, mutation.

## 14. Failure Cases
Discovering real defects is expected. Each defect is fixed test-first, with a regression test named after the failure.

## 15. Observability
The soak run is also a test of the observability: metrics and alerts behave correctly under faults (asserted in the chaos tests).

## 16. Security
Re-run the safety and secret scans. Verify there are no new network hosts beyond the allowlist in the soak logs.

## 17. Verification
```bash
pnpm test && pnpm test:integration && pnpm test:e2e
pnpm vitest run apps/engine/test/chaos
pnpm tsx apps/engine/bench/pipeline-throughput.bench.ts   # etc.
pnpm stryker run
```

## 18. Commit Strategy
1. `test: add fault injector`
2. `test(chaos): add <failure> scenario` (grouped logically)
3. `test(bench): add performance benchmarks`
4. `perf: <specific fix>` (as needed)
5. `ci: add nightly mutation, chaos and perf workflow`
6. `docs: phase 23 readiness review`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
