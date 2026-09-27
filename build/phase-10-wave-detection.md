# Phase 10 — Wave Detection

| Field | Value |
|---|---|
| Milestone | M2 Signals |
| Depends on | 08, 09 |
| Size | L |
| Requirements | FR-WAV-001..004 |

## 1. Objective
Implement the wave engine: per-token state machines seeded by smart buys, the `wave-features@1.0.0` computation, gates, `wave-score@1.0.0`, terminal conditions, and persisted, traceable ENTRY signals behind the flush barrier. No trading yet.

## 2. Context (read first)
- [12](../docs/12-wave-detection-spec.md) (entire)
- [09](../docs/09-real-time-data-architecture.md) §7–9 (quality, lateness, flush barrier)
- [28](../docs/28-decision-logging-spec.md) §3 (signal contents)
- [07](../docs/07-database-schema.md) §3.4 (waves, transitions, signals)

## 3. Dependencies
Phases 08 and 09 DONE.

## 4. Inputs
`smart.buy.detected`/`smart.sell.detected`, `MarketStateView`, the reference primary pool, the scheduler, and the persister flush barrier.

## 5. Outputs
- Migration `0009_waves_signals.sql`.
- `modules/wave`: `WaveFeatures` (pure), `WaveGates` (pure), `WaveScore` (pure), `WaveStateMachine` (pure transitions), `WaveService` (orchestration, ticks, persistence), and repositories.
- A temporary run context: until Phase 15's RunManager, the wave service needs a `runId`. Introduce a minimal `RunContext` port (`currentRunId()`), with a dev implementation that creates a normal `SIMULATION`/`LIVE_FEED` run row at boot (funding and portfolio come in later phases). Phase 15 replaces it with RunManager.
- Watch requests: on `SEEDED`, wave requests `watchPool(pool, P1)` via a port (temporary direct call; Phase 15 moves this to the WatchSetPolicy).
- Events: `wave.updated`, `signal.emitted`.

## 6. Files To Create
```text
packages/db/migrations/0009_waves_signals.sql
apps/engine/src/modules/wave/
  index.ts ports.ts service.ts
  domain/{features.ts,gates.ts,score.ts,state-machine.ts,windows.ts,types.ts}
  adapters/{wave-repository.ts,signal-repository.ts}
  __tests__/{features.test.ts,gates.test.ts,score.test.ts,state-machine.test.ts,service.test.ts}
apps/engine/src/modules/strategy/run-context.ts          # temporary dev RunContext (replaced in 15)
apps/engine/test/scenarios/builders/scenario-dsl.ts       # scenario builder (used from here on)
fixtures/scenarios/wave-*.ndjson + expected.json
```

## 7. Files To Modify
- `packages/core` event types (internal `wave.updated`, `signal.emitted` payload types)

## 8. Database Changes
`0009_waves_signals.sql`: `strategy.waves`, `strategy.wave_transitions`, `strategy.signals` per [07](../docs/07-database-schema.md) §3.4.

## 9. API Changes
None.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**10.1 Scenario DSL**
- Tests first: the DSL builds a deterministic `EventEnvelope[]` with `receivedAt` offsets, slots, wallets with scores, pool states and trades. The output validates against the schemas.
- Implement `scenario()` builder per [23](../docs/23-testing-strategy.md) §3.

**10.2 Windows helper**
- Tests first: slot/time-based window extraction from the trade buffer, non-overlapping `W_short`/`W_base`, and since-seed.

**10.3 Features (pure)**
- Tests first: each feature in [12](../docs/12-wave-detection-spec.md) §4 on a crafted input with hand-computed expected values. Nulls where the inputs are missing. Per-input quality/age captured. `data_gap_overlap` detection.

**10.4 Gates (pure)**
- Tests first: each gate passes at the limit and fails beyond it. The result list contains all gates with value/limit.

**10.5 Score (pure)**
- Tests first: each component formula at key points. Weights from config. Breakdown sums to the score. Golden score for the "classic wave" fixture.

**10.6 State machine (pure)**
- Tests first: every transition in [12](../docs/12-wave-detection-spec.md) §5, including expiry via ticks, hard invalidations, the reseed cooldown, and one active wave per token per run.

**10.7 WaveService**
- Tests first (scenario, SimulatedClock): seeding on `smart.buy.detected` (with the pool resolution wait/timeout). Ticks every `wave.tick_ms` for active waves. Transitions persisted with features. On CONFIRMED → flush barrier (`flushUpTo(causationWatermark)`) → persist the signal → emit `signal.emitted`, exactly once per wave. Late events never retro-confirm. Pool watch requested on seed and released on terminal state (+linger).

**10.8 Golden scenarios**
- Implement fixtures and tests: classic wave (confirm), single whale (expire), over-extended (G_EXTENSION), smart dump (invalidated), liquidity pull (hard invalidation), stale stream (no confirm), and late event (no retro effect).

**10.9 Real-data dry run**
- Run the engine against live data for ≥ 6 hours with seeded wallets. Record wave counts by terminal state and example signals (IDs) in the verification note. No trading occurs (no executor exists yet).

## 12. Acceptance Criteria
1. All golden wave scenarios pass with exact expected transitions and scores.
2. Signals are persisted with complete feature snapshots, quality, trigger event IDs and a causation watermark. Every trigger event ID exists in `market.events` (flush barrier test).
3. Replay-order independence within lateness tolerance (test).
4. The live dry run produces waves and signals on real data (evidence recorded).

## 13. Tests
Unit (pure domain), scenario (service + SimulatedClock), integration (repositories, flush barrier with the real persister).

## 14. Failure Cases
- Pool resolution timeout → wave `REJECTED:NO_SUPPORTED_POOL`.
- Market data stale during `BUILDING` → cannot confirm. Expiry continues.
- Persistence failure at signal commit → the signal is not emitted (retry via the persister backlog policy). The wave stays CONFIRMED-pending and is logged.

## 15. Observability
Metrics: `paperbot_waves_active{state}`, `paperbot_signals_total{kind}`, `L3_signal` latency. Logs: `wave.transition` (info), `signal.emitted` (info with score and key features).

## 16. Security
None specific.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- wave
pnpm test:integration --filter @paperbot/engine -- wave
```
Plus dry-run evidence.

## 18. Commit Strategy
1. `feat(db): add waves, transitions and signals`
2. `test: add scenario dsl`
3. `feat(wave): add feature computation`
4. `feat(wave): add gates and score model v1`
5. `feat(wave): add wave state machine`
6. `feat(wave): add wave service with flush-barrier signals`
7. `test(wave): add golden wave scenarios`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M2 marked.
