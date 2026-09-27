# 31 — Build Phases (Overview)

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [30-development-plan.md](30-development-plan.md) |
| Downstream | `/build/phase-XX-*.md` (one per phase), [BUILD-MASTER-PLAN.md](../BUILD-MASTER-PLAN.md), [build/PHASE-STATUS.md](../build/PHASE-STATUS.md) |

The phases below come from the architecture: module dependency rules ([04](04-technical-spec.md) §6), migration FK order ([07](07-database-schema.md) §8), and the data flow ([05](05-system-architecture.md) §5). They are the refined version of the initial hypothesis in the product brief. Differences from that hypothesis:

- **Added:** Phase 03, *Provider verification & client foundation*. No provider-specific code is written before facts are verified.
- **Split:** "Market Data Infrastructure" became Pool Streams & Decoders (07) + Market State Engine (08). "Dashboard" became Foundation (19) + two screen phases (20, 21). An explicit Engine API phase (18) was added.
- **Reordered:** Economics (11) → Portfolio (12) → Risk (13) → Paper Executor (14) → Decision Engine (15). The executor needs the ledger and decision/risk records, and the decision engine needs all of them.
- **AI after the trading core** (16), with a no-op port introduced in 15. Replay (17) comes after AI so the AI replay cache is covered.
- **Integration testing** (23) consolidates the chaos/perf suites. Each earlier phase still ships its own tests (TDD).

## Phase table

| # | Phase | Depends on | Milestone | Main FRs | Size |
|---|---|---|---|---|---|
| 00 | [Repository & development environment](../build/phase-00-repository-environment.md) | — | M0 | FR-SAF-003 (scan skeleton) | S |
| 01 | [Core foundation (core + config packages)](../build/phase-01-core-foundation.md) | 00 | M0 | FR-CFG-001..005, FR-SAF-001/004 | M |
| 02 | [Database foundation](../build/phase-02-database-foundation.md) | 01 | M0 | FR-SAF-002 (DB), NFR-DATA-* | M |
| 03 | [Provider verification & client foundation](../build/phase-03-provider-verification.md) | 01 | M1 | FR-ING-008, FR-SAF-005 | M |
| 04 | [Event pipeline core](../build/phase-04-event-pipeline.md) | 02, 03 | M1 | FR-ING-005/006/009, FR-OBS-002 | L |
| 05 | [Wallet activity ingestion (Helius)](../build/phase-05-wallet-ingestion.md) | 04 | M1 | FR-ING-001/007, FR-WAL-001 (registry) | L |
| 06 | [Reference data: venues, tokens, pools](../build/phase-06-reference-data.md) | 04 | M1 | FR-REF-001..003 | M |
| 07 | [Pool streams & venue decoders](../build/phase-07-pool-streams-decoders.md) | 05, 06 | M1 | FR-ING-002 | L |
| 08 | [Market state engine](../build/phase-08-market-state.md) | 07 | M1 | FR-MKT-001..004, FR-ING-003/004 | L |
| 09 | [Wallet intelligence](../build/phase-09-wallet-intelligence.md) | 05, 08 | M2 | FR-WAL-001..004 | L |
| 10 | [Wave detection](../build/phase-10-wave-detection.md) | 08, 09 | M2 | FR-WAV-001..004 | L |
| 11 | [Economics engine](../build/phase-11-economics-engine.md) | 07 (fixtures), 01 | M3 | FR-ECO-001 | M |
| 12 | [Portfolio accounting](../build/phase-12-portfolio-accounting.md) | 02, 11 | M3 | FR-PFL-001..004 | L |
| 13 | [Risk engine](../build/phase-13-risk-engine.md) | 08, 12 | M3 | FR-RSK-001..004 | M |
| 14 | [Paper trading engine](../build/phase-14-paper-trading-engine.md) | 11, 12, 13 | M3 | FR-EXE-001..006, FR-ECO-002 | L |
| 15 | [Decision engine & strategy runtime](../build/phase-15-decision-engine.md) | 10, 14 | M3 | FR-DEC-001..003, FR-LOG-001/002 | L |
| 16 | [AI agent layer](../build/phase-16-ai-agents.md) | 15 | M4 | FR-AI-001..004, FR-DEC-002, FR-WAL-005 | M |
| 17 | [Replay & backtesting](../build/phase-17-replay-backtesting.md) | 15, 16 | M4 | FR-RPL-001..004 | L |
| 18 | [Engine API (REST + WebSocket)](../build/phase-18-engine-api.md) | 15, 17 | M5 | FR-UI-002 (server), FR-LOG-001 | L |
| 19 | [Dashboard foundation](../build/phase-19-dashboard-foundation.md) | 18 | M5 | FR-UI-004, FR-SAF-006 | M |
| 20 | [Dashboard: portfolio, positions, market, wallets](../build/phase-20-dashboard-core-screens.md) | 19 | M5 | FR-UI-001..003 | L |
| 21 | [Dashboard: AI, performance, system, runs, trace](../build/phase-21-dashboard-intel-screens.md) | 20 | M5 | FR-UI-001, FR-LOG-001 | L |
| 22 | [Observability hardening](../build/phase-22-observability.md) | 18 | M6 | FR-OBS-001..003 | M |
| 23 | [Integration, failure & performance testing](../build/phase-23-integration-testing.md) | 21, 22 | M6 | NFR-REL-*, §5 of [23](23-testing-strategy.md) | L |
| 24 | [Deployment](../build/phase-24-deployment.md) | 23 | M6 | NFR-SEC-*, [25](25-infrastructure-deployment.md) | M |
| 25 | [Paper trading validation](../build/phase-25-paper-validation.md) | 24 | M7 | Success metrics [01](01-product-spec.md) §7 | L (calendar time) |
| 26 | [Shadow trading preparation](../build/phase-26-shadow-preparation.md) | 25 | M8 | FR-EXE-001 (Shadow) | M |

Size: S ≈ < 1 day of agent work; M ≈ 1–3 days; L ≈ 3–6 days.

## Dependency graph

```text
00 → 01 → 02 ─┐
      └→ 03 ──┴→ 04 → 05 ─┬→ 07 → 08 ─┬→ 09 → 10 ─┐
                  └→ 06 ──┘     │      │           │
                                └→ 11 ─┴→ 12 → 13 → 14 → 15 → 16 → 17 → 18 → 19 → 20 → 21 ─┐
                                                                       └→ 22 ────────────────┴→ 23 → 24 → 25 → 26
```

Phases on parallel branches (e.g. 06 vs 05, 11 vs 09–10, 22 vs 19–21) may run in any order once their dependencies are `DONE`. The canonical sequential order is by number.

## Uniform phase document structure

Every `/build/phase-XX-*.md` contains: Objective · Context · Dependencies · Inputs · Outputs · Files To Create · Files To Modify · Database Changes · API Changes · Environment Variables · Implementation Tasks (test-first) · Acceptance Criteria · Tests · Failure Cases · Observability · Security · Verification · Commit Strategy · Definition Of Done.

A global Definition of Done applies to every phase in addition to the phase-specific list:

- [ ] `pnpm verify` passes locally (and CI is green on the pushed branch).
- [ ] New code has tests written first. Coverage floors are met ([23](23-testing-strategy.md) §6).
- [ ] No `Date.now()`/`Math.random()`/float money/`process.env` outside the allowed places.
- [ ] `pnpm check:safety` passes (no signing, no secrets).
- [ ] Docs updated for any contract/behaviour change. ADR added for deviations.
- [ ] `build/verification/phase-XX.md` written with evidence.
- [ ] `build/PHASE-STATUS.md` updated.
