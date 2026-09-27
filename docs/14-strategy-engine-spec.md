# 14 — Strategy Engine Specification (Decision Engine, Exit Manager, Runs)

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [12](12-wave-detection-spec.md), [15](15-risk-engine-spec.md), [16](16-paper-trading-spec.md), [13](13-ai-agent-architecture.md), [26](26-configuration-reference.md) |
| Downstream | [28](28-decision-logging-spec.md), [19](19-backtesting-spec.md), [20](20-dashboard-spec.md) |
| Used by phases | 15 (core), 16 (AI hook), 17 (replay) |

## 1. Components

| Component | Responsibility |
|---|---|
| `RunManager` | Create/start/stop runs, bind strategy + config versions, seed RNG, fund the paper account, restore on restart |
| `WatchSetPolicy` | Translate waves/positions into watch requests ([10](10-market-data-spec.md) §3) |
| `DecisionEngine` | Pure `decide(input) → Decision` for entries (and manual/exit commands) |
| `ExitManager` | Evaluate exit rules for open positions on state changes and ticks → exit decisions |
| `OrderPlanner` | Turn an `ENTER`/`EXIT` decision into a proposed order (size, slippage, priority fee) |
| `EntryPause` | Aggregate automatic and manual pause reasons |

The strategy module wires these together: `signal → DecisionEngine → OrderPlanner → risk → execution`.

## 2. Strategy versioning

- Strategy id: `smart-money-wave`. Version: semver, registered in `strategy.strategy_versions` with the git SHA.
- The **code** version bumps (strategy semver) when decision/exit logic changes. The **config** version (content hash) changes when parameters change. Model versions (`wave-score@x`, `risk@x`, `decision@x`, `economics@x`, `wallet-score@x`) are constants in code and recorded on every record they produce.
- Bump rules: logic change that can alter a decision → minor bump (pre-1.0). Pure refactor with identical replay output → patch. A replay-equality test on the golden scenario set proves "identical".

## 3. DecisionEngine `decision@1.0.0`

### 3.1 Input

```ts
interface EntryDecisionInput {
  signal: EntrySignal;                       // from wave
  now: EpochMicros;
  market: { pool: PoolQualitySnapshot; solUsd: Q<string> };
  portfolio: PortfolioSummary;               // read-only snapshot
  entryPause: { paused: boolean; reasons: string[] };
  ai: AiAdvisory | null;                     // present only if returned within timeout (13 §6)
  config: StrategyConfig;
}
```

### 3.2 Logic (ordered; first matching rule wins)

1. `entryPause.paused` → `REJECT [ENTRIES_PAUSED:<reasons>]`.
2. `signal.worstQuality ≠ FRESH` or a market input is not FRESH at `now` → `REJECT [DATA_STALE:<inputs>]`.
3. `now − signal.signalAt > strategy.max_signal_age_ms` (default 2,000) → `REJECT [SIGNAL_EXPIRED]`.
4. Price moved since signal: `|mid_now / signal.midPrice − 1| > strategy.max_price_drift_since_signal_pct` (default 10%) → `REJECT [PRICE_DRIFT]`.
5. An open position already exists for the token → `REJECT [ALREADY_IN_POSITION]`.
6. `ai.mode = VETO` and `ai` present and `ai.verdict = 'VETO'` with `ai.confidence ≥ ai.veto_min_confidence` → `REJECT [AI_VETO]` (`ai_effect = VETOED`).
7. `ai.mode = VETO` and `ai` absent (timeout/error/disabled): apply `ai.on_timeout`: `PROCEED` (default) → continue; `REJECT` → `REJECT [AI_UNAVAILABLE]`.
8. Otherwise → `ENTER` with `reason_codes = ['WAVE_CONFIRMED', ...gate summaries]`.

`WAIT` is used when a decision is deferred by design. In v1.0 it is produced only when `ai.mode = VETO`, `ai.on_timeout = WAIT`, and the AI call is still in flight within `ai.entry_review_timeout_ms`. The engine re-decides once when the AI result arrives or the timeout fires.

The explanation text is generated from a **template** keyed by reason codes and filled with numbers from the input. It is never LLM-generated ([28](28-decision-logging-spec.md) §5).

### 3.3 Order planning (entry)

- `size_usd = sizing.position_size_usd` (default 2). `size_lamports = floor(size_usd / sol_usd × 1e9)`.
- `sizing.mode = FIXED_USD` (MVP). `FIXED_FRACTION` (a % of equity) is defined but disabled until validated.
- Priority fee: `economics.priority_fee.mode = FIXED` (lamports) or `ESTIMATE` (Helius `getPriorityFeeEstimate` at `level`, cached ≤ 10 s, fallback FIXED).
- `max_slippage_bps` from `risk.max_slippage_bps`.
- Quote at decision: economics `quoteSwap(poolStateAtDecision, amountIn)` → `expected_out`, `price_impact_bps`, fee estimates. `min_out = expected_out × (1 − max_slippage_bps/10⁴)`.
- The proposed order goes to risk ([15](15-risk-engine-spec.md)).

## 4. Entry flow sequence

```text
wave ─signal.emitted─▶ strategy
  strategy: persister.flushUpTo(signal.causationWatermark)      (flush barrier)
  strategy: persist signal (wave already persisted it; idempotent upsert)
  strategy: [optional] await AI advisory ≤ timeout               (13 §6)
  strategy: DecisionEngine.decide() → Decision
  if ENTER: OrderPlanner.plan() → ProposedOrder
            risk.evaluate(ProposedOrder, portfolio, market) → RiskDecision
            persist Decision + RiskDecision (one transaction)
            if APPROVED/APPROVED_REDUCED: execution.submit(order)  → order.* events
            else: decision stays ENTER but order not created; wave → REJECTED (reason: RISK_REJECTED:<codes>)
  else:     persist Decision; wave → REJECTED
```

Decision and risk decision are persisted **before** any order exists (write-ahead). The API "trace" joins them.

## 5. ExitManager `exit@1.0.0`

### 5.1 Exit plan (resolved at entry, stored in `sim.positions.exit_plan`)

| Rule | Config | Default |
|---|---|---|
| Take profit | `exit.take_profit_pct` | +100% (on mid price vs entry exec price) |
| Scale out | `exit.scale_out` list of `{at_pct, sell_fraction}` | `[{at_pct: 100, sell_fraction: 0.5}]`. Take profit then applies to the remainder via the trailing stop. |
| Stop loss | `exit.stop_loss_pct` | −35% |
| Trailing stop | `exit.trailing.activate_at_pct`, `exit.trailing.distance_pct` | activate at +50%, trail 25% from peak |
| Max hold | `exit.max_hold_minutes` | 120 |
| Smart exit | `exit.smart_exit.min_wallets` selling within `exit.smart_exit.window_s` | 2 wallets / 300 s |
| Liquidity collapse | `exit.liquidity_collapse_pct` within `W_short` | −40% |
| Run stop | `run.close_positions_on_stop` | false |

The exit plan is frozen at entry so config changes do not alter open positions mid-flight (reproducibility).

### 5.2 Evaluation

- Triggered on `market.state.changed` for held pools and on a tick every `exit.tick_ms` (1,000 ms).
- Price for rules: **FRESH on-chain mid price**. If not FRESH, rules that depend on price are **not evaluated** (no decisions on stale prices). Time and smart-exit rules still evaluate, and the resulting exit fills will require FRESH state anyway.
- Rule priority when several fire in the same evaluation: `LIQUIDITY_EXIT` > `STOP_LOSS` > `SMART_EXIT` > `TRAILING_STOP` > `TAKE_PROFIT` / `SCALE_OUT` > `TIME_EXIT`. One exit decision per evaluation per position.
- Exit decision → `EXIT` action with intent and quantity (full remaining, or the scale-out fraction, rounded down to raw units; if the remainder after scale-out would be below `exit.min_remainder_usd` (0.30), sell the full amount).
- Exit orders pass through risk with exit semantics (FR-RSK-003).
- While an exit order is `SUBMITTED`, no new exit decision is generated for the position (except escalation from `SCALE_OUT` to a full exit, which is queued).

### 5.3 Failed exits

- `SLIPPAGE_EXCEEDED` → re-decide on the next evaluation. For `STOP_LOSS`/`LIQUIDITY_EXIT`, escalate slippage tolerance per `exit.emergency_slippage_bps` steps (e.g. 1,500 → 3,000 → 5,000) up to `exit.max_emergency_slippage_bps`. Each attempt is recorded.
- `STALE_MARKET_DATA` → retry every `execution.retry_stale_exit_ms` until FRESH. The position shows a `EXIT_BLOCKED_STALE` warning.
- `SIMULATED_TX_FAILURE` → retry immediately (a new order with a new idempotency leg), up to `exit.max_attempts` (10), then alert. Retries continue at a back-off.

## 6. Run lifecycle

```text
STARTING ─▶ RUNNING ─▶ STOPPING ─▶ STOPPED
    │           │ ledger invariant violation ─▶ HALTED_INVARIANT
    │           │ unrecoverable error        ─▶ HALTED_ERROR
    │           └ replay reached end         ─▶ COMPLETED
```

- Start: validate config → create `strategy.runs` row → fund the ledger (`RUN_FUNDING`: cash SOL = initial USD / SOL_USD at start, with that SOL/USD recorded) → `run.started`.
- Only one LIVE_FEED run may be `RUNNING` per engine (MVP). Replay runs execute in a separate engine invocation (CLI `pnpm replay`) or a worker, so they never share live state.
- Restart recovery: find the `RUNNING` run → rebuild the portfolio from the ledger → load open positions and exit plans → resolve in-flight orders ([16](16-paper-trading-spec.md) §9) → resubscribe the watch set → resume.

## 7. Entry pause reasons (automatic)

`MANUAL`, `DAILY_LOSS_LIMIT`, `DRAWDOWN_KILL`, `CONSECUTIVE_LOSS_COOLDOWN`, `WALLET_STREAM_DOWN`, `PERSISTENCE_BACKLOG`, `CLOCK_DRIFT`, `SOL_USD_STALE`, `WATCH_CAPACITY_CRITICAL`, `MODULE_CIRCUIT_OPEN`, `CREDIT_BUDGET_EXHAUSTED`. Each has a clear-condition. Automatic reasons clear themselves when resolved (except `DRAWDOWN_KILL`, which requires a manual resume, and `DAILY_LOSS_LIMIT`, which clears at the UTC day boundary). Every change emits `entries.paused`/`entries.resumed`.

## 8. Tests (minimum)

- DecisionEngine truth table: each rule in §3.2 in isolation and in priority combinations (table-driven).
- AI modes: `OFF` ignores AI. `ADVISORY` never changes the action. `VETO` can only turn ENTER into REJECT. The timeout behaviours `PROCEED`/`REJECT`/`WAIT`.
- ExitManager: each rule fires at its threshold on the simulated clock. Priority resolution. Scale-out then trailing. Stale price suppresses price rules. Emergency slippage escalation.
- End-to-end scenario (headless): wallet buys → wave → signal → ENTER → fill → price rises → TP/scale-out → trailing exit → PnL equals the hand-computed expected value.
- Restart recovery restores positions and continues exits.
