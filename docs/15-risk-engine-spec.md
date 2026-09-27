# 15 — Risk Engine Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [14](14-strategy-engine-spec.md), [18](18-portfolio-accounting-spec.md), [10](10-market-data-spec.md), [17](17-economics-engine-spec.md) |
| Downstream | [16](16-paper-trading-spec.md), [28](28-decision-logging-spec.md) |
| Used by phases | 13 |

## 1. Principles

- Risk is **deterministic code**. No AI input can loosen a limit.
- Every proposed order (entry and exit, including manual) is evaluated. **Every check** runs and records `{code, passed, value, limit, severity}`, even after an earlier failure, so the audit shows all violations.
- The verdict is one of `APPROVED`, `APPROVED_REDUCED` (size reduced to fit a limit), or `REJECTED`.
- Exits are never blocked by exposure, frequency or loss limits. Exits can only be blocked (or deferred) by conditions that would make a fill fake: stale data, or an unavailable pool.
- Risk reads portfolio and market state through read-only views and has no side effects except writing its verdict.
- Model version: `risk@1.0.0`.

## 2. Interface

```ts
interface ProposedOrder {
  runId: RunId; decisionId: DecisionId; positionId?: PositionId;
  side: 'BUY' | 'SELL'; intent: OrderIntent;
  tokenMint: string; poolAddress: string;
  amountInRaw: bigint;                // lamports (BUY) / token raw (SELL)
  quote: SwapQuote;                   // from economics at decision time
  maxSlippageBps: number; priorityFeeLamports: bigint;
  signal?: EntrySignal;               // entries
}
interface RiskEngine {
  evaluate(order: ProposedOrder, ctx: RiskContext): RiskDecision;   // pure
}
interface RiskContext {
  now: EpochMicros;
  portfolio: PortfolioSummary;        // cash, reserved cash, exposure, open positions, daily realized pnl, peak equity, consecutive losses, trades last hour
  market: PoolQualitySnapshot;        // reserves quality/age, liquidity, venue support
  token: TokenSafetySnapshot;
  config: RiskConfig;
}
```

`evaluate` is pure (a table-driven unit test per check). The service wraps it, persists `strategy.risk_decisions`, and emits `risk.evaluated`.

## 3. Check catalog (entries)

| Code | Check | Default limit | On fail |
|---|---|---|---|
| `RSK_POSITION_SIZE` | `size_usd ≤ sizing.max_position_size_usd` | 2.00 | Reduce to the limit (`APPROVED_REDUCED`) |
| `RSK_MIN_ORDER` | `size_usd ≥ sizing.min_order_usd` (after any reduction) | 0.50 | Reject |
| `RSK_CASH_AVAILABLE` | Free cash ≥ size + estimated fees + priority fee + ATA rent (if a new token account is needed) + `risk.cash_buffer_lamports` | buffer 0.01 SOL | Reduce if possible, else reject |
| `RSK_MAX_OPEN_POSITIONS` | open + opening positions < `sizing.max_open_positions` | 3 | Reject |
| `RSK_MAX_EXPOSURE` | exposure_usd + size ≤ `sizing.max_total_exposure_usd` | 6.00 | Reduce, else reject |
| `RSK_TOKEN_CONCENTRATION` | no existing position in the token | — | Reject |
| `RSK_DAILY_LOSS` | today's realized PnL (UTC) + unrealized PnL of open positions > −`risk.max_daily_loss_usd` | 4.00 | Reject + pause entries (`DAILY_LOSS_LIMIT`) |
| `RSK_DRAWDOWN_KILL` | equity drawdown from peak < `risk.max_drawdown_pct` | 40% | Reject + pause (`DRAWDOWN_KILL`, manual resume) |
| `RSK_CONSECUTIVE_LOSSES` | consecutive losing closed trades < `risk.max_consecutive_losses` or cooldown elapsed | 5 / 60 min | Reject + pause (`CONSECUTIVE_LOSS_COOLDOWN`) |
| `RSK_TRADE_FREQUENCY` | entries in the last 60 min < `risk.max_entries_per_hour` | 6 | Reject |
| `RSK_SLIPPAGE_TOLERANCE` | `max_slippage_bps ≤ risk.max_slippage_bps` | 1,000 | Reject (config/programming error) |
| `RSK_PRICE_IMPACT` | quote `price_impact_bps ≤ risk.max_price_impact_bps` | 300 | Reduce size until satisfied (≥ min order), else reject |
| `RSK_COST_RATIO` | estimated round-trip fixed + variable costs / size ≤ `risk.max_cost_ratio` | 0.25 | Reject (the economic reality gate, [17](17-economics-engine-spec.md) §8) |
| `RSK_MIN_LIQUIDITY` | liquidity_usd ≥ `risk.min_liquidity_usd` | 10,000 | Reject |
| `RSK_POOL_SUPPORTED` | venue supported, pool `ACTIVE`, curve not complete (for curve venues) | — | Reject |
| `RSK_DATA_FRESH` | pool reserves FRESH for entry (≤ 3 s), SOL/USD FRESH, wallet stream healthy | [09](09-real-time-data-architecture.md) §7 | Reject |
| `RSK_ENTRY_LATENCY` | `now − seed smart buy event_time ≤ risk.max_entry_latency_s` | 90 s | Reject |
| `RSK_SIGNAL_LATENCY` | `now − received_at of latest triggering event ≤ risk.max_trigger_age_ms` | 5,000 ms | Reject |
| `RSK_WAVE_DATA_GAP` | no unresolved data gap overlapping the wave window | — | Reject |

## 4. Token safety gates (entries)

| Code | Check | Default |
|---|---|---|
| `RSK_MINT_AUTHORITY` | mint authority revoked (null) | required (`risk.require_mint_authority_revoked = true`) |
| `RSK_FREEZE_AUTHORITY` | freeze authority revoked | required |
| `RSK_TOKEN2022_EXTENSIONS` | no forbidden extensions (`transferFeeConfig`, `permanentDelegate`, `nonTransferable`, `transferHook`, frozen default state) | forbidden list in config |
| `RSK_HOLDER_CONCENTRATION` | top-10 holders (excluding pool vaults) ≤ `risk.max_top10_holder_pct` | 50% (skipped with flag if unknown and `risk.require_holder_data = false`) |
| `RSK_TOKEN_AGE` | token age ≤ `risk.max_token_age_s` when known | 43,200 s |
| `RSK_SAFETY_FRESH` | latest safety observation age ≤ `reference.safety_max_age_minutes` | 15 |

## 5. Check catalog (exits)

| Code | Check | On fail |
|---|---|---|
| `RSK_EXIT_POSITION_EXISTS` | position OPEN/CLOSING with qty ≥ requested | Reject (programming error, alert) |
| `RSK_EXIT_DATA_FRESH` | pool reserves FRESH for fill (≤ 3 s) | `REJECTED` with `retryable = true`. ExitManager retries (§5.3 of [14](14-strategy-engine-spec.md)). |
| `RSK_EXIT_POOL_AVAILABLE` | pool ACTIVE (or migrated pool resolved) | Retryable reject + alert |
| `RSK_EXIT_SLIPPAGE_CAP` | `max_slippage_bps ≤ exit.max_emergency_slippage_bps` | Reduce tolerance to the cap |

Exit checks explicitly **skip** exposure, daily loss, frequency, and cost ratio.

## 6. Sizing reduction algorithm

When several checks allow reduction, the approved size = the minimum of every check's maximum allowed size. For `RSK_PRICE_IMPACT`, the maximum size is solved from the CP formula (closed form for x·y=k with fee; [17](17-economics-engine-spec.md) §4.3). If the approved size < `sizing.min_order_usd` → `REJECTED`.

## 7. Kill switch and pauses

- `DRAWDOWN_KILL` and `DAILY_LOSS_LIMIT` are raised by risk as **entry pause reasons** ([14](14-strategy-engine-spec.md) §7), not just rejections, so subsequent signals are rejected quickly and visibly.
- Pauses never cancel exits.

## 8. Configuration

All limits are under `sizing.*` and `risk.*` in the strategy config ([26](26-configuration-reference.md)). Validation rules include: `max_total_exposure_usd ≥ position_size_usd`, `max_open_positions ≥ 1`, `max_daily_loss_usd < initial_balance_usd`, and `max_slippage_bps ≤ 5,000`.

## 9. Tests (minimum)

- Table-driven: each check passes at the limit and fails just beyond it (boundary tests).
- All checks recorded even when the first fails.
- Reduction: exposure and impact reductions compose (minimum), and falling below the minimum order → reject.
- Exits bypass entry-only limits (daily loss breached, exits still approved).
- Pause reasons raised and cleared correctly (UTC day rollover on the simulated clock).
- Property test: the approved size never exceeds any limit. Cash never goes negative across a random order sequence (with portfolio).
