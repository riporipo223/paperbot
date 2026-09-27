# Documentation Index

The map of every document: what it is for, what it depends on, which build phases read it, and where it sits in the source-of-truth hierarchy.

## 1. Source-of-truth hierarchy

```text
Tier 1  Product requirements ........ docs/00, 01, 02
Tier 2  System architecture ......... docs/03, 05, 06, 09, 32 (ADRs)
Tier 3  Technical specifications .... docs/04, 07, 08, 26, 27
Tier 4  Domain specifications ....... docs/10–25, 28, 29
Tier 5  Build-phase documents ....... docs/30, 31, build/phase-*.md, BUILD-MASTER-PLAN.md
Tier 6  Implementation (code)
Tier 7  Tests
```

A higher tier overrides a lower tier. Conflicts are resolved and recorded per [CONTRIBUTING.md](CONTRIBUTING.md) §1. ADRs in [docs/32](docs/32-architecture-decision-records.md) record architecture-level resolutions and override any older statement they explicitly supersede.

## 2. Reading paths

| Goal | Path |
|---|---|
| Understand the product | 00 → 01 → 02 |
| Understand the architecture | ARCHITECTURE.md → 05 → 09 → 06 → 32 |
| Implement a phase | CLAUDE.md → this index → `build/phase-XX` → its Context docs |
| Review safety | 21 → 16 §2, §8 → 29 → 02 §1 |
| Review money correctness | 17 → 16 → 18 → 15 |
| Review strategy logic | 11 → 12 → 14 → 15 |

## 3. Root documents

| Document | Purpose |
|---|---|
| [README.md](README.md) | Project entry point, status, orientation |
| [ARCHITECTURE.md](ARCHITECTURE.md) | One-page architecture summary and key guarantees |
| [DEVELOPMENT.md](DEVELOPMENT.md) | Local setup and commands (target state after Phase 00) |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Source-of-truth rules, TDD, commits, code rules, PRs |
| [CLAUDE.md](CLAUDE.md) | Operating contract for AI coding agents |
| [DOCUMENTATION-INDEX.md](DOCUMENTATION-INDEX.md) | This map |
| [BUILD-MASTER-PLAN.md](BUILD-MASTER-PLAN.md) | The full implementation sequence (derived from the phase docs) |
| [build/PHASE-STATUS.md](build/PHASE-STATUS.md) | Build progress tracker (the single source of truth for status) |

## 4. Specification documents (`/docs`)

"Read by phases" lists the phases whose *Context* section links the document. Every phase also implements requirement IDs from 02.

| # | Document | Tier | Purpose | Upstream | Read by phases |
|---|---|---|---|---|---|
| 00 | [Project overview](docs/00-project-overview.md) | 1 | Vision, modes, principles, economic reality check, glossary | — | All (via CLAUDE.md) |
| 01 | [Product spec](docs/01-product-spec.md) | 1 | Goals, users, flows, MVP scope, non-goals, success metrics | 00 | 25 |
| 02 | [Product requirements](docs/02-product-requirements.md) | 1 | Functional requirements with stable IDs (FR-*) | 00, 01 | All (FR IDs in phase headers) |
| 03 | [System requirements](docs/03-system-requirements.md) | 2 | Real-time classes, latency budgets, time, reliability, integrity | 01, 02 | 23 (and 25 updates it) |
| 04 | [Technical spec](docs/04-technical-spec.md) | 3 | Stack choices with reasons, repo layout, module anatomy, conventions, dependency rules | 02, 03 | 00, 01 |
| 05 | [System architecture](docs/05-system-architecture.md) | 2 | Components, responsibilities, data flow, modes, concurrency, failure isolation | 02–04 | 04 |
| 06 | [Data architecture](docs/06-data-architecture.md) | 2 | Data categories, sources of truth, tiers, retention, high-frequency strategy, ownership | 03, 05 | 02, 04, 08, 22 |
| 07 | [Database schema](docs/07-database-schema.md) | 3 | Full DDL, relationships, indexes, partitions, retention, migration order | 05, 06 | 02, 04, 06, 08, 09, 10, 12 |
| 08 | [API spec](docs/08-api-spec.md) | 3 | REST endpoints, WS protocol, auth, versioning | 05, 07 | 18, 19, 20 |
| 09 | [Real-time data architecture](docs/09-real-time-data-architecture.md) | 2 | Time model, envelope, event catalog, dedup, ordering, quality, backpressure, reconnection | 03, 05–07 | 01, 03, 04, 05, 07, 08, 10, 14, 17, 22, 23 |
| 10 | [Market data spec](docs/10-market-data-spec.md) | 4 | Data tiers, watch-set policy, venues & decoders, price/liquidity math, market state model | 09, 27, 06 | 03, 05, 06, 07, 08, 11, 15 |
| 11 | [Wallet intelligence](docs/11-wallet-intelligence-spec.md) | 4 | Wallet lifecycle, backfill, round trips, score model, live smart activity | 10, 09, 07 | 05, 09, 17, 25 |
| 12 | [Wave detection](docs/12-wave-detection-spec.md) | 4 | Features, state machine, gates, score, signals | 10, 11, 09 | 10 |
| 13 | [AI agent architecture](docs/13-ai-agent-architecture.md) | 4 | Agent mapping, AI boundaries, modes, schemas, failures, replay cache | 05, 11, 12, 14, 21 | 15, 16, 17 |
| 14 | [Strategy engine](docs/14-strategy-engine-spec.md) | 4 | Decision engine, order planning, exit manager, runs, entry pauses | 12, 13, 15, 16, 26 | 15, 16 |
| 15 | [Risk engine](docs/15-risk-engine-spec.md) | 4 | Check catalog (entry/token/exit), reduction, kill switches | 14, 17, 18, 10 | 13 |
| 16 | [Paper trading](docs/16-paper-trading-spec.md) | 4 | Executor abstraction, order lifecycle, execution algorithm, recovery | 17, 15, 10, 09 | 11, 14, 26 |
| 17 | [Economics engine](docs/17-economics-engine-spec.md) | 4 | Cost components, swap math, latency/failure/MEV models, cost-ratio gate, limitations | 10, 27, 03 | 06, 07, 11, 12, 13, 14, 25 |
| 18 | [Portfolio accounting](docs/18-portfolio-accounting-spec.md) | 4 | Ledger, chart of accounts, journal templates, PnL, analytics, invariants | 16, 17, 07 | 12, 13, 14, 20, 21, 25 |
| 19 | [Backtesting & replay](docs/19-backtesting-spec.md) | 4 | One-pipeline replay, orderings, determinism, studies, sensitivity | 09, 06, 14, 13 | 04, 09, 17, 25 |
| 20 | [Dashboard spec](docs/20-dashboard-spec.md) | 4 | Screens, realtime behaviour, design system, accessibility | 01, 08, 18, 28, 22 | 19, 20, 21 |
| 21 | [Security spec](docs/21-security-spec.md) | 4 | Threat model, simulation isolation, secrets, API/UI/AI/DB security, live boundary | 02, 03, 05 | 00, 01, 03, 14, 16, 18, 19, 24, 26 |
| 22 | [Observability spec](docs/22-observability-spec.md) | 4 | Logs, metrics, system events, health, alerts, latency baselines | 03, 09, 28 | 04, 18, 21, 22, 25 |
| 23 | [Testing strategy](docs/23-testing-strategy.md) | 4 | TDD, test levels, fixtures, scenario harness, golden scenarios, failure matrix | 02, 03, domain specs | 00, 15, 23 |
| 24 | [Development environment](docs/24-development-environment.md) | 4 | Tools, versions register, env vars, commands, CI | 04, 21, 23 | 00 (all phases use the commands) |
| 25 | [Infrastructure & deployment](docs/25-infrastructure-deployment.md) | 4 | Environments, hosting options, image, topology, runbook, costs | 05, 06, 21, 22 | 24 |
| 26 | [Configuration reference](docs/26-configuration-reference.md) | 3 | Every config key with defaults and validation rules | domain specs | 01, 08, 09, 16, 21 |
| 27 | [Data provider reference](docs/27-data-provider-reference.md) | 3 | Provider facts with verification status, limits, budgets, checklist | 03, 09, 10 | 03, 05, 06, 26 |
| 28 | [Decision logging](docs/28-decision-logging-spec.md) | 4 | Causal chain, record contents, trace API, templates, reason codes | 12, 14–16, 07 | 01, 10, 13, 15, 18, 21, 22 |
| 29 | [Future live trading](docs/29-future-live-trading-architecture.md) | 4 | Shadow/live boundary and preconditions (informational) | 05, 16, 21 | 26 |
| 30 | [Development plan](docs/30-development-plan.md) | 5 | Build strategy, ordering rationale, milestones, working agreement, risks | all | All |
| 31 | [Build phases](docs/31-build-phases.md) | 5 | Phase table, dependency graph, uniform structure, global DoD | 30 | All |
| 32 | [ADRs & decision points](docs/32-architecture-decision-records.md) | 2 | All architecture decisions and the open decision register | all | All (24 resolves DP-02/03/08) |
| 33 | [Documentation audit](docs/33-documentation-audit.md) | — | Consistency audit results and the final checklist | all | — |

## 5. Build-phase documents (`/build`)

| Phase | Document | Key docs |
|---|---|---|
| 00 | [Repository & environment](build/phase-00-repository-environment.md) | 04, 21, 23, 24 |
| 01 | [Core foundation](build/phase-01-core-foundation.md) | 04, 09, 21, 26, 28 |
| 02 | [Database foundation](build/phase-02-database-foundation.md) | 06, 07 |
| 03 | [Provider verification](build/phase-03-provider-verification.md) | 09, 10, 21, 27 |
| 04 | [Event pipeline](build/phase-04-event-pipeline.md) | 05, 06, 07, 09, 19, 22 |
| 05 | [Wallet ingestion](build/phase-05-wallet-ingestion.md) | 09, 10, 11, 27 |
| 06 | [Reference data](build/phase-06-reference-data.md) | 07, 10, 17, 27 |
| 07 | [Pool streams & decoders](build/phase-07-pool-streams-decoders.md) | 09, 10, 17 |
| 08 | [Market state](build/phase-08-market-state.md) | 06, 07, 09, 10, 26 |
| 09 | [Wallet intelligence](build/phase-09-wallet-intelligence.md) | 07, 11, 19, 26 |
| 10 | [Wave detection](build/phase-10-wave-detection.md) | 07, 09, 12, 28 |
| 11 | [Economics engine](build/phase-11-economics-engine.md) | 10, 16, 17 |
| 12 | [Portfolio accounting](build/phase-12-portfolio-accounting.md) | 07, 17, 18 |
| 13 | [Risk engine](build/phase-13-risk-engine.md) | 15, 17, 18, 28 |
| 14 | [Paper trading engine](build/phase-14-paper-trading-engine.md) | 09, 16, 17, 18, 21 |
| 15 | [Decision engine](build/phase-15-decision-engine.md) | 10, 13, 14, 23, 28 |
| 16 | [AI agents](build/phase-16-ai-agents.md) | 13, 14, 21, 26 |
| 17 | [Replay & backtesting](build/phase-17-replay-backtesting.md) | 09, 11, 13, 19 |
| 18 | [Engine API](build/phase-18-engine-api.md) | 08, 21, 22, 28 |
| 19 | [Dashboard foundation](build/phase-19-dashboard-foundation.md) | 08, 20, 21 |
| 20 | [Dashboard core screens](build/phase-20-dashboard-core-screens.md) | 08, 18, 20 |
| 21 | [Dashboard intelligence screens](build/phase-21-dashboard-intel-screens.md) | 18, 20, 22, 26, 28 |
| 22 | [Observability hardening](build/phase-22-observability.md) | 06, 09, 22, 28 |
| 23 | [Integration & failure testing](build/phase-23-integration-testing.md) | 03, 09, 23 |
| 24 | [Deployment](build/phase-24-deployment.md) | 21, 25, 32 |
| 25 | [Paper validation](build/phase-25-paper-validation.md) | 01, 11, 17, 18, 19, 22 |
| 26 | [Shadow preparation](build/phase-26-shadow-preparation.md) | 16, 21, 27, 29 |

## 6. Maintenance rules

- Adding or renaming a document → update this index, [docs/31](docs/31-build-phases.md) (if it's a phase), and the upstream/downstream headers of the related docs.
- Every document keeps a header table with Status, Upstream, Downstream, and Used by phases.
- The provider reference is **27**, and decision logging is **28** ([ADR-0019](docs/32-architecture-decision-records.md#adr-0019-document-numbering)).
