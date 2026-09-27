# 02 — Product Requirements (PRD)

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [00](00-project-overview.md), [01](01-product-spec.md) |
| Downstream | [03-system-requirements.md](03-system-requirements.md) and all domain specs |
| Used by phases | All. Each phase lists the FR IDs it satisfies. |

Requirement keywords follow RFC 2119: **MUST**, **SHOULD**, **MAY**. Each requirement has a stable ID. Build phases cite these IDs, and tests reference them in their names or descriptions (for example `it('FR-EXE-004: rejects fill on stale pool state')`).

## 1. Modes and safety (SAF)

| ID | Requirement |
|---|---|
| FR-SAF-001 | The system MUST support run modes `SIMULATION`, `SHADOW`, `LIVE` as a typed enum. Only `SIMULATION` is operational in the MVP. |
| FR-SAF-002 | Starting a run in `LIVE` mode MUST fail at config validation, at engine bootstrap, and at the database (CHECK constraint). |
| FR-SAF-003 | The codebase MUST NOT contain transaction signing, keypair loading, `sendTransaction`/`sendRawTransaction` calls, or private-key handling. CI MUST enforce this with static checks. |
| FR-SAF-004 | The engine MUST refuse to start if any environment variable name or value looks like a private key, seed phrase or keypair ([21](21-security-spec.md) §4). |
| FR-SAF-005 | The Solana RPC client MUST allowlist read-only methods and throw on any other method. |
| FR-SAF-006 | Every UI surface showing money MUST be labelled as simulated. |

## 2. Configuration (CFG)

| ID | Requirement |
|---|---|
| FR-CFG-001 | All strategy, risk, sizing, economics, freshness and AI parameters MUST come from a centralized, schema-validated config ([26](26-configuration-reference.md)). |
| FR-CFG-002 | Config versions MUST be immutable and content-addressed (SHA-256 of canonical JSON). |
| FR-CFG-003 | Every run MUST reference exactly one strategy version and one config version. |
| FR-CFG-004 | Initial balance (default $20) and position size (default $2) MUST be configurable. |
| FR-CFG-005 | Invalid config MUST be rejected with field-level errors. The engine MUST NOT start a run with invalid config. |

## 3. Market data ingestion (ING)

| ID | Requirement |
|---|---|
| FR-ING-001 | The system MUST ingest tracked-wallet swap activity from Solana in real time (push-based) ([10](10-market-data-spec.md)). |
| FR-ING-002 | The system MUST ingest pool state (reserves) and trades for watched pools in real time for supported venues. |
| FR-ING-003 | The system MUST poll market snapshot metrics (price, liquidity, volume, txn counts) from reference providers within their rate limits. |
| FR-ING-004 | The system MUST maintain a SOL/USD reference price with freshness tracking. |
| FR-ING-005 | Every ingested event MUST be normalized into the canonical event envelope with `event_time` (when known), `provider_time` (when provided), `received_at`, `normalized_at`, and `slot` (when applicable). |
| FR-ING-006 | Ingestion MUST deduplicate events by a deterministic source event key. |
| FR-ING-007 | Ingestion MUST automatically reconnect with exponential backoff and jitter, detect gaps, and attempt a bounded backfill. |
| FR-ING-008 | Provider rate limits and credit budgets MUST be enforced client-side. |
| FR-ING-009 | All normalized events that can influence decisions MUST be persisted to the event log for replay. |

## 4. Reference data (REF)

| ID | Requirement |
|---|---|
| FR-REF-001 | The system MUST resolve token metadata: mint, decimals, token program, mint authority, freeze authority, supply, symbol/name (untrusted). |
| FR-REF-002 | The system MUST discover and classify pools for a token (venue, pool type, base/quote, vaults) and choose a primary tradable pool. |
| FR-REF-003 | Tokens whose only liquid pools are unsupported pool types MUST be marked `UNSUPPORTED_POOL_TYPE`. |

## 5. Market state (MKT)

| ID | Requirement |
|---|---|
| FR-MKT-001 | A single market state engine MUST produce the per-pool/per-token state consumed by strategy, executor, dashboard, AI and analytics. |
| FR-MKT-002 | Market state MUST expose data-quality flags (`FRESH`, `STALE`, `DEGRADED`, `UNKNOWN`) and data age per stream. |
| FR-MKT-003 | Pool state updates MUST be applied monotonically by slot. Older-slot updates MUST be ignored and counted. |
| FR-MKT-004 | The system MUST derive OHLCV bars (1m, 5m, 1h) from internal trades/prices for charting. |

## 6. Wallet intelligence (WAL)

| ID | Requirement |
|---|---|
| FR-WAL-001 | Operator MUST be able to import, add, and change the status of wallets (`TRACKED`, `CANDIDATE`, `IGNORED`, `BLOCKED`). |
| FR-WAL-002 | The system MUST backfill wallet swap history within a configured credit budget. |
| FR-WAL-003 | The system MUST reconstruct round-trip trades (FIFO) per wallet and token. |
| FR-WAL-004 | The system MUST compute a deterministic, versioned wallet score with a stored component breakdown. |
| FR-WAL-005 | AI MAY classify wallet style. The label MUST be stored separately and MUST NOT change the deterministic score. |

## 7. Wave detection (WAV)

| ID | Requirement |
|---|---|
| FR-WAV-001 | A wave MUST be seeded when a smart wallet buy is detected for a token with a supported pool. |
| FR-WAV-002 | The wave engine MUST compute the feature set of [12](12-wave-detection-spec.md) §4 on each relevant event and timer tick. |
| FR-WAV-003 | Wave state transitions MUST follow the state machine in [12](12-wave-detection-spec.md) §5 and MUST be persisted. |
| FR-WAV-004 | An entry signal MUST include the complete feature snapshot, score breakdown, model version, data quality, and triggering event IDs. |

## 8. Decision and risk (DEC, RSK)

| ID | Requirement |
|---|---|
| FR-DEC-001 | The decision engine MUST be deterministic given its inputs and output `ENTER`, `WAIT`, `EXIT`, or `REJECT` with reason codes and a template-generated explanation. |
| FR-DEC-002 | AI output MUST only be able to downgrade `ENTER` to `REJECT` (veto) when `ai.mode = VETO`. It MUST NOT upgrade, size, or trigger trades. |
| FR-DEC-003 | Exit rules MUST include take-profit, stop-loss, trailing stop, max hold time, smart-wallet exit, and liquidity collapse, each individually configurable. |
| FR-RSK-001 | Every order (entry and exit) MUST pass the risk engine, which records every check's value and limit. |
| FR-RSK-002 | Risk MUST enforce position size, max open positions, max total exposure, max daily loss, max drawdown kill-switch, trade frequency, max slippage, max price impact, min liquidity, token safety gates, data freshness, and max entry latency. |
| FR-RSK-003 | Exit orders MUST NOT be blocked by exposure/frequency limits. They MAY be blocked only by data-quality or venue-state checks, and the block MUST be recorded. |
| FR-RSK-004 | Operator MUST be able to pause and resume new entries. |

## 9. Paper execution and economics (EXE, ECO)

| ID | Requirement |
|---|---|
| FR-EXE-001 | A `TradingExecutor` interface MUST exist with `PaperExecutor` (operational), `ShadowExecutor` (stub until Phase 26), and `RealExecutor` (stub that always throws). |
| FR-EXE-002 | Paper orders MUST follow the lifecycle in [16](16-paper-trading-spec.md) §4 with idempotency keys. |
| FR-EXE-003 | Fills MUST be computed from the pool state at *simulated execution time* (decision time + sampled latency), not decision time. |
| FR-EXE-004 | A fill MUST NOT be produced if the pool state at execution time is not `FRESH`. The order MUST fail with `STALE_MARKET_DATA`. |
| FR-EXE-005 | Orders whose execution price breaches `max_slippage_bps` versus the decision quote MUST fail with `SLIPPAGE_EXCEEDED`, charging network fees as a failed Solana transaction would. |
| FR-EXE-006 | The executor MUST simulate random execution failure per the failure model, reproducibly from the run seed. |
| FR-ECO-001 | The economics engine MUST model DEX fees, network base fee, priority fee, ATA rent lock/refund, price impact from reserves, latency, and adverse movement. |
| FR-ECO-002 | Each fill MUST store: mid price at signal, at decision, and at execution; execution price; price impact; slippage; each fee component; and net result. |

## 10. Portfolio accounting (PFL)

| ID | Requirement |
|---|---|
| FR-PFL-001 | Accounting MUST use a double-entry ledger in integer base units per asset ([18](18-portfolio-accounting-spec.md)). |
| FR-PFL-002 | The system MUST maintain cash, equity, positions, exposure, realized/unrealized PnL, ROI, fees, slippage cost, drawdown. |
| FR-PFL-003 | The system MUST compute the analytics listed in [18](18-portfolio-accounting-spec.md) §7. |
| FR-PFL-004 | Ledger invariants MUST be checked after every journal transaction. A violation MUST halt the run (fail-safe). |

## 11. AI (AI)

| ID | Requirement |
|---|---|
| FR-AI-001 | Every AI response MUST be validated against a zod schema. Invalid output MUST be discarded and recorded as `INVALID_SCHEMA`. |
| FR-AI-002 | AI calls on the decision path MUST have a timeout with deterministic fallback (`ai.on_timeout`). |
| FR-AI-003 | AI outputs MUST be stored separately from authoritative trade state, with model ID, prompt version, input hash, latency, status. |
| FR-AI-004 | AI spend MUST be bounded by a daily budget. Exceeding it disables AI calls (the fallback path applies). |

## 12. Replay and backtesting (RPL)

| ID | Requirement |
|---|---|
| FR-RPL-001 | Replay MUST feed stored events through the same pipeline code as live, with a simulated clock. |
| FR-RPL-002 | Replay MUST support `AS_RECEIVED` (default) and `BY_EVENT_TIME` ordering. |
| FR-RPL-003 | Replay of a live run with the same seed and recorded AI outputs MUST reproduce identical decisions and fills. |
| FR-RPL-004 | Operator MUST be able to compare metrics across runs. |

## 13. Dashboard (UI)

| ID | Requirement |
|---|---|
| FR-UI-001 | The dashboard MUST implement the screens in [20](20-dashboard-spec.md) §3: Portfolio, Live Market, Wallets, Positions, AI Command Center, Performance, System, Runs & Config, Decision Trace. |
| FR-UI-002 | Live views MUST update from the engine WebSocket stream, with resync on sequence gaps. |
| FR-UI-003 | Stale-data and degraded-mode states MUST be visible on every screen that shows affected data. |
| FR-UI-004 | The dashboard MUST require operator authentication. |

## 14. Observability and decision logging (OBS, LOG)

| ID | Requirement |
|---|---|
| FR-OBS-001 | Structured JSON logs MUST be emitted for all event categories in [22](22-observability-spec.md) §3, with correlation IDs. |
| FR-OBS-002 | Stage latencies MUST be measured and persisted, and exposed as percentiles. |
| FR-OBS-003 | Provider health, reconnects, gaps, rate-limit hits and credit usage MUST be recorded. |
| FR-LOG-001 | "Why did the system enter/reject TOKEN_X?" MUST be answerable from stored data via the decision-trace API ([28](28-decision-logging-spec.md)). |
| FR-LOG-002 | Runs MUST store strategy version, config version, git SHA, AI model versions, RNG seed, data sources, and replay parameters. |
