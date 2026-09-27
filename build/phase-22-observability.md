# Phase 22 — Observability Hardening

| Field | Value |
|---|---|
| Milestone | M6 Production paper |
| Depends on | 18 |
| Size | M |
| Requirements | FR-OBS-001..003, FR-LOG-001 (audit job), NFR-TIME-003 |

## 1. Objective
Close the observability gaps: a log catalog audit, full metrics coverage, provider health monitoring, the clock offset monitor, alert rules with optional webhook delivery, event loop lag monitoring, the scheduled trace-completeness audit, retention/archive jobs, and optional client render metrics.

## 2. Context (read first)
- [22](../docs/22-observability-spec.md) (entire)
- [09](../docs/09-real-time-data-architecture.md) §2.3 (clock), §4 (stages)
- [06](../docs/06-data-architecture.md) §3 (retention/archive)
- [28](../docs/28-decision-logging-spec.md) §7

## 3. Dependencies
Phase 18 DONE (it can run in parallel with 19–21).

## 4. Inputs
Existing logs, metrics, recorders, and system events from earlier phases.

## 5. Outputs
- `modules/observability`: `ProviderHealthMonitor`, `ClockOffsetMonitor`, `AlertEngine` (+ webhook notifier with rate limiting), `EventLoopMonitor`, `LatencyBudgetWatcher`, and `MaintenanceJobs` (partitions, retention, archive, trace audit).
- A test that asserts the log catalog coverage (every `event` key in [22](../docs/22-observability-spec.md) §3 is emitted somewhere, verified via a registry of log event constants).
- `POST /api/v1/system/client-metrics` + the dashboard sampler (optional task).

## 6. Files To Create
```text
apps/engine/src/modules/observability/
  provider-health-monitor.ts clock-offset-monitor.ts alert-engine.ts alert-webhook.ts event-loop-monitor.ts latency-budget-watcher.ts
  maintenance/{partition-job.ts,retention-job.ts,archive-job.ts,trace-audit-job.ts}
  log-events.ts               # registry of log event keys (constants)
  __tests__/...
apps/engine/src/modules/api/routes/client-metrics.ts   (optional)
apps/dashboard/lib/stream/render-sampler.ts             (optional)
```

## 7. Files To Modify
- Modules that log free-form event strings → use the `log-events.ts` constants.
- `modules/api/routes/system.ts` (include health, alerts, clock offset).

## 8. Database Changes
None (uses `ops.*`).

## 9. API Changes
The optional client-metrics endpoint (already in [08](../docs/08-api-spec.md) §4.1).

## 10. Environment Variables
None new (the alert webhook URL is config).

## 11. Implementation Tasks

**22.1 Log event registry + coverage test**
- Tests first: every key in [22](../docs/22-observability-spec.md) §3 exists in the registry, and a static scan finds at least one usage of each. Free-form `event` strings are banned by a lint rule (custom ESLint rule or a safety scanner rule).

**22.2 ProviderHealthMonitor**
- Tests first: status derivation (UP/DEGRADED/DOWN/RATE_LIMITED) from adapter signals. Writes to `ops.provider_health` every 30 s and on change. The `paperbot_provider_up` gauge.

**22.3 ClockOffsetMonitor**
- Tests first: injected offsets → WARN above `max_offset_ms` and an entry pause above `halt_offset_ms` (via EntryPause). Method fallback order. Samples are persisted.

**22.4 EventLoopMonitor**
- Tests first: the lag histogram. p99 > 100 ms → WARN (injected blocking in the test).

**22.5 LatencyBudgetWatcher**
- Tests first: per-stage rolling 5-minute p95 vs budgets (the provisional ceilings from [03](../docs/03-system-requirements.md) §2 until baselines are set) → `latency.budget.exceeded`.

**22.6 AlertEngine + webhook**
- Tests first: each alert rule in [22](../docs/22-observability-spec.md) §7 fires and resolves. The webhook payload has no secrets. Rate limit of 1 per rule per 10 min. Webhook failures don't affect the engine.

**22.7 Maintenance jobs**
- Tests first (integration): daily partition ensure, retention drop, archive-before-drop (writes NDJSON.gz to a local directory or an S3-compatible target via a pluggable writer, and verifies row counts), and the trace audit job (writes the `paperbot_trace_completeness_ratio` gauge and a system event).

**22.8 Client render metrics (optional)**
- Tests first: the dashboard samples ≤ 1/s render latency per channel and posts it via the server route. The engine records the `ui_render` stage.

## 12. Acceptance Criteria
1. The log catalog is fully covered, and free-form event keys are banned.
2. Provider health, clock offset, event loop lag and latency budget monitoring are live.
3. The alerts appear on the System page and, if configured, via the webhook.
4. The maintenance jobs run on schedule (tests + one manual run recorded).

## 13. Tests
Unit and integration as listed.

## 14. Failure Cases
- The archive target is unavailable → the partition is not dropped, an alert is raised, and it retries the next day.
- The alert webhook fails → logged, no retry storm.

## 15. Observability
This phase is observability. Verify with the System page screenshots in the verification note.

## 16. Security
The alert payloads and archives contain no secrets. The archive target credentials come only from env.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- observability
pnpm test:integration --filter @paperbot/engine -- maintenance
```

## 18. Commit Strategy
1. `feat(observability): add log event registry and coverage check`
2. `feat(observability): add provider health and clock offset monitors`
3. `feat(observability): add event loop and latency budget watchers`
4. `feat(observability): add alert engine with webhook`
5. `feat(observability): add maintenance jobs`
6. `feat(observability): add client render metrics`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
