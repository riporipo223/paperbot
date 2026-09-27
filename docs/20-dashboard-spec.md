# 20 — Dashboard Specification (incl. Design System)

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [01](01-product-spec.md) §4, [08](08-api-spec.md), [18](18-portfolio-accounting-spec.md), [28](28-decision-logging-spec.md), [22](22-observability-spec.md) |
| Downstream | Phases 19–21 |
| Used by phases | 19, 20, 21, 23 (E2E) |

## 1. Architecture

```text
Browser ──HTTPS──▶ Next.js on Vercel
  │                 ├─ middleware: session check (operator auth)
  │                 ├─ server components / route handlers ──Bearer ENGINE_API_TOKEN──▶ Engine REST
  │                 └─ /api/ws-token route ──▶ Engine POST /auth/ws-token → short-lived token
  └──WSS (token)──────────────────────────────────────────────────────────▶ Engine /api/v1/stream
```

- **Next.js App Router**, TypeScript strict, React Server Components for initial data. Client components subscribe to the WS stream for live updates.
- **Data layer**: `@paperbot/api-contract` zod schemas. A typed fetch client (server-side) and a `useStream(channel)` hook (client) that merges snapshot + deltas, detects `seq` gaps, and resubscribes.
- **State**: TanStack Query for REST caches. The stream hook writes into the query cache so views don't hold duplicate stores.
- **No DB access, no engine token in the browser.** Only the short-lived WS token reaches the client.
- **Auth** (MVP, [ADR-0012](32-architecture-decision-records.md#adr-0012-single-operator-auth-for-mvp)): a single operator password (`DASHBOARD_PASSWORD_HASH`, argon2id) → encrypted, HTTP-only, SameSite=Strict session cookie (`iron-session`, `SESSION_SECRET`), 12-hour expiry. Rate-limited login. Supabase Auth or OAuth are the documented upgrade path.

## 2. Global UI elements

- **Top bar**: run selector (active run default), mode badge **"SIMULATION · virtual funds"** (always visible, FR-SAF-006), engine status dot, entries paused/active chip with reasons, SOL/USD with quality, and the server-time clock.
- **Degraded banner**: shown when any component is DEGRADED/STALE, or entries are paused automatically. Lists the reasons and links to System.
- **Navigation**: Portfolio · Live Market · Wallets · Positions · AI Command Center · Performance · System · Runs & Config. Decision Trace opens from any decision, order or trade.

## 3. Screens

### 3.1 Portfolio (`/`)
- KPI tiles: equity (USD, SOL), cash (free/reserved), rent locked, net PnL (realized/unrealized), ROI (SOL & USD), exposure, drawdown, open positions / max.
- Equity curve (SOL and USD toggle) with drawdown shading, and entry/exit markers.
- Open positions table (compact) and recent fills.
- Equity decomposition: trading PnL vs SOL revaluation ([18](18-portfolio-accounting-spec.md) §5.1).

### 3.2 Live Market (`/market`)
- Watchlist table: token (symbol **escaped, untrusted, with the mint shown**), pool/venue, price (Q), 5m change, liquidity, volume m5, buys/sells m5, tx/min, wave state and score, data-quality dots per stream.
- Wave candidates panel: active waves sorted by score, with feature mini-bars and gate status chips.
- Token detail drawer: price chart (Lightweight Charts, derived vs reference bars visually distinct, smart-wallet buys/sells as markers, our entries/exits as markers), live trades tape (tracked wallets highlighted), and safety facts.

### 3.3 Wallet Intelligence (`/wallets`)
- Table: address (short + copy), label, status, score, confidence, flags, AI style (advisory badge), 30-day trades, win rate, profit factor, last activity.
- Wallet detail: score components chart, round trips table, swaps timeline, backfill job status, status controls (TRACKED/CANDIDATE/IGNORED/BLOCKED), notes.
- Import dialog (CSV/JSON paste) with per-row validation results.

### 3.4 Active Positions (`/positions`)
- Per position: token, entry time, entry exec price, current mid (Q), PnL (SOL/USD/%), MFE/MAE, holding time, size, fees paid, slippage/impact at entry, latency breakdown, exit plan with live distances (to TP / SL / trailing level / time left), status warnings (e.g. `EXIT_BLOCKED_STALE`).
- Actions: **Close (paper)**, with confirmation.

### 3.5 AI Command Center (`/ai`)
- Agent status cards: mode, enabled, last call, success/timeout/invalid rates, latency p95, budget used today.
- Live signal feed: signals with score, key features, decision action and reasons, risk outcome, and AI verdict (VETO mode) or advisory note.
- Rejected trades list with filterable reason codes.
- Risk decisions list, with failed checks highlighted (value vs limit).

### 3.6 Performance (`/performance`)
- All metrics of [18](18-portfolio-accounting-spec.md) §7, with the sample-size warning.
- Charts: equity curve, drawdown, PnL distribution histogram, PnL by exit reason, cost breakdown stacked bar, holding-time distribution, latency percentiles by stage.
- Run comparison view (select 2–4 runs).

### 3.7 System (`/system`)
- Providers: status per channel, last message age, reconnects, rate-limit hits, credit usage vs budget (with conserve mode indicator).
- Streams: subscriptions per connection (capacity bar), per-subject freshness.
- Latency: p50/p95/p99 per stage (live-updating), detection latency distribution.
- Gaps, dead letters, recent system events (filter by level/type), clock offset.

### 3.8 Runs & Config (`/runs`)
- Runs list and detail (versions, seed, sources, status, and replay parameters).
- Start LIVE_FEED run / start replay (form). Stop run. Pause/resume entries.
- Config versions: view (JSON with schema-driven form), diff between versions, and create a new version from an existing one (validation errors inline). Config is never edited in place.

### 3.9 Decision Trace (`/decisions/[id]`)
- Renders `DecisionTrace` ([28](28-decision-logging-spec.md) §4): narrative first, then a vertical timeline (trigger swaps → wave transitions → signal → AI → decision → risk checks → order → fill → position), each step expandable to raw values, with the latency per stage annotated on the connectors, and a completeness indicator.

## 4. Real-time behaviour

- Subscribe on mount, unsubscribe on unmount. Shared WS connection per tab (singleton client).
- Gap detection → resubscribe → snapshot replaces local state.
- Reconnect with backoff. While disconnected, show "Live updates paused, reconnecting…" and grey out live values (data-age counters keep running using server time offset).
- Ages ("3.2 s ago") are computed from server timestamps plus the measured offset between server and browser clocks (from WS `at`). The browser clock is never trusted for recorded data.
- Target: ≤ 1 s from engine state change to render (UI-LIVE, [03](03-system-requirements.md) §1), measured by the `ws_broadcast` stage plus a client-side render timing sample posted to `/api/v1/system/client-metrics` (optional, Phase 22).

## 5. Design system

### 5.1 Principles
Dense, calm, and honest. It is a professional monitoring tool: information density over decoration, and clarity about uncertainty (quality, staleness, simulation) over visual flourish.

### 5.2 Tokens (Tailwind theme + CSS variables)

| Token group | Values |
|---|---|
| Color, neutral | `--bg` (#0b0d10 dark default / #ffffff light), `--surface`, `--surface-2`, `--border`, `--text`, `--text-muted` |
| Color, semantic | `--positive` (green 500-ish), `--negative` (red 500-ish), `--warning` (amber), `--info` (blue), `--sim` (violet, used only by the simulation badge) |
| Data quality | FRESH = `--positive` dot, DEGRADED = `--warning`, STALE = `--negative` outline dot, UNKNOWN = `--text-muted` hollow dot. **Always paired with a text label or tooltip** (never color alone). |
| Typography | Inter (UI), JetBrains Mono (numbers, addresses, hashes). Tabular numerals for all numeric columns. |
| Scale | Spacing 4 px base. Radius 6 px. Font sizes 12/13/14/16/20/24. |
| Motion | ≤ 150 ms transitions. Number changes flash the background for 400 ms (can be disabled; respects `prefers-reduced-motion`). |

### 5.3 Number formatting rules
- USD: `$1,234.56`. Values < $0.01 are shown with 4 significant digits.
- SOL: up to 6 decimals with trailing zeros trimmed. Lamports are shown only in detail views.
- Token prices: significant-digit formatting (e.g. `0.0₅4213` subscript-zero notation for tiny prices), with the full value in a tooltip.
- Percentages: signed (`+12.4%`, `−3.1%`). bps as `41 bps`.
- PnL uses color **plus** sign **plus** arrow glyph (accessible without color).
- Addresses: `AbCd…WxYz` with a copy button. The full address is in a tooltip, in monospace.

### 5.4 Components (shadcn/ui based)
`KpiTile`, `QualityDot`, `QValue` (value + age + quality), `SimBadge`, `StatusChip`, `DataTable` (virtualized, sortable, column visibility), `Sparkline`, `PriceChart` (Lightweight Charts wrapper), `MetricChart` (Recharts wrapper), `TraceTimeline`, `ReasonCodeBadge` (code → human label map), `ConfirmDialog`, `JsonView`, `ConfigForm` (generated from JSON Schema), `Banner`.

### 5.5 Accessibility
WCAG 2.1 AA contrast in both themes. Full keyboard navigation. Focus rings visible. Tables have proper headers. Live regions are polite and throttled.

### 5.6 Untrusted content
Token symbols and names are rendered as plain text only (React escaping), truncated to 24 characters, with non-printable and bidi-override characters stripped. Metadata URIs are never auto-loaded (no images fetched from token metadata in the MVP).

## 6. Testing

- Component tests (Vitest + Testing Library) for formatting, QValue staleness transitions, and stream-merge/gap logic.
- Playwright E2E against the engine running on fixture replay data: login, portfolio renders, position appears after a fixture entry, decision trace renders a complete chain, a stale-data banner appears when the fixture stream stalls.
- Visual regression optional (Playwright screenshots) for key screens.
