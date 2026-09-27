# Phase Status

The single source of truth for build progress. Update this file in the same commit that completes a phase.

**Documentation stage:** COMPLETE (awaiting operator review and approval). **No phase may start until the operator approves the documentation.**

| Phase | Title | Status | Completed (date) | Commit | Verification |
|---|---|---|---|---|---|
| 00 | Repository & development environment | NOT STARTED | | | |
| 01 | Core foundation | NOT STARTED | | | |
| 02 | Database foundation | NOT STARTED | | | |
| 03 | Provider verification & client foundation | NOT STARTED | | | |
| 04 | Event pipeline core | NOT STARTED | | | |
| 05 | Wallet activity ingestion | NOT STARTED | | | |
| 06 | Reference data | NOT STARTED | | | |
| 07 | Pool streams & venue decoders | NOT STARTED | | | |
| 08 | Market state engine | NOT STARTED | | | |
| 09 | Wallet intelligence | NOT STARTED | | | |
| 10 | Wave detection | NOT STARTED | | | |
| 11 | Economics engine | NOT STARTED | | | |
| 12 | Portfolio accounting | NOT STARTED | | | |
| 13 | Risk engine | NOT STARTED | | | |
| 14 | Paper trading engine | NOT STARTED | | | |
| 15 | Decision engine & strategy runtime | NOT STARTED | | | |
| 16 | AI agent layer | NOT STARTED | | | |
| 17 | Replay & backtesting | NOT STARTED | | | |
| 18 | Engine API | NOT STARTED | | | |
| 19 | Dashboard foundation | NOT STARTED | | | |
| 20 | Dashboard core screens | NOT STARTED | | | |
| 21 | Dashboard intelligence screens | NOT STARTED | | | |
| 22 | Observability hardening | NOT STARTED | | | |
| 23 | Integration, failure & performance testing | NOT STARTED | | | |
| 24 | Deployment | NOT STARTED | | | |
| 25 | Paper trading validation | NOT STARTED | | | |
| 26 | Shadow trading preparation | NOT STARTED | | | |

Status values: `NOT STARTED` · `IN PROGRESS` · `BLOCKED (reason)` · `DONE`.

## Milestones

| Milestone | After phase | Reached |
|---|---|---|
| M0 Foundation | 02 | |
| M1 Truthful data | 08 | |
| M2 Signals | 10 | |
| M3 Headless paper trading | 15 | |
| M4 Reproducible | 17 | |
| M5 Operator UI | 21 | |
| M6 Production paper | 24 | |
| M7 Evidence | 25 | |
| M8 Shadow-ready | 26 | |

## Blockers / open decisions

See [docs/32-architecture-decision-records.md](../docs/32-architecture-decision-records.md) → *Open decision points register*.
