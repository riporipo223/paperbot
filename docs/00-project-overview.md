# 00 — Project Overview

| Field | Value |
|---|---|
| Status | Draft v0.1 (awaiting review/approval) |
| Owner | Project operator |
| Upstream | None (root document) |
| Downstream | Every other document |
| Used by phases | All |

## 1. What we are building

**Paperbot** is an AI-assisted research and simulation platform for trading Solana memecoins. It watches real on-chain activity from real wallets, detects when "smart money" starts buying a newly active token, decides whether a tradeable *wave* is forming, and executes that decision **in simulation only**, with realistic fees, slippage, price impact, latency and failed executions.

The first version (MVP) runs **only in SIMULATION mode**:

- Market data, wallet activity, prices, liquidity and volume are **real**.
- Money, orders, fills and positions are **virtual**.
- There are **no private keys, no seed phrases, no signing, no deposits, no withdrawals, no real orders**.

The system exists to answer one question with evidence:

> Does the smart-money wave strategy have **positive expectancy after realistic costs** at small position sizes, and under what conditions?

## 2. Why simulation is not a toy

The architecture is built as if it will one day handle real money. Simulation exists to find out, before any capital is at risk, whether the strategy survives real market conditions. It must therefore be:

- **Correct**: accounting reconciles to the lamport. No fake fills.
- **Observable**: every decision is traceable to the data that caused it.
- **Reproducible**: any run can be replayed from stored events and produce the same decisions.
- **Realistic**: costs, latency and failure are modelled rather than ignored.
- **Safe**: live execution is structurally impossible in this stage (see [21-security-spec.md](21-security-spec.md)).

## 3. Operating modes

| Mode | Market data | Money | Execution | MVP status |
|---|---|---|---|---|
| `SIMULATION` | Real | Virtual | `PaperExecutor` | **Implemented (MVP)** |
| `SHADOW` | Real | Virtual | `ShadowExecutor`: paper fill plus a read-only real quote for comparison | Preparation only (Phase 26) |
| `LIVE` | Real | Real | `RealExecutor` | **Not implemented.** A stub that always throws. Blocked in code, config, and database. |

## 4. Strategy in one paragraph

We track a curated set of wallets that have historically traded memecoins profitably. When one or more high-scoring wallets buy a token, we start watching that token's pool in real time. We measure follow-on buyers, buy/sell pressure, transaction and volume acceleration, price extension and liquidity. A deterministic scoring model decides whether a wave is forming. If it is, and every deterministic risk check passes, we simulate an entry sized by config (default $2 from a $20 bankroll). We then monitor the position and exit on take-profit, stop-loss, trailing stop, maximum hold time, smart-wallet exits, or liquidity collapse. See [12-wave-detection-spec.md](12-wave-detection-spec.md) and [14-strategy-engine-spec.md](14-strategy-engine-spec.md).

## 5. Guiding principles

1. **Event-driven and time-aware.** Every event carries several timestamps, and every latency is measured ([09-real-time-data-architecture.md](09-real-time-data-architecture.md)).
2. **One market state.** Chart, strategy, AI, executor and analytics all read the same normalized market state ([10-market-data-spec.md](10-market-data-spec.md)).
3. **Deterministic core, advisory AI.** LLMs interpret and explain. They can only *veto*. They can never size, force or execute a trade ([13-ai-agent-architecture.md](13-ai-agent-architecture.md)).
4. **Same pipeline for live and replay.** Backtests feed historical events through the exact code used live ([19-backtesting-spec.md](19-backtesting-spec.md)).
5. **Fail safe.** Missing or stale data never produces a fill. When in doubt the system does not trade.
6. **Modular monolith.** One engine process with strict internal module boundaries. No Kafka, no Kubernetes, no microservices in the MVP ([05-system-architecture.md](05-system-architecture.md)).
7. **Documentation first.** Docs are the source of truth. Implementation follows docs, and conflicts are recorded ([32-architecture-decision-records.md](32-architecture-decision-records.md)).
8. **Test-driven.** Every build task starts with a failing test (see [CONTRIBUTING.md](../CONTRIBUTING.md) and [23-testing-strategy.md](23-testing-strategy.md)).

## 6. Economic reality check (important)

At a $2 position size, **fixed costs dominate**. Rough, unverified order-of-magnitude figures that the economics engine will model precisely:

- Solana base fee is 5,000 lamports per signature, per transaction (buy and sell).
- Priority fees vary with congestion.
- Opening a new associated token account locks roughly 0.002 SOL of rent. That is refundable only when the account is closed, which is another transaction.
- DEX fees are roughly 0.25% to 1%+ depending on venue.

With SOL at $150–$250, the rent lock alone can be **15–25% of a $2 position** while the position is open. Round-trip network plus DEX costs can take several percent more. The strategy must clear these costs to have positive expectancy. The simulator exists to measure this honestly, not to hide it. See [17-economics-engine-spec.md](17-economics-engine-spec.md).

## 7. Scope summary

**In MVP scope:** real-data ingestion (Helius, DexScreener, GeckoTerminal, optional CoinGecko), event pipeline and persistence, market state, wallet intelligence and scoring, wave detection, deterministic decision and risk engines, paper executor with economics model, double-entry portfolio accounting, event replay and backtesting, advisory AI agents, operator dashboard, observability, and a deployment runbook.

**Out of MVP scope:** live execution, key management, Jupiter swap execution, multi-chain, multi-user SaaS, ML model training infrastructure, mobile apps, CLMM/DLMM pool economics (tokens whose only pools are CLMM/DLMM are rejected, not approximated).

Full detail: [01-product-spec.md](01-product-spec.md) and [02-product-requirements.md](02-product-requirements.md).

## 8. Glossary

| Term | Definition |
|---|---|
| **Smart wallet** | A tracked wallet whose current wallet score is at least `wallet.min_wallet_score` ([11](11-wallet-intelligence-spec.md)). |
| **Wave** | A per-token state machine that starts when the first smart wallet buys and ends when it is confirmed, invalidated or expired ([12](12-wave-detection-spec.md)). |
| **Signal** | An immutable record that a wave crossed the entry threshold (or an exit condition fired), together with its full feature snapshot. |
| **Decision** | The deterministic engine's outcome for a signal: `ENTER`, `WAIT`, `EXIT` or `REJECT`, with reason codes. |
| **Risk decision** | The risk engine's verdict on a proposed order: `APPROVED`, `APPROVED_REDUCED` or `REJECTED`, with per-check results. |
| **Paper order / fill** | A simulated order and its simulated execution result. |
| **Run** | One execution of a strategy version plus config version in a given mode and source (live feed or replay). All trading records belong to a run. |
| **Event** | A normalized, timestamped, immutable fact entering the pipeline (for example `wallet.swap.detected`). |
| **Market state** | The in-memory, per-pool and per-token view derived from events, with data-quality flags. |
| **Data quality** | `FRESH`, `STALE`, `DEGRADED` or `UNKNOWN`, computed per stream and subject ([09](09-real-time-data-architecture.md) §7). |
| **Lamport** | 10⁻⁹ SOL, the smallest SOL unit. |
| **Raw amount** | An integer token amount in base units, before applying decimals. |
| **bps** | Basis points. 1 bps = 0.01%. |
| **Slot** | Solana's unit of block production (roughly 400 ms). Used for ordering. |
| **ATA** | Associated Token Account. Holding a token requires one, and it costs rent. |
| **CP pool** | Constant-product AMM (x·y=k), including virtual-reserve bonding curves. |
| **Phase** | One sequential build unit, with its own document under `/build`. |

## 9. How to read this repository

1. [DOCUMENTATION-INDEX.md](../DOCUMENTATION-INDEX.md): the map of every document.
2. [ARCHITECTURE.md](../ARCHITECTURE.md): the one-page architecture summary.
3. [BUILD-MASTER-PLAN.md](../BUILD-MASTER-PLAN.md): the implementation sequence.
4. [CLAUDE.md](../CLAUDE.md): the operating contract for AI coding agents.
5. [build/PHASE-STATUS.md](../build/PHASE-STATUS.md): which phase is current.
