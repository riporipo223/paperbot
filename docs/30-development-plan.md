# 30 — Development Plan

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | All docs 00–29, 32 |
| Downstream | [31-build-phases.md](31-build-phases.md), `/build/phase-*.md`, [BUILD-MASTER-PLAN.md](../BUILD-MASTER-PLAN.md) |
| Used by phases | All |

## 1. Strategy

Build **bottom-up along the data flow**, keeping the system runnable and testable at every step:

1. **Foundations**: repo, core primitives, config, DB, safety guards. Nothing trades, but everything later depends on these being right (time, money, IDs, events).
2. **Truthful data**: verify providers, then build the pipeline, ingestion, reference data, pool decoding and market state. By the end the engine shows real, quality-flagged market state for watched subjects.
3. **Intelligence**: wallet scoring and wave detection produce signals (persisted, traceable). Still no trading.
4. **Trading core**: economics (pure), portfolio ledger, risk, paper executor, then the decision engine wiring it all together. The first end-to-end headless simulation happens here (Phase 15).
5. **AI and replay**: advisory agents (optional path) and deterministic replay using the same pipeline.
6. **Interface**: engine API, then the dashboard.
7. **Hardening**: observability, chaos/perf tests, deployment.
8. **Validation**: continuous paper run, baselines, sensitivity analysis, verdict report. Then shadow preparation.

## 2. Why this order

| Constraint | Consequence for ordering |
|---|---|
| Replay equivalence depends on time, randomness and events being abstracted from day one | Clock/Rng/EventEnvelope in Phase 01, EventSource abstraction in Phase 04 |
| Provider facts are unverified ([27](27-data-provider-reference.md)) | Verification spike (Phase 03) before any provider-specific ingestion |
| Paper fills need exact pool math, which needs decoders and verified fees | Decoders (07) → market state (08) → economics (11) → executor (14) |
| Executor needs funds reservation and ledger | Portfolio (12) before executor (14) |
| Orders reference decisions and risk verdicts (FKs) | Decision/risk tables in Phase 13 before orders in Phase 14 |
| Dashboard needs stable read models | API (18) after the trading core (15) and replay (17) |
| Latency budgets must be measured, not invented | Baselines in Phase 25, provisional ceilings before |

## 3. Milestones

| Milestone | After phase | Demonstrable outcome |
|---|---|---|
| **M0 Foundation** | 02 | `pnpm verify` green. Migrations apply. Safety scans enforce no-signing. Config loads and hashes. |
| **M1 Truthful data** | 08 | Engine connects to real providers, tracks wallets, decodes pool state and trades for supported venues, and exposes quality-flagged market state. Events persisted. |
| **M2 Signals** | 10 | Real smart-wallet buys produce waves and persisted, traceable entry signals (no trading). |
| **M3 Headless paper trading** | 15 | Full loop in SIMULATION on real data: decisions → risk → paper fills → ledger. Golden scenarios S01–S18 pass. |
| **M4 Reproducible** | 17 | Live-vs-replay equivalence proven. Parameter studies possible. |
| **M5 Operator UI** | 21 | All dashboard screens, including the decision trace. |
| **M6 Production paper** | 24 | Deployed engine + dashboard running continuously. |
| **M7 Evidence** | 25 | Validation report with baselines and a strategy verdict. |
| **M8 Shadow-ready** | 26 | Shadow comparison infrastructure (read-only quotes), still no signing. |

## 4. Working agreement for every phase

1. Read the phase doc and every doc it lists under **Context**.
2. Check `build/PHASE-STATUS.md`. All dependency phases must be `DONE`.
3. Inspect the repository state. Reconcile with the phase **Inputs**. Report discrepancies before coding.
4. For each task: TDD (Red → Green → Refactor), smallest correct change, commit per logical unit.
5. Run `pnpm verify` (and integration tests where Docker is available).
6. Walk through the **Acceptance Criteria** and **Definition of Done**. Record evidence in `build/verification/phase-XX.md` (commands run, outputs, metrics, deviations).
7. Update docs if behaviour or contracts changed (same commit or an immediately following docs commit). Add an ADR for any deviation.
8. Mark the phase `DONE` in `build/PHASE-STATUS.md` with the date and commit SHA.

## 5. Risk register (delivery)

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Provider limits make T1 coverage too thin on Free tier | Medium | High | Watch-set priorities, conserve mode, webhook alternative, documented upgrade (DP-07) |
| Venue layouts/fees differ from assumptions | Medium | High | Verification + golden fixtures from real trades before `supported=true` |
| Replay non-determinism creeps in | Medium | High | Lint bans (Date.now/Math.random), equivalence test in CI from Phase 17 |
| Scope creep in the dashboard | Medium | Medium | Screens fixed by [20](20-dashboard-spec.md); extras go to backlog |
| AI integration distracts from core | Low | Medium | AI off-path by default. Phase 16 after the trading core works. |
| Docker unavailable in the agent environment | Medium | Low | Unit + in-memory scenario tests don't need Docker. Integration tests run in CI. Record in verification notes. |

## 6. Backlog (explicitly deferred)

CLMM/DLMM venues · Jupiter routing in simulation · multi-strategy concurrent runs · multi-user · social sentiment ingestion · ML-based scoring · automatic wallet mining on by default · cold-archive tooling beyond basic NDJSON · Grafana dashboards · LIVE mode (separate stage).
