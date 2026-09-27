# 17 — Economics Engine Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [10](10-market-data-spec.md) §4–5, [27](27-data-provider-reference.md), [03](03-system-requirements.md) |
| Downstream | [16](16-paper-trading-spec.md), [15](15-risk-engine-spec.md), [18](18-portfolio-accounting-spec.md) |
| Used by phases | 11 (implementation), 13, 14 |

## 1. Purpose and principles

The economics engine answers: *if we had actually sent this swap, what would we have paid and received?*

- **Pure functions only.** No I/O, clock or randomness inside. Randomness (latency, failure, MEV draws) is passed in as pre-drawn uniform values from the run's seeded `Rng`, so results are reproducible and testable.
- **Integer math where the chain uses integers** (raw amounts, lamports), floor rounding exactly as programs do (verified per venue). Decimal math is used for prices and ratios.
- Model version constant: `economics@1.0.0`, recorded on every fill.
- **Uncalibrated parameters are labelled.** Anything not measured from real data (latency, failure rate, MEV) carries `calibration: 'UNCALIBRATED'` in config and in the validation report, and must be varied in sensitivity analysis ([19](19-backtesting-spec.md) §7).

## 2. Cost components

| Component | Modelled as | Source of parameters |
|---|---|---|
| DEX LP fee | Venue fee model applied per the program's arithmetic | `ref.venues.fee_model` (verified from official program source/IDL in Phase 03/06) |
| Protocol / creator fees | Same (separate fields) | Same |
| Price impact | CP curve math from reserves at execution time | T1 pool state |
| Adverse movement | Mid at execution vs mid at decision (emerges from real market data during latency) | T1 pool state over time |
| Slippage (vs expectation) | Actual out vs expected out at decision | Derived |
| Network base fee | `5,000 lamports × signatures` (1 signature) per transaction, including failed ones | Solana constant (verify in Phase 03) |
| Priority fee | `compute_unit_price (micro-lamports) × compute_unit_limit / 10⁶` | Config FIXED or Helius estimate |
| ATA rent | Rent-exempt minimum for a token account (165 bytes for SPL Token; larger for Token-2022 with extensions), locked on first buy of a mint, refunded on close | `getMinimumBalanceForRentExemption` at startup (cached) |
| Tip (Jito etc.) | Optional fixed lamports per tx | Config (default 0; future) |
| MEV (sandwich) | Probabilistic degradation bounded by slippage tolerance | Config, UNCALIBRATED |
| Execution failure | Probabilistic + deterministic causes | Config, UNCALIBRATED + rules |
| Latency | Sampled delay between decision and execution | Config distribution, UNCALIBRATED |

## 3. Venue fee model schema

```ts
const FeeModel = z.object({
  kind: z.enum(['CP_INPUT_FEE', 'CP_OUTPUT_FEE', 'CURVE_QUOTE_SIDE_FEE']),
  lpFeeBps: z.number().int().min(0).max(1000),
  protocolFeeBps: z.number().int().min(0).max(1000),
  creatorFeeBps: z.number().int().min(0).max(1000).default(0),
  // For curves: fee always charged on the quote (SOL) side: added to SOL in for buys, deducted from SOL out for sells
  rounding: z.enum(['FLOOR', 'CEIL_FEE']),   // how the program rounds fee amounts
  verifiedAt: z.string().datetime().nullable(),
  verificationRef: z.string().nullable(),
});
```

Fee values are **not** hard-coded in the economics module. They come from `ref.venues` rows, seeded by a migration with `supported = false`, and flipped to `supported = true` only after verification ([27](27-data-provider-reference.md) §10). Candidate values to verify (UNVERIFIED): pump.fun curve ~1% total, PumpSwap ~0.25–0.30% total, Raydium AMM v4 0.25%, Raydium CPMM tiered (pool-specific config account). Where a venue's fee is **pool-specific** (e.g. CPMM config accounts, dynamic fee tiers), the decoder reads the fee from the pool state and the fee model says `source: 'POOL_STATE'`.

## 4. Swap math

### 4.1 CP AMM, fee on input (`CP_INPUT_FEE`)

```text
fee_total   = ceil_or_floor(amount_in × (lp + protocol + creator) / 10_000)   // per venue rounding
in_net      = amount_in − fee_total
amount_out  = floor( reserve_out × in_net / (reserve_in + in_net) )
```

### 4.2 Virtual-reserve bonding curve (`CURVE_QUOTE_SIDE_FEE`)

Buy (SOL → token), with `vS`, `vT` virtual reserves:
```text
fee         = fee(sol_in)                    // on SOL side (verify: included in or added to sol_in)
sol_net     = sol_in − fee
tokens_out  = floor( vT × sol_net / (vS + sol_net) )
tokens_out  = min(tokens_out, real_token_reserves)     // cannot exceed real reserves → otherwise INSUFFICIENT_LIQUIDITY or partial per program rule (verify)
```
Sell (token → SOL):
```text
sol_gross   = floor( vS × tokens_in / (vT + tokens_in) )
fee         = fee(sol_gross)
sol_out     = sol_gross − fee ; require sol_gross ≤ real_sol_reserves
```
The exact rounding and fee placement MUST match the verified program source. Unit tests compare against at least 5 recorded real trades per venue (from logs: input amounts plus observed outputs), matching within ±1 raw unit.

### 4.3 Max input for a price-impact limit (used by risk reduction)

For CP with input fee factor `γ = 1 − f`, pure-curve impact `I` (fraction) relative to mid for input `x`:
`exec_price_no_fee = x / out_no_fee`, with `out_no_fee = R_out·x/(R_in + x)` → `exec/mid = (R_in + x)/R_in` → `I = x / R_in`.
So `x_max = I_max × R_in` (pre-fee input, in reserve units). Tested against brute-force search.

### 4.4 Definitions (bps, signed so that + means worse for us)

| Metric | Formula |
|---|---|
| `mid_price` | reserve-based quote per base (10 §5) |
| `exec_price` | quote spent / base received (buy); quote received / base sold (sell), **including** fees |
| `price_impact_bps` | Curve-only impact: `(exec_price_no_fee / mid_at_exec − 1) × 10⁴` (buy); `(1 − exec_price_no_fee / mid_at_exec) × 10⁴` (sell) |
| `fee_bps` | fees in quote terms / notional |
| `adverse_move_bps` | `(mid_at_exec / mid_at_decision − 1) × 10⁴` for buys; negated for sells |
| `slippage_bps` | `(expected_out_at_decision − actual_out) / expected_out_at_decision × 10⁴` |
| `total_cost_usd` | LP+protocol+creator fees + network + priority + tip + (impact + adverse move, as value vs mid at decision) in USD. Rent is excluded (refundable) and reported separately. |

## 5. Latency model

```ts
LatencyModel = {
  kind: 'LOGNORMAL', medianMs: 1200, p95Ms: 4000, minMs: 400, maxMs: 20000,   // UNCALIBRATED defaults
  congestionMultiplier: { source: 'NONE' | 'PRIORITY_FEE_PERCENTILE', ... }
}
```
- `μ = ln(median)`, `σ = (ln(p95) − μ) / 1.6449`. Sample `L = clamp(exp(μ + σ·Φ⁻¹(u)))` with `u` drawn from `rng.fork('latency')`.
- The simulated execution time is `decided_at + system_overhead + L`. `system_overhead` is the **measured** time until the order is persisted (live). In replay it is the recorded value from the original run if one exists, else the configured `economics.latency.replay_overhead_ms` (default 50).
- **Calibration limitation**: without sending real transactions, landing latency cannot be measured directly. Phase 25 reports results under latency ×0.5 / ×1 / ×2 / ×4, and Phase 26 (shadow) can compare against observed landing times of tracked-wallet follow-on trades as a proxy. This is documented as a known limitation.

## 6. Failure model

Evaluated in this order at execution time; the first cause that applies wins:

| Cause | Rule | Fees charged |
|---|---|---|
| `STALE_MARKET_DATA` | pool state at exec time not FRESH (> `freshness.pool_state.fill_ms`) | None (not sent: a real bot would not send without a quote) |
| `POOL_UNAVAILABLE` | pool closed/migrated with no successor, or curve complete for curve venue buys | None |
| `INSUFFICIENT_LIQUIDITY` | output < 1 raw unit, or exceeds real reserves | Base + priority (the tx would fail on chain) |
| `SIMULATED_TX_FAILURE` | `u_fail < p_fail`, where `p_fail = economics.failure.base_prob` (0.03, UNCALIBRATED) × congestion factor | Base + priority |
| `SLIPPAGE_EXCEEDED` | `actual_out < min_out` (after the MEV adjustment) | Base + priority |
| — | otherwise → **FILLED** | Base + priority + DEX fees |

`u_fail` comes from `rng.fork('failure')`. Failed-with-fee outcomes post a `FAILED_TX_FEE` ledger transaction ([18](18-portfolio-accounting-spec.md)).

## 7. MEV / sandwich model (UNCALIBRATED, enabled by default)

- With probability `economics.mev.sandwich_prob` (0.10) for **buys** (and `sell_prob` 0.05 for sells), the execution is degraded by `min(extra, remaining_tolerance)`, where `extra = economics.mev.extraction_fraction_of_tolerance` (0.5) × (`max_slippage_bps` − current slippage). This models that attackers extract up to the tolerance.
- This can push fills toward `min_out` but not beyond it (a sandwich that would breach `min_out` fails the victim's tx; the model then yields `SLIPPAGE_EXCEEDED` with probability `economics.mev.fail_instead_prob`, default 0.2).
- Rationale: memecoin swaps with wide tolerances are common sandwich targets. Omitting this would be optimistic. Parameters are varied in sensitivity analysis.

## 8. Economic reality gate (cost ratio)

For entries, risk computes the estimated round-trip cost ratio using the current state and assumptions:
```text
fixed    = 2 × (base_fee + priority_fee + tip)               // buy + sell
variable = 2 × fee_bps_notional + 2 × est_impact_bps_notional
ratio    = (fixed_in_usd + variable_in_usd) / size_usd
```
If `ratio > risk.max_cost_ratio` (0.25) → reject `RSK_COST_RATIO`. At $2 size and elevated priority fees this gate will bind. That is intended: it makes the cost of small sizes explicit.

ATA rent is not a cost but a **cash requirement**. `RSK_CASH_AVAILABLE` includes it. Validation reports show rent-locked cash as a share of equity over time.

## 9. Known model limitations (documented, accepted for MVP)

1. **Own-trade market impact is not fed back into the pool.** Paper trades don't move real reserves. At $2 size against ≥ $10k liquidity, the effect is < 0.1% (bounded by `RSK_PRICE_IMPACT`).
2. **Latency, failure and MEV parameters are uncalibrated.** Mitigated with sensitivity analysis, conservative defaults, and the shadow phase.
3. **Transaction ordering within a slot is unknown.** Execution uses the latest state at or before the simulated execution time.
4. **Route choice**: we assume a direct swap on the primary pool (no aggregator routing). This is conservative for price. Jupiter routing is a shadow-phase comparison.
5. **WSOL wrap/unwrap** is assumed to happen within the swap transaction (temporary account created and closed in the same tx: rent-neutral, compute included in the CU limit).

## 10. Interfaces

```ts
interface Economics {
  quoteSwap(state: PoolReserves, venue: VenueFeeModel, side: Side, amountInRaw: bigint): SwapQuote;
  maxInputForImpact(state: PoolReserves, side: Side, maxImpactBps: number): bigint;
  networkCost(params: { priorityFeeLamports: bigint; signatures: number; tipLamports: bigint }): NetworkCost;
  sampleLatency(model: LatencyModel, u: number): DurationMicros;
  resolveExecution(input: {
    order: PaperOrder; stateAtExec: QualifiedPoolState; venue: VenueFeeModel;
    uFail: number; uMev: number; uMevFail: number; ataExists: boolean; rentLamports: bigint;
    closeAtaOnFullExit: boolean; isFullExit: boolean; solUsd: Decimal; midAtSignal?: Decimal;
  }): ExecutionResult;   // FILLED with full breakdown | FAILED with cause and fees charged
}
```

`ExecutionResult.breakdown` is stored verbatim in `sim.fills.breakdown`.

## 11. Tests (minimum)

- Golden tests per venue: recorded real trades reproduced within ±1 raw unit (requires Phase 03/07 fixtures).
- Property tests: `amount_out` monotonic increasing in `amount_in`; `exec_price ≥ mid` for buys (≤ for sells); fees ≥ 0; k non-decreasing after a swap with fees.
- `maxInputForImpact` agrees with brute force within 1 raw unit.
- Latency sampler reproduces the configured median/p95 over 100k draws (±2%).
- Failure ordering: STALE beats SLIPPAGE, and so on (table-driven).
- MEV never makes a fill worse than `min_out`.
- Determinism: the same `u` values give the same result.
