# 33 — Documentation Consistency Audit

| Field | Value |
|---|---|
| Status | Completed 2026-09-27 (documentation stage) |
| Upstream | All documents |
| Downstream | Operator review / approval |

## 1. Method

1. **Automated checks** (run during the audit; the scripts were throwaway):
   - Every relative markdown link and anchor across the 69 documents resolves. Result: **0 broken links**.
   - Every configuration key referenced in backticks (`wave.*`, `risk.*`, etc.) exists in [26](26-configuration-reference.md). Result: **3 mismatches found and fixed** (§3).
   - Every requirement ID (FR-*/NFR-*) referenced anywhere is defined in [02](02-product-requirements.md)/[03](03-system-requirements.md), and every FR is referenced by at least one phase document. Result: **0 undefined, 0 untraced**.
   - Every phase document contains the 19 required sections in order. Result: **27/27 pass**.
2. **Manual cross-reading** of defaults, enums, event names, state machines, migration order and module dependencies across documents (§3–4).

## 2. Contradictions in the source brief and their resolutions

| # | Issue | Resolution |
|---|---|---|
| B1 | Provider reference filename given as both `27-…` and `28-data-provider-reference.md` | Canonical **27**. Decision logging is **28** ([ADR-0019](32-architecture-decision-records.md#adr-0019-document-numbering)). |
| B2 | "Partial fills where appropriate" vs atomic Solana AMM swaps | No partial fills for single orders. Scale-out uses separate orders ([ADR-0022](32-architecture-decision-records.md#adr-0022-no-partial-fills-for-atomic-amm-swaps)). |
| B3 | Supabase suggested as DB vs event-log volume on the free tier | The schema is portable. Hosting is an open decision with a recommendation ([ADR-0018](32-architecture-decision-records.md#adr-0018-database-hosting)). |
| B4 | Vercel suggested for deployment vs a long-lived real-time engine | Dashboard on Vercel, engine on a persistent host ([ADR-0005](32-architecture-decision-records.md#adr-0005-engine-hosted-on-a-persistent-container-host-not-vercel)). |
| B5 | Skills referenced in the brief (`/ecc:*`, `/caveman:*`, SwiftUI skills) | Not installed in the documentation environment and not applicable to a TypeScript/Next.js stack (no Swift). Their intent is applied through documented rules: concise English commits split per logical unit ([CONTRIBUTING.md](../CONTRIBUTING.md) §3), a design-system section ([20](20-dashboard-spec.md) §5), architecture docs, and TDD everywhere ([23](23-testing-strategy.md) §1). |
| B6 | "Real-time" used loosely | Defined per subsystem with latency classes and provisional ceilings, to be replaced by measured baselines ([03](03-system-requirements.md) §1–2, Phase 25). |
| B7 | $2 position size vs fixed on-chain costs (rent, fees) | Made explicit: the economic reality check ([00](00-project-overview.md) §6), the cost-ratio risk gate ([17](17-economics-engine-spec.md) §8), and rent as a cash requirement. |

## 3. Inconsistencies found during authoring and fixed

| # | Where | Problem | Fix |
|---|---|---|---|
| F1 | 09, 10, 27 | SOL/USD source priority conflicted with the CoinGecko Demo monthly cap | DexScreener primary, GeckoTerminal secondary, CoinGecko 5-minute cross-check, aligned in all three docs |
| F2 | 07 §8 vs phase order | Orders (Phase 14) have FKs to decisions/risk decisions, which were planned for later migrations | `decisions` + `risk_decisions` moved to migration 0011 (Phase 13). The AI FK is added in 0013 (Phase 16). `intel.wallets` moved to 0005 (Phase 05) because ingestion needs the registry. |
| F3 | 15 | Section numbering broke the references "§4.3" from 10/12 | Renumbered. References updated to §4. |
| F4 | 05 | Event name `fill.recorded` not in the catalog | Uses `order.filled` (carries the fill) |
| F5 | 09 §5.2 | Internal events introduced by later specs were missing (`smart.sell.detected`, `wallet.swap.unparsed`, `tracked.set.changed`) | Added to the catalog |
| F6 | 26 | Keys introduced in phase docs were missing (`pipeline.module_failure_threshold`, `watch.debug_pools`, `watch.debug_auto_watch_wallet_buys`, `market_state.change_coalesce_hz`, `market_state.state_history_seconds`) | Added with defaults |
| F7 | 07 `ref.venues` | Phase 06 requires "supported ⇒ verified", but the DDL lacked it | CHECK constraint added |
| F8 | 13 §8 | `ai.max_calls_per_hour` didn't exist | Corrected to `ai.agents.<AGENT>.max_calls_per_hour` |
| F9 | 07 §4 | `freshness.pool_state_fill_max_age_ms` didn't exist | Corrected to `freshness.pool_state.fill_ms` |
| F10 | 19 §3 | Replay run parameters looked like config keys | Clarified as per-run parameters. Engine defaults under `replay.*`. |
| F11 | 20 §4 | A client-metrics endpoint was referenced but not in the API spec | Added to [08](08-api-spec.md) §4.1 |
| F12 | 26 §5 | The config immutability check depended on runtime writing to the repo | Replaced by a CI "additions only" check for `config/strategies/**` |
| F13 | 21 §2 vs Phase 00 | Partial ban of `@solana/web3.js` vs the full ban in lint | The entire package is banned. `@solana/kit` is allowed only for encoding helpers. |
| F14 | Phase 10 | Referenced a non-existent `runs` column (`dev_bootstrap`) | Wording fixed (normal run row via the temporary RunContext) |
| F15 | Phase 15 | Used `pnpm trace:audit` without a task implementing it | Task 15.11 added (trace builder + audit CLI, reused by the API in Phase 18) |
| F16 | Phase 01 vs Phase 00 | The env guard legitimately contains patterns the safety scanner bans | Explicit scanner allowlist for the env guard files |
| F17 | 23 §4 vs Phase 15 | `FakeLlmClient` doesn't exist before Phase 16 | `FakeAiAdvisor` at the port level in Phase 15 |
| F18 | 24 §5 | CLI commands used by phases were missing | Added `replay:verify`, `replay:study`, `run:start/stop`, `ai:research`, `dashboard:hash-password` |

## 4. Cross-document consistency verified (manual)

- **Defaults**: risk ([15](15-risk-engine-spec.md) ↔ [26](26-configuration-reference.md)), wave ([12](12-wave-detection-spec.md) ↔ 26), exits ([14](14-strategy-engine-spec.md) ↔ 26), wallet scoring ([11](11-wallet-intelligence-spec.md) ↔ 26), economics ([17](17-economics-engine-spec.md) ↔ 26), freshness ([09](09-real-time-data-architecture.md) §7 ↔ 26), AI ([13](13-ai-agent-architecture.md) ↔ 26): all match.
- **Enums**: order status/intent/failure reason, wave states, position status, AI agents/status, risk outcomes, run mode/source/status, and data quality agree between the DDL ([07](07-database-schema.md)) and the domain specs. Phase 01 adds a test asserting code enums equal the documented lists.
- **State machines**: wave ([12](12-wave-detection-spec.md) §5 ↔ DDL), order ([16](16-paper-trading-spec.md) §4 ↔ DDL + trigger), run ([14](14-strategy-engine-spec.md) §6 ↔ DDL).
- **Module dependencies** ([04](04-technical-spec.md) §6) agree with the phase order ([31](31-build-phases.md)) and migration order ([07](07-database-schema.md) §8).
- **Safety**: LIVE is blocked in config ([26](26-configuration-reference.md) §4), bootstrap ([21](21-security-spec.md) §3), DB ([07](07-database-schema.md) §3.4), and the executor factory ([16](16-paper-trading-spec.md) §2). The RPC allowlist ([27](27-data-provider-reference.md) §4.4, [21](21-security-spec.md) §2) and host allowlist ([26](26-configuration-reference.md) §3) are consistent.

## 5. Known open items (not defects: explicit decision points)

| Item | Where tracked |
|---|---|
| Provider facts are provisional (official sites were unreachable from the documentation environment) | [27](27-data-provider-reference.md) §1, §10. Phase 03 is a mandatory verification gate. |
| Wallet activity source (WS vs webhook) | ADR-0009 / DP-01, decided in Phases 03/05 |
| Venue support list and fee schedules | DP-04, Phases 03/06/07/11 (golden trade reproduction) |
| DB and engine hosting | ADR-0018 / DP-02 / DP-03, Phase 24 |
| Seed wallet list | DP-05 (operator-provided) |
| Uncalibrated economics parameters (latency, failure, MEV) | [17](17-economics-engine-spec.md) §9. Sensitivity analysis in Phase 25. Shadow comparison in Phase 26. |
| Latency budgets are provisional | [03](03-system-requirements.md) §2, replaced by Phase 25 baselines |

## 6. Final documentation audit checklist

**Product**
- [x] Product goal defined ([01](01-product-spec.md) §1)
- [x] MVP scope defined ([01](01-product-spec.md) §5, [00](00-project-overview.md) §7)
- [x] Non-goals defined ([01](01-product-spec.md) §6)
- [x] User flows defined ([01](01-product-spec.md) §4)

**Architecture**
- [x] System architecture defined ([05](05-system-architecture.md))
- [x] Component boundaries defined ([04](04-technical-spec.md) §3, §6; [05](05-system-architecture.md) §4)
- [x] Data flows defined ([05](05-system-architecture.md) §5; [06](06-data-architecture.md) §2)
- [x] Event flows defined ([09](09-real-time-data-architecture.md) §1, §5)

**Market data**
- [x] Provider responsibilities defined ([10](10-market-data-spec.md) §2; [27](27-data-provider-reference.md) §3)
- [x] Real-time strategy defined ([10](10-market-data-spec.md) §2–3; [09](09-real-time-data-architecture.md))
- [x] Timestamp model defined ([09](09-real-time-data-architecture.md) §2)
- [x] Latency measurement defined ([09](09-real-time-data-architecture.md) §2.2, §4; [03](03-system-requirements.md) §2)
- [x] Data freshness defined ([09](09-real-time-data-architecture.md) §7)
- [x] Failure recovery defined ([09](09-real-time-data-architecture.md) §9–14)

**Trading**
- [x] Strategy defined ([12](12-wave-detection-spec.md), [14](14-strategy-engine-spec.md))
- [x] Entry logic defined ([12](12-wave-detection-spec.md) §5–7; [14](14-strategy-engine-spec.md) §3–4)
- [x] Exit logic defined ([14](14-strategy-engine-spec.md) §5)
- [x] Risk rules defined ([15](15-risk-engine-spec.md))
- [x] Paper execution defined ([16](16-paper-trading-spec.md))
- [x] Economics defined ([17](17-economics-engine-spec.md))

**AI**
- [x] Agent responsibilities defined ([13](13-ai-agent-architecture.md) §1, §5)
- [x] AI/non-AI boundaries defined ([13](13-ai-agent-architecture.md) §2)
- [x] AI failure behaviour defined ([13](13-ai-agent-architecture.md) §7)
- [x] Output schemas defined ([13](13-ai-agent-architecture.md) §5)

**Database**
- [x] Schema defined ([07](07-database-schema.md) §3)
- [x] Relationships defined ([07](07-database-schema.md) §2)
- [x] Indexes defined ([07](07-database-schema.md) §3)
- [x] High-frequency data strategy defined ([06](06-data-architecture.md) §4; [07](07-database-schema.md) §5–7)

**Dashboard**
- [x] Main screens defined ([20](20-dashboard-spec.md) §3)
- [x] Real-time updates defined ([20](20-dashboard-spec.md) §4; [08](08-api-spec.md) §5)
- [x] Portfolio UX defined ([20](20-dashboard-spec.md) §3.1, §3.4)
- [x] AI observability defined ([20](20-dashboard-spec.md) §3.5)

**Security**
- [x] Simulation isolation defined ([21](21-security-spec.md) §2–3)
- [x] Secrets policy defined ([21](21-security-spec.md) §4)
- [x] Future live execution boundary defined ([21](21-security-spec.md) §9; [29](29-future-live-trading-architecture.md))

**Testing**
- [x] Unit tests ([23](23-testing-strategy.md) §2)
- [x] Integration tests
- [x] Event tests
- [x] Simulation tests (golden scenarios §4.1)
- [x] Replay tests ([19](19-backtesting-spec.md) §9)
- [x] E2E tests
- [x] Failure tests (§5 matrix)

**Build**
- [x] All phases defined (27 phase documents)
- [x] Dependencies defined ([31](31-build-phases.md), each phase §3)
- [x] Tasks defined (247 tasks across phases)
- [x] Acceptance criteria defined (each phase §12)
- [x] Definition of done defined (each phase §19 + global DoD)

## 7. Verdict

The documentation set is internally consistent to the extent that automated and manual checks can establish. Remaining uncertainty is concentrated in **external facts** (provider limits, venue layouts/fees), and it is gated by Phase 03 verification before any provider-specific code is written. **Ready for operator review.**
