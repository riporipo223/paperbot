# 12 — Wave Detection Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [10](10-market-data-spec.md), [11](11-wallet-intelligence-spec.md), [09](09-real-time-data-architecture.md) |
| Downstream | [14](14-strategy-engine-spec.md), [28](28-decision-logging-spec.md) |
| Used by phases | 10 |

## 1. Purpose

Turn "one or more smart wallets bought token X" into a deterministic, auditable judgement: **is a tradeable wave forming right now?** The output is a stream of wave state transitions and, at most once per wave, an `ENTRY` signal.

A wave is **not** a prediction that price will rise 100%. It is a measurable pattern: smart buying followed by broad participation, accelerating activity, supportive order flow, adequate liquidity, and price that has moved but is not already over-extended.

## 2. Inputs

| Input | From | Freshness requirement (feature use) |
|---|---|---|
| `smart.buy.detected`, `smart.sell.detected` | wallet-intel | Wallet stream healthy |
| Pool trades ring buffer | market-state (T1 trades) | Trade stream `HEALTHY` |
| Pool reserves / mid price / liquidity | market-state (T1 state) | ≤ `freshness.pool_state.wave_ms` (5 s) |
| Market snapshot (volume, txns) | market-state (T2) | ≤ `freshness.snapshot.wave_ms` (45 s) |
| Token safety + pool reference | reference | Present |
| Timer ticks | scheduler | every `wave.tick_ms` (1,000 ms) per active wave |

## 3. Windows

All windows are evaluated in **slot/event time**, not arrival time ([09](09-real-time-data-architecture.md) §8).

| Name | Default | Meaning |
|---|---|---|
| `W_cluster` | 600 s | Look-back for counting distinct smart wallets buying the token |
| `W_short` | 60 s | Recent activity window |
| `W_base` | 300 s | Baseline activity window (preceding `W_short`, non-overlapping) |
| `W_follow` | since seed | Window from the seed smart buy to now |
| `T_expire` | 900 s | A wave not confirmed within this time after seeding → `EXPIRED` |

## 4. Feature set `wave-features@1.0.0`

Computed on every relevant event for the token and on each tick. Every feature value is stored with its inputs' quality and age.

| Feature | Definition |
|---|---|
| `smart_wallet_count` | Distinct smart wallets with a BUY in `W_cluster` |
| `smart_score_sum` | Σ current wallet scores of those wallets (each counted once) |
| `smart_buy_sol` | Σ quote amount of those buys (SOL) |
| `smart_net_flow_sol` | Smart buys − smart sells in `W_cluster` (SOL) |
| `seed_age_s` | Now − seed swap event time |
| `first_detect_latency_ms` | Detection latency of the seed swap (`detect_ms_est`) |
| `follow_unique_buyers` | Distinct non-tracked traders with BUY trades since the seed |
| `follow_buy_sol` | Σ quote amount of those buys |
| `buy_count_short`, `sell_count_short` | Trade counts in `W_short` |
| `buy_ratio_short_vol` | buy SOL / (buy SOL + sell SOL) in `W_short` (null if no volume) |
| `tx_rate_short` | trades / s in `W_short` |
| `tx_rate_base` | trades / s in `W_base` |
| `tx_accel` | `tx_rate_short / max(tx_rate_base, ε_rate)` with ε_rate = 1/300 per s |
| `vol_rate_short`, `vol_rate_base`, `vol_accel` | Same for SOL volume |
| `price_at_seed` | Mid price at the pool state nearest to the seed slot (≤ seed slot) |
| `price_now` | Current mid price |
| `price_ext_pct` | `(price_now / price_at_seed − 1) × 100` |
| `price_change_short_pct` | Mid price change over `W_short` |
| `liquidity_usd` | Current liquidity |
| `liquidity_change_short_pct` | Liquidity change over `W_short` |
| `pool_age_s`, `token_age_s` | From pool creation / mint creation (first seen, if unknown → null) |
| `snapshot_buys_m5`, `snapshot_sells_m5`, `snapshot_vol_m5_usd` | From T2 snapshot (cross-check) |
| `top10_holder_pct` | From safety observations (null if not fetched) |
| `data_gap_overlap` | True if an unresolved gap overlaps `W_cluster` for any input stream |

Features are computed by a **pure** function `computeWaveFeatures(inputs, params, now) → WaveFeatures`. The service layer only gathers inputs.

## 5. State machine

```text
            smart.buy (first)                 gates pass & score ≥ confirm_threshold
 (none) ─────────────────────▶ SEEDED ──▶ BUILDING ───────────────────────────────▶ CONFIRMED ──(decision ENTER + fill)──▶ ENTERED
                                 │           │                                         │
                                 │           │ score < invalidate_threshold after       │ decision REJECT (risk/AI/etc)
                                 │           │ min_observe_s, or hard invalidation       ▼
                                 │           ▼                                      REJECTED
                                 │      INVALIDATED
                                 │
                 now − seeded_at > T_expire (not confirmed) ──▶ EXPIRED
 ENTERED: position lifecycle takes over; wave ends EXHAUSTED when the position closes, or smart net flow turns negative (informational)
```

| From → To | Condition |
|---|---|
| ∅ → `SEEDED` | `smart.buy.detected` for a token with no active wave in this run, **and** the token has a primary supported pool, or pool resolution is pending (≤ `wave.pool_resolution_timeout_s`, 10 s, else `REJECTED: NO_SUPPORTED_POOL`) |
| `SEEDED` → `BUILDING` | Pool streams `FRESH` and first feature computation succeeded |
| `BUILDING` → `CONFIRMED` | All **gates** (§6) pass **and** `wave_score ≥ wave.entry_score_threshold` (default 0.70) **and** `seed_age_s ≤ risk.max_entry_latency_s` |
| `BUILDING` → `INVALIDATED` | Hard invalidation (§6.2), or `wave_score < wave.invalidate_threshold` (0.25) after `wave.min_observe_s` (30 s) |
| `SEEDED`/`BUILDING` → `EXPIRED` | Not confirmed within `T_expire` |
| `CONFIRMED` → `ENTERED` | Entry fill recorded (strategy) |
| `CONFIRMED` → `REJECTED` | Decision REJECT, or entry order failed and `wave.retry_after_failed_entry = false` |
| `ENTERED` → `EXHAUSTED` | Position closed |

Additional smart buys during `SEEDED`/`BUILDING` update features and do not create a new wave. After a terminal state, a new wave for the same token may be seeded only after `wave.reseed_cooldown_s` (1,800 s). This prevents chasing the same token repeatedly.

Every transition is persisted to `strategy.wave_transitions` with features and score, and emitted as `wave.updated`.

## 6. Gates and scoring `wave-score@1.0.0`

### 6.1 Gates (all must pass to confirm; each failure recorded by code)

| Code | Condition (defaults) |
|---|---|
| `G_MIN_SMART` | `smart_wallet_count ≥ wave.min_smart_wallets` (2) |
| `G_MIN_FOLLOW` | `follow_unique_buyers ≥ wave.min_follow_buyers` (8) |
| `G_LIQUIDITY` | `liquidity_usd ≥ risk.min_liquidity_usd` (10,000) |
| `G_EXTENSION` | `price_ext_pct ≤ wave.max_price_extension_pct` (60) |
| `G_NOT_DUMPING` | `buy_ratio_short_vol ≥ wave.min_buy_ratio` (0.55) |
| `G_SMART_NOT_EXITING` | `smart_net_flow_sol > 0` |
| `G_DATA_FRESH` | Pool state, trade stream and wallet stream `FRESH`; no `data_gap_overlap` |
| `G_TOKEN_AGE` | `token_age_s ≤ risk.max_token_age_s` (43,200) when known. Unknown → pass with flag. |
| `G_POOL_SUPPORTED` | Primary pool venue `supported = true` |

### 6.2 Hard invalidation (terminal)

- Liquidity falls > `wave.liquidity_collapse_pct` (40%) within `W_short`.
- ≥ `wave.invalidate_on_smart_sells` (2) distinct smart wallets sell before confirmation.
- `price_ext_pct < −wave.max_drawdown_from_seed_pct` (−35%).
- Token safety gate fails ([15](15-risk-engine-spec.md) §4).

### 6.3 Score

Each component is normalized to [0,1]:

| Component | Formula | Weight |
|---|---|---|
| `S_smart` | `1 − exp(−smart_score_sum / λ)`, λ = `wave.smart_lambda` (1.5) | 0.30 |
| `S_follow` | `min(1, follow_unique_buyers / wave.follow_target)` (25) | 0.20 |
| `S_pressure` | `clamp((buy_ratio_short_vol − 0.5) / 0.3)` | 0.15 |
| `S_accel` | `clamp((tx_accel − 1) / (wave.accel_target − 1))` (target 3.0); average with the same for `vol_accel` | 0.15 |
| `S_momentum` | Triangular on `price_ext_pct`: 0 at ≤ 0%, rising to 1 at `wave.momentum_sweet_spot_pct` (20%), falling to 0 at `max_price_extension_pct` (60%) | 0.10 |
| `S_liquidity` | `clamp(log10(liquidity_usd / min_liquidity_usd))` (10× min → 1) | 0.10 |

`wave_score = Σ weight_k · S_k`. Weights are config (they must sum to 1, validated). The breakdown `{component: {value, normalized, weight, contribution}}` is stored with every transition and signal.

**Initial parameters are hypotheses, not optimized values.** Parameter search happens only through replay runs with new config versions ([19](19-backtesting-spec.md) §7), never by editing code.

## 7. Signal emission

On `BUILDING → CONFIRMED`, wave emits exactly one `signal.emitted` of kind `ENTRY`:

```ts
interface EntrySignal {
  signalId: SignalId; runId: RunId; waveId: WaveId;
  tokenMint: string; poolAddress: string;
  signalAt: EpochMicros;
  score: number; scoreModelVersion: 'wave-score@1.0.0';
  scoreBreakdown: ScoreBreakdown; gates: GateResult[];
  features: WaveFeatures; featureModelVersion: 'wave-features@1.0.0';
  dataQuality: Record<InputName, { quality: DataQuality; ageMs: number | null; source: string }>;
  worstQuality: DataQuality;
  triggerEventIds: EventId[];      // seed swap + all smart swaps in cluster + latest state event
  causationWatermark: number;      // ingestSeq (flush barrier)
  midPriceQuote: string;
}
```

The signal is persisted (after the flush barrier) before strategy acts on it.

## 8. Exit-side wave signals

While `ENTERED`, wave keeps computing features for the held token and exposes them to the ExitManager (`smart_net_flow_sol`, `buy_ratio_short_vol`, `liquidity_change_short_pct`). EXIT signals are produced by the ExitManager ([14](14-strategy-engine-spec.md) §5), not by wave.

## 9. Performance

- Feature computation is O(trades in windows) with incremental ring-buffer aggregates. Target ≤ 1 ms per evaluation for ≤ 5,000 trades in window.
- The tick-driven evaluation only runs for active waves (bounded by the watch set).

## 10. Tests (minimum)

- Fixture scenario "classic wave": 2 smart buys, 20 follow-on buyers, accelerating trades → CONFIRMED at the expected event, with the expected score (golden value).
- "Single whale, no follow" → EXPIRED.
- "Over-extended": price +120% by the second smart buy → gate `G_EXTENSION` fails, no signal.
- "Smart wallets dump" → INVALIDATED.
- "Liquidity pull" → INVALIDATED (hard).
- "Stale pool stream" → cannot confirm (`G_DATA_FRESH`).
- Late event after the watermark does not retroactively confirm or invalidate.
- Determinism: shuffled arrival order within lateness tolerance → identical features at each watermark.
- Reseed cooldown respected.
