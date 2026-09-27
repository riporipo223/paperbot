# Phase 04 — Event Pipeline Core

| Field | Value |
|---|---|
| Milestone | M1 Truthful data |
| Depends on | 02, 03 |
| Size | L |
| Requirements | FR-ING-005, FR-ING-006, FR-ING-009, FR-OBS-002 (latency recorder), NFR-REL-003 (persistence backlog) |

## 1. Objective
Build the shared event pipeline that every source feeds: dedup, sequencing, typed bus with per-subject ordered dispatch and backpressure, async batched persistence with a flush barrier, the `EventSource` abstraction (live and replay stubs), latency recording, dead letters, and the engine module lifecycle (composition root).

## 2. Context (read first)
- [09](../docs/09-real-time-data-architecture.md) (entire, especially §1, §3–6, §8–10)
- [05](../docs/05-system-architecture.md) §7–9
- [06](../docs/06-data-architecture.md) §4
- [07](../docs/07-database-schema.md) §3.2 (`market.events`), §3.6 (`ops.*`), §5
- [19](../docs/19-backtesting-spec.md) §1, §4 (interface only)
- [22](../docs/22-observability-spec.md) §3–5

## 3. Dependencies
Phases 02 and 03 DONE.

## 4. Inputs
DB with `market.events` and `ops.*`. Clients from Phase 03. Core envelope + registry.

## 5. Outputs
- `modules/pipeline`: `Dedup`, `Sequencer`, `EventBus`, `EventPersister`, `EventSource` interface + `LiveEventSource` (adapter registry) + `ReplayEventSource` skeleton (reads `market.events` in order; full replay orchestration in Phase 17), `DeadLetterSink`.
- `modules/observability` (minimal): `LatencyRecorder` (HDR histogram in memory + batched writes to `ops.latency_samples`), `SystemEventLog`, a metrics registry with the pipeline metrics.
- `bootstrap/composition-root.ts`: constructs modules in dependency order with lifecycle hooks (`start`, `stop`), graceful shutdown.
- `modules/ingestion/shared/credit-sink.ts`: persists credit usage hourly to `ops.credit_usage`.

## 6. Files To Create
```text
apps/engine/src/modules/pipeline/
  index.ts ports.ts service.ts
  domain/{dedup.ts,sequencer.ts,subject-queue.ts,lateness.ts,backpressure-policy.ts}
  adapters/{event-persister.ts,event-row-mapper.ts,dead-letter-sink.ts,live-event-source.ts,replay-event-source.ts}
  bus.ts
  __tests__/..., adapters/__tests__/*.int.test.ts
apps/engine/src/modules/observability/
  index.ts latency-recorder.ts system-event-log.ts metrics.ts
  __tests__/...
apps/engine/src/bootstrap/{composition-root.ts,lifecycle.ts,shutdown.ts}
apps/engine/src/modules/ingestion/shared/credit-sink.ts
```

## 7. Files To Modify
- `apps/engine/src/main.ts` (use the composition root; start/stop)
- `packages/core/src/events/event-types.ts` (register `stream.status.changed` payload schema)

## 8. Database Changes
None new (uses 0002, 0004). The persister calls `ensurePartitions` on a missing-partition error.

## 9. API Changes
None (the `/metrics` endpoint arrives with the API in Phase 18; the registry exists now).

## 10. Environment Variables
None new.

## 11. Implementation Tasks

**04.1 Dedup**
- Tests first: the first key passes, a repeat within TTL is dropped, and after TTL it passes again. The `max_keys` bound evicts the oldest. It is clock-driven (SimulatedClock). A duplicate increments the counter.
- Implement: an LRU with age-based eviction ([09](../docs/09-real-time-data-architecture.md) §6).

**04.2 Sequencer**
- Tests first: `ingestSeq` is strictly increasing per session. `sessionId` is stable within a process. Stamps are attached without mutating the input (immutability).

**04.3 Lateness / watermark helper**
- Tests first: the watermark per subject = max slot − lateness. An event below the watermark gets `late=true`. Events without a slot are never late by slot rules.

**04.4 EventBus with per-subject ordering**
- Tests first: handlers for the same subject receive events in `ingestSeq` order, even if async handlers take different times. Different subjects interleave. A handler exception → dead letter + counter, and the bus continues. A consumer's `lastProcessedSeq` is tracked. `drain()` resolves when the queues are empty.
- Tests first (backpressure): at the bound, `pool.state.updated` coalesces (latest per pool kept). Other types block the producer (the promise resolves when capacity frees). The `BUS_BACKPRESSURE` warn is emitted once per episode.
- Implement: typed `publish<T>()`, `subscribe(type | '*', handler, {consumer})`.

**04.5 Event row mapper**
- Tests first: an envelope → `market.events` row → an envelope round-trip is lossless (µs timestamps, bigint slots, jsonb payload). Payload re-validation runs on read.

**04.6 EventPersister**
- Tests first (integration): batches flush at `flush_ms` or `batch_size`. `flushUpTo(seq)` resolves only after all events ≤ seq are committed. A missing partition triggers `ensurePartitions` and a retry. DB down → the buffer grows and `backlog` state is reported at 50%. Coalescible events are dropped first at 100%. Non-coalescible events are never dropped. Duplicate rows (same dedup key) are ignored (`ON CONFLICT DO NOTHING`).
- Implement: multi-row insert (or COPY), retries with backoff, and a status observable.

**04.7 EventSource abstraction**
- Tests first: `LiveEventSource` with two fake adapters emits envelopes into Dedup → Sequencer → Bus → Persister. `ReplayEventSource` reads rows ordered by `(session_id start, ingest_seq)` for a time range and emits the same envelopes (compared to what was persisted).
- Implement the interface per [19](../docs/19-backtesting-spec.md) §4 (the full replay driver is in Phase 17).

**04.8 Dead-letter sink**
- Tests first: invalid payloads and handler failures produce `ops.dead_letters` rows with the error code and a truncated payload.

**04.9 LatencyRecorder**
- Tests first: `record(stage, durationUs, refId)` updates the in-memory histogram (percentiles correct on known data) and enqueues DB samples. Batch writes to `ops.latency_samples` work (integration). The stage names are validated against the catalog in [09](../docs/09-real-time-data-architecture.md) §4.

**04.10 SystemEventLog and metrics registry**
- Tests first: `log(type, level, data)` writes to `ops.system_events` and to the logger. The metrics registry exposes the pipeline metrics from [22](../docs/22-observability-spec.md) §5.

**04.11 Composition root and lifecycle**
- Tests first: modules start in dependency order and stop in reverse. SIGTERM triggers a graceful shutdown: stop sources → drain bus → flush persister → close DB, within 20 s (fake timers).

**04.12 Credit sink**
- Tests first: credit budget totals are persisted hourly (upsert on `(provider, window_start)`).

**04.13 Pipeline throughput smoke benchmark**
- Implement `apps/engine/bench/pipeline.bench.ts`: synthetic 10k events through Dedup→Bus→(no-op consumers)→Persister (Testcontainers). Record the result in the verification note (no pass/fail gate yet).

## 12. Acceptance Criteria
1. Events from fake live adapters are deduplicated, sequenced, dispatched in per-subject order, and persisted to `market.events` in batches.
2. `flushUpTo` provides the flush-barrier guarantee (test).
3. The persistence backlog policy behaves as specified, including the entry-pause signal hook (`PERSISTENCE_BACKLOG` emitted).
4. `ReplayEventSource` re-emits persisted events identically (envelope equality excluding nothing).
5. Latency samples and system events are persisted.
6. Graceful shutdown drains and flushes.

## 13. Tests
Unit (domain), integration (persister, replay source, recorder, dead letters), and a benchmark smoke.

## 14. Failure Cases
- A handler throws → dead letter. After `pipeline.module_failure_threshold` (default 20 in 60 s) the module circuit opens and `MODULE_CIRCUIT_OPEN` is emitted (the entry-pause hook, consumed in Phase 15).
- DB outage → backlog behaviour.
- Invalid envelope from an adapter → rejected before sequencing, dead-lettered.

## 15. Observability
Metrics: `paperbot_events_ingested_total`, `_duplicate_total`, `_late_total`, `_invalid_total`, `paperbot_bus_queue_depth`, `paperbot_persist_buffer_events`, `paperbot_dead_letters_total`. Stage latencies `L1_ingest` (adapters stamp it) and a persister flush latency. System events: `PERSISTENCE_BACKLOG`, `DATA_LOSS_RISK`, `CIRCUIT_OPEN`.

## 16. Security
Dead-letter payloads are truncated (≤ 8 KB) and must not contain secrets (adapters strip auth before creating envelopes).

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- pipeline observability
pnpm test:integration --filter @paperbot/engine -- pipeline
pnpm tsx apps/engine/bench/pipeline.bench.ts
```

## 18. Commit Strategy
1. `feat(pipeline): add dedup, sequencer and lateness helpers`
2. `feat(pipeline): add typed event bus with per-subject ordering`
3. `feat(pipeline): add batched event persister with flush barrier`
4. `feat(pipeline): add event source abstraction and replay reader`
5. `feat(observability): add latency recorder, system events, metrics`
6. `feat(engine): add composition root and graceful shutdown`
7. `test(pipeline): add throughput smoke benchmark`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Benchmark numbers recorded in the verification note.
