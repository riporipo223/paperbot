# 11 — Wallet Intelligence Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [10](10-market-data-spec.md), [09](09-real-time-data-architecture.md), [07](07-database-schema.md) §3.3 |
| Downstream | [12](12-wave-detection-spec.md), [13](13-ai-agent-architecture.md) |
| Used by phases | 05 (registry), 09 (intelligence) |

## 1. Purpose

Maintain a set of wallets whose historical trading behaviour suggests that their buys are informative. Score them deterministically, keep the scores current, and deliver their live swaps to wave detection with minimal latency.

## 2. Wallet lifecycle

```text
          import/add                 score ≥ promote rules?
  (none) ───────────▶ CANDIDATE ─────────────────────────▶ TRACKED
                         │   ▲                                 │
               operator  │   │ operator / score < demote rules │
                         ▼   │                                 ▼
                      IGNORED ◀──────── operator ────────── BLOCKED (never tracked; e.g. dev/insider/wash)
```

- `TRACKED`: subscribed live (P2 watch priority). Buys can seed waves when the score ≥ `wallet.min_wallet_score`.
- `CANDIDATE`: history is scored but the wallet is not streamed live.
- `IGNORED`: kept for reference, not scored periodically.
- `BLOCKED`: excluded everywhere. Its buys are ignored even if observed in pool trades.
- Automatic promotion/demotion is **off by default** (`wallet.auto_promote = false`). Rules, when enabled: promote if score ≥ `promote_score` and confidence ≥ `promote_confidence`; demote if score < `demote_score` for two consecutive recomputations. Each change is logged as a system event.

## 3. Sources of wallets (MVP)

1. **Seed import**: operator-provided CSV/JSON (`address,label,tags`). Default status `CANDIDATE`.
2. **Manual add** via the API/dashboard.
3. **Mining (Phase 09, optional capability, off by default)**: from closed waves and trades we observed, collect early buyers of tokens that later rose ≥ `wallet.mining.min_peak_multiple` within `wallet.mining.window_minutes`, and add them as `CANDIDATE` with `source = MINED`. This uses only data we already hold (no extra credits).

The system never claims a wallet is "smart" by label alone. Only the score counts.

## 4. History backfill

For each wallet with a queued `intel.backfill_jobs` row:

1. `getSignaturesForAddress(wallet, {before: cursor, limit: 1000})`, paged back to `wallet.backfill.lookback_days` (default 30) or `max_signatures` (default 2,000).
2. For each signature, `getTransaction` (jsonParsed, `maxSupportedTransactionVersion: 0`) → decode with venue decoders, else the balance-delta parser ([10](10-market-data-spec.md) §4.2).
3. Persist swaps to `intel.wallet_swaps` (`origin = BACKFILL`). SOL/USD values at the time come from GeckoTerminal SOL/USDC OHLCV (cached daily/hourly bars). If unavailable, `value_usd` is null and quote-denominated metrics are used.
4. Credits are charged to the job (`credits_used`). Stop with `BUDGET_EXHAUSTED` at `wallet.backfill.max_credits_per_job` (default 5,000) or when the global `CreditBudget` refuses.
5. Jobs run in a background worker with the lowest RPC priority, and are resumable from `cursor`.

A Helius Enhanced Transactions (parsed history) path MAY replace steps 1–2 if Phase 03 verifies its availability and credit cost on the plan in use. It is an adapter choice behind the same `WalletHistorySource` port.

## 5. Round-trip reconstruction

Per (wallet, token), ordered by `(slot, log_index)`:

- FIFO lot matching: each `BUY` creates a lot `(qty, cost_quote, time)`. Each `SELL` consumes lots FIFO and produces realized PnL per consumed portion.
- A **round trip** opens at the first buy after a flat position and closes when the position returns to ≤ `dust_threshold` (default 0.1% of peak quantity). Partial sells stay in the same round trip.
- Tokens received by transfer (no swap) → a lot with unknown cost. The round trip is flagged `UNKNOWN_COST` and excluded from PnL metrics.
- Costs include the swap's quote amount. Network fees are included when known (`fee_lamports`, if the wallet was the fee payer).
- Output: `intel.wallet_round_trips` with `computation_version = 'round-trip@1.0.0'`.

## 6. Wallet score model `wallet-score@1.0.0`

Deterministic, pure function `scoreWallet(roundTrips, swaps, params, asOf) → WalletScore`. The parameters live in config (`wallet.scoring.*`) and are part of the config version.

### 6.1 Evaluation window

Only round trips **closed** within `[asOf − lookback_days, asOf]` count (default 30 days). Open round trips are excluded, except that they contribute to the activity count.

### 6.2 Components

| # | Component | Raw metric | Normalization → [0,1] | Default weight |
|---|---|---|---|---|
| C1 | Profitability | Median ROI per round trip (net of known fees) | `logistic(x; mid=0.10, k=8)` | 0.25 |
| C2 | Expectancy | Mean PnL per trade in SOL / mean cost (per-trade expectancy ratio) | `logistic(x; mid=0.05, k=10)` | 0.20 |
| C3 | Hit quality | Win rate, with wins defined as ROI ≥ +10% (small wins don't count) | linear clamp (0.25 → 0, 0.65 → 1) | 0.15 |
| C4 | Profit factor | Σ gains / Σ losses | `clamp(log2(pf) / log2(4))` (pf=1 → 0, pf≥4 → 1) | 0.15 |
| C5 | Early entry | Median `(max price in 60 min after buy) / entry price` over buys where price history is available | `clamp((x − 1) / 2)` (3× → 1) | 0.15 (redistributed if < 5 observations) |
| C6 | Consistency | Share of active weeks with positive PnL | linear | 0.10 |

`raw_score = Σ w_i · C_i` (weights renormalized over available components).

### 6.3 Confidence (sample-size shrinkage)

`confidence = n / (n + k)` with `n` = closed round trips in window and `k = wallet.scoring.shrinkage_k` (default 10).
`score = confidence · raw_score + (1 − confidence) · prior` where `prior = wallet.scoring.prior` (default 0.30).
If `n < wallet.min_sample_trades` (default 10), the wallet can be stored but is **not smart** regardless of score (flag `INSUFFICIENT_SAMPLE`).

### 6.4 Penalty flags (deterministic heuristics)

| Flag | Rule (defaults) | Effect |
|---|---|---|
| `BOT_LIKE` | Median hold < 20 s **and** > 100 swaps/day average | score × 0.5 |
| `SNIPER_ONLY` | > 80% of buys in the same slot as pool creation, or the next 2 slots | score × 0.6 (their edge is unreplicable at our latency) |
| `INSIDER_SUSPECT` | Received the token by transfer from the mint authority/creator before trading, or bought in the pool-creation tx | `BLOCKED` recommendation (not automatic), score × 0.3 |
| `WASH_SUSPECT` | ≥ 5 round trips on the same token within 1 h with near-zero net | score × 0.5 |
| `STALE` | No swaps in the last 7 days | score × 0.8 |

Flags are stored in `intel.wallet_scores.flags`. Multipliers compound. The final score is clamped to [0,1].

### 6.5 Output

```ts
interface WalletScore {
  wallet: string; modelVersion: 'wallet-score@1.0.0'; computedAt: EpochMicros;
  windowStart: EpochMicros; windowEnd: EpochMicros;
  score: number; confidence: number; sampleSize: number;
  components: Record<'C1'|'C2'|'C3'|'C4'|'C5'|'C6', { raw: string | null; normalized: number | null; weight: number }>;
  flags: WalletFlag[];
}
```

Numbers are stored as strings in jsonb (decimal precision), with normalized values as JS numbers (non-monetary).

### 6.6 Recomputation

- On backfill completion.
- On each closed round trip observed live (incremental).
- Daily full recompute for all `TRACKED`/`CANDIDATE` wallets (scheduler, worker thread).
- A new score row with `is_current = true` flips the previous one to false (in one transaction), then emits `wallet.score.updated`.

## 7. Live swap path

1. `wallet.swap.detected` arrives (09 §5).
2. wallet-intel persists it to `intel.wallet_swaps` (`origin = LIVE`) asynchronously. The hot path does not wait for this.
3. If the wallet is `TRACKED`, its current score ≥ `wallet.min_wallet_score`, it has no `INSUFFICIENT_SAMPLE` flag and is not `BLOCKED`, and `side = BUY` and quote value ≥ `wallet.min_buy_value_sol` (default 0.1 SOL, to filter dust/test buys): emit `smart.buy.detected{walletScoreSnapshot, swap}`.
4. If `side = SELL` and the wallet holds a smart score: emit `smart.sell.detected` (used by the exit rule `SMART_EXIT` and wave exhaustion).

The score used is the one current **at event processing time**, captured in the event (`walletScoreSnapshot`) for reproducibility.

## 8. Avoiding look-ahead and survivorship bias

- In replay, wallet scores are resolved **as of the simulated time**: the latest `intel.wallet_scores` row with `computed_at ≤ now` (score rows are append-only, which makes this possible). A replay with `wallet.score_resolution = FROZEN_AT_START` uses scores as of the replay start.
- Validation reports ([30](30-development-plan.md), Phase 25) must use wallets whose scoring window ends **before** the evaluation window (out-of-sample). The run records which scoring cut-off was used.

## 9. AI classification (advisory only)

The `WALLET_CLASSIFIER` agent ([13](13-ai-agent-architecture.md)) may label a wallet's style (`SNIPER`, `MOMENTUM`, `SWING`, `INSIDER_LIKE`, `BOT_LIKE`, `UNCLEAR`) with rationale. The label is shown in the UI next to the deterministic score. It **never** changes the score, status or flags automatically. The operator may act on it manually.

## 10. Tests (minimum)

- FIFO round trips: partial sells, multiple buys, dust threshold, unknown-cost transfers (property-based: Σ realized PnL = Σ proceeds − Σ consumed cost).
- Score determinism: same inputs → same score (snapshot tests per fixture wallet).
- Each penalty flag triggers on a crafted fixture and not on its near-miss.
- Look-ahead guard: replay at time T never sees scores computed after T.
- Live path: a TRACKED wallet with a sufficient score and a BUY emits `smart.buy.detected`. A BLOCKED or insufficient-sample wallet does not.
