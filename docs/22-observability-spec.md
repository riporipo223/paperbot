# 22 — Observability Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [03](03-system-requirements.md) §1–2, [09](09-real-time-data-architecture.md) §4, [28](28-decision-logging-spec.md) |
| Downstream | [20](20-dashboard-spec.md) §3.7, [25](25-infrastructure-deployment.md) |
| Used by phases | 01 (logger), 04 (latency recorder), every phase (logs/metrics), 22 (hardening), 25 (baselines) |

## 1. Pillars

| Pillar | Tooling | Storage |
|---|---|---|
| Structured logs | pino JSON to stdout | Platform log drain (host) |
| Metrics | prom-client registry, `/metrics` | Scraped (optional) + summaries persisted |
| Latency samples | `LatencyRecorder` | `ops.latency_samples` (partitioned, 14-day retention) + in-memory HDR histograms for live percentiles |
| System events | `SystemEventLog` | `ops.system_events` |
| Provider health | `ProviderHealthMonitor` | `ops.provider_health` (every 30 s per channel + on change) |
| Decision traces | DB records | [28](28-decision-logging-spec.md) |

## 2. Log format

```json
{"level":"info","time":"2026-09-27T10:15:02.123456Z","module":"strategy","event":"decision.made",
 "runId":"…","waveId":"…","signalId":"…","decisionId":"…","tokenMint":"…",
 "action":"ENTER","reasonCodes":["WAVE_CONFIRMED"],"latencyUs":{"L4_decision":820},"msg":"entry decision"}
```

- Required keys: `level`, `time` (server clock, µs), `module`, `event` (catalog key), `msg`.
- Correlation keys (when known): `runId`, `sessionId`, `eventId`, `waveId`, `signalId`, `decisionId`, `riskDecisionId`, `orderId`, `positionId`, `correlationId`.
- Levels: `debug` (per-event noise, off in prod), `info` (state changes), `warn` (degradation), `error` (failures needing attention), `fatal` (halts).
- Redaction per [21](21-security-spec.md) §4.
- Sampling: per-event debug logs for high-frequency streams are sampled (`observability.debug_sample_rate`, 0.01).

## 3. Log event catalog (minimum)

| Category | `event` keys |
|---|---|
| Market event | `event.ingested` (debug, sampled), `event.invalid`, `event.duplicate` (debug), `event.late` |
| Wallet event | `wallet.swap.detected`, `wallet.score.updated`, `wallet.status.changed`, `backfill.progress`, `backfill.done` |
| Signal | `wave.transition`, `signal.emitted` |
| AI decision | `ai.call.started` (debug), `ai.output.recorded`, `ai.call.failed`, `ai.budget.exceeded` |
| Risk decision | `risk.evaluated` |
| Decision | `decision.made` |
| Paper order | `order.created`, `order.submitted`, `order.cancelled`, `order.expired` |
| Paper fill | `order.filled`, `order.failed` |
| Position change | `position.opened`, `position.updated` (debug), `position.closed` |
| Portfolio change | `portfolio.snapshot` (debug), `ledger.invariant.violation` (fatal) |
| Provider | `provider.connected`, `provider.disconnected`, `provider.reconnecting`, `provider.rate_limited`, `provider.error`, `credit.conserve_mode` |
| Latency | `latency.budget.exceeded` (warn, when a stage p95 over 5 min exceeds its provisional ceiling) |
| System | `engine.started`, `engine.stopping`, `run.started`, `run.stopped`, `run.halted`, `entries.paused`, `entries.resumed`, `clock.drift`, `gap.detected`, `gap.resolved`, `dead_letter` |

## 4. System event types (`ops.system_events.type`)

`ENGINE_START`, `ENGINE_STOP`, `CONFIG_LOADED`, `RUN_STARTED`, `RUN_STOPPED`, `RUN_HALTED`, `ENTRIES_PAUSED`, `ENTRIES_RESUMED`, `PROVIDER_STATUS`, `RECONNECT`, `GAP_DETECTED`, `GAP_RESOLVED`, `RATE_LIMITED`, `CIRCUIT_OPEN`, `CIRCUIT_CLOSED`, `CREDIT_CONSERVE_MODE`, `CLOCK_DRIFT`, `PERSISTENCE_BACKLOG`, `DATA_LOSS_RISK`, `RECONCILIATION_MISMATCH`, `LEDGER_INVARIANT_VIOLATION`, `WATCH_CAPACITY_EXCEEDED`, `WATCH_CAPACITY_CRITICAL`, `RECOVERY_REPORT`, `PARTITION_MAINTENANCE`, `RETENTION_RUN`, `AI_BUDGET_EXCEEDED`.

## 5. Metrics (Prometheus names)

| Metric | Type | Labels |
|---|---|---|
| `paperbot_events_ingested_total` | counter | source, type |
| `paperbot_events_invalid_total` / `_duplicate_total` / `_late_total` | counter | source, type |
| `paperbot_stage_latency_seconds` | histogram | stage |
| `paperbot_detect_latency_slots` | histogram | source |
| `paperbot_bus_queue_depth` | gauge | queue |
| `paperbot_persist_buffer_events` | gauge | — |
| `paperbot_provider_up` | gauge (0/1) | provider, channel |
| `paperbot_provider_reconnects_total` | counter | provider |
| `paperbot_provider_rate_limited_total` | counter | provider, channel |
| `paperbot_helius_credits_used` | gauge | window (hour/day/month) |
| `paperbot_ws_subscriptions` | gauge | provider, kind |
| `paperbot_stream_age_seconds` | gauge | stream, subject_kind (top-N only to bound cardinality) |
| `paperbot_waves_active` | gauge | state |
| `paperbot_signals_total` | counter | kind |
| `paperbot_decisions_total` | counter | action |
| `paperbot_risk_rejections_total` | counter | code |
| `paperbot_orders_total` | counter | side, status, failure_reason |
| `paperbot_equity_usd` / `paperbot_equity_sol` / `paperbot_drawdown_ratio` | gauge | run |
| `paperbot_entries_paused` | gauge | reason |
| `paperbot_ai_calls_total` | counter | agent, status |
| `paperbot_ai_cost_usd_total` | counter | agent |
| `paperbot_dead_letters_total` | counter | module |
| `paperbot_clock_offset_ms` | gauge | — |
| `paperbot_trace_completeness_ratio` | gauge | run |

Cardinality rule: no labels with unbounded values (mints, wallets, signatures) except the capped top-N stream age gauge.

## 6. Health endpoints

- `/health` (liveness): process up and event loop responsive (lag < 1 s).
- `/ready` (readiness): DB reachable, migrations at the expected version, config valid, at least one provider stream connected (LIVE_FEED mode).
- Event loop lag monitored (`perf_hooks.monitorEventLoopDelay`). Lag p99 > 100 ms → WARN (the hot path is being blocked).

## 7. Alerts (MVP: surfaced on dashboard + optional webhook)

| Alert | Condition | Severity |
|---|---|---|
| Stream down | Wallet or held-pool stream disconnected > 30 s | High |
| Entries auto-paused | Any automatic pause reason | Medium |
| Stale exit | A position `EXIT_BLOCKED_STALE` > 60 s | High |
| Ledger invariant | Any violation | Critical |
| Credit conserve mode | Entered | Medium |
| Persistence backlog | Buffer > 50% | High |
| Clock drift | > max offset | Medium |
| Dead letters | > 10 in 5 min | Medium |
| Trace completeness | < 1.0 | Medium |

Delivery: dashboard banner/System page always. Optional outbound alert webhook (`observability.alert_webhook_url`, e.g. a Discord/Slack incoming webhook) with rate limiting. No PII or secrets are included.

## 8. Latency baselines (Phase 25)

The provisional ceilings in [03](03-system-requirements.md) §2 are replaced by measured baselines after ≥ 72 h of continuous paper trading:

| Stage | p50 | p95 | p99 | Measured on (host, date) |
|---|---|---|---|---|
| detect_slots | TBD | TBD | TBD | TBD |
| detect_ms_est | TBD | TBD | TBD | TBD |
| L1_ingest … L8_portfolio | TBD | TBD | TBD | TBD |
| e2e_system | TBD | TBD | TBD | TBD |

New budgets = measured p95 × 1.5 (headroom). They are then encoded in `observability.latency_budgets` config, and regressions raise `latency.budget.exceeded`.
