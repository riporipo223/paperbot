# 18 — Portfolio Accounting Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [16](16-paper-trading-spec.md), [17](17-economics-engine-spec.md), [07](07-database-schema.md) §3.5 |
| Downstream | [15](15-risk-engine-spec.md), [20](20-dashboard-spec.md), [19](19-backtesting-spec.md) |
| Used by phases | 12 (implementation), 14, 15, 25 |

## 1. Principles

- **Double-entry ledger** in integer base units per asset ([ADR-0013](32-architecture-decision-records.md#adr-0013-double-entry-ledger)). Every movement of value is a balanced journal transaction.
- The ledger is the **source of truth** for money. Positions, cash, snapshots and analytics are derived and must reconcile.
- Cash asset is **SOL** by default (`run.cash_asset = SOL`), because Solana memecoin trades settle in SOL ([ADR-0006](32-architecture-decision-records.md#adr-0006-sol-native-cash-accounting-with-usd-reporting)). Reporting is in USD and SOL. `USDC` cash is supported by the schema but disabled in the MVP config.
- Accounting never uses floats. USD values are `Decimal`, computed at display/snapshot time from raw amounts × recorded SOL/USD.

## 2. Sign convention

`amount_raw` is signed: **debit positive, credit negative**. For each `(transaction_id, asset)`, Σ `amount_raw` = 0.

## 3. Chart of accounts

| Account | Type | Assets | Meaning |
|---|---|---|---|
| `EQUITY_FUNDING` | Equity | SOL | Virtual funding at run start (credit) |
| `CASH` | Asset | SOL | Free and reserved cash (reservations are not ledger entries; see §6) |
| `POSITION` | Asset | token mint | Token holdings per mint (`position_id` set) |
| `RENT_DEPOSIT` | Asset | SOL | Rent locked in open token accounts |
| `EXCHANGE` | Clearing | SOL, token mint | Counterparty clearing for swaps (trading-account method) |
| `FEE_LP` | Expense | SOL or token | DEX LP fee |
| `FEE_PROTOCOL` | Expense | SOL or token | Protocol/creator fees |
| `FEE_NETWORK` | Expense | SOL | Base fee |
| `FEE_PRIORITY` | Expense | SOL | Priority fee |
| `FEE_TIP` | Expense | SOL | Tips (future) |
| `ADJUSTMENT` | Equity | any | Operator/recovery corrections (should stay empty; alerts if used) |

## 4. Journal templates

**RUN_FUNDING** (initial SOL = floor(initial_balance_usd / sol_usd_at_start × 10⁹)):
```text
SOL: CASH +F ; EQUITY_FUNDING −F
```

**FILL, BUY** (CP input-fee venue; `a` = lamports in, fees `lp`, `pr` inside `a`; network `n`, priority `p`, rent `r` if new ATA; tokens out `q`):
```text
SOL:   CASH −(a + n + p + r) ; EXCHANGE +(a − lp − pr) ; FEE_LP +lp ; FEE_PROTOCOL +pr ; FEE_NETWORK +n ; FEE_PRIORITY +p ; RENT_DEPOSIT +r
TOKEN: POSITION +q ; EXCHANGE −q
```

**FILL, SELL** (tokens in `q`; gross SOL from curve `g`; fees `lp`, `pr` deducted from `g`; refund `r` if closing the ATA):
```text
TOKEN: POSITION −q ; EXCHANGE +q
SOL:   EXCHANGE −g ; FEE_LP +lp ; FEE_PROTOCOL +pr ; FEE_NETWORK +n ; FEE_PRIORITY +p ;
       CASH +(g − lp − pr − n − p + r) ; RENT_DEPOSIT −r
```
Venue variants (e.g. fee charged on the output token) are expressed with the same accounts in the fee's asset.

**FAILED_TX_FEE** (tx failed on chain):
```text
SOL: CASH −(n + p) ; FEE_NETWORK +n ; FEE_PRIORITY +p
```

Each template is a pure function `journalFor(executionResult) → LedgerTransaction`, property-tested for balance.

## 5. Positions and PnL

- One position per (run, token) at a time (DB unique partial index).
- **Cost basis method: weighted average cost in lamports.** Entry cost basis = `a + n + p` (SOL spent on the swap including DEX fees and tx fees; rent excluded because it is refundable).
- Additional buys into an open position are not produced by v1 strategy, but the formula supports them: `basis += a + n + p`, `qty += q`.
- Sell of `q` from `qty`: `released = floor(basis × q / qty)`. `proceeds_net = g − lp − pr − n − p`. `realized += proceeds_net − released`. `basis −= released`. `qty −= q`.
- When `qty = 0` → position `CLOSED`, with `realized_pnl_quote_raw` final. Rounding remainder: any residual `basis` at close is added to realized (so the per-position sum is exact).
- **Unrealized PnL** (open): `mark_value − basis`, where `mark_value` depends on `portfolio.mark_method`:
  - `LIQUIDATION` (default, conservative): economics quote for selling the full `qty` at the current FRESH state, net of DEX fees and one network+priority fee.
  - `MID`: `qty × mid`.
- Marks use FRESH state for `PORTFOLIO_MARK` (≤ 30 s). If stale, the last mark is used with `marks_quality = STALE` shown.
- MFE/MAE (`max_favorable_pct`, `max_adverse_pct`) are updated on each FRESH mark.

### 5.1 USD reporting

- `realized_pnl_usd(trade) = realized_lamports / 1e9 × sol_usd_at_exit_fill`.
- `equity_usd = (cash + rent_deposit) / 1e9 × sol_usd_now + Σ mark_value_usd`.
- Equity change decomposes into **trading PnL** (Σ realized + unrealized, in SOL, valued at the respective SOL/USD) and **SOL revaluation** (the effect of SOL/USD moving on cash held). Both are shown so strategy performance isn't confused with SOL price moves.
- ROI is reported in both SOL terms (`equity_sol / initial_sol − 1`, the strategy's own performance) and USD terms.

## 6. Reservations

- Reservations (cash for pending BUYs, tokens for pending SELLs) are held in memory by the portfolio service and derived from `SUBMITTED` orders on restart. They are not ledger entries (nothing has moved yet).
- `free_cash = CASH balance − Σ reserved`. Risk uses `free_cash`.

## 7. Analytics (definitions)

Computed per run by a pure `computePerformance(trades, snapshots, orders, latencySamples)`:

| Metric | Definition |
|---|---|
| Total trades | Closed positions (each position is one trade, however many exit orders) |
| Winning / losing / breakeven trades | Net realized PnL > 0 / < 0 / = 0 (SOL) |
| Win rate | winning / total |
| Average win / average loss | Mean net PnL of winners / losers (SOL and USD) |
| Largest win / largest loss | Max / min net PnL |
| Profit factor | Σ winning PnL / |Σ losing PnL| (∞ shown as "n/a: no losses") |
| Expectancy | Mean net PnL per trade (SOL, USD), plus **expectancy ratio** = mean PnL / mean entry cost |
| Realized PnL | Σ closed net PnL |
| Unrealized PnL | Σ open (mark method stated) |
| Net PnL after costs | Realized + unrealized (already net of all fees). Also shown: **gross PnL before costs** and **total costs by component**. |
| ROI | (equity − initial) / initial (SOL and USD) |
| Maximum drawdown | Max peak-to-trough decline of equity (SOL basis primary, USD secondary), from snapshots + every fill |
| Consecutive losses | Maximum streak and current streak |
| Average holding time | Mean and median `closed_at − opened_at` |
| Average execution latency | Mean/median/p95 of `executed_at − decided_at` (simulated), plus system latency `e2e_system` (measured) |
| Order outcomes | Fill rate, failure counts by reason, stale-exit incidents, exit attempts per trade |
| Cost breakdown | LP, protocol, network, priority, tip, price impact, adverse move, MEV degradation (USD and % of notional) |
| Exposure | Time-weighted average exposure; % of time with ≥ 1 open position |
| Rent lock | Average and max rent-locked cash as % of equity |
| Sample-size warning | Shown when trades < 30 ("results not statistically meaningful") |

Win rate is always shown next to profit factor, expectancy and drawdown, never alone (FR-PFL-003).

## 8. Snapshots

- `sim.portfolio_snapshots` every `portfolio.snapshot_interval_s` (60), plus on each fill, run start and run end.
- Snapshots are derived. A rebuild job can regenerate them from the ledger + historical marks (from `market.pool_states`) for validation.

## 9. Invariants (checked after every ledger transaction)

1. Per transaction and asset: Σ = 0.
2. `CASH` balance ≥ 0.
3. For each open position: `POSITION` balance for (run, mint) = `positions.qty_raw`.
4. `RENT_DEPOSIT` balance = Σ rent of open ATAs.
5. `EQUITY_FUNDING` changes only via `RUN_FUNDING`.
6. Closed positions have `qty_raw = 0` and `basis = 0`.
7. Σ realized PnL over closed positions = −(`EXCHANGE` SOL balance attributable to closed positions) − Σ fees on those positions (reconciliation identity, checked by the nightly/job test).

A violation → run `HALTED_INVARIANT`, FATAL system event, no further orders (FR-PFL-004).

## 10. Tests (minimum)

- Journal templates balance (property-based over random amounts).
- Round trip: fund → buy → partial sell → full sell → invariants hold. Realized PnL equals the hand calculation (golden).
- Rent lock/refund flows.
- Failed tx fees reduce cash and do not touch positions.
- Analytics on a synthetic trade set match hand-computed values (profit factor, expectancy, drawdown, streaks).
- Restart: the portfolio rebuilt from the ledger equals the in-memory state before the restart.
