# 26 — Configuration Reference

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | All domain specs (they define the semantics). This file is the single registry of keys and defaults. |
| Downstream | `packages/config` (zod schema), `config/*.yaml`, dashboard config editor |
| Used by phases | 01 (schema skeleton), each domain phase adds its section, 15 (strategy file complete) |

## 1. Structure

Two kinds of configuration:

| Kind | File(s) | Hashed into config version? | Changes require |
|---|---|---|---|
| **Strategy config** (anything that can change a trading decision or simulated outcome) | `config/strategies/smart-money-wave/<semver>.yaml` → stored in `strategy.config_versions` | **Yes** (`config_hash`) | A new config version (immutable) |
| **Engine config** (operational: providers, limits, pipeline, retention, API) | `config/base.yaml` + `config/env/<env>.yaml` | No, but its SHA-256 and the relevant parts are snapshotted into `strategy.runs.data_sources` | Engine restart |
| **Secrets** | Environment variables only ([24](24-development-environment.md) §4) | Never | Restart |

Load order: `base.yaml` → `env/<PAPERBOT_ENV>.yaml` → strategy file (or DB config version) → env var secrets. Validate with zod, deep-freeze, compute the hash (`sha256` of canonical JSON: sorted keys, no whitespace, numbers normalized).

Legacy/flat names from the product brief map to canonical keys:

| Brief name | Canonical key |
|---|---|
| `INITIAL_BALANCE` | `portfolio.initial_balance_usd` |
| `POSITION_SIZE` | `sizing.position_size_usd` |
| `MAX_OPEN_POSITIONS` | `sizing.max_open_positions` |
| `MAX_TOTAL_EXPOSURE` | `sizing.max_total_exposure_usd` |
| `MAX_DAILY_LOSS` | `risk.max_daily_loss_usd` |
| `MAX_SLIPPAGE` | `risk.max_slippage_bps` |
| `MIN_LIQUIDITY` | `risk.min_liquidity_usd` |
| `MIN_WALLET_SCORE` | `wallet.min_wallet_score` |
| `MAX_TOKEN_AGE` | `risk.max_token_age_s` |
| `MAX_ENTRY_LATENCY` | `risk.max_entry_latency_s` |
| `TAKE_PROFIT` | `exit.take_profit_pct` |
| `STOP_LOSS` | `exit.stop_loss_pct` |
| `MAX_HOLD_TIME` | `exit.max_hold_minutes` |

## 2. Strategy config: `smart-money-wave` 0.1.0 (defaults)

All defaults are **initial hypotheses**, not optimized values. Keys marked `UNCALIBRATED` model real-world behaviour that hasn't been measured.

```yaml
strategy:
  id: smart-money-wave
  version: 0.1.0
  max_signal_age_ms: 2000
  max_price_drift_since_signal_pct: 10

run:
  mode: SIMULATION                 # SIMULATION | SHADOW (Phase 26+) ; LIVE rejected
  cash_asset: SOL                  # SOL | USDC (USDC disabled in MVP)
  close_positions_on_stop: false

portfolio:
  initial_balance_usd: 20
  mark_method: LIQUIDATION         # LIQUIDATION | MID
  snapshot_interval_s: 60

sizing:
  mode: FIXED_USD                  # FIXED_USD | FIXED_FRACTION (disabled)
  position_size_usd: 2
  max_position_size_usd: 2
  min_order_usd: 0.5
  max_open_positions: 3
  max_total_exposure_usd: 6

risk:
  max_daily_loss_usd: 4
  max_drawdown_pct: 40
  max_consecutive_losses: 5
  consecutive_loss_cooldown_minutes: 60
  max_entries_per_hour: 6
  max_slippage_bps: 1000
  max_price_impact_bps: 300
  max_cost_ratio: 0.25
  min_liquidity_usd: 10000
  max_token_age_s: 43200
  max_entry_latency_s: 90
  max_trigger_age_ms: 5000
  cash_buffer_lamports: 10000000   # 0.01 SOL
  require_mint_authority_revoked: true
  require_freeze_authority_revoked: true
  forbidden_token2022_extensions: [transferFeeConfig, permanentDelegate, nonTransferable, transferHook, defaultAccountStateFrozen]
  max_top10_holder_pct: 50
  require_holder_data: false

wallet:
  min_wallet_score: 0.65
  min_sample_trades: 10
  min_buy_value_sol: 0.1
  auto_promote: false
  promote_score: 0.70
  promote_confidence: 0.60
  demote_score: 0.50
  score_resolution: AS_OF_TIME     # AS_OF_TIME | FROZEN_AT_START (replay)
  scoring:
    model_version: wallet-score@1.0.0
    lookback_days: 30
    shrinkage_k: 10
    prior: 0.30
    win_threshold_roi: 0.10
    weights: { C1: 0.25, C2: 0.20, C3: 0.15, C4: 0.15, C5: 0.15, C6: 0.10 }
    c1_logistic: { mid: 0.10, k: 8 }
    c2_logistic: { mid: 0.05, k: 10 }
    c3_linear: { zero_at: 0.25, one_at: 0.65 }
    c4_pf_cap: 4
    c5_cap_multiple: 3
    c5_min_observations: 5
    penalties:
      bot_like: { max_median_hold_s: 20, min_swaps_per_day: 100, multiplier: 0.5 }
      sniper_only: { max_slots_after_creation: 2, min_share: 0.8, multiplier: 0.6 }
      insider_suspect: { multiplier: 0.3 }
      wash_suspect: { min_round_trips_per_hour_same_token: 5, multiplier: 0.5 }
      stale: { inactive_days: 7, multiplier: 0.8 }
  backfill:
    lookback_days: 30
    max_signatures: 2000
    max_credits_per_job: 5000
  mining:
    enabled: false
    min_peak_multiple: 3
    window_minutes: 60

wave:
  model_version: wave-score@1.0.0
  tick_ms: 1000
  cluster_window_s: 600
  short_window_s: 60
  base_window_s: 300
  expire_s: 900
  min_observe_s: 30
  pool_resolution_timeout_s: 10
  min_smart_wallets: 2
  min_follow_buyers: 8
  entry_score_threshold: 0.70
  invalidate_threshold: 0.25
  max_price_extension_pct: 60
  max_drawdown_from_seed_pct: 35
  min_buy_ratio: 0.55
  liquidity_collapse_pct: 40
  invalidate_on_smart_sells: 2
  smart_lambda: 1.5
  follow_target: 25
  accel_target: 3.0
  momentum_sweet_spot_pct: 20
  weights: { S_smart: 0.30, S_follow: 0.20, S_pressure: 0.15, S_accel: 0.15, S_momentum: 0.10, S_liquidity: 0.10 }
  reseed_cooldown_s: 1800
  retry_after_failed_entry: false
  require_snapshot_features: false

exit:
  model_version: exit@1.0.0
  tick_ms: 1000
  take_profit_pct: 100
  scale_out: [ { at_pct: 100, sell_fraction: 0.5 } ]
  stop_loss_pct: 35
  trailing: { activate_at_pct: 50, distance_pct: 25 }
  max_hold_minutes: 120
  smart_exit: { min_wallets: 2, window_s: 300 }
  liquidity_collapse_pct: 40
  min_remainder_usd: 0.30
  emergency_slippage_bps: [1500, 3000, 5000]
  max_emergency_slippage_bps: 5000
  max_attempts: 10

economics:
  model_version: economics@1.0.0
  network:
    base_fee_lamports_per_signature: 5000       # verify (Phase 03)
    signatures_per_tx: 1
  priority_fee:
    mode: FIXED                                 # FIXED | ESTIMATE
    compute_unit_limit: 200000
    compute_unit_price_micro_lamports: 50000    # → 10,000 lamports; UNCALIBRATED
    estimate_level: Medium                      # used when mode=ESTIMATE (Helius levels; verify)
  tip_lamports: 0
  close_ata_on_full_exit: true
  latency: { kind: LOGNORMAL, median_ms: 1200, p95_ms: 4000, min_ms: 400, max_ms: 20000, replay_overhead_ms: 50, calibration: UNCALIBRATED }
  failure: { base_prob: 0.03, calibration: UNCALIBRATED }
  mev: { enabled: true, sandwich_prob: 0.10, sell_prob: 0.05, extraction_fraction_of_tolerance: 0.5, fail_instead_prob: 0.2, calibration: UNCALIBRATED }

execution:
  retry_stale_exit_ms: 2000
  recovery_grace_ms: 5000

freshness:                                      # max ages (ms) per consumer (09 §7)
  pool_state: { wave_ms: 5000, entry_ms: 3000, fill_ms: 3000, exit_ms: 5000, mark_ms: 30000 }
  snapshot:   { wave_ms: 45000, entry_ms: 45000, exit_ms: 45000, mark_ms: 120000 }
  sol_usd:    { default_ms: 120000, mark_ms: 300000 }
  wallet_stream: { heartbeat_ms: 30000 }
  trades: { idle_confirm_ms: 60000 }

ai:
  mode: ADVISORY                   # OFF | ADVISORY | VETO
  on_timeout: PROCEED              # PROCEED | REJECT | WAIT (VETO mode only)
  entry_review_timeout_ms: 1500
  veto_min_confidence: 0.7
  daily_budget_usd: 1.0
  models:
    ENTRY_REVIEWER:     { provider: anthropic, model: claude-haiku-4-5-20251001, max_tokens: 400, temperature: 0 }
    TOKEN_CONTEXT:      { provider: anthropic, model: claude-sonnet-5, max_tokens: 400, temperature: 0 }
    WALLET_CLASSIFIER:  { provider: anthropic, model: claude-sonnet-5, max_tokens: 500, temperature: 0 }
    SIGNAL_EXPLAINER:   { provider: anthropic, model: claude-sonnet-5, max_tokens: 600, temperature: 0 }
    POST_TRADE_ANALYST: { provider: anthropic, model: claude-sonnet-5, max_tokens: 800, temperature: 0 }
    RESEARCH:           { provider: anthropic, model: claude-sonnet-5, max_tokens: 2000, temperature: 0 }
  agents:
    ENTRY_REVIEWER:     { enabled: true, max_calls_per_hour: 30 }
    TOKEN_CONTEXT:      { enabled: true, max_calls_per_hour: 60 }
    WALLET_CLASSIFIER:  { enabled: true, max_calls_per_hour: 30, refresh_days: 7 }
    SIGNAL_EXPLAINER:   { enabled: false, max_calls_per_hour: 30 }
    POST_TRADE_ANALYST: { enabled: true, max_calls_per_hour: 20 }
    RESEARCH:           { enabled: true, max_calls_per_hour: 5 }
  pricing: {}                      # model → {input_per_mtok_usd, output_per_mtok_usd}; filled & verified in Phase 16
```

## 3. Engine config (`config/base.yaml`, defaults)

```yaml
ingestion:
  wallet_source: helius_ws           # helius_ws | helius_webhook  (ADR-0009)
  safety_factor: 0.8                 # use 80% of documented rate limits
  helius:
    network: mainnet
    commitment: confirmed
    max_ws_connections: 5            # Free plan (verify)
    max_subscriptions_per_connection: 50   # UNKNOWN → set from Phase 03 verification
    rpc_rate_limit_per_s: 10         # Free plan (verify)
    monthly_credit_budget: 1000000   # Free plan (verify)
    credit_costs: { rpc: 1, gpa: 10, das: 10, webhook_event: 1, stream_per_mb: 20 }  # verify
  ws: { ping_ms: 15000, backoff_base_ms: 500, backoff_cap_ms: 30000, alert_after_attempts: 5 }
  dexscreener: { poll_ms: 10000, pairs_rate_limit_per_min: 300, profiles_rate_limit_per_min: 60 }
  geckoterminal: { rate_limit_per_min: 30, allocation: { trades: 0.5, ohlcv: 0.3, discovery: 0.2 } }
  coingecko: { enabled: false, poll_ms: 300000 }
  sol_usd: { poll_ms: 30000 }
watch:
  max_watched_pools: 20
  max_tracked_wallets: 100
  linger_seconds: 120
  capture_mode: STRATEGY             # STRATEGY | BROAD (19 §2)
pipeline:
  dedup: { ttl_seconds: 86400, max_keys: 2000000 }
  persist: { flush_ms: 250, batch_size: 500, max_buffer: 100000 }
  lateness_slots: 12
  queues: { adapter_inbound: 10000, enrichment: 1000, bus_per_subject: 5000 }
market_state:
  trade_window_minutes: 60
  pool_state_persist_interval_ms: 1000
  sol_usd_max_divergence_bps: 100
reference:
  safety_refresh_minutes: 10
  safety_max_age_minutes: 15
clock:
  max_offset_ms: 250
  halt_offset_ms: 2000
  offset_check_minutes: 10
  ntp_server: pool.ntp.org
  slot_poll_ms: 2000
retention:
  events_days: 30
  trades_days: 30
  snapshots_days: 30
  pool_states_days: 30
  bars_1m_days: 180
  bars_5m_days: 730
  bars_1h_days: 730
  ops_days: 14
  archive: { enabled: false, target: "" }   # e.g. s3://bucket/prefix (credentials via env)
api:
  port: 8080
  rate_limit_per_s: 60
  ws: { max_queue: 1000, coalesce_hz: 4, ping_ms: 15000 }
observability:
  debug_sample_rate: 0.01
  alert_webhook_url: ""
  latency_budgets: {}                # filled after Phase 25 baselines
security:
  allowed_hosts: [ "mainnet.helius-rpc.com", "api.helius.xyz", "api.dexscreener.com", "api.geckoterminal.com", "api.coingecko.com", "api.anthropic.com" ]   # verify exact hosts in Phase 03
features:
  shadow_enabled: false
replay:
  page_size: 5000
  ai_cache_scope: SOURCE_RUN
  ai_live_on_miss: false
```

## 4. Cross-field validation rules

| Rule | Error code |
|---|---|
| `run.mode ∈ {SIMULATION}` (or `SHADOW` only if `features.shadow_enabled`) | `SAF_MODE_NOT_ALLOWED` |
| `sizing.position_size_usd ≤ sizing.max_position_size_usd ≤ sizing.max_total_exposure_usd` | `CFG_SIZING_INCONSISTENT` |
| `sizing.min_order_usd ≤ sizing.position_size_usd` | `CFG_SIZING_INCONSISTENT` |
| `risk.max_daily_loss_usd < portfolio.initial_balance_usd` | `CFG_RISK_INCONSISTENT` |
| `wave.weights` and `wallet.scoring.weights` each sum to 1 (±1e-9) | `CFG_WEIGHTS_SUM` |
| `wave.invalidate_threshold < wave.entry_score_threshold` | `CFG_WAVE_THRESHOLDS` |
| `wave.momentum_sweet_spot_pct < wave.max_price_extension_pct` | `CFG_WAVE_MOMENTUM` |
| `exit.stop_loss_pct > 0`, `exit.take_profit_pct > 0`, scale-out fractions in (0,1), and `at_pct` ascending | `CFG_EXIT_INVALID` |
| `risk.max_slippage_bps ≤ exit.max_emergency_slippage_bps ≤ 5000` | `CFG_SLIPPAGE` |
| `economics.latency.min_ms ≤ median_ms ≤ p95_ms ≤ max_ms` | `CFG_LATENCY` |
| `freshness.pool_state.fill_ms ≤ freshness.pool_state.exit_ms` | `CFG_FRESHNESS` |
| `ai.mode = VETO` requires `ai.agents.ENTRY_REVIEWER.enabled` | `CFG_AI` |

## 5. Change policy

- Strategy config files are **immutable once merged to `main`**. Create `0.1.1.yaml` (or a DB-only version via the API) instead of editing. A CI check (`scripts/check-config-immutability.ts`) fails if a PR modifies or deletes an existing file under `config/strategies/` (additions only).
- Every new key must be added to this document, the zod schema, and a default in the YAML **in the same commit**.
