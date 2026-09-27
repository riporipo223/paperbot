# 09 — Real-Time Data Architecture

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [03](03-system-requirements.md), [05](05-system-architecture.md), [06](06-data-architecture.md), [07](07-database-schema.md) |
| Downstream | [10](10-market-data-spec.md), [12](12-wave-detection-spec.md), [16](16-paper-trading-spec.md), [19](19-backtesting-spec.md), [22](22-observability-spec.md) |
| Used by phases | 01 (time/envelope), 04 (pipeline), 05, 07, 08, 17 |

## 1. Pipeline overview

```text
 Provider socket / HTTP response
        │  (1) stamp received_at  [adapter, first line of the message handler]
        ▼
 Adapter parse + schema validation (zod)          ── invalid → ops.dead_letters + counter
        │
        ▼
 Normalizer → EventEnvelope                       (2) stamp normalized_at
        │
        ▼
 Enricher (optional, async; e.g. getTransaction)  (2b) enrichment latency recorded
        │
        ▼
 Dedup (LRU source_event_key)                     ── duplicate → counter, dropped
        │
        ▼
 Sequencer: assign ingest_seq, session_id
        │
        ├──────────────▶ EventPersister (async batch → market.events)
        ▼
 EventBus.publish(event)  → per-subject ordered dispatch
        │
        ▼
 Subscribers (market-state, wallet-intel, wave, observability …)   (3) stamp processed_at per consumer
```

All three event sources (live adapters, webhook receiver, replay reader) enter at the **Dedup** step with fully-formed envelopes. Everything from Dedup onward is shared code.

## 2. Canonical time model

### 2.1 Timestamps

| Field | Set by | Source | Precision | Notes |
|---|---|---|---|---|
| `slot` | Normalizer | On-chain context (`context.slot` in WS notifications, tx `slot`) | exact | Primary on-chain ordering key |
| `event_time` | Normalizer | Block time (`blockTime`, seconds) or provider trade time | 1 s for block time | May be null at normalization and filled after enrichment. Never fabricated. |
| `provider_time` | Normalizer | Provider-attached timestamp (e.g. DexScreener `pairUpdatedAt`?, GeckoTerminal trade `block_timestamp`) | provider-dependent | Exact field names verified in Phase 03 ([27](27-data-provider-reference.md)) |
| `received_at` | Adapter | `clock.now()` at first byte handling | µs | Wall clock (NTP-disciplined) |
| `normalized_at` | Normalizer | `clock.now()` | µs | |
| `processed_at` | Each consumer | `clock.now()` | µs | Recorded as latency samples, not on the event row |
| `signal_at`, `decided_at`, `evaluated_at`, `submitted_at`, `executed_at` | Respective modules | `clock.now()` (live) / simulated clock (replay) | µs | |

Durations are measured with `clock.monotonicMicros()` differences (not wall-clock subtraction), then stored as `duration_us`.

### 2.2 Slot clock (estimating event time at sub-second resolution)

Solana `blockTime` has 1-second resolution, and it is not delivered in `logsSubscribe` notifications. To measure detection latency usefully:

- `SlotClock` maintains a mapping `slot → first_observed_at` from every stream notification carrying `context.slot`, plus a periodic `getSlot` (commitment `confirmed`) poll every `clock.slot_poll_ms` (default 2,000 ms; 1 credit each, budgeted).
- `estimated_slot_time(slot)` interpolates or extrapolates using the observed slot rate (EWMA; nominal ~400 ms/slot).
- Detection latency is reported three ways:
  1. `detect_slots = tip_slot_at_receipt − event_slot` (exact in slots).
  2. `detect_ms_est = received_at − estimated_slot_time(event_slot)` (estimate, flagged `approx`).
  3. `detect_ms_blocktime = received_at − blockTime` (when block time is known, ±1 s resolution).

All three are persisted in `ops.latency_samples` (stages `detect_slots`, `detect_ms_est`, `detect_ms_blocktime`).

### 2.3 Clock discipline and drift

- The host runs chrony/NTP ([25](25-infrastructure-deployment.md)). At startup and every `clock.offset_check_minutes` (10), the engine measures its offset. Method order: (1) `chronyc tracking` if available, (2) an SNTP query to `clock.ntp_server`, (3) HTTP `Date` header comparison against two providers (coarse, ±500 ms). The result goes to `ops.clock_offsets`.
- If |offset| > `clock.max_offset_ms` (default 250), the engine raises `CLOCK_DRIFT` (WARN). If |offset| > `clock.halt_offset_ms` (default 2,000), it pauses entries (`ENTRIES_PAUSED: CLOCK_DRIFT`).
- The browser clock is never used for any recorded timestamp. The dashboard displays server timestamps and computes "age" using the server time offset from the WS `at` field.

### 2.4 Simulated time (replay)

- `SimulatedClock.now()` returns the *replay timestamp* of the event currently being processed. Under `AS_RECEIVED` ordering that is the event's original `received_at`. Under `BY_EVENT_TIME` it is `event_time`, falling back to `received_at`.
- Before delivering each event, the replay driver advances the clock to that event's timestamp and **fires all scheduler timers due at or before it**, in timestamp order. This is how exit time-outs, wave expiries and paper-fill latencies behave identically to live.
- Paper fill latency in replay: the fill is scheduled at `decided_at + sampled_latency` on the simulated clock and executes against the market state as of that simulated time (the latest state event whose replay timestamp ≤ the fill time).
- Monotonic durations in replay derive from simulated time. Real processing time during replay is measured separately as `replay_wall_us` for performance analysis only.

## 3. Event envelope

```ts
export interface EventEnvelope<TType extends EventType = EventType, TPayload = unknown> {
  eventId: EventId;                // UUIDv7
  type: TType;                     // see §5
  schemaVersion: number;           // payload schema version for this type
  source: SourceId;                // 'helius.ws' | 'helius.webhook' | 'helius.rpc' | 'dexscreener' | 'geckoterminal' | 'coingecko' | 'internal'
  sourceEventKey: string;          // deterministic dedup key (see §6)
  subject: { kind: 'wallet' | 'token' | 'pool' | 'system'; id: string };
  sessionId: SessionId;            // ingest session (engine process lifetime)
  ingestSeq: number;               // assigned by Sequencer; monotonic within session
  time: {
    slot?: number;
    eventTime?: EpochMicros;
    providerTime?: EpochMicros;
    receivedAt: EpochMicros;
    normalizedAt: EpochMicros;
  };
  quality: DataQuality;            // quality at normalization (e.g. DEGRADED if from fallback source)
  late: boolean;                   // set by lateness check (§8)
  causationId?: EventId;           // e.g. enrichment result → originating notification
  correlationId?: string;          // e.g. wave id once assigned (not on raw events)
  payload: TPayload;               // zod-validated per (type, schemaVersion)
}
```

A zod schema registry maps `(type, schemaVersion) → payload schema`. The persister stores the envelope fields as columns and `payload` as jsonb. Replay rehydrates and re-validates every event.

## 4. Latency measurement points

| Stage key | Start | End |
|---|---|---|
| `detect_*` | on-chain slot/time | `received_at` (§2.2) |
| `L1_ingest` | `received_at` | `normalized_at` |
| `L1b_enrich` | enrichment request | enrichment response |
| `L2_state` | `normalized_at` (or enrichment done) | market-state `processed_at` |
| `L3_signal` | market-state `processed_at` | wave evaluation done / `signal_at` |
| `L4_decision` | `signal_at` | `decided_at` |
| `L5_risk` | `decided_at` | risk `evaluated_at` |
| `L6_order` | risk `evaluated_at` | order `submitted_at` (after DB commit) |
| `L7_fill_compute` | scheduled exec time reached | fill computed |
| `L8_portfolio` | fill computed | ledger + position committed |
| `e2e_system` | trigger event `received_at` | order `submitted_at` |
| `data_age_at_signal` | market data `received_at` of each input | `signal_at` (per input) |
| `ws_broadcast` | engine state change | WS frame written |

The answers to "How long did it take us to detect this wallet buy?" (`detect_*` for the trigger event) and "How old was the market data when the signal was generated?" (`data_age_at_signal`, stored in `strategy.signals.data_quality`) are both queryable per signal.

## 5. Event catalog

### 5.1 Ingested events (persisted to `market.events`, replayable)

| Type | Subject | Source(s) | Key payload fields |
|---|---|---|---|
| `wallet.swap.detected` | wallet | helius.ws (+ helius.rpc enrichment) / helius.webhook | signature, logIndex, slot, wallet, venueId, poolAddress, tokenMint, side, tokenAmountRaw, quoteMint, quoteAmountRaw, feeLamports, parseMethod |
| `pool.trade.observed` | pool | helius.ws (log decoder), geckoterminal (fallback/reference) | signature, logIndex, slot, trader, side, baseAmountRaw, quoteAmountRaw, reservesAfter? |
| `pool.state.updated` | pool | helius.ws (`accountSubscribe`), helius.rpc (`getAccountInfo` on resubscribe) | slot, baseReserveRaw, quoteReserveRaw, curveComplete?, rawAccountHash |
| `market.snapshot.observed` | token | dexscreener, geckoterminal | priceUsd, liquidityUsd, volume m5/h1/h24, buys/sells m5/h1, fdv, mcap, pairCreatedAt |
| `price.reference.updated` | system (`SOL/USD`) | dexscreener (primary) / geckoterminal / coingecko (cross-check) | symbol, price |
| `token.discovered` | token | helius.rpc, dexscreener | mint, decimals, program, authorities, metadata (untrusted) |
| `pool.discovered` | pool | dexscreener, geckoterminal, onchain | address, venue, base/quote mints, vaults, createdAt |
| `stream.status.changed` | system | internal (adapter) | provider, channel, status, reason (connect/disconnect/reconnect/gap) |
| `timer.tick` | system | internal (scheduler) | Not persisted. Replay regenerates ticks from the simulated clock. |

### 5.2 Internal domain events (bus only; persisted in owning tables)

`smart.buy.detected`, `smart.sell.detected`, `pool.selected`, `token.rejected`, `watch.requested`, `watch.released`, `market.state.changed`, `wallet.score.updated`, `wave.updated`, `signal.emitted`, `decision.made`, `risk.evaluated`, `order.created`, `order.submitted`, `order.filled`, `order.failed`, `position.opened`, `position.updated`, `position.closed`, `portfolio.updated`, `ai.output.recorded`, `entries.paused`, `entries.resumed`, `run.started`, `run.stopped`, `run.halted`.

These are **derived** from ingested events plus config plus seed, so replay recreates them. They are not replay inputs.

## 6. Deduplication and idempotency

| Source | `sourceEventKey` |
|---|---|
| helius.ws logs (wallet/pool) | `sol:<signature>:<logIndex>` for decoded program events; `sol:<signature>:<wallet>` for wallet-level swap summaries |
| helius.ws account | `acct:<account>:<slot>` |
| helius.webhook | `sol:<signature>:<wallet>` (same key space as ws, so both sources dedupe against each other) |
| dexscreener | `ds:<pairAddress>:<providerUpdatedAt or received second>` |
| geckoterminal trades | `sol:<tx_hash>:<index>` (same key space as on-chain trades, for cross-source dedup) |
| geckoterminal snapshots | `gt:<pool>:<received second>` |
| coingecko | `cg:<id>:<last_updated_at>` |

- The dedup LRU holds keys for `pipeline.dedup.ttl_seconds` (default 86,400) up to `pipeline.dedup.max_keys` (default 2,000,000). Eviction is by age.
- **Cross-source merge rule:** when the same on-chain trade arrives from the on-chain log decoder *and* GeckoTerminal, the first one wins. If the later one carries fields the first lacked (e.g. `trader`), an `enrichment` is applied to the market-state trade record, but no second event is emitted.
- Consumers must also be **idempotent** on `eventId` (a restart may redeliver events that were published but not yet acknowledged; see §10).

## 7. Data quality states

| State | Definition |
|---|---|
| `FRESH` | Latest value age ≤ `max_age` for the consuming component, and the stream is healthy (connected, no unresolved gap). |
| `STALE` | Age > `max_age`, or the stream is disconnected. |
| `DEGRADED` | A value exists and is within `max_age`, but comes from a fallback/reference source (e.g. GeckoTerminal price instead of on-chain reserves), or the stream has an unresolved gap, or the provider is rate-limited. |
| `UNKNOWN` | No value yet, or the value failed validation. |

Max ages are **per consumer**, not per stream ([26](26-configuration-reference.md) `freshness.*`):

| Input | Wave feature use | Entry decision | Paper fill | Exit rule evaluation | Portfolio mark |
|---|---|---|---|---|---|
| Pool state (on-chain reserves) | 5 s | 3 s | 3 s | 5 s | 30 s |
| Pool trades stream | stream healthy + last msg ≤ 60 s or pool idle confirmed | — | — | — | — |
| Market snapshot (DexScreener/GT) | 45 s | 45 s | never used | 45 s (fallback, flagged DEGRADED) | 120 s (fallback) |
| SOL/USD | 120 s | 120 s | 120 s | 120 s | 300 s |
| Wallet stream health (heartbeat) | 30 s | 30 s | — | — | — |

Rules:
- **Paper fills require `FRESH` on-chain pool state.** No exceptions (FR-EXE-004).
- Entry decisions require `FRESH` pool state, a healthy wallet stream, and `FRESH` SOL/USD. Otherwise → `REJECT` with `DATA_STALE:<input>`.
- Exits with non-`FRESH` pool state produce an exit *decision*, but the fill fails with `STALE_MARKET_DATA` and is retried every `execution.retry_stale_exit_ms` (default 2,000). The position is marked `CLOSING` with a visible warning. The system never marks a stale exit as filled.
- Signals persist per-input quality, and `worst_quality` = the most severe.

## 8. Ordering, lateness and out-of-order events

1. **Per-pool state**: a `pool.state.updated` is applied only if `slot > last_applied_slot` for that pool. Equal slot with a different account hash → keep the one with the higher `ingestSeq` (latest write within the slot). Older → dropped, counter `mkt_state_out_of_order_total`.
2. **Trades and wallet swaps**: appended to per-subject windows keyed by `(slot, logIndex)`. Windows compute features by `event_time`/slot, not arrival order.
3. **Watermark**: for each windowed computation, `watermark = max_seen_slot − pipeline.lateness_slots` (default 12 slots ≈ 5 s). Events with `slot < watermark` at arrival are marked `late = true`. They are still stored and added to windows (so later computations are correct), but they **never retroactively change past signals or decisions**. Late counts are exposed per source.
4. **Snapshots (REST)**: applied if `provider_time` (or `received_at` when there is none) is newer than the last applied value for that subject/source.
5. **Cross-subject ordering** is not guaranteed and not required. Decisions depend on per-subject state plus the portfolio, and portfolio mutations are serialized on the event loop.

## 9. Persistence and the flush barrier

- The persister buffers envelopes and flushes every `pipeline.persist.flush_ms` (250) or `batch_size` (500).
- **Flush barrier:** before `strategy.signals`/`strategy.decisions` rows are committed, the strategy module calls `persister.flushUpTo(ingestSeq)`. It resolves when every event with `ingestSeq ≤` the trigger's watermark is durable. That watermark is stored as `causation_watermark` on the signal. This guarantees that every decision's inputs exist in the event log for replay.
- If the DB is unavailable: the buffer grows to `pipeline.persist.max_buffer` (default 100,000 events). Past 50% of that, entries pause (`ENTRIES_PAUSED: PERSISTENCE_BACKLOG`). At 100%, the oldest *coalescible* events (`pool.state.updated` superseded by later slots) are dropped first. Non-coalescible events are never dropped; ingestion subscriptions for non-held subjects are released to shed load, and a `DATA_LOSS_RISK` FATAL system event is raised.

## 10. Backpressure

| Queue | Bound (config) | Overflow policy |
|---|---|---|
| Adapter inbound (per connection) | 10,000 msgs | Pause reading from the socket (TCP backpressure). Alarm if paused > 5 s. |
| Enrichment requests | 1,000 | Prioritize held positions > watched pools > wallet swaps for new tokens > backfill. Beyond the bound, drop lowest-priority enrichment and mark the source event `DEGRADED`. |
| Bus dispatch (per subject) | 5,000 | Coalesce `pool.state.updated` (latest-wins). Other types: block producer. `BUS_BACKPRESSURE` WARN. |
| Persister buffer | 100,000 | §9 |
| WS client send queues | 1,000 per client | Latest-wins snapshot resync ([08](08-api-spec.md) §5.1) |

Consumers acknowledge by returning from their handler (async handlers are awaited per subject). The bus tracks the `lastProcessedSeq` per consumer. On a graceful shutdown the bus drains. On a crash, events after the last persisted `ingestSeq` are lost from the log. They were never the basis of a committed decision because of the flush barrier.

## 11. Connection management and reconnection

For every streaming adapter:

1. **Connect** with TLS. Stamp `stream.status.changed{status:'CONNECTED'}`.
2. **Subscribe** to the watch set. Track subscription IDs ↔ subjects.
3. **Heartbeat**: WS ping every `ingestion.ws.ping_ms` (default 15,000). Missing pong for 2 intervals → treat as disconnected. Also track `last_message_at` per subscription, with a subject-specific idle expectation.
4. **Disconnect** → status `DISCONNECTED`. All dependent streams become `STALE` immediately.
5. **Reconnect** with exponential backoff and full jitter: `delay = random(0, min(cap, base·2^attempt))`, base 500 ms, cap 30 s. Attempts are unlimited. After `ingestion.ws.alert_after_attempts` (5), raise a WARN system event.
6. **Resubscribe** all subjects. For each pool, immediately fetch the current state with `getAccountInfo` (so state becomes `FRESH` without waiting for the next change).
7. **Gap handling**: record `ops.data_gaps{gap_start = last message time, gap_end = reconnect time, from_slot, to_slot}`.
   - Wallets: `getSignaturesForAddress(wallet, {until: last_seen_signature, limit})` for each tracked wallet, credit-budgeted and prioritized by wallet score. Recovered swaps are emitted as normal events with `late = true` if they fall behind the watermark.
   - Pool trades: backfill from GeckoTerminal trades (last 300 trades) when the gap is ≤ its coverage. Mark `PARTIAL` or `UNRECOVERABLE` otherwise.
   - A wave whose window overlaps an unresolved gap gets quality `DEGRADED`, which blocks entries per the risk gate `WAVE_DATA_GAP`.
8. **Provider limits on connections and subscriptions** (e.g. Helius Free: 5 concurrent WS connections, per [27](27-data-provider-reference.md)): the `SubscriptionManager` packs subscriptions across connections up to verified limits and refuses new watch requests beyond capacity (`WATCH_CAPACITY_EXCEEDED`), with priority held > watched > tracked-wallet.

## 12. Rate limits and credit budgets

- Each REST provider has a token-bucket limiter configured from the verified limit table ([27](27-data-provider-reference.md)), set to `safety_factor` (default 0.8) of the documented limit.
- HTTP 429 → respect `Retry-After` if present, else exponential backoff. Status `RATE_LIMITED` for the channel.
- **Helius credit budget**: `CreditBudget` tracks estimated credits per call type (RPC call = 1, `getProgramAccounts`/archival = 10, DAS/enhanced = per docs, streaming = per-MB rate) against the plan's monthly allowance. It exposes spend rate vs pace. When the projected month-end usage exceeds `ingestion.helius.monthly_credit_budget` × 0.9, it enters **conserve mode**: suspend backfills, reduce `getSlot` polling, cap watched pools to held positions plus top-N waves. All of this is visible on the System page. Credit costs are verified in Phase 03.

## 13. Retries and circuit breakers

| Call type | Retry policy | Circuit breaker |
|---|---|---|
| RPC reads (enrichment) | 3 attempts, 200/400/800 ms + jitter, only on network/5xx/429 | Open after 10 consecutive failures in 30 s. Half-open probe after 15 s. |
| REST pollers | Next poll cycle is the retry (no tight retries) | Open after 5 consecutive failures. Probe every 60 s. |
| AI calls | No retry on the decision path (timeout → fallback). 1 retry for async agents. | Open after 5 failures in 5 min |
| DB writes | Retry with backoff up to 30 s for transient errors | Persistence backlog policy (§9) |

## 14. Provider downtime matrix

| Down | Effect | Entries | Exits |
|---|---|---|---|
| Helius WS | Wallet + pool streams STALE | Paused (`WALLET_STREAM_DOWN`) | Fill attempts fail as STALE until recovery. The exit decision is recorded. Operator alerted. |
| Helius RPC | Enrichment fails. Non-pump venues' wallet swaps are unparsed and marked DEGRADED. | Paused for affected venues | Unaffected if WS is up (reserves via account subscription) |
| DexScreener | Snapshot features DEGRADED (GeckoTerminal fallback) or STALE | Allowed only if `wave.require_snapshot_features = false` | Unaffected |
| GeckoTerminal | Trade-level fallback/backfill unavailable | Unaffected if on-chain trade stream is up | Unaffected |
| SOL/USD primary (DexScreener) | SOL/USD from the secondary source (GeckoTerminal SOL/USDC pool) → DEGRADED | Allowed while DEGRADED ≤ 120 s old. Paused if STALE. | Unaffected (SOL-quoted) |
| Anthropic | AI TIMEOUT/ERROR | Per `ai.on_timeout` | Unaffected |
| Database | Persistence backlog | Paused | Proceed only if write-ahead succeeds (retry) |
