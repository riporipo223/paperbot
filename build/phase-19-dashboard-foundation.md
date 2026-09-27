# Phase 19 — Dashboard Foundation

| Field | Value |
|---|---|
| Milestone | M5 Operator UI |
| Depends on | 18 |
| Size | M |
| Requirements | FR-UI-004, FR-SAF-006, FR-UI-002 (client side), FR-UI-003 (global banner) |

## 1. Objective
Create the Next.js dashboard app: authentication, layout and navigation, the design system (tokens + base components), the typed server-side API client, WS token minting, the `useStream` hook with gap detection and resync, global status elements (simulation badge, entries paused chip, degraded banner), and E2E test infrastructure.

## 2. Context (read first)
- [20](../docs/20-dashboard-spec.md) §1, §2, §4, §5
- [08](../docs/08-api-spec.md) §2, §5
- [21](../docs/21-security-spec.md) §6
- ADR-0011, ADR-0012

## 3. Dependencies
Phase 18 DONE.

## 4. Inputs
The engine API + contracts package.

## 5. Outputs
- `apps/dashboard` (Next.js App Router, TS strict, Tailwind, shadcn/ui).
- Auth: login page, argon2id verification, iron-session cookie, middleware guard, login rate limit, and a `pnpm dashboard:hash-password` script.
- Server API client (`lib/engine.ts`) using `@paperbot/api-contract` for parsing.
- `/api/ws-token` route handler.
- `lib/stream/*`: singleton WS client, `useStream(channel)`, seq-gap resync, reconnect, and server-time offset.
- Design system tokens + base components: `SimBadge`, `QualityDot`, `QValue`, `KpiTile`, `StatusChip`, `DataTable`, `Banner`, `ConfirmDialog`, `JsonView`, and formatting utilities.
- The app shell with navigation and placeholder pages for all screens.
- Playwright setup with an engine running on fixture data (the E2E harness script).

## 6. Files To Create
```text
apps/dashboard/{package.json,next.config.ts,tsconfig.json,tailwind.config.ts,postcss.config.js,middleware.ts}
apps/dashboard/app/{layout.tsx,page.tsx,login/page.tsx,market/page.tsx,wallets/page.tsx,positions/page.tsx,ai/page.tsx,performance/page.tsx,system/page.tsx,runs/page.tsx,decisions/[id]/page.tsx}
apps/dashboard/app/api/{ws-token/route.ts,auth/login/route.ts,auth/logout/route.ts}
apps/dashboard/lib/{engine.ts,session.ts,auth.ts,format/*.ts,stream/{client.ts,use-stream.ts,time-offset.ts}}
apps/dashboard/components/ds/{sim-badge.tsx,quality-dot.tsx,q-value.tsx,kpi-tile.tsx,status-chip.tsx,data-table.tsx,banner.tsx,confirm-dialog.tsx,json-view.tsx}
apps/dashboard/components/shell/{top-bar.tsx,nav.tsx,degraded-banner.tsx}
apps/dashboard/styles/tokens.css
apps/dashboard/__tests__/{format.test.ts,use-stream.test.tsx,q-value.test.tsx,auth.test.ts}
apps/dashboard/e2e/{playwright.config.ts,global-setup.ts,login.spec.ts,shell.spec.ts}
scripts/e2e-engine.ts          # boots engine + DB with fixture replay feeding a LIVE_FEED-like run for E2E
scripts/hash-password.ts
```

## 7. Files To Modify
- `.dependency-cruiser.cjs` (dashboard rules active)
- CI: dashboard build + bundle secret scan + E2E job
- [24](../docs/24-development-environment.md) §2 versions

## 8. Database Changes
None.

## 9. API Changes
None (consumer only).

## 10. Environment Variables
`ENGINE_HTTP_URL`, `ENGINE_API_TOKEN`, `NEXT_PUBLIC_ENGINE_WS_URL`, `NEXT_PUBLIC_APP_ENV`, `DASHBOARD_PASSWORD_HASH`, `SESSION_SECRET`.

## 11. Implementation Tasks

**19.1 App scaffold + lint/depcruise + CSP headers**
- Tests first: a build succeeds. Depcruise forbids importing the engine/db. The CSP header is present on responses (a Next.js config test or E2E check).

**19.2 Formatting utilities**
- Tests first: USD, SOL, tiny-price subscript notation, bps, signed percentages, address shortening, and relative ages from server time (per [20](../docs/20-dashboard-spec.md) §5.3).

**19.3 Auth**
- Tests first: a correct password sets the cookie. A wrong one → 401 and the rate limit after 10 attempts/15 min. The middleware redirects unauthenticated users to /login. The session expires after 12 h.

**19.4 Engine client (server-side)**
- Tests first: responses are parsed with the contract schemas. Schema mismatches surface as errors. The service token is sent only server-side (a bundle scan test).

**19.5 WS token route + stream client**
- Tests first (mock WS server): the token is fetched from `/api/ws-token`, then connect, subscribe, snapshot, and deltas merge into the TanStack Query cache. A seq gap → resubscribe → snapshot replaces the state. Reconnect with backoff. The server-time offset is computed from `at`.

**19.6 Design system components**
- Tests first: `QValue` renders a quality label (text/tooltip, not color-only) and turns STALE when the age exceeds a threshold (fake timers). `SimBadge` is always rendered in the shell. PnL shows the sign + arrow + color.

**19.7 Shell + degraded banner + entries chip**
- Tests first: the banner appears when a `system` channel delta reports a degraded component or paused entries.

**19.8 E2E harness**
- `scripts/e2e-engine.ts` boots Postgres (Testcontainers) + engine + fixture replay. Playwright logs in and sees the shell with live status. Wired into CI.

## 12. Acceptance Criteria
1. Login-protected dashboard with the shell, simulation badge, and live system status via WS.
2. Stream gap/resync is proven by tests.
3. No engine token or secret in the client bundle (CI scan).
4. E2E login + shell test passes in CI.

## 13. Tests
Unit/component (Vitest + Testing Library), E2E (Playwright).

## 14. Failure Cases
- Engine unreachable → pages show an error state with retry. WS reconnect status shown.
- An expired WS token → the client refreshes the token and reconnects.

## 15. Observability
Client errors are logged to the console only in the MVP. Optional client metrics post in Phase 22.

## 16. Security
Per [21](../docs/21-security-spec.md) §6: CSP, no `dangerouslySetInnerHTML` (lint), HTTP-only cookies, and a login rate limit.

## 17. Verification
```bash
pnpm --filter @paperbot/dashboard test
pnpm --filter @paperbot/dashboard build
pnpm test:e2e
```

## 18. Commit Strategy
1. `feat(dashboard): scaffold next.js app with csp and lint rules`
2. `feat(dashboard): add formatting utilities`
3. `feat(dashboard): add operator auth`
4. `feat(dashboard): add engine client and ws token route`
5. `feat(dashboard): add stream client with gap resync`
6. `feat(dashboard): add design system base components`
7. `feat(dashboard): add app shell and degraded banner`
8. `test(dashboard): add e2e harness`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
