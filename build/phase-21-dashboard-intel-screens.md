# Phase 21 — Dashboard: AI Command Center, Performance, System, Runs & Config, Decision Trace

| Field | Value |
|---|---|
| Milestone | M5 Operator UI |
| Depends on | 20 |
| Size | L |
| Requirements | FR-UI-001 (remaining screens), FR-LOG-001 (UI), FR-RPL-004 (UI), FR-PFL-003 (UI) |

## 1. Objective
Implement the observability and investigation screens: AI Command Center, Performance (with run comparison), System, Runs & Config (start/stop runs, replays, config versions with diff and validation), and the Decision Trace view.

## 2. Context (read first)
- [20](../docs/20-dashboard-spec.md) §3.5–3.9
- [28](../docs/28-decision-logging-spec.md) §4
- [18](../docs/18-portfolio-accounting-spec.md) §7
- [22](../docs/22-observability-spec.md) §3–7
- [26](../docs/26-configuration-reference.md) (config editor semantics)

## 3. Dependencies
Phase 20 DONE.

## 4. Inputs
Engine API (trace, analytics, system, runs, config schema), the design system, and the charts.

## 5. Outputs
Five screens + `TraceTimeline`, `ReasonCodeBadge` (code → label map from the core registry), `ConfigForm` (JSON-Schema driven), a config diff viewer, and a run comparison view.

## 6. Files To Create
```text
apps/dashboard/app/ai/page.tsx, app/performance/page.tsx, app/performance/compare/page.tsx, app/system/page.tsx, app/runs/page.tsx, app/runs/[id]/page.tsx, app/runs/config/[id]/page.tsx, app/decisions/[id]/page.tsx
apps/dashboard/components/ai/{agent-status-cards.tsx,signal-feed.tsx,rejected-list.tsx,risk-decisions-list.tsx}
apps/dashboard/components/performance/{metrics-grid.tsx,pnl-distribution.tsx,pnl-by-exit-reason.tsx,cost-breakdown.tsx,holding-time.tsx,latency-percentiles.tsx,compare-table.tsx}
apps/dashboard/components/system/{providers-panel.tsx,streams-panel.tsx,latency-panel.tsx,gaps-table.tsx,system-events-table.tsx,dead-letters.tsx}
apps/dashboard/components/runs/{runs-table.tsx,start-run-form.tsx,start-replay-form.tsx,config-form.tsx,config-diff.tsx,controls.tsx}
apps/dashboard/components/trace/{trace-timeline.tsx,trace-step.tsx,narrative.tsx,completeness.tsx}
apps/dashboard/components/ds/reason-code-badge.tsx
apps/dashboard/__tests__/... ; apps/dashboard/e2e/{trace.spec.ts,performance.spec.ts,system.spec.ts,runs.spec.ts,ai.spec.ts}
```

## 7. Files To Modify
None outside the dashboard (API gaps are handled as in Phase 20 §9).

## 8. Database Changes
None.

## 9. API Changes
None expected.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**21.1 ReasonCodeBadge**
- Tests first: every code in the core registry has a human label (a test iterates the registry). Unknown codes render raw with a warning style.

**21.2 Decision Trace view**
- Tests first: renders the narrative first, then the timeline steps in causal order with the latency per connector. Expanding a step shows the raw values. The completeness indicator shows missing links. Works for ENTER, REJECT (risk and gate failures highlighted) and EXIT decisions.

**21.3 AI Command Center**
- Tests first: agent cards (mode, rates, p95 latency, budget). The signal feed updates live. The rejected list filters by reason code. AI outputs are labelled advisory, and VETO effects are shown distinctly.

**21.4 Performance**
- Tests first: all metrics from [18](../docs/18-portfolio-accounting-spec.md) §7, with the sample-size warning when < 30 trades. Win rate is never displayed without PF/expectancy/drawdown adjacent (a layout test). Charts render. The compare view shows 2–4 runs side by side with coverage caveats.

**21.5 System**
- Tests first: providers panel (status, last message age, reconnects, credits vs budget, conserve mode), streams capacity bars, latency percentiles per stage (live), gaps/dead letters/system events tables with filters, and the clock offset.

**21.6 Runs & Config**
- Tests first: the start LIVE_FEED run form offers no mode selector other than SIMULATION (FR-SAF-006). The replay form (range, ordering, seed, AI cache scope). Stop run with confirmation. Pause/resume entries. The config view uses a JSON-Schema form with inline validation errors from `/config/validate`. "Save as new version" → hash-idempotent. The diff between two versions.

**21.7 E2E**
- Trace for the S01 entry (complete). Performance renders metrics after the fixture completes. System shows a stream disconnect when the fixture engine simulates one. Runs: create a new config version and start a replay run over the fixture range.

## 12. Acceptance Criteria
1. All nine screens from [20](../docs/20-dashboard-spec.md) §3 exist (with Phase 20).
2. "Why did we enter TOKEN_X?" is answerable in the UI from a position or decision in ≤ 2 clicks.
3. Config editing never mutates an existing version.
4. E2E specs pass.

## 13. Tests
Component and E2E.

## 14. Failure Cases
- Trace incomplete (expired raw events) → the completeness warning shows which parts use snapshots.
- Config validation errors → field-level messages. The save button is disabled.

## 15. Observability
None beyond Phase 19.

## 16. Security
Command forms require confirmation. There is no LIVE option anywhere (a test asserts no "LIVE" string in the run-form options).

## 17. Verification
```bash
pnpm --filter @paperbot/dashboard test
pnpm test:e2e
```

## 18. Commit Strategy
1. `feat(dashboard): add reason code badges`
2. `feat(dashboard): add decision trace view`
3. `feat(dashboard): add ai command center`
4. `feat(dashboard): add performance and run comparison`
5. `feat(dashboard): add system screen`
6. `feat(dashboard): add runs and config management`
7. `test(dashboard): add e2e for intelligence screens`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M5 marked.
