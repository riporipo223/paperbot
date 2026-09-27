# 01 — Product Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [00-project-overview.md](00-project-overview.md) |
| Downstream | [02-product-requirements.md](02-product-requirements.md), [20-dashboard-spec.md](20-dashboard-spec.md) |
| Used by phases | 19–21 (dashboard), 25 (validation) |

## 1. Product goal

Give one operator a trustworthy laboratory for evaluating a smart-money wave strategy on Solana memecoins using **real market data** and **virtual money**. The operator should be able to:

1. See what the strategy sees, as it happens.
2. Understand *why* each trade was taken or rejected.
3. Measure strategy quality after realistic costs, using expectancy, net PnL, drawdown and execution quality rather than win rate alone.
4. Replay any historical period through the same pipeline and compare strategy versions.
5. Build confidence, or disconfirm the strategy, before any real-money stage is designed.

## 2. Users

| Persona | Description | Needs |
|---|---|---|
| **Operator** (primary, only MVP user) | Technical individual running the system for research. | Start/stop runs, curate wallets, tune config versions, watch live state, investigate trades, compare runs. |
| **AI coding agent** (builder) | Claude Code or similar, implementing phases. | Unambiguous docs, contracts and acceptance criteria. |
| **Future reviewer** | Someone auditing a historical decision. | A reconstructable decision trace. |

MVP is **single-operator**. There are no multi-tenant or multi-user accounts ([ADR-0012](32-architecture-decision-records.md#adr-0012-single-operator-auth-for-mvp)).

## 3. Value proposition

- **Realism over optimism.** The system will not report a profit that real execution could not have achieved. That means no costless fills, no instant fills and no fills on stale data.
- **Explainability.** Every entry answers "Why did we enter TOKEN_X?" with stored facts.
- **Reproducibility.** Same events + same strategy version + same config + same seed = same decisions.

## 4. Core user flows

### UF-1 Start a simulation run
1. Operator selects the strategy (`smart-money-wave`) and a config version, or creates one from the editor. The config is validated against its schema.
2. Operator confirms mode `SIMULATION`. `LIVE` is not offered.
3. Engine creates a `strategy.runs` row: initial virtual balance (default $20, converted to SOL at the recorded SOL/USD price), RNG seed, git SHA.
4. Dashboard shows the run as `RUNNING` with a live portfolio.

### UF-2 Watch a wave form and a paper entry happen
1. A tracked smart wallet buys token X. The dashboard's Live Market and AI Command Center show a new **wave candidate** (`SEEDED`).
2. The engine subscribes to X's pool stream. Wave features update live.
3. A second smart wallet buys, follow-on buyers accelerate, and the score crosses the threshold. A **signal** appears.
4. The decision engine produces `ENTER`, and the risk engine approves (or rejects with reasons).
5. A paper order is created, simulated latency elapses, and a fill is computed from the pool state *at execution time*.
6. The position appears in Active Positions with entry price, fees, slippage, price impact and latency breakdown.

### UF-3 Exit
1. Price reaches the take-profit level, or another exit rule fires (stop, trailing, time, smart-wallet sells, liquidity collapse).
2. An exit decision, a sell order and a fill are produced. Realized PnL is computed net of all costs, including the rent refund when the token account is closed.
3. The trade appears in Performance with a full trace.

### UF-4 Investigate a trade ("Why did we enter?")
1. Operator opens a trade or decision.
2. The **Decision Trace** view shows, in order: triggering wallet swaps (with wallet scores), wave feature timeline, signal score breakdown, data-quality states and ages, AI advisory output (if any), each risk check (value vs limit), order, fill economics, and timestamps with latency per stage.

### UF-5 Curate wallets
1. Operator imports seed wallets (CSV/JSON) or adds one manually.
2. Engine backfills history (credit-budgeted), reconstructs round trips, and computes the wallet score.
3. Operator sees score components and sets status (`TRACKED`, `CANDIDATE`, `IGNORED`, `BLOCKED`).

### UF-6 Replay / backtest
1. Operator picks a time range for which stored events exist, a strategy version and a config version.
2. Engine runs `ReplayEventSource` with a simulated clock, producing a new run with `source = REPLAY`.
3. Operator compares run metrics side by side (net PnL, expectancy, drawdown, profit factor, trade count, latency).

### UF-7 Monitor system health
1. The System page shows provider and stream status, reconnects, gaps, rate-limit and credit usage, event latency percentiles, stale-data warnings and errors.
2. When the system is degraded, a banner shows that new entries are paused, and why.

### UF-8 Emergency controls
1. **Pause entries**: no new positions. Exits continue.
2. **Close position**: a manual paper exit through the same risk and executor path.
3. **Stop run**: stop processing and snapshot the final portfolio. Open positions are marked to market and remain `OPEN` (config `run.close_positions_on_stop` can force paper exits).

## 5. MVP scope

| Area | In scope |
|---|---|
| Data | Helius (standard WebSockets + RPC, optional webhooks), DexScreener REST, GeckoTerminal REST, CoinGecko (optional SOL/USD reference) |
| Venues | Constant-product pools: pump.fun bonding curve, PumpSwap AMM, Raydium AMM v4, Raydium CPMM. Exact support list is confirmed in Phase 03 ([27](27-data-provider-reference.md)). |
| Strategy | `smart-money-wave` v0.1.0 |
| Execution | `PaperExecutor` only |
| AI | Advisory agents (wallet classification, token context, signal explanation, post-trade analysis). Default `ai.mode = ADVISORY`. |
| Accounting | Double-entry ledger, SOL-native cash with USD reporting |
| Replay | Event replay of stored normalized events through the same pipeline |
| Dashboard | Portfolio, Live Market, Wallet Intelligence, Active Positions, AI Command Center, Performance, System, Runs/Config, Decision Trace |
| Deployment | Engine as a Docker container on a persistent host; dashboard on Vercel; PostgreSQL (local Docker for dev; Supabase or self-hosted for hosted) |

## 6. Non-goals (MVP)

- Any real-money execution, signing, key custody, deposits or withdrawals.
- Jupiter swap execution. A Jupiter *quote* (read-only) is considered only in Phase 26 for shadow comparison.
- CLMM/DLMM/orderbook economics (Orca Whirlpools, Meteora DLMM, Raydium CLMM, Phoenix/OpenBook). Tokens without a supported CP pool are **rejected**, not approximated.
- Multi-chain support.
- Multi-user accounts, billing, public SaaS.
- ML training pipelines or GPU infrastructure.
- Mobile apps.
- Social/Telegram/Twitter sentiment ingestion (future research candidate).
- Guaranteeing or targeting a specific win rate or return.

## 7. Success metrics

Product success is **not** "the strategy is profitable." It is:

| Metric | Target |
|---|---|
| Decision traceability | 100% of decisions have a complete, queryable trace (automated check). |
| Accounting integrity | 0 ledger invariant violations over the validation period. |
| Replay determinism | Replaying a stored live-feed run yields identical decisions and fills (same seed, recorded AI outputs), with byte-equal decision hashes. |
| No fake fills | 0 fills created on `STALE`/`UNKNOWN` pool state (enforced and tested). |
| Latency visibility | p50/p95/p99 available for every pipeline stage. |
| Uptime of paper pipeline | Engine continuous run ≥ 7 days during validation, with automatic recovery from provider disconnects. |
| Strategy verdict | After Phase 25, a written evidence-based report: expectancy after costs, drawdown, cost breakdown, sensitivity to latency/slippage assumptions. |

## 8. Product risks

| Risk | Mitigation |
|---|---|
| Free-tier data limits make real-time coverage partial | Tiered data design ([10](10-market-data-spec.md)). Credit budget manager. Explicit freshness flags. Paid upgrade path documented ([27](27-data-provider-reference.md)). |
| Cost model optimism | Conservative defaults, sensitivity analysis in validation, shadow comparison later. |
| Survivorship bias in wallet selection | Scores computed out-of-sample. Wallet history cut-off before the evaluation period ([11](11-wallet-intelligence-spec.md) §8). |
| Memecoin rug/honeypot risk | Deterministic token safety gates (mint/freeze authority, liquidity, pool type) ([15](15-risk-engine-spec.md)). |
| Prompt injection through token metadata | AI is advisory, schema-bound and veto-only. Token strings are treated as untrusted data ([21](21-security-spec.md) §7). |
| Operator over-trusts simulation | Dashboard labels every figure as simulated. Validation report lists model assumptions. |
