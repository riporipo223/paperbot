# Phase 17 — Replay & Backtesting

| Field | Value |
|---|---|
| Milestone | M4 Reproducible |
| Depends on | 15, 16 |
| Size | L |
| Requirements | FR-RPL-001..004 |

## 1. Objective
Run stored events through the unchanged pipeline with a simulated clock and a recorded AI client. Prove live-vs-replay equivalence and determinism, support both orderings, read cold archives, and provide parameter studies with the required sensitivity set and run comparison.

## 2. Context (read first)
- [19](../docs/19-backtesting-spec.md) (entire)
- [09](../docs/09-real-time-data-architecture.md) §2.4
- [13](../docs/13-ai-agent-architecture.md) §10
- [11](../docs/11-wallet-intelligence-spec.md) §8

## 3. Dependencies
Phases 15 and 16 DONE.

## 4. Inputs
`ReplayEventSource` (Phase 04 skeleton), RunManager, RecordedLlmClient, and as-of-time wallet scores.

## 5. Outputs
- `modules/replay`: `ReplayDriver` (clock advance + timer firing + paging/prefetch), `ReplayRunFactory` (a module graph without ingestion + network guard), `DeterminismVerifier`, `ArchiveReader` (NDJSON.gz), and `StudyRunner`.
- CLI: `pnpm replay`, `pnpm replay:study`, `pnpm replay:verify`.
- Reports: `reports/<study-id>.md` generation.
- `/api/v1/analytics/compare` backing query (the endpoint itself comes in Phase 18; the query service is built here).

## 6. Files To Create
```text
apps/engine/src/modules/replay/
  index.ts replay-driver.ts replay-run-factory.ts determinism-verifier.ts archive-reader.ts study-runner.ts report-writer.ts
  __tests__/...
apps/engine/src/modules/portfolio/compare.ts        # run comparison query service
apps/engine/src/cli/{replay.ts,replay-study.ts,replay-verify.ts}
apps/engine/test/replay/{live-vs-replay.test.ts,timers.test.ts,as-of-scores.test.ts,archive.test.ts,network-guard.test.ts}
```

## 7. Files To Modify
- `modules/pipeline/adapters/replay-event-source.ts` (ordering modes, archive integration, upcasters)
- `bootstrap/composition-root.ts` (replay composition variant)

## 8. Database Changes
None (runs with `source=REPLAY` use existing columns).

## 9. API Changes
None (Phase 18).

## 10. Environment Variables
None (replay has no provider access).

## 11. Implementation Tasks

**17.1 ReplayDriver**
- Tests first: advancing the SimulatedClock to each event's replay timestamp fires due timers first. `AS_RECEIVED` and `BY_EVENT_TIME` ordering. Prefetch paging with bounded memory. At the end, advance to `replay.to` and complete the run.

**17.2 Replay run factory + network guard**
- Tests first: the replay graph has no ingestion module. Any socket attempt throws (guard). Wallet scores resolve as of time (or frozen at start).

**17.3 Live-vs-replay equivalence**
- Tests first: feed a fixture stream through the live path (fake adapters + SystemClock under fake timers) and persist events. Replay with the same seed and SOURCE_RUN AI cache. Decision/fill content hashes are identical (per [19](../docs/19-backtesting-spec.md) §6).

**17.4 DeterminismVerifier**
- Tests first: identical runs → no divergence. An injected nondeterminism (e.g. unordered map iteration in a test double) → reports the first divergence with both records.

**17.5 ArchiveReader**
- Tests first: write a day's partition to NDJSON.gz (with the archiver from Phase 02's callback, now implemented), read it back, and verify equal envelopes and equal replay results versus the warm table.

**17.6 StudyRunner + sensitivity set**
- Tests first: generates config versions for the vary grid (immutable, content-addressed), runs them sequentially (or with bounded parallelism in separate processes), enforces the train/held-out ranges, and writes a report with a metrics table and the number of configurations tried.
- Implements the required sensitivity preset `--preset validation` ([19](../docs/19-backtesting-spec.md) §7).

**17.7 Run comparison service**
- Tests first: metrics for N runs side by side from `computePerformance`. Differences are highlighted. Coverage caveat notes are included when the capture modes differ.

**17.8 Replay performance**
- Benchmark: replay a recorded day (from the Phase 15 live run) at MAX speed. Record the throughput vs real time in the verification note (target ≥ 20×).

## 12. Acceptance Criteria
1. Live-vs-replay equivalence test passes in CI.
2. `pnpm replay:verify -- --run <live-run-id>` on the Phase 15 24 h run shows no divergence (or only the documented overhead caveat, flagged).
3. Studies generate reports with held-out metrics.
4. Replay makes zero network calls.

## 13. Tests
Replay tests, archive tests, determinism tests, and the benchmark.

## 14. Failure Cases
- Missing events in range (retention) without an archive → the replay refuses to start with `RPL_DATA_UNAVAILABLE` and lists the missing days.
- A schema version without an upcaster → `RPL_SCHEMA_UNSUPPORTED`.

## 15. Observability
Replay progress logs (events/s, simulated time, ETA). The `replay_wall_us` metric.

## 16. Security
The network guard and the absence of the ingestion module are both tested.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- replay
pnpm replay:verify -- --run <phase15-run-id>
pnpm replay:study -- --base <configId> --preset validation --train 2026-..,2026-.. --holdout 2026-..,2026-..
```

## 18. Commit Strategy
1. `feat(replay): add replay driver with timer-accurate clock`
2. `feat(replay): add isolated replay run factory with network guard`
3. `test(replay): add live-vs-replay equivalence`
4. `feat(replay): add determinism verifier`
5. `feat(replay): add archive reader and writer`
6. `feat(replay): add study runner, sensitivity preset and reports`
7. `feat(portfolio): add run comparison service`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M4 marked.
