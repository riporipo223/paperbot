# Paperbot

A research and **simulation** platform for AI-assisted trading of Solana memecoins. It uses real on-chain market data, real wallet activity and realistic trading economics, with **virtual money only**.

> **SIMULATION ONLY.** No private keys, no seed phrases, no wallet signing, no deposits, no withdrawals, no real orders. Live execution is structurally impossible in this project stage. See [docs/21-security-spec.md](docs/21-security-spec.md).

## Status

**Documentation stage complete. Awaiting review and approval.** No application code has been written yet. Implementation follows [BUILD-MASTER-PLAN.md](BUILD-MASTER-PLAN.md), phase by phase, only after approval. Progress: [build/PHASE-STATUS.md](build/PHASE-STATUS.md).

## What it does

1. Tracks wallets with historically strong memecoin trading performance, scored deterministically.
2. Detects their buys on Solana in real time.
3. Watches the token's pool on-chain: follow-on buyers, buy/sell pressure, acceleration, liquidity, price extension.
4. Decides deterministically whether a "wave" is forming, applies strict risk rules, and simulates an entry ($2 of a $20 virtual bankroll by default; both configurable).
5. Simulates execution realistically: latency, price impact from real reserves, DEX and network fees, priority fees, token-account rent, MEV, and failed transactions.
6. Manages exits (take profit, scale-out, trailing stop, stop loss, time, smart-wallet exits, liquidity collapse).
7. Accounts for everything in a double-entry ledger, and explains every decision ("Why did we enter TOKEN_X?").
8. Replays history through the exact same pipeline for backtesting and parameter studies.

AI (LLMs) is **advisory**. It can explain, classify, and optionally veto an entry, but it can never size, force or execute a trade.

## Architecture in one picture

```text
Solana (via Helius WS/RPC) · DexScreener · GeckoTerminal · CoinGecko
                 │ read-only
                 ▼
┌────────────── Engine (Node.js/TypeScript modular monolith, Docker) ──────────────┐
│ ingestion → pipeline (dedup, sequence, event log) → market state (quality-flagged) │
│ wallet intelligence → wave detection → decision engine → risk → PaperExecutor     │
│ → double-entry portfolio ledger ; AI advisory ; replay ; REST+WS API ; metrics     │
└───────────────┬────────────────────────────────────────────┬──────────────────────┘
                ▼                                            ▼
          PostgreSQL                                Dashboard (Next.js on Vercel)
```

Details: [ARCHITECTURE.md](ARCHITECTURE.md).

## Where to start

| You are… | Read |
|---|---|
| The operator/reviewer | [docs/00-project-overview.md](docs/00-project-overview.md) → [docs/01-product-spec.md](docs/01-product-spec.md) → [docs/32-architecture-decision-records.md](docs/32-architecture-decision-records.md) (open decisions) |
| An AI coding agent | [CLAUDE.md](CLAUDE.md) → [DOCUMENTATION-INDEX.md](DOCUMENTATION-INDEX.md) → [BUILD-MASTER-PLAN.md](BUILD-MASTER-PLAN.md) |
| A developer | [DEVELOPMENT.md](DEVELOPMENT.md), [CONTRIBUTING.md](CONTRIBUTING.md) |

## Repository layout (target)

```text
apps/engine        # the real-time engine (modular monolith)
apps/dashboard     # Next.js operator dashboard
packages/core      # pure domain primitives (time, money, events, enums)
packages/config    # configuration schema + loader
packages/db        # migrations, DB types, partition manager
packages/api-contract  # shared REST/WS schemas
config/            # engine + versioned strategy configs
fixtures/          # recorded provider data and scenario streams
docs/              # specifications (source of truth)
build/             # phase-by-phase implementation documents
```

## License

To be decided by the operator.
