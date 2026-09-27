# 19 — Backtesting and Event Replay Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [09](09-real-time-data-architecture.md) §2.4, [06](06-data-architecture.md), [14](14-strategy-engine-spec.md), [13](13-ai-agent-architecture.md) §10 |
| Downstream | [23](23-testing-strategy.md), Phase 25 validation |
| Used by phases | 04 (EventSource interface), 17 (implementation), 25 |

## 1. Principle

**One pipeline.** Replay is not a separate backtester. It is the production pipeline driven by a different `EventSource`, `Clock` and `LlmClient`:

```text
LiveEventSource   ─┐                      SystemClock     | SimulatedClock
ReplayEventSource ─┴─▶ Dedup → Sequencer → Bus → market-state → wave → strategy → risk → execution → portfolio
                                            AnthropicClient | RecordedLlmClient
```

Anything that behaves differently in replay (other than those three ports) is a bug.

## 2. What replay can and cannot reproduce

| Aspect | Reproducible? | How |
|---|---|---|
| Market inputs the system saw | Yes | `market.events` (warm) or archives (cold) |
| Timing of arrivals (latency as experienced) | Yes, under `AS_RECEIVED` | `received_at` preserved |
| Wallet scores as of each moment | Yes | Append-only `intel.wallet_scores`, resolved as of simulated time |
| Paper latency/failure/MEV draws | Yes | Same `rng_seed` + per-order forked streams |
| AI outputs | Yes, if recorded | `RecordedLlmClient` by input hash |
| Events the system never subscribed to | **No** | Replay cannot invent data outside the watch set of the original run. A config with a different watch policy sees only recorded subjects. The report flags this "coverage limitation". |
| Market reaction to our own trades | No (neither does live paper) | [17](17-economics-engine-spec.md) §9 |

**Coverage caveat:** replays of *different strategy configs* over recorded data are biased toward tokens that the *original* configuration chose to watch. To reduce this bias, a "broad capture" mode (`watch.capture_mode = BROAD`) can be enabled during data collection. It watches every token touched by any tracked or candidate wallet, within credit budgets. Validation reports state which capture mode produced the data.

## 3. Replay modes

| Option | Values | Default |
|---|---|---|
| `replay.ordering` | `AS_RECEIVED` (by `session_id`, `ingest_seq`, across sessions by session start), `BY_EVENT_TIME` (by `slot`, then `event_time`, then `received_at`) | `AS_RECEIVED` |
| `replay.speed` | `MAX` (as fast as possible), `REALTIME`, `xN` | `MAX` |
| `replay.from` / `replay.to` | timestamps | required |
| `replay.rng_seed` | integer | the original run's seed if `replayOfRunId` is given, else random (recorded) |
| `replay.ai_cache_scope` | `SOURCE_RUN`, `ANY`, `NONE` | `SOURCE_RUN` |
| `replay.ai_live_on_miss` | boolean | false |
| `wallet.score_resolution` | `AS_OF_TIME`, `FROZEN_AT_START` | `AS_OF_TIME` |

`AS_RECEIVED` reproduces what the live system experienced, including our detection latency (ADR-0008). `BY_EVENT_TIME` answers "what if our detection were perfect?". It is useful for measuring latency cost, and it is reported as an idealized result.

## 4. `ReplayEventSource`

- Reads `market.events` in pages (`replay.page_size`, default 5,000) with keyset pagination on the ordering key. Reading happens on a separate async reader that prefetches ahead (bounded).
- Reads cold archives (NDJSON.gz per day) through the same row decoder when the warm tables don't cover the range.
- Each row is rehydrated into an `EventEnvelope` and **re-validated** against its payload schema version (schema upcasters handle old versions).
- Before emitting each event: `simulatedClock.advanceTo(replayTs(event))`, which fires due timers first (scheduler), then emits.
- At the end: advance the clock to `replay.to`, fire remaining timers, stop the run (`COMPLETED`), and write final snapshots.

## 5. Isolation

- A replay creates a new `strategy.runs` row (`source = REPLAY`, `replay_of_run_id` optional) with its own ledger and positions. It shares reference and intel tables read-only (as-of-time resolution).
- Replay runs execute in a **separate process** (`pnpm replay --config <id> --from ... --to ...`) or a worker thread with its own module graph. They never share the live engine's in-memory state or provider connections.
- Replay has **no provider access**. The ingestion module is not constructed. Any accidental network call fails the test harness (network guard).

## 6. Determinism verification

`replay.verify = true` (also used in tests):
1. Replay a LIVE_FEED run with its seed and `SOURCE_RUN` AI cache.
2. Compare, in order: the signals, decisions (action + reason codes), risk outcomes, orders, and fills (amounts, prices, costs), by content hash per record. IDs are excluded (they are regenerated) and replaced with a canonical ordinal.
3. Any difference → a report of the first divergence with both records.

Sources of divergence that must be eliminated by design: ambient time, ambient randomness, iteration over unordered maps (use insertion-ordered maps or sort explicitly), floating-point math, wall-clock-dependent timeouts (all timeouts via the scheduler), and DB-generated values used in logic.

Known accepted divergence: `system_overhead` in the latency model uses the recorded value when available ([17](17-economics-engine-spec.md) §5). Otherwise the configured constant is used, and the report flags it.

## 7. Parameter studies and sensitivity analysis

- A parameter study = N replay runs over the same range with different **config versions** (never code edits). CLI: `pnpm replay:study --base <configId> --vary wave.entry_score_threshold=0.6,0.7,0.8 --vary economics.latency.medianMs=600,1200,2400`.
- **Required sensitivity set** for any validation claim (Phase 25): latency ×{0.5, 1, 2, 4}; `economics.failure.base_prob` ∈ {0, 0.03, 0.10}; MEV off/on/double; `risk.max_slippage_bps` ∈ {500, 1000, 2000}; priority fee low/median/high.
- **Overfitting controls**: a study must declare a training range and a held-out range. Reported metrics must include the held-out range. The number of configurations tried is reported with the results (multiple-comparison awareness).
- Output: a comparison table via `/api/v1/analytics/compare` and a markdown report generated to `reports/<study-id>.md`.

## 8. Performance target

Replay throughput ≥ 20× real time on a laptop-class machine for a typical day of recorded events (to be measured in Phase 23). The bottleneck is expected to be DB writes of trading records, which are few.

## 9. Tests (minimum)

- Golden scenario replays: fixture event streams → expected decisions/fills (these are the core end-to-end tests of [23](23-testing-strategy.md)).
- Live-vs-replay equivalence: run a fixture stream through the live path (fake adapters + SystemClock driven by fake timers) and through replay. Identical decision hashes.
- Timers: wave expiry and max-hold exits fire at the same simulated times.
- The as-of-time wallet score is never from the future.
- Archive reader equals warm-table reader for the same day.
- Network guard: replay with ingestion absent makes zero network calls.
