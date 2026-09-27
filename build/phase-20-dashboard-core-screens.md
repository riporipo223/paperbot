# Phase 20 — Dashboard: Portfolio, Positions, Live Market, Wallets

| Field | Value |
|---|---|
| Milestone | M5 Operator UI |
| Depends on | 19 |
| Size | L |
| Requirements | FR-UI-001 (these screens), FR-UI-002, FR-UI-003, FR-WAL-001 (UI), FR-SAF-006 |

## 1. Objective
Implement the four operational screens with live updates: Portfolio, Active Positions (including paper close), Live Market (watchlist, wave candidates, token drawer with chart), and Wallet Intelligence (list, detail, import, status controls).

## 2. Context (read first)
- [20](../docs/20-dashboard-spec.md) §3.1–3.4, §5
- [08](../docs/08-api-spec.md) §4.3–4.6, §5.2
- [18](../docs/18-portfolio-accounting-spec.md) §5.1, §7

## 3. Dependencies
Phase 19 DONE.

## 4. Inputs
The shell, design system, stream hook, and engine API.

## 5. Outputs
The four screens, the chart components (`PriceChart` with Lightweight Charts and `MetricChart` with Recharts), and tests.

## 6. Files To Create
```text
apps/dashboard/app/page.tsx (portfolio), app/positions/page.tsx, app/market/page.tsx, app/wallets/page.tsx, app/wallets/[address]/page.tsx
apps/dashboard/components/portfolio/{kpi-row.tsx,equity-curve.tsx,equity-decomposition.tsx,recent-fills.tsx}
apps/dashboard/components/positions/{positions-table.tsx,position-card.tsx,exit-plan-distances.tsx,close-position-button.tsx}
apps/dashboard/components/market/{watchlist-table.tsx,wave-candidates.tsx,token-drawer.tsx,trades-tape.tsx,safety-facts.tsx}
apps/dashboard/components/charts/{price-chart.tsx,metric-chart.tsx}
apps/dashboard/components/wallets/{wallets-table.tsx,wallet-detail.tsx,score-components.tsx,round-trips-table.tsx,import-dialog.tsx,status-control.tsx}
apps/dashboard/lib/untrusted.ts                    # sanitize/truncate token strings for display
apps/dashboard/__tests__/... ; apps/dashboard/e2e/{portfolio.spec.ts,positions.spec.ts,market.spec.ts,wallets.spec.ts}
```

## 7. Files To Modify
None outside the dashboard.

## 8. Database Changes
None.

## 9. API Changes
None expected. If a missing read field is discovered → add it to [08](../docs/08-api-spec.md) and to the contracts + engine in a separate commit, referencing this phase.

## 10. Environment Variables
None new.

## 11. Implementation Tasks

**20.1 Untrusted string display**
- Tests first: control/bidi characters stripped, truncation to 24 characters, the mint always displayed alongside, and no HTML injection (render test with `<script>` in the symbol).

**20.2 Portfolio screen**
- Tests first (component with mocked data): the KPI values and formats. The equity curve toggles SOL/USD. The decomposition shows trading PnL vs SOL revaluation. It updates on a `portfolio.updated` delta.

**20.3 Positions screen + paper close**
- Tests first: rows show entry/exit plan distances computed from live marks. The stale mark state is visible. A `EXIT_BLOCKED_STALE` warning appears. The close button → confirm dialog → POST with `Idempotency-Key` → the order appears via the stream.

**20.4 Charts**
- Tests first: `PriceChart` renders derived vs reference bars with distinct styles and markers for smart buys/sells and our entries/exits (rendering mocked in jsdom; visual check in E2E screenshots).

**20.5 Live Market screen**
- Tests first: the watchlist sorts and updates via `market:watchlist`. Wave candidates show the score breakdown bars and gate chips. The token drawer subscribes to `market:pool:<address>` on open and unsubscribes on close.

**20.6 Wallets screens**
- Tests first: table filters/sorts. The detail page shows the score components and flags, with the AI style labelled "advisory". Import dialog per-row results. Status change with confirmation.

**20.7 E2E**
- Playwright over the fixture engine: the portfolio shows the funded balance. After the S01 fixture progresses, a position appears and later closes. The market watchlist shows the token with a quality dot. The wallet import adds rows.

## 12. Acceptance Criteria
1. All four screens are functional with live updates and quality/staleness indicators.
2. Paper close works end to end, and the trace shows `EXIT_MANUAL`.
3. Untrusted strings are safely displayed (tests).
4. E2E specs pass in CI.

## 13. Tests
Component and E2E.

## 14. Failure Cases
- Stream disconnect → live values greyed, "reconnecting" shown, and the ages keep counting.
- An API error on the close action → error toast, no optimistic state change.

## 15. Observability
None beyond Phase 19.

## 16. Security
No external images. Addresses and strings are escaped. Commands require confirmation and idempotency keys.

## 17. Verification
```bash
pnpm --filter @paperbot/dashboard test
pnpm test:e2e -- --grep "portfolio|positions|market|wallets"
```

## 18. Commit Strategy
1. `feat(dashboard): add untrusted string display helpers`
2. `feat(dashboard): add portfolio screen`
3. `feat(dashboard): add positions screen with paper close`
4. `feat(dashboard): add price and metric charts`
5. `feat(dashboard): add live market screen`
6. `feat(dashboard): add wallet intelligence screens`
7. `test(dashboard): add e2e for core screens`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
