# 03 — System Requirements (Non-Functional)

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [01](01-product-spec.md), [02](02-product-requirements.md) |
| Downstream | [04](04-technical-spec.md), [05](05-system-architecture.md), [09](09-real-time-data-architecture.md), [22](22-observability-spec.md), [23](23-testing-strategy.md) |
| Used by phases | 03, 04, 07, 08, 22, 23, 25 |

## 1. Definition of "real-time" (per subsystem)

"Real-time" is never used without a class. Every stream and subsystem is assigned one of these latency classes. The numbers are **provisional ceilings for design**. They are **not** measured guarantees. Phase 25 replaces them with measured baselines, recorded in [22](22-observability-spec.md) §8.

| Class | Meaning | Delivery model | Provisional design ceiling (p95) |
|---|---|---|---|
| **RT-STREAM** | Push from provider, processed on arrival | WebSocket / webhook | Detection ≤ 3 s after the event's slot is confirmed (to be measured) |
| **RT-INTERNAL** | In-process computation on the hot path | Function calls on the event bus | Per-stage budgets in §2 |
| **NRT-POLL** | Near-real-time polling | REST polling within rate limits | Data age ≤ 30 s |
| **REFERENCE** | Slow reference/backfill data | REST, batch | Minutes to hours. Never used for fills. |
| **UI-LIVE** | Dashboard update after engine state change | WebSocket to browser | ≤ 1 s from engine state change to render |

Subsystem assignments:

| Subsystem | Class | Notes |
|---|---|---|
| Tracked-wallet swap detection | RT-STREAM | Helius standard WS `logsSubscribe` (or webhook alternative, [ADR-0009](32-architecture-decision-records.md#adr-0009-wallet-activity-source)). |
| Watched pool trades and reserves | RT-STREAM | `logsSubscribe` + `accountSubscribe` on pool accounts. |
| Market snapshots (volume, txns, liquidity) | NRT-POLL | DexScreener / GeckoTerminal. |
| SOL/USD reference | NRT-POLL | 30–60 s cadence. |
| OHLCV history, token discovery lists, wallet history backfill | REFERENCE | |
| Normalization → market state → wave → decision → risk → order | RT-INTERNAL | |
| Paper fill | Simulated latency (model), executed on the engine clock | Not a performance target. It is a modelled delay. |

## 2. Internal latency budgets (provisional)

Measured per event with high-resolution monotonic timing. These are ceilings used to detect regressions until Phase 25 establishes baselines.

| Stage | From → To | Provisional p95 ceiling |
|---|---|---|
| L1 ingest | socket message received → envelope normalized | ≤ 2 ms (excluding any RPC fetch) |
| L1b enrichment fetch | normalized → enriched (e.g. `getTransaction` when needed) | Measured only. Network-bound, reported separately. |
| L2 state | normalized → market state applied | ≤ 5 ms |
| L3 signal | state applied → wave features/score updated (signal emitted if any) | ≤ 10 ms |
| L4 decision | signal → decision (AI off/async) | ≤ 5 ms |
| L5 risk | decision → risk verdict | ≤ 5 ms |
| L6 order | risk approved → paper order persisted (`SUBMITTED`) | ≤ 100 ms (includes a DB round trip) |
| L7 fill | submitted → fill computed at simulated execution time | modelled latency + ≤ 10 ms compute |
| L8 portfolio | fill → ledger + position + snapshot persisted | ≤ 100 ms |

End-to-end *system-attributable* latency (receipt of trigger event → order `SUBMITTED`) is the sum of L1–L6 plus enrichment. It is stored per decision as `system_latency_us`.

## 3. Time and clock requirements

| ID | Requirement |
|---|---|
| NFR-TIME-001 | All timestamps MUST be generated server-side from an injected `Clock`. Direct `Date.now()` / `new Date()` in domain code is forbidden (lint rule). |
| NFR-TIME-002 | Wall-clock timestamps MUST have microsecond resolution (`EpochMicros`). Durations MUST use a monotonic source. |
| NFR-TIME-003 | The host MUST run NTP (chrony or equivalent). The engine MUST record measured clock offset at startup and every 10 minutes, and flag `CLOCK_DRIFT` above `clock.max_offset_ms` (default 250 ms). |
| NFR-TIME-004 | Replay MUST use a `SimulatedClock` driven by event timestamps. No wall-clock reads in the pipeline during replay. |

## 4. Throughput and capacity (MVP sizing)

Estimates for design only, to be validated in Phase 23:

| Dimension | MVP design target | Headroom requirement |
|---|---|---|
| Tracked wallets | 20–200 | Design must not assume < 1,000 |
| Concurrently watched pools | 5–30 | Bounded by provider connection/credit limits |
| Ingested events | Up to 200 events/s burst, 20 events/s sustained | Pipeline tested at 10× sustained (200/s) with recorded streams |
| Concurrent open positions | ≤ `sizing.max_open_positions` (default 3) | Engine tested with 50 |
| Event log growth | ~0.5–5 GB/month depending on watch set | Partitioned. Retention configurable. |

## 5. Reliability

| ID | Requirement |
|---|---|
| NFR-REL-001 | Provider disconnects MUST be recovered automatically without restarting the engine. |
| NFR-REL-002 | An engine restart MUST restore portfolio state from the ledger. Open positions resume monitoring. In-flight paper orders are resolved per [16](16-paper-trading-spec.md) §9. |
| NFR-REL-003 | A database outage MUST pause new entries. Exits whose persistence fails MUST be retried. No decision is acted on without its write-ahead record. |
| NFR-REL-004 | AI outage MUST NOT affect accounting or deterministic decisions beyond the configured fallback. |
| NFR-REL-005 | No single failure may produce a fill on stale/unknown data or a ledger imbalance. |

## 6. Data integrity

| ID | Requirement |
|---|---|
| NFR-DATA-001 | Money MUST be represented as integer base units (`bigint`) or arbitrary-precision decimals. Never IEEE floats ([ADR-0014](32-architecture-decision-records.md#adr-0014-integer-base-units-and-decimal-math)). |
| NFR-DATA-002 | Authoritative trading records (orders, fills, ledger, decisions, risk decisions, signals) MUST be written transactionally and never updated destructively (append or status transitions with history). |
| NFR-DATA-003 | Event log records MUST be immutable. |
| NFR-DATA-004 | All external payloads MUST be schema-validated (zod) at the boundary. Invalid payloads are rejected and counted, never partially applied. |

## 7. Security

See [21-security-spec.md](21-security-spec.md). Key NFRs:

- NFR-SEC-001: No private keys or seed phrases anywhere, in any environment, during the MVP.
- NFR-SEC-002: Secrets (API keys) only in environment variables / platform secret stores, never in DB, logs, AI prompts, or frontend bundles.
- NFR-SEC-003: Engine API authenticated, with CORS restricted to the dashboard origin.
- NFR-SEC-004: Untrusted strings (token names, symbols, URIs) MUST be escaped in the UI and fenced as data in AI prompts.

## 8. Maintainability

- Modular monolith with enforced module boundaries (dependency-cruiser rules, [04](04-technical-spec.md) §6).
- Strict TypeScript (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`).
- Coverage floors: domain modules ≥ 90% lines; overall ≥ 80% ([23](23-testing-strategy.md)).
- All public module interfaces documented with TSDoc.

## 9. Portability and cost

- MVP MUST run on free tiers where possible: Helius Free, DexScreener (keyless), GeckoTerminal public API, CoinGecko Demo (optional), Vercel Hobby, local Postgres.
- Hosted engine requires a small always-on host (not free; ~$5–10/month class VPS). See [25](25-infrastructure-deployment.md).
- Upgrading any provider tier MUST be a config change plus limit table update, not a code rewrite.

## 10. Compliance and terms

- Provider terms of use MUST be recorded in [27](27-data-provider-reference.md) and respected (attribution, caching, redistribution limits).
- The dashboard is private (single operator). No public redistribution of provider data.
